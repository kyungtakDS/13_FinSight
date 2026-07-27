# 프로젝트: FinSight

카드·은행 CSV 명세서를 업로드하면 지출 구조를 분석해 보여주는 핀테크 SaaS.

## 기술 스택
- Next.js 15 (App Router) / React 19
- TypeScript strict mode
- Tailwind CSS
- Supabase (Auth + Postgres + RLS)
- Polar (`@polar-sh/nextjs`) — 구독 결제
- Anthropic Claude API (`@anthropic-ai/sdk`, 모델 `claude-opus-5`)
- 배포: Vercel
- 테스트: Vitest

## 아키텍처 규칙

- CRITICAL: 외부 API(Anthropic / Supabase service role / Polar) 호출은 **서버 사이드에서만** 한다. 클라이언트 컴포넌트나 브라우저 번들에서 직접 호출하지 말 것.
- CRITICAL: 모든 사용자 데이터 테이블은 **RLS를 활성화하고 `auth.uid()` 기반 정책**을 붙인다. 정책 없는 테이블 생성 금지.
- CRITICAL: **권한·결제 상태를 담는 컬럼(`profiles.plan`, `subscriptions.*`)에는 사용자 쓰기 정책을 만들지 않는다.** RLS는 행 단위라 UPDATE를 허용하면 그 행의 모든 컬럼이 열린다 — 사용자가 자기 `plan`을 `pro`로 바꿀 수 있다는 뜻이다. 이 컬럼들은 웹훅이 service role로만 쓴다.
- CRITICAL: **결제·권한에 영향을 주는 값을 클라이언트 입력에서 받지 않는다.** 대상 사용자 id는 항상 서버 세션에서 읽는다. 요청 바디나 쿼리 파라미터로 들어온 user id를 신뢰하지 말 것.
- CRITICAL: **모든 기간 경계는 `Asia/Seoul` 기준으로 계산한다.** 배포 환경(Vercel)은 UTC로 돈다. 월별 버킷팅·보관 기간 컷오프를 UTC로 계산하면 KST 매월 1일 오전 거래가 전월로 들어가 합계가 틀린다.
- CRITICAL: **금액 집계·기간 추이·반복 결제 탐지는 코드(TS/SQL)로 계산한다. LLM에게 산술을 시키지 말 것.** LLM은 가맹점명 카테고리 분류와 자연어 요약·인사이트에만 쓴다. 이유: LLM 산술은 검증이 불가능하고, 거래 원문 전체를 프롬프트에 넣으면 토큰이 폭증한다.
- CRITICAL: 카드번호·계좌번호·**주민등록번호**로 보이는 문자열은 파싱 단계에서 마스킹한 뒤 저장한다 (`1234-****-****-5678`). 거래 내역 자체는 원본 그대로 저장한다. 카드번호 판정에는 Luhn 검증을 써서 승인번호·일련번호를 오인 마스킹하지 않는다.
- CRITICAL: **계좌이체 거래의 적요에는 실명이 들어간다.** 이런 행은 LLM 분류 입력에서 제외한다 — 사람 이름을 외부 API로 보내지 않기 위해서이고, 어차피 LLM도 사람 이름은 분류하지 못한다.
- 시크릿 환경변수에 `NEXT_PUBLIC_` 접두사를 붙이지 않는다.
- 컴포넌트는 `src/components/`, 타입은 `src/types/`, 순수 로직은 `src/lib/`, 외부 API 래퍼는 `src/services/`에 둔다.
- 서버 상태는 Server Component에서 가져온다. `useEffect` + `fetch`로 초기 데이터를 받지 말 것.

## 언어
- CRITICAL: UI 문자열, 에러 메시지, LLM 출력은 **모두 한국어**다. 코드 식별자·코드 주석·커밋 메시지는 영어를 쓴다.
- 금액은 원화 기준(`₩1,234,567`), 날짜는 `2026-07-27` 형식.
- CSV는 UTF-8과 CP949(EUC-KR)를 모두 받는다. 국내 카드사 명세서는 CP949가 흔하다.

## 개발 프로세스
- CRITICAL: 새 기능 구현 시 반드시 테스트를 먼저 작성하고, 테스트가 통과하는 구현을 작성할 것 (TDD)
- CRITICAL: `.ts`/`.tsx` 파일은 TDD Guard 훅(`scripts/hooks/tdd-guard.sh`)이 막는다. 대응 테스트 파일이 **먼저** 존재해야 쓸 수 있다.
  - 면제: `page.tsx` / `layout.tsx` / `loading.tsx` / `error.tsx` / `not-found.tsx`, `src/types/` 하위, `*.json` / `*.css` / `*.config.*`
  - **면제 아님: `route.ts`.** API 라우트는 같은 디렉토리에 `route.test.ts`를 먼저 만들어야 한다.
- 커밋 메시지는 conventional commits 형식을 따를 것 (feat:, fix:, docs:, refactor:)

## 명령어
npm run dev      # 개발 서버
npm run build    # 프로덕션 빌드
npm run lint     # ESLint
npm run test     # Vitest 1회 실행 (watch 모드 금지 — CI/훅에서 멈춘다)
