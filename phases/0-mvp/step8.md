# Step 8: dashboard-ui

## 읽어야 할 파일

- `/docs/UI_GUIDE.md` — **이 step의 사양이다. 색상 슬롯·차트 규칙·컴포넌트 클래스를 그대로 따르라.**
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
3. `categoryBreakdown` / `periodTrend` / `periodOverPeriodDelta` 계산
4. plan이 `pro`면 `detectRecurring` / `detectAnomalies` 실행, `insights` 테이블 캐시 확인 후 없으면 `generateInsights` 호출하고 저장
5. 결과를 컴포넌트에 props로 내린다

`useEffect` + `fetch`로 초기 데이터를 받지 마라 (CLAUDE.md 규칙).

### 컴포넌트 (전부 `*.test.tsx` 먼저)

**1. `src/components/dashboard/SummaryTiles.tsx`**
이번 달 총지출(히어로 숫자), 전월 대비 증감, 거래 건수, 활성 구독 수.
증감은 증가 `#d03b3b` / 감소 `#0ca30c` + 화살표 아이콘. 색만으로 의미를 전달하지 마라.

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
`POST /api/statements`로 업로드. 드래그앤드롭 + 파일 선택.
진행 상태를 한국어로: `"파일을 읽는 중…"` → `"거래를 분류하는 중…"` → `"완료: 234건 추가, 12건 중복"`.
에러는 API가 준 한국어 메시지를 그대로 노출한다.

**8. `src/components/dashboard/CategoryEditor.tsx`** (Client Component)
거래 목록에서 카테고리를 고치는 셀렉트. `PATCH /api/transactions/[id]/category` 호출.

**9. `src/components/PlanGate.tsx`**
`canAccess(plan, feature)`가 false면 자식 대신 잠금 카드를 렌더한다.
문구: `"Pro 플랜에서 이용할 수 있습니다."` + 업그레이드 버튼(step 9의 `/api/checkout`으로 연결).
**블러 처리한 가짜 데이터를 보여주지 마라** — 정직하게 잠금 상태를 표시한다.

**10. 보관 기간 안내**
Free 사용자가 3개월 이전 데이터를 가진 경우, 추이 차트 위에 `"최근 3개월만 표시하고 있습니다. 이전 데이터는 삭제되지 않았으며 Pro에서 모두 보실 수 있습니다."`

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
   - 초기 데이터를 `useEffect`+`fetch`로 받지 않는가?
   - UI 문자열이 전부 한국어인가?
3. 결과에 따라 `phases/0-mvp/index.json`의 step 8을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 생성한 컴포넌트 목록 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 사용자 개입 필요 → `"status": "blocked"` + `"blocked_reason"` 후 즉시 중단

## 금지사항

- 이중 y축(y축 2개) 차트를 만들지 마라. 이유: 두 스케일을 겹치면 상관관계를 조작해 보여주게 된다. 차트를 나누거나 지수화하라.
- 시리즈 색을 순위에 따라 배정하지 마라. 이유: 필터를 바꿀 때마다 색이 바뀌면 사용자가 학습한 매핑이 깨진다. 색은 카테고리(엔티티)를 따른다.
- 잠긴 기능에 블러 처리한 가짜 데이터를 보여주지 마라. 이유: 금융 앱에서 가짜 숫자는 신뢰를 깬다.
- `UI_GUIDE.md` 안티패턴 표의 항목을 쓰지 마라 (glass morphism, 그라데이션 텍스트, "Powered by AI" 배지, 글로우 애니메이션, 보라색 브랜딩, gradient orb).
- 모든 데이터 포인트에 숫자 라벨을 찍지 마라. 이유: 차트가 표가 되어버린다. 표가 필요하면 표 보기 토글을 쓴다.
- 기존 테스트를 깨뜨리지 마라.
