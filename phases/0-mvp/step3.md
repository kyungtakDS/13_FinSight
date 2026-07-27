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

`docs/ARCHITECTURE.md`의 데이터 모델 표대로 **5개** 테이블을 만든다: `profiles`, `subscriptions`, `statements`, `transactions`, `insights`.

**테이블을 더 만들지 마라.** 반복 결제와 가맹점→카테고리 맵은 둘 다 `transactions`에서 파생되는 값이라 별도 테이블이 필요 없다. 저장하면 원본과 어긋날 수 있는 사본이 하나 더 생기고, 거래를 지울 때마다 재계산해 맞춰줘야 한다. 파생값은 조회 시점에 계산한다.

요구사항:

- 모든 테이블에 `id uuid primary key default gen_random_uuid()`, `created_at timestamptz not null default now()`
- `profiles.id`는 `references auth.users(id) on delete cascade`, `plan text not null default 'free' check (plan in ('free','pro'))`
- 그 외 모든 테이블에 `user_id uuid not null references auth.users(id) on delete cascade`
- `statements`: 기존 필드에 `created_at timestamptz not null default now()`, `classification_status text not null default 'pending' check (classification_status in ('pending','processing','completed','failed'))`, `classification_started_at timestamptz`, `classified_at timestamptz`
  - **`classification_started_at`이 반드시 필요하다.** step 7이 `processing`으로 조건부 전이해 동시 실행을 막는데, 함수가 중간에 죽으면 상태가 `processing`에 영구히 남는다. 그러면 "다시 분류하기"가 계속 `409`를 반환해 **DE-05(분류 실패 복구) 경로 자체가 막힌다.** 시작 시각을 남겨야 오래된 `processing`을 회수할 수 있다.
  - 인덱스: `statements(user_id, created_at desc)` — 목록 조회와 레이트 리밋 카운트가 쓴다.
- `transactions`: `statement_id uuid not null references statements(id) on delete cascade`, `txn_date date not null`, `description text not null`, `merchant text not null`, **`amount numeric(14,0) not null`**, `category text`, `category_source text check (category_source in ('llm','user'))`, **`is_transfer boolean not null default false`**, `raw jsonb not null`, `fingerprint text not null`
  - **`is_transfer`를 빠뜨리지 마라.** step 1의 `ParsedTransaction.isTransfer`, step 2의 `isTransferRow()`, step 7의 `where ... and is_transfer = false` 가 전부 이 컬럼을 전제한다. 없으면 이체 행이 LLM 분류 입력에서 걸러지지 않고 **제3자의 실명이 외부 API로 나간다**(CLAUDE.md CRITICAL).
  - **소수점 자리를 두지 마라.** 원화에 소수점이 없고, `docs/ADR.md`와 step 5가 "정수 원 단위로 다룬다"를 전제한다. `numeric(14,2)`로 두면 반올림 오차가 들어올 자리를 만드는 셈이다.
  - **`category_source`의 의미를 정확히 지켜라:** `'user'`는 사용자가 직접 지정했다는 뜻이고 **재분류가 절대 덮어써서는 안 되는 표시**다. `'llm'`은 자동 분류된 값이라 갱신해도 된다. 가맹점→카테고리 맵도 이 컬럼으로 파생한다(step 7).
- **`unique (user_id, fingerprint)`** — 같은 명세서를 두 번 올려도 중복되지 않게
- 인덱스: `transactions(user_id, txn_date desc)`, `transactions(user_id, category)`, **`transactions(user_id, merchant)`**
  - `(user_id, merchant)`는 step 7이 매번 쓴다 — 가맹점→카테고리 맵 조회와 가맹점 단위 일괄 수정.
- `insights`: `payload jsonb not null`, `model text not null`, **`unique (user_id)`** — 사용자당 한 행. 새로 생성하면 덮어쓴다.
  - 인사이트를 명세서별로 쌓아두지 마라. **최신 한 벌이면 충분하다** — 지난달 인사이트를 다시 볼 화면이 없고, 한 행으로 두면 캐시 키와 무효화 규칙이 통째로 사라진다. 업로드할 때마다 덮어써지므로 낡을 일도 없다.
- `subscriptions`: `polar_subscription_id text unique`, `status text not null`, `current_period_end timestamptz`, `unique (user_id)`

### 2. `supabase/migrations/0002_rls.sql`

