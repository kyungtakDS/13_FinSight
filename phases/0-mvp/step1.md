# Step 1: core-types

## 읽어야 할 파일

- `/CLAUDE.md`
- `/docs/ARCHITECTURE.md` — 특히 "데이터 모델" 표와 "분석 파이프라인" 표
- `/docs/PRD.md` — 요금제 표
- `/docs/UI_GUIDE.md` — 카테고리 시리즈 표 (카테고리 목록의 원천)
- `src/lib/format.ts` (step 0 산출물)

## 작업

`src/types/` 아래에 이 프로젝트의 공용 타입과 zod 스키마를 정의한다.
`src/types/`는 TDD Guard 훅 면제 구역이므로 테스트 없이 작성할 수 있다. **단, 로직을 넣지 마라** — 타입 선언과 zod 스키마 정의만 둔다. 함수 구현은 `src/lib/`에 속한다.

### `src/types/transaction.ts`

```ts
/** UI_GUIDE.md 카테고리 시리즈 표와 슬롯 순서가 일치해야 한다 */
export const CATEGORIES = [
  "식비", "교통", "구독", "쇼핑", "의료", "주거·공과금", "문화·여가", "기타",
] as const;
export type Category = (typeof CATEGORIES)[number];

export interface Transaction {
  id: string;
  statementId: string;
  txnDate: string;        // "2026-07-27"
  description: string;    // 원본 적요
  merchant: string;       // 정규화된 가맹점명
  amount: number;         // 지출은 양수, 입금/환불은 음수
  category: Category | null;
  categorySource: "llm" | "user" | null;
  raw: Record<string, string>;   // 원본 CSV 행 그대로
}

/** CSV 파싱 직후, DB 저장 전 상태 */
export type ParsedTransaction = Omit<Transaction, "id" | "statementId" | "category" | "categorySource">;
```

zod 스키마 `transactionSchema`, `parsedTransactionSchema`도 같은 파일에 export 한다.

### `src/types/statement.ts`

`Statement` 인터페이스: `id`, `userId`, `filename`, `sourceHint`(카드사 추정, nullable), `periodStart`, `periodEnd`, `rowCount`, `createdAt`.

### `src/types/analysis.ts`

분석 결과 타입. **모두 코드가 계산한 결과를 담는 그릇이지 계산 로직이 아니다.**

```ts
export interface CategoryBreakdown {
  category: Category;
  total: number;
  count: number;
  share: number;          // 0~1
}

export interface PeriodPoint {
  period: string;         // "2026-07" (월별) 또는 "2026-W30" (주별)
  total: number;
  byCategory: Partial<Record<Category, number>>;
}

export interface RecurringCharge {
  id: string;
  merchant: string;
  medianAmount: number;
  intervalDays: number;
  occurrences: number;
  firstSeenAt: string;
  lastSeenAt: string;
  status: "active" | "dormant";   // dormant = 누수 후보
}

export interface AnomalyFlag {
  transactionId: string;
  reason: "amount_outlier" | "new_merchant_high_amount" | "duplicate_suspect";
  severity: "warning" | "critical";
  detail: string;         // 한국어 설명
}

/** LLM이 생성하는 유일한 서술형 결과 */
export interface Insight {
  summary: string;              // 한국어 3~5문장
  savingSuggestions: {
    title: string;
    detail: string;
    estimatedMonthlySaving: number;
  }[];
}
```

`insightSchema`를 zod로도 정의한다 — step 6에서 Claude structured outputs의 JSON Schema로 변환해 쓴다.

### `src/types/plan.ts`

```ts
export type Plan = "free" | "pro";
export type Feature = "recurring_detection" | "anomaly_detection" | "ai_insights" | "csv_export" | "unlimited_history";
export interface SubscriptionRecord {
  userId: string;
  polarSubscriptionId: string | null;
  status: "active" | "canceled" | "revoked" | "past_due" | "none";
  currentPeriodEnd: string | null;
}
```

`FREE_RETENTION_DAYS = 90` 상수도 여기에 둔다.

### `src/types/index.ts`

위 모듈들을 re-export 하는 배럴 파일.

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트:
   - `CATEGORIES` 배열의 순서와 개수가 `docs/UI_GUIDE.md`의 카테고리 시리즈 표(8슬롯)와 정확히 일치하는가?
   - `src/types/` 안에 함수 구현이나 부수효과가 없는가? (타입 + zod 스키마 + 상수만)
   - `Transaction.raw`가 원본 CSV 행을 담도록 되어 있는가? (`docs/ADR.md` ADR-005)
3. 결과에 따라 `phases/0-mvp/index.json`의 step 1을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 정의한 타입 이름 목록과 파일 경로를 한 줄 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 사용자 개입 필요 → `"status": "blocked"` + `"blocked_reason"` 후 즉시 중단

## 금지사항

- `src/types/`에 함수 구현을 넣지 마라. 이유: TDD Guard 훅의 면제 구역이라 테스트 없이 로직이 들어가는 우회로가 된다.
- 카테고리를 임의로 추가·변경하지 마라. 이유: `UI_GUIDE.md`의 색상 슬롯과 1:1 대응해야 하고, 그 팔레트는 CVD 검증을 통과한 고정 순서다.
- DB 접근 코드나 Supabase 타입 생성기를 쓰지 마라. 이유: step 3의 범위다.
- 기존 테스트를 깨뜨리지 마라.
