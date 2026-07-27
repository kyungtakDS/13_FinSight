# Step 4: auth-flow

## 읽어야 할 파일

- `/CLAUDE.md` — 언어 규칙(UI 문자열 한국어), 서버 사이드 규칙
- `/docs/ARCHITECTURE.md` — 디렉토리 구조의 `app/(marketing)`, `app/(app)/dashboard`, `app/auth`
- `/docs/UI_GUIDE.md` — 입력 필드·버튼 클래스, 색상
- `phases/0-mvp/USER_JOURNEY.md` — **9장 "계정 상태" 다이어그램이 이 step의 사양이다.** DE-01·DE-02·DE-09
- `src/lib/supabase/{client,server}.ts` (step 3 산출물) — 새로 만들지 말고 이것을 쓴다
- `src/lib/plan.ts` (step 3 산출물)

## 작업

Supabase Auth 기반 이메일 로그인/회원가입과 세션 유지를 붙인다. 소셜 로그인은 MVP 범위 밖이다.

### 1. `src/middleware.ts`

`@supabase/ssr`의 세션 갱신 미들웨어. `src/lib/supabase/middleware.ts`에 헬퍼(`updateSession(request)`)를 두고 `middleware.ts`는 그것을 호출만 한다.

- `matcher`로 정적 자산(`_next/static`, `_next/image`, `favicon.ico`, 이미지 확장자)을 제외한다.
- 미인증 사용자가 `/dashboard` 이하로 접근하면 `/auth/login?next=<원래경로>`로 리다이렉트한다.
- 인증된 사용자가 `/auth/login` 또는 `/auth/signup`에 접근하면 `/dashboard`로 리다이렉트한다.

`src/lib/supabase/middleware.test.ts`를 먼저 작성하라 (TDD Guard). 리다이렉트 판정 로직을 순수 함수(`shouldRedirect(pathname, isAuthed): string | null`)로 분리해 테스트하면 쉽다.

### 2. 페이지

`page.tsx`는 TDD Guard 면제이므로 테스트 없이 작성 가능하다. 단, 폼 제출 로직은 Client Component로 분리하고 그쪽은 테스트가 필요하다.

- `src/app/auth/login/page.tsx` — 이메일 + 비밀번호. "로그인", "계정이 없으신가요? 회원가입"
- `src/app/auth/signup/page.tsx` — 이메일 + 비밀번호 + 비밀번호 확인. "회원가입"
- `src/app/auth/callback/route.ts` — **`route.ts`는 TDD Guard 면제가 아니다. `route.test.ts`를 먼저 만들어라.** `code`를 세션으로 교환(`exchangeCodeForSession`)하고 `next` 파라미터 또는 `/dashboard`로 리다이렉트한다.
- `src/app/auth/error/page.tsx` — 한국어 에러 안내

**아래 세 화면은 여정에서 길이 끊기는 지점을 막는다. 빠뜨리면 사용자가 되돌아올 방법이 없다.**

- `src/app/auth/verify/page.tsx` (DE-02) — 가입 직후와, 미인증 상태로 로그인을 시도했을 때 도착하는 화면.
  `"[이메일]로 인증 메일을 보냈습니다. 받은 편지함에 없으면 스팸함도 확인해주세요."` + **인증 메일 재발송 버튼**(`resend`).
  미인증을 `"이메일 또는 비밀번호가 올바르지 않습니다."`로 뭉뚱그리지 마라 — 사용자는 비밀번호를 계속 다시 입력하게 된다.
- `src/app/auth/reset/page.tsx` (DE-01) — 이메일 입력 → `resetPasswordForEmail`.
  존재하지 않는 이메일이어도 **같은 문구**를 보여준다(계정 존재 여부가 새면 안 된다):
  `"해당 이메일로 재설정 링크를 보냈습니다."`
- `src/app/auth/reset/confirm/page.tsx` — 메일 링크로 들어온 복구 세션에서 새 비밀번호를 설정한다. 8자 미만이면 제출 전 차단.
  로그인 화면에 `"비밀번호를 잊으셨나요?"` 링크를 반드시 건다. 이 링크가 없으면 위 두 화면은 도달 불가능하다.

### 3. 컴포넌트

- `src/components/auth/AuthForm.tsx` (Client Component) — `AuthForm.test.tsx` 먼저.
  - props: `mode: "login" | "signup"`
  - 제출 시 `createBrowserClient`로 `signInWithPassword` / `signUp` 호출
  - 로딩 중 버튼 비활성화, 에러는 **한국어로** 표시 (Supabase 영문 에러를 그대로 노출하지 마라 — 매핑 테이블을 두고 `"이메일 또는 비밀번호가 올바르지 않습니다."` 같은 문구로 바꾼다)
  - 비밀번호 8자 미만이면 제출 전에 `"비밀번호는 8자 이상이어야 합니다."`
  - **가입에서 계정 존재 여부를 노출하지 마라.** Supabase의 `User already registered`를 그대로 보여주면 이메일 열거가 된다. 재설정 화면과 같은 원칙으로, 성공·중복 모두 `"인증 메일을 보냈습니다. 받은 편지함을 확인해주세요."`로 통일한다.

