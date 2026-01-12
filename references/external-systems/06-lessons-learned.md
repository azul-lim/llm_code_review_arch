# 외부 시스템 분석 - 우리 설계 반영 사항

> 외부 시스템 분석을 통해 도출된 개선 권장사항

## 1. Executive Summary

### 핵심 발견

| 발견 | 출처 | 영향도 |
|------|------|--------|
| **검증 에이전트 패턴** | CodeRabbit | ⭐⭐⭐⭐⭐ |
| **PR 압축 전략** | PR-Agent | ⭐⭐⭐⭐⭐ |
| **하이브리드 탐지** | Copilot | ⭐⭐⭐⭐ |
| **신뢰도 점수** | CodeGuru | ⭐⭐⭐⭐ |
| **JSON 스키마 출력** | PR-Agent | ⭐⭐⭐ |

### 즉시 적용 권장

```
우선순위 1: Validator 강화 (CodeRabbit 패턴)
우선순위 2: PR Compression 추가 (PR-Agent 패턴)
우선순위 3: Confidence Score 도입 (CodeGuru 패턴)
```

---

## 2. Validator 강화 (CodeRabbit 패턴 적용)

### 2.1 현재 vs 개선

```
현재 설계:
┌─────────────────────────────────────────────────────────────┐
│  Validator                                                  │
│  ┌──────────────┐    ┌──────────────────────┐              │
│  │ Stage A      │ →  │ Stage B              │              │
│  │ (이슈 검증)  │    │ (라인 추출)          │              │
│  │              │    │                      │              │
│  │ LLM 판단만   │    │ LLM 판단만           │              │
│  └──────────────┘    └──────────────────────┘              │
└─────────────────────────────────────────────────────────────┘

개선 설계 (CodeRabbit 패턴):
┌─────────────────────────────────────────────────────────────┐
│  Enhanced Validator                                         │
│                                                             │
│  ┌──────────────┐    ┌──────────────────────┐              │
│  │ Stage A      │    │ Verification Agent   │              │
│  │ (이슈 검증)  │ →  │ (실행 기반 검증) ★   │              │
│  │              │    │                      │              │
│  │ LLM 판단     │    │ - grep/ast-grep 실행 │              │
│  │              │    │ - 라인 존재 확인     │              │
│  │              │    │ - 패턴 매칭 검증     │              │
│  └──────────────┘    └──────────┬───────────┘              │
│                                 │                          │
│                      검증 성공  │  검증 실패               │
│                          ↓      │      ↓                   │
│                     ┌───────────┴───────────┐              │
│                     │                       │              │
│                     ▼                       ▼              │
│              ┌──────────────┐        ┌──────────────┐      │
│              │ Stage B      │        │ 필터링       │      │
│              │ (라인 추출)  │        │ (제외)       │      │
│              └──────────────┘        └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 검증 스크립트 생성

```python
# 새로 추가할 컴포넌트
class VerificationScriptGenerator:
    """리뷰 이슈에 대한 검증 스크립트 생성"""

    def generate(self, issue: ReviewIssue, file_content: str) -> str:
        """이슈 유형별 검증 스크립트 생성"""

        if issue.type == "missing_import":
            return self._verify_import(issue)
        elif issue.type == "unused_variable":
            return self._verify_unused(issue)
        elif issue.type == "null_check":
            return self._verify_null_check(issue)
        elif issue.type == "line_reference":
            return self._verify_line_exists(issue)
        else:
            return self._generic_verify(issue)

    def _verify_import(self, issue: ReviewIssue) -> str:
        """import 관련 검증"""
        return f'''
# 해당 모듈이 실제로 사용되는지 확인
grep -n "{issue.suggested_import}" {issue.file_path}
# 이미 import 되어 있는지 확인
grep -n "^import.*{issue.module_name}" {issue.file_path}
grep -n "^from.*{issue.module_name}.*import" {issue.file_path}
'''

    def _verify_line_exists(self, issue: ReviewIssue) -> str:
        """라인 참조 검증"""
        return f'''
# 해당 라인이 존재하고 내용이 일치하는지 확인
sed -n '{issue.line_start},{issue.line_end}p' {issue.file_path} | grep -q "{issue.code_snippet[:50]}"
echo $?
'''
```

### 2.3 샌드박스 실행

```python
class SandboxExecutor:
    """검증 스크립트 안전 실행"""

    def __init__(self, workspace: str):
        self.workspace = workspace
        self.timeout = 5  # 초

    def execute(self, script: str) -> VerificationResult:
        """샌드박스 환경에서 스크립트 실행"""
        try:
            # 제한된 환경에서 실행
            result = subprocess.run(
                ['sh', '-c', script],
                cwd=self.workspace,
                capture_output=True,
                timeout=self.timeout,
                text=True
            )
            return VerificationResult(
                success=result.returncode == 0,
                output=result.stdout,
                error=result.stderr
            )
        except subprocess.TimeoutExpired:
            return VerificationResult(
                success=False,
                error="Verification timeout"
            )
