# Chain-of-Thought 개선

## 1. 현재 문제

### 1.1 암묵적 COT

현재 프롬프트는 추론을 암묵적으로 기대:

```
현재 프롬프트 (예시):
"코드를 시뮬레이션하고 버그 가능성을 탐색하세요"
```

**문제점**:
- LLM이 자체적으로 추론 경로 결정
- 일관성 없는 분석 결과
- 중요 단계 누락 가능
- 추론 과정 확인 불가

### 1.2 예시: 동일 코드, 다른 분석

```c
// 분석 대상 코드
if (ptr != NULL) {
    free(ptr);
}
```

**시도 1 결과**: "null 체크가 적절히 수행됨"
**시도 2 결과**: "free 후 dangling pointer 위험"
**시도 3 결과**: "메모리 안전성 개선됨, 추가 검토 불필요"

→ 동일 코드에 대해 다른 분석 결과

---

## 2. 개선 목표

| 목표 | 측정 방법 |
|------|-----------|
| 분석 일관성 | 동일 입력 → 유사 결과 |
| 완전성 | 필수 분석 단계 모두 수행 |
| 추적 가능성 | 결론에 대한 근거 확인 가능 |
| 품질 | 버그 탐지율 향상 |

---

## 3. 명시적 COT 구조

### 3.1 추론 단계 정의

```
┌─────────────────────────────────────────────────────────────────┐
│                    명시적 COT 구조                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: 변경점 식별 (Identification)                           │
│  ├── 어떤 라인이 변경되었는가?                                  │
│  ├── 변경 유형은? (추가/수정/삭제)                              │
│  └── 변경된 코드의 역할은?                                      │
│                                                                 │
│  Step 2: 의도 파악 (Intent)                                     │
│  ├── 개발자가 해결하려는 문제는?                                │
│  ├── 명세와 일치하는가?                                         │
│  └── 변경의 예상 효과는?                                        │
│                                                                 │
│  Step 3: 동작 시뮬레이션 (Simulation)                           │
│  ├── 정상 경로: 유효한 입력일 때                                │
│  ├── Null 경로: null/빈 값일 때                                 │
│  ├── 경계 경로: 경계값일 때                                     │
│  └── 에러 경로: 실패 시                                         │
│                                                                 │
│  Step 4: 문제 탐색 (Problem Detection)                          │
│  ├── 시뮬레이션에서 문제 발견?                                  │
│  ├── 버그 패턴 체크리스트 검토                                  │
│  └── 구조적 문제 확인                                           │
│                                                                 │
│  Step 5: 결론 도출 (Conclusion)                                 │
│  ├── 발견된 문제 요약                                           │
│  ├── 각 문제의 심각도                                           │
│  └── 개선 방안                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 단계별 출력 형식

```typescript
interface ReasoningStep {
  step_number: number;
  step_name: string;
  action: string;           // 수행한 분석 행동
  observation: string;      // 관찰된 사실
  inference: string;        // 추론 결과
  issues_found?: string[];  // 이 단계에서 발견된 문제
}

interface ReasoningTrace {
  steps: ReasoningStep[];
  conclusion: string;
  confidence: number;       // 분석 신뢰도
}
```

---

## 4. 시뮬레이션 프레임워크

### 4.1 시뮬레이션 시나리오

각 변경에 대해 필수 시뮬레이션 시나리오:

| 시나리오 | 입력 조건 | 검증 포인트 |
|----------|-----------|-------------|
| **Happy Path** | 정상 유효 값 | 기대 동작 수행 |
| **Null Input** | null 포인터 | null 체크 존재 |
| **Empty Input** | 빈 배열/문자열 | 빈 입력 처리 |
| **Boundary Min** | 0, INT_MIN | 언더플로우 없음 |
| **Boundary Max** | MAX, INT_MAX | 오버플로우 없음 |
| **Error Return** | 실패 반환값 | 에러 처리 적절 |
| **Partial Failure** | 중간 실패 | 일관성 유지 |

### 4.2 시뮬레이션 프롬프트

```markdown
## Step 3: 동작 시뮬레이션

다음 시나리오별로 변경된 코드의 동작을 추적하세요.

### Scenario 1: Happy Path
- 입력: 유효한 정상 값
- 코드를 한 줄씩 실행하며 상태 변화를 추적하세요.
- 예상 결과와 실제 동작이 일치하는지 확인하세요.

### Scenario 2: Null Input
- 입력: null 포인터 또는 빈 값
- 코드가 이 경우를 어떻게 처리하는지 추적하세요.
- null 역참조가 발생하는지 확인하세요.

