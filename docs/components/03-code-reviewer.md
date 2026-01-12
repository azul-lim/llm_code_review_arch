# Code Reviewer 컴포넌트

## 1. 개요

### 1.1 목적
코드 변경점을 분석하여 버그, 문제점을 탐지하고 개선안을 제시합니다. 파이프라인의 핵심 컴포넌트입니다.

### 1.2 위치
파이프라인의 세 번째 단계로, 파일당 1회 실행됩니다.

```
[MatchResult]    ─┬─→ [Code Reviewer] ──→ [ReviewResult]
[FunctionCode]   ─┤
[FileDiff]       ─┤
[PRFullDiff]     ─┘ (optional)
```

---

## 2. 입출력 명세

### 2.1 입력 (Input)

```typescript
interface ReviewerInput {
  match_result: MatchResult;       // Change Matcher 출력
  function_code: string;           // 변경점 포함 함수의 전체 코드
  file_diff: FileDiff;             // 파일 변경점
  file_name: string;               // 파일명
  pr_full_diff?: string;           // PR 전체 diff (선택, 컨텍스트 크기에 따라)
}
```

### 2.2 출력 (Output)

```typescript
interface ReviewResult {
  file_name: string;
  interpretation: ChangeInterpretation;  // 변경점 해석
  issues: Issue[];                       // 발견된 이슈
  reasoning_trace: ReasoningTrace;       // COT 추론 과정
}

interface ChangeInterpretation {
  summary: string;                 // 변경 요약
  purpose: string;                 // 변경 목적
  impact: string;                  // 영향 범위
  details: InterpretationDetail[];
}

interface InterpretationDetail {
  line_range: string;              // "142-145"
  description: string;             // 이 부분이 하는 일
  before_behavior: string;         // 변경 전 동작
  after_behavior: string;          // 변경 후 동작
}

interface Issue {
  id: string;                      // "ISS-001"
  type: IssueType;
  severity: IssueSeverity;
  line_start: number;
  line_end: number;
  title: string;                   // 간결한 제목
  description: string;             // 상세 설명
  code_snippet: string;            // 문제 코드
  suggestion?: string;             // 개선 방향
  suggested_code?: string;         // 개선 코드
  related_spec?: string;           // 관련 명세 ID
}

type IssueType =
  | "bug"              // 명확한 버그
  | "logic_error"      // 논리 오류
  | "null_safety"      // null/포인터 안전성
  | "memory"           // 메모리 관련 (leak, overflow 등)
  | "concurrency"      // 동시성 문제
  | "performance"      // 성능 문제
  | "security"         // 보안 취약점
  | "error_handling"   // 에러 처리 누락
  | "style"            // 코딩 스타일
  | "structure";       // 구조적 문제

type IssueSeverity =
  | "critical"         // 반드시 수정 필요
  | "major"            // 수정 권장
  | "minor"            // 개선 제안
  | "info";            // 정보성

interface ReasoningTrace {
  steps: ReasoningStep[];
  conclusion: string;
}

interface ReasoningStep {
  step_number: number;
  action: string;                  // 분석 행동
  observation: string;             // 관찰 결과
  inference: string;               // 추론
}
```