```

---

## 3. PR Compression 추가 (PR-Agent 패턴 적용)

### 3.1 필요성

```
현재 문제:
- 50+ 파일 PR → 모든 파일 처리 시도
- 컨텍스트 윈도우 초과 가능성
- 비효율적 토큰 사용

PR-Agent 해결책:
- 우선순위 기반 파일 선택
- 토큰 예산 내 적합화
- 중요 파일 우선 처리
```

### 3.2 구현 설계

```python
class PRCompressor:
    """대형 PR 압축 처리"""

    def __init__(self, config: CompressionConfig):
        self.token_budget = config.token_budget  # e.g., 200K for QWEN3
        self.priority_rules = config.priority_rules

    def compress(self, pr: PRInfo) -> CompressedPR:
        """PR 압축"""
        # 1. 파일 우선순위 계산
        ranked_files = self._rank_files(pr.files)

        # 2. 토큰 예산 배분
        selected_files = []
        used_tokens = 0
        skipped_files = []

        for file in ranked_files:
            file_tokens = self._count_tokens(file)

            if used_tokens + file_tokens <= self.token_budget * 0.8:
                selected_files.append(file)
                used_tokens += file_tokens
            else:
                # 파일 일부만 포함 또는 스킵
                if file_tokens > self.token_budget * 0.3:
                    # 대형 파일: 중요 부분만 추출
                    partial = self._extract_important_parts(file)
                    selected_files.append(partial)
                else:
                    skipped_files.append(file)

        return CompressedPR(
            files=selected_files,
            skipped=skipped_files,
            total_tokens=used_tokens
        )

    def _rank_files(self, files: List[FileInfo]) -> List[FileInfo]:
        """파일 중요도 랭킹"""
        def score(f: FileInfo) -> float:
            s = 0.0

            # 1. 경로 기반 점수
            if 'src/' in f.path or 'lib/' in f.path:
                s += 100
            elif 'test' in f.path.lower():
                s -= 30
            elif any(x in f.path for x in ['config', '.json', '.yaml']):
                s -= 20

            # 2. 변경량 점수
            s += min(f.additions + f.deletions, 200) * 0.5

            # 3. 확장자 점수
            if f.extension in ['.c', '.cpp', '.h']:
                s += 50  # Storage FW 핵심
            elif f.extension in ['.py', '.java']:
                s += 30

            # 4. PR Description과의 관련성 (있다면)
            if self._is_mentioned_in_description(f):
                s += 80

            return s

        return sorted(files, key=score, reverse=True)
```

### 3.3 파이프라인 통합

```
┌─────────────────────────────────────────────────────────────┐
│  현재 파이프라인:                                           │
│  PR → Refiner → [모든 파일] → Matcher → Reviewer → ...      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  개선 파이프라인:                                           │
│                                                             │
│  PR → Refiner → PR Compressor → [선택된 파일] → Matcher ... │
│                      │                                      │
│                      └→ [스킵 파일] → 간략 처리             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Confidence Score 도입 (CodeGuru 패턴 적용)

### 4.1 설계

```python
@dataclass
class ScoredReviewItem:
    """신뢰도 점수가 포함된 리뷰 아이템"""
    issue: ReviewIssue
    confidence: float  # 0.0 - 1.0
    confidence_breakdown: dict
    display_priority: str  # "high", "medium", "low"

class ConfidenceCalculator:
    """리뷰 아이템 신뢰도 계산"""

    # 가중치 설정
    WEIGHTS = {
        'verification_passed': 0.30,    # 검증 스크립트 통과
        'static_analyzer_agrees': 0.25, # 정적 분석 도구 일치
        'line_verified': 0.15,          # 라인 번호 검증됨
        'pattern_matched': 0.15,        # 코드 패턴 매칭
        'context_relevant': 0.15,       # 컨텍스트 관련성
    }

    def calculate(self, item: ReviewItem, validations: dict) -> float:
        score = 0.0
        breakdown = {}

        for key, weight in self.WEIGHTS.items():
            if validations.get(key, False):
                score += weight
                breakdown[key] = weight
            else:
                breakdown[key] = 0.0

        return ScoredReviewItem(
            issue=item,
            confidence=min(score, 1.0),
            confidence_breakdown=breakdown,
            display_priority=self._get_priority(score)
        )

    def _get_priority(self, score: float) -> str:
        if score >= 0.8:
            return "high"
        elif score >= 0.5:
            return "medium"
        else:
            return "low"
```

