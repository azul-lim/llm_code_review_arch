# 개선 시스템 흐름

## 개요

개선된 시스템은 역할별로 분리된 컴포넌트와 파이프라인 구조를 사용합니다.

## 시퀀스 다이어그램

```mermaid
sequenceDiagram
    participant RS as Review System
    participant PL as Pipeline
    participant REF as Description Refiner
    participant MAT as Change Matcher
    participant REV as Code Reviewer
    participant VAL as Review Validator
    participant SUM as Summary Generator
    participant LLM as LLM

    RS->>PL: PR 정보 전달

    %% Step 1: Description 정제
    PL->>REF: PR Description
    REF->>LLM: 정제 요청
    LLM-->>REF: RefinedSpec
    REF-->>PL: 구조화된 명세

    %% Step 2-4: 파일별 처리 (병렬)
    par 파일별 병렬 처리
        PL->>MAT: File 1 + RefinedSpec
        MAT->>LLM: 매칭 분석
        LLM-->>MAT: MatchResult
        MAT-->>PL: 매칭 결과 1

        PL->>REV: MatchResult 1 + Code
        REV->>LLM: 리뷰 요청
        LLM-->>REV: ReviewResult
        REV-->>PL: 리뷰 결과 1

        PL->>VAL: ReviewResult 1 + Context
        VAL->>LLM: 검증 요청 (Stage A)
        LLM-->>VAL: Validation Result
        VAL->>LLM: 위치 추출 (Stage B)
        LLM-->>VAL: Inline Positions
        VAL-->>PL: ValidatedReview 1
    and
        PL->>MAT: File N + RefinedSpec
        MAT->>LLM: 매칭 분석
        LLM-->>MAT: MatchResult
        MAT-->>PL: 매칭 결과 N

        PL->>REV: MatchResult N + Code
        REV->>LLM: 리뷰 요청
        LLM-->>REV: ReviewResult
        REV-->>PL: 리뷰 결과 N

        PL->>VAL: ReviewResult N + Context
        VAL->>LLM: 검증 + 위치 추출
        VAL-->>PL: ValidatedReview N
    end

    %% Step 5: 요약 생성
    PL->>SUM: 전체 ValidatedReviews
    SUM->>LLM: 요약 요청
    LLM-->>SUM: PRSummary
    SUM-->>PL: PR 요약

    PL-->>RS: 최종 결과 반환
```

## 파이프라인 아키텍처

```mermaid
flowchart TB
    subgraph Input["입력"]
        PR[PR Description]
        FILES[Files + Diffs]
    end

    subgraph Pipeline["Review Pipeline"]
        subgraph Stage1["Stage 1: 정제"]
            REF[Description Refiner]
        end

        subgraph Stage2["Stage 2-4: 파일별 처리"]
            subgraph Parallel["병렬 처리"]
                subgraph File1["File 1"]
                    MAT1[Change Matcher]
                    REV1[Code Reviewer]
                    VAL1[Review Validator]
                    MAT1 --> REV1 --> VAL1
                end
                subgraph File2["File 2"]
                    MAT2[Change Matcher]
                    REV2[Code Reviewer]
                    VAL2[Review Validator]
                    MAT2 --> REV2 --> VAL2
                end
                subgraph FileN["File N"]
                    MATN[Change Matcher]
                    REVN[Code Reviewer]
                    VALN[Review Validator]
                    MATN --> REVN --> VALN
                end
            end
        end

        subgraph Stage3["Stage 5: 요약"]
            SUM[Summary Generator]
        end
    end

    subgraph Output["출력"]
        REVIEWS[Validated Reviews]
        SUMMARY[PR Summary]
    end

    PR --> REF
    REF --> Stage2
    FILES --> Stage2
    Stage2 --> SUM
    VAL1 --> REVIEWS
    VAL2 --> REVIEWS
    VALN --> REVIEWS
    SUM --> SUMMARY

    style Pipeline fill:#e6ffe6,stroke:#009900
    style Parallel fill:#e6f3ff,stroke:#0066cc
```

## 컴포넌트 상세

