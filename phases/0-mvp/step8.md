# Step 8: dashboard-ui

## 읽어야 할 파일

- `/docs/UI_GUIDE.md` — **이 step의 사양이다. 색상 슬롯·차트 규칙·컴포넌트 클래스를 그대로 따르라.**
- `phases/0-mvp/USER_JOURNEY.md` — **11장 "화면 × 상태 매트릭스"가 이 step의 두 번째 사양이다.** 각 화면이 비었을 때·부분적일 때·실패했을 때·권한이 없을 때를 전부 처리해야 한다. 성공 경로만 그리면 이 step은 실패다.
- `/docs/PRD.md` — 핵심 기능 2~5번과 요금제 표
- `/docs/ARCHITECTURE.md` — 데이터 흐름의 `[대시보드 조회]` 구간
- `src/lib/analytics/` 전체 (step 5) — 집계는 여기 함수를 호출한다. UI에서 다시 계산하지 마라.
- `src/lib/plan.ts` (step 3) — `canAccess`, `retentionCutoff`
- `src/services/anthropic.ts` (step 6) — `generateInsights`
- `src/lib/format.ts` (step 0) — `formatKRW`, `formatDate`
- `src/app/(app)/layout.tsx`, `src/app/(app)/dashboard/page.tsx` (step 4)

## 작업

PRD의 분석 4종을 대시보드에 렌더한다. 차트 라이브러리는 `recharts`.

### 데이터 로딩

`src/app/(app)/dashboard/page.tsx` (Server Component, TDD 면제):
1. 세션 + `profiles.plan` 조회
2. `transactions` 조회 → `applyRetention(txns, plan)` 적용
3. **거래가 하나도 없으면 빈 상태를 렌더하고 끝낸다.** 업로드 영역 + `/help/csv` 도움말 링크 + 샘플 CSV 내려받기. 빈 대시보드에 업로드 박스만 두면 CSV를 어디서 받는지 모르는 사용자가 여기서 멈춘다(DE-03).
4. `categoryBreakdown` / `periodTrend` / `periodOverPeriodDelta` 계산. 히어로 기준 기간은 **가장 최근 `statements.period_end`가 속한 기간**이다.
5. **`detectRecurring` / `detectAnomalies`는 plan과 무관하게 실행한다.** Free에도 건수·총액을 보여줘야 하기 때문이다(ADR-010). 전부 코드 계산이라 비용이 없다. 잠그는 것은 목록이지 계산이 아니다.
6. plan이 `pro`일 때만 `insights` 테이블 캐시를 확인하고, 없으면 `generateInsights`를 호출해 저장한다. **이 호출이 실패해도 나머지 화면은 정상 렌더한다** — LLM 하나 때문에 대시보드 전체가 죽으면 안 된다.
7. 결과를 컴포넌트에 props로 내린다

`useEffect` + `fetch`로 초기 데이터를 받지 마라 (CLAUDE.md 규칙).

### 컴포넌트 (전부 `*.test.tsx` 먼저)

**1. `src/components/dashboard/SummaryTiles.tsx`**
히어로 숫자는 **"이번 달"이 아니라 가장 최근 명세서 기간의 총지출**이다(DE-10). 명세서는 지난달 것이 이번 달에 나오므로, 7월에 6월 명세서를 올린 사용자의 "이번 달 총지출"은 ₩0이다. 첫 화면이 0원이면 앱이 고장 난 것으로 보인다.
기간을 숫자 옆에 반드시 표기한다: `"2026년 6월 · ₩2,340,900"`.
그 밖에 전월 대비 증감, 거래 건수, 활성 구독 수.
증감은 증가 `#d03b3b` / 감소 `#0ca30c` + 화살표 아이콘. 색만으로 의미를 전달하지 마라.
비교할 이전 기간이 없으면 증감 타일에 `"비교할 이전 기간이 없습니다"`를 표시한다 — `0%`나 `—`로 얼버무리지 마라.

**2. `src/components/dashboard/CategoryBreakdownChart.tsx`**
가로 막대. `UI_GUIDE.md`의 8슬롯을 **카테고리 → 색 고정 매핑**으로 쓴다. 필터로 카테고리가 줄어도 남은 것의 색이 바뀌면 안 된다.
- 막대 데이터 끝 4px 라운드, 기준선 쪽은 각지게
- 인접 막대 사이 2px 표면 간격
- 4개 이하면 직접 라벨, 그 이상이면 범례 + 호버 툴팁
- 표 보기 토글 제공

**3. `src/components/dashboard/TrendChart.tsx`**
월별 총지출 선 그래프 + 카테고리별 누적 영역 토글.
- 선 2px, 마커 8px 이상
- **크로스헤어 + 툴팁 필수**
- **y축은 하나만.** 이중 축을 만들지 마라.
- 거래 없는 달도 0으로 표시 (step 5의 `periodTrend`가 이미 채워준다)
- 최댓값·최솟값·최신값만 직접 라벨. 모든 점에 숫자를 찍지 마라.