### 4.2 출력 활용

```python
class ReviewOutputFormatter:
    """신뢰도 기반 출력 포맷팅"""

    def format(self, items: List[ScoredReviewItem],
               min_confidence: float = 0.5) -> ReviewOutput:
        """신뢰도 기준으로 필터링 및 정렬"""

        # 1. 최소 신뢰도 필터링
        filtered = [i for i in items if i.confidence >= min_confidence]

        # 2. 우선순위별 그룹화
        high = [i for i in filtered if i.display_priority == "high"]
        medium = [i for i in filtered if i.display_priority == "medium"]
        low = [i for i in filtered if i.display_priority == "low"]

        # 3. 출력 구조화
        return ReviewOutput(
            critical_issues=high,      # 반드시 확인 필요
            suggested_issues=medium,   # 확인 권장
            optional_issues=low,       # 참고용
            filtered_count=len(items) - len(filtered)
        )
```

---

## 5. 하이브리드 탐지 고려 (Copilot 패턴)

### 5.1 정적 분석 도구 연동

```
장기적으로 고려:

┌─────────────────────────────────────────────────────────────┐
│  Static Analyzers (폐쇄망 사용 가능)                        │
│                                                             │
│  - cppcheck (C/C++)                                         │
│  - MISRA checker (임베디드)                                 │
│  - Custom rules (사내 규칙)                                 │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  LLM Review (컨텍스트 기반 분석)                            │
│                                                             │
│  Input:                                                     │
│  - Code diff                                                │
│  - Static analyzer results ★ (참고 정보로 전달)            │
│  - PR description                                           │
│                                                             │
│  Output:                                                    │
│  - LLM issues (정적 분석 결과 참조하여 중복 방지)           │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 프롬프트 통합

```python
# 정적 분석 결과를 LLM 프롬프트에 포함
REVIEW_PROMPT_WITH_STATIC = """
## 정적 분석 결과 (참고)
다음은 정적 분석 도구에서 발견한 이슈입니다:
{static_analysis_results}

## 지시사항
1. 위 정적 분석 결과와 중복되지 않는 이슈에 집중하세요
2. 정적 분석으로 발견하기 어려운 로직 오류, 설계 문제에 집중하세요
3. 정적 분석 결과가 맞다면 별도로 언급하지 마세요

## 코드 변경
{code_diff}
"""
```

---

## 6. 구현 우선순위

### 6.1 Phase 1: 즉시 적용 (높은 ROI)

| 항목 | 복잡도 | 효과 | 우선순위 |
|------|--------|------|----------|
| PR Compression | 중간 | 높음 | ⭐⭐⭐⭐⭐ |
| Confidence Score | 낮음 | 중간 | ⭐⭐⭐⭐ |
| JSON Schema 강화 | 낮음 | 중간 | ⭐⭐⭐⭐ |

### 6.2 Phase 2: 중기 적용 (Validator 강화)

| 항목 | 복잡도 | 효과 | 우선순위 |
|------|--------|------|----------|
| Verification Scripts | 중간 | 높음 | ⭐⭐⭐⭐⭐ |
| Sandbox Execution | 중간 | 중간 | ⭐⭐⭐⭐ |

### 6.3 Phase 3: 장기 고려 (인프라 필요)

| 항목 | 복잡도 | 효과 | 우선순위 |
|------|--------|------|----------|
| Static Analyzer 연동 | 높음 | 높음 | ⭐⭐⭐ |
| Codegraph 구축 | 높음 | 중간 | ⭐⭐ |
| Feedback Learning | 높음 | 중간 | ⭐⭐ |

---

## 7. 다음 단계

### 7.1 문서 업데이트 필요

1. `docs/components/05-review-validator.md` - Verification Agent 추가
2. `docs/improvements/call-optimization.md` - PR Compression 섹션 추가
3. `schemas/output/review-result.schema.json` - Confidence Score 필드 추가

### 7.2 설계 검토 필요

1. PR Compression이 기존 파이프라인에 미치는 영향
2. Verification Script의 실행 환경 (폐쇄망 제약)
3. Confidence Score 임계값 결정

---

## 8. 참조

- [01-coderabbit.md](01-coderabbit.md) - CodeRabbit 상세 분석
- [02-pr-agent.md](02-pr-agent.md) - PR-Agent 상세 분석
- [03-github-copilot.md](03-github-copilot.md) - GitHub Copilot 상세 분석
- [04-amazon-codeguru.md](04-amazon-codeguru.md) - Amazon CodeGuru 상세 분석
- [05-comparison.md](05-comparison.md) - 비교 분석
