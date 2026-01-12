# LLM 호출 최적화

## 1. 현재 문제

### 1.1 단일 호출 과부하

현재 시스템은 호출 횟수 최소화를 위해 파일당 1회 호출에 모든 작업을 수행:

```
현재: 1 호출 = 4 역할
┌────────────────────────────────────────────────────┐
│  - PR Description 정제                              │
│  - Description ↔ 변경 매칭                          │
│  - 코드 리뷰 (해석 + 문제 탐지 + 제안)              │
│  - (파일 완료 후) 요약 생성                         │
└────────────────────────────────────────────────────┘
```

### 1.2 문제점

| 문제 | 영향 |
|------|------|
| 프롬프트 복잡도 | 이해하기 어렵고 일관성 저하 |
| 작업 간 간섭 | 한 작업의 결과가 다른 작업 품질에 영향 |
| 디버깅 어려움 | 문제 발생 시 원인 추적 곤란 |
| 부분 실패 | 한 작업 실패 시 전체 재시도 필요 |

---

## 2. 개선 전략

### 2.1 역할별 분리

```
개선: N 호출 = N 역할 (각각 1개)
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│ Refiner    │→ │ Matcher    │→ │ Reviewer   │→ │ Summarizer │
│ (1회/PR)   │  │ (1회/파일) │  │ (1회/파일) │  │ (1회/PR)   │
└────────────┘  └────────────┘  └────────────┘  └────────────┘
```

### 2.2 호출 횟수 비교

| 시나리오 | 현재 | 개선 후 | 증가율 |
|----------|------|---------|--------|
| 5파일 PR | 5회 | 12회 | 2.4x |
| 10파일 PR | 10회 | 22회 | 2.2x |
| 20파일 PR | 20회 | 42회 | 2.1x |

**계산식**:
- 현재: `파일 수`
- 개선: `1 + (파일 수 × 2) + 1` = `2 + 파일 수 × 2`

---

## 3. 최적화 기법

### 3.1 병렬 처리

독립적인 파일들은 동시 처리:

```python
async def process_files_parallel(files: List[FileInfo], refined_spec: RefinedSpec):
    # Description Refiner는 이미 완료됨 (1회)

    # 파일별 처리를 병렬로
    tasks = []
    for file in files:
        task = process_single_file(file, refined_spec)
        tasks.append(task)

    # 동시 실행 (최대 동시성 제한)
    results = await asyncio.gather(*tasks, limit=4)

    # Summary Generator (1회)
    summary = await generate_summary(results)

    return summary
```

**효과**:
- 5파일 PR 기준: 순차 12회 → 병렬 시 3-4 라운드로 감소
- 전체 소요 시간 50-70% 단축 가능

### 3.2 조건부 스킵

불필요한 단계 건너뛰기:

```python
def should_skip_matching(file: FileInfo, refined_spec: RefinedSpec) -> bool:
    """매칭 단계 스킵 조건"""
    # 1. 단순 코멘트 변경
    if is_comment_only_change(file.diff):
        return True

    # 2. 명세와 무관한 파일
    if not has_potential_match(file.name, refined_spec):
        return True

    return False

def should_skip_full_review(file: FileInfo, match_result: MatchResult) -> bool:
    """상세 리뷰 스킵 조건"""
    # 1. 변경량이 매우 작음 (< 5 라인)
    if file.additions + file.deletions < 5:
        return True

    # 2. 매칭 결과가 없고 예상치 못한 변경도 없음
    if not match_result.matches and not match_result.unexpected_changes:
        return True

    return False
```

**스킵 가능 케이스**:
| 케이스 | 스킵 대상 | 대체 처리 |
|--------|-----------|-----------|
| 코멘트만 변경 | Matcher, Reviewer | 간단 요약만 |
| 설정 파일 | Reviewer | 형식 검증만 |
| 테스트 파일 | 전체 또는 간소화 | 선택적 처리 |

### 3.3 결과 캐싱

동일 입력에 대한 재사용:

