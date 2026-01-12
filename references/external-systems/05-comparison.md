# 외부 시스템 비교 분석

## 1. 전체 비교표

### 1.1 기본 특성

| 항목 | CodeRabbit | PR-Agent | Copilot | CodeGuru |
|------|------------|----------|---------|----------|
| **유형** | SaaS | OSS + SaaS | SaaS | SaaS |
| **LLM** | GPT-4.5, Claude | GPT-4, Claude, Custom | GitHub AI | ML (비-LLM) |
| **Self-hosted** | ❌ (Enterprise만) | ✅ | ❌ | ❌ |
| **컨텍스트** | Codegraph + Semantic | PR Compression | Agentic Gathering | Static Analysis |
| **검증** | Verification Agents | JSON Schema | CodeQL + LLM | Confidence Score |
| **플랫폼** | GitHub, GitLab, BB | GitHub, GitLab, BB, Azure | GitHub만 | GitHub, BB, CodeCommit |

### 1.2 성능 메트릭

| 메트릭 | CodeRabbit | PR-Agent | Copilot | CodeGuru |
|--------|------------|----------|---------|----------|
| **버그 탐지율** | 44% | N/A | ~55% | N/A |
| **False Positive** | 높음 | 중간 | 낮음 | 낮음 |
| **노이즈 레벨** | 높음 | 중간 | 낮음 | 낮음 |
| **응답 속도** | 중간 | 빠름 (~30s) | 빠름 | 중간 |
| **커스터마이징** | 높음 | 높음 | 중간 | 낮음 |

### 1.3 시장 점유율 (2025)

| 시스템 | PR 처리량 | 조직 수 |
|--------|----------|---------|
| CodeRabbit | 632,256 PRs | 7,478 orgs |
| GitHub Copilot | 561,382 PRs | 29,316 orgs |
| PR-Agent | N/A | N/A (OSS) |

## 2. 아키텍처 패턴 비교

### 2.1 컨텍스트 수집

```
CodeRabbit:
┌──────────────────────────────────────────────┐
│  Pre-built Context                           │
│  - Codegraph (dependency map)                │
│  - Semantic Index (embeddings)               │
│  - Team Guidelines                           │
│  → 풍부한 컨텍스트, 높은 인프라 비용         │
└──────────────────────────────────────────────┘

PR-Agent:
┌──────────────────────────────────────────────┐
│  On-demand Compression                       │
│  - PR Diff only                              │
│  - Token-aware fitting                       │
│  - Priority-based selection                  │
│  → 가벼움, 컨텍스트 제한적                   │
└──────────────────────────────────────────────┘

Copilot:
┌──────────────────────────────────────────────┐
│  Agentic Tool Calling                        │
│  - Dynamic context gathering                 │
│  - File exploration on-demand                │
│  - Repository structure awareness            │
│  → 유연함, 추가 지연 발생                    │
└──────────────────────────────────────────────┘
```

### 2.2 검증 메커니즘

```
┌────────────────────────────────────────────────────────────────────┐
│                        Validation Approaches                        │
└────────────────────────────────────────────────────────────────────┘

CodeRabbit: Agentic Verification ★ (가장 강력)
┌─────────────────────────────────────────────────────────────────┐
│  LLM Output → Generate Script → Execute in Sandbox → Verify     │
│                                                                 │
│  예: "import X 추가 필요" → grep으로 X 존재 확인 → 검증 후 출력 │
└─────────────────────────────────────────────────────────────────┘

PR-Agent: Schema Validation (기본)
┌─────────────────────────────────────────────────────────────────┐
│  LLM Output → JSON Parse → Schema Validate → Output             │
│                                                                 │
│  구조적 검증만, 내용 검증 없음                                  │
└─────────────────────────────────────────────────────────────────┘

Copilot: Hybrid Detection (도구 연동)
┌─────────────────────────────────────────────────────────────────┐
│  CodeQL + ESLint → Deterministic Issues                         │
│  LLM → Contextual Issues                                        │
│  → 결과 통합 및 중복 제거                                       │
└─────────────────────────────────────────────────────────────────┘

CodeGuru: Confidence Score (ML 기반)
┌─────────────────────────────────────────────────────────────────┐
│  ML Model → Issue + Confidence Score → Threshold Filter         │
│                                                                 │
│  피드백 학습으로 지속 개선                                      │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 프롬프트 전략

| 시스템 | 프롬프트 구조 | 특징 |
|--------|-------------|------|
| **CodeRabbit** | Multi-stage with Tool Results | 도구 결과를 "instructive manner"로 전달 |
| **PR-Agent** | Single Prompt + JSON Schema | 압축된 단일 프롬프트 |
| **Copilot** | Tool-calling Enabled | 필요시 추가 정보 요청 |

## 3. 폐쇄망/Self-hosted 옵션

### 3.1 비교

| 시스템 | Self-hosted | 폐쇄망 지원 | 비고 |
|--------|-------------|------------|------|
| **PR-Agent** | ✅ 완전 지원 | ✅ | Custom LLM endpoint 지원 |
| CodeRabbit | ⚠️ Enterprise만 | ⚠️ | 별도 협의 필요 |
| Copilot | ❌ | ❌ | GitHub 종속 |
| CodeGuru | ❌ | ❌ | AWS 종속 |

### 3.2 Self-hosted 구성 (PR-Agent)

```yaml
# 폐쇄망 환경 구성 예시
services:
  pr-agent:
    image: codiumai/pr-agent:latest
    environment:
      # 내부 LLM 서버 연결
      - OPENAI_API_BASE=http://internal-llm-server:8080/v1
      - OPENAI_KEY=dummy  # 내부 서버용
      # 또는 Ollama
      - OLLAMA_API_BASE=http://ollama:11434
    networks:
      - internal

  ollama:
    image: ollama/ollama
    volumes:
      - ./models:/root/.ollama/models
    # GPU 지원
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

