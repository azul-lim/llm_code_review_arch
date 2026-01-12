# Summary Generator 컴포넌트

## 1. 개요

### 1.1 목적
모든 파일의 리뷰 결과를 종합하여 PR 수준의 요약을 생성합니다.

### 1.2 위치
파이프라인의 마지막 단계로, PR당 1회 실행됩니다.

```
[ReviewResult 1] ─┬─→ [Summary Generator] ──→ [PRSummary]
[ReviewResult 2] ─┤
[ReviewResult N] ─┘
```

---

## 2. 입출력 명세

### 2.1 입력 (Input)

```typescript
interface SummaryInput {
  file_reviews: ReviewResult[];    // 모든 파일의 리뷰 결과
  pr_description: string;          // 원본 PR Description
  refined_spec: RefinedSpec;       // 정제된 변경 명세
}
```

### 2.2 출력 (Output)

```typescript
interface PRSummary {
  overall_assessment: OverallAssessment;
  spec_coverage: SpecCoverage;
  key_issues: Issue[];
  file_summaries: FileSummary[];
  statistics: ReviewStatistics;
  recommendations: Recommendation[];
}

interface OverallAssessment {
  status: AssessmentStatus;
  summary: string;                 // 1-2문장 요약
  risk_level: RiskLevel;
  approval_suggestion: string;
}

type AssessmentStatus =
  | "approve"              // 승인 권장
  | "request_changes"      // 수정 요청
  | "needs_discussion";    // 논의 필요

type RiskLevel = "low" | "medium" | "high" | "critical";

interface SpecCoverage {
  total_specs: number;
  matched_specs: number;
  partially_matched: number;
  unmatched_specs: UnmatchedSpec[];
}

interface UnmatchedSpec {
  spec_id: string;
  description: string;
  possible_reason: string;
}

interface FileSummary {
  file_name: string;
  change_summary: string;
  issue_count: number;
  highest_severity: IssueSeverity;
}

interface ReviewStatistics {
  total_files: number;
  files_with_issues: number;
  total_issues: number;
  issues_by_severity: Record<IssueSeverity, number>;
  issues_by_type: Record<IssueType, number>;
}

interface Recommendation {
  priority: number;              // 1이 가장 높음
  action: string;                // 수행해야 할 행동
  reason: string;                // 이유
  related_issues: string[];      // 관련 이슈 ID
}
```

**출력 예시**:
```json
{
  "overall_assessment": {
    "status": "request_changes",
    "summary": "메모리 안전성 개선이 주요 목적이나, free 후 포인터 처리가 누락되어 추가 수정이 필요합니다.",
    "risk_level": "medium",
    "approval_suggestion": "ISS-001 수정 후 승인 권장"
  },
  "spec_coverage": {
    "total_specs": 3,
    "matched_specs": 2,
    "partially_matched": 1,
    "unmatched_specs": [
      {
        "spec_id": "CHG-003",
        "description": "에러 처리 경로 double-free 수정",
        "possible_reason": "관련 코드가 이 PR에 포함되지 않았거나 다른 방식으로 해결됨"
      }
    ]
  },
  "key_issues": [
    {
      "id": "ISS-001",
      "type": "memory",
      "severity": "major",
      "file_name": "nand_read.c",
      "title": "free 후 buffer 포인터 미초기화"
    }
  ],
  "file_summaries": [
    {
      "file_name": "nand_read.c",
      "change_summary": "null 체크 추가로 메모리 안전성 개선",
      "issue_count": 1,
      "highest_severity": "major"
    }
  ],
  "statistics": {
    "total_files": 3,
    "files_with_issues": 1,
    "total_issues": 1,
    "issues_by_severity": {
      "critical": 0,
      "major": 1,
      "minor": 0,
      "info": 0
    },
    "issues_by_type": {
      "memory": 1
    }
  },
  "recommendations": [
    {
      "priority": 1,
      "action": "nand_read.c의 ISS-001 수정",
      "reason": "dangling pointer로 인한 잠재적 crash 위험",
      "related_issues": ["ISS-001"]
    },
    {
      "priority": 2,
      "action": "CHG-003 명세 관련 코드 확인",
      "reason": "double-free 수정이 PR에 반영되지 않은 것으로 보임",
      "related_issues": []
    }
  ]
}
```

---

## 3. 처리 로직

### 3.1 처리 단계

