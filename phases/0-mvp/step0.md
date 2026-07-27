# Step 0: project-setup

## 읽어야 할 파일

- `/CLAUDE.md`
- `/docs/ARCHITECTURE.md`
- `/.claude/settings.json` — 이 프로젝트에 걸린 훅을 반드시 확인하라
- `/scripts/hooks/tdd-guard.sh`

## ⚠ 가장 먼저 할 일

`.claude/settings.json`의 **Stop 훅이 매 세션 종료 시 `npm run lint && npm run build && npm run test`를 실행한다.** `package.json`이 없거나 세 스크립트 중 하나라도 없으면 이 step은 종료되지 못하고 무한 재시도에 빠진다.

따라서 **가장 먼저 `package.json`을 세 스크립트가 모두 정의된 상태로 만들고 `npm install`을 끝내라.** 나머지 작업은 그 다음이다.

`test` 스크립트는 반드시 **1회 실행 후 종료**해야 한다 (`vitest run`). watch 모드(`vitest`)를 쓰면 훅이 영원히 멈춘다.

## 작업

이 디렉토리에는 이미 `CLAUDE.md`, `docs/`, `scripts/`, `phases/`, `.git`이 있다. **`create-next-app`을 쓰지 마라** — 기존 파일과 충돌하고 대화형 프롬프트에서 멈춘다. 아래 파일들을 직접 작성하고 `npm install`로 의존성을 설치하라.

### 1. `package.json`

```jsonc
{
  "name": "finsight",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint .",
    "test": "vitest run"
  }
}
```

의존성:

- runtime: `next@^15`, `react@^19`, `react-dom@^19`, `@supabase/supabase-js`, `@supabase/ssr`, `@anthropic-ai/sdk`, `@polar-sh/nextjs`, **`zod@^4`**, `papaparse`, `iconv-lite`, **`recharts@^3`**
- dev: `typescript`, `@types/node`, `@types/react`, `@types/react-dom`, `@types/papaparse`, `eslint`, `eslint-config-next`, `@eslint/eslintrc`, `tailwindcss@^4`, `@tailwindcss/postcss`, `postcss`, `vitest`, `@vitejs/plugin-react`, `jsdom`, **`@testing-library/react@^16`**, `@testing-library/jest-dom`

**버전을 낮추지 마라. 셋 다 이유가 있다:**
- `zod@^4` — `@anthropic-ai/sdk/helpers/zod`의 타입 정의가 `import * as z from 'zod/v4'`로 시작한다. zod 3.x에는 그 서브패스가 없어 step 6의 구조화 출력이 컴파일되지 않는다.
- `recharts@^3` — 2.x는 peerDependencies에 React 19가 없다. 3.x부터 `^19.0.0`이 들어간다.
- `@testing-library/react@^16` — React 19 지원은 v16부터다.

### 2. 설정 파일

- `tsconfig.json` — `"strict": true`, `"paths": { "@/*": ["./src/*"] }`, Next.js 플러그인
- `next.config.ts`
- `eslint.config.mjs` — `eslint-config-next` 기반 flat config
- `postcss.config.mjs` — `@tailwindcss/postcss`
- `vitest.config.ts` — **`environment: "node"` (기본값)**, `globals: true`, `setupFiles`, `@/*` alias 를 `tsconfig`와 동일하게 매핑. **`include` 패턴이 `src/**/*.test.{ts,tsx}`를 잡도록 하라.**
- `vitest.setup.ts` — `@testing-library/jest-dom` import

**기본 환경을 `jsdom`으로 잡지 마라.** Anthropic TypeScript SDK 공식 문서가 **jsdom 환경을 지원하지 않는다**고 명시한다. 전역을 jsdom으로 두면 step 6의 `anthropic.test.ts`가 미지원 환경에서 돌아 원인을 찾기 어려운 실패가 난다.

대신 **컴포넌트 테스트 파일 최상단에만** 다음 한 줄을 붙인다:

```ts
// @vitest-environment jsdom
```

이 docblock 방식은 vitest 버전에 무관하게 동작한다(`environmentMatchGlobs`는 최신 vitest에서 deprecated). `src/lib/`·`src/services/`·`route.test.ts`는 전부 node 환경이 맞다 — DOM이 필요 없고, SDK와 Node 내장 `crypto`를 쓴다.

### 3. 최소 앱 셸

- `src/app/layout.tsx` — `<html lang="ko">`, `metadata`의 `title`/`description`은 한국어
- `src/app/page.tsx` — 임시 플레이스홀더 (step 10에서 랜딩으로 교체)
- `src/app/globals.css` — `@import "tailwindcss";` + `docs/UI_GUIDE.md`의 배경/잉크 색을 CSS 커스텀 프로퍼티로 선언. 다크 고정이므로 `prefers-color-scheme` 분기를 만들지 마라.

