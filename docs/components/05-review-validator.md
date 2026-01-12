# Review Validator 컴포넌트

## 1. 개요

### 1.1 목적
Code Reviewer가 생성한 리뷰 아이템의 품질을 검증하고, 할루시네이션을 필터링하며, inline comment를 위한 정확한 라인 번호를 추출합니다.

### 1.2 위치
Code Reviewer 이후, Summary Generator 이전에 위치합니다.

```
[Code Reviewer] ──→ [Review Validator] ──→ [Summary Generator]
                          │
                          ├─→ 유효한 리뷰만 통과
                          └─→ 라인 번호 보정
```

### 1.3 현재 문제

현재 시스템은 검증 단계에서도 단일 LLM 호출로 여러 작업을 수행:

| 작업 | 설명 |
|------|------|
| 변경점 실제성 확인 | review item이 실제 변경점을 가리키는지 |
| 설명 적합성 | 설명이 코드 변경과 일치하는지 |
| 제안 코드 적절성 | 제안된 코드가 문법적/논리적으로 올바른지 |
| 인코딩 검증 | 코드 스니펫의 인코딩이 깨지지 않았는지 |
| 할루시네이션 탐지 | 존재하지 않는 문제를 지적하지 않았는지 |
| 라인 번호 추출 | inline comment 위치 계산 |

---

## 2. 입출력 명세

### 2.1 입력 (Input)

```typescript
interface ValidatorInput {
  // 원본 컨텍스트
  pr_description: string;
  refined_spec: RefinedSpec;
  file_diff: FileDiff;
  function_code: string;
  pr_full_diff?: string;

  // 검증 대상
  review_result: ReviewResult;
}
```

### 2.2 출력 (Output)

```typescript
interface ValidatedReviewResult {
  file_name: string;
  validated_issues: ValidatedIssue[];
  filtered_issues: FilteredIssue[];      // 필터링된 이슈 (사유 포함)
  validation_summary: ValidationSummary;
}

interface ValidatedIssue {
  // 원본 Issue 필드 모두 포함
  original_issue: Issue;

  // 검증 결과
  validation: IssueValidation;

  // 라인 번호 (inline comment용)
  inline_position: InlinePosition;
}

interface IssueValidation {
  is_valid: boolean;
  checks: ValidationCheck[];
  confidence: number;
}

interface ValidationCheck {
  check_type: ValidationCheckType;
  passed: boolean;
  reason: string;
}

type ValidationCheckType =
  | "change_exists"        // 변경점이 실제로 존재하는지
  | "description_accurate" // 설명이 정확한지
  | "suggestion_valid"     // 제안 코드가 유효한지
  | "encoding_ok"          // 인코딩이 정상인지
  | "not_hallucination"    // 할루시네이션이 아닌지
  | "line_range_valid";    // 라인 범위가 유효한지

interface InlinePosition {
  // diff 기준 위치
  diff_line_start: number;
  diff_line_end: number;

  // 원본 파일 기준 위치
  file_line_start: number;
  file_line_end: number;

  // 변경 타입
  position_type: "added" | "modified" | "context";

  // 정확도
  position_confidence: number;
}

interface FilteredIssue {
  original_issue: Issue;
  filter_reason: string;
  failed_checks: ValidationCheckType[];
}

interface ValidationSummary {
  total_issues: number;
  valid_issues: number;
  filtered_issues: number;
  filter_rate: number;
  common_filter_reasons: string[];
}
```