**출력 예시**:
```json
{
  "file_name": "nand_read.c",
  "interpretation": {
    "summary": "nand_read 함수에 null 체크 추가",
    "purpose": "메모리 해제 시 null 포인터 접근 방지",
    "impact": "nand_read 함수의 안정성 향상",
    "details": [
      {
        "line_range": "143-145",
        "description": "buffer 해제 전 null 체크 조건문",
        "before_behavior": "buffer가 null이어도 free() 호출",
        "after_behavior": "buffer가 null이 아닐 때만 free() 호출"
      }
    ]
  },
  "issues": [
    {
      "id": "ISS-001",
      "type": "memory",
      "severity": "major",
      "line_start": 143,
      "line_end": 145,
      "title": "free 후 buffer 포인터 미초기화",
      "description": "buffer를 free한 후 NULL로 설정하지 않아 dangling pointer 위험이 있습니다.",
      "code_snippet": "if (buffer != NULL) {\n    free(buffer);\n}",
      "suggestion": "free 후 buffer = NULL 추가",
      "suggested_code": "if (buffer != NULL) {\n    free(buffer);\n    buffer = NULL;\n}",
      "related_spec": "CHG-001"
    }
  ],
  "reasoning_trace": {
    "steps": [
      {
        "step_number": 1,
        "action": "변경점 식별",
        "observation": "142-145 라인에 null 체크 조건문 추가됨",
        "inference": "메모리 안전성 개선 의도로 보임"
      },
      {
        "step_number": 2,
        "action": "변경 후 코드 시뮬레이션",
        "observation": "buffer가 NULL일 때 free 호출 방지됨",
        "inference": "null 포인터 free 버그는 해결됨"
      },
      {
        "step_number": 3,
        "action": "추가 문제점 탐색",
        "observation": "free 후 buffer 포인터가 그대로 남아있음",
        "inference": "이후 코드에서 dangling pointer 접근 가능성 있음"
      }
    ],
    "conclusion": "null 체크 추가는 적절하나, dangling pointer 방지를 위해 추가 조치 권장"
  }
}
```

---

## 3. 처리 로직

### 3.1 처리 단계

```
┌─────────────────────────────────────────────────────────────────┐
│                       Code Reviewer                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Phase 1: 변경점 해석 (Interpretation)                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ 1. 변경 범위 파악                                          │ │
│  │ 2. 변경 전/후 동작 비교                                    │ │
│  │ 3. 개발자 의도 추론                                        │ │
│  │ 4. 해석 문서화                                             │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  Phase 2: COT 기반 분석 (Chain-of-Thought)                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Step 1: 코드 동작 시뮬레이션                               │ │
│  │   - 정상 경로 실행 추적                                    │ │
│  │   - 엣지 케이스 실행 추적                                  │ │
│  │   - 에러 경로 실행 추적                                    │ │
│  │                                                            │ │
│  │ Step 2: 버그 패턴 탐색                                     │ │
│  │   - Null/포인터 안전성                                     │ │
│  │   - 메모리 관리 (leak, overflow, use-after-free)          │ │
│  │   - 경계 조건                                              │ │
│  │   - 에러 처리 누락                                         │ │
│  │   - 동시성 문제                                            │ │
│  │                                                            │ │
│  │ Step 3: 구조적 문제 검토                                   │ │
│  │   - 코드 복잡도                                            │ │
│  │   - 중복 코드                                              │ │
│  │   - 일관성 위반                                            │ │
│  │                                                            │ │
│  │ Step 4: 다른 변경과의 연계 분석                            │ │
│  │   - PR 내 다른 파일과의 상호작용                           │ │
│  │   - 인터페이스 일관성                                      │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  Phase 3: 이슈 생성 및 제안                                     │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ 1. 발견된 문제 구조화                                      │ │
│  │ 2. 심각도 평가                                             │ │
│  │ 3. 개선 코드 생성                                          │ │
│  │ 4. 결과 검증                                               │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 버그 패턴 체크리스트

Storage Firmware 특화 체크리스트:

| 카테고리 | 체크 항목 |
|----------|-----------|
| **Null Safety** | - 포인터 사용 전 null 체크<br>- 함수 반환값 null 체크<br>- 배열 인덱스 유효성 |
| **Memory** | - malloc/free 짝<br>- buffer overflow<br>- use-after-free<br>- double-free<br>- memory leak |
| **Concurrency** | - lock/unlock 짝<br>- race condition<br>- deadlock 가능성 |
| **Error Handling** | - 에러 코드 전파<br>- 리소스 정리<br>- 실패 경로 처리 |
| **Firmware Specific** | - 레지스터 접근 순서<br>- 타이밍 제약<br>- 인터럽트 안전성 |

### 3.3 심각도 판단 기준

| 심각도 | 기준 | 예시 |
|--------|------|------|
| critical | 시스템 crash, 데이터 손실, 보안 취약점 | null 포인터 역참조, buffer overflow |
| major | 기능 오동작, 메모리 누수 | 에러 처리 누락, 리소스 미해제 |
| minor | 잠재적 문제, 코드 품질 저하 | 불필요한 연산, 스타일 위반 |
| info | 정보 제공, 문서화 제안 | 주석 부족, 명명 규칙 제안 |

---

## 4. 프롬프트 설계

### 4.1 시스템 프롬프트

```
당신은 Storage Firmware 전문 코드 리뷰어입니다.
변경된 코드를 분석하여 버그와 문제점을 찾고 개선안을 제시합니다.

