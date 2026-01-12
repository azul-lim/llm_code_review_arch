# Amazon CodeGuru Reviewer 아키텍처 분석

> AWS의 ML 기반 코드 리뷰 서비스
> 참고: 2025년 11월 신규 가입 종료 예정

## 1. 개요

| 항목 | 내용 |
|------|------|
| 유형 | SaaS (AWS 통합) |
| 기술 | ML (비-LLM: 로지스틱 회귀 + CNN) |
| 언어 | Java, Python |
| 플랫폼 | GitHub, Bitbucket, CodeCommit, S3 |
| 특화 | AWS Best Practices, 보안 분석 |

**중요**: CodeGuru는 LLM이 아닌 전통적 ML 기반. 하지만 설계 패턴은 참고 가치 있음.

## 2. 아키텍처

### 2.1 ML 모델 구조

```
┌─────────────────────────────────────────────────────────────┐
│              Amazon CodeGuru Reviewer ML Pipeline           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Training Phase                           │
│                                                             │
│  Data Sources:                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ - Amazon Internal Code Base (millions of lines)      │   │
│  │ - Open Source Projects                               │   │
│  │ - AWS Documentation                                  │   │
│  │ - Code Changes with Quality Improvements             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Training Techniques:                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 1. Rule Mining: AWS API 사용 패턴에서 규칙 추출      │   │
│  │ 2. Locality Sensitive Models: 유사 코드 패턴 탐지   │   │
│  │ 3. Logistic Regression: 이슈 분류                   │   │
│  │ 4. CNN (Convolutional Neural Networks): 패턴 인식   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   Inference Phase                           │
│                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐   │
│  │ Code Input  │ → │ Static      │ → │ ML Detectors    │   │
│  │ (PR / Repo) │   │ Analysis    │   │                 │   │
│  └─────────────┘   │ (AST, CFG)  │   │ - Resource Leak │   │
│                    └─────────────┘   │ - Security Vuln │   │
│                                      │ - AWS Practices │   │
│                                      │ - Code Smells   │   │
│                                      └────────┬────────┘   │
│                                               │            │
│                                               ▼            │
│                                      ┌─────────────────┐   │
│                                      │ Recommendations │   │
│                                      │ with Confidence │   │
│                                      └─────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 탐지 카테고리

| 카테고리 | 탐지 항목 | 기법 |
|----------|----------|------|
| **Resource Leaks** | 미닫힌 연결, 스트림 | Data Flow + CNN |
| **Security** | SQL Injection, XSS | Automated Reasoning |
| **AWS Best Practices** | API 사용 패턴 | Rule Mining |
| **Code Quality** | 중복, 복잡도 | Pattern Matching |
| **Concurrency** | Race Conditions | Static Analysis |

### 2.3 Automated Reasoning (보안 분석)

```
┌─────────────────────────────────────────────────────────────┐
│               Automated Reasoning for Security              │
└─────────────────────────────────────────────────────────────┘

Source → Sink 분석:

┌──────────┐                              ┌──────────┐
│  Source  │   ────── Data Flow ──────→   │   Sink   │
│          │                              │          │
│ - User   │   함수 경계를 넘어           │ - SQL    │
│   Input  │   추적 및 분석               │ - File   │
│ - HTTP   │                              │ - Exec   │
│   Params │                              │ - Output │
└──────────┘                              └──────────┘

특징:
- 다중 함수 경계 추적
- Taint Analysis
- Symbolic Execution
- 99% 검증 정확도 (AWS 주장)
```

## 3. 피드백 학습 시스템

### 3.1 Continuous Learning

```python
# 개념적 구조
class FeedbackLearning:
    """사용자 피드백 기반 모델 개선"""

    def process_feedback(self, recommendation_id: str, feedback: str):
        """
        feedback: "ThumbsUp" | "ThumbsDown"
        """
        if feedback == "ThumbsUp":
            # 해당 패턴의 confidence 증가
            self.reinforce_pattern(recommendation_id)
        else:
            # 해당 패턴 재학습 또는 필터링
            self.suppress_pattern(recommendation_id)

    def iterative_improvement(self):
        """주기적 모델 업데이트"""
        # 1. 피드백 데이터 수집
        # 2. 라벨로 활용하여 재학습
        # 3. False Positive 감소
        pass
```

### 3.2 Confidence Score

```
각 추천에 confidence score 부여:

High Confidence (80%+):
  - 명확한 버그 패턴
  - 검증된 보안 취약점
  - 문서화된 AWS anti-pattern

Medium Confidence (50-80%):
  - 잠재적 이슈
  - 컨텍스트 의존적

Low Confidence (<50%):
  - 제안 수준
  - 스타일 관련
```

## 4. 우리 설계에 대한 시사점

### 4.1 적용 가능한 패턴

| CodeGuru 패턴 | 우리 설계 적용 |
|--------------|---------------|
| **Confidence Score** | Validator의 신뢰도 점수 |
| **Feedback Loop** | 리뷰 결과 피드백 수집 |
| **Source-Sink Analysis** | 보안 이슈 탐지 강화 |
| **Category-based Detection** | 이슈 유형별 분류 체계 |

### 4.2 Confidence Score 적용

```python
# 우리 Validator에 적용
class ValidatedReviewItem:
    issue: ReviewIssue
    confidence: float  # 0.0 - 1.0
    confidence_factors: List[str]

    @classmethod
    def calculate_confidence(cls, issue: ReviewIssue, validations: dict) -> float:
        """신뢰도 계산"""
        score = 0.5  # 기본값

        # 검증 통과 시 가산
        if validations['change_exists']:
            score += 0.15
        if validations['code_pattern_verified']:
            score += 0.15
        if validations['not_hallucination']:
            score += 0.15

        # 정적 분석 도구 일치 시 가산
        if validations['static_analyzer_agrees']:
            score += 0.20

        return min(score, 1.0)
```

### 4.3 피드백 시스템 설계

```python
# 장기적 품질 개선을 위한 피드백 수집
class ReviewFeedbackCollector:
    """리뷰 결과 피드백 수집"""

    def collect_feedback(self, review_id: str, feedback: dict):
        """
        feedback = {
            'issue_id': str,
            'action': 'accepted' | 'rejected' | 'modified',
            'reason': str (optional),
            'actual_fix': str (optional)
        }
        """
        # 1. 피드백 저장
        self.store_feedback(review_id, feedback)

        # 2. 프롬프트 개선을 위한 분석
        if feedback['action'] == 'rejected':
            self.analyze_rejection(review_id, feedback)

    def generate_improvement_report(self) -> dict:
        """주기적 개선 리포트"""
        return {
            'false_positive_rate': self.calculate_fp_rate(),
            'common_rejection_reasons': self.get_rejection_patterns(),
            'suggested_prompt_changes': self.suggest_improvements()
        }
```

### 주의사항

1. **서비스 종료 예정**: 2025년 11월 신규 가입 불가
2. **언어 제한**: Java, Python만 지원
3. **비-LLM**: 맥락 이해 능력 제한적

## 5. 참조

- [Amazon CodeGuru Reviewer Documentation](https://docs.aws.amazon.com/codeguru/latest/reviewer-ug/welcome.html)
- [How CodeGuru Reviewer Works](https://docs.aws.amazon.com/codeguru/latest/reviewer-ug/how-codeguru-reviewer-works.html)
- [CodeGuru Reviewer FAQs](https://aws.amazon.com/codeguru/reviewer/faqs/)
