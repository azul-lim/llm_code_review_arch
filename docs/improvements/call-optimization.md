# 듀얼 모델 병렬 처리 전략

> 이 문서는 QWEN3 + GPT-OSS 듀얼 모델 환경에서의 병렬 처리 전략을 정의합니다.
> Task 및 Workflow 상세는 `docs/04-task-workflow-design.md` 참조

---

## 1. 모델 특성 비교

| 항목 | QWEN3 (주) | GPT-OSS (보조) |
|------|-----------|----------------|
| 컨텍스트 | 256K | 128K+ |
| 동시성 | 8개 | 50개/분 |
| 품질 | 기준 | 동등 수준 |
| 특이사항 | 중국어 출력 가능성 | - |

---

## 2. 현재 병목 분석

```
현재 상황 (50 파일 기준):
- 파일당: 리뷰 100s + 검증 100s = 200s
- 순차 처리: 50 × 200s = 10,000s ≈ 166분 ❌

QWEN3만 사용 (8 동시):
- 리뷰: ceil(50/8) × 100s = 700s
- 검증: ceil(50/8) × 100s = 700s
- 총: 1,400s ≈ 23분 ❌

목표: 10분(600s) 이내
```

---

## 3. Task별 모델 할당

| Task | 기본 모델 | 폴백 | 이유 |
|------|----------|------|------|
| **P1** | QWEN3 | - | 복잡한 이해/구조화 |
| **F1** | GPT-OSS | QWEN3 | 매칭은 상대적으로 단순 |
| **F2** | GPT-OSS | - | 표면 검사는 단순 |
| **F3** | QWEN3 | - | 복잡한 추론 필수 (COT) |
| **F4** | GPT-OSS | QWEN3 | 규칙 기반 |
| **F5** | GPT-OSS | QWEN3 | 통합/포맷팅은 단순 |
| **F6** | GPT-OSS | - | 넓은 컨텍스트 필요 |
| **V1-V4** | GPT-OSS | - | 단순 검증 작업 |
| **P2** | QWEN3 | - | 전체 요약은 복잡 |

---

## 4. 스트리밍 파이프라인

```python
import asyncio
from asyncio import Semaphore, Queue

class DualModelPipeline:
    def __init__(self):
        self.qwen_sem = Semaphore(8)
        self.review_queue = Queue()
        self.gptoss_rate_limiter = RateLimiter(50, 60)  # 50/min

    async def process_pr(self, files: List[FileInfo], refined_spec: RefinedSpec):
        # 검증 워커 시작 (백그라운드)
        validator_task = asyncio.create_task(
            self.validation_worker()
        )

        # 리뷰 작업 병렬 실행
        review_tasks = []
        for file in files:
            task = asyncio.create_task(
                self.review_file(file, refined_spec)
            )
            review_tasks.append(task)

        # 모든 리뷰 완료 대기
        review_results = await asyncio.gather(*review_tasks)

        # 검증 완료 신호
        await self.review_queue.put(None)  # Sentinel
        validated_reviews = await validator_task

        return validated_reviews

    async def review_file(self, file: FileInfo, spec: RefinedSpec):
        async with self.qwen_sem:  # 8개 제한
            result = await self.qwen_client.review(file, spec)
            # 즉시 검증 큐에 추가
            await self.review_queue.put(result)
            return result

    async def validation_worker(self):
        validated = []
        while True:
            result = await self.review_queue.get()
            if result is None:  # 종료 신호
                break

            # GPT-OSS로 검증 (V1-V4 병렬)
            async with self.gptoss_rate_limiter:
                validated_result = await self.gptoss_client.validate(result)
                validated.append(validated_result)

        return validated
```

---

## 5. 동적 부하 분산

```python
class AdaptiveLoadBalancer:
    """실시간 응답 시간 기반 동적 분산"""

    def __init__(self):
        self.qwen_avg_time = 100.0
        self.gptoss_avg_time = 100.0

    def assign_model(self, file: FileInfo, task: str) -> str:
        """Task/파일별 최적 모델 할당"""

        # F3, P1, P2는 항상 QWEN3
        if task in ["P1", "F3", "P2"]:
            return "qwen3"

        # 큰 파일은 컨텍스트 큰 QWEN3
        if file.total_tokens > 100000:
            return "qwen3"

        # 현재 부하 기반 결정
        qwen_pending = self.qwen_pending_count
        gptoss_pending = self.gptoss_pending_count

        qwen_expected = qwen_pending * self.qwen_avg_time / 8
        gptoss_expected = gptoss_pending * self.gptoss_avg_time

        if qwen_expected < gptoss_expected:
            return "qwen3"
        else:
            return "gptoss"
```

---

## 6. 시간 예산 (30 파일 기준)

```
목표: 10분 (600초) 이내 완료

┌─────────────────────────────────────────────────────────────┐
│  P1 (1회)           │████│                    ~30초        │
├─────────────────────────────────────────────────────────────┤
│  파일 처리 (30개)                                           │
│                                                             │
│  QWEN3 (F3): 30파일 ÷ 8동시 × ~20초 = ~75초               │
│  GPT-OSS (F1,F2,F4,F5,V1-V4): 병렬 처리 = ~200초          │
│                                                             │
│  ※ 파이프라인 효과로 실제는 더 빠름                        │
│  ※ 예상 총 파일 처리: ~300초                               │
├─────────────────────────────────────────────────────────────┤
│  F6 (선택)          │██│                      ~30초        │
├─────────────────────────────────────────────────────────────┤
│  P2 (1회)           │████│                    ~30초        │
├─────────────────────────────────────────────────────────────┤
│  Buffer                                         ~210초      │
├─────────────────────────────────────────────────────────────┤
│  Total                                          ~600초      │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 구현 설정

```yaml
# config/dual-model.yaml
models:
  primary:
    name: "qwen3"
    max_concurrent: 8
    context_limit: 256000
    use_for:
      - P1  # Description Refiner
      - F3  # Logic Bug Detection
      - P2  # PR Summary

  secondary:
    name: "gptoss"
    rate_limit: 50  # per minute
    context_limit: 128000
    use_for:
      - F1  # Intent Matching
      - F2  # Surface Error Detection
      - F4  # Domain Rule Validation
      - F5  # Review Synthesis
      - F6  # Cross-file Analysis
      - V1  # Description Accuracy
      - V2  # Location Verification
      - V3  # Suggestion Validity
      - V4  # Hallucination Detection

pipeline:
  strategy: "streaming"

  skip_conditions:
    comment_only: true
    config_files: true
    generated_files: true

timeout:
  pr_max_seconds: 600  # 10분
  file_max_seconds: 120
  task_max_seconds: 60
```

---

## 관련 문서

- Task/Workflow 설계: [04-task-workflow-design.md](../04-task-workflow-design.md)
- 환경 제약: [03-current-environment.md](../03-current-environment.md)
