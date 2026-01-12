# CodeRabbit 아키텍처 분석

> 가장 성숙한 AI 코드 리뷰 플랫폼
> $88M 펀딩 (Series B, 2025)

## 1. 개요

| 항목 | 내용 |
|------|------|
| 유형 | SaaS (일부 Enterprise는 Private Cloud) |
| LLM | GPT-4.5, o3, o4-mini, Claude Opus 4, Sonnet 4 |
| 플랫폼 | GitHub, GitLab, Bitbucket, Azure DevOps |
| 특화 | 대규모 코드베이스, Agentic Validation |

## 2. 아키텍처

### 2.1 인프라 구조 (Google Cloud Run 기반)

```
┌─────────────────────────────────────────────────────────────┐
│                    Webhook Handler                          │
│            (Cloud Run - Lightweight Service)                │
│         - Billing/Subscription Check                        │
│         - Push to Cloud Tasks Queue                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Cloud Tasks Queue                         │
│              (Decoupling & Burst Handling)                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│               Execution Service (Cloud Run)                 │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           2-Layer Sandboxing                         │   │
│  │  ┌─────────────────────────────────────────────┐    │   │
│  │  │  Gen2 microVM (Full Linux cgroup)           │    │   │
│  │  │  ┌─────────────────────────────────────┐    │    │   │
│  │  │  │  Jailkit (Process Isolation)        │    │    │   │
│  │  │  │  - Code Review Execution            │    │    │   │
│  │  │  │  - Verification Scripts             │    │    │   │
│  │  │  └─────────────────────────────────────┘    │    │   │
│  │  └─────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Config: 3600s timeout, 8 concurrent/instance              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 리뷰 파이프라인

```
┌─────────────────────────────────────────────────────────────┐
│                   1. Context Engineering                    │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Codegraph  │  │  Semantic   │  │  Team Guidelines    │ │
│  │  Mapping    │  │  Index      │  │  & Instructions     │ │
│  │             │  │  (Embeddings)│  │                     │ │
│  │ - Defs      │  │             │  │ - Custom Rules      │ │
│  │ - Refs      │  │ - Functions │  │ - Exceptions        │ │
│  │ - Co-change │  │ - Classes   │  │ - Style Guide       │ │
│  └─────────────┘  │ - Tests     │  └─────────────────────┘ │
│                   │ - Prior PRs │                          │
│                   └─────────────┘                          │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   2. Tool Integration                       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  40+ Linters & Security Scanners                     │  │
│  │  - ESLint, Prettier, Stylelint                       │  │
│  │  - Semgrep, Snyk, Trivy                              │  │
│  │  - ast-grep (Structural Analysis)                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  → 도구 결과를 "instructive manner"로 LLM에 전달           │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   3. LLM Reasoning                          │
│                                                             │
│  - Context + Tool Results → Reasoning Model                 │
│  - Multi-model Support (GPT-4.5, Claude, etc.)              │
│  - MCP Integration (Jira, Linear, Web Query)                │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│              4. Agentic Code Validation ★                   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Verification Agents ("Tools in Jail")       │  │
│  │                                                       │  │
│  │  Before posting comments:                             │  │
│  │  1. Generate shell/Python verification scripts        │  │
│  │  2. Execute in sandboxed environment                  │  │
│  │  3. Confirm assumptions with actual code              │  │
│  │  4. Extract evidence from codebase                    │  │
│  │                                                       │  │
│  │  Examples:                                            │  │
│  │  - grep/ast-grep to verify code patterns              │  │
│  │  - Check if suggested import actually exists          │  │
│  │  - Validate line numbers and code structure           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  → Hallucination Prevention: 검증 실패 시 코멘트 제외       │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   5. Output Formatting                      │
│                                                             │
│  - Line-specific comments                                   │
│  - PR Summary with key findings                             │
│  - Suggested fixes (one-click apply)                        │
│  - Walk-through documentation                               │
└─────────────────────────────────────────────────────────────┘
```

## 3. 핵심 기술 상세

### 3.1 Codegraph

```python
# 개념적 구조
class Codegraph:
    """경량 의존성 맵"""

    def __init__(self, repo):
        self.definitions = {}    # symbol -> file:line
        self.references = {}     # symbol -> [file:line, ...]
        self.co_changes = {}     # file -> [frequently_changed_together]

    def get_impact_scope(self, changed_files: List[str]) -> List[str]:
        """변경 파일의 영향 범위 파악"""
        impacted = set()
        for file in changed_files:
            # 이 파일을 참조하는 다른 파일들
            for ref in self.references.get(file, []):
                impacted.add(ref)
            # 자주 함께 변경되는 파일들
            for co in self.co_changes.get(file, []):
                impacted.add(co)
        return list(impacted)
