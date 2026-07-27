# Step 2: csv-parser

## 읽어야 할 파일

- `/CLAUDE.md` — 마스킹 규칙과 언어 규칙
- `/docs/ARCHITECTURE.md` — 데이터 흐름의 `[업로드]` 구간
- `/docs/ADR.md` — ADR-005 (원본 보관)
- `src/types/transaction.ts` (step 1 산출물) — `ParsedTransaction`을 그대로 쓴다. 새 타입을 만들지 마라.

## 작업

`src/lib/csv.ts`에 CSV → `ParsedTransaction[]` 변환기를 구현한다. 순수 함수만 — 파일 I/O도 DB 접근도 하지 않는다. 입력은 `ArrayBuffer`(업로드된 바이트)다.

**TDD Guard 훅 때문에 `src/lib/csv.test.ts`를 먼저 작성해야 한다.**

### 공개 인터페이스

```ts
export interface ParseResult {
  transactions: ParsedTransaction[];
  periodStart: string;
  periodEnd: string;
  detectedEncoding: "utf-8" | "cp949";
  sourceHint: string | null;      // 헤더 패턴으로 추정한 카드사/은행명
  skippedRows: { rowNumber: number; reason: string }[];  // reason은 한국어
}

export function parseStatementCsv(buffer: ArrayBuffer, filename: string): ParseResult;
```

### 구현 요구사항

**1. 인코딩 감지.** UTF-8로 디코딩했을 때 U+FFFD(치환 문자)가 나오면 CP949로 재디코딩한다. `iconv-lite`를 쓴다. 국내 카드사 명세서는 CP949가 흔하다.

**2. 헤더 매핑.** 카드사마다 컬럼명이 다르다. 다음 후보군을 소문자·공백제거 후 매칭한다 (완전일치 우선, 없으면 부분일치):

| 목표 필드 | 후보 헤더 |
|---|---|
| `txnDate` | 거래일자, 거래일, 이용일, 이용일자, 승인일자, 날짜, date |
| `description` | 적요, 내용, 가맹점명, 이용가맹점, 거래내용, 상호, description, merchant |
| `amount` | 금액, 이용금액, 거래금액, 승인금액, 출금액, 결제금액, amount |

매핑에 실패하면 `ParseResult`를 던지지 말고 **한국어 에러를 throw** 한다: `"CSV에서 거래일자/적요/금액 컬럼을 찾지 못했습니다. 지원되는 컬럼명을 확인해주세요."`

**3. 값 정규화.**
- 날짜: `2026.07.27`, `2026/07/27`, `26.07.27`, `20260727` 등을 `2026-07-27`로. 파싱 불가 행은 `skippedRows`에 넣고 계속 진행한다.
- 금액: `₩`, `,`, 공백, `원` 제거. 괄호 표기(`(1,234)`)는 음수로 해석. **지출이 양수**가 되도록 부호를 맞춘다 — 출금/승인 컬럼은 양수, 입금/환불 컬럼은 음수.
- `merchant`: `description`에서 정규화. 뒤에 붙는 지점/일련번호(`스타벅스 강남2호점` → `스타벅스`), 앞의 카드사 접두어, `(주)`·`주식회사` 제거. 원본은 `description`에 그대로 남긴다.

**4. 마스킹 (CRITICAL).** 저장 전에 카드번호·계좌번호 패턴을 마스킹한다.
- 카드번호: 13~16자리 연속 숫자 또는 `4자리-4자리-4자리-4자리` → `1234-****-****-5678`
- 계좌번호: 10자리 이상 숫자에 `-`가 2개 이상 섞인 패턴 → 앞 3자리·뒤 3자리만 남기고 마스킹
- **`description`, `merchant`, `raw`의 모든 값에 적용한다.** `raw`에 마스킹 안 된 원본이 남으면 마스킹의 의미가 없다.

**5. `raw` 보존.** 마스킹을 거친 뒤의 원본 행 전체를 `Record<string, string>`으로 담는다 (ADR-005: 컬럼 매핑 수정 시 재처리용).

**6. 행 수 상한.** 50,000행을 넘으면 한국어 에러를 throw 한다. 넘지 않으면 전부 처리한다 — 횟수·행수 제한은 요금제와 무관하다(ADR-006).

**7. `periodStart` / `periodEnd`** 는 파싱된 거래의 최소·최대 `txnDate`.

### 중복 방지 키

`src/lib/dedupe.ts`에 거래 지문을 만든다 (`dedupe.test.ts` 먼저).

```ts
export function transactionFingerprint(t: Pick<ParsedTransaction, "txnDate" | "amount" | "description">): string;
```

`txnDate|amount|description`을 정규화해 SHA-256 hex로 반환한다. Node 내장 `crypto`를 쓴다.

### 테스트에 반드시 포함할 케이스

- UTF-8 / CP949 각각의 샘플 (CP949는 `iconv-lite`로 인코딩해 만든다)
- 카드사별로 다른 헤더 3종 이상
- 날짜 포맷 4종
- 괄호 음수, 통화기호, 천단위 콤마
- **마스킹**: 카드번호가 `description`과 `raw` 양쪽에서 모두 마스킹되는지 단언
- 매핑 실패 시 한국어 에러 메시지
- 같은 거래 2건의 지문이 동일한지

## Acceptance Criteria

```bash
npm run lint
npm run build
npm run test
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트:
   - `src/lib/csv.ts`가 `fs`·`fetch`·Supabase를 import 하지 않는가? (순수 함수여야 한다)
   - 마스킹이 `raw`에도 적용되는 테스트가 있는가?
   - `src/types/transaction.ts`의 `ParsedTransaction`을 재사용했는가? (중복 타입 정의 금지)
3. 결과에 따라 `phases/0-mvp/index.json`의 step 2를 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary"`에 `parseStatementCsv` / `transactionFingerprint` 시그니처와 파일 경로 요약
   - 3회 시도 후 실패 → `"status": "error"` + `"error_message"`
   - 사용자 개입 필요 → `"status": "blocked"` + `"blocked_reason"` 후 즉시 중단

## 금지사항

- 이 step에서 LLM을 호출하지 마라. 이유: 카테고리 분류는 step 6의 범위다. 여기서 `category`는 항상 미설정으로 둔다.
- DB에 저장하지 마라. 이유: 파서는 순수 함수여야 테스트가 빠르고 결정론적이다. 저장은 step 7.
- 파싱 실패 행 때문에 전체를 중단하지 마라. 이유: 명세서 한 행이 깨졌다고 나머지 수백 건을 버리면 안 된다. `skippedRows`에 기록하고 계속한다.
- 마스킹을 `description`에만 적용하고 `raw`를 건너뛰지 마라. 이유: `raw`가 그대로 DB에 들어가므로 마스킹이 무효가 된다.
- 기존 테스트를 깨뜨리지 마라.
