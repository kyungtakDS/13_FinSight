# Step 11: deploy-prep

## 읽어야 할 파일

- `/CLAUDE.md`
- `/docs/ARCHITECTURE.md`, `/docs/ADR.md`, `/docs/PRD.md`
- `.env.example` (step 0)
- `supabase/migrations/` 전체 (step 3)
- `src/app/api/webhook/polar/route.ts` (step 9)
- `phases/0-mvp/index.json` — 앞 step들의 `summary`를 읽어 실제 산출물을 확인하라

## 작업

배포 가능한 상태로 마무리하고, 사람이 따라 할 수 있는 문서를 남긴다. **이 step에서 실제 배포를 시도하지 마라** — 자격증명이 없어 반드시 실패한다. 문서와 설정 파일까지가 범위다.

### 1. `README.md`

한국어로 작성한다. 포함할 것:

**개요** — 무엇을 하는 서비스인지 3문장.

**로컬 실행**
```bash
npm install
cp .env.example .env.local   # 값 채우기
npm run dev
```

**환경변수 표** — 각 변수의 이름 / 어디서 얻는지 / 서버 전용인지 / 없으면 어떤 기능이 죽는지. `.env.example`과 항목이 정확히 일치해야 한다.

| 변수 | 출처 | 노출 | 없으면 |
|---|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase 프로젝트 설정 | 공개 | 로그인 불가 |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase 프로젝트 설정 | **서버 전용** | 결제 웹훅 반영 불가 |
| `ANTHROPIC_API_KEY` | console.anthropic.com | **서버 전용** | 카테고리 분류·인사이트 불가 |
| … | | | |

**Supabase 설정 절차**
1. 프로젝트 생성
2. `supabase/migrations/` 의 SQL을 순서대로 실행 (`0001` → `0002` → `0003` → `0004`). SQL Editor 붙여넣기 또는 `supabase db push`.
3. Authentication → Email 공급자 활성화, `Site URL`과 `Redirect URLs`에 `/auth/callback` 등록
4. **RLS 확인:** Table Editor에서 5개 테이블 모두 RLS가 켜져 있는지 눈으로 확인. 꺼져 있으면 배포하지 마라.
5. Authentication → Email Templates에서 **비밀번호 재설정 메일**이 활성화돼 있는지 확인. `Redirect URLs`에 `/auth/reset/confirm`도 등록한다.

**Polar 설정 절차**
1. sandbox 조직 생성 → `POLAR_SERVER=sandbox`로 먼저 검증
2. Pro 상품 생성 → 상품 ID를 `POLAR_PRODUCT_ID_PRO`에
3. Organization Access Token 발급 → `POLAR_ACCESS_TOKEN`
4. 웹훅 엔드포인트 등록: `https://<도메인>/api/webhook/polar`, 시크릿을 `POLAR_WEBHOOK_SECRET`에
5. 구독하는 이벤트: `subscription.active`, `subscription.updated`, `subscription.canceled`, `subscription.uncanceled`, `subscription.revoked`
6. 검증 후 production 조직으로 같은 절차 반복

**Vercel 배포 절차**
1. 레포 연결, 프레임워크 자동 감지
2. 환경변수 전부 등록 (`NEXT_PUBLIC_SITE_URL`은 배포 도메인)
3. 배포 후 Supabase Redirect URLs와 Polar 웹훅 URL을 실제 도메인으로 갱신
4. **웹훅은 로컬에서 받을 수 없다.** 로컬 테스트는 `polar` CLI 또는 터널링 도구로 포워딩한다.

**엔드투엔드 확인 체크리스트** — 배포 후 사람이 직접 밟을 순서.
정상 경로뿐 아니라 **복구 경로**를 반드시 밟는다. 여기서 막히면 사용자는 되돌아올 방법이 없다(`phases/0-mvp/USER_JOURNEY.md` 2장).