```python
class CachedRefiner:
    def __init__(self):
        self._cache = {}

    def process(self, pr_description: str) -> RefinedSpec:
        cache_key = hashlib.md5(pr_description.encode()).hexdigest()

        if cache_key in self._cache:
            return self._cache[cache_key]

        result = self._call_llm(pr_description)
        self._cache[cache_key] = result

        return result
```

**캐싱 대상**:
| 컴포넌트 | 캐싱 가능성 | 이유 |
|----------|-------------|------|
| Description Refiner | 높음 | PR당 1회, 동일 입력 |
| Change Matcher | 중간 | 동일 파일-명세 조합 |
| Code Reviewer | 낮음 | 컨텍스트 의존적 |
| Summary Generator | 낮음 | 항상 다른 입력 |

### 3.4 배치 결합

작은 파일들 묶어서 처리:

```python
def batch_small_files(files: List[FileInfo], max_tokens: int = 3000) -> List[List[FileInfo]]:
    """작은 파일들을 배치로 그룹화"""
    batches = []
    current_batch = []
    current_tokens = 0

    for file in sorted(files, key=lambda f: count_tokens(f.diff)):
        file_tokens = count_tokens(file.diff)

        if file_tokens > max_tokens * 0.5:
            # 큰 파일은 개별 처리
            batches.append([file])
        elif current_tokens + file_tokens <= max_tokens:
            current_batch.append(file)
            current_tokens += file_tokens
        else:
            if current_batch:
                batches.append(current_batch)
            current_batch = [file]
            current_tokens = file_tokens

    if current_batch:
        batches.append(current_batch)

    return batches
```

---

## 4. 하이브리드 전략

### 4.1 컴포넌트 결합 옵션

일부 컴포넌트는 결합 처리 가능:

**옵션 A: 완전 분리 (권장)**
```
Refiner → Matcher → Reviewer → Summarizer
(별개 호출)
```

**옵션 B: Matcher + Reviewer 결합**
```
Refiner → [Matcher + Reviewer] → Summarizer
(매칭과 리뷰를 한 호출에)
```

**옵션 C: 상황별 선택**
```python
def choose_strategy(file: FileInfo, refined_spec: RefinedSpec) -> str:
    # 큰 변경은 분리
    if file.additions + file.deletions > 100:
        return "separated"

    # 명세와 관련 높은 파일은 분리 (정확도 우선)
    if has_high_relevance(file.name, refined_spec):
        return "separated"

    # 그 외는 결합
    return "combined"
```

### 4.2 전략별 트레이드오프

| 전략 | 호출 수 | 품질 | 비용 | 복잡도 |
|------|---------|------|------|--------|
| 완전 분리 | 많음 | 높음 | 높음 | 낮음 |
| 부분 결합 | 중간 | 중간 | 중간 | 중간 |
| 조건부 | 상황별 | 높음 | 중간 | 높음 |

---

## 5. 구현 가이드

### 5.1 파이프라인 설정

```yaml
# config/pipeline.yaml
optimization:
  parallel:
    enabled: true
    max_workers: 4

  skip:
    comment_only_files: true
    small_changes_threshold: 5
    unrelated_files: true

  cache:
    refiner:
      enabled: true
      ttl_seconds: 3600
    matcher:
      enabled: false

  batch:
    enabled: true
    max_tokens_per_batch: 3000
    min_files_to_batch: 2

  hybrid:
    combine_matcher_reviewer: false
    adaptive_strategy: false
```

### 5.2 모니터링 메트릭

```python
@dataclass
class OptimizationMetrics:
    total_files: int
    actual_calls: int
    theoretical_calls: int  # 최적화 없이
    skipped_calls: int
    cache_hits: int
    batched_files: int
    parallel_batches: int

    @property
    def call_reduction_rate(self) -> float:
        return 1 - (self.actual_calls / self.theoretical_calls)

    @property
    def cache_hit_rate(self) -> float:
        return self.cache_hits / (self.cache_hits + self.actual_calls)
```

---

## 6. 예상 효과

### 6.1 호출 횟수 감소 (조건부 스킵 적용)

| PR 유형 | 파일 수 | 이론적 호출 | 최적화 후 | 감소율 |
|---------|---------|-------------|-----------|--------|
| 소규모 | 3 | 8 | 5-6 | 25-37% |
| 중규모 | 10 | 22 | 14-18 | 18-36% |
| 대규모 | 30 | 62 | 35-45 | 27-43% |