**출력 예시**:
```json
{
  "file_name": "drivers/nand/nand_read.c",
  "validated_issues": [
    {
      "original_issue": {
        "id": "ISS-001",
        "type": "memory",
        "severity": "major",
        "line_start": 143,
        "line_end": 145,
        "title": "free 후 buffer 포인터 미초기화",
        "description": "buffer를 free한 후 NULL로 설정하지 않아 dangling pointer 위험",
        "code_snippet": "if (buffer != NULL) {\n    free(buffer);\n}",
        "suggested_code": "if (buffer != NULL) {\n    free(buffer);\n    buffer = NULL;\n}"
      },
      "validation": {
        "is_valid": true,
        "checks": [
          {
            "check_type": "change_exists",
            "passed": true,
            "reason": "143-145 라인에 해당 코드 변경 확인됨"
          },
          {
            "check_type": "description_accurate",
            "passed": true,
            "reason": "free 후 포인터 초기화 누락이 실제로 확인됨"
          },
          {
            "check_type": "suggestion_valid",
            "passed": true,
            "reason": "제안 코드가 문법적으로 올바르고 문제를 해결함"
          },
          {
            "check_type": "encoding_ok",
            "passed": true,
            "reason": "코드 스니펫 인코딩 정상"
          },
          {
            "check_type": "not_hallucination",
            "passed": true,
            "reason": "diff에서 해당 변경 확인, 실제 존재하는 이슈"
          },
          {
            "check_type": "line_range_valid",
            "passed": true,
            "reason": "라인 범위가 diff 범위 내에 있음"
          }
        ],
        "confidence": 0.95
      },
      "inline_position": {
        "diff_line_start": 5,
        "diff_line_end": 7,
        "file_line_start": 143,
        "file_line_end": 145,
        "position_type": "added",
        "position_confidence": 0.98
      }
    }
  ],
  "filtered_issues": [
    {
      "original_issue": {
        "id": "ISS-003",
        "title": "변수명이 명확하지 않음",
        "line_start": 200,
        "line_end": 200
      },
      "filter_reason": "지적된 라인(200)이 변경되지 않은 영역이며, diff 범위를 벗어남",
      "failed_checks": ["change_exists", "line_range_valid"]
    }
  ],
  "validation_summary": {
    "total_issues": 3,
    "valid_issues": 2,
    "filtered_issues": 1,
    "filter_rate": 0.33,
    "common_filter_reasons": ["변경되지 않은 영역 지적"]
  }
}
```

---

## 3. 처리 로직

### 3.1 개선된 분리 구조

현재 단일 호출을 **2단계**로 분리:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Review Validator Pipeline                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Stage A: Issue Validation (이슈 검증)                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ 각 Issue에 대해:                                           │ │
│  │ 1. 변경점 실제성 확인 (diff에 존재하는가?)                 │ │
│  │ 2. 설명 정확성 검토 (코드와 설명이 일치하는가?)            │ │
│  │ 3. 제안 코드 유효성 (문법/논리적으로 올바른가?)            │ │
│  │ 4. 인코딩 검증 (깨진 문자 없는가?)                         │ │
│  │ 5. 할루시네이션 탐지 (존재하지 않는 문제인가?)             │ │
│  │                                                            │ │
│  │ 출력: 각 이슈의 valid/invalid 상태 및 사유                 │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  Stage B: Line Position Extraction (라인 위치 추출)             │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ 유효한 Issue들에 대해:                                     │ │
│  │ 1. diff hunk 분석                                          │ │
│  │ 2. code_snippet과 diff 매칭                                │ │
│  │ 3. inline comment 위치 계산                                │ │
│  │    - diff 기준 라인 번호                                   │ │
│  │    - 원본 파일 기준 라인 번호                              │ │
│  │ 4. 위치 신뢰도 계산                                        │ │
│  │                                                            │ │
│  │ 출력: 정확한 inline position                               │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 검증 체크 상세

#### Check 1: change_exists (변경점 존재 확인)

```python
def check_change_exists(issue: Issue, file_diff: FileDiff) -> ValidationCheck:
    """이슈가 지적한 라인이 실제로 변경된 영역인지 확인"""

    # diff hunk 범위 추출
    changed_ranges = extract_changed_ranges(file_diff)

    # 이슈 라인이 변경 범위에 포함되는지
    issue_range = range(issue.line_start, issue.line_end + 1)

    for changed_range in changed_ranges:
        if ranges_overlap(issue_range, changed_range):
            return ValidationCheck(
                check_type="change_exists",
                passed=True,
                reason=f"{issue.line_start}-{issue.line_end} 라인이 변경 범위 내에 있음"
            )

    return ValidationCheck(
        check_type="change_exists",
        passed=False,
        reason=f"지적된 라인이 변경되지 않은 영역임"
    )
```

#### Check 2: description_accurate (설명 정확성)

```python
def check_description_accurate(
    issue: Issue,
    function_code: str,
    file_diff: FileDiff
) -> ValidationCheck:
    """설명이 실제 코드 변경과 일치하는지 확인"""

    # code_snippet이 실제 코드에 존재하는지
    if issue.code_snippet not in function_code:
        return ValidationCheck(
            check_type="description_accurate",
            passed=False,
            reason="code_snippet이 실제 코드에서 발견되지 않음"
        )

    # LLM을 통한 의미적 일치 확인 (필요시)
    # ...

    return ValidationCheck(
        check_type="description_accurate",
        passed=True,
        reason="설명과 코드 변경이 일치함"
    )
```