**4. `src/components/dashboard/RecurringList.tsx`** (Pro 전용)
활성 구독을 금액 내림차순 표. `merchant`, `medianAmount`, `intervalDays`("월 1회"/"연 1회"로 한국어 변환), `lastSeenAt`.
- `status: "dormant"`는 하단에 "결제가 멈춘 구독" 섹션으로 분리
- **"쓰지 않는 구독"이라고 단정하지 마라.** 명세서만으로는 사용 여부를 알 수 없다. 문구는 `"매달 빠져나가는 구독입니다. 필요한지 확인해보세요."` 수준으로.
- 연 환산액(`medianAmount × 12/주기`)을 함께 보여주면 판단에 도움이 된다

**5. `src/components/dashboard/AnomalyList.tsx`** (Pro 전용)
`severity`별 아이콘 + 한국어 `detail`. `critical`은 `#d03b3b`, `warning`은 `#fab219`. 각 항목은 해당 거래로 드릴다운.

**6. `src/components/dashboard/InsightsPanel.tsx`** (Pro 전용)
`summary` 본문 + `savingSuggestions` 카드 목록(절감액 내림차순, `formatKRW` 적용).
LLM 결과임을 알리는 짧은 각주를 둔다. **"Powered by AI" 배지를 만들지 마라** (UI_GUIDE 안티패턴).

**7. `src/components/dashboard/UploadDropzone.tsx`** (Client Component)
드래그앤드롭 + 파일 선택. **호출은 두 번이다**(ADR-008): `POST /api/statements` → 응답을 받아 화면을 갱신한 뒤 → `POST /api/statements/[id]/classify`.

진행 상태 문구는 `USER_JOURNEY.md` 6장 표를 그대로 쓴다. 임의로 바꾸지 마라.
`"파일을 읽는 중…"` → `"거래를 저장하는 중…"` → **`"234건 추가 · 12건 중복 · 3건 건너뜀"`(여기서 이미 거래 목록을 볼 수 있다)** → `"새 가맹점 28곳을 분류하는 중…"` → `"완료. 6월 지출 ₩2,340,900"`

- `skippedRows`가 있으면 **접힌 목록으로 행 번호와 사유를 노출한다.** 조용히 사라지게 두지 마라 — 3건이 말없이 빠지는 것과 "3건 건너뜀 · 어느 행인지 보기"는 신뢰에서 완전히 다르다.
- 2단계가 실패해도 1단계 결과는 화면에 남긴다. `"카테고리 분류에 실패했습니다. 거래는 모두 저장되어 있습니다."` + 다시 분류하기.
- 컬럼 매핑 실패 에러(감지된 헤더 포함)를 그대로 노출하고 **도움말 `/help/csv` 링크를 함께 건다**(DE-04).
- 모바일에서 드래그 없이 파일 선택만으로 완주할 수 있어야 한다(UC-19).

**8. `src/components/dashboard/CategoryEditor.tsx`** (Client Component)
거래 목록에서 카테고리를 고치는 셀렉트. `PATCH /api/transactions/[id]/category` 호출.
기본값은 `applyToMerchant: true`이고, 반영 결과를 알린다: `"스타벅스 42건을 식비로 바꿨습니다."` (DE-06)
한 건만 바꾸고 싶은 경우를 위해 체크박스로 끌 수 있게 한다.

**9. `src/components/PlanGate.tsx`**
`canAccess(plan, feature)`가 false면 자식 대신 잠금 카드를 렌더한다.
문구: `"Pro 플랜에서 이용할 수 있습니다."` + 업그레이드 버튼(step 9의 `/api/checkout`으로 연결).
**블러 처리한 가짜 데이터를 보여주지 마라** — 정직하게 잠금 상태를 표시한다.

`summary` prop을 받아 잠금 카드 안에 **실제 계산한 요약 수치**를 표시한다(ADR-010):
- 구독 목록 잠금 → `"반복 결제 12건 · 월 ₩84,300"`
- 이상 거래 잠금 → `"이상 거래 3건"`

이 수치는 Free 사용자에게도 실제로 `detectRecurring` / `detectAnomalies`를 돌려 얻은 값이다. 가짜 데이터가 아니므로 안티패턴에 해당하지 않는다. 반대로 **AI 인사이트에는 요약을 붙이지 마라** — 전체가 잠긴다.

**10. 미분류 배너** — `category`가 null인 거래가 있으면 대시보드 상단에
`"거래 34건이 아직 분류되지 않았습니다."` + **다시 분류하기** 버튼(step 7의 `POST /api/statements/[id]/classify`). 분류가 실패했거나 사용자가 도중에 나갔을 때 되돌아올 유일한 경로다(DE-05, UC-17).

