# Step 6: anthropic-service

## 읽어야 할 파일

- `/CLAUDE.md` — LLM은 분류와 서술에만 쓴다는 CRITICAL 규칙, 언어 규칙
- `/docs/ARCHITECTURE.md` — "분석 파이프라인" 표, 데이터 흐름의 LLM 호출 ①②
- `/docs/ADR.md` — ADR-003, ADR-004
- `src/types/transaction.ts` (`CATEGORIES`), `src/types/analysis.ts` (`Insight`, `insightSchema`) — step 1 산출물
- `src/lib/analytics/` (step 5 산출물) — 인사이트의 **입력**이 되는 집계 결과

## 작업

`src/services/anthropic.ts`에 Claude API 래퍼를 만든다. LLM 호출은 **딱 두 개**다. `anthropic.test.ts`를 먼저 작성하라 (TDD Guard). 테스트는 SDK를 `vi.mock`으로 가짜 응답을 주고 검증한다 — 실제 API를 호출하지 마라.

**이 테스트 파일에 `// @vitest-environment jsdom`을 붙이지 마라.** Anthropic SDK는 jsdom 환경을 지원하지 않는다고 공식 문서에 명시돼 있다. step 0이 기본 환경을 `node`로 잡아뒀으니 아무것도 붙이지 않으면 된다.

### 공통 규칙

- 모델은 `claude-opus-5`. 다른 모델을 쓰지 마라.
- 클라이언트는 함수 내부에서 lazy 생성한다. `ANTHROPIC_API_KEY`가 없으면 한국어 에러: `"ANTHROPIC_API_KEY 환경변수가 설정되지 않았습니다."` **모듈 최상단에서 키를 읽으면 키 없이 `npm run build`가 실패한다.**
- 파일 최상단에 `import "server-only";`
- 응답은 **구조화 출력으로 스키마를 강제**한다. 직접 JSON 파싱 + 재시도 로직을 만들지 마라.
  ```ts
  import { zodOutputFormat } from "@anthropic-ai/sdk/helpers/zod";
  const res = await client.messages.parse({
    model: "claude-opus-5",
    max_tokens: 16000,
    output_config: { effort: "low", format: zodOutputFormat(Schema) },
    messages: [...],
  });
  res.parsed_output   // 타입이 붙은 결과
  ```
  `zodOutputFormat`은 **인자를 하나만** 받는다 (`@anthropic-ai/sdk/helpers/zod`에서 확인됨):
  ```ts
  declare function zodOutputFormat<T extends z.ZodType>(zodObject: T): AutoParseableOutputFormat<z.infer<T>>;
  ```
  이 헬퍼의 타입 정의는 `import * as z from 'zod/v4'`로 시작한다 — **zod 4.x가 필요하다.** step 0이 `zod@^4`를 설치했는지 먼저 확인하라. 3.x면 `zod/v4` 서브패스가 없어 컴파일되지 않는다.
  그래도 타입 에러가 나면 `node_modules/@anthropic-ai/sdk/helpers/zod.d.mts`에서 실제 시그니처를 읽어 맞춰라. 최후 수단으로 zod 대신 raw JSON Schema를 `output_config.format`에 직접 넣을 수 있다.
- `max_tokens`는 넉넉히 잡는다(16000). Claude Opus 5는 **thinking이 기본 활성**이고 `max_tokens`가 thinking + 응답을 함께 제한하므로, 작게 잡으면 응답이 중간에 잘린다.
- `stop_reason === "refusal"`을 먼저 확인한 뒤 결과를 읽어라. 거절 시 한국어 에러를 throw 한다.
- `temperature` / `top_p` / `top_k` / `budget_tokens`를 쓰지 마라 — Claude Opus 5에서 400 에러다. 깊이 조절은 `output_config.effort`로 한다.

### LLM 호출 ① — 가맹점 카테고리 분류

```ts
export async function classifyMerchants(merchants: string[]): Promise<Record<string, Category>>;
```

- 입력은 **유니크 가맹점명 목록**이다. 거래 원문 전체를 넣지 마라 (ADR-003: 토큰 폭증).
- 200개씩 배치로 나눠 호출하고 결과를 병합한다.
- 출력 스키마는 `{ results: { merchant: string; category: enum(CATEGORIES) }[] }`. enum으로 제한해 모델이 없는 카테고리를 지어내지 못하게 한다.
- 응답에 없는 가맹점은 `"기타"`로 채운다.
- 시스템 프롬프트는 한국어로 각 카테고리의 경계를 정의한다. 특히 헷갈리는 것을 명시하라: 배달 앱은 `식비`, 주유·주차·택시는 `교통`, OTT·통신 부가서비스·클라우드 요금은 `구독`, 관리비·전기·가스는 `주거·공과금`.
- `effort: "low"` — 분류는 단순 작업이다.
- 프롬프트 캐싱: 시스템 프롬프트가 512토큰을 넘으면 `cache_control: { type: "ephemeral" }`을 붙여라. 배치가 여러 번 돌기 때문에 효과가 있다.