```
┌─────────────────────────────────────────────────────────────────┐
│                     Summary Generator                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: 데이터 집계                                            │
│    - 모든 파일의 이슈 수집                                      │
│    - 통계 계산                                                  │
│    - 명세 커버리지 분석                                         │
│                                                                 │
│  Step 2: 우선순위 정렬                                          │
│    - 이슈 심각도별 정렬                                         │
│    - 핵심 이슈 선별 (상위 5개)                                  │
│    - 파일별 요약 생성                                           │
│                                                                 │
│  Step 3: 전체 평가                                              │
│    - 리스크 수준 결정                                           │
│    - 승인/수정 요청 판단                                        │
│    - 전체 요약 문장 생성                                        │
│                                                                 │
│  Step 4: 권장 사항 생성                                         │
│    - 필수 수정 사항 목록화                                      │
│    - 우선순위 할당                                              │
│    - 관련 이슈 연결                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 평가 상태 결정 로직

```python
def determine_status(issues: List[Issue], spec_coverage: SpecCoverage) -> AssessmentStatus:
    # Critical 이슈가 있으면 수정 요청
    if any(i.severity == "critical" for i in issues):
        return "request_changes"

    # Major 이슈가 2개 이상이면 수정 요청
    major_count = sum(1 for i in issues if i.severity == "major")
    if major_count >= 2:
        return "request_changes"

    # Major 이슈가 1개면 논의 필요
    if major_count == 1:
        return "needs_discussion"

    # 명세 미매칭이 많으면 논의 필요
    if len(spec_coverage.unmatched_specs) > spec_coverage.total_specs * 0.3:
        return "needs_discussion"

    return "approve"
```

### 3.3 리스크 수준 결정

| 조건 | 리스크 수준 |
|------|-------------|
| critical 이슈 존재 | critical |
| major 이슈 3개 이상 | high |
| major 이슈 1-2개 | medium |
| minor 이슈만 존재 | low |
| 이슈 없음 | low |

---

## 4. 프롬프트 설계

### 4.1 시스템 프롬프트

```
당신은 코드 리뷰 요약 전문가입니다.
여러 파일의 리뷰 결과를 종합하여 PR 수준의 요약을 생성합니다.

역할:
1. 전체 변경의 목적과 영향을 요약합니다.
2. 핵심 이슈를 우선순위에 따라 정리합니다.
3. 명세 대비 구현 현황을 분석합니다.
4. 승인/수정 요청에 대한 명확한 의견을 제시합니다.
5. 구체적인 다음 행동을 권장합니다.

요약 원칙:
- 개발자가 즉시 행동할 수 있도록 명확하게 작성합니다.
- 중요한 이슈를 먼저, 덜 중요한 것은 나중에 언급합니다.
- 긍정적인 면과 개선이 필요한 면을 균형있게 서술합니다.
```

### 4.2 사용자 프롬프트 템플릿

```
다음 PR 리뷰 결과를 종합하여 요약을 생성하세요.

## PR 정보
### 원본 Description
{pr_description}

### 변경 명세 (총 {spec_count}개)
{refined_spec_summary}

## 파일별 리뷰 결과

