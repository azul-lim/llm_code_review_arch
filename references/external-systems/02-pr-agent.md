# PR-Agent (Qodo Merge) 아키텍처 분석

> 오픈소스 AI 코드 리뷰의 대표 주자
> Self-hosted 가능, 폐쇄망 환경에 적합

## 1. 개요

| 항목 | 내용 |
|------|------|
| 유형 | 오픈소스 (Apache 2.0) + Enterprise SaaS |
| LLM | OpenAI, Claude, Deepseek, Custom |
| 플랫폼 | GitHub, GitLab, Bitbucket, Azure DevOps, Gitea |
| 특화 | Self-hosted, PR Compression, 단일 LLM 호출 |
| GitHub | https://github.com/qodo-ai/pr-agent |

## 2. 아키텍처

### 2.1 전체 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    PR-Agent Architecture                    │
└─────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │  Git Platform   │
                    │ (GitHub/GitLab) │
                    └────────┬────────┘
                             │ Webhook / CLI / Action
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                     PR-Agent Core                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 Command Parser                       │   │
│  │  /describe, /review, /improve, /ask, /test          │   │
│  └──────────────────────────┬──────────────────────────┘   │
│                             │                              │
│  ┌──────────────────────────▼──────────────────────────┐   │
│  │              PR Compression Engine ★                 │   │
│  │                                                      │   │
│  │  1. Diff Extraction                                 │   │
│  │  2. Token-Aware Patch Fitting                       │   │
│  │  3. Context Selection (relevant files only)         │   │
│  │  4. Compression to fit LLM context window           │   │
│  └──────────────────────────┬──────────────────────────┘   │
│                             │                              │
│  ┌──────────────────────────▼──────────────────────────┐   │
│  │                 LLM Interface                        │   │
│  │                                                      │   │
│  │  - OpenAI (GPT-4, GPT-4 Turbo)                      │   │
│  │  - Anthropic (Claude)                               │   │
│  │  - Deepseek                                         │   │
│  │  - Custom OpenAI-compatible endpoints               │   │
│  └──────────────────────────┬──────────────────────────┘   │
│                             │                              │
│  ┌──────────────────────────▼──────────────────────────┐   │
│  │              JSON Output Parser                      │   │
│  │                                                      │   │
│  │  - Structured output validation                     │   │
│  │  - Schema-based parsing                             │   │
│  └──────────────────────────┬──────────────────────────┘   │
│                             │                              │
└─────────────────────────────┼───────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  Platform API   │
                    │  (PR Comments)  │
                    └─────────────────┘
```

### 2.2 핵심 Tools (Commands)

| Tool | 기능 | LLM 호출 |
|------|------|---------|
| `/describe` | PR 설명 자동 생성 | 1회 |
| `/review` | 코드 리뷰 수행 | 1회 |
| `/improve` | 개선 제안 | 1회 |
| `/ask <question>` | 코드 질의응답 | 1회 |
| `/test` | 테스트 제안 | 1회 |

**핵심 특징**: 각 도구가 **단일 LLM 호출**로 동작 (~30초)

### 2.3 PR Compression Strategy ★

```
┌─────────────────────────────────────────────────────────────┐
│                PR Compression Pipeline                       │
└─────────────────────────────────────────────────────────────┘

Input: Large PR (e.g., 50 files, 10K+ lines changed)
       ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 1: Priority Ranking                                   │
│                                                             │
│  - Important files first (src > test > config)              │
│  - Changed lines count                                      │
│  - File extension relevance                                 │
└────────────────────────────┬────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 2: Token Budget Allocation                            │
│                                                             │
│  Total Budget: ~100K tokens (model dependent)               │
│  - System prompt: ~2K                                       │
│  - PR metadata: ~1K                                         │
│  - File patches: ~90K (adaptive)                            │
│  - Output reserve: ~7K                                      │
└────────────────────────────┬────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 3: Adaptive Patch Fitting                             │
│                                                             │
│  for file in ranked_files:                                  │
│      patch = get_diff(file)                                 │
│      tokens = count_tokens(patch)                           │
│                                                             │
│      if tokens <= remaining_budget:                         │
│          include_full_patch(patch)                          │
│      else:                                                  │
│          include_truncated_patch(patch, remaining)          │
│          break                                              │
└────────────────────────────┬────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 4: Context Assembly                                   │
│                                                             │
│  Final Prompt:                                              │
│  - System instructions                                      │
│  - PR title, description                                    │
│  - Compressed file patches                                  │
│  - Output format specification (JSON)                       │
└─────────────────────────────────────────────────────────────┘

Output: Single prompt fitting within context window
```

## 3. 프롬프트 전략

### 3.1 /review 명령 프롬프트 구조

```yaml
# 개념적 구조 (실제 코드 기반)

system_prompt: |
  You are a code review expert. Analyze the PR and provide:
  - Key issues (bugs, security, performance)
  - Improvement suggestions
  - Code quality assessment

  Output format: JSON with specific schema

user_prompt: |
  ## PR Information
  Title: {title}
  Description: {description}

  ## File Changes
  {compressed_patches}

  ## Instructions
  1. Focus on significant issues
  2. Provide actionable feedback
  3. Use specific line references

