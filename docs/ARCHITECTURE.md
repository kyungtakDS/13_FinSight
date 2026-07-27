# 아키텍처

## 디렉토리 구조
```
src/
├── app/                    # 페이지 + API 라우트
│   ├── (marketing)/        # 랜딩, 요금제 (비인증)
│   ├── (app)/dashboard/    # 대시보드 (인증 필요)
│   ├── auth/               # 로그인 / 콜백
│   └── api/                # 라우트 핸들러
├── components/             # UI 컴포넌트
├── types/                  # TypeScript 타입 정의 (TDD 훅 면제 구역)
├── lib/                    # 순수 로직 — csv 파서, 집계, 탐지, supabase 클라이언트
└── services/               # 외부 API 래퍼 — anthropic, polar
supabase/migrations/        # SQL 마이그레이션 (테이블 + RLS)
```

## 데이터 모델

| 테이블 | 역할 | 핵심 컬럼 |
|--------|------|-----------|
| `profiles` | auth.users 확장 | `id`(FK), `plan`(`free`\|`pro`) |
| `subscriptions` | Polar 구독 상태 미러 | `user_id`, `polar_subscription_id`, `status`, `current_period_end` |
| `statements` | 업로드 1건 = 1 row | `user_id`, `filename`, `source_hint`, `period_start`, `period_end`, `row_count` |
| `transactions` | **개별 거래 원본** | `user_id`, `statement_id`, `txn_date`, `description`, `merchant`, `amount`, `category`, `category_source`, `raw` (jsonb 원본 행) |
| `recurring_charges` | 탐지된 반복 결제 | `user_id`, `merchant`, `median_amount`, `interval_days`, `last_seen_at`, `status`(`active`\|`dormant`) |
| `insights` | LLM 생성 요약/제안 | `user_id`, `statement_id`, `kind`, `payload`(jsonb), `model` |

거래는 **원본 그대로 보관한다.** 명세서를 누적할수록 기간별 추이와 반복 결제 탐지가 정확해지기 때문이다. `transactions.raw`에 원본 CSV 행을 jsonb로 남겨 컬럼 매핑이 틀렸을 때 재처리할 수 있게 한다.

중복 방지: `(user_id, txn_date, amount, description)` 해시를 유니크 키로 두어 같은 명세서를 두 번 올려도 거래가 중복되지 않게 한다.

## 분석 파이프라인 — 무엇을 코드로, 무엇을 LLM으로

이 분리가 이 시스템의 핵심 설계다.

| 분석 | 담당 | 이유 |
|------|------|------|
| 기간별 지출 추이 | **코드** (SQL 집계) | 산술은 결정론적이어야 하고 검증 가능해야 한다 |
| 카테고리별 합계 | **코드** (SQL 집계) | 위와 동일 |
| 가맹점 → 카테고리 분류 | **LLM** | `(주)우아한형제들` → 배달. 규칙으로는 못 잡는다 |
| 반복 결제(구독) 탐지 | **코드** (규칙) | 동일 가맹점 + 금액 편차 ≤10% + 간격 25~35일 등. 규칙이 더 정확하고 공짜 |
| 이상 거래 탐지 | **코드** (통계) | 카테고리별 중앙값 대비 이상치(MAD 기반) |
| 요약 & 절약 인사이트 | **LLM** | 집계 결과를 **입력으로 받아** 한국어 서술 생성 |

LLM 호출은 두 번뿐이다: ① 미분류 가맹점 목록 → 카테고리 배열, ② 집계 결과 요약본 → 인사이트. 거래 원문 전체를 프롬프트에 넣지 않는다.

## 데이터 흐름

```
[업로드]
CSV 파일 (Client Component)
  → POST /api/statements
    → Supabase 세션 검증
    → lib/csv: 인코딩 감지 → 헤더 매핑 → 마스킹 → Transaction[]
    → statements + transactions INSERT (중복 스킵)
    → 미분류 merchant 목록 추출
    → services/anthropic.classifyMerchants() ─ LLM 호출 ①
    → transactions.category UPDATE
    → lib/recurring: 반복 결제 탐지 → recurring_charges UPSERT
    → { statementId, insertedCount } 응답

[대시보드 조회]
/dashboard (Server Component)
  → lib/analytics: SQL 집계 (카테고리별 / 월별 / 이상치)
  → plan이 pro면: services/anthropic.generateInsights(집계결과) ─ LLM 호출 ②
     (insights 테이블에 캐시. 같은 statement에 대해 재호출하지 않는다)
  → 차트 + 카드 렌더
```

## 플랜 게이팅

`lib/plan.ts`의 단일 함수가 판정한다. UI와 API 양쪽에서 같은 함수를 쓴다.

```ts
canAccess(plan: Plan, feature: Feature): boolean
retentionCutoff(plan: Plan): Date | null   // free → 90일 전, pro → null
```

Free 사용자의 조회 쿼리에는 `txn_date >= retentionCutoff(plan)` 필터가 붙는다. **데이터는 지우지 않는다** — 가려질 뿐이고 업그레이드하면 즉시 보인다.

## 패턴
- Server Components 기본. 인터랙션(업로드, 카테고리 수정, 차트 툴팁)이 필요한 곳만 Client Component.
- API 라우트는 얇게. 검증 → `lib/` 호출 → 응답. 비즈니스 로직을 라우트 핸들러에 넣지 않는다.
- Supabase 클라이언트는 3종: `lib/supabase/server.ts`(RSC/라우트), `lib/supabase/client.ts`(브라우저), `lib/supabase/middleware.ts`(세션 갱신).

## 상태 관리
서버 상태는 Server Component가 직접 조회한다. 클라이언트 상태는 `useState`/`useReducer`만 쓴다. 전역 상태 라이브러리를 도입하지 않는다.
