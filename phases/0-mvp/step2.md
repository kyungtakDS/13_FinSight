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

매핑에 실패하면 `ParseResult`를 던지지 말고 **한국어 에러를 throw** 한다. **감지한 헤더를 메시지에 실어라** — "지원되는 컬럼명을 확인하세요"만으로는 사용자가 다음에 뭘 해야 할지 알 수 없다(USER_JOURNEY DE-04):

```
`CSV에서 거래일자·적요·금액 컬럼을 찾지 못했습니다. 이 파일의 첫 줄은 [${headers.join(", ")}] 입니다.`
```

**3. 값 정규화.**
- 날짜: `2026.07.27`, `2026/07/27`, `26.07.27`, `20260727` 등을 `2026-07-27`로. 파싱 불가 행은 `skippedRows`에 넣고 계속 진행한다.
- 금액: `₩`, `,`, 공백, `원` 제거. 괄호 표기(`(1,234)`)는 음수로 해석. **지출이 양수**가 되도록 부호를 맞춘다 — 출금/승인 컬럼은 양수, 입금/환불 컬럼은 음수.
- `merchant`: `description`에서 정규화. 뒤에 붙는 지점/일련번호(`스타벅스 강남2호점` → `스타벅스`), 앞의 카드사 접두어, `(주)`·`주식회사` 제거. 원본은 `description`에 그대로 남긴다.
  **이 정규화를 `export function normalizeMerchant(description: string): string` 로 따로 빼라.** 가맹점명이 카테고리 맵의 키가 되므로(ADR-009), 파싱과 조회가 반드시 같은 문자열을 만들어야 한다. 규칙이 두 곳에 복사되면 같은 가게가 두 항목으로 갈린다.

**4. 마스킹 (CRITICAL).** 저장 전에 민감 식별자를 마스킹한다. `src/lib/mask.ts`로 분리하고 `mask.test.ts`를 먼저 쓴다.

- **주민등록번호**: `\d{6}[-\s]?[1-4]\d{6}` → `901010-*******`.
  국내에서 가장 민감한 식별자다. 은행 거래내역에는 예금주 확인 정보로 섞여 들어올 수 있다. **이걸 빠뜨리면 다른 마스킹을 아무리 잘해도 의미가 없다.**
- **카드번호**: 13~19자리 숫자(구분자 `-`·공백·없음 모두) → `1234-****-****-5678`.
  **반드시 Luhn 검증을 통과한 것만 마스킹하라.** 국내 명세서에는 13~16자리 승인번호·거래일련번호가 흔한데, 자릿수만 보고 마스킹하면 그것들이 `raw`에서 지워진다. `raw`는 컬럼 매핑을 고쳤을 때 재처리하려고 남기는 사본인데(ADR-005), 과잉 마스킹하면 그 목적이 무너진다.
- **계좌번호**: 10자리 이상 숫자에 `-`가 2개 이상 섞인 패턴 → 앞 3자리·뒤 3자리만 남긴다.
- **`description`, `merchant`, `raw`의 모든 값에 적용한다.** `raw`에 마스킹 안 된 원본이 남으면 마스킹의 의미가 없다.

**4-1. 이체 행 식별 (CRITICAL — 개인정보).**

```ts
export function isTransferRow(description: string, raw: Record<string, string>): boolean;
```

은행 거래내역의 이체 행은 적요에 **가맹점이 아니라 사람 실명**이 들어간다(`홍길동`, `김철수`). 이 행의 `merchant`를 LLM에 보내면 **제3자의 실명을 외부 API로 전송**하는 것이 된다. 랜딩의 개인정보 문구는 "가맹점명이 전송됩니다"라고 쓰는데, 사용자는 그것을 "내가 송금한 사람 이름"으로 읽지 않는다.

- 판정: 적요/거래구분 컬럼에 `이체`, `송금`, `입금`, `출금`, `ATM`, `현금서비스` 등이 포함되거나, 카드 명세서가 아닌 은행 거래내역 헤더로 감지된 경우
- 해당 행은 `isTransfer: true`로 표시한다. **파싱 단계에서 지우지는 마라** — 지출 집계에는 여전히 필요하다.
- step 7이 이 플래그를 보고 LLM 분류 입력에서 제외하고 `기타`로 둔다.
- 어차피 LLM도 사람 이름은 카테고리로 분류하지 못한다. 정확도와 개인정보가 같은 방향을 가리키는 드문 경우다.

**5. `raw` 보존.** 마스킹을 거친 뒤의 원본 행 전체를 `Record<string, string>`으로 담는다 (ADR-005: 컬럼 매핑 수정 시 재처리용).

**6. 행 수 상한.** 50,000행을 넘으면 한국어 에러를 throw 한다. 넘지 않으면 전부 처리한다 — 횟수·행수 제한은 요금제와 무관하다(ADR-006).

**7. `periodStart` / `periodEnd`** 는 파싱된 거래의 최소·최대 `txnDate`.

