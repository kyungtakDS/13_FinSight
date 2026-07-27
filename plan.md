# FinSight MVP 계획

카드·은행 CSV 명세서를 업로드하면 지출 구조를 분석해 보여주는 핀테크 SaaS.
랜딩 → 가입 → 업로드 → 대시보드 → 결제 → 배포까지 한 사이클을 끝낸다.

| | |
|---|---|
| 작성 | 2026-07-27 |
| 브랜치 | `feat-0-mvp` |
| 상태 | 설계 완료, 코드 생성 전 |
| 실행 | `C:\miniconda3\envs\flood_risk311\python.exe scripts/execute.py 0-mvp` |

이 문서는 **전체 지도**다. 세부 사양은 각 파일에 있고, 여기서는 무엇을 왜 그렇게 정했는지와 어디를 보면 되는지를 정리한다.

> **`docs/` 아래 파일은 `scripts/execute.py:184`가 12개 step 프롬프트마다 통째로 주입한다.** 그래서 `docs/`는 짧게 유지하고, 분량이 큰 여정 문서와 이 계획서는 `docs/` 밖에 둔다.

---

## 1. 문서 지도

| 파일 | 역할 | 주입 |
|---|---|---|
| `CLAUDE.md` | 매 step에 강제되는 CRITICAL 규칙 | ✅ |
| `docs/PRD.md` | 목표·사용자·기능 7종·요금제·알려진 한계 | ✅ |
| `docs/ARCHITECTURE.md` | 5테이블 데이터 모델·코드/LLM 분담·데이터 흐름 | ✅ |
| `docs/ADR.md` | 결정 13건과 트레이드오프 | ✅ |
| `docs/UI_GUIDE.md` | 색·차트 규칙·컴포넌트 클래스 | ✅ |
| `phases/0-mvp/USER_JOURNEY.md` | 여정 다이어그램 6개·유스케이스 30건·화면×상태 매트릭스 | ❌ (필요한 step만 참조) |
| `phases/0-mvp/step0~11.md` | 12개 실행 단위 | 해당 step만 |
| `DEPLOY_CHECKLIST.md` | 자격증명이 없어 미룬 수동 검증 항목. 각 step이 append, step 11이 정리 | ❌ |
| `plan.md` | 이 문서 | ❌ |

---

## 2. 무엇을 만드는가

### 핵심 기능 7종

1. **CSV 업로드 & 정규화** — 카드사별 컬럼명·인코딩(UTF-8/CP949) 자동 인식, 카드/계좌/주민번호 마스킹
2. **카테고리별 지출 분류** — 가맹점명을 Claude가 8종으로 분류, 사용자 수정 우선
3. **기간별 지출 추이** — 월별 총지출과 전월 대비 증감
4. **구독 누수 / 이상 거래 탐지** — 규칙·통계 기반
5. **AI 요약 & 절약 인사이트** — 집계 결과를 근거로 한국어 서술
6. **구독 결제 (Polar)** — Free/Pro 구분, 웹훅으로 상태 동기화
7. **계정 수명 관리** — 비밀번호 재설정, 미인증 안내, 계정·명세서 삭제

### 요금제

명세서는 월 1회 나오므로 **횟수 제한은 의미가 없다.** 보관 기간과 기능으로 나눈다.

| | Free | Pro |
|---|---|---|
| 업로드 / 분석 횟수 | 무제한 | 무제한 |
| 데이터 보관 | 최근 3개월 | 무제한 |
| 카테고리별 분류 | ✅ | ✅ |
| 기간별 지출 추이 | 최근 3개월만 | 전 기간 |
| 구독 누수 / 이상 거래 | **건수·총액만** | 전체 목록 |
| AI 요약 & 절약 인사이트 | ❌ | ✅ |
| 데이터 내보내기 (CSV) | ❌ | ✅ |

초과 보관분은 **삭제하지 않고 가린다.** 업그레이드하면 즉시 복구된다.

---

## 3. 기술 스택

