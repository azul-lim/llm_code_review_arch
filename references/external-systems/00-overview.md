# 외부 LLM 코드 리뷰 시스템 분석 개요

> 조사일: 2025년 1월
> 목적: 우리 설계와 비교하여 개선점 도출

## 조사 대상 시스템

| 시스템 | 유형 | 주요 특징 |
|--------|------|----------|
| **CodeRabbit** | SaaS | 가장 성숙한 AI 코드리뷰, Agentic Validation |
| **PR-Agent (Qodo)** | 오픈소스 | Self-hosted 가능, PR 압축 전략 |
| **GitHub Copilot** | SaaS | GitHub 네이티브 통합, CodeQL 연동 |
| **Amazon CodeGuru** | SaaS | ML 기반 (비-LLM), 패턴 학습 |
| **Sourcery** | SaaS | Python 특화, IDE 통합 |

## 핵심 발견 사항

### 1. 아키텍처 패턴

```
공통 파이프라인 구조:
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│ Context │ → │ LLM     │ → │ Validate│ → │ Output  │
│ Build   │   │ Analysis│   │ /Filter │   │ Format  │
└─────────┘   └─────────┘   └─────────┘   └─────────┘
```

- **CodeRabbit**: Codegraph + Semantic Index + Verification Agents
- **PR-Agent**: PR Compression + Single LLM Call + JSON Output
- **Copilot**: CodeQL Integration + Agentic Tool Calling

### 2. 검증(Validation) 메커니즘

| 시스템 | 검증 방식 |
|--------|----------|
| CodeRabbit | Verification Agents (ast-grep, shell scripts) |
| PR-Agent | JSON Schema 기반 출력 검증 |
| Copilot | CodeQL + ESLint 연동 |
| CodeGuru | ML confidence score + 피드백 학습 |

### 3. 할루시네이션 대응

- **CodeRabbit**: "Tools in Jail" - 샌드박스에서 검증 스크립트 실행
- **PR-Agent**: 단일 호출로 복잡도 최소화
- **Copilot**: 결정론적 도구(CodeQL)와 LLM 결합

### 4. 성능 벤치마크 (Greptile 2025)

| 도구 | 버그 탐지율 | 특징 |
|------|------------|------|
| Greptile | 82% | 높은 탐지율, 일부 False Positive |
| Cursor | ~55% | 중간 수준 |
| Copilot | ~55% | 낮은 노이즈, 낮은 탐지율 |
| CodeRabbit | 44% | 상세한 피드백, 높은 노이즈 |

## 우리 설계에 대한 시사점

### 적용 가능한 패턴

1. **Codegraph (CodeRabbit)** → 우리의 Change Matcher 강화
2. **PR Compression (PR-Agent)** → 대용량 PR 처리 최적화
3. **Verification Agents (CodeRabbit)** → Validator 강화
4. **Tool Integration (Copilot)** → 정적 분석 도구 연동

### 주의사항

1. **노이즈 vs 탐지율 트레이드오프**
2. **Self-hosted 환경** 제약 고려
3. **단일 호출 vs 다단계** 선택 기준

## 문서 구조

```
references/external-systems/
├── 00-overview.md          # 이 문서
├── 01-coderabbit.md        # CodeRabbit 상세 분석
├── 02-pr-agent.md          # PR-Agent 상세 분석
├── 03-github-copilot.md    # GitHub Copilot 상세 분석
├── 04-amazon-codeguru.md   # Amazon CodeGuru 상세 분석
├── 05-comparison.md        # 비교 분석
└── 06-lessons-learned.md   # 우리 설계 반영 사항
```

## 참조 링크

- [CodeRabbit Docs](https://docs.coderabbit.ai/)
- [PR-Agent GitHub](https://github.com/qodo-ai/pr-agent)
- [GitHub Copilot Code Review](https://docs.github.com/en/copilot/concepts/agents/code-review)
- [Amazon CodeGuru](https://docs.aws.amazon.com/codeguru/latest/reviewer-ug/welcome.html)
- [Greptile Benchmarks 2025](https://www.greptile.com/benchmarks)
- [State of AI Code Review 2025](https://pullflow.com/state-of-ai-code-review-2025)