```mermaid
flowchart LR
    subgraph Refiner["Description Refiner"]
        R_IN[PR Description]
        R_PROC[구조 분석 &<br/>항목 추출]
        R_OUT[RefinedSpec]
        R_IN --> R_PROC --> R_OUT
    end

    subgraph Matcher["Change Matcher"]
        M_IN1[RefinedSpec]
        M_IN2[File Diff]
        M_PROC[매칭 분석]
        M_OUT[MatchResult]
        M_IN1 --> M_PROC
        M_IN2 --> M_PROC
        M_PROC --> M_OUT
    end

    subgraph Reviewer["Code Reviewer"]
        RV_IN1[MatchResult]
        RV_IN2[Function Code]
        RV_PROC[COT 분석 &<br/>버그 탐지]
        RV_OUT[ReviewResult]
        RV_IN1 --> RV_PROC
        RV_IN2 --> RV_PROC
        RV_PROC --> RV_OUT
    end

    subgraph Validator["Review Validator"]
        V_IN1[ReviewResult]
        V_IN2[Context]
        V_PROC[검증 &<br/>위치 추출]
        V_OUT[ValidatedReview]
        V_IN1 --> V_PROC
        V_IN2 --> V_PROC
        V_PROC --> V_OUT
    end

    subgraph Summary["Summary Generator"]
        S_IN[ValidatedReviews]
        S_PROC[종합 &<br/>우선순위화]
        S_OUT[PRSummary]
        S_IN --> S_PROC --> S_OUT
    end

    R_OUT --> M_IN1
    M_OUT --> RV_IN1
    RV_OUT --> V_IN1
    V_OUT --> S_IN

    style Refiner fill:#ffe6e6,stroke:#cc6666
    style Matcher fill:#e6e6ff,stroke:#6666cc
    style Reviewer fill:#e6ffe6,stroke:#66cc66
    style Validator fill:#fff0e6,stroke:#cc9966
    style Summary fill:#f0e6ff,stroke:#9966cc
```

## Review Validator 상세 흐름

```mermaid
flowchart TD
    subgraph Input["검증 입력"]
        I1[ReviewResult]
        I2[PR Description]
        I3[RefinedSpec]
        I4[File Diff]
        I5[Function Code]
    end

    subgraph PreFilter["사전 필터링 (LLM 없이)"]
        PF1{라인 범위<br/>유효?}
        PF2{인코딩<br/>정상?}
        PF3{필수 필드<br/>존재?}
        PF1 -->|No| FILTERED
        PF2 -->|No| FILTERED
        PF3 -->|No| FILTERED
    end

    subgraph StageA["Stage A: 이슈 검증"]
        A1[변경점 존재 확인]
        A2[설명 정확성 검토]
        A3[제안 코드 유효성]
        A4[할루시네이션 탐지]
        A1 --> A2 --> A3 --> A4
    end

    subgraph StageB["Stage B: 라인 위치 추출"]
        B1[code_snippet 매칭]
        B2[diff hunk 분석]
        B3[inline position 계산]
        B1 --> B2 --> B3
    end

    subgraph Output["검증 출력"]
        VALID[Validated Issues]
        FILTERED[Filtered Issues]
        POSITIONS[Inline Positions]
    end

    I1 --> PreFilter
    I2 --> StageA
    I3 --> StageA
    I4 --> StageA
    I5 --> StageA

    PreFilter -->|Pass| StageA
    StageA -->|Valid| StageB
    StageA -->|Invalid| FILTERED
    StageB --> VALID
    StageB --> POSITIONS

    style PreFilter fill:#fff0f0,stroke:#cc6666
    style StageA fill:#f0f0ff,stroke:#6666cc
    style StageB fill:#f0fff0,stroke:#66cc66
```

## 검증 체크 상세

```mermaid
flowchart LR
    subgraph Checks["검증 체크 항목"]
        C1["change_exists<br/>변경점 실제 존재"]
        C2["description_accurate<br/>설명 정확성"]
        C3["suggestion_valid<br/>제안 코드 유효성"]
        C4["encoding_ok<br/>인코딩 정상"]
        C5["not_hallucination<br/>할루시네이션 아님"]
        C6["line_range_valid<br/>라인 범위 유효"]
    end

    subgraph Result["결과"]
        ALL_PASS{모든 체크<br/>통과?}
        VALID[is_valid: true]
        INVALID[is_valid: false]
    end

    C1 --> ALL_PASS
    C2 --> ALL_PASS
    C3 --> ALL_PASS
    C4 --> ALL_PASS
    C5 --> ALL_PASS
    C6 --> ALL_PASS

    ALL_PASS -->|Yes| VALID
    ALL_PASS -->|No| INVALID

    style C1 fill:#e6f3ff,stroke:#0066cc
    style C2 fill:#e6f3ff,stroke:#0066cc
    style C3 fill:#e6f3ff,stroke:#0066cc
    style C4 fill:#e6f3ff,stroke:#0066cc
    style C5 fill:#e6f3ff,stroke:#0066cc
    style C6 fill:#e6f3ff,stroke:#0066cc
```

## 데이터 흐름 상세