#### Check 3: suggestion_valid (제안 코드 유효성)

```python
def check_suggestion_valid(issue: Issue) -> ValidationCheck:
    """제안된 코드가 유효한지 확인"""

    if not issue.suggested_code:
        return ValidationCheck(
            check_type="suggestion_valid",
            passed=True,  # 제안이 없으면 통과
            reason="제안 코드 없음 (선택 사항)"
        )

    # 기본 문법 체크 (괄호 짝, 세미콜론 등)
    syntax_issues = check_basic_syntax(issue.suggested_code)
    if syntax_issues:
        return ValidationCheck(
            check_type="suggestion_valid",
            passed=False,
            reason=f"제안 코드 문법 오류: {syntax_issues}"
        )

    # 원본 코드와의 일관성
    if not is_consistent_with_original(issue.code_snippet, issue.suggested_code):
        return ValidationCheck(
            check_type="suggestion_valid",
            passed=False,
            reason="제안 코드가 원본과 일관성이 없음"
        )

    return ValidationCheck(
        check_type="suggestion_valid",
        passed=True,
        reason="제안 코드가 유효함"
    )
```

#### Check 4: encoding_ok (인코딩 검증)

```python
def check_encoding_ok(issue: Issue) -> ValidationCheck:
    """코드 스니펫의 인코딩이 정상인지 확인"""

    texts_to_check = [
        issue.code_snippet,
        issue.suggested_code,
        issue.description
    ]

    for text in texts_to_check:
        if text and has_encoding_issues(text):
            return ValidationCheck(
                check_type="encoding_ok",
                passed=False,
                reason="인코딩 깨짐 감지"
            )

    return ValidationCheck(
        check_type="encoding_ok",
        passed=True,
        reason="인코딩 정상"
    )

def has_encoding_issues(text: str) -> bool:
    """인코딩 문제 감지"""
    # 대체 문자, 깨진 UTF-8 시퀀스 등 확인
    suspicious_patterns = [
        '\ufffd',  # 대체 문자
        '�',       # 깨진 문자
        '\x00',    # null 바이트
    ]
    return any(p in text for p in suspicious_patterns)
```

#### Check 5: not_hallucination (할루시네이션 탐지)

```python
def check_not_hallucination(
    issue: Issue,
    file_diff: FileDiff,
    function_code: str
) -> ValidationCheck:
    """할루시네이션인지 확인"""

    # 1. 언급된 변수/함수가 실제로 존재하는지
    mentioned_identifiers = extract_identifiers(issue.description)
    code_identifiers = extract_identifiers(function_code)

    missing = mentioned_identifiers - code_identifiers
    if missing:
        return ValidationCheck(
            check_type="not_hallucination",
            passed=False,
            reason=f"코드에 없는 식별자 언급: {missing}"
        )

    # 2. 라인 번호가 파일 범위 내인지
    max_line = get_max_line_from_diff(file_diff)
    if issue.line_end > max_line:
        return ValidationCheck(
            check_type="not_hallucination",
            passed=False,
            reason=f"라인 번호가 파일 범위를 초과: {issue.line_end} > {max_line}"
        )

    # 3. code_snippet이 실제 diff에 존재하는지
    if not snippet_in_diff(issue.code_snippet, file_diff):
        return ValidationCheck(
            check_type="not_hallucination",
            passed=False,
            reason="code_snippet이 diff에서 발견되지 않음"
        )

    return ValidationCheck(
        check_type="not_hallucination",
        passed=True,
        reason="실제 존재하는 이슈로 확인됨"
    )
```

### 3.3 라인 위치 추출 로직

