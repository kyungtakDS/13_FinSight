# Step 7: statements-api

## 읽어야 할 파일

- `/CLAUDE.md` — 서버 사이드 규칙, TDD Guard가 `route.ts`를 면제하지 않는다는 점
- `/docs/ARCHITECTURE.md` — 데이터 흐름의 `[업로드]` 구간이 이 step의 사양이다
- `/docs/ADR.md` — ADR-008(2단계 분리), ADR-009(파생 맵), ADR-013(파생값 비저장)
- `phases/0-mvp/USER_JOURNEY.md` — **6장 "업로드 상태 머신"의 상태별 문구 표가 응답 메시지의 사양이다**
- `src/lib/csv.ts`, `src/lib/dedupe.ts` (step 2)
- `src/lib/supabase/server.ts`, `src/lib/plan.ts` (step 3)
- `src/lib/analytics/recurring.ts` (step 5)
- `src/services/anthropic.ts` (step 6)
- `src/types/` 전체

## 작업

업로드 파이프라인을 API 라우트로 연결한다. **비즈니스 로직을 라우트 핸들러에 쓰지 마라** — 라우트는 검증 → `lib` 호출 → 응답만 한다.

**업로드는 2단계다(ADR-008).** 저장과 분류를 한 요청에 묶지 마라. 분류는 LLM 호출이라 10~30초가 걸리고 실패할 수 있는데, 묶어두면 실패했을 때 파싱 결과까지 함께 잃고 재시도할 방법이 없다.

### 1. `src/lib/ingest.ts` — 1단계: 파싱과 저장 (`ingest.test.ts` 먼저)

```ts
export interface IngestResult {
  statementId: string;
  insertedCount: number;
  duplicateCount: number;
  skippedRows: { rowNumber: number; reason: string }[];
  knownMerchantCount: number;    // 과거 거래에서 파생한 맵으로 바로 분류된 수
  unknownMerchantCount: number;  // 2단계에서 LLM에 물어야 할 수
}

export async function ingestStatement(args: {
  userId: string;
  filename: string;
  buffer: ArrayBuffer;
  db: SupabaseClient;          // 주입받는다 — 내부에서 생성하지 마라
}): Promise<IngestResult>;
```

순서:
1. `parseStatementCsv(buffer, filename)` — 실패 시 한국어 에러 전파
2. `statements` INSERT (`period_start`, `period_end`, `source_hint`, `row_count`)
3. 각 거래에 `transactionFingerprint`를 붙여 `transactions`에 **upsert** — `onConflict: "user_id,fingerprint"` + `ignoreDuplicates: true`.
   - **1,000행 단위로 나눠 보내라.** step 2가 상한을 50,000행으로 잡았는데 한 번에 보내면 `raw` jsonb까지 포함된 요청 바디가 PostgREST 한도를 넘는다. supabase-js는 자동 분할하지 않는다. 자기 사양의 상한에서 죽는 코드를 만들지 마라.
   - 삽입된 행 수는 `.select("id")`를 체이닝해 반환된 행 수로 센다(`ignoreDuplicates`는 중복 행을 반환하지 않는다). 중복 수 = 보낸 수 − 삽입된 수.
   - 청크 진행 상황을 셀 수 있으므로 1단계 응답 품질도 같이 올라간다.
4. **이미 분류된 과거 거래에서 가맹점→카테고리 맵을 만들어** 아는 가맹점의 `category`를 즉시 채운다. **여기서 LLM을 호출하지 마라.**

   맵은 **별도 테이블이 아니라 `transactions`에서 파생한다.** 이미 가지고 있는 데이터다:
   ```sql
   select distinct on (merchant) merchant, category, category_source
   from transactions
   where user_id = $1 and category is not null
   order by merchant, (category_source = 'user') desc
   ```
   `order by`의 두 번째 키가 **사용자가 고친 값을 LLM 값보다 우선**하게 만든다. 이 한 쿼리가 "2회차부터 새 가맹점만 분류"와 "수정이 다음 달에도 유지"를 둘 다 해결한다.

5. 맵에 없는 가맹점 수를 `unknownMerchantCount`로 반환하고 종료한다.

이 단계는 LLM을 타지 않으므로 2초 안에 끝난다. 사용자는 여기서 이미 거래 목록을 본다.

### 2. `src/lib/classify.ts` — 2단계: 분류와 재탐지 (`classify.test.ts` 먼저)

```ts
export async function classifyStatement(args: {
  userId: string;
  statementId: string;          // 소유권 확인용
  db: SupabaseClient;
  classify?: typeof classifyMerchants;   // 테스트에서 갈아끼운다
  makeInsights?: typeof generateInsights;
}): Promise<{ classifiedCount: number; insightsGenerated: boolean }>;
```

