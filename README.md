# LLM Code Review Architecture

LLM 기반 코드 리뷰 시스템 개선을 위한 설계 문서 프로젝트

## 개요

이 프로젝트는 기존 LLM Code Review 시스템의 개선을 위한 **설계 문서**를 포함합니다. 설계 산출물은 AI 모델(Copilot, Claude Code 등)에게 제공되어 기존 시스템 개선 구현에 활용됩니다.

## 대상

- **주요 대상**: Storage Firmware 코드
- **추가 대상**: 비-FW 과제 코드

## 프로젝트 구조

```
docs/           # 설계 문서
schemas/        # JSON Schema (인터페이스 정의)
examples/       # 예시 데이터
diagrams/       # 아키텍처 다이어그램
```

## 문서 목차

| 문서 | 설명 |
|------|------|
| [00-problem-analysis.md](docs/00-problem-analysis.md) | 현재 시스템 문제점 분석 |
| [01-overview.md](docs/01-overview.md) | 개선 시스템 개요 |
| [02-architecture.md](docs/02-architecture.md) | 전체 아키텍처 설계 |
| [components/](docs/components/) | 컴포넌트별 상세 설계 |
| [prompts/](docs/prompts/) | 프롬프트 설계 |
| [improvements/](docs/improvements/) | 개선 제안 |

## 활용 방법

1. 설계 문서를 AI 모델에게 제공
2. AI가 설계를 이해하고 기존 Python 시스템 개선 코드 생성
3. 생성된 코드 검토 및 적용

## 관련 문서

- [CLAUDE.md](CLAUDE.md) - AI 모델용 프로젝트 컨텍스트