정상 경로
- [ ] 회원가입 → 이메일 인증 → 로그인
- [ ] 샘플 CSV 업로드 → **1단계 결과가 먼저 보이고** 그 다음 분류가 채워지는지
- [ ] 같은 파일 재업로드 → 추가 0건 · 중복 N건이 **성공으로** 표시되는지
- [ ] 카테고리 수정 → **같은 가맹점 전체에 반영**되고 새로고침 후 유지되는지
- [ ] 두 번째 명세서 업로드 → **이미 아는 가맹점은 LLM 없이 즉시 분류**되는지
- [ ] Free 상태에서 구독·이상 거래의 **건수는 보이고 목록만 잠겨** 있는지
- [ ] Polar sandbox 결제 → `profiles.plan`이 `pro`로 바뀌는지
- [ ] Pro 상태에서 구독 목록·이상 거래·인사이트가 보이는지
- [ ] 구독 해지 → 기간 만료 전까지 Pro 유지되는지

보안 (배포 전 필수)
- [ ] **브라우저 콘솔에서 `supabase.from('profiles').update({plan:'pro'})`가 실패하는지** — 성공하면 결제가 무의미하다
- [ ] **`/api/checkout?customerExternalId=<남의 uuid>`가 403인지**
- [ ] 다른 계정의 `statementId`로 classify·삭제·내보내기를 호출하면 거부되는지
- [ ] 내보낸 CSV에서 `=`로 시작하는 가맹점명이 `'=`로 이스케이프됐는지 (엑셀로 열어 확인)
- [ ] 업로드한 CSV에 넣은 테스트용 주민번호·카드번호가 DB의 `raw`에서도 마스킹됐는지
- [ ] 이체 행이 섞인 파일을 올린 뒤, LLM 요청 로그에 사람 이름이 없는지

복구 경로
- [ ] 비밀번호 재설정 메일 → 링크 → 새 비밀번호 → 로그인
- [ ] 잘못 올린 명세서 삭제 → 거래가 함께 사라지고 인사이트가 무효화되는지
- [ ] KST 매월 1일 오전 거래가 들어간 파일 → 월 합계가 그 달로 잡히는지
- [ ] 미인증 상태로 로그인 시도 → **자격 실패가 아니라** 인증 안내 화면이 뜨는지
- [ ] 컬럼명이 다른 CSV 업로드 → 에러에 **감지된 헤더가 실려 있고** 도움말 링크가 있는지
- [ ] `ANTHROPIC_API_KEY`를 일부러 비우고 업로드 → **거래는 저장되고** 미분류 배너 + 다시 분류하기가 뜨는지
- [ ] 결제 직후 대시보드 → 확인 대기 문구가 나온 뒤 Pro로 바뀌는지
- [ ] 계정 삭제 → 재로그인 불가, 다른 테이블 행도 함께 사라졌는지

격리
- [ ] **다른 계정으로 로그인해 앞 계정의 거래가 보이지 않는지** (RLS 검증)
- [ ] 다른 계정의 `statementId`로 `/api/statements/[id]/classify`를 호출하면 거부되는지

**한계와 다음 단계** — `docs/PRD.md`의 MVP 제외 사항을 옮기고, 아래 파싱 한계를 명시한다:

- **할부 거래** — 카드사에 따라 총액 행과 회차 행이 함께 나오는 경우가 있어 이중 계상될 수 있다. 현재는 보이는 대로 집계한다.
- **부분취소·정정 거래** — 원거래 참조가 필요해 자동 상계하지 않는다. 취소 행은 음수로 잡히므로 총액은 맞지만, 개별 거래 목록에는 두 행이 모두 남는다.
- **카드별 구분** — 여러 파일을 올리면 한 대시보드에 합쳐진다.

같은 내용을 `/help/csv`에도 적는다. 모르고 빠진 것과 알고 넘어간 것은 다르고, 사용자가 숫자를 검증하다 이 차이를 먼저 발견하면 신뢰를 잃는다.

### 2. `DEPLOY_CHECKLIST.md` (레포 루트)

앞선 step들이 자격증명이 없어 미룬 수동 검증 항목이 `- [ ] (미검증)` 형태로 쌓여 있다. **이 파일을 정리해 완성한다** — 없으면 새로 만든다.

- 위 "엔드투엔드 확인 체크리스트"를 이 파일로 옮기고, README에서는 이 파일을 링크한다
- 앞선 step이 남긴 미검증 항목을 Supabase / Polar / Vercel 섹션으로 분류한다
- **`docs/` 아래에 두지 마라.** `execute.py:184`가 `docs/*.md`를 12개 step 프롬프트마다 주입하므로, 항목이 쌓이는 문서를 거기 두면 모든 step의 토큰이 는다.