### 6.2 처리 시간 감소 (병렬 처리 적용)

병렬화로 호출 수 증가에도 전체 시간 감소 가능:

```
순차 처리: 호출 수 × 평균 응답 시간
병렬 처리: ceil(호출 수 / 병렬도) × 평균 응답 시간

예시 (10파일, 22호출, 평균 3초, 병렬도 4):
- 순차: 22 × 3 = 66초
- 병렬: ceil(22/4) × 3 = 18초
- 감소율: 73%
```

---

## 7. 마이그레이션 계획

### 7.1 단계별 적용

1. **Phase 1**: 역할 분리 (컴포넌트화)
   - 프롬프트 분리
   - 인터페이스 정의
   - 기본 파이프라인 구현

2. **Phase 2**: 병렬 처리 도입
   - 비동기 처리 구현
   - 동시성 제한 설정

3. **Phase 3**: 최적화 기법 적용
   - 조건부 스킵 구현
   - 캐싱 구현
   - 배치 처리 구현

4. **Phase 4**: 모니터링 및 튜닝
   - 메트릭 수집
   - 임계값 조정
   - 성능 최적화

---

## 8. 듀얼 모델 병렬 처리 전략

### 8.1 모델 특성 비교

| 항목 | QWEN3 (주) | GPT-OSS (보조) |
|------|-----------|----------------|
| 컨텍스트 | 256K | 128K+ |
| 동시성 | 8개 | 50개/분 |
| 품질 | 기준 | 동등 수준 |
| 특이사항 | 중국어 출력 가능성 | - |

### 8.2 현재 병목 분석

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

### 8.3 듀얼 모델 병렬 전략

#### 전략 A: 역할 분담 (권장)

```
┌─────────────────────────────────────────────────────────────────┐
│                     Description Refiner                          │
│                        (QWEN3, 1회)                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                              ▼
┌─────────────────────────┐    ┌─────────────────────────┐
│       QWEN3             │    │       GPT-OSS           │
│   (8개 동시 처리)       │    │   (최대 50개/분)        │
│                         │    │                         │
│  ┌───────────────────┐  │    │  ┌───────────────────┐  │
│  │ Change Matcher    │  │    │  │ Review Validator  │  │
│  │      +            │  │    │  │ (Stage A + B)     │  │
│  │ Code Reviewer     │  │    │  │                   │  │
│  └───────────────────┘  │    │  └───────────────────┘  │
└────────────┬────────────┘    └────────────┬────────────┘
             │                              │
             └──────────────┬───────────────┘
                            ▼
              ┌─────────────────────────────┐
              │     Summary Generator       │
              │        (QWEN3, 1회)         │
              └─────────────────────────────┘
```

**처리 흐름**:
1. QWEN3: Matcher + Reviewer 결합 처리 (8개 동시)
2. 리뷰 완료된 파일은 즉시 GPT-OSS에 전달
3. GPT-OSS: Validator 처리 (파이프라인 방식)

**시간 계산 (50 파일)**:
```python
# QWEN3: Matcher + Reviewer (8 동시, 100s/파일)
review_batches = ceil(50 / 8)  # = 7 배치
review_time = 7 * 100  # = 700s

# GPT-OSS: Validator (파이프라인으로 리뷰와 겹침)
# 첫 리뷰 결과는 100s 후 도착, 이후 연속 도착
# GPT-OSS는 충분한 처리량 (50/min > 필요량)
# 마지막 배치 검증 시간만 추가
validator_additional = 100s  # 마지막 배치 검증

# 총 시간
total = review_time + validator_additional  # = 800s ≈ 13분
```

아직 목표(10분) 초과 → 추가 최적화 필요

#### 전략 B: 부하 분산 (확장)

