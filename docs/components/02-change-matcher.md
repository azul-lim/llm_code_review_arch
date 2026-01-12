# Change Matcher 컴포넌트

## 1. 개요

### 1.1 목적
PR Description에서 추출한 변경 명세와 실제 파일 변경점을 비교하여 일치 여부를 분석합니다.

### 1.2 위치
파이프라인의 두 번째 단계로, 파일당 1회 실행됩니다.

```
[RefinedSpec] ─┬─→ [Change Matcher] ──→ [MatchResult]
[FileDiff]    ─┘          │
                          └─→ Code Reviewer에 전달
```

---

## 2. 입출력 명세

### 2.1 입력 (Input)

```typescript
interface MatcherInput {
  refined_spec: RefinedSpec;     // Description Refiner 출력
  file_diff: FileDiff;           // 파일의 변경점
  file_name: string;             // 분석 대상 파일명
}

interface FileDiff {
  hunks: DiffHunk[];             // diff 청크 목록
  additions: number;             // 추가된 라인 수
  deletions: number;             // 삭제된 라인 수
}

interface DiffHunk {
  old_start: number;
  old_lines: number;
  new_start: number;
  new_lines: number;
  content: string;               // unified diff 형식
}
```

**입력 예시**:
```json
{
  "refined_spec": {
    "changes": [
      {
        "id": "CHG-001",
        "description": "nand_read() 함수에서 버퍼 해제 전 null 체크 추가",
        "expected_files": ["nand_read.c"],
        "change_type": "fix",
        "keywords": ["null check", "buffer", "free", "nand_read"]
      }
    ],
    "context": "버퍼 관리 메모리 누수 수정"
  },
  "file_diff": {
    "hunks": [
      {
        "old_start": 142,
        "new_start": 142,
        "content": "@@ -142,6 +142,8 @@ int nand_read(...)\n     // read data\n+    if (buffer != NULL) {\n         free(buffer);\n+    }\n"
      }
    ]
  },
  "file_name": "nand_read.c"
}
```

### 2.2 출력 (Output)

```typescript
interface MatchResult {
  file_name: string;
  matches: Match[];              // 매칭된 항목들
  unmatched_specs: string[];     // 이 파일에서 찾지 못한 명세 ID
  unexpected_changes: UnexpectedChange[];  // 명세에 없는 변경
  analysis_notes: string;        // 분석 메모
}

interface Match {
  spec_id: string;               // 매칭된 명세 ID
  line_start: number;            // 변경 시작 라인
  line_end: number;              // 변경 끝 라인
  match_type: MatchType;         // 매칭 유형
  confidence: number;            // 신뢰도 (0.0 ~ 1.0)
  evidence: string;              // 매칭 근거
}

type MatchType = "exact" | "partial" | "inferred";

interface UnexpectedChange {
  line_start: number;
  line_end: number;
  description: string;           // 변경 내용 설명
  potential_intent: string;      // 추론된 의도
}
```

**출력 예시**:
```json
{
  "file_name": "nand_read.c",
  "matches": [
    {
      "spec_id": "CHG-001",
      "line_start": 143,
      "line_end": 145,
      "match_type": "exact",
      "confidence": 0.95,
      "evidence": "nand_read 함수 내 free() 호출 전 null 체크 조건문 추가됨"
    }
  ],
  "unmatched_specs": [],
  "unexpected_changes": [],
  "analysis_notes": "CHG-001 명세가 정확히 구현됨"
}
```

---

## 3. 처리 로직

### 3.1 처리 단계

```
┌─────────────────────────────────────────────────────────────┐
│                      Change Matcher                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Step 1: 파일 관련성 판단                                   │
│    - 파일명이 expected_files에 포함되는지 확인              │
│    - 키워드가 diff에 존재하는지 확인                        │
│    - 관련 없으면 빈 결과 반환                               │
│                                                             │
│  Step 2: 변경점 분석                                        │
│    - diff 청크별로 변경 내용 파싱                           │
│    - 추가/삭제/수정 라인 식별                               │
│    - 변경된 함수/블록 범위 파악                             │
│                                                             │
│  Step 3: 명세-변경 매칭                                     │
│    - 각 ChangeItem에 대해 관련 변경 탐색                    │
│    - 키워드 매칭                                            │
│    - 의미적 유사성 평가                                     │
│                                                             │
│  Step 4: 결과 생성                                          │
│    - 매칭 결과 구조화                                       │
│    - 미매칭 명세 식별                                       │
│    - 예상치 못한 변경 기록                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 매칭 유형 정의

| 유형 | 정의 | 신뢰도 범위 |
|------|------|-------------|
| exact | 명세와 변경이 정확히 일치 | 0.9 ~ 1.0 |
| partial | 명세의 일부만 반영됨 | 0.5 ~ 0.89 |
| inferred | 직접적 언급 없으나 관련성 추론 | 0.3 ~ 0.49 |

### 3.3 신뢰도 계산 기준

```python
def calculate_confidence(spec: ChangeItem, diff_content: str) -> float:
    score = 0.0

    # 1. 파일명 일치 (+0.2)
    if file_name in spec.expected_files:
        score += 0.2

    # 2. 키워드 매칭 (+0.1 per keyword, max 0.4)
    matched_keywords = count_keyword_matches(spec.keywords, diff_content)
    score += min(matched_keywords * 0.1, 0.4)

    # 3. 변경 유형 일치 (+0.2)
    if infer_change_type(diff_content) == spec.change_type:
        score += 0.2

    # 4. 의미적 유사성 (+0.2)
    semantic_score = calculate_semantic_similarity(spec.description, diff_content)
    score += semantic_score * 0.2

    return min(score, 1.0)