**이 함수는 명세서 단위가 아니라 사용자 단위로 동작한다.** 미분류 거래는 여러 명세서에 걸쳐 있을 수 있다. `statementId`는 소유권 확인에만 쓴다. 이 점을 함수 주석에 적어라 — 이름만 보면 명세서 하나만 처리하는 것처럼 읽힌다.

순서:
1. 이 사용자의 거래 중 **`category`가 null이고 `is_transfer = false`인 것의 유니크 `merchant` 목록**을 조회
   - **이체 행은 반드시 제외한다 (CRITICAL, 개인정보).** 적요에 제3자의 실명이 들어 있어 외부 API로 보내면 안 된다. 이체 행은 LLM 없이 `기타`로 채운다.
2. 1단계와 같은 파생 맵 쿼리로 이미 아는 가맹점을 걸러내고, **남은 것만** `classifyMerchants()`에 넘긴다. **목록이 비면 LLM을 호출하지 마라.**
3. 결과로 `transactions.category`를 UPDATE (`category_source: "llm"`). **`category_source: "user"`인 거래는 건드리지 마라.**
   - 분류 결과를 따로 저장할 곳은 없다. `transactions.category`가 곧 맵이다.
4. **plan이 `pro`면 `generateInsights()`를 호출해 `insights`에 UPSERT** (`onConflict: "user_id"`, 사용자당 한 행).
   - 입력은 방금 갱신된 거래로 계산한 집계 결과다. `detectRecurring`·`detectAnomalies`도 여기서 계산해 넘기되 **결과를 DB에 저장하지 마라** — 대시보드가 조회 시점에 같은 함수로 다시 계산한다. 순수 함수라 재계산 비용이 없고, 저장하면 거래를 지울 때마다 맞춰줘야 하는 사본이 하나 더 생긴다.
   - **인사이트 생성은 여기가 유일한 자리다.** 대시보드 렌더에서 LLM을 부르지 마라 — 페이지 응답이 20~30초 막힌다(ADR-008, ADR-011).
   - 이 호출이 실패해도 1~3단계 결과는 유지하고 `insightsGenerated: false`로 반환한다.

**멱등이어야 한다.** 같은 명세서에 두 번 호출해도 결과가 같아야 사용자가 "다시 분류하기"를 눌러도 안전하다.

### 3. `src/app/api/statements/route.ts` (**`route.test.ts` 먼저**)

`POST` — `multipart/form-data`의 `file` 필드.

- 세션 없으면 `401` + `{ error: "로그인이 필요합니다." }`
- 파일 없음 / `.csv`가 아님 → `400` + 한국어 메시지
- 파일 크기 10MB 초과 → `413` + `"파일이 너무 큽니다. 10MB 이하로 올려주세요."` (`file.size`로 **버퍼에 읽기 전에** 판정한다)
- 파일 최상단에 `export const runtime = "nodejs";` — `iconv-lite`가 Node `Buffer`를 쓴다. Edge 런타임으로 잡히면 CP949 디코딩이 죽는다.
- 성공 → `200` + `IngestResult`
- 파싱 에러 → `400` + 파서가 던진 한국어 메시지
- 그 외 예외 → `500` + `"분석 중 오류가 발생했습니다."` (내부 에러 원문을 클라이언트에 노출하지 마라. 서버 로그에만 남긴다.)

`GET` — 이 사용자의 `statements` 목록을 최신순으로. `applyRetention`을 거치지 않는다 (명세서 목록 자체는 가리지 않고, 거래 조회에서 가린다).

### 4. `src/app/api/statements/[id]/classify/route.ts` (**`route.test.ts` 먼저**)

`POST` — `classifyStatement`를 호출한다. 클라이언트가 1단계 응답을 받은 직후에 부르고, 대시보드의 "다시 분류하기" 버튼도 같은 엔드포인트를 부른다(DE-05).

- 세션 없으면 `401`
- 남의 `statementId`면 `404` (RLS가 이미 막지만 응답 코드를 위해 확인한다)
- LLM 실패 → `502` + `"카테고리 분류에 실패했습니다. 거래는 모두 저장되어 있습니다."`
  **이때 이미 저장된 거래를 롤백하지 마라.** 미분류 상태로 남기는 것이 이 설계의 목적이다.
- `export const maxDuration = 120;` 을 선언한다. 기본값은 넉넉하지만 이 라우트가 가장 오래 걸린다는 사실을 코드에 남겨둔다.

### 5. `src/app/api/transactions/[id]/category/route.ts` (**`route.test.ts` 먼저**)

`PATCH` — 사용자가 카테고리를 직접 고칠 때. body `{ category: Category; applyToMerchant?: boolean }`.

