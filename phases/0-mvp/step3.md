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
- `transactions`: `statement_id uuid not null references statements(id) on delete cascade`, `txn_date date not null`, `description text not null`, `merchant text not null`, **`amount numeric(14,0) not null`**, `category text`, `category_source text check (category_source in ('llm','user'))`, `raw jsonb not null`, `fingerprint text not null`
  - **소수점 자리를 두지 마라.** 원화에 소수점이 없고, `docs/ADR.md`와 step 5가 "정수 원 단위로 다룬다"를 전제한다. `numeric(14,2)`로 두면 반올림 오차가 들어올 자리를 만드는 셈이다.
  - **`category_source`의 의미를 정확히 지켜라:** `'user'`는 *이 거래 하나*를 사용자가 직접 지정했다는 뜻이고, 가맹점 맵 갱신이 **절대 덮어써서는 안 되는 표시**다. `'llm'`은 맵에서 파생된 값이라 재분류로 갱신해도 된다. 맵(`merchant_categories.source`)은 가맹점의 기본값을, 이 컬럼은 거래 단위 예외를 기록한다 — 둘은 다른 사실이다.
- **`unique (user_id, fingerprint)`** — 같은 명세서를 두 번 올려도 중복되지 않게
- 인덱스: `transactions(user_id, txn_date desc)`, `transactions(user_id, category)`, **`transactions(user_id, merchant)`**, `recurring_charges(user_id, status)`
  - `(user_id, merchant)`는 step 7의 두 쿼리가 매번 쓴다 — 미분류 가맹점 스캔과 가맹점 단위 카테고리 일괄 수정.
- `merchant_categories`: `merchant text not null`, `category text not null`, `source text not null check (source in ('llm','user'))`, `updated_at timestamptz not null default now()`, **`unique (user_id, merchant)`**. 사용자별 가맹점→카테고리 맵이다(ADR-009). 이게 있어야 2회차 업로드에서 같은 가맹점을 LLM에 다시 묻지 않고, 사용자의 수정이 다음 달에도 유지된다.
- `recurring_charges`: `merchant text not null`, `median_amount numeric(14,0)`, `interval_days int`, `occurrences int`, `first_seen_at date`, `last_seen_at date`, `status text check (status in ('active','dormant'))`, `unique (user_id, merchant)`
- `insights`: `statement_id uuid references statements(id) on delete cascade`, `kind text not null`, `payload jsonb not null`, `model text not null`, **`cache_key text not null`**, **`unique (user_id, cache_key)`**
  - step 6이 `insightCacheKey()`를 만드는데 1차 설계에는 그 값을 담을 컬럼도, UPSERT를 가능하게 할 유니크 키도 없었다. 캐시를 강제할 수단이 없으면 대시보드를 열 때마다 LLM을 다시 부른다.
- `subscriptions`: `polar_subscription_id text unique`, `status text not null`, `current_period_end timestamptz`, `unique (user_id)`

### 2. `supabase/migrations/0002_rls.sql`

**CRITICAL — 모든 테이블에 RLS를 켜고 정책을 붙인다.** 정책 없는 테이블이 하나라도 남으면 이 step은 실패다.

- `alter table <t> enable row level security;` × 7
- 데이터 테이블(`statements`, `transactions`, `merchant_categories`, `recurring_charges`, `insights`)에 select / insert / update / delete 정책. 조건은 `auth.uid() = user_id`.
- **`profiles`와 `subscriptions`에는 사용자 UPDATE / INSERT / DELETE 정책을 만들지 마라. select만 허용한다** (`profiles`는 `auth.uid() = id`). 쓰기는 웹훅과 트리거가 service role로만 한다.

  **이유 (권한 상승 취약점, ADR-012):** RLS는 **행 단위** 정책이다. `profiles`에 `for update using (auth.uid() = id)`를 붙이면 그 행의 **모든 컬럼**이 열린다. 즉 브라우저에서 이 한 줄로 결제가 우회된다:

  ```js
  await supabase.from("profiles").update({ plan: "pro" }).eq("id", myId)   // 통과해버린다
  ```

  Supabase는 `authenticated` 롤에 기본 DML 권한을 부여하므로 PostgREST로 그대로 노출된다. `plan`은 웹훅만 쓰는 컬럼이니 사용자 UPDATE 정책 자체를 만들지 않는 것이 옳다.

  나중에 `profiles`에 사용자가 편집할 컬럼(닉네임 등)이 생기면 정책을 여는 대신 **컬럼 단위 권한**을 쓴다:
  ```sql
  revoke update on public.profiles from authenticated;
  grant update (display_name) on public.profiles to authenticated;
  ```

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

컷오프 날짜는 **KST 기준**으로 계산한다(step 0의 `toKstDateString`). 조회 쿼리에 `.gte("txn_date", …)`로 들어가는 값이라 하루가 밀리면 경계일 거래가 통째로 사라진다.

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
   - **`profiles`와 `subscriptions`에 사용자 UPDATE 정책이 없는가?** (`grep -n "on profiles" supabase/migrations/0002_rls.sql` 로 확인 — `for update`가 나오면 안 된다)
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
- **`profiles`에 사용자 UPDATE 정책을 만들지 마라.** 이유: RLS는 행 단위라 `plan` 컬럼까지 열린다. 사용자가 스스로 `pro`가 될 수 있다(ADR-012).
- 보관 기간이 지난 데이터를 삭제하는 로직을 만들지 마라. 이유: ADR-007 — 가리기만 하고 지우지 않는다.
- `service_role` 키를 `NEXT_PUBLIC_` 접두사가 붙은 변수로 읽지 마라. 이유: 브라우저에 노출된다.
- 기존 테스트를 깨뜨리지 마라.