```mermaid
flowchart TD
    subgraph Input["입력 데이터"]
        direction LR
        I1["PRInput"]
        I2["pr_description: string"]
        I3["files: FileInfo[]"]
        I1 --- I2
        I1 --- I3
    end

    subgraph Internal["내부 데이터"]
        direction LR
        INT1["RefinedSpec"]
        INT2["MatchResult[]"]
        INT3["ReviewResult[]"]
        INT4["ValidatedReview[]"]
    end

    subgraph Output["출력 데이터"]
        direction LR
        O1["PROutput"]
        O2["validated_reviews: ValidatedReview[]"]
        O3["summary: PRSummary"]
        O1 --- O2
        O1 --- O3
    end

    I1 -->|"Refiner"| INT1
    I3 -->|"Matcher"| INT2
    INT1 -->|"Matcher"| INT2
    INT2 -->|"Reviewer"| INT3
    INT3 -->|"Validator"| INT4
    INT4 -->|"Summarizer"| O1

    style Input fill:#e6f3ff,stroke:#0066cc
    style Internal fill:#fff0e6,stroke:#cc9966
    style Output fill:#e6ffe6,stroke:#009900
```

## 호출 최적화

```mermaid
flowchart TD
    subgraph Optimization["최적화 전략"]
        direction TB

        subgraph Parallel["병렬 처리"]
            P1[파일 1]
            P2[파일 2]
            P3[파일 N]
        end

        subgraph Skip["조건부 스킵"]
            SK1{코멘트만?}
            SK2{관련 없음?}
            SK3[스킵]
        end

        subgraph Cache["캐싱"]
            CA1[Refiner 결과]
            CA2[재사용]
        end

        subgraph ValidatorOpt["Validator 최적화"]
            VO1[사전 필터링]
            VO2{이슈 수<br/>적음?}
            VO3[1회 결합 호출]
            VO4[2단계 분리]
        end
    end

    SK1 -->|Yes| SK3
    SK2 -->|Yes| SK3
    CA1 --> CA2
    VO1 --> VO2
    VO2 -->|Yes| VO3
    VO2 -->|No| VO4

    style Parallel fill:#e6f3ff,stroke:#0066cc
    style Skip fill:#fff0f0,stroke:#cc6666
    style Cache fill:#f0fff0,stroke:#66cc66
    style ValidatorOpt fill:#fff0e6,stroke:#cc9966
```

## 현재 vs 개선 비교

```mermaid
flowchart LR
    subgraph Current["현재 시스템"]
        direction TB
        C1["1회 LLM 호출"]
        C2["4가지 역할 혼합"]
        C3["검증 없음 또는<br/>1회 호출에 포함"]
        C1 --> C2 --> C3
    end

    subgraph Improved["개선 시스템"]
        direction TB
        I1["5개 분리 컴포넌트"]
        I2["역할별 명확한 책임"]
        I3["독립된 Validator<br/>(2단계 분리)"]
        I1 --> I2 --> I3
    end

    Current -->|"개선"| Improved

    style Current fill:#ffeeee,stroke:#cc6666
    style Improved fill:#eeffee,stroke:#66cc66
```

## 듀얼 모델 병렬 처리

```mermaid
sequenceDiagram
    participant RS as Review System
    participant PL as Pipeline
    participant Q as QWEN3 (8동시)
    participant G as GPT-OSS (50/min)

    RS->>PL: PR 정보 전달

    %% Step 1: Description Refiner
    PL->>Q: Description 정제
    Q-->>PL: RefinedSpec

    %% Step 2: 파일 분류
    PL->>PL: 파일 분류<br/>(대형/중형/소형/스킵)

    %% Step 3: 병렬 리뷰 + 스트리밍 검증
    par QWEN3 리뷰 (대형+중형)
        loop 파일 배치 (8개씩)
            PL->>Q: Match+Review 결합 요청
            Q-->>PL: ReviewResult
            PL->>G: 즉시 검증 요청
        end
    and GPT-OSS 리뷰 (소형)
        loop 소형 파일 배치
            PL->>G: Match+Review 결합 요청
            G-->>PL: ReviewResult
            PL->>G: 검증 요청
        end
    and GPT-OSS 검증 (스트리밍)
        loop 리뷰 완료 즉시
            G-->>PL: ValidatedReview
        end
    end

    %% Step 4: Summary
    PL->>Q: Summary 요청
    Q-->>PL: PRSummary

    PL-->>RS: 최종 결과 반환
```

## 듀얼 모델 아키텍처

