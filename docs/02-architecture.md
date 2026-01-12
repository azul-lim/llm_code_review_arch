# 상세 아키텍처 설계

## 1. 시스템 아키텍처

### 1.1 레이어 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                       Interface Layer                          │
│                   (입출력 데이터 변환)                           │
├─────────────────────────────────────────────────────────────────┤
│                       Pipeline Layer                           │
│                   (컴포넌트 오케스트레이션)                       │
├─────────────────────────────────────────────────────────────────┤
│                      Component Layer                           │
│  ┌─────────┬─────────┬──────────┬───────────┬─────────┐       │
│  │ Refiner │ Matcher │ Reviewer │ Validator │ Summary │       │
│  └─────────┴─────────┴──────────┴───────────┴─────────┘       │
├─────────────────────────────────────────────────────────────────┤
│                        LLM Layer                               │
│                   (LLM 호출 추상화)                              │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 레이어 책임

| 레이어 | 책임 | 주요 클래스/모듈 |
|--------|------|------------------|
| Interface | 외부 시스템과의 데이터 변환 | `InputAdapter`, `OutputAdapter` |
| Pipeline | 컴포넌트 실행 순서 관리, 병렬 처리 | `ReviewPipeline`, `PipelineExecutor` |
| Component | 개별 분석 로직 수행 | `DescriptionRefiner`, `ChangeMatcher`, `CodeReviewer`, `ReviewValidator`, `SummaryGenerator` |
| LLM | LLM API 호출, 재시도, 오류 처리 | `LLMClient`, `PromptBuilder` |

---

## 2. 컴포넌트 상세

### 2.1 컴포넌트 인터페이스

모든 컴포넌트는 동일한 인터페이스를 구현:

```python
from abc import ABC, abstractmethod
from typing import TypeVar, Generic

InputT = TypeVar('InputT')
OutputT = TypeVar('OutputT')

class Component(ABC, Generic[InputT, OutputT]):
    """모든 파이프라인 컴포넌트의 기본 인터페이스"""

    @abstractmethod
    def process(self, input_data: InputT) -> OutputT:
        """입력을 처리하여 출력 반환"""
        pass

    @abstractmethod
    def validate_input(self, input_data: InputT) -> bool:
        """입력 데이터 유효성 검증"""
        pass

    @abstractmethod
    def get_prompt(self, input_data: InputT) -> str:
        """LLM에 전달할 프롬프트 생성"""
        pass
```

### 2.2 컴포넌트별 구조

#### Description Refiner

```
┌────────────────────────────────────────┐
│         Description Refiner            │
├────────────────────────────────────────┤
│ Input:                                 │
│   - pr_description: string             │
│                                        │
│ Process:                               │
│   1. 자유 형식 텍스트 파싱             │
│   2. 변경 항목 추출                    │
│   3. 구조화된 형식으로 변환            │
│                                        │
│ Output:                                │
│   - refined_spec: RefinedSpec          │
│     - changes: ChangeItem[]            │
│     - context: string                  │
└────────────────────────────────────────┘
```

#### Change Matcher

```
┌────────────────────────────────────────┐
│           Change Matcher               │
├────────────────────────────────────────┤
│ Input:                                 │
│   - refined_spec: RefinedSpec          │
│   - file_diff: FileDiff                │
│   - file_name: string                  │
│                                        │
│ Process:                               │
│   1. 명세의 각 변경 항목 확인          │
│   2. 파일 내 관련 변경점 탐색          │
│   3. 일치 여부 및 위치 매핑            │
│                                        │
│ Output:                                │
│   - match_result: MatchResult          │
│     - matches: Match[]                 │
│     - unmatched_specs: string[]        │
│     - unexpected_changes: string[]     │
└────────────────────────────────────────┘
```

#### Code Reviewer

```
┌────────────────────────────────────────┐
│           Code Reviewer                │
├────────────────────────────────────────┤
│ Input:                                 │
│   - match_result: MatchResult          │
│   - function_code: string              │
│   - file_diff: FileDiff                │
│   - pr_full_diff: string (optional)    │
│                                        │
│ Process:                               │
│   1. 변경점 해석 생성                  │
│   2. COT 기반 코드 분석                │
│      - 동작 시뮬레이션                 │
│      - 버그 패턴 탐색                  │
│      - 구조적 문제 검토                │
│   3. 리뷰 아이템 생성                  │
│                                        │
│ Output:                                │
│   - review_result: ReviewResult        │
│     - interpretation: string           │
│     - issues: Issue[]                  │
│     - suggestions: Suggestion[]        │
└────────────────────────────────────────┘
```

#### Review Validator