```
┌─────────────────────────────────────────────────────────────────┐
│                     Description Refiner                          │
│                        (QWEN3, 1회)                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │  그룹 A     │     │  그룹 B     │     │  그룹 C     │
  │ (17 파일)   │     │ (17 파일)   │     │ (16 파일)   │
  │             │     │             │     │             │
  │ QWEN3      │     │ QWEN3      │     │ GPT-OSS    │
  │ Review     │     │ Review     │     │ Review     │
  │     ↓       │     │     ↓       │     │     ↓       │
  │ GPT-OSS    │     │ GPT-OSS    │     │ GPT-OSS    │
  │ Validate   │     │ Validate   │     │ Validate   │
  └─────────────┘     └─────────────┘     └─────────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             ▼
              ┌─────────────────────────────┐
              │     Summary Generator       │
              │        (QWEN3, 1회)         │
              └─────────────────────────────┘
```

**처리량 계산**:
```python
# QWEN3 리뷰 용량: 8 동시 × 6배치(분) = 48개/10분
# GPT-OSS 리뷰 용량: 50개/분 × 10분 = 500개 (제한 없음)
# GPT-OSS 검증 용량: 충분

# 파일 배분
qwen3_files = 34  # 8 동시 × ceil(34/8) × 100s = 500s
gptoss_files = 16  # 16개 × 100s / 병렬 = ~200s

# 최종 예상 시간
# QWEN3 리뷰: 500s
# + 마지막 검증: 100s
# = 600s = 10분 ✓
```

### 8.4 파이프라인 최적화

#### 8.4.1 스트리밍 파이프라인

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
            # Matcher + Reviewer 결합 호출
            result = await self.qwen_client.combined_review(file, spec)

            # 즉시 검증 큐에 추가
            await self.review_queue.put(result)

            return result

    async def validation_worker(self):
        validated = []
        while True:
            result = await self.review_queue.get()
            if result is None:  # 종료 신호
                break

            # GPT-OSS로 검증
            async with self.gptoss_rate_limiter:
                validated_result = await self.gptoss_client.validate(result)
                validated.append(validated_result)

        return validated
```

#### 8.4.2 동적 부하 분산

```python
class AdaptiveLoadBalancer:
    """실시간 응답 시간 기반 동적 분산"""

    def __init__(self):
        self.qwen_avg_time = 100.0
        self.gptoss_avg_time = 100.0
        self.file_queue = []

    def assign_model(self, file: FileInfo) -> str:
        """파일별 최적 모델 할당"""

        # 큰 파일은 컨텍스트 큰 QWEN3
        if file.total_tokens > 100000:
            return "qwen3"

        # 현재 부하 기반 결정
        qwen_pending = self.qwen_pending_count
        gptoss_pending = self.gptoss_pending_count

        qwen_expected_time = qwen_pending * self.qwen_avg_time / 8
        gptoss_expected_time = gptoss_pending * self.gptoss_avg_time

        if qwen_expected_time < gptoss_expected_time:
            return "qwen3"
        else:
            return "gptoss"
```

### 8.5 컴포넌트 결합 전략

대용량 컨텍스트(256K/128K+) 활용하여 호출 횟수 감소:

#### 8.5.1 결합 가능 컴포넌트

```
Option 1: Matcher + Reviewer 결합 (권장)
┌──────────────────────────────────────────┐
│  Combined Match & Review                 │
│                                          │
│  Input:                                  │
│    - RefinedSpec                         │
│    - File Diff                           │
│    - Function Code                       │
│                                          │
│  Output:                                 │
│    - MatchResult (embedded)              │
│    - ReviewResult                        │
│                                          │
│  장점: 리뷰 시 매칭 정보 활용 용이       │
│  단점: 프롬프트 복잡도 증가              │
└──────────────────────────────────────────┘