output_schema:
  type: object
  properties:
    review:
      type: object
      properties:
        estimated_effort_to_review: string
        score: string
        relevant_tests: string
        security_concerns: string
    code_feedback:
      type: array
      items:
        type: object
        properties:
          relevant_file: string
          language: string
          suggestion: string
          relevant_line: string
          existing_code: string
          improved_code: string
```

### 3.2 JSON 기반 출력 강제

```python
# PR-Agent의 접근법
def get_llm_response(prompt: str, output_schema: dict) -> dict:
    """JSON 출력을 강제하는 LLM 호출"""

    # 1. System prompt에 JSON 출력 명시
    # 2. Output schema를 프롬프트에 포함
    # 3. 응답에서 JSON 파싱 시도
    # 4. 파싱 실패 시 재시도 또는 fallback

    response = llm.complete(
        prompt=prompt,
        response_format={"type": "json_object"}  # OpenAI
    )

    return json.loads(response)
```

## 4. 배포 옵션

### 4.1 GitHub Actions (권장)

```yaml
# .github/workflows/pr-agent.yml
name: PR-Agent

on:
  pull_request:
    types: [opened, synchronize]
  issue_comment:
    types: [created]

jobs:
  pr-agent:
    runs-on: ubuntu-latest
    steps:
      - uses: qodo-ai/pr-agent@main
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          openai_key: ${{ secrets.OPENAI_KEY }}
```

### 4.2 Self-Hosted Docker

```yaml
# docker-compose.yml
version: '3'
services:
  pr-agent:
    image: codiumai/pr-agent:latest
    environment:
      - OPENAI_KEY=${OPENAI_KEY}
      - GITHUB_TOKEN=${GITHUB_TOKEN}
      # 또는 Custom LLM endpoint
      - OPENAI_API_BASE=http://your-llm-server/v1
    ports:
      - "8080:8080"
```

### 4.3 CLI 모드

```bash
# 로컬 개발/테스트
pip install pr-agent

pr-agent review https://github.com/owner/repo/pull/123
```

## 5. 설정 (Configuration)

### 5.1 TOML 설정 파일

```toml
# .pr_agent.toml

[config]
model = "gpt-4-turbo"
model_turbo = "gpt-4-turbo"
fallback_models = ["gpt-4", "gpt-3.5-turbo"]
max_model_tokens = 128000
custom_model_max_tokens = 128000

[pr_reviewer]
require_score_review = true
require_tests_review = true
require_estimate_effort_to_review = true
require_security_review = true
num_code_suggestions = 4

[pr_description]
publish_labels = true
publish_description_as_comment = false
add_original_user_description = true
generate_ai_title = false

[custom_labels]
bug = "A bug fix"
feature = "A new feature"
refactor = "Code refactoring"
```

### 5.2 프롬프트 커스터마이징

```toml
[pr_reviewer]
extra_instructions = """
Focus on:
1. Memory safety issues (this is a C++ project)
2. Thread safety concerns
3. Performance implications

Ignore:
- Formatting issues (handled by clang-format)
- Naming conventions (handled by linter)
"""
```

## 6. 우리 설계에 대한 시사점

### 적용 가능한 패턴

| PR-Agent 패턴 | 우리 설계 적용 |
|--------------|---------------|
| **PR Compression** | 대형 PR 처리 시 파일 우선순위화 |
| **Single LLM Call** | Matcher+Reviewer 결합 시 참고 |
| **TOML 설정** | 커스텀 리뷰 규칙 설정 체계 |
| **JSON Output** | Structured Output 강제 |
| **Self-hosted** | 폐쇄망 환경 참조 아키텍처 |

### PR Compression 적용

```python
# 우리 설계에 적용
class PRCompressor:
    def __init__(self, token_budget: int = 200000):  # QWEN3 256K 기준
        self.budget = token_budget

    def compress(self, files: List[FileInfo]) -> List[FileInfo]:
        """우선순위 기반 파일 선택"""
        # 1. 중요도 순 정렬
        ranked = self.rank_files(files)

        # 2. 토큰 예산 내 파일 선택
        selected = []
        used = 0
        for f in ranked:
            tokens = self.count_tokens(f)
            if used + tokens <= self.budget * 0.8:  # 여유 확보
                selected.append(f)
                used += tokens
            else:
                # 일부만 포함 또는 스킵
                break

        return selected

    def rank_files(self, files: List[FileInfo]) -> List[FileInfo]:
        """파일 중요도 랭킹"""
        def score(f):
            score = 0
            # 소스 파일 우선
            if f.path.startswith('src/'): score += 100
            # 변경량 가중치
            score += min(f.additions + f.deletions, 500)
            # 테스트는 후순위
            if 'test' in f.path.lower(): score -= 50
            return score

        return sorted(files, key=score, reverse=True)
```

### 주의사항

1. **단일 호출의 한계**: 복잡한 리뷰는 품질 저하 가능
2. **압축 손실**: 중요한 컨텍스트가 누락될 수 있음
3. **검증 부재**: PR-Agent는 별도 Validation 단계 없음

## 7. 참조

- [PR-Agent GitHub Repository](https://github.com/qodo-ai/pr-agent)
- [PR-Agent Documentation](https://qodo-merge-docs.qodo.ai/)
- [Qodo Blog: Unveiling PR-Agent](https://www.qodo.ai/blog/unveiling-the-future-of-streamlined-software-development/)