### LLM 호출 ② — 요약 & 절약 인사이트

```ts
export async function generateInsights(input: {
  breakdown: CategoryBreakdown[];
  trend: PeriodPoint[];
  recurring: RecurringCharge[];
  anomalies: AnomalyFlag[];
}): Promise<Insight>;
```

- 입력은 **step 5가 계산해둔 집계 결과**다. 개별 거래 배열을 넣지 마라.
- 출력 스키마는 step 1의 `insightSchema`.
- 시스템 프롬프트 요구사항:
  - 출력은 **한국어**, 금액은 `₩1,234,567` 형식
  - `summary`는 3~5문장
  - `savingSuggestions`는 **입력에 있는 숫자를 인용**해야 한다. "식비를 줄이세요" 같은 일반론 금지. `"배달 결제 18건 ₩342,000, 전월 대비 +41%"`처럼 구체적으로.
  - **입력에 없는 숫자를 만들어내지 마라.** 계산이 필요하면 입력의 값을 그대로 인용한다.
  - `estimatedMonthlySaving`은 제안이 실행됐을 때의 월 절감 추정액(원 단위 정수)
  - 최대 5개 제안, 절감액 내림차순
- `effort: "medium"`

### 캐시

**캐시 키 함수를 만들지 마라.** 인사이트는 `insights` 테이블에 **사용자당 한 행**(unique `user_id`)으로 저장되고, step 7의 2단계가 업로드할 때마다 덮어쓴다. 호출 시점이 이미 "새 데이터가 들어왔을 때"로 제한돼 있으므로 별도의 무효화 규칙이 필요 없다. **대시보드(step 8)는 이 행을 읽기만 하고 `generateInsights`를 부르지 않는다**(ADR-011).

### 테스트에 반드시 포함할 케이스

- `ANTHROPIC_API_KEY` 미설정 시 한국어 에러
- `classifyMerchants`: 250개 입력 → 2번 호출되는지 (배치 분할), 응답에 없는 가맹점이 `"기타"`로 채워지는지
- `classifyMerchants`에 넘긴 프롬프트에 **거래 금액·날짜가 포함되지 않는지** 단언 (가맹점명만 나가야 한다)
- `generateInsights`에 넘긴 프롬프트에 **개별 거래 배열이 포함되지 않는지** 단언
- `stop_reason: "refusal"` 응답 시 한국어 에러 throw
- 호출 파라미터에 `temperature`가 없는지 단언

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트:
   - 모델 문자열이 `claude-opus-5`인가?
   - `import "server-only"`가 있는가?
   - 환경변수를 모듈 최상단에서 읽지 않는가? (키 없이 build 통과)
   - 프롬프트에 개별 거래 원문이 들어가지 않는가?
   - `temperature`/`budget_tokens`를 쓰지 않는가?
3. 결과에 따라 `phases/0-mvp/index.json`의 step 6을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 두 함수 시그니처와 사용 모델·effort 설정 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 사용자 개입 필요 → `"status": "blocked"` + `"blocked_reason"` 후 즉시 중단

## 금지사항

- LLM에게 합계·비율·증감률을 계산시키지 마라. 이유: ADR-003. 검증 불가능하고, 사용자가 "이 숫자 왜 이래?"라고 물으면 답할 수 없다.
- 거래 원문 배열을 프롬프트에 넣지 마라. 이유: 명세서 한 건에 수만 토큰이 든다. 유니크 가맹점명은 수백 토큰이다.
- 실제 Anthropic API를 테스트에서 호출하지 마라. 이유: 키가 없으면 실패하고, 있어도 테스트가 느리고 비결정론적이 된다.
- `temperature`, `top_p`, `top_k`, `thinking.budget_tokens`를 쓰지 마라. 이유: Claude Opus 5에서 400 에러다.
- `claude-opus-5` 외의 모델로 바꾸지 마라. 이유: 비용을 이유로 모델을 낮추는 것은 사용자 결정이다.
- 기존 테스트를 깨뜨리지 마라.