### 4. 보호 레이아웃

`src/app/(app)/layout.tsx` — 서버에서 세션과 `profiles.plan`을 읽어 헤더에 이메일과 플랜 배지(Free/Pro)를 표시한다. 세션이 없으면 `redirect("/auth/login")`.

`src/app/(app)/dashboard/page.tsx` — 이 step에서는 "아직 업로드한 명세서가 없습니다" 빈 상태만. 실제 대시보드는 step 8.

### 5. 로그아웃

`src/components/auth/SignOutButton.tsx` (Client Component, 테스트 먼저) — `signOut()` 후 `/`로 이동.

### 6. 계정 삭제 (DE-09)

`src/app/api/account/route.ts` — **`route.test.ts` 먼저.**

`DELETE` 요청. **user id는 오직 서버 세션에서만 읽는다.** 요청 바디·쿼리·헤더로 들어온 user id를 절대 쓰지 마라 — 이 라우트는 service role로 `auth.admin.deleteUser()`를 부르므로, 입력을 신뢰하는 순간 **임의 계정 삭제 도구**가 된다.

세션에서 얻은 id로 `admin.ts`의 service role 클라이언트가 `auth.admin.deleteUser(userId)`를 호출한다. 나머지 테이블은 전부 `on delete cascade`이므로 추가 삭제 쿼리를 쓰지 마라.

`src/app/(app)/settings/page.tsx` — 계정 설정. 삭제는 **자기 이메일을 다시 입력해야** 실행되게 한다.
문구: `"계정과 모든 거래 내역이 삭제됩니다. 되돌릴 수 없습니다."`

**이 기능을 생략하지 마라.** ADR-005(거래 원문 전량 보관) + ADR-007(삭제하지 않음)이 합쳐지면 떠난 사용자의 금융 거래 원문이 무기한 남는다. 삭제 경로는 그 부채를 갚는 최소 장치이고, cascade 덕분에 실제 코드는 짧다.

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트:
   - `src/app/auth/callback/route.ts`와 `src/app/api/account/route.ts` 각각에 `route.test.ts`가 존재하는가?
   - 로그인 화면에서 비밀번호 재설정으로 갈 수 있는가? (링크가 없으면 화면을 만들어도 도달 불가)
   - 미인증 상태가 자격 실패와 구분되어 안내되는가?
   - 클라이언트 컴포넌트가 `SUPABASE_SERVICE_ROLE_KEY`를 참조하지 않는가?
   - 사용자에게 보이는 문자열이 전부 한국어인가? (Supabase 원문 에러가 그대로 노출되지 않는가)
   - `src/lib/supabase/`의 기존 클라이언트를 재사용했는가? (새 클라이언트 생성 코드 중복 금지)
3. 결과에 따라 `phases/0-mvp/index.json`의 step 4를 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 생성한 라우트·페이지·컴포넌트 경로 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 외부 자격증명·대시보드 설정이 필요한 검증은 루트 `DEPLOY_CHECKLIST.md`에 `- [ ] (미검증)`으로 append 하고 step은 `"completed"`로 처리한다. 코드·로컬 테스트 자체를 진행할 수 없는 경우에만 `"status": "blocked"` + `"blocked_reason"` 후 중단한다.

## 금지사항

- 소셜 로그인(OAuth) 공급자를 붙이지 마라. 이유: MVP 범위 밖이고 각 공급자 콘솔 설정이 필요해 반드시 blocked 된다.
- Supabase 영문 에러 메시지를 그대로 UI에 노출하지 마라. 이유: 한국어 우선이 CRITICAL 규칙이다.
- 대시보드 실제 콘텐츠를 만들지 마라. 이유: step 8의 범위다. 여기서는 빈 상태까지.
- 세션 검사를 클라이언트 컴포넌트에서만 하지 마라. 이유: 우회 가능하다. 미들웨어 + 서버 레이아웃 양쪽에서 막는다.
- **`DELETE /api/account`에서 요청이 준 user id를 쓰지 마라.** 이유: service role 라우트라 그 순간 임의 계정 삭제 도구가 된다. 세션에서만 읽는다.
- 가입 실패 사유로 계정 존재 여부를 노출하지 마라. 이유: 이메일 열거 취약점이다.
- `profiles`를 클라이언트에서 UPDATE 하는 코드를 쓰지 마라. 이유: step 3에서 정책 자체를 만들지 않았고, 만들면 `plan` 컬럼이 열린다(ADR-012).
- 기존 테스트를 깨뜨리지 마라.