| 영역 | 선택 | 버전 고정 이유 |
|---|---|---|
| 프런트/서버 | Next.js 15 App Router + React 19 + TS strict | |
| 스타일 | Tailwind CSS v4 | `@tailwindcss/postcss` |
| 차트 | recharts | **`^3`** — 2.x는 peerDeps에 React 19 없음 |
| 인증·DB | Supabase (Auth + Postgres + RLS) | `@supabase/ssr` |
| 결제 | Polar | `@polar-sh/nextjs` |
| LLM | Claude `claude-opus-5` | `@anthropic-ai/sdk` + **`zod@^4`** — SDK의 zod 헬퍼가 `zod/v4` 서브패스를 import |
| CSV | papaparse + iconv-lite | CP949 대응. Node 런타임 필수 |
| 테스트 | Vitest, **기본 `node` 환경** | Anthropic SDK가 jsdom 미지원. 컴포넌트 테스트만 파일 상단에 `// @vitest-environment jsdom` |
| | `@testing-library/react@^16` | React 19 지원은 v16부터 |
| 배포 | Vercel | fluid compute 기본 300초 |

---

## 4. 데이터 모델 — 5개 테이블

| 테이블 | 역할 |
|---|---|
| `profiles` | `auth.users` 확장, `plan`(free/pro) |
| `subscriptions` | Polar 구독 상태 미러 |
| `statements` | 업로드 1건 = 1행 |
| `transactions` | **개별 거래 원본** + `raw` jsonb |
| `insights` | LLM 생성 요약/제안, 사용자당 1행 |

**이게 전부다.** 반복 결제·이상 거래·카테고리 합계·가맹점 맵은 전부 `transactions`에서 계산되는 파생값이라 저장하지 않는다(ADR-013). 저장하면 원본과 어긋날 수 있는 사본이 생기고, 거래를 지울 때마다 맞춰줘야 한다.

핵심 제약: `unique(user_id, fingerprint)` · `unique(user_id)` on `insights`/`subscriptions` · 인덱스 `(user_id, txn_date desc)` `(user_id, category)` `(user_id, merchant)`

**`profiles`·`subscriptions`에는 사용자 쓰기 정책이 없다.** RLS는 행 단위라 UPDATE를 열면 `plan` 컬럼까지 열리고, 브라우저에서 `update({plan:'pro'})` 한 줄로 결제가 우회된다.

`statements`에는 분류 중복 실행을 막기 위한 `classification_status` · `classification_started_at` · `classified_at`을 둔다. 이는 분석 파생값 저장이 아니라 외부 LLM 작업의 실행 상태다. `classification_started_at`이 있어야 함수가 분류 도중 죽어 `processing`에 갇힌 명세서를 10분 뒤 회수할 수 있다 — 없으면 "다시 분류하기"가 영구히 409를 반환한다.

원자성이 필요한 두 작업은 Postgres 함수로 둔다(`0004_functions.sql`). supabase-js는 여러 문장을 한 트랜잭션으로 묶지 못하므로, 명세서 삭제와 카테고리 일괄 수정은 각각 인사이트 무효화와 함께 한 번에 끝나야 한다. 두 함수 모두 `security invoker`라 RLS가 그대로 적용된다.

---

## 5. 데이터 흐름

```
1단계  POST /api/statements                 (약 2초, 즉시 응답)
  파싱 → 마스킹 → 1000행씩 upsert(지문 중복 스킵)
  → 과거 거래에서 파생한 가맹점 맵으로 아는 것은 즉시 분류 (LLM 없음)
  → 사용자는 여기서 이미 거래 목록을 본다

2단계  POST /api/statements/[id]/classify   (10~30초, 실패해도 1단계는 남음)
  → 맵에 없는 가맹점만 LLM ①  (이체 행 제외 — 적요에 실명이 들어간다)
  → transactions.category UPDATE
  → pro면 LLM ② → insights UPSERT

대시보드                                     ← LLM 호출 0건
  → 거래 1회 조회 → applyRetention으로 표시분 분리
  → 집계·탐지는 TS 순수 함수로 계산 (탐지는 전체 이력 기준)
  → insights는 읽기만
```

### 코드 vs LLM

| 분석 | 담당 | 이유 |
|---|---|---|
| 기간별 추이 · 카테고리 합계 | **코드** | 산술은 검증 가능해야 한다 |
| 가맹점 → 카테고리 | **LLM** | `(주)우아한형제들` → 배달. 규칙으론 못 잡는다 |
| 반복 결제 · 이상 거래 | **코드** | 규칙·통계가 더 정확하고 공짜 |
| 요약 · 절약 제안 | **LLM** | 집계 결과를 *입력으로 받아* 서술 |

LLM 호출은 명세서당 **최대 2번**이고, 둘 다 쓰기 경로에서만 일어난다. 동일 명세서의 동시·반복 classify는 상태 전이로 한 번만 실행되게 한다.