- 세션 검증, 소유권은 RLS가 보장하지만 응답 코드를 위해 명시적으로도 확인한다
- `category_source: "user"`로 저장 — 이후 LLM 재분류가 덮어쓰지 못한다
- 유효하지 않은 카테고리면 `400` + `"올바르지 않은 카테고리입니다."`
- **`applyToMerchant`가 true면(기본값) 같은 `merchant`의 모든 거래를 `category_source: "user"`로 함께 UPDATE 한다** (DE-06). 이렇게 하면 다음 달 업로드에서 파생 맵이 자동으로 이 값을 물려받는다 — 따로 저장할 곳이 필요 없다.
  이유: 한 건만 고치면 같은 가맹점 수십 건이 그대로 남아 사용자가 같은 수정을 반복하게 되고, 다음 달에 또 틀린다. 응답에 반영된 건수를 실어 보내라 — 화면에서 `"스타벅스 42건을 식비로 바꿨습니다."`를 보여줄 수 있어야 한다.

### 6. `src/app/api/statements/[id]/route.ts` (**`route.test.ts` 먼저**)

`DELETE` — 잘못 올린 명세서를 지운다.

엉뚱한 파일을 올렸을 때 되돌릴 방법이 지금 없다. 카테고리 수정으로도, 재업로드로도 안 되고 유일한 탈출구가 계정 삭제다. `transactions`가 `on delete cascade`라 실제 코드는 짧다.

- 세션 검증 + 소유권 확인(RLS가 막지만 404를 주기 위해 명시적으로도)
- 응답에 삭제된 거래 수를 실어 `"명세서와 거래 234건을 삭제했습니다."`를 보여줄 수 있게 한다
- 삭제 후 추가 정리 작업은 **없다.** 반복 결제·이상 거래·가맹점 맵이 전부 `transactions`에서 조회 시점에 파생되므로 자동으로 맞는다. 파생값을 저장하지 않기로 한 설계의 이득이다.

### 7. `src/app/api/export/route.ts` (**`route.test.ts` 먼저**)

`GET` — Pro 전용 CSV 내보내기. `docs/PRD.md` 요금제 표가 파는 기능인데 1차 설계의 어느 step에도 구현이 없었다.

- `canAccess(plan, "csv_export")`가 false면 `403` + `"Pro 플랜에서 이용할 수 있습니다."`
- 보관 기간 필터를 적용한 거래를 UTF-8 BOM 포함 CSV로 반환 (BOM이 없으면 엑셀에서 한글이 깨진다)
- 헤더: `거래일자,적요,가맹점,금액,카테고리`
- `Content-Disposition: attachment; filename="finsight-2026-07-27.csv"`

**CSV 수식 인젝션을 막아라 (CRITICAL).**

```ts
function csvCell(v: string): string {
  const s = /^[=+\-@\t\r]/.test(v) ? `'${v}` : v;
  return `"${s.replace(/"/g, '""')}"`;
}
```

가맹점명은 **사용자가 올린 파일에서 온 문자열**이다. `=HYPERLINK("http://evil","클릭")` 같은 값이 그대로 나가면 내려받은 CSV를 엑셀에서 열 때 수식으로 실행된다. 내보낸 파일은 회계 담당자에게 전달되기도 한다.

### 8. 업로드 제한

**횟수 제한도, 레이트 리밋도 만들지 마라** (ADR-006). Free/Pro 모두 업로드 무제한이고, 플랜은 조회 범위와 기능으로만 갈린다.

남용 방지 상한은 **공개 배포 전 과제**로 미룬다. 지금은 사용자가 몇 명뿐이고, LLM 비용의 실질 상한은 Anthropic 콘솔의 지출 한도가 잡아준다. 카운팅 쿼리와 임계값 튜닝을 지금 넣으면 정상 사용자를 잠글 위험만 생긴다. `docs/SECURITY.md`에 "공개 전 추가할 것"으로 한 줄 남긴다.

### 테스트에 반드시 포함할 케이스

- 미인증 요청 → 401
- 같은 CSV를 두 번 ingest → 두 번째는 `insertedCount: 0`, `duplicateCount`가 행 수와 같음
- **`ingestStatement`가 LLM을 전혀 호출하지 않는지** (1단계는 LLM 없이 끝나야 한다)
- **과거 거래에서 파생한 맵으로 1단계에서 분류가 채워지고 `unknownMerchantCount`에 세지 않는지**
- **파생 맵이 `category_source: "user"` 값을 `"llm"` 값보다 우선하는지**
- `category_source: "user"`인 거래가 재분류에서 제외되는지
- 미분류 가맹점이 0개면 `classifyMerchants`가 호출되지 않는지
- **`classifyStatement`를 두 번 호출해도 결과가 같은지** (멱등성)
- **분류가 실패해도 이미 저장된 거래가 남아 있는지** (롤백하지 않음)
- **인사이트 생성이 실패해도 분류·반복결제 결과가 유지되고 `insightsGenerated: false`가 반환되는지**
- **plan이 `free`면 `generateInsights`가 호출되지 않는지**
- **`isTransfer: true`인 거래의 `merchant`가 `classifyMerchants`에 넘어가지 않는지** (개인정보 — 반드시 단언하라)
- **5만 행 입력이 1,000행 단위로 나뉘어 전송되는지** (호출 횟수 단언)
- **내보내기: `=cmd`로 시작하는 가맹점명이 `'=cmd`로 이스케이프되는지**
- **내보내기: Free 플랜은 403인지**
- **명세서 삭제: 남의 `statementId`는 404, 거래가 함께 사라지는지**
- **`applyToMerchant: true`로 수정하면 같은 가맹점 전체가 `category_source: "user"`로 바뀌는지**
- 파서가 던진 한국어 에러가 400 응답 본문에 그대로 실리는지
- 예상치 못한 예외의 원문이 응답에 **노출되지 않는지**

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트:
   - 각 `route.ts` 옆에 `route.test.ts`가 있는가? (3개)
   - 라우트 핸들러에 파싱·집계 로직이 들어가 있지 않은가? (`lib/`로 위임)
   - `ingestStatement` / `classifyStatement`가 `db`를 주입받는가? (내부 생성 금지)
   - **1단계(`ingestStatement`)에 LLM 호출이 없는가?**
   - 업로드 횟수를 제한하는 코드가 없는가?
   - `transactions` upsert가 `user_id,fingerprint` 충돌을 무시하는가?