### Scenario 3: Boundary Values
- 입력: 0, -1, MAX_VALUE 등 경계값
- 오버플로우/언더플로우 가능성을 확인하세요.
- 배열 인덱스 범위를 확인하세요.

### Scenario 4: Error Path
- 이전 함수 호출이 실패를 반환한 경우
- 에러가 적절히 처리/전파되는지 확인하세요.
- 리소스가 정리되는지 확인하세요.
```

---

## 5. 버그 패턴 체크리스트

### 5.1 Storage Firmware 특화 체크리스트

```markdown
## Step 4: 버그 패턴 검토

다음 체크리스트를 순서대로 확인하고, 각 항목의 결과를 기록하세요.

### Memory Safety
□ [ ] 포인터 사용 전 null 체크
□ [ ] malloc 반환값 확인
□ [ ] free 후 포인터 NULL 설정
□ [ ] buffer 크기 확인 후 접근
□ [ ] 배열 인덱스 범위 검증

### Resource Management
□ [ ] malloc/free 짝 확인
□ [ ] lock/unlock 짝 확인
□ [ ] open/close 짝 확인
□ [ ] 에러 경로에서 리소스 해제

### Error Handling
□ [ ] 함수 반환값 확인
□ [ ] 에러 코드 전파
□ [ ] 실패 시 상태 일관성

### Logic
□ [ ] 조건문 경계 (< vs <=)
□ [ ] 루프 종료 조건
□ [ ] switch-case break
□ [ ] 연산자 우선순위

### Concurrency (해당 시)
□ [ ] 공유 자원 동기화
□ [ ] deadlock 가능성
□ [ ] race condition
```

### 5.2 체크리스트 출력 형식

```json
{
  "checklist_results": {
    "memory_safety": {
      "null_check_before_use": {
        "status": "pass",
        "evidence": "143라인에 null 체크 존재"
      },
      "ptr_null_after_free": {
        "status": "fail",
        "evidence": "145라인에서 free 후 ptr 미초기화",
        "issue_id": "ISS-001"
      }
    },
    "resource_management": {
      "malloc_free_pair": {
        "status": "pass",
        "evidence": "malloc(120) - free(145) 짝 확인"
      }
    }
  }
}
```

---

## 6. 프롬프트 구조화

### 6.1 COT 유도 시스템 프롬프트

```markdown
당신은 Storage Firmware 전문 코드 리뷰어입니다.

## 분석 방법

반드시 다음 5단계를 순서대로 수행하고, 각 단계의 결과를 `reasoning_trace`에 기록하세요.

### Step 1: 변경점 식별
변경된 코드를 식별하고 그 역할을 파악합니다.
- 어떤 라인이 변경되었나요?
- 추가/수정/삭제 중 무엇인가요?
- 변경된 코드는 어떤 기능을 수행하나요?

### Step 2: 의도 파악
개발자의 의도를 추론합니다.
- 어떤 문제를 해결하려 했나요?
- 명세(MatchResult)와 일치하나요?
- 변경의 예상 효과는 무엇인가요?

### Step 3: 동작 시뮬레이션
다음 시나리오별로 코드 실행을 추적합니다.
- Happy Path: 정상 입력
- Null Input: null/빈 값
- Boundary: 경계값
- Error: 실패 경로

### Step 4: 버그 패턴 검토
체크리스트를 순서대로 확인합니다.
- Memory Safety 항목
- Resource Management 항목
- Error Handling 항목
- Logic 항목

### Step 5: 결론 도출
발견된 문제를 정리합니다.
- 각 문제의 심각도
- 구체적인 개선 코드

## 출력 규칙

