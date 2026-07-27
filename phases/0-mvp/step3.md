# Step 3: supabase-schema

## 읽어야 할 파일

- `/CLAUDE.md` — RLS 규칙, 시크릿 규칙
- `/docs/ARCHITECTURE.md` — "데이터 모델" 표가 이 step의 사양이다
- `/docs/ADR.md` — ADR-001(RLS), ADR-005(원본 보관), ADR-007(가리기)
- `src/types/` 전체 (step 1 산출물) — 컬럼과 타입이 일치해야 한다
- `src/lib/dedupe.ts` (step 2 산출물) — `fingerprint` 컬럼의 출처

## 작업

SQL 마이그레이션과 Supabase 클라이언트 헬퍼를 만든다. **이 step에서는 실제 Supabase 프로젝트에 접속하지 않는다** — 파일만 작성한다. 적용 절차는 step 11의 README에서 문서화한다.

### 1. `supabase/migrations/0001_init.sql`

`docs/ARCHITECTURE.md`의 데이터 모델 표대로 **7개** 테이블을 만든다: `profiles`, `subscriptions`, `statements`, `transactions`, `merchant_categories`, `recurring_charges`, `insights`.

요구사항:

- 모든 테이블에 `id uuid primary key default gen_random_uuid()`, `created_at timestamptz not null default now()`
- `profiles.id`는 `references auth.users(id) on delete cascade`, `plan text not null default 'free' check (plan in ('free','pro'))`
- 그 외 모든 테이블에 `user_id uuid not null references auth.users(id) on delete cascade`
- `transactions`: `statement_id uuid not null references statements(id) on delete cascade`, `txn_date date not null`, `description text not null`, `merchant text not null`, `amount numeric(14,2) not null`, `category text`, `category_source text check (category_source in ('llm','user'))`, `raw jsonb not null`, `fingerprint text not null`
- **`unique (user_id, fingerprint)`** — 같은 명세서를 두 번 올려도 중복되지 않게
- 인덱스: `transactions(user_id, txn_date desc)`, `transactions(user_id, category)`, `recurring_charges(user_id, status)`
- `merchant_categories`: `merchant text not null`, `category text not null`, `source text not null check (source in ('llm','user'))`, `updated_at timestamptz not null default now()`, **`unique (user_id, merchant)`**. 사용자별 가맹점→카테고리 맵이다(ADR-009). 이게 있어야 2회차 업로드에서 같은 가맹점을 LLM에 다시 묻지 않고, 사용자의 수정이 다음 달에도 유지된다.
- `recurring_charges`: `merchant text not null`, `median_amount numeric(14,2)`, `interval_days int`, `occurrences int`, `first_seen_at date`, `last_seen_at date`, `status text check (status in ('active','dormant'))`, `unique (user_id, merchant)`
- `insights`: `statement_id uuid references statements(id) on delete cascade`, `kind text not null`, `payload jsonb not null`, `model text not null`
- `subscriptions`: `polar_subscription_id text unique`, `status text not null`, `current_period_end timestamptz`, `unique (user_id)`

### 2. `supabase/migrations/0002_rls.sql`

**CRITICAL — 모든 테이블에 RLS를 켜고 정책을 붙인다.** 정책 없는 테이블이 하나라도 남으면 이 step은 실패다.

- `alter table <t> enable row level security;` × 7
- 각 테이블에 select / insert / update / delete 정책. 조건은 `auth.uid() = user_id` (`profiles`는 `auth.uid() = id`).
- `subscriptions`는 **사용자가 직접 쓰지 못하게** 한다: select만 `auth.uid() = user_id`로 허용하고 insert/update/delete 정책을 만들지 않는다. 웹훅이 service role로 쓴다.

### 3. `supabase/migrations/0003_profile_trigger.sql`

`auth.users`에 행이 생기면 `profiles`에 `plan='free'`로 자동 삽입하는 트리거(`security definer`).

### 4. Supabase 클라이언트 3종

- `src/lib/supabase/client.ts` — 브라우저용, `createBrowserClient` (anon key)
- `src/lib/supabase/server.ts` — RSC/라우트 핸들러용, `createServerClient` + `cookies()`
- `src/lib/supabase/admin.ts` — service role 키를 쓰는 클라이언트. **웹훅 전용.**

**CRITICAL 구현 규칙:** 환경변수는 **모듈 최상단이 아니라 함수 내부에서** 읽어라. 최상단에서 읽으면 키 없이 `npm run build`가 실패한다. 키가 없으면 한국어 에러를 throw 한다:
`"SUPABASE_SERVICE_ROLE_KEY 환경변수가 설정되지 않았습니다."`

`admin.ts`에는 파일 최상단에 `import "server-only";`를 넣어 클라이언트 번들 유입을 컴파일 타임에 막는다.

TDD Guard 훅 때문에 세 파일 각각에 대응하는 테스트를 먼저 만들어라. 테스트는 실제 접속이 아니라 **환경변수 누락 시 한국어 에러를 던지는지**와 **필요한 env 키 이름을 읽는지**를 검증한다 (`vi.stubEnv` 사용).

### 5. `src/lib/plan.ts`

`plan.test.ts` 먼저.

```ts
export function canAccess(plan: Plan, feature: Feature): boolean;
export function retentionCutoff(plan: Plan, now?: Date): Date | null;  // free → now-90일, pro → null
```

`docs/PRD.md`의 요금제 표가 사양이다. Free에 허용되는 것은 카테고리 분류와 최근 3개월 추이뿐이고, `recurring_detection` / `anomaly_detection` / `ai_insights` / `csv_export` / `unlimited_history`는 Pro 전용이다.

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
grep -c "enable row level security" supabase/migrations/0002_rls.sql   # 7 이상이어야 한다
```

## 검증 절차

1. 위 AC 커맨드를 실행한다. `grep` 결과가 7 미만이면 실패로 간주하고 정책을 보완하라.
2. 아키텍처 체크리스트:
   - 7개 테이블 전부 RLS가 켜져 있고 정책이 있는가?
   - `transactions`에 `unique (user_id, fingerprint)`가 있는가?
   - `admin.ts`에 `import "server-only"`가 있는가?
   - 환경변수를 모듈 최상단에서 읽는 코드가 없는가? (키 없이 build가 통과해야 한다)
3. 결과에 따라 `phases/0-mvp/index.json`의 step 3을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 생성한 마이그레이션 파일명과 클라이언트 3종 경로, `canAccess`/`retentionCutoff` 시그니처 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 사용자 개입 필요 → `"status": "blocked"` + `"blocked_reason"` 후 즉시 중단

## 금지사항

- 실제 Supabase 프로젝트에 접속하거나 마이그레이션을 적용하려 하지 마라. 이유: 이 step은 파일 작성까지다. 자격증명이 없어 반드시 실패한다.
- RLS 없는 테이블을 만들지 마라. 이유: 금융 데이터에서 계정 간 격리는 애플리케이션 코드가 아니라 DB가 보장해야 한다.
- 보관 기간이 지난 데이터를 삭제하는 로직을 만들지 마라. 이유: ADR-007 — 가리기만 하고 지우지 않는다.
- `service_role` 키를 `NEXT_PUBLIC_` 접두사가 붙은 변수로 읽지 마라. 이유: 브라우저에 노출된다.
- 기존 테스트를 깨뜨리지 마라.