---

## 6. 결정 13건 (ADR)

| # | 결정 |
|---|---|
| 001 | Supabase — 격리를 앱이 아니라 DB(RLS)가 보장 |
| 002 | Polar — MoR, Next.js 어댑터. 국내 간편결제는 미지원 |
| 003 | **산술은 코드가, 언어는 LLM이** |
| 004 | structured outputs로 스키마 강제 |
| 005 | 거래 원본 전량 보관 (누적돼야 추이·구독 탐지가 성립) |
| 006 | 요금제는 횟수 아닌 보관기간+기능 |
| 007 | 초과분은 가리고 지우지 않음 (+계정 삭제 경로 필수) |
| 008 | **업로드를 저장/분류 2단계로** — 타임아웃이 아니라 실패 복구 때문 |
| 009 | 가맹점 맵은 **테이블이 아니라 쿼리** |
| 010 | 게이팅은 기능이 아니라 **목록 단위** (+필터는 표시에만) |
| 011 | **LLM은 쓰기 경로에서만.** 페이지 렌더에서 부르지 않음 |
| 012 | 권한 상승 가능한 값은 사용자 쓰기 경로에서 분리 |
| 013 | **파생값은 저장하지 않고 조회 시점에 계산** |

---

## 7. 실행 단위 12개

| # | name | 산출물 | 주의 |
|---|---|---|---|
| 0 | `project-setup` | package.json → 의존성 → 설정 → `lib/format.ts` | ⚠ **package.json이 최우선.** Stop 훅이 매 세션 `lint && build && test`를 돌린다. `create-next-app` 금지 |
| 1 | `core-types` | `src/types/` 전체 | types/는 TDD 면제 — 로직 금지 |
| 2 | `csv-parser` | `parseStatementCsv`, `normalizeMerchant`, 마스킹, 지문 | 지문에 `occurrence` 필수(정상 중복 보존), 카드번호는 Luhn 통과분만 마스킹, 주민번호 패턴 포함 |
| 3 | `supabase-schema` | 마이그레이션 4종 + 클라이언트 3종 + `plan.ts` | AC: RLS 5테이블 grep, 분류 상태 필드 |
| 4 | `auth-flow` | 로그인·가입·콜백·미인증 안내·비밀번호 재설정·계정 삭제 | 계정 삭제는 세션 id만 사용 |
| 5 | `analytics-engine` | 순수 함수 6종 | KST 버킷팅, MAD z>3.5, 탐지는 전체 이력 |
| 6 | `anthropic-service` | `classifyMerchants` / `generateInsights` | `max_tokens: 16000`, `temperature` 금지 |
| 7 | `statements-api` | `ingest.ts` + `classify.ts` + 라우트 5개 | 1단계에 LLM 금지, 1000행 청크, 단기 남용 방지, 인사이트 무효화, 내보내기 CSV 수식 이스케이프 |
| 8 | `dashboard-ui` | Server Component + 컴포넌트 15종 | LLM 호출 0건, 상태 매트릭스 전 칸 처리 |
| 9 | `billing` | Polar 라우트 4개 + `syncSubscription` | checkout 세션 검증, 웹훅 순서 가드, `past_due`는 기간 만료까지 유예 |
| 10 | `landing-page` | 랜딩 + 요금제 + `/help/csv` | 안티패턴 금지, 한계 명시 |
| 11 | `deploy-prep` | README · DEPLOY_CHECKLIST · SECURITY · 샘플 CSV | 실제 배포 시도 금지 |

각 step 파일은 자기완결적이다 — 읽어야 할 파일, 시그니처 수준 지시, 실행 가능한 AC 커맨드, "X를 하지 마라 / 이유: Y" 형식의 금지사항을 갖는다.

---

## 8. 개발 제약 (훅)

- **Stop 훅** — 매 세션 종료 시 `npm run lint && npm run build && npm run test`. `package.json`이 없으면 무한 재시도에 빠진다. `test`는 반드시 `vitest run`(watch 금지).
- **TDD Guard 훅** — `.ts`/`.tsx`는 대응 테스트가 **먼저** 있어야 쓸 수 있다.
  - 면제: `page.tsx`/`layout.tsx`/`loading.tsx`/`error.tsx`/`not-found.tsx`, `src/types/`, `*.config.*`
  - **면제 아님: `route.ts`** — 모든 API 라우트는 `route.test.ts`가 먼저 필요하다.