3. 결과에 따라 `phases/0-mvp/index.json`의 step 7을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 라우트 경로와 `ingestStatement` 시그니처 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 사용자 개입 필요 → `"status": "blocked"` + `"blocked_reason"` 후 즉시 중단

## 금지사항

- 업로드/분석 **횟수 제한을 넣지 마라**. 이유: 명세서는 월 1회 나온다. 횟수 제한은 제품을 안 쓰게 만들 뿐이다(ADR-006).
- 내부 예외 메시지나 스택 트레이스를 클라이언트 응답에 넣지 마라. 이유: DB 구조와 키 정보가 새어나간다.
- 원본 CSV 파일을 스토리지에 업로드하지 마라. 이유: `transactions.raw`에 행 단위로 보관하면 충분하고, 파일 스토리지는 추가 유출 경로다.
- 라우트 핸들러 안에서 Supabase 클라이언트를 새로 만들지 마라. 이유: `lib/supabase/server.ts`가 쿠키 처리를 이미 한다.
- 사용자가 수정한 카테고리를 LLM 결과로 덮어쓰지 마라. 이유: 사용자 입력이 항상 우선이다.
- **파싱·저장과 LLM 분류를 한 요청에 묶지 마라.** 이유: 분류가 실패하면 파싱 결과까지 잃고, 사용자는 30초를 기다린 끝에 아무것도 얻지 못한 채 처음부터 다시 해야 한다(ADR-008).
- **분류 실패 시 이미 저장한 거래를 롤백하지 마라.** 이유: 미분류 상태로 남기고 재시도하게 두는 것이 이 설계의 목적이다.
- **이미 아는 가맹점을 LLM에 다시 묻지 마라.** 이유: 명세서는 매달 온다. 과거 거래에서 파생한 맵을 먼저 보지 않으면 같은 질문에 매달 돈을 낸다(ADR-009).
- **파생 가능한 값을 저장하는 테이블을 새로 만들지 마라.** 이유: 반복 결제·가맹점 맵은 `transactions`에서 계산되는 값이다. 저장하면 원본과 어긋날 수 있는 사본이 생기고, 거래를 지울 때마다 맞춰줘야 한다.
- **이체 행의 `merchant`를 LLM에 보내지 마라.** 이유: 제3자의 실명이다. 우리가 공지한 "가맹점명 전송"의 범위를 넘는다.
- 5만 행을 한 번에 upsert 하지 마라. 이유: 요청 바디 한도를 넘어 자기 사양의 상한에서 죽는다.
- 내보내기 CSV 셀을 이스케이프 없이 쓰지 마라. 이유: `=`로 시작하는 가맹점명이 엑셀에서 수식으로 실행된다.
- CSV 파싱 라우트를 Edge 런타임으로 두지 마라. 이유: `iconv-lite`가 Node `Buffer`를 쓴다.
- 기존 테스트를 깨뜨리지 마라.