### 3. `docs/SECURITY.md`

짧게. 무엇을 저장하고 무엇을 저장하지 않는지, 어디로 데이터가 나가는지.
- 저장: 거래 원본 행, 마스킹된 적요, 카테고리, 집계 결과, 구독 상태
- 저장 안 함: 카드 전체번호, 계좌 전체번호, **주민등록번호**, 원본 CSV 파일
- 외부 전송: 가맹점명(유니크 목록)과 집계 결과가 Anthropic Claude API로 전송됨. 개별 거래 원문은 전송하지 않음. 이미 분류된 가맹점은 다시 전송하지 않음. **계좌이체 거래는 적요에 실명이 들어가므로 전송 대상에서 제외됨.**
- 격리: 모든 테이블 RLS. **플랜·구독 컬럼은 사용자 쓰기 불가** — 웹훅이 service role로만 갱신
- 삭제: 설정 화면의 계정 삭제로 사용자가 직접 전 데이터를 지울 수 있음. 명세서 단위 삭제도 가능
- 남용 방지: 사용자당 시간당 업로드 20건 제한, 동일 명세서 중복 분류 차단(ADR-006 운영 가드레일). LLM 비용의 최종 상한은 Anthropic 콘솔의 지출 한도로 별도 설정한다

랜딩의 개인정보 문단도 이 범위와 일치시켜라. "가맹점명이 전송됩니다"만 쓰면 사용자는 그것을 "내가 송금한 사람 이름"으로 읽지 않는다 — 이체 행을 제외한다는 사실을 함께 적는다.

### 4. `sample/` 샘플 CSV

테스트용 CSV 2개를 넣는다. `sample/card-utf8.csv`(UTF-8), `sample/card-cp949.csv`(CP949). 각 30행 내외, 가맹점명은 실존하지 않는 가상의 이름으로. 실제 개인 데이터를 넣지 마라.

### 5. 최종 점검

- `.env.example`과 README 환경변수 표의 항목이 일치하는지
- `docs/`와 `CLAUDE.md`에 `{...}` 플레이스홀더가 남아 있지 않은지
- `git grep -nE "(sk-ant-|service_role|POLAR_ACCESS_TOKEN=)[A-Za-z0-9]"` 로 커밋된 시크릿이 없는지

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
test -f README.md && test -f docs/SECURITY.md && test -f DEPLOY_CHECKLIST.md
grep -q "SUPABASE_SERVICE_ROLE_KEY" README.md
! grep -rn "{프로젝트명}\|{기능 1}\|{결정 사항}" CLAUDE.md docs/
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트:
   - README의 환경변수 표가 `.env.example`과 정확히 일치하는가?
   - RLS 확인 단계가 배포 절차에 들어 있는가?
   - 샘플 CSV에 실제 개인정보가 없는가?
   - 커밋된 시크릿이 없는가?
3. 결과에 따라 `phases/0-mvp/index.json`의 step 11을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 README·DEPLOY_CHECKLIST·SECURITY·sample 경로 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 실제 배포·대시보드 설정처럼 사용자 승인이 필요한 항목은 루트 `DEPLOY_CHECKLIST.md`에 `미검증`으로 남긴다. 문서·코드·로컬 검증이 완료됐으면 step은 `"completed"`로 처리하며, 외부 검증 미실행만으로 `"blocked"` 처리하지 않는다.

## 금지사항

- 실제 배포를 시도하지 마라 (`vercel deploy`, `supabase db push` 등). 이유: 자격증명이 없어 실패하고, 있더라도 배포는 사람이 승인할 일이다.
- 실제 키를 README나 `.env.example`에 적지 마라. 이유: 커밋에 시크릿이 남는다.
- 샘플 CSV에 실제 거래 내역이나 실존 카드번호를 넣지 마라.
- README를 영어로 쓰지 마라. 이유: 한국어 우선이 CRITICAL 규칙이다.
- 기존 테스트를 깨뜨리지 마라.
