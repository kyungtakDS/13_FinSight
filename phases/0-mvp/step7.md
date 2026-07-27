# Step 7: statements-api

## 읽어야 할 파일

- `/CLAUDE.md` — 서버 사이드 규칙, TDD Guard가 `route.ts`를 면제하지 않는다는 점
- `/docs/ARCHITECTURE.md` — 데이터 흐름의 `[업로드]` 구간이 이 step의 사양이다
- `/docs/ADR.md` — ADR-008(2단계 분리), ADR-009(가맹점 맵)
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
  knownMerchantCount: number;    // 맵에서 바로 분류된 수
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
3. 각 거래에 `transactionFingerprint`를 붙여 `transactions`에 **upsert** — `onConflict: "user_id,fingerprint"` + `ignoreDuplicates: true`. 삽입된 행 수와 중복 수를 센다.
4. **`merchant_categories`에서 이 사용자의 맵을 읽어 아는 가맹점의 `category`를 즉시 채운다** (`category_source`는 맵의 `source`를 따른다). **여기서 LLM을 호출하지 마라.**
5. 맵에 없는 가맹점 수를 `unknownMerchantCount`로 반환하고 종료한다.

이 단계는 LLM을 타지 않으므로 2초 안에 끝난다. 사용자는 여기서 이미 거래 목록을 본다.

### 2. `src/lib/classify.ts` — 2단계: 분류와 재탐지 (`classify.test.ts` 먼저)

```ts
export async function classifyStatement(args: {
  userId: string;
  statementId: string;
  db: SupabaseClient;
  classify?: typeof classifyMerchants;   // 테스트에서 갈아끼운다
}): Promise<{ classifiedCount: number; recurringCount: number }>;
```

순서:
1. 이 사용자의 거래 중 **`category`가 null인 것의 유니크 `merchant` 목록**을 조회
2. 그중 `merchant_categories`에 없는 것만 `classifyMerchants()`에 넘긴다. **목록이 비면 LLM을 호출하지 마라.**
3. 결과를 `merchant_categories`에 UPSERT (`onConflict: "user_id,merchant"`, `source: "llm"`) — **`source: "user"`인 행은 덮어쓰지 마라.**
4. 맵을 근거로 `transactions.category`를 UPDATE. 여기서도 `category_source: "user"`인 거래는 건드리지 마라.
5. 이 사용자의 전체 거래로 `detectRecurring()` → `recurring_charges` UPSERT (`onConflict: "user_id,merchant"`)
   - 반복 결제 탐지는 **명세서 한 건이 아니라 누적 전체**를 봐야 의미가 있다.

**멱등이어야 한다.** 같은 명세서에 두 번 호출해도 결과가 같아야 사용자가 "다시 분류하기"를 눌러도 안전하다.

### 3. `src/app/api/statements/route.ts` (**`route.test.ts` 먼저**)

`POST` — `multipart/form-data`의 `file` 필드.

- 세션 없으면 `401` + `{ error: "로그인이 필요합니다." }`
- 파일 없음 / `.csv`가 아님 → `400` + 한국어 메시지
- 파일 크기 10MB 초과 → `413` + `"파일이 너무 큽니다. 10MB 이하로 올려주세요."`
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
- **`applyToMerchant`가 true면(기본값) 같은 `merchant`의 모든 거래에 반영하고 `merchant_categories`에 `source: "user"`로 UPSERT 한다** (DE-06).
  이유: 한 건만 고치면 같은 가맹점 수십 건이 그대로 남아 사용자가 같은 수정을 반복하게 되고, 다음 달에 또 틀린다. 응답에 반영된 건수를 실어 보내라 — 화면에서 `"스타벅스 42건을 식비로 바꿨습니다."`를 보여줄 수 있어야 한다.

### 6. 업로드 제한

**횟수 제한을 만들지 마라** (ADR-006). Free/Pro 모두 업로드 무제한이다. 플랜은 조회 범위와 기능으로만 갈린다.

### 테스트에 반드시 포함할 케이스

- 미인증 요청 → 401
- 같은 CSV를 두 번 ingest → 두 번째는 `insertedCount: 0`, `duplicateCount`가 행 수와 같음
- **`ingestStatement`가 LLM을 전혀 호출하지 않는지** (1단계는 LLM 없이 끝나야 한다)
- **맵에 이미 있는 가맹점은 1단계에서 분류가 채워지고 `unknownMerchantCount`에 세지 않는지**
- `category_source: "user"`인 거래가 재분류에서 제외되는지
- 미분류 가맹점이 0개면 `classifyMerchants`가 호출되지 않는지
- **`classifyStatement`를 두 번 호출해도 결과가 같은지** (멱등성)
- **분류가 실패해도 이미 저장된 거래가 남아 있는지** (롤백하지 않음)
- **`applyToMerchant: true`로 수정하면 같은 가맹점 전체가 바뀌고 `merchant_categories`에 `source: "user"`로 남는지**
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
- **이미 아는 가맹점을 LLM에 다시 묻지 마라.** 이유: 명세서는 매달 온다. `merchant_categories`를 먼저 조회하지 않으면 같은 질문에 매달 돈을 낸다(ADR-009).
- 기존 테스트를 깨뜨리지 마라.