```

---

## 4. 프롬프트 설계

### 4.1 시스템 프롬프트

```
당신은 코드 리뷰 시스템의 변경점 분석 전문가입니다.
PR Description에서 추출된 변경 명세와 실제 코드 변경을 비교하여 일치 여부를 분석합니다.

역할:
1. 변경 명세가 이 파일의 변경에 반영되었는지 판단합니다.
2. 각 매칭에 대해 구체적인 근거를 제시합니다.
3. 명세에 없는 예상치 못한 변경을 식별합니다.
4. 분석 결과를 지정된 JSON 형식으로 출력합니다.

분석 원칙:
- 불확실한 경우 낮은 신뢰도로 표시합니다.
- 한 변경이 여러 명세와 관련될 수 있습니다.
- 파일명이 expected_files에 없어도 키워드로 관련성을 찾을 수 있습니다.
```

### 4.2 사용자 프롬프트 템플릿

```
다음 변경 명세와 파일 변경점을 비교 분석하세요.

## 변경 명세
{refined_spec_json}

## 분석 대상 파일
파일명: {file_name}

## 파일 변경점 (Diff)
```diff
{file_diff}
```

## 분석 요청
1. 각 명세 항목(CHG-XXX)이 이 파일의 변경에 반영되었는지 확인하세요.
2. 반영된 경우 정확한 라인 번호와 매칭 근거를 제시하세요.
3. 명세에 없는 변경이 있다면 식별하세요.

## 출력 형식
{output_schema}
```

---

## 5. 특수 케이스 처리

### 5.1 파일 관련성 없음

파일이 어떤 명세와도 관련 없는 경우:

```json
{
  "file_name": "unrelated_file.c",
  "matches": [],
  "unmatched_specs": ["CHG-001", "CHG-002"],
  "unexpected_changes": [
    {
      "line_start": 10,
      "line_end": 15,
      "description": "주석 수정",
      "potential_intent": "코드 가독성 개선"
    }
  ],
  "analysis_notes": "이 파일은 명세된 변경과 직접적인 관련이 없습니다."
}
```

### 5.2 부분 매칭

명세가 여러 파일에 걸쳐 구현된 경우:

```json
{
  "matches": [
    {
      "spec_id": "CHG-002",
      "match_type": "partial",
      "confidence": 0.6,
      "evidence": "참조 카운트 증가 로직만 구현됨, 감소 로직은 다른 파일에 있을 것으로 추정"
    }
  ]
}
```

### 5.3 중복 변경

하나의 변경이 여러 명세와 관련된 경우:

```json
{
  "matches": [
    {
      "spec_id": "CHG-001",
      "line_start": 50,
      "line_end": 55,
      "confidence": 0.8
    },
    {
      "spec_id": "CHG-003",
      "line_start": 50,
      "line_end": 55,
      "confidence": 0.7
    }
  ]
}
```

---

## 6. 성능 최적화

### 6.1 조기 종료 조건

```python
def should_skip_matching(spec: ChangeItem, file_name: str, diff: str) -> bool:
    # 파일명이 명시적으로 제외된 경우
    if spec.expected_files and file_name not in spec.expected_files:
        # 키워드도 없으면 스킵
        if not any(kw in diff for kw in spec.keywords):
            return True
    return False
```

### 6.2 배치 처리

작은 파일들은 배치로 묶어서 처리:

```python
def batch_match(files: List[FileInfo], refined_spec: RefinedSpec) -> List[MatchResult]:
    # 작은 파일들 그룹화 (총 토큰 수 기준)
    batches = group_by_token_count(files, max_tokens=2000)

    results = []
    for batch in batches:
        batch_result = process_batch(batch, refined_spec)
        results.extend(batch_result)

    return results
```

---

## 7. 테스트 케이스

### 7.1 정상 케이스

| 케이스 | 입력 특징 | 예상 결과 |
|--------|-----------|-----------|
| 완전 일치 | 명세와 변경이 1:1 대응 | 모든 match_type이 "exact" |
| 부분 일치 | 명세의 일부만 이 파일에서 구현 | match_type이 "partial" |
| 다중 매칭 | 하나의 변경이 여러 명세와 관련 | 동일 라인에 여러 Match |

### 7.2 엣지 케이스

| 케이스 | 처리 방법 |
|--------|-----------|
| 빈 Diff | 모든 명세가 unmatched_specs에 포함 |
| 바이너리 파일 | 스킵, 분석 불가 메모 |
| 매우 큰 Diff | 청크 단위로 분할 처리 |

---

## 8. 관련 문서

- 스키마: [schemas/internal/match-result.schema.json](../../schemas/internal/match-result.schema.json)
- 프롬프트 템플릿: [prompts/templates/matcher.md](../prompts/templates/matcher.md)
- 예시: [examples/matcher/](../../examples/)