```
┌────────────────────────────────────────┐
│          Review Validator              │
├────────────────────────────────────────┤
│ Input:                                 │
│   - review_result: ReviewResult        │
│   - pr_description: string             │
│   - refined_spec: RefinedSpec          │
│   - file_diff: FileDiff                │
│   - function_code: string              │
│                                        │
│ Process:                               │
│   Stage A: 이슈 검증                   │
│     1. 변경점 실제성 확인              │
│     2. 설명 정확성 검토                │
│     3. 제안 코드 유효성                │
│     4. 인코딩 검증                     │
│     5. 할루시네이션 탐지               │
│   Stage B: 라인 위치 추출              │
│     1. inline comment 위치 계산        │
│     2. diff/파일 라인 매핑             │
│                                        │
│ Output:                                │
│   - validated_result: ValidatedReview  │
│     - validated_issues: Issue[]        │
│     - filtered_issues: Issue[]         │
│     - inline_positions: Position[]     │
└────────────────────────────────────────┘
```

#### Summary Generator

```
┌────────────────────────────────────────┐
│         Summary Generator              │
├────────────────────────────────────────┤
│ Input:                                 │
│   - validated_reviews: ValidatedReview[]│
│   - pr_description: string             │
│                                        │
│ Process:                               │
│   1. 전체 리뷰 결과 집계               │
│   2. 주요 이슈 우선순위 정렬           │
│   3. PR 수준 요약 생성                 │
│                                        │
│ Output:                                │
│   - summary: PRSummary                 │
│     - overall_assessment: string       │
│     - key_issues: Issue[]              │
│     - statistics: Stats                │
└────────────────────────────────────────┘
```

---

## 3. 파이프라인 설계

### 3.1 파이프라인 실행 흐름

```python
class ReviewPipeline:
    """코드 리뷰 파이프라인 오케스트레이터"""

    def __init__(self):
        self.refiner = DescriptionRefiner()
        self.matcher = ChangeMatcher()
        self.reviewer = CodeReviewer()
        self.validator = ReviewValidator()
        self.summarizer = SummaryGenerator()

    def execute(self, pr_input: PRInput) -> PROutput:
        # Step 1: PR Description 정제 (1회)
        refined_spec = self.refiner.process(pr_input.description)

        # Step 2-4: 파일별 처리 (병렬 가능)
        validated_reviews = []
        for file in pr_input.files:
            # Step 2: 변경 매칭
            match_result = self.matcher.process(
                MatcherInput(refined_spec, file.diff, file.name)
            )

            # Step 3: 코드 리뷰
            review = self.reviewer.process(
                ReviewerInput(match_result, file.function_code, file.diff)
            )

            # Step 4: 리뷰 검증 (할루시네이션 필터링 + 라인 위치 추출)
            validated = self.validator.process(
                ValidatorInput(
                    review_result=review,
                    pr_description=pr_input.description,
                    refined_spec=refined_spec,
                    file_diff=file.diff,
                    function_code=file.function_code
                )
            )
            validated_reviews.append(validated)

        # Step 5: 요약 생성 (1회)
        summary = self.summarizer.process(
            SummarizerInput(validated_reviews, pr_input.description)
        )

        return PROutput(validated_reviews, summary)
```

### 3.2 병렬 처리 설계

```
                    ┌─────────────────┐
                    │ Description     │
                    │ Refiner         │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ File 1   │   │ File 2   │   │ File N   │
        │ Pipeline │   │ Pipeline │   │ Pipeline │
        │          │   │          │   │          │
        │ Matcher  │   │ Matcher  │   │ Matcher  │
        │    ↓     │   │    ↓     │   │    ↓     │
        │ Reviewer │   │ Reviewer │   │ Reviewer │
        │    ↓     │   │    ↓     │   │    ↓     │
        │Validator │   │Validator │   │Validator │
        └────┬─────┘   └────┬─────┘   └────┬─────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Summary         │
                    │ Generator       │
                    └─────────────────┘
```

### 3.3 오류 처리 전략

| 오류 유형 | 처리 전략 | 재시도 |
|-----------|-----------|--------|
| LLM 응답 파싱 실패 | 재시도 후 기본값 | 최대 3회 |
| LLM API 오류 | 지수 백오프 재시도 | 최대 5회 |
| 입력 검증 실패 | 즉시 실패, 에러 반환 | 없음 |
| 타임아웃 | 재시도 후 스킵 | 최대 2회 |

---

## 4. 데이터 모델

### 4.1 핵심 데이터 타입