1. 각 단계를 건너뛰지 마세요.
2. 관찰된 사실과 추론을 구분하세요.
3. 문제가 없는 경우에도 "문제 없음 확인"을 기록하세요.
4. 불확실한 경우 확신도를 낮게 표시하세요.
```

### 6.2 단계별 출력 예시

```json
{
  "reasoning_trace": {
    "steps": [
      {
        "step_number": 1,
        "step_name": "변경점 식별",
        "action": "diff에서 변경된 라인 분석",
        "observation": "142-145라인에 if 조건문 추가됨. buffer != NULL 체크 후 free() 호출",
        "inference": "null 포인터 free 방지를 위한 가드 조건 추가"
      },
      {
        "step_number": 2,
        "step_name": "의도 파악",
        "action": "명세(CHG-001)와 비교",
        "observation": "명세: 'nand_read()에서 버퍼 해제 전 null 체크 추가'",
        "inference": "명세와 정확히 일치. 메모리 안전성 개선 의도"
      },
      {
        "step_number": 3,
        "step_name": "동작 시뮬레이션",
        "action": "4가지 시나리오 실행 추적",
        "observation": "Happy Path: 정상 동작. Null Input: free 스킵됨(정상). Boundary: 해당없음. Error: 리턴 전 free 호출됨",
        "inference": "null 체크는 정상 동작. 단, free 후 buffer 상태 미정"
      },
      {
        "step_number": 4,
        "step_name": "버그 패턴 검토",
        "action": "Memory Safety 체크리스트 확인",
        "observation": "null_check: PASS, malloc_check: PASS, ptr_null_after_free: FAIL - 145라인에서 free 후 buffer 포인터 그대로",
        "inference": "dangling pointer 위험 존재",
        "issues_found": ["free 후 buffer = NULL 누락"]
      },
      {
        "step_number": 5,
        "step_name": "결론 도출",
        "action": "발견 문제 정리",
        "observation": "1개 문제 발견: dangling pointer",
        "inference": "major 수준 이슈. free 후 buffer = NULL 추가 필요"
      }
    ],
    "conclusion": "null 체크 추가는 적절하나, free 후 포인터 초기화가 누락되어 dangling pointer 위험 존재",
    "confidence": 0.9
  }
}
```

---

## 7. 품질 보장 메커니즘

### 7.1 단계 완전성 검증

```python
def validate_reasoning_completeness(trace: ReasoningTrace) -> List[str]:
    """모든 필수 단계가 수행되었는지 검증"""
    required_steps = [
        "변경점 식별",
        "의도 파악",
        "동작 시뮬레이션",
        "버그 패턴 검토",
        "결론 도출"
    ]

    missing = []
    completed_steps = {s.step_name for s in trace.steps}

    for required in required_steps:
        if required not in completed_steps:
            missing.append(required)

    return missing
```

### 7.2 시뮬레이션 완전성 검증

```python
def validate_simulation_coverage(step: ReasoningStep) -> List[str]:
    """시뮬레이션 단계에서 모든 시나리오가 수행되었는지 검증"""
    if step.step_name != "동작 시뮬레이션":
        return []

    required_scenarios = ["Happy Path", "Null Input", "Boundary", "Error"]
    observation = step.observation.lower()

    missing = []
    for scenario in required_scenarios:
        if scenario.lower() not in observation:
            missing.append(scenario)

    return missing
```

### 7.3 재시도 전략

검증 실패 시:

```python
async def process_with_validation(input_data, max_retries=2):
    for attempt in range(max_retries + 1):
        result = await call_llm(input_data)

        # 완전성 검증
        missing_steps = validate_reasoning_completeness(result.reasoning_trace)

        if not missing_steps:
            return result

        if attempt < max_retries:
            # 누락된 단계를 강조하여 재시도
            input_data = add_emphasis(input_data, missing_steps)
        else:
            # 최대 재시도 후에도 실패
            return result.with_warning(f"누락된 단계: {missing_steps}")
```

---

## 8. 기대 효과

### 8.1 정량적 개선

| 지표 | 현재 (추정) | 개선 후 (목표) |
|------|-------------|----------------|
| 분석 일관성 | 60-70% | 90%+ |
| 필수 단계 완료율 | 70-80% | 95%+ |
| 버그 탐지율 | 기준선 | +20-30% |
| False Positive | 기준선 | -30% |

### 8.2 정성적 개선

- **추적 가능성**: 모든 결론에 대한 근거 확인 가능
- **디버깅 용이**: 문제 발생 시 어느 단계에서 실패했는지 파악
- **개선 용이**: 특정 단계의 프롬프트만 수정 가능
- **신뢰성**: 체계적 분석으로 리뷰 품질 신뢰도 향상

---

## 9. 구현 가이드

### 9.1 프롬프트 마이그레이션

1. 기존 프롬프트에 단계별 구조 추가
2. 출력 스키마에 `reasoning_trace` 필드 추가
3. 검증 로직 구현
4. 재시도 로직 구현

### 9.2 설정

```yaml
# config/cot.yaml
cot:
  required_steps:
    - "변경점 식별"
    - "의도 파악"
    - "동작 시뮬레이션"
    - "버그 패턴 검토"
    - "결론 도출"

  simulation_scenarios:
    - "Happy Path"
    - "Null Input"
    - "Boundary"
    - "Error"

  validation:
    enabled: true
    retry_on_incomplete: true
    max_retries: 2

  checklist:
    memory_safety: true
    resource_management: true
    error_handling: true
    logic: true
    concurrency: auto  # 동시성 코드일 때만
```

---

## 10. 관련 문서

- 프롬프트 전략: [prompt-strategy.md](../prompts/prompt-strategy.md)
- Task/Workflow 설계: [04-task-workflow-design.md](../04-task-workflow-design.md)
- 호출 최적화: [call-optimization.md](call-optimization.md)