## 분석 원칙

1. **변경점 해석 우선**
   - 먼저 변경의 의도와 영향을 이해합니다.
   - 개발자가 해결하려는 문제를 파악합니다.

2. **단계적 추론 (Chain-of-Thought)**
   - 각 분석 단계를 명시적으로 기록합니다.
   - 관찰 → 추론 → 결론 순서로 진행합니다.

3. **시뮬레이션 기반 분석**
   - 코드의 실행 경로를 머릿속으로 추적합니다.
   - 정상 경로와 에러 경로 모두 고려합니다.
   - 엣지 케이스(null, 0, 최대값 등)를 테스트합니다.

4. **정확한 위치 지정**
   - 문제가 있는 정확한 라인 번호를 명시합니다.
   - 관련 코드 스니펫을 포함합니다.

5. **실행 가능한 제안**
   - 문제 설명에 그치지 않고 구체적인 수정 코드를 제시합니다.
   - 제안된 코드는 컴파일 가능해야 합니다.

## 분석 대상 버그 패턴

### Memory Safety
- Null pointer dereference
- Buffer overflow/underflow
- Use-after-free
- Double-free
- Memory leak

### Logic Errors
- Off-by-one errors
- Incorrect conditionals
- Missing break in switch
- Incorrect operator (=, ==, ===)

### Error Handling
- Unchecked return values
- Missing error propagation
- Resource cleanup on error path

### Concurrency (if applicable)
- Race conditions
- Missing synchronization
- Deadlock potential
```

### 4.2 사용자 프롬프트 템플릿

```
다음 코드 변경을 분석하세요.

## 파일 정보
- 파일명: {file_name}
- 변경 명세 매칭 결과:
{match_result_json}

## 변경점 (Diff)
```diff
{file_diff}
```

## 변경된 함수 전체 코드
```c
{function_code}
```