**8. 빈 결과 처리.** 파싱 후 거래가 **0건이면** `ParseResult`를 반환하지 말고 한국어 에러를 throw 한다:
`"읽을 수 있는 거래가 없습니다. 건너뛴 행: N건"` + `skippedRows` 요약.
빈 배열의 최소·최대를 구하면 `periodStart`/`periodEnd`가 `Infinity`나 `undefined`가 되어 이후 전부 오염된다. 헤더만 있는 파일, 전 행의 날짜가 깨진 파일이 실제로 들어온다.

### 중복 방지 키

`src/lib/dedupe.ts`에 거래 지문을 만든다 (`dedupe.test.ts` 먼저).

```ts
export function transactionFingerprint(
  t: Pick<ParsedTransaction, "txnDate" | "amount" | "description">,
  occurrence: number,          // 이 파일 안에서 동일 조합이 몇 번째로 나왔는지 (0부터)
): string;

/** 파싱 결과 배열에 occurrence 번호를 매겨 지문을 붙인다 */
export function withFingerprints(txns: ParsedTransaction[]): (ParsedTransaction & { fingerprint: string })[];
```

`txnDate|amount|description|occurrence`를 정규화해 SHA-256 hex로 반환한다. Node 내장 `crypto`를 쓴다.

**`occurrence`를 빼지 마라.** 같은 날 같은 가게에서 같은 금액을 두 번 결제하는 일은 흔하다(커피 두 잔, 왕복 교통비). `occurrence` 없이 지문을 만들면 두 번째 거래가 중복으로 간주돼 **조용히 사라진다.** 게다가 step 5의 `duplicate_suspect` 이상 탐지는 정확히 "같은 가맹점·같은 금액이 24시간 이내 2건 이상"을 찾는데, 그 데이터를 파서가 이미 지워버린 뒤라 영원히 탐지되지 않는다.

`occurrence`를 넣어도 **같은 파일을 다시 올리면 같은 지문 집합이 나오므로** 재업로드 중복 방지는 그대로 동작한다. 파일 안에서 등장 순서대로 0, 1, 2… 를 매기면 된다.

### 테스트에 반드시 포함할 케이스

- UTF-8 / CP949 각각의 샘플 (CP949는 `iconv-lite`로 인코딩해 만든다)
- 카드사별로 다른 헤더 3종 이상
- 날짜 포맷 4종
- 괄호 음수, 통화기호, 천단위 콤마
- **마스킹**: 카드번호가 `description`과 `raw` 양쪽에서 모두 마스킹되는지 단언
- **마스킹**: 주민등록번호 패턴이 마스킹되는지
- **마스킹**: Luhn을 통과하지 못하는 16자리 승인번호는 **마스킹되지 않는지** (과잉 마스킹 방지)
- **이체 행**: `홍길동` 같은 적요가 `isTransfer: true`로 표시되는지
- **빈 결과**: 헤더만 있는 CSV, 전 행 날짜가 깨진 CSV → 한국어 에러 throw
- 매핑 실패 시 한국어 에러 메시지
- **같은 파일을 두 번 파싱하면 지문 집합이 동일한지** (재업로드 중복 방지가 동작해야 한다)
- **같은 날·같은 가맹점·같은 금액 거래 2건이 서로 다른 지문을 갖는지** (정상 중복이 살아남아야 한다)

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
   - 외부 자격증명·대시보드 설정이 필요한 검증은 루트 `DEPLOY_CHECKLIST.md`에 `- [ ] (미검증)`으로 append 하고 step은 `"completed"`로 처리한다. 코드·로컬 테스트 자체를 진행할 수 없는 경우에만 `"status": "blocked"` + `"blocked_reason"` 후 중단한다.

## 금지사항

- 이 step에서 LLM을 호출하지 마라. 이유: 카테고리 분류는 step 6의 범위다. 여기서 `category`는 항상 미설정으로 둔다.
- DB에 저장하지 마라. 이유: 파서는 순수 함수여야 테스트가 빠르고 결정론적이다. 저장은 step 7.
- 파싱 실패 행 때문에 전체를 중단하지 마라. 이유: 명세서 한 행이 깨졌다고 나머지 수백 건을 버리면 안 된다. `skippedRows`에 기록하고 계속한다.
- 마스킹을 `description`에만 적용하고 `raw`를 건너뛰지 마라. 이유: `raw`가 그대로 DB에 들어가므로 마스킹이 무효가 된다.
- 자릿수만 보고 카드번호로 판정하지 마라. 이유: 승인번호·거래일련번호가 오인 마스킹되어 `raw`의 재처리 가치가 사라진다. Luhn을 통과한 것만 마스킹한다.
- 이체 행을 파싱 단계에서 버리지 마라. 이유: 지출 집계에는 필요하다. 버리지 말고 `isTransfer`로 표시만 하고, LLM 전송 제외는 step 7이 한다.
- 거래 0건인 결과를 정상 반환하지 마라. 이유: `periodStart`/`periodEnd`가 `Infinity`가 되어 이후 전 단계가 오염된다.
- 기존 테스트를 깨뜨리지 마라.