## 4. 품질 vs 노이즈 트레이드오프

```
              높은 탐지율
                  ↑
                  │     ★ Greptile (82%)
                  │
                  │
                  │         ★ Cursor/Copilot (~55%)
                  │
                  │              ★ CodeRabbit (44%)
                  │
                  └──────────────────────────────→ 낮은 노이즈
                                                   (높은 정밀도)

관찰:
- 높은 탐지율 = 더 많은 False Positive
- 낮은 노이즈 = 놓치는 이슈 존재
- 트레이드오프 존재

권장:
- 중요 코드 (보안, 핵심 로직): 높은 탐지율 선호
- 일반 코드: 낮은 노이즈 선호
- 적응형: 파일/컨텍스트에 따라 조절
```

## 5. 우리 환경 적합성 평가

### 5.1 환경 조건

| 조건 | 요구사항 |
|------|----------|
| 네트워크 | 폐쇄망 |
| 모델 | QWEN3 (256K), GPT-OSS (128K) |
| 시간 제약 | PR당 10분 |
| 규모 | 20-50+ 파일/PR |

### 5.2 시스템별 적합성

| 시스템 | 적합성 | 이유 |
|--------|--------|------|
| **PR-Agent** | ⭐⭐⭐⭐⭐ | Self-hosted, Custom LLM, 빠른 처리 |
| CodeRabbit | ⭐⭐ | 아키텍처 참고만, SaaS 제한 |
| Copilot | ⭐ | GitHub 종속, 폐쇄망 불가 |
| CodeGuru | ⭐⭐ | ML 패턴 참고만 |

### 5.3 채택 권장 패턴

```
우선 채택:
1. PR-Agent의 PR Compression 전략
2. PR-Agent의 TOML 설정 체계
3. PR-Agent의 JSON Schema 기반 출력

선택 채택:
4. CodeRabbit의 Verification Agents 패턴 (Validator 강화)
5. CodeGuru의 Confidence Score 시스템
6. Copilot의 Hybrid Detection (정적분석 연동)

참고만:
7. CodeRabbit의 Codegraph (인프라 부담)
8. CodeRabbit의 Semantic Index (인프라 부담)
```

## 6. 결론

### 6.1 핵심 인사이트

1. **검증이 핵심**: 모든 시스템이 어떤 형태로든 검증 메커니즘 보유
2. **컨텍스트 효율성**: 대용량 PR 처리를 위한 압축/선별 필수
3. **하이브리드 접근**: LLM + 결정론적 도구 조합이 트렌드
4. **노이즈 관리**: 탐지율과 정밀도 사이 균형 필요

### 6.2 우리 설계 방향

```
현재 설계:
Refiner → Matcher → Reviewer → Validator → Summarizer

강화 방향:
1. Validator: CodeRabbit 스타일 검증 스크립트 추가
2. Reviewer: 정적 분석 도구 결과 통합 (Copilot 패턴)
3. 전체: PR Compression 적용 (PR-Agent 패턴)
4. 출력: Confidence Score 추가 (CodeGuru 패턴)
```