```python
def extract_inline_position(
    issue: Issue,
    file_diff: FileDiff
) -> InlinePosition:
    """inline comment를 위한 정확한 라인 위치 추출"""

    # 1. code_snippet으로 diff에서 정확한 위치 찾기
    diff_match = find_snippet_in_diff(issue.code_snippet, file_diff)

    if diff_match:
        return InlinePosition(
            diff_line_start=diff_match.diff_line,
            diff_line_end=diff_match.diff_line + diff_match.line_count - 1,
            file_line_start=diff_match.file_line,
            file_line_end=diff_match.file_line + diff_match.line_count - 1,
            position_type=diff_match.change_type,
            position_confidence=0.95
        )

    # 2. 라인 번호로 역추적
    hunk_match = find_line_in_hunks(issue.line_start, file_diff.hunks)

    if hunk_match:
        return InlinePosition(
            diff_line_start=hunk_match.diff_line,
            diff_line_end=hunk_match.diff_line + (issue.line_end - issue.line_start),
            file_line_start=issue.line_start,
            file_line_end=issue.line_end,
            position_type=hunk_match.change_type,
            position_confidence=0.7
        )

    # 3. 근사치 반환
    return InlinePosition(
        diff_line_start=0,
        diff_line_end=0,
        file_line_start=issue.line_start,
        file_line_end=issue.line_end,
        position_type="context",
        position_confidence=0.3
    )
```

---

## 4. 프롬프트 설계

### 4.1 Stage A: Issue Validation 프롬프트

**시스템 프롬프트**:
```
당신은 코드 리뷰 품질 검증 전문가입니다.
Code Reviewer가 생성한 리뷰 아이템을 검증하여 유효성을 판단합니다.

## 검증 항목

각 리뷰 아이템에 대해 다음을 확인하세요:

1. **변경점 존재 (change_exists)**
   - 지적된 라인이 실제로 변경된 영역인가?
   - diff에서 해당 변경을 찾을 수 있는가?

2. **설명 정확성 (description_accurate)**
   - 설명이 실제 코드 변경과 일치하는가?
   - code_snippet이 실제 코드에 존재하는가?

3. **제안 코드 유효성 (suggestion_valid)**
   - 제안된 코드가 문법적으로 올바른가?
   - 제안이 문제를 실제로 해결하는가?

4. **인코딩 정상 (encoding_ok)**
   - 깨진 문자나 이상한 인코딩이 없는가?

5. **할루시네이션 여부 (not_hallucination)**
   - 존재하지 않는 변수/함수를 언급하지 않았는가?
   - 없는 문제를 지적하지 않았는가?

6. **라인 범위 유효 (line_range_valid)**
   - 라인 번호가 파일/diff 범위 내에 있는가?

## 검증 규칙

- 하나라도 실패하면 해당 이슈는 is_valid: false
- 각 체크에 대해 passed와 구체적인 reason 제공
- 불확실한 경우 보수적으로 판단 (false 처리)
```

**사용자 프롬프트 템플릿**:
```
다음 코드 리뷰 결과의 유효성을 검증하세요.

## 컨텍스트

### PR Description
{pr_description}

### 변경 명세 분석
{refined_spec_json}

### 파일 Diff
```diff
{file_diff}
```

### 변경된 함수 코드
```c
{function_code}
```

## 검증 대상: 리뷰 아이템들

{review_items_json}

## 출력 형식

각 이슈에 대해 6가지 체크 결과를 JSON으로 출력하세요:

{validation_schema}
```

### 4.2 Stage B: Line Position 프롬프트

**시스템 프롬프트**:
```
당신은 diff 분석 및 라인 매핑 전문가입니다.
리뷰 아이템의 code_snippet을 기반으로 정확한 inline comment 위치를 계산합니다.

## 위치 계산 규칙

1. **diff_line**: unified diff 형식에서의 라인 번호 (헤더 제외)
2. **file_line**: 실제 파일에서의 라인 번호
3. **position_type**:
   - "added": 새로 추가된 라인
   - "modified": 수정된 라인 (- 와 + 조합)
   - "context": 변경되지 않은 컨텍스트 라인

## 매칭 우선순위

1. code_snippet의 정확한 문자열 매칭
2. 유사 패턴 매칭 (공백 무시)
3. 라인 번호 기반 역추적
```

**사용자 프롬프트 템플릿**:
```
다음 유효한 리뷰 아이템들의 inline comment 위치를 계산하세요.

## Diff
```diff
{file_diff}
```

## 리뷰 아이템들

{validated_issues_json}

## 출력 형식

각 이슈에 대해 inline_position을 계산하세요:

{position_schema}
```

---

## 5. 최적화 전략

### 5.1 검증 단계 분리 vs 결합