**11. `src/components/dashboard/UpgradeConfirm.tsx`** (Client Component)
`/dashboard?upgraded=1`로 돌아왔는데 아직 `plan === "free"`일 때 렌더한다(DE-07). Polar 리다이렉트가 웹훅보다 먼저 도착할 수 있기 때문이다.
- `"결제를 확인하고 있습니다. 잠시만 기다려주세요."` + step 9의 플랜 확인 엔드포인트를 1.5초 간격으로 최대 20초 폴링
- `pro`가 되면 배너를 지우고 화면을 새로 그린다
- 20초 안에 안 되면 `"결제는 완료되었지만 반영이 늦어지고 있습니다. 잠시 후 새로고침하거나 구독 관리에서 확인해주세요."` + 구독 관리 링크
- **이것을 결제 실패로 표시하지 마라.** 돈은 이미 나갔다.

**12. 보관 기간 안내**
Free 사용자가 3개월 이전 데이터를 가진 경우, 추이 차트 위에 `"최근 3개월만 표시하고 있습니다. 이전 데이터는 삭제되지 않았으며 Pro에서 모두 보실 수 있습니다."`
업로드 직후에도 같은 안내를 띄운다 — 12개월치를 올린 사용자가 3개월만 보이는 이유를 그 자리에서 알아야 한다(UC-22).

### 접근성

- 시리즈가 2개 이상이면 범례는 **항상** 있다. 1개면 제목이 이름을 대신하므로 범례를 만들지 않는다.
- 모든 차트에 표 보기 토글. 색을 못 구분하는 사용자도 값을 읽을 수 있어야 한다.
- 상태 색에는 아이콘 + 라벨을 동반한다.

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트:
   - 컴포넌트가 집계를 다시 계산하지 않고 `src/lib/analytics/`를 쓰는가?
   - 카테고리 → 색 매핑이 `UI_GUIDE.md` 8슬롯 순서와 일치하고, 고정 매핑인가?
   - 이중 y축 차트가 없는가?
   - Pro 전용 컴포넌트가 `PlanGate`로 감싸여 있는가?
   - **`USER_JOURNEY.md` 11장 매트릭스의 8개 화면 × 4개 상태가 전부 처리되었는가?**
   - **히어로 숫자가 "이번 달"이 아니라 최근 명세서 기간 기준인가?**
   - **구독·이상 거래 요약 수치가 Free에도 표시되는가?** (계산까지 잠그면 안 된다)
   - 미분류 배너와 다시 분류하기 버튼이 있는가?
   - 초기 데이터를 `useEffect`+`fetch`로 받지 않는가?
   - UI 문자열이 전부 한국어인가?
3. 결과에 따라 `phases/0-mvp/index.json`의 step 8을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 생성한 컴포넌트 목록 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 사용자 개입 필요 → `"status": "blocked"` + `"blocked_reason"` 후 즉시 중단

## 금지사항

- 이중 y축(y축 2개) 차트를 만들지 마라. 이유: 두 스케일을 겹치면 상관관계를 조작해 보여주게 된다. 차트를 나누거나 지수화하라.
- 시리즈 색을 순위에 따라 배정하지 마라. 이유: 필터를 바꿀 때마다 색이 바뀌면 사용자가 학습한 매핑이 깨진다. 색은 카테고리(엔티티)를 따른다.
- 잠긴 기능에 블러 처리한 가짜 데이터를 보여주지 마라. 이유: 금융 앱에서 가짜 숫자는 신뢰를 깬다. (잠금 카드의 요약 수치는 실제 계산값이므로 이에 해당하지 않는다.)
- Free 사용자에게 `detectRecurring`·`detectAnomalies` **계산 자체를 건너뛰지 마라.** 이유: 건수·총액을 보여줘야 하고, 그게 유일한 전환 근거다(ADR-010).
- 성공 경로만 렌더하지 마라. 이유: 빈 상태·부분 상태·실패 상태가 실제 사용에서 훨씬 자주 나온다. 매트릭스의 모든 칸이 이 step의 요구사항이다.
- 결제 반영이 늦은 것을 결제 실패로 표시하지 마라. 이유: 돈은 이미 나갔고, 사용자는 사기당했다고 느낀다.
- `UI_GUIDE.md` 안티패턴 표의 항목을 쓰지 마라 (glass morphism, 그라데이션 텍스트, "Powered by AI" 배지, 글로우 애니메이션, 보라색 브랜딩, gradient orb).
- 모든 데이터 포인트에 숫자 라벨을 찍지 마라. 이유: 차트가 표가 되어버린다. 표가 필요하면 표 보기 토글을 쓴다.
- 기존 테스트를 깨뜨리지 마라.
