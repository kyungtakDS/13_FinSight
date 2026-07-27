# Step 9: billing

## 읽어야 할 파일

- `/docs/PRD.md` — 요금제 표가 이 step의 사양이다
- `/docs/ADR.md` — ADR-002(Polar), ADR-006(횟수 아닌 보관기간·기능), ADR-007(가리기)
- `/docs/ARCHITECTURE.md` — "플랜 게이팅"
- `src/lib/plan.ts` (step 3) — `canAccess`, `retentionCutoff`. 여기 로직을 복제하지 마라.
- `src/lib/supabase/admin.ts` (step 3) — 웹훅은 service role로 쓴다
- `src/components/PlanGate.tsx` (step 8)

## 작업

`@polar-sh/nextjs` 어댑터로 결제를 붙인다. 헬퍼는 세 개다: `Checkout`, `CustomerPortal`, `Webhooks`.

**세 파일 모두 `route.ts`이므로 대응 `route.test.ts`를 먼저 만들어야 한다** (TDD Guard).

### 1. `src/app/api/checkout/route.ts`

```ts
import { Checkout } from "@polar-sh/nextjs";

export const GET = Checkout({
  accessToken: process.env.POLAR_ACCESS_TOKEN!,
  successUrl: `${process.env.NEXT_PUBLIC_SITE_URL}/dashboard?upgraded=1`,
  server: process.env.POLAR_SERVER as "sandbox" | "production",
});
```

호출은 `/api/checkout?products=<POLAR_PRODUCT_ID_PRO>&customerExternalId=<supabase user id>` 형태다.
`customerExternalId`에 **Supabase user id를 넣는 것이 핵심**이다 — 웹훅에서 이 값으로 사용자를 찾는다. 업그레이드 버튼을 만들 때 이 파라미터를 빠뜨리지 마라.

### 2. `src/app/api/portal/route.ts`

```ts
export const GET = CustomerPortal({
  accessToken: process.env.POLAR_ACCESS_TOKEN!,
  getCustomerId: async (req) => { /* 세션에서 user id → external id */ },
  server: ...,
});
```

미인증이면 `/auth/login`으로 리다이렉트한다.

### 3. `src/app/api/webhook/polar/route.ts`

```ts
export const POST = Webhooks({
  webhookSecret: process.env.POLAR_WEBHOOK_SECRET!,
  onSubscriptionActive: async (payload) => { /* → pro */ },
  onSubscriptionRevoked: async (payload) => { /* → free */ },
  onSubscriptionCanceled: async (payload) => { /* 기간 만료까지는 pro 유지 */ },
  onSubscriptionUpdated: async (payload) => { /* 상태·기간 동기화 */ },
});
```

동기화 로직은 `src/lib/billing.ts`에 두고 (`billing.test.ts` 먼저) 라우트는 호출만 한다.

```ts
export async function syncSubscription(args: {
  db: SupabaseClient;        // service role 클라이언트를 주입받는다
  externalId: string;        // = supabase user id
  polarSubscriptionId: string;
  status: SubscriptionRecord["status"];
  currentPeriodEnd: string | null;
}): Promise<void>;
```

- `subscriptions` UPSERT (`onConflict: "user_id"`)
- `profiles.plan`을 갱신: `status`가 `active`이거나, `canceled`인데 `currentPeriodEnd`가 미래면 `"pro"`. 그 외 `"free"`.
- **해지 즉시 강등하지 마라.** 이미 결제한 기간은 사용할 수 있어야 한다.
- 웹훅은 **멱등**이어야 한다. 같은 이벤트가 두 번 와도 결과가 같아야 한다 — UPSERT로 구현하고 카운터를 증가시키는 식의 처리를 하지 마라.
- `externalId`로 프로필을 못 찾으면 조용히 무시하고 200을 반환한다. 5xx를 반환하면 Polar가 계속 재시도한다.

### 4. 업그레이드 진입점

- `src/components/UpgradeButton.tsx` (테스트 먼저) — `/api/checkout?products=...&customerExternalId=...`로 이동
- `PlanGate`의 잠금 카드와 헤더의 Free 배지에서 이 버튼을 쓴다
- 헤더에 Pro 사용자용 "구독 관리"(→ `/api/portal`) 링크

### 5. 보관 기간 만료 처리

**없다.** ADR-007에 따라 데이터를 지우지 않는다. `applyRetention`이 조회 시점에 가릴 뿐이므로, 강등되어도 삭제 작업이 필요 없고 업그레이드하면 즉시 전부 보인다. **삭제 크론이나 배치를 만들지 마라.**

### 테스트에 반드시 포함할 케이스

- `syncSubscription`: `active` → `plan: "pro"`
- `canceled` + `currentPeriodEnd` 미래 → `"pro"` 유지
- `canceled` + `currentPeriodEnd` 과거 → `"free"`
- `revoked` → `"free"`
- 같은 페이로드 2회 처리 → 결과 동일 (멱등성)
- 존재하지 않는 `externalId` → 예외 없이 종료
- 웹훅 라우트: 서명이 틀린 요청이 거부되는지 (어댑터가 처리하지만 라우트가 시크릿을 넘기는지 확인)

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트:
   - 세 `route.ts` 각각에 `route.test.ts`가 있는가?
   - `syncSubscription`이 `db`를 주입받는가?
   - `plan.ts`의 판정 로직을 복제하지 않았는가?
   - 데이터 삭제 코드가 없는가? (ADR-007)
   - 업로드/분석 횟수를 제한하는 코드가 없는가? (ADR-006)
   - 웹훅 핸들러가 실패 시 5xx를 던지지 않는가?
3. 결과에 따라 `phases/0-mvp/index.json`의 step 9를 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 세 라우트 경로와 `syncSubscription` 시그니처 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - **Polar 대시보드에서 상품·웹훅을 만들어야 하는 등 실제 자격증명이 필요하면** → `"status": "blocked"`, `"blocked_reason"`에 필요한 환경변수 이름을 적고 즉시 중단. 코드 작성과 모킹 테스트까지는 키 없이 끝낼 수 있으니, 그것까지 마친 뒤 판단하라.

## 금지사항

- 분석 횟수로 플랜을 나누지 마라. 이유: 명세서는 월 1회 나온다(ADR-006).
- 해지 즉시 Pro 기능을 끊지 마라. 이유: 이미 결제한 기간의 권리다.
- 보관 기간이 지난 데이터를 삭제하는 크론·배치를 만들지 마라. 이유: ADR-007. 조회에서 가리기만 한다.
- 웹훅에서 실패 시 5xx를 반환하지 마라. 이유: Polar가 무한 재시도한다. 처리 불가는 로그를 남기고 200을 반환한다.
- 웹훅에서 anon 클라이언트를 쓰지 마라. 이유: RLS 때문에 다른 사용자 행을 쓸 수 없다. service role 클라이언트가 필요하다.
- `POLAR_ACCESS_TOKEN` 등에 `NEXT_PUBLIC_` 접두사를 붙이지 마라. 이유: 브라우저에 노출된다.
- 기존 테스트를 깨뜨리지 마라.
