# GitHub Copilot Code Review 아키텍처 분석

> GitHub 네이티브 AI 코드 리뷰
> 2024년 10월 Private Preview → 2025년 GA

## 1. 개요

| 항목 | 내용 |
|------|------|
| 유형 | SaaS (GitHub 통합) |
| LLM | GitHub AI (내부 모델) |
| 플랫폼 | GitHub 전용 |
| 특화 | CodeQL 연동, Agentic Tool Calling |
| 가격 | Pro $10/월, Business $19/월 |

## 2. 아키텍처

### 2.1 전체 구조

```
┌─────────────────────────────────────────────────────────────┐
│                GitHub Copilot Code Review                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      Trigger Layer                          │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Manual     │  │  Auto via   │  │  @copilot mention   │ │
│  │  Request    │  │  Repo Rules │  │  in PR comment      │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│              Hybrid Detection Engine ★                      │
│                                                             │
│  ┌──────────────────────┐  ┌────────────────────────────┐  │
│  │  Deterministic Tools │  │  LLM-based Analysis        │  │
│  │                      │  │                            │  │
│  │  - CodeQL (Security) │  │  - Pattern Recognition     │  │
│  │  - ESLint            │  │  - Contextual Issues       │  │
│  │  - Other Linters     │  │  - Improvement Suggestions │  │
│  │                      │  │                            │  │
│  │  → 높은 정확도       │  │  → 유연한 분석             │  │
│  │  → False Positive 낮음│  │  → 할루시네이션 가능성    │  │
│  └──────────────────────┘  └────────────────────────────┘  │
│                                                             │
│              ↓ 결과 통합 및 중복 제거 ↓                     │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│            Agentic Context Gathering (2025 New)             │
│                                                             │
│  Tool Calling으로 프로젝트 컨텍스트 능동적 수집:            │
│  - 디렉토리 구조 탐색                                       │
│  - 관련 파일 참조                                           │
│  - 의존성 분석                                              │
│  - 아키텍처 이해                                            │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                  Output & Integration                       │
│                                                             │
│  - Line-specific comments                                   │
│  - One-click fix suggestions                                │
│  - Handoff to Copilot Coding Agent (자동 수정 PR 생성)     │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Hybrid Detection 상세

```
┌─────────────────────────────────────────────────────────────┐
│              Deterministic + LLM Hybrid                     │
└─────────────────────────────────────────────────────────────┘

                    PR Diff Input
                         │
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │  CodeQL  │   │  ESLint  │   │   LLM    │
    │          │   │          │   │  Review  │
    │ Security │   │  Style   │   │          │
    │ Bugs     │   │  Lint    │   │ Context  │
    │ Vulns    │   │          │   │ Analysis │
    └────┬─────┘   └────┬─────┘   └────┬─────┘
         │              │              │
         └──────────────┼──────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │  Result Merger  │
              │                 │
              │ - Deduplication │
              │ - Prioritization│
              │ - Formatting    │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  PR Comments    │
              └─────────────────┘

장점:
- 결정론적 도구: 높은 정확도, 낮은 오탐
- LLM: 맥락 이해, 복잡한 패턴 탐지
- 결합: 상호 보완적 커버리지
```

### 2.3 Custom Instructions

```markdown
# .github/copilot-instructions.md (또는 instructions.md)

## Review Focus
- Prioritize test coverage comments
- Check for proper error handling
- Verify API contract compliance

## Style Preferences
- Prefer functional programming patterns
- Enforce strict TypeScript types
- Use async/await over callbacks

## Ignore
- Generated files in /dist
- Third-party code in /vendor
```

## 3. 핵심 기능

### 3.1 Repository Rules 기반 자동 리뷰

```yaml
# 모든 PR에 자동 Copilot 리뷰 적용
Repository Settings → Rules → Rulesets

- Target: All branches
- Rules:
  - Require Copilot code review
  - Block merge until review complete (optional)