```mermaid
flowchart TB
    subgraph Input["입력"]
        PR[PR Description]
        FILES[Files + Diffs]
    end

    subgraph Classifier["파일 분류기 (로컬)"]
        CL1{파일 크기?}
        CL2[대형 >100K]
        CL3[중형 10K-100K]
        CL4[소형 <10K]
        CL5[스킵 대상]
    end

    subgraph QWEN["QWEN3 Pool (8 동시)"]
        REF[Description Refiner]
        QR1[Match+Review 1]
        QR2[Match+Review 2]
        QRN[Match+Review N]
        SUM[Summary Generator]
    end

    subgraph GPTOSS["GPT-OSS Pool (50/min)"]
        GR1[Match+Review<br/>(소형 파일)]
        VAL[Review Validator<br/>(스트리밍)]
    end

    subgraph Skip["스킵 처리"]
        SKIP[기본 정보만 생성<br/>(No LLM)]
    end

    subgraph Output["출력"]
        REVIEWS[Validated Reviews]
        SUMMARY[PR Summary]
    end

    PR --> REF
    REF --> Classifier
    FILES --> Classifier

    CL1 --> CL2 --> QR1
    CL1 --> CL2 --> QR2
    CL1 --> CL2 --> QRN
    CL1 --> CL3 --> QR1
    CL1 --> CL3 --> QRN
    CL1 --> CL4 --> GR1
    CL1 --> CL5 --> SKIP

    QR1 --> VAL
    QR2 --> VAL
    QRN --> VAL
    GR1 --> VAL

    VAL --> REVIEWS
    SKIP --> REVIEWS
    REVIEWS --> SUM
    SUM --> SUMMARY

    style QWEN fill:#e6f3ff,stroke:#0066cc
    style GPTOSS fill:#fff0e6,stroke:#cc9966
    style Skip fill:#f0f0f0,stroke:#999999
```

## 스트리밍 파이프라인 상세

```mermaid
flowchart LR
    subgraph Timeline["시간 흐름 (600초 목표)"]
        T0["0s"]
        T100["100s"]
        T200["200s"]
        T500["500s"]
        T600["600s"]
    end

    subgraph QWEN_Timeline["QWEN3 작업"]
        Q_REF["Refiner<br/>0-10s"]
        Q_B1["배치1<br/>10-110s"]
        Q_B2["배치2<br/>110-210s"]
        Q_BN["...<br/>배치5<br/>410-510s"]
        Q_SUM["Summary<br/>580-600s"]
    end

    subgraph GPTOSS_Timeline["GPT-OSS 작업"]
        G_R1["소형파일 리뷰<br/>10-200s"]
        G_V1["검증 시작<br/>110s~"]
        G_VN["검증 계속<br/>~580s"]
    end

    Q_REF --> Q_B1 --> Q_B2 --> Q_BN --> Q_SUM
    Q_B1 -.->|"결과 전달"| G_V1
    G_R1 -.->|"결과 전달"| G_V1
    G_V1 --> G_VN -.->|"완료"| Q_SUM

    style Timeline fill:#f0f0f0,stroke:#666666
    style QWEN_Timeline fill:#e6f3ff,stroke:#0066cc
    style GPTOSS_Timeline fill:#fff0e6,stroke:#cc9966
```

## 성능 최적화 전략

```mermaid
flowchart TD
    subgraph Strategies["최적화 전략"]
        direction TB

        subgraph DualModel["듀얼 모델 활용"]
            DM1["QWEN3: 대형 파일 + Refiner + Summary"]
            DM2["GPT-OSS: 소형 파일 + 전체 Validation"]
            DM3["품질 동등 → 부하 분산 가능"]
        end

        subgraph Streaming["스트리밍 파이프라인"]
            ST1["리뷰 완료 즉시 검증 시작"]
            ST2["대기 시간 최소화"]
            ST3["GPU 활용률 극대화"]
        end

        subgraph Combining["컴포넌트 결합"]
            CB1["Matcher + Reviewer 결합"]
            CB2["Validator Stage A+B 조건부 결합"]
            CB3["호출 횟수 33% 감소"]
        end

        subgraph Skipping["조건부 스킵"]
            SK1["코멘트만 변경: 리뷰 스킵"]
            SK2["설정 파일: 간소화 처리"]
            SK3["생성 파일: 자동 스킵"]
        end
    end

    DualModel --> Streaming --> Combining --> Skipping

    style DualModel fill:#e6f3ff,stroke:#0066cc
    style Streaming fill:#fff0e6,stroke:#cc9966
    style Combining fill:#e6ffe6,stroke:#66cc66
    style Skipping fill:#ffe6e6,stroke:#cc6666
```
