# Description Refiner 컴포넌트

## 1. 개요

### 1.1 목적
PR Description의 자유 형식 텍스트를 분석하여 구조화된 변경 명세로 변환합니다.

### 1.2 위치
파이프라인의 첫 번째 단계로, PR당 1회 실행됩니다.

```
[PR Description] ──→ [Description Refiner] ──→ [RefinedSpec]
                            │
                            └─→ 후속 컴포넌트에서 공유
```

---

## 2. 입출력 명세

### 2.1 입력 (Input)

```typescript
interface RefinerInput {
  pr_description: string;  // 원본 PR Description 텍스트
}
```

**입력 예시**:
```
Fix memory leak in buffer management

Changes:
- Added null check before freeing buffer in nand_read()
- Modified buffer allocation logic to track reference count
- Fixed double-free bug in error handling path

Also updated comments for clarity.
```

### 2.2 출력 (Output)

```typescript
interface RefinedSpec {
  changes: ChangeItem[];
  context: string;
  raw_description: string;
}

interface ChangeItem {
  id: string;                    // 고유 식별자 (예: "CHG-001")
  description: string;           // 변경 내용 설명
  expected_files: string[];      // 예상 관련 파일 (추론)
  change_type: ChangeType;       // 변경 유형
  keywords: string[];            // 핵심 키워드
}

type ChangeType = "add" | "modify" | "delete" | "fix" | "refactor";
```

**출력 예시**:
```json
{
  "changes": [
    {
      "id": "CHG-001",
      "description": "nand_read() 함수에서 버퍼 해제 전 null 체크 추가",
      "expected_files": ["nand_read.c", "nand.c"],
      "change_type": "fix",
      "keywords": ["null check", "buffer", "free", "nand_read"]
    },
    {
      "id": "CHG-002",
      "description": "버퍼 할당 로직에 참조 카운트 추적 기능 추가",
      "expected_files": ["buffer.c", "buffer.h"],
      "change_type": "modify",
      "keywords": ["buffer allocation", "reference count"]
    },
    {
      "id": "CHG-003",
      "description": "에러 처리 경로에서 double-free 버그 수정",
      "expected_files": [],
      "change_type": "fix",
      "keywords": ["double-free", "error handling"]
    }
  ],
  "context": "버퍼 관리의 메모리 누수 수정",
  "raw_description": "Fix memory leak in buffer management..."
}
```

---

## 3. 처리 로직

### 3.1 처리 단계

```
┌─────────────────────────────────────────────────────────────┐
│                    Description Refiner                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Step 1: 구조 분석                                          │
│    - 제목/본문 분리                                         │
│    - 리스트 항목 추출                                       │
│    - 섹션 구분                                              │
│                                                             │
│  Step 2: 변경 항목 추출                                     │
│    - 각 변경 사항 식별                                      │
│    - 변경 유형 분류                                         │
│    - 관련 파일명 추론                                       │
│                                                             │
│  Step 3: 키워드 추출                                        │
│    - 함수명, 변수명 식별                                    │
│    - 기술 용어 추출                                         │
│    - 검색용 키워드 생성                                     │
│                                                             │
│  Step 4: 구조화                                             │
│    - JSON 형식으로 변환                                     │
│    - ID 할당                                                │
│    - 컨텍스트 요약 생성                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 변경 유형 분류 기준

| 유형 | 판단 기준 | 키워드 예시 |
|------|-----------|-------------|
| add | 새로운 기능/코드 추가 | "add", "new", "implement", "create" |
| modify | 기존 코드 수정 | "modify", "change", "update", "adjust" |
| delete | 코드/기능 제거 | "remove", "delete", "drop", "deprecate" |
| fix | 버그 수정 | "fix", "resolve", "correct", "patch" |
| refactor | 구조 개선 (동작 유지) | "refactor", "restructure", "reorganize" |

### 3.3 파일명 추론 규칙

| 패턴 | 추론 방법 |
|------|-----------|
| 함수명 언급 | 함수명에서 파일명 유추 (예: `nand_read()` → `nand_read.c`, `nand.c`) |
| 모듈명 언급 | 모듈명을 파일명으로 (예: "buffer module" → `buffer.c`) |
| 명시적 언급 | 경로/파일명 직접 추출 |
| 추론 불가 | 빈 배열 반환 |

---

## 4. 프롬프트 설계

### 4.1 시스템 프롬프트

```
당신은 코드 리뷰 시스템의 PR Description 분석 전문가입니다.
주어진 PR Description을 분석하여 구조화된 변경 명세를 생성합니다.