### 4. 스모크 테스트

`src/lib/format.test.ts`를 **먼저** 작성한 뒤 `src/lib/format.ts`를 구현하라 (TDD Guard 훅이 역순을 막는다).

```ts
export function formatKRW(amount: number): string    // 1234567 → "₩1,234,567"
export function formatDate(d: Date | string): string // → "2026-07-27"

/** KST 기준 날짜 문자열. 배포 환경이 UTC라 이 변환 없이는 월 경계가 밀린다. */
export function toKstDateString(d: Date): string      // → "2026-07-27"
/** KST 기준 월 버킷 */
export function toKstMonthKey(d: Date): string        // → "2026-07"
```

`formatKRW`는 음수(`-₩1,234`)와 0(`₩0`)을 처리해야 한다.

**KST 변환을 직접 `+9시간` 산술로 구현하지 마라.** `Intl.DateTimeFormat("sv-SE", { timeZone: "Asia/Seoul" })`를 쓰면 `YYYY-MM-DD`가 바로 나온다. 테스트에는 **경계 케이스를 반드시 넣어라** — `2026-08-01T00:30:00+09:00`(= UTC로는 7월 31일 15:30)이 `"2026-08-01"`과 `"2026-08"`로 나와야 한다. 이 한 줄이 월별 합계 전체의 정확성을 좌우한다.

### 5. `.env.example`

값 없이 키만 나열한다. 실제 값을 채우지 마라.

```
NEXT_PUBLIC_SITE_URL=
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
ANTHROPIC_API_KEY=
POLAR_ACCESS_TOKEN=
POLAR_WEBHOOK_SECRET=
POLAR_PRODUCT_ID_PRO=
POLAR_SERVER=sandbox
```

### 6. `.gitignore` 보강

기존 내용을 지우지 말고 아래가 없으면 추가하라: `node_modules/`, `.next/`, `.env*.local`, `coverage/`, `next-env.d.ts`

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
```

세 커맨드가 모두 성공해야 한다.

## 검증 절차

1. 위 AC 커맨드를 순서대로 실행한다.
2. 아키텍처 체크리스트:
   - `src/` 하위 구조가 `docs/ARCHITECTURE.md`의 디렉토리 구조를 따르는가?
   - `tsconfig.json`에 `strict: true`가 있는가?
   - `package.json`의 `test`가 `vitest run`인가? (watch 모드가 아닌가)
   - `vitest.config.ts`의 기본 환경이 `node`인가? (jsdom 전역 금지)
   - `zod`가 4.x, `recharts`가 3.x, `@testing-library/react`가 16.x로 설치됐는가? (`npm ls zod recharts @testing-library/react`)
3. 결과에 따라 `phases/0-mvp/index.json`의 step 0을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 생성한 설정 파일과 `src/lib/format.ts` 시그니처를 한 줄 요약
   - 3회 시도 후 실패 → `"status": "error"`, `"error_message"`에 구체적 에러
   - 외부 자격증명·대시보드 설정이 필요한 검증은 루트 `DEPLOY_CHECKLIST.md`에 `- [ ] (미검증)`으로 append 하고 step은 `"completed"`로 처리한다. 코드·로컬 테스트 자체를 진행할 수 없는 경우에만 `"status": "blocked"` + `"blocked_reason"` 후 중단한다.

## 금지사항

- `create-next-app`을 실행하지 마라. 이유: 기존 파일(`CLAUDE.md`, `docs/`)과 충돌하고 대화형 프롬프트에서 멈춘다.
- `test` 스크립트를 watch 모드로 두지 마라. 이유: Stop 훅이 종료되지 않아 step이 무한 대기한다.
- 실제 API 키를 `.env.example`이나 코드에 넣지 마라. 이유: 커밋에 시크릿이 남는다.
- Supabase / Anthropic / Polar 연동 코드를 이 step에서 쓰지 마라. 이유: 각각 step 3, 6, 9의 범위다. 여기서는 의존성 설치까지만 한다.
- 라이트 모드 스타일을 만들지 마라. 이유: 다크 고정이 설계 결정이다.
- vitest 기본 환경을 `jsdom`으로 두지 마라. 이유: Anthropic SDK가 jsdom을 지원하지 않는다고 공식 문서에 명시돼 있다. step 6에서 원인 파악이 어려운 실패로 돌아온다.
- `zod`를 3.x로 설치하지 마라. 이유: SDK의 zod 헬퍼가 `zod/v4` 서브패스를 import 한다.