```

### 3.2 Copilot Coding Agent 연동

```
리뷰 코멘트 → @copilot mention → 자동 수정 PR 생성

1. Copilot이 리뷰 코멘트 작성
2. 개발자가 "@copilot fix this" 요청
3. Copilot Coding Agent가:
   - 코드 수정 작성
   - Stacked PR 생성
   - 리뷰 대기
```

### 3.3 Multi-Platform Support

| 플랫폼 | 기능 |
|--------|------|
| github.com | Full PR review |
| VS Code | Real-time suggestions |
| Visual Studio | Integrated review |
| JetBrains | Extension support |
| Xcode | Apple platform support |

## 4. 성능 특성

### 4.1 벤치마크 결과 (Greptile 2025)

| 메트릭 | 값 | 평가 |
|--------|-----|------|
| 버그 탐지율 | ~55% | 중간 |
| False Positive | 낮음 | 좋음 |
| 노이즈 레벨 | 낮음 | 좋음 |
| 응답 속도 | 빠름 | 좋음 |

**특징**: 탐지율은 낮지만 노이즈가 적어 개발자 피로도 낮음

### 4.2 강점 vs 약점

| 강점 | 약점 |
|------|------|
| GitHub 네이티브 통합 | GitHub 전용 |
| CodeQL 연동 | 탐지율 낮음 |
| 낮은 설정 필요 | 커스터마이징 제한 |
| Coding Agent 연동 | On-premise 불가 |

## 5. 우리 설계에 대한 시사점

### 적용 가능한 패턴

| Copilot 패턴 | 우리 설계 적용 |
|-------------|---------------|
| **Hybrid Detection** | LLM + 정적분석 도구 결합 |
| **Custom Instructions** | 팀별 리뷰 가이드라인 적용 |
| **Agentic Tool Calling** | Context 수집 자동화 |
| **One-click Fix** | 자동 수정 제안 UI |

### Hybrid Detection 적용 아이디어

```python
# 우리 설계에 적용
class HybridReviewer:
    """결정론적 도구 + LLM 하이브리드"""

    def __init__(self):
        self.static_analyzers = [
            CppCheckAnalyzer(),      # C/C++ 정적 분석
            MisraChecker(),          # MISRA 규칙 검사
            CustomRuleChecker(),     # 사내 규칙
        ]
        self.llm_reviewer = LLMReviewer()

    async def review(self, file: FileInfo) -> ReviewResult:
        # 1. 정적 분석 (병렬)
        static_results = await asyncio.gather(*[
            analyzer.analyze(file)
            for analyzer in self.static_analyzers
        ])

        # 2. LLM 리뷰 (정적 분석 결과 참조)
        llm_result = await self.llm_reviewer.review(
            file,
            static_context=static_results  # 정적 분석 결과를 컨텍스트로
        )

        # 3. 결과 통합
        return self.merge_results(static_results, llm_result)

    def merge_results(self, static: List, llm: ReviewResult) -> ReviewResult:
        """중복 제거 및 우선순위화"""
        # 정적 분석과 LLM이 같은 이슈 지적 → 신뢰도 높음
        # 정적 분석만 지적 → 확실한 이슈
        # LLM만 지적 → Validator로 검증 필요
        pass
```

### 주의사항

1. **GitHub 종속**: github.com 외 환경 미지원
2. **온프레미스 불가**: 폐쇄망 환경 사용 불가
3. **커스터마이징 제한**: instructions.md 수준만 가능

## 6. 참조

- [GitHub Copilot Code Review Docs](https://docs.github.com/en/copilot/concepts/agents/code-review)
- [Using Copilot Code Review](https://docs.github.com/copilot/using-github-copilot/code-review/using-copilot-code-review)
- [New Features in Copilot Code Review (2025)](https://github.blog/changelog/2025-10-28-new-public-preview-features-in-copilot-code-review-ai-reviews-that-see-the-full-picture/)
