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
| `insights` | LLM 생성 요약/제안 | `user_id`, `payload`(jsonb), `model`, unique `(user_id)` — 최신 한 벌만 |

**테이블은 이 5개가 전부다.** 반복 결제, 가맹점→카테고리 맵, 카테고리별 합계는 전부 `transactions`에서 계산되는 **파생값이라 저장하지 않는다**(ADR-013). 저장하면 원본과 어긋날 수 있는 사본이 생기고, 거래를 지울 때마다 재계산해 맞춰줘야 한다. LLM 결과(`insights`)만 예외인데, 그건 계산이 아니라 돈과 시간이 드는 외부 호출의 산물이기 때문이다.

거래는 **원본 그대로 보관한다.** 명세서를 누적할수록 기간별 추이와 반복 결제 탐지가 정확해지기 때문이다. `transactions.raw`에 원본 CSV 행을 jsonb로 남겨 컬럼 매핑이 틀렸을 때 재처리할 수 있게 한다.

금액은 `numeric(14,0)` — **원 단위 정수**다. 원화에 소수점이 없고, 소수 자리를 두면 반올림 오차가 들어올 자리가 생긴다.

중복 방지: `(txn_date, amount, description, occurrence)` 해시를 `fingerprint`로 두고 `unique (user_id, fingerprint)`를 건다. `occurrence`는 **한 파일 안에서 동일 조합이 몇 번째로 등장했는지**다. 이게 없으면 같은 날 같은 가게에서 같은 금액을 두 번 결제한 정상 거래가 중복으로 지워지고, 그 데이터를 근거로 하는 `duplicate_suspect` 이상 탐지가 영원히 발동하지 않는다.

가맹점→카테고리 맵은 **테이블이 아니라 쿼리다.** 이미 분류된 거래에서 뽑아낸다:

```sql
select distinct on (merchant) merchant, category
from transactions
where user_id = $1 and category is not null
order by merchant, (category_source = 'user') desc
```

`category_source = 'user'`를 우선하므로 사용자가 고친 값이 LLM 값을 이긴다. 이 한 쿼리가 "2회차부터 새 가맹점만 LLM에 묻기"와 "수정이 다음 달에도 유지"를 둘 다 해결한다.

## 분석 파이프라인 — 무엇을 코드로, 무엇을 LLM으로

이 분리가 이 시스템의 핵심 설계다.

| 분석 | 담당 | 이유 |
|------|------|------|
| 기간별 지출 추이 | **코드** (TS 순수 함수) | 산술은 결정론적이어야 하고 검증 가능해야 한다 |
| 카테고리별 합계 | **코드** (TS 순수 함수) | 위와 동일 |
| 가맹점 → 카테고리 분류 | **LLM** | `(주)우아한형제들` → 배달. 규칙으로는 못 잡는다 |
| 반복 결제(구독) 탐지 | **코드** (규칙) | 동일 가맹점 + 금액 편차 ≤10% + 간격 25~35일 등. 규칙이 더 정확하고 공짜 |
| 이상 거래 탐지 | **코드** (통계) | 카테고리별 중앙값 대비 이상치(MAD 기반) |
| 요약 & 절약 인사이트 | **LLM** | 집계 결과를 **입력으로 받아** 한국어 서술 생성 |

LLM 호출은 두 번뿐이고 **둘 다 업로드 2단계에서만 일어난다**(ADR-011): ① 미분류 가맹점 목록 → 카테고리 배열, ② 집계 결과 요약본 → 인사이트. 거래 원문 전체를 프롬프트에 넣지 않는다. 조회 경로(Server Component, 컴포넌트)에는 LLM 호출이 하나도 없어야 한다.

## 데이터 흐름

```
[업로드 — 2단계로 나눈다. 이유는 ADR-008]

1단계  POST /api/statements                       (약 2초, 즉시 응답)
  → Supabase 세션 검증
  → lib/csv: 인코딩 감지 → 헤더 매핑 → 마스킹 → Transaction[]
  → statements + transactions INSERT (지문으로 중복 스킵)
  → 파생 맵(위 쿼리)에 있는 가맹점은 즉시 category 채움 (LLM 없이)
  → { statementId, insertedCount, duplicateCount, skippedRows,
      unknownMerchantCount } 응답 → 사용자는 여기서 이미 거래 목록을 본다

2단계  POST /api/statements/[id]/classify          (10~30초, 실패해도 1단계는 남는다)
  → 파생 맵에 없는 merchant만 추출 (이체 행 제외 — 실명이 들어간다)
  → services/anthropic.classifyMerchants() ─ LLM 호출 ①
  → transactions.category UPDATE (category_source='llm')
  → plan이 pro면:
      services/anthropic.generateInsights(집계결과) ─ LLM 호출 ②
      → insights UPSERT (onConflict: user_id)
  → { classifiedCount, insightsGenerated } 응답

[대시보드 조회]  ── LLM을 부르지 않는다 (ADR-011)
/dashboard (Server Component)
  → transactions 조회. 보관 기간 필터는 쿼리에 건다 (.gte("txn_date", cutoff))
  → lib/analytics(TS 순수 함수)로 집계: 카테고리별 / 기간별 / 전월 대비
  → 탐지(반복결제·이상거래)는 보관 기간 필터 없이 전체 이력으로 계산 (ADR-010 따름정리)
  → insights 테이블에서 캐시를 읽기만 한다. 없으면 "준비 중" + 생성 버튼
  → 차트 + 카드 렌더
```

집계는 **TS 순수 함수**가 한다(SQL 뷰나 RPC를 만들지 않는다). 조회한 거래 배열을 `src/lib/analytics/`에 넘기는 방식이라 테스트가 빠르고 결정론적이다. 보관 기간이 조회 범위를 제한하므로 메모리에 올라오는 행 수도 유계다. 이력이 커져 이 방식이 버거워지면 그때 SQL 집계로 옮긴다 — MVP 범위에서는 필요 없다.

## 플랜 게이팅

`lib/plan.ts`의 단일 함수가 판정한다. UI와 API 양쪽에서 같은 함수를 쓴다.

```ts
canAccess(plan: Plan, feature: Feature): boolean
retentionCutoff(plan: Plan): Date | null   // free → 90일 전, pro → null
```

Free 사용자의 조회 쿼리에는 `txn_date >= retentionCutoff(plan)` 필터가 붙는다. **데이터는 지우지 않는다** — 가려질 뿐이고 업그레이드하면 즉시 보인다.

게이팅은 **목록 단위**다(ADR-010). 구독 탐지·이상 거래는 Free에서도 **건수와 총액을 계산해 보여주고**, 세부 목록만 잠근다. 실제로 계산한 값이므로 가짜 데이터가 아니다. AI 인사이트는 전체가 Pro다.

## 패턴
- Server Components 기본. 인터랙션(업로드, 카테고리 수정, 차트 툴팁)이 필요한 곳만 Client Component.
- API 라우트는 얇게. 검증 → `lib/` 호출 → 응답. 비즈니스 로직을 라우트 핸들러에 넣지 않는다.
- Supabase 클라이언트는 3종: `lib/supabase/server.ts`(RSC/라우트), `lib/supabase/client.ts`(브라우저), `lib/supabase/middleware.ts`(세션 갱신).

## 상태 관리
서버 상태는 Server Component가 직접 조회한다. 클라이언트 상태는 `useState`/`useReducer`만 쓴다. 전역 상태 라이브러리를 도입하지 않는다.
