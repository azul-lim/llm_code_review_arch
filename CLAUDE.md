# CLAUDE.md

이 파일은 AI 모델(Claude, Copilot 등)이 이 프로젝트를 이해하고 기존 시스템 개선에 활용할 수 있도록 컨텍스트를 제공합니다.

## 프로젝트 목적

**기존 LLM Code Review 시스템의 개선을 위한 설계 문서 프로젝트**

이 프로젝트의 산출물은 AI 모델에게 제공되어 기존 Python 기반 코드 리뷰 시스템을 개선하는 데 사용됩니다. 따라서 모든 문서는 AI가 이해하고 구현할 수 있는 형태로 작성됩니다.

## 대상 시스템

- **주요 대상**: Storage Firmware 코드
- **추가 대상**: 비-FW 과제 코드
- **기존 시스템**: Python 기반, QWEN3 LLM 사용

## 프로젝트 구조

```
llm_code_review_arch/
├── CLAUDE.md                      # 이 파일 (AI용 컨텍스트)
├── README.md                      # 프로젝트 개요
│
├── docs/                          # 설계 문서
│   ├── 00-problem-analysis.md     # 현재 시스템 문제점 분석
│   ├── 03-current-environment.md  # 환경 제약 조건
│   ├── 04-task-workflow-design.md # ★ Task/Workflow 설계 (핵심) ★
│   │
│   ├── components/                # 참조용 (P1, P2 매핑)
│   │   ├── 01-description-refiner.md  # → P1
│   │   └── 04-summary-generator.md    # → P2
│   │
│   ├── prompts/                   # 프롬프트 설계
│   │   └── prompt-strategy.md
│   │
│   └── improvements/              # 개선 전략
│       ├── call-optimization.md   # 듀얼 모델 병렬 처리
│       └── cot-enhancement.md     # COT 개선 (F3 적용)
│
├── references/                    # 외부 시스템 분석
│   └── external-systems/
│       ├── 00-overview.md
│       ├── 01-coderabbit.md
│       ├── 02-pr-agent.md
│       ├── 03-github-copilot.md
│       ├── 04-amazon-codeguru.md
│       ├── 05-comparison.md
│       └── 06-lessons-learned.md
│
├── schemas/                       # JSON Schema (업데이트 필요)
│   ├── input/
│   ├── internal/
│   └── output/
│
├── examples/                      # 예시 데이터
│
└── diagrams/                      # 다이어그램
    └── current-flow.md            # 현재 시스템 흐름
```

## 문서 읽는 순서

1. `docs/00-problem-analysis.md` - 현재 시스템의 문제점 이해
2. `docs/03-current-environment.md` - 환경 제약 조건 파악
3. **`docs/04-task-workflow-design.md` - ★ Task/Workflow 설계 (핵심) ★**
4. `docs/improvements/call-optimization.md` - 듀얼 모델 병렬 처리 전략
5. `references/external-systems/05-comparison.md` - 외부 시스템 비교 참조

## 시스템 개요

### 기존 시스템 (문제점)
```
파일당 2회 호출: 리뷰 1회 + 검증 1회
└─ 리뷰 1회에서 6개+ Task 동시 수행 → 품질 불안정
```

### 신규 설계 (Task 분화)
```
PR Level:
├─ P1: PR Description Refinement [QWEN3]
└─ P2: PR Summary Generation [QWEN3]

File Level (병렬):
├─ F1: Intent Matching [GPT-OSS]
├─ F2: Surface Error Detection [GPT-OSS]
├─ F3: Logic Bug Detection [QWEN3, COT] ★핵심★
├─ F4: Domain Rule Validation [GPT-OSS]
├─ F5: Review Synthesis [GPT-OSS]
└─ F6: Cross-file Analysis [Optional]

Validation (병렬, AND 로직):
├─ V1: Description Accuracy [GPT-OSS]
├─ V2: Location Verification [GPT-OSS]
├─ V3: Suggestion Validity [GPT-OSS]
└─ V4: Hallucination Detection [GPT-OSS]
```

### 환경 제약
| 항목 | 제약 |
|------|------|
| 네트워크 | 폐쇄망 |
| 주 모델 | QWEN3 (256K context, 8개 동시) |
| 보조 모델 | GPT-OSS (128K+ context, 50개/분) |
| 시간 제한 | PR당 10분 |
| PR 규모 | 일반 20~30개, 최대 50+ 파일 |

### 듀얼 모델 병렬 처리 전략
```
QWEN3 (품질 우선) ──┐
                    ├─→ 스트리밍 파이프라인 ─→ 10분 이내 완료
GPT-OSS (속도 우선) ┘

모델 할당:
- QWEN3 필수: P1, F3, P2 (복잡한 추론)
- GPT-OSS 기본: F1, F2, F4, F5, V1-V4 (단순 작업)
- 동적 라우팅: Rate limit에 따라 폴백
```

## 설계 원칙

1. **명확한 역할 분리**: 각 Task는 단일 책임 원칙 준수
2. **구조화된 I/O**: JSON Schema로 명확한 인터페이스 정의
3. **단계적 추론**: F3에서 COT 명시적 구조화
4. **예시 기반 설명**: 모든 설계에 구체적 예시 포함

## AI 구현 가이드

### 구현 순서 (권장)

```
Phase 1: 핵심 Task
├─ P1: Description Refiner
├─ F3: Logic Bug Detection (COT 포함)
├─ F5: Review Synthesis
├─ V1-V4: Validation Tasks
└─ P2: PR Summary

Phase 2: 보조 Task
├─ F1: Intent Matching
├─ F2: Surface Error Detection
└─ F4: Domain Rule Validation

Phase 3: 고급 기능
├─ F6: Cross-file Analysis
├─ Dynamic Model Router
└─ Pipeline Orchestrator
```

### 구현 체크리스트

1. **`docs/04-task-workflow-design.md` 숙지** - Task 정의 및 Workflow
2. **기존 코드 분석** - 현재 프롬프트 및 JSON 구조 확인
3. **스키마 정의** - `schemas/`에 각 Task I/O 스키마 (신규 정의 필요)
4. **프롬프트 작성** - Task별 프롬프트 템플릿
5. **테스트** - 실제 PR 데이터로 동작 검증

### 주의사항

- 각 Task는 단일 목적으로 유지
- JSON Schema 엄격히 준수
- 에러 발생 시 해당 이슈만 스킵 (전체 실패 방지)
- 모델 사용량/시간 로깅 필수