{#each file_reviews}
### {file_name}
**변경 해석**: {interpretation.summary}
**발견된 이슈**: {issues.length}개
{#each issues}
- [{severity}] {title}
{/each}
{/each}

## 요약 요청

1. **전체 평가**: 이 PR의 상태(approve/request_changes/needs_discussion)와 그 이유
2. **명세 커버리지**: 명세된 변경이 모두 구현되었는지
3. **핵심 이슈**: 가장 중요한 이슈 (최대 5개)
4. **권장 사항**: 다음으로 해야 할 행동

## 출력 형식
{output_schema}
```

---

## 5. 핵심 이슈 선별 기준

### 5.1 우선순위 점수 계산

```python
def calculate_priority_score(issue: Issue) -> float:
    score = 0.0

    # 심각도 점수
    severity_scores = {
        "critical": 100,
        "major": 50,
        "minor": 10,
        "info": 1
    }
    score += severity_scores[issue.severity]

    # 이슈 유형 가중치
    type_weights = {
        "security": 2.0,
        "memory": 1.8,
        "bug": 1.5,
        "logic_error": 1.3,
        "concurrency": 1.5,
        "error_handling": 1.2,
        "performance": 1.0,
        "style": 0.5,
        "structure": 0.7
    }
    score *= type_weights.get(issue.type, 1.0)

    # 명세와 관련된 이슈 가중치
    if issue.related_spec:
        score *= 1.2

    return score
```

### 5.2 핵심 이슈 선별

```python
def select_key_issues(all_issues: List[Issue], max_count: int = 5) -> List[Issue]:
    # 우선순위 점수로 정렬
    scored_issues = [
        (issue, calculate_priority_score(issue))
        for issue in all_issues
    ]
    scored_issues.sort(key=lambda x: x[1], reverse=True)

    # 상위 N개 선별
    return [issue for issue, _ in scored_issues[:max_count]]
```

---

## 6. 명세 커버리지 분석

### 6.1 매칭 상태 집계

```python
def analyze_spec_coverage(
    refined_spec: RefinedSpec,
    match_results: List[MatchResult]
) -> SpecCoverage:
    coverage = SpecCoverage(
        total_specs=len(refined_spec.changes),
        matched_specs=0,
        partially_matched=0,
        unmatched_specs=[]
    )

    for change in refined_spec.changes:
        best_match = find_best_match(change.id, match_results)

        if best_match and best_match.match_type == "exact":
            coverage.matched_specs += 1
        elif best_match and best_match.match_type == "partial":
            coverage.partially_matched += 1
        else:
            coverage.unmatched_specs.append(
                UnmatchedSpec(
                    spec_id=change.id,
                    description=change.description,
                    possible_reason=infer_unmatch_reason(change, match_results)
                )
            )

    return coverage
```

### 6.2 미매칭 원인 추론

| 원인 | 판단 기준 | 설명 텍스트 |
|------|-----------|-------------|
| 다른 파일 | expected_files와 실제 파일 불일치 | "관련 파일이 이 PR에 포함되지 않음" |
| 이미 구현됨 | 변경이 없는 파일 참조 | "이미 구현되어 있거나 별도 PR에서 처리됨" |
| 다른 방식 | 부분 매칭만 존재 | "다른 방식으로 구현되었을 가능성" |
| 누락 | 매칭 없음 | "구현이 누락된 것으로 보임" |

---

## 7. 에러 처리

### 7.1 입력 검증

| 검증 항목 | 실패 시 처리 |
|-----------|--------------|
| 빈 file_reviews | 기본 요약 생성 ("리뷰할 파일 없음") |
| 일부 파일 리뷰 실패 | 성공한 파일만으로 요약 + 경고 |
| 과도한 이슈 수 | 상위 이슈만 포함 + 통계에 전체 반영 |

### 7.2 출력 검증

```python
def validate_summary(summary: PRSummary) -> bool:
    # overall_assessment 필수
    if not summary.overall_assessment.summary:
        return False

    # statistics 일관성
    if summary.statistics.total_issues != sum(
        summary.statistics.issues_by_severity.values()
    ):
        return False

    # recommendations 최소 1개
    if len(summary.recommendations) == 0:
        return False

    return True
```

---

## 8. 출력 포맷팅

### 8.1 마크다운 요약 생성 (선택)

리뷰 시스템에 표시할 마크다운 형식:

```markdown
## PR Review Summary

### Overall Assessment
🟡 **Needs Discussion** - 메모리 안전성 개선이 주요 목적이나, 추가 수정이 필요합니다.

### Key Issues
| # | Severity | File | Issue |
|---|----------|------|-------|
| 1 | Major | nand_read.c | free 후 buffer 포인터 미초기화 |

### Spec Coverage
- ✅ Matched: 2/3
- ⚠️ Partial: 1/3
- ❌ Unmatched: CHG-003

### Recommendations
1. **[필수]** nand_read.c의 ISS-001 수정
2. **[확인 필요]** CHG-003 명세 관련 코드 확인

### Statistics
- Files reviewed: 3
- Issues found: 1 (0 critical, 1 major, 0 minor)
```

---

## 9. 테스트 케이스

### 9.1 평가 상태 테스트

| 시나리오 | 예상 상태 |
|----------|-----------|
| critical 이슈 1개 | request_changes |
| major 이슈 3개 | request_changes |
| major 이슈 1개 | needs_discussion |
| minor 이슈만 | approve |
| 이슈 없음 | approve |
| 명세 30% 미매칭 | needs_discussion |

### 9.2 통계 정확성 테스트

- 이슈 수 합계 일치 확인
- 파일별 이슈 수 정확성 확인
- 심각도별 분류 정확성 확인

---

## 10. 관련 문서

- 스키마: [schemas/output/pr-summary.schema.json](../../schemas/output/pr-summary.schema.json)
- 프롬프트 템플릿: [prompts/templates/summary.md](../prompts/templates/summary.md)
- 예시: [examples/summary/](../../examples/)