```

### 3.2 Semantic Index

```python
# 개념적 구조
class SemanticIndex:
    """임베딩 기반 코드 검색"""

    def __init__(self):
        self.embeddings = {}  # chunk_id -> vector
        self.chunks = {}      # chunk_id -> code_content

    def search_by_purpose(self, query: str, top_k: int = 5) -> List[CodeChunk]:
        """목적/의도 기반 검색 (키워드가 아닌)"""
        query_embedding = self.embed(query)
        # 유사한 구현, 과거 해결 방법 등 검색
        return self.vector_search(query_embedding, top_k)
```

### 3.3 Verification Agents

```python
# 개념적 구조
class VerificationAgent:
    """검증 에이전트 - "Tools in Jail" 패턴"""

    def verify_comment(self, comment: ReviewComment, codebase: str) -> bool:
        """리뷰 코멘트 검증"""

        # 1. 검증 스크립트 생성
        script = self.generate_verification_script(comment)

        # 2. 샌드박스에서 실행
        result = self.execute_in_sandbox(script, codebase)

        # 3. 검증 결과 확인
        return result.confirms_assumption

    def generate_verification_script(self, comment: ReviewComment) -> str:
        """검증용 스크립트 생성"""
        # Examples:
        # - grep으로 패턴 존재 확인
        # - ast-grep으로 구조적 매칭
        # - 라인 번호 유효성 검사
        pass
```

## 4. 확장 기능 (2025)

| 기능 | 설명 |
|------|------|
| CLI Integration | 터미널에서 직접 리뷰 요청 |
| Pre-merge Checks | 머지 전 자동 검증 |
| VS Code Extension | IDE 내 실시간 리뷰 |
| Unit Test Generation | 테스트 코드 자동 생성 |
| MCP Integration | Jira, Linear, 외부 문서 연동 |

## 5. 우리 설계에 대한 시사점

### 적용 가능

| CodeRabbit 패턴 | 우리 설계 적용 |
|----------------|---------------|
| Codegraph | Change Matcher 강화 - 영향 범위 분석 |
| Semantic Index | 유사 코드/과거 PR 참조 |
| Verification Agents | **Validator 핵심 강화** |
| Tool Integration | 정적 분석 도구 연동 |

### 주의사항

1. **노이즈 문제**: CodeRabbit은 많은 코멘트로 인한 피로감 보고됨
2. **리소스**: Codegraph/Semantic Index는 인프라 필요
3. **폐쇄망**: SaaS 모델이므로 직접 적용 불가

### Validator 강화 아이디어

```
현재 우리 설계:
Validator → Stage A (이슈 검증) → Stage B (라인 추출)

CodeRabbit 패턴 적용:
Validator → Stage A (이슈 검증)
              ↓
          Verification Script 생성
              ↓
          샌드박스 실행 (grep, ast-grep 등)
              ↓
          검증 성공만 통과
              ↓
         Stage B (라인 추출)
```

## 6. 참조

- [CodeRabbit Documentation](https://docs.coderabbit.ai/)
- [How CodeRabbit built its AI code review agent](https://cloud.google.com/blog/products/ai-machine-learning/how-coderabbit-built-its-ai-code-review-agent-with-google-cloud-run)
- [Agentic Code Validation Blog](https://www.coderabbit.ai/blog/how-coderabbits-agentic-code-validation-helps-with-code-reviews)
- [Accurate AI Code Reviews on Massive Codebases](https://www.coderabbit.ai/blog/how-coderabbit-delivers-accurate-ai-code-reviews-on-massive-codebases)
