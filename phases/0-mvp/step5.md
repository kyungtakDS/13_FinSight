# Step 5: analytics-engine

## 읽어야 할 파일

- `/CLAUDE.md` — **"금액 집계·기간 추이·반복 결제 탐지는 코드로 계산한다. LLM에게 산술을 시키지 말 것."**
- `/docs/ARCHITECTURE.md` — "분석 파이프라인 — 무엇을 코드로, 무엇을 LLM으로" 표
- `/docs/ADR.md` — ADR-003
- `/docs/PRD.md` — 핵심 기능 3·4번
- `src/types/analysis.ts`, `src/types/transaction.ts` (step 1 산출물) — 반환 타입을 그대로 쓴다
- `src/lib/plan.ts` (step 3 산출물)

## 작업

PRD의 분석 4종 중 **LLM이 필요 없는 3종**을 순수 함수로 구현한다. 입력은 `Transaction[]`, 출력은 `src/types/analysis.ts`의 타입들. DB 접근·네트워크 호출을 하지 않는다.

각 파일마다 테스트를 **먼저** 작성하라 (TDD Guard).

### 1. `src/lib/analytics/breakdown.ts`

```ts
export function categoryBreakdown(txns: Transaction[]): CategoryBreakdown[];
```

카테고리별 합계·건수·비중. 지출(양수 `amount`)만 집계한다 — 환불/입금(음수)은 해당 카테고리 합계에서 차감한다. 결과는 `total` 내림차순. `category`가 `null`인 거래는 `"기타"`로 집계한다. `share`는 전체 지출 합 대비 비율이고, 전체가 0이면 모두 0.

### 2. `src/lib/analytics/trend.ts`

```ts
export function periodTrend(txns: Transaction[], granularity: "month" | "week"): PeriodPoint[];
```

기간별 총지출과 카테고리별 내역. **거래가 없는 중간 기간도 `total: 0`으로 채운다** — 차트에 구멍이 생기면 추이가 왜곡된다. 기간 오름차순 정렬.

```ts
export function periodOverPeriodDelta(points: PeriodPoint[]): {
  current: PeriodPoint;
  previous: PeriodPoint | null;
  deltaAmount: number;
  deltaRatio: number | null;      // previous가 0이면 null
  topIncreasedCategory: Category | null;
} | null;
```

### 3. `src/lib/analytics/recurring.ts`

```ts
export function detectRecurring(txns: Transaction[], now?: Date): RecurringCharge[];
```

반복 결제(구독) 탐지 — **규칙 기반. LLM을 쓰지 마라.**

판정 조건 (전부 만족해야 반복 결제로 인정):
- 동일 `merchant`로 **3회 이상** 발생
- 금액 편차: 각 금액이 중앙값의 **±10% 이내**
- 결제 간격: 연속 간격의 중앙값이 `7±2`(주간), `30±5`(월간), `90±10`(분기), `365±15`(연간) 중 하나에 들어감
- `intervalDays`는 간격 중앙값을 반올림한 값

**누수 판정:** 마지막 결제 이후 `intervalDays * 2`를 초과해 다음 결제가 없으면 `status: "dormant"` — 해지했거나 데이터가 끊긴 것. 그 외는 `"active"`.
(이 프로젝트에서 "누수 후보"란 *결제는 계속되는데 안 쓰는 것*을 뜻하지만, 사용 여부는 명세서로 알 수 없다. 따라서 **활성 구독 전체를 금액 내림차순으로 보여주고 사용자가 판단하게** 한다. `dormant`는 "이미 끊긴 것"으로 별도 표시한다. UI 문구를 이 의미에 맞게 쓰라 — "쓰지 않는 구독"이라고 단정하지 마라.)

### 4. `src/lib/analytics/anomaly.ts`

```ts
export function detectAnomalies(txns: Transaction[]): AnomalyFlag[];
```

**통계 기반. LLM을 쓰지 마라.**

- `amount_outlier`: 같은 카테고리 내에서 **MAD(중앙값 절대편차)** 기준 수정 z-score `|z| > 3.5`. 표본이 5건 미만인 카테고리는 건너뛴다. MAD가 0이면 건너뛴다(0으로 나누기 방지).
- `new_merchant_high_amount`: 처음 등장한 `merchant`이면서 금액이 전체 지출 상위 5% 이상
- `duplicate_suspect`: 같은 `merchant`·같은 금액이 **24시간 이내** 2건 이상
- `severity`: 수정 z-score `> 5` 또는 duplicate는 `"critical"`, 나머지 `"warning"`
- `detail`은 **한국어**이고 숫자를 포함해야 한다. 예: `"이 카테고리 평소 결제액(중앙값 ₩23,000)의 8배입니다."`

### 5. `src/lib/analytics/index.ts`

배럴 export + 보관 기간 필터:

```ts
export function applyRetention(txns: Transaction[], plan: Plan, now?: Date): Transaction[];
```

`plan.ts`의 `retentionCutoff`를 써서 컷오프 이전 거래를 **제외**한다 (삭제가 아니라 필터 — ADR-007).

### 테스트에 반드시 포함할 케이스

- `categoryBreakdown`: 환불(음수) 차감, 전체 0일 때 share 0, `null` 카테고리 → 기타
- `periodTrend`: 거래 없는 중간 달이 0으로 채워지는지
- `detectRecurring`: 월간 구독 6회 → 탐지됨 / 금액이 30% 튀는 건 → 미탐지 / 2회만 → 미탐지 / 마지막 결제가 3개월 전 월간 구독 → `dormant`
- `detectAnomalies`: MAD가 0인 카테고리에서 예외가 나지 않는지, 표본 5건 미만 스킵
- `applyRetention`: free는 90일 이전 제외, pro는 전량 유지

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트:
   - `src/lib/analytics/` 안에서 `@anthropic-ai/sdk`나 `@supabase`를 import 하지 않는가? (순수 함수여야 한다)
   - `src/types/analysis.ts`의 타입을 재사용했는가? (중복 정의 금지)
   - `detail` 문자열이 한국어인가?
3. 결과에 따라 `phases/0-mvp/index.json`의 step 5를 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 5개 함수 시그니처와 파일 경로 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 사용자 개입 필요 → `"status": "blocked"` + `"blocked_reason"` 후 즉시 중단

## 금지사항

- 이 step에서 LLM을 호출하지 마라. 이유: CRITICAL 규칙이자 이 프로젝트의 핵심 설계다(ADR-003). 산술은 검증 가능해야 한다.
- DB에서 데이터를 읽지 마라. 이유: 순수 함수로 두어야 테스트가 결정론적이다. 조회는 step 7·8에서 붙인다.
- 부동소수점으로 금액을 누적하며 반올림하지 마라. 이유: 원 단위 오차가 누적된다. 정수 원 단위로 다루고 표시할 때만 포맷한다.
- 평균(mean)과 표준편차로 이상치를 잡지 마라. 이유: 지출 분포는 꼬리가 길어 평균이 이상치에 끌려간다. MAD를 쓰라고 명시한 이유다.
- 기존 테스트를 깨뜨리지 마라.