| 전략 | 호출 수 | 장점 | 단점 |
|------|---------|------|------|
| 완전 분리 | 2회 | 역할 명확, 품질 높음 | 호출 증가 |
| 결합 | 1회 | 호출 적음 | 현재 문제 유지 |
| **하이브리드** | 1-2회 | 균형 | 조건부 로직 필요 |

**권장: 하이브리드 전략**

```python
def validate_review(input_data: ValidatorInput) -> ValidatedReviewResult:
    # 빠른 사전 필터링 (LLM 없이)
    pre_filtered = pre_filter_issues(input_data.review_result.issues)

    if len(pre_filtered.valid) == 0:
        # 모든 이슈가 사전 필터링됨 - LLM 호출 불필요
        return create_empty_result(pre_filtered.filtered)

    if len(pre_filtered.valid) <= 3:
        # 적은 이슈 - 1회 결합 호출
        return single_call_validation(input_data, pre_filtered.valid)
    else:
        # 많은 이슈 - 2단계 분리
        validation_result = stage_a_validation(input_data, pre_filtered.valid)
        position_result = stage_b_position(input_data, validation_result.valid_issues)
        return merge_results(validation_result, position_result)
```

### 5.2 사전 필터링 (LLM 없이)

LLM 호출 전에 명백한 문제 필터링:

```python
def pre_filter_issues(issues: List[Issue], file_diff: FileDiff) -> PreFilterResult:
    """LLM 없이 수행 가능한 빠른 필터링"""
    valid = []
    filtered = []

    diff_range = get_diff_line_range(file_diff)
    diff_content = file_diff.raw_diff

    for issue in issues:
        # 1. 라인 범위 체크
        if issue.line_end < diff_range.start or issue.line_start > diff_range.end:
            filtered.append((issue, "라인이 diff 범위 밖"))
            continue

        # 2. 인코딩 체크
        if has_encoding_issues(issue.code_snippet or ""):
            filtered.append((issue, "인코딩 깨짐"))
            continue

        # 3. 빈 필드 체크
        if not issue.title or not issue.description:
            filtered.append((issue, "필수 필드 누락"))
            continue

        valid.append(issue)

    return PreFilterResult(valid=valid, filtered=filtered)
```

---

## 6. 에러 처리

### 6.1 검증 실패 시 처리

| 상황 | 처리 |
|------|------|
| 전체 검증 실패 | 원본 리뷰 결과 그대로 반환 + 경고 |
| 일부 이슈 검증 실패 | 실패한 이슈만 필터링 |
| 라인 위치 추출 실패 | 원본 라인 번호 사용 + 낮은 confidence |

### 6.2 fallback 전략

```python
def validate_with_fallback(input_data: ValidatorInput) -> ValidatedReviewResult:
    try:
        return full_validation(input_data)
    except LLMTimeoutError:
        # LLM 타임아웃 시 사전 필터링 결과만 반환
        return pre_filter_only(input_data)
    except ValidationParseError:
        # 응답 파싱 실패 시 원본 반환
        return passthrough_original(input_data)
```

---

## 7. 테스트 케이스

### 7.1 검증 테스트

| 케이스 | 입력 | 예상 결과 |
|--------|------|-----------|
| 유효한 이슈 | 실제 변경 지적 | is_valid: true, 모든 체크 통과 |
| 할루시네이션 | 없는 변수 언급 | is_valid: false, not_hallucination 실패 |
| 범위 초과 | diff 밖 라인 지적 | is_valid: false, line_range_valid 실패 |
| 인코딩 깨짐 | 깨진 문자 포함 | is_valid: false, encoding_ok 실패 |
| 잘못된 제안 | 문법 오류 코드 | is_valid: false, suggestion_valid 실패 |

### 7.2 라인 위치 테스트

| 케이스 | 예상 confidence |
|--------|-----------------|
| 정확한 snippet 매칭 | 0.95+ |
| 유사 패턴 매칭 | 0.7-0.9 |
| 라인 번호 역추적 | 0.5-0.7 |
| 매칭 실패 | 0.3 이하 |

---

## 8. 관련 문서

- 스키마: [schemas/internal/validated-review.schema.json](../../schemas/internal/validated-review.schema.json)
- Code Reviewer: [03-code-reviewer.md](03-code-reviewer.md)
- Summary Generator: [04-summary-generator.md](04-summary-generator.md)