규칙:
1. 각 변경 항목을 독립적으로 식별합니다.
2. 변경 유형을 정확히 분류합니다: add, modify, delete, fix, refactor
3. 파일명 추론 시 불확실하면 빈 배열을 반환합니다.
4. 키워드는 코드에서 검색 가능한 형태로 추출합니다.
5. 응답은 반드시 지정된 JSON 형식으로 출력합니다.
```

### 4.2 사용자 프롬프트 템플릿

```
다음 PR Description을 분석하여 구조화된 변경 명세를 생성하세요.

## PR Description
{pr_description}

## 출력 형식
다음 JSON 스키마에 맞춰 응답하세요:

{output_schema}

## 분석 시 주의사항
- Storage Firmware 코드베이스입니다.
- 함수명은 주로 snake_case를 사용합니다.
- 파일 확장자는 주로 .c, .h입니다.
```

---

## 5. 에러 처리

### 5.1 입력 검증

| 검증 항목 | 조건 | 처리 |
|-----------|------|------|
| 빈 입력 | `pr_description`가 비어있음 | 기본 RefinedSpec 반환 |
| 과도한 길이 | 토큰 수 초과 | 앞부분만 처리 + 경고 |
| 파싱 불가 | 구조 없는 텍스트 | 전체를 단일 변경 항목으로 처리 |

### 5.2 출력 검증

```python
def validate_output(result: RefinedSpec) -> bool:
    # 최소 1개의 변경 항목 필요
    if len(result.changes) == 0:
        return False

    # 각 변경 항목 검증
    for change in result.changes:
        if not change.id or not change.description:
            return False
        if change.change_type not in VALID_CHANGE_TYPES:
            return False

    return True
```

---

## 6. 캐싱 전략

동일 PR에 대해 여러 파일을 처리할 때 Description Refiner는 1회만 실행됩니다.

```python
class DescriptionRefinerWithCache:
    def __init__(self):
        self._cache = {}

    def process(self, input_data: RefinerInput) -> RefinedSpec:
        cache_key = hash(input_data.pr_description)

        if cache_key in self._cache:
            return self._cache[cache_key]

        result = self._do_process(input_data)
        self._cache[cache_key] = result
        return result
```

---

## 7. 테스트 케이스

### 7.1 정상 케이스

| 케이스 | 입력 특징 | 예상 출력 |
|--------|-----------|-----------|
| 구조화된 입력 | 리스트 형식의 변경 사항 | 각 항목이 개별 ChangeItem |
| 자유 형식 | 문장 형태 설명 | 문장에서 변경 항목 추출 |
| 혼합 형식 | 제목 + 리스트 + 설명 | 모든 정보 통합 |

### 7.2 엣지 케이스

| 케이스 | 처리 방법 |
|--------|-----------|
| 빈 Description | 기본 구조 반환, 빈 changes 배열 |
| 비영어 Description | 그대로 처리 (다국어 지원) |
| 코드 블록 포함 | 코드에서 파일명/함수명 추출 |

---

## 8. 관련 문서

- 스키마: [schemas/internal/refined-spec.schema.json](../../schemas/internal/refined-spec.schema.json)
- 프롬프트 템플릿: [prompts/templates/refiner.md](../prompts/templates/refiner.md)
- 예시: [examples/refiner/](../../examples/)