```python
from dataclasses import dataclass
from typing import List, Optional
from enum import Enum

class IssueSeverity(Enum):
    CRITICAL = "critical"    # 반드시 수정 필요
    MAJOR = "major"          # 수정 권장
    MINOR = "minor"          # 개선 제안
    INFO = "info"            # 정보성 코멘트

class IssueType(Enum):
    BUG = "bug"                      # 버그/오류
    LOGIC_ERROR = "logic_error"      # 논리 오류
    PERFORMANCE = "performance"       # 성능 문제
    SECURITY = "security"            # 보안 문제
    STYLE = "style"                  # 코딩 스타일
    STRUCTURE = "structure"          # 구조적 문제

@dataclass
class ChangeItem:
    """PR Description에서 추출한 변경 항목"""
    id: str
    description: str
    expected_files: List[str]
    change_type: str  # add, modify, delete, fix

@dataclass
class RefinedSpec:
    """정제된 PR 명세"""
    changes: List[ChangeItem]
    context: str
    raw_description: str

@dataclass
class Match:
    """명세와 실제 변경의 매칭"""
    spec_id: str
    file_name: str
    line_start: int
    line_end: int
    match_type: str  # exact, partial, none
    confidence: float

@dataclass
class MatchResult:
    """파일별 매칭 결과"""
    file_name: str
    matches: List[Match]
    unmatched_specs: List[str]
    unexpected_changes: List[str]

@dataclass
class Issue:
    """발견된 이슈"""
    id: str
    type: IssueType
    severity: IssueSeverity
    file_name: str
    line_start: int
    line_end: int
    title: str
    description: str
    code_snippet: str
    suggestion: Optional[str]
    suggested_code: Optional[str]

@dataclass
class ReviewResult:
    """파일별 리뷰 결과"""
    file_name: str
    interpretation: str
    issues: List[Issue]
    reasoning_trace: str  # COT 추론 과정

@dataclass
class PRSummary:
    """PR 전체 요약"""
    overall_assessment: str
    key_issues: List[Issue]
    total_files: int
    files_with_issues: int
    issue_counts: dict  # severity별 개수
```

### 4.2 스키마 정의 위치

상세 JSON Schema는 `schemas/` 디렉토리에 정의:
- `schemas/input/` - 입력 데이터 스키마
- `schemas/internal/` - 내부 데이터 스키마
- `schemas/output/` - 출력 데이터 스키마

---

## 5. LLM 레이어 설계

### 5.1 LLM Client 인터페이스

```python
class LLMClient:
    """LLM API 호출 추상화"""

    def __init__(self, model: str, config: LLMConfig):
        self.model = model
        self.config = config

    def generate(
        self,
        prompt: str,
        max_tokens: int = 4096,
        temperature: float = 0.1,
        response_format: Optional[dict] = None
    ) -> LLMResponse:
        """LLM 호출 및 응답 반환"""
        pass

    def generate_json(
        self,
        prompt: str,
        schema: dict,
        max_tokens: int = 4096
    ) -> dict:
        """JSON 형식 응답 생성"""
        pass
```

### 5.2 프롬프트 빌더

```python
class PromptBuilder:
    """컴포넌트별 프롬프트 생성"""

    def __init__(self, template_dir: str):
        self.templates = self._load_templates(template_dir)

    def build(
        self,
        component: str,
        context: dict,
        few_shot_examples: Optional[List[dict]] = None
    ) -> str:
        """템플릿 기반 프롬프트 생성"""
        template = self.templates[component]
        prompt = template.render(context)

        if few_shot_examples:
            prompt = self._add_examples(prompt, few_shot_examples)

        return prompt
```

---

## 6. 설정 및 구성

### 6.1 파이프라인 설정

```yaml
# config/pipeline.yaml
pipeline:
  parallel_files: true
  max_parallel_workers: 4

components:
  refiner:
    enabled: true
    cache_enabled: true

  matcher:
    enabled: true
    confidence_threshold: 0.7

  reviewer:
    enabled: true
    include_full_diff: auto  # auto, always, never
    cot_enabled: true

  validator:
    enabled: true
    pre_filter_enabled: true
    hallucination_check: true
    encoding_check: true
    line_position_extraction: true

  summarizer:
    enabled: true
    max_key_issues: 5

llm:
  model: "qwen3"
  max_tokens: 4096
  temperature: 0.1
  retry_count: 3
  timeout_seconds: 60
```

---

## 7. 다음 단계

- [components/01-description-refiner.md](components/01-description-refiner.md): Description Refiner 상세 설계
- [components/02-change-matcher.md](components/02-change-matcher.md): Change Matcher 상세 설계
- [components/03-code-reviewer.md](components/03-code-reviewer.md): Code Reviewer 상세 설계
- [components/04-summary-generator.md](components/04-summary-generator.md): Summary Generator 상세 설계
- [components/05-review-validator.md](components/05-review-validator.md): Review Validator 상세 설계