Option 2: Validator 2단계 결합
┌──────────────────────────────────────────┐
│  Combined Validation                     │
│                                          │
│  Stage A + B 통합:                       │
│    - 이슈 검증                           │
│    - 라인 위치 추출                      │
│                                          │
│  이슈 수가 적을 때 (≤5개) 적용          │
└──────────────────────────────────────────┘
```

#### 8.5.2 결합 시 호출 횟수 비교

| 시나리오 | 완전 분리 | Matcher+Reviewer 결합 | 감소율 |
|----------|----------|----------------------|--------|
| 10 파일 | 1 + 10×3 + 1 = 32회 | 1 + 10×2 + 1 = 22회 | 31% |
| 30 파일 | 1 + 30×3 + 1 = 92회 | 1 + 30×2 + 1 = 62회 | 33% |
| 50 파일 | 1 + 50×3 + 1 = 152회 | 1 + 50×2 + 1 = 102회 | 33% |

### 8.6 최종 권장 전략

```
┌─────────────────────────────────────────────────────────────────┐
│                    PR 처리 시작                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: Description Refiner (QWEN3, 1회, ~10s)                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: 파일 분류 및 배분                                       │
│                                                                  │
│  규칙:                                                           │
│    - 대형 파일 (>100K tokens): QWEN3 전용                        │
│    - 중형 파일 (10K-100K tokens): 부하 기반 동적 분배            │
│    - 소형 파일 (<10K tokens): 배치 결합 후 GPT-OSS               │
│    - 스킵 대상 (코멘트만, 설정파일): 리뷰 생략                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                ┌────────────┼────────────┐
                ▼            ▼            ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐
│    QWEN3 Pool    │ │ GPT-OSS Pool │ │    Skip Pool     │
│   (8 동시)       │ │ (50/min)     │ │   (No LLM)       │
│                  │ │              │ │                  │
│ Match+Review     │ │ Match+Review │ │  기본 정보만     │
│ (결합 호출)      │ │ (결합 호출)  │ │  생성            │
└────────┬─────────┘ └──────┬───────┘ └────────┬─────────┘
         │                  │                  │
         └──────────────────┼──────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: Validation (GPT-OSS, 스트리밍 파이프라인)               │
│                                                                  │
│    - 리뷰 완료 즉시 검증 시작                                    │
│    - 이슈 ≤5개: Stage A+B 결합                                   │
│    - 이슈 >5개: Stage A → B 분리                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 4: Summary Generator (QWEN3, 1회, ~20s)                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     PR 처리 완료                                  │
│                    (목표: <10분)                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.7 예상 성능 (50 파일 PR)

| 단계 | 처리 방식 | 예상 시간 |
|------|----------|----------|
| Refiner | QWEN3 1회 | 10s |
| 파일 분류 | 로컬 처리 | <1s |
| Match+Review (40개) | QWEN3 8동시 | ceil(40/8)×100s = 500s |
| Match+Review (10개) | GPT-OSS 동시 | (병렬로 QWEN과 겹침) |
| Validation | GPT-OSS 스트리밍 | (파이프라인으로 대부분 겹침) +100s |
| Summary | QWEN3 1회 | 20s |
| **총계** | - | **~630s ≈ 10.5분** |

**추가 최적화로 10분 달성**:
- 소형 파일 배치 처리: 10개→3배치 = 호출 30% 감소
- 스킵 대상 파일 제외: 평균 10-20% 파일 스킵
- 캐싱: 유사 PR 재처리 시 Refiner 스킵

### 8.8 구현 설정

```yaml
# config/dual-model.yaml
models:
  primary:
    name: "qwen3"
    max_concurrent: 8
    context_limit: 256000
    use_for:
      - refiner
      - large_file_review
      - summary

  secondary:
    name: "gptoss"
    rate_limit: 50  # per minute
    context_limit: 128000
    use_for:
      - small_file_review
      - validation
      - batch_review

pipeline:
  strategy: "streaming"  # streaming, batch, adaptive

  file_assignment:
    large_file_threshold: 100000  # tokens
    small_file_threshold: 10000   # tokens
    batch_max_files: 5
    batch_max_tokens: 50000

  skip_conditions:
    comment_only: true
    config_files: true
    generated_files: true
    test_files: "optional"  # true, false, optional

  component_combining:
    matcher_reviewer: true
    validator_stages: "adaptive"  # combined, separated, adaptive

timeout:
  pr_max_seconds: 600  # 10분
  file_max_seconds: 120
  component_max_seconds: 60
```

---

## 9. 관련 문서

- 아키텍처: [02-architecture.md](../02-architecture.md)
- COT 개선: [cot-enhancement.md](cot-enhancement.md)
- 환경 제약: [03-current-environment.md](../03-current-environment.md)
