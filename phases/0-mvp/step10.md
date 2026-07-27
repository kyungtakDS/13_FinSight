# Step 10: landing-page

## 읽어야 할 파일

- `/docs/PRD.md` — 목표·사용자·핵심 기능·요금제 표
- `/docs/UI_GUIDE.md` — **안티패턴 표를 특히 주의하라. 랜딩 페이지가 AI 슬롭이 되기 가장 쉽다.**
- `src/app/page.tsx` (step 0의 플레이스홀더 — 이것을 교체한다)
- `src/components/UpgradeButton.tsx` (step 9)
- `src/lib/format.ts` (step 0)

## 작업

`src/app/(marketing)/` 아래에 랜딩과 요금제를 만든다. 비인증 사용자가 보는 화면이다.

### 1. `src/app/(marketing)/layout.tsx`

간단한 헤더(로고 · 요금제 · 로그인 · 시작하기)와 푸터. 헤더는 대시보드 헤더와 다른 컴포넌트다.

### 2. `src/app/page.tsx` (랜딩)

섹션 구성:

**히어로** — 한 문장으로 무엇인지 말한다. 예: `"카드 명세서 CSV를 올리면, 어디에 얼마를 쓰는지 5초 안에 보여드립니다."`
CTA 두 개: `"무료로 시작하기"`(→ `/auth/signup`), `"요금제 보기"`(→ `/pricing`).
- 배경 gradient orb, 그라데이션 텍스트, glass morphism을 쓰지 마라.
- 스크린샷이나 정적 목업 대신 **실제 컴포넌트로 만든 샘플 대시보드**를 보여준다. 데이터는 `src/lib/sample-data.ts`의 가짜 거래에서 step 5의 실제 집계 함수를 통과시켜 만든다 (`sample-data.test.ts` 먼저). 이렇게 하면 제품이 바뀌면 랜딩도 같이 바뀐다.
- 샘플임을 작게 명시한다: `"예시 데이터입니다."`

**기능 4종** — PRD 핵심 기능 2~5번. 각각 한 줄 설명 + 해당 컴포넌트의 축소판. 아이콘 나열 카드 4개로 때우지 마라.

**어떻게 동작하나** — 3단계: CSV 업로드 → 자동 분류 → 분석 결과. 각 단계 한 문장.

**개인정보** — 짧고 정직하게. `"거래 내역은 계정별로 격리되어 저장되며, 카드·계좌번호는 저장 전에 마스킹됩니다. 분석을 위해 가맹점명이 Anthropic Claude API로 전송됩니다."`
**이 문단을 과장하지 마라.** 실제로 하는 일만 쓴다 — LLM에 가맹점명이 나가는 것은 사실이므로 숨기지 않는다.

**마지막 CTA**

### 3. `src/app/(marketing)/pricing/page.tsx`

`docs/PRD.md`의 요금제 표를 그대로 렌더한다. 표의 값을 마음대로 바꾸지 마라.
- Free / Pro 2열
- **"분석 횟수 무제한"을 양쪽 모두에 명시한다.** 이게 이 제품의 차별점이다.
- Pro 가격은 `NEXT_PUBLIC_PRO_PRICE_KRW` 환경변수로 읽고, 없으면 `"가격 준비 중"`을 표시한다. 하드코딩하지 마라.
- 로그인 상태면 `UpgradeButton`, 아니면 `/auth/signup`으로 보낸다.

### 4. 메타데이터

`layout.tsx`의 `metadata`: 한국어 `title`/`description`, `openGraph` 기본값. `lang="ko"` 확인.

### 테스트

`page.tsx`는 TDD 면제지만 아래는 테스트가 필요하다:
- `src/lib/sample-data.ts` — 샘플 거래 생성기. 집계 함수를 통과했을 때 카테고리가 8종 모두 나오고 최소 3개월치가 있는지 단언.
- `src/components/marketing/PricingTable.tsx` — 요금제 표. Free/Pro 항목 수가 같은지, 가격 환경변수가 없을 때 `"가격 준비 중"`이 나오는지.

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트:
   - `UI_GUIDE.md` 안티패턴 표의 7항목 중 하나라도 쓰지 않았는가? (grep으로 `backdrop-blur`, `bg-gradient-to`, `blur-3xl`을 확인하라)
   - 요금제 표 내용이 `docs/PRD.md`와 일치하는가?
   - 샘플 대시보드가 목업 이미지가 아니라 실제 컴포넌트 + `src/lib/analytics/`를 쓰는가?
   - Pro 가격이 하드코딩되어 있지 않은가?
   - 모든 문구가 한국어인가?
3. 결과에 따라 `phases/0-mvp/index.json`의 step 10을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 생성한 페이지·컴포넌트 경로 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 사용자 개입 필요 → `"status": "blocked"` + `"blocked_reason"` 후 즉시 중단

## 금지사항

- `UI_GUIDE.md` 안티패턴 표의 항목을 쓰지 마라. 특히 배경 gradient orb, 그라데이션 텍스트, glass morphism — 랜딩에서 가장 흔한 실수다.
- 가짜 고객 후기, 가짜 로고 월("이런 곳에서 씁니다"), 가짜 사용자 수를 만들지 마라. 이유: 사실이 아니고, 금융 서비스에서 이런 것이 발각되면 신뢰가 끝난다.
- 개인정보 문단에서 실제 동작을 숨기지 마라. 이유: 가맹점명이 외부 API로 나가는 것은 사실이므로 명시해야 한다.
- Pro 가격을 코드에 하드코딩하지 마라. 이유: Polar 상품 가격과 어긋나면 그대로 클레임이 된다.
- 대시보드 컴포넌트를 랜딩용으로 복제하지 마라. 이유: 두 벌이 되면 곧 갈라진다. 같은 컴포넌트에 샘플 데이터를 넣어라.
- 기존 테스트를 깨뜨리지 마라.