{#if pr_full_diff}
## PR 전체 변경 컨텍스트
```diff
{pr_full_diff}
```
{/if}

## 분석 요청

### Phase 1: 변경점 해석
변경의 목적과 영향을 설명하세요.

### Phase 2: 단계적 분석 (COT)
다음 단계를 명시적으로 수행하세요:

1. **코드 시뮬레이션**: 변경된 코드의 실행을 추적하세요.
   - 정상 입력일 때
   - null/빈 값일 때
   - 경계값일 때
   - 에러 발생 시

2. **버그 패턴 탐색**: 위 체크리스트를 기준으로 확인하세요.

3. **구조 검토**: 코드 품질 문제를 확인하세요.

4. **연계 분석**: 다른 변경과의 상호작용을 확인하세요.

### Phase 3: 이슈 및 제안
발견된 문제에 대해 구체적인 수정 코드와 함께 제시하세요.

## 출력 형식
{output_schema}
```

---

## 5. COT 상세 설계

### 5.1 추론 단계 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    COT 추론 구조                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Step N]                                                   │
│  ├── Action: 수행하는 분석 행동                              │
│  │   예: "143라인의 조건문을 분석"                           │
│  │                                                          │
│  ├── Observation: 관찰된 사실                                │
│  │   예: "buffer가 NULL인지 확인 후 free 호출"              │
│  │                                                          │
│  └── Inference: 추론 결과                                    │
│      예: "NULL 체크는 추가되었으나, free 후 포인터 처리 없음" │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 시뮬레이션 시나리오

각 변경에 대해 다음 시나리오를 시뮬레이션:

| 시나리오 | 입력 조건 | 검증 포인트 |
|----------|-----------|-------------|
| Happy Path | 정상 입력값 | 기대 동작 수행 |
| Null Input | null 포인터/빈 배열 | null 체크 존재 |
| Boundary | 0, MAX, 경계값 | 오버플로우 없음 |
| Error Path | 실패 반환 | 리소스 정리, 에러 전파 |
| Partial | 중간 실패 | 일관된 상태 유지 |

---

## 6. Full Diff 포함 전략

### 6.1 포함 조건

```python
def should_include_full_diff(
    file_diff_tokens: int,
    function_code_tokens: int,
    full_diff_tokens: int,
    context_limit: int
) -> str:
    base_tokens = file_diff_tokens + function_code_tokens
    remaining = context_limit - base_tokens - PROMPT_OVERHEAD

    if remaining >= full_diff_tokens:
        return "full"          # 전체 포함
    elif remaining >= full_diff_tokens * 0.3:
        return "summary"       # 요약 포함
    else:
        return "none"          # 제외
```

### 6.2 Full Diff 요약 전략

전체 포함이 불가능할 때:

```python
def summarize_full_diff(full_diff: str, current_file: str) -> str:
    """현재 파일과 관련된 부분만 추출"""
    related_sections = []

    for file_diff in parse_diff(full_diff):
        if file_diff.file == current_file:
            continue  # 현재 파일은 이미 있음

        # 관련성 판단
        if has_shared_functions(file_diff, current_file):
            related_sections.append(summarize_section(file_diff))
        elif has_shared_types(file_diff, current_file):
            related_sections.append(summarize_section(file_diff))

    return "\n".join(related_sections)
```

---

## 7. 에러 처리

### 7.1 입력 검증

| 검증 항목 | 실패 시 처리 |
|-----------|--------------|
| 빈 function_code | diff만으로 분석, 제한적 결과 |
| 파싱 불가 diff | 원본 텍스트로 분석 시도 |
| 토큰 초과 | function_code 우선, diff 잘라냄 |

### 7.2 출력 검증

```python
def validate_review_result(result: ReviewResult) -> bool:
    # interpretation 필수
    if not result.interpretation.summary:
        return False

    # reasoning_trace 필수
    if len(result.reasoning_trace.steps) == 0:
        return False

    # issue 검증
    for issue in result.issues:
        if not issue.id or not issue.title:
            return False
        if issue.line_start > issue.line_end:
            return False

    return True
```

---

## 8. 테스트 케이스

### 8.1 버그 탐지 케이스

| 버그 유형 | 입력 코드 특징 | 예상 출력 |
|-----------|----------------|-----------|
| Null deref | 체크 없이 포인터 사용 | critical 이슈 |
| Memory leak | malloc 후 free 누락 | major 이슈 |
| Buffer overflow | 배열 범위 초과 접근 | critical 이슈 |
| Error 미처리 | 반환값 무시 | major 이슈 |

### 8.2 False Positive 방지

| 상황 | 올바른 처리 |
|------|-------------|
| 이미 외부에서 null 체크됨 | 중복 체크 요구 안 함 |
| 정적 분석 한계 | 확신 없으면 info로 |
| 의도적 패턴 | 컨텍스트 고려 |

---

## 9. 관련 문서

- 스키마: [schemas/output/review-result.schema.json](../../schemas/output/review-result.schema.json)
- 프롬프트 템플릿: [prompts/templates/reviewer.md](../prompts/templates/reviewer.md)
- 예시: [examples/reviewer/](../../examples/)