**CRITICAL — 모든 테이블에 RLS를 켜고 정책을 붙인다.** 정책 없는 테이블이 하나라도 남으면 이 step은 실패다.

- `alter table <t> enable row level security;` × 5
- 데이터 테이블(`statements`, `transactions`, `insights`)에 select / insert / update / delete 정책. 조건은 `auth.uid() = user_id`.
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

### 3-1. `supabase/migrations/0004_functions.sql`

step 7이 원자적으로 처리해야 하는 두 작업을 Postgres 함수로 만든다. **supabase-js는 여러 문장을 한 트랜잭션으로 묶지 못하므로 RPC가 아니면 원자성을 얻을 수 없다.** step 7은 SQL 마이그레이션을 만들지 않으니 여기서 준비한다.

```sql
-- 명세서 삭제 + 인사이트 무효화를 한 트랜잭션에서
create function delete_statement(p_statement_id uuid) returns int ...
-- 거래는 cascade로 지워지고, 같은 트랜잭션에서 이 사용자의 insights 행을 삭제한다.
-- 삭제된 거래 수를 반환한다.

-- 카테고리 일괄 수정 + 인사이트 무효화
create function set_merchant_category(p_merchant text, p_category text) returns int ...
```

두 함수 모두 `security invoker`로 두어 **RLS가 그대로 적용되게 한다** — `security definer`로 만들면 남의 행도 지울 수 있는 구멍이 된다. 대상 `user_id`는 인자로 받지 말고 함수 안에서 `auth.uid()`를 쓴다.

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

`docs/PRD.md`의 요금제 표가 사양이다. Free에도 `recurring_summary` / `anomaly_summary` 계산과 표시를 허용한다. `recurring_details` / `anomaly_details` / `ai_insights` / `csv_export` / `unlimited_history`만 Pro 전용이다. 탐지 전체를 하나의 기능 플래그로 잠그지 마라.

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
grep -c "enable row level security" supabase/migrations/0002_rls.sql   # 5 이상이어야 한다
```

## 검증 절차

1. 위 AC 커맨드를 실행한다. `grep` 결과가 5 미만이면 실패로 간주하고 정책을 보완하라.
2. 아키텍처 체크리스트:
   - 5개 테이블 전부 RLS가 켜져 있고 정책이 있는가?
   - **`profiles`와 `subscriptions`에 사용자 UPDATE 정책이 없는가?** (`grep -n "on profiles" supabase/migrations/0002_rls.sql` 로 확인 — `for update`가 나오면 안 된다)
   - `transactions`에 `unique (user_id, fingerprint)`가 있는가?
   - `admin.ts`에 `import "server-only"`가 있는가?
   - 환경변수를 모듈 최상단에서 읽는 코드가 없는가? (키 없이 build가 통과해야 한다)
3. 결과에 따라 `phases/0-mvp/index.json`의 step 3을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 생성한 마이그레이션 파일명과 클라이언트 3종 경로, `canAccess`/`retentionCutoff` 시그니처 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 외부 자격증명·대시보드 설정이 필요한 검증은 루트 `DEPLOY_CHECKLIST.md`에 `- [ ] (미검증)`으로 append 하고 step은 `"completed"`로 처리한다. 코드·로컬 테스트 자체를 진행할 수 없는 경우에만 `"status": "blocked"` + `"blocked_reason"` 후 중단한다.

## 금지사항

- 실제 Supabase 프로젝트에 접속하거나 마이그레이션을 적용하려 하지 마라. 이유: 이 step은 파일 작성까지다. 자격증명이 없어 반드시 실패한다.
- RLS 없는 테이블을 만들지 마라. 이유: 금융 데이터에서 계정 간 격리는 애플리케이션 코드가 아니라 DB가 보장해야 한다.
- **`profiles`에 사용자 UPDATE 정책을 만들지 마라.** 이유: RLS는 행 단위라 `plan` 컬럼까지 열린다. 사용자가 스스로 `pro`가 될 수 있다(ADR-012).
- 보관 기간이 지난 데이터를 삭제하는 로직을 만들지 마라. 이유: ADR-007 — 가리기만 하고 지우지 않는다.
- `service_role` 키를 `NEXT_PUBLIC_` 접두사가 붙은 변수로 읽지 마라. 이유: 브라우저에 노출된다.
- 기존 테스트를 깨뜨리지 마라.
