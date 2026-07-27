# Step 7: statements-api

## 읽어야 할 파일

- `/CLAUDE.md` — 서버 사이드 규칙, TDD Guard가 `route.ts`를 면제하지 않는다는 점
- `/docs/ARCHITECTURE.md` — 데이터 흐름의 `[업로드]` 구간이 이 step의 사양이다
- `src/lib/csv.ts`, `src/lib/dedupe.ts` (step 2)
- `src/lib/supabase/server.ts`, `src/lib/plan.ts` (step 3)
- `src/lib/analytics/recurring.ts` (step 5)
- `src/services/anthropic.ts` (step 6)
- `src/types/` 전체

## 작업

업로드 파이프라인을 API 라우트로 연결한다. **비즈니스 로직을 라우트 핸들러에 쓰지 마라** — 라우트는 검증 → `lib` 호출 → 응답만 한다. 파이프라인 본체는 `src/lib/ingest.ts`에 둔다.

### 1. `src/lib/ingest.ts` (`ingest.test.ts` 먼저)

```ts
export interface IngestResult {
  statementId: string;
  insertedCount: number;
  duplicateCount: number;
  skippedRows: { rowNumber: number; reason: string }[];
  classifiedMerchantCount: number;
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
4. 이 사용자의 **`category`가 null인 거래의 유니크 `merchant` 목록**을 조회 → `classifyMerchants()` 호출 → `transactions.category` UPDATE (`category_source: "llm"`).
   - 목록이 비면 LLM을 호출하지 마라.
   - **이미 사용자가 수정한 거래(`category_source: "user"`)는 덮어쓰지 마라.**
5. 이 사용자의 전체 거래를 조회해 `detectRecurring()` → `recurring_charges` UPSERT (`onConflict: "user_id,merchant"`)
   - 반복 결제 탐지는 **명세서 한 건이 아니라 누적 전체**를 봐야 의미가 있다.

의존성(`db`, 분류 함수)을 인자로 주입해 테스트에서 갈아끼울 수 있게 하라.

### 2. `src/app/api/statements/route.ts` (**`route.test.ts` 먼저**)

`POST` — `multipart/form-data`의 `file` 필드.

- 세션 없으면 `401` + `{ error: "로그인이 필요합니다." }`
- 파일 없음 / `.csv`가 아님 → `400` + 한국어 메시지
- 파일 크기 10MB 초과 → `413` + `"파일이 너무 큽니다. 10MB 이하로 올려주세요."`
- 성공 → `200` + `IngestResult`
- 파싱 에러 → `400` + 파서가 던진 한국어 메시지
- 그 외 예외 → `500` + `"분석 중 오류가 발생했습니다."` (내부 에러 원문을 클라이언트에 노출하지 마라. 서버 로그에만 남긴다.)

`GET` — 이 사용자의 `statements` 목록을 최신순으로. `applyRetention`을 거치지 않는다 (명세서 목록 자체는 가리지 않고, 거래 조회에서 가린다).

### 3. `src/app/api/transactions/[id]/category/route.ts` (**`route.test.ts` 먼저**)

`PATCH` — 사용자가 카테고리를 직접 고칠 때. body `{ category: Category }`.
- 세션 검증, 소유권은 RLS가 보장하지만 응답 코드를 위해 명시적으로도 확인한다
- `category_source: "user"`로 저장 — 이후 LLM 재분류가 덮어쓰지 못한다
- 유효하지 않은 카테고리면 `400` + `"올바르지 않은 카테고리입니다."`

### 4. 업로드 제한

**횟수 제한을 만들지 마라** (ADR-006). Free/Pro 모두 업로드 무제한이다. 플랜은 조회 범위와 기능으로만 갈린다.

### 테스트에 반드시 포함할 케이스

- 미인증 요청 → 401
- 같은 CSV를 두 번 ingest → 두 번째는 `insertedCount: 0`, `duplicateCount`가 행 수와 같음
- `category_source: "user"`인 거래가 재분류에서 제외되는지
- 미분류 가맹점이 0개면 `classifyMerchants`가 호출되지 않는지
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
   - 각 `route.ts` 옆에 `route.test.ts`가 있는가?
   - 라우트 핸들러에 파싱·집계 로직이 들어가 있지 않은가? (`lib/ingest.ts`로 위임)
   - `ingestStatement`가 `db`를 주입받는가? (내부 생성 금지)
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
- 기존 테스트를 깨뜨리지 마라.