- **Bash 훅** — `rm -rf`, `git push --force`, `git reset --hard`, `DROP TABLE` 차단.

---

## 9. 실행 절차

```bash
C:\miniconda3\envs\flood_risk311\python.exe scripts/execute.py 0-mvp
```

harness 문서의 `python3`는 이 머신에서 동작하지 않는다(PATH에 python 없음).

step을 순차 실행하며 step 단위로 커밋하고, 실패 시 최대 3회 자가 교정한다.

**모든 step은 키 없이 코드·모킹 테스트까지 완주한다.** 환경변수는 함수 내부에서 지연 검증하고 외부 서비스 호출은 모킹한다. 실제 Supabase/Anthropic/Polar 연결과 배포 확인은 step 11의 수동 배포 체크리스트에 `미검증`으로 남긴다. 외부 자격증명 부재만으로 step을 `blocked` 처리하지 않는다.

`blocked`는 코드·로컬 테스트 자체를 계속할 수 없는 사용자 결정이나 로컬 의존성 부재에만 사용한다. 수동 통합 검증은 실행 흐름을 중단시키지 않는다.

| 환경변수 | 출처 |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` / `SUPABASE_SERVICE_ROLE_KEY` | Supabase 프로젝트 |
| `ANTHROPIC_API_KEY` | console.anthropic.com |
| `POLAR_ACCESS_TOKEN` / `POLAR_WEBHOOK_SECRET` / `POLAR_PRODUCT_ID_PRO` / `POLAR_SERVER` | Polar (sandbox 먼저) |
| `NEXT_PUBLIC_SITE_URL` | 배포 도메인 |

---

## 10. 배포 전 반드시 확인

`phases/0-mvp/step11.md`에 전체 체크리스트가 있다. 그중 **하나라도 실패하면 배포하면 안 되는 것**:

- [ ] 브라우저 콘솔에서 `supabase.from('profiles').update({plan:'pro'})`가 **실패**하는가
- [ ] `/api/checkout?customerExternalId=<남의 uuid>`가 403인가
- [ ] 다른 계정으로 로그인해 앞 계정의 거래가 보이지 않는가 (RLS)
- [ ] 업로드한 테스트용 주민번호·카드번호가 DB의 `raw`에서도 마스킹됐는가
- [ ] 이체 행이 섞인 파일 업로드 후 LLM 요청에 사람 이름이 없는가
- [ ] 내보낸 CSV에서 `=`로 시작하는 가맹점명이 `'=`로 이스케이프됐는가 (엑셀로 열어 확인)
- [ ] KST 매월 1일 오전 거래가 그 달 합계에 잡히는가

---

## 11. 범위 밖 (알고 넘어간 것)

**기능**
은행 API 실시간 연동 · 예산 알림 · 팀/가족 공유 · 다중 통화 · 모바일 앱 · 영수증 OCR · 카드별·용도별(사업/개인) 구분 · 수동 컬럼 매핑 UI · 이상 거래 무시 처리

**파싱 한계** (문서로 공지)
할부 이중 계상 가능성 · 부분취소 자동 상계 안 함 · 여러 파일이 한 대시보드에 합쳐짐

**운영**
장기 미접속 데이터 자동 정리

가장 먼저 후회할 항목은 **이상 거래 무시 처리**다. MAD 임계값(3.5)이 실제 데이터에서 얼마나 잡는지 보고 판단한다.

---

## 12. 이 계획이 거쳐온 검토

| 회차 | 관점 | 주요 변경 |
|---|---|---|
| 01 | 초안 | 가드레일 문서 5종 + step 12개 |
| 02 | **사용자 여정** | 막다른 길 13건 발견 → 2단계 업로드, 가맹점 맵, 목록 단위 게이팅, `/help/csv`, 복구 경로 6종 |
| 03 | **아키텍처** | 실행 차단 3건(zod v3 / jsdom / React 19 peer), 설계 모순 3건, 스키마 누락 4건 |
| 04 | **보안·에러** | 결제 우회(RLS), 계정 오염(checkout 파라미터), UTC 월경계, 5만행 INSERT 등 15건 |
| 05 | **MVP 적정성** | 테이블 2개·이상탐지 2종·주별 집계·레이트리밋·캐시키 제거. 7→5 테이블 |

5회차에서 잘라낸 것은 대부분 2~4회차에서 더한 것이다. 검토를 거듭할수록 커지는 경향을 마지막에 한 번 되돌렸다.
