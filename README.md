# Memory Policy Simulator

운영체제 수업의 **가상 메모리 페이지 교체 정책** 과제를 위한 시뮬레이터 프로젝트입니다. 기본 제공 정책인 **FIFO**를 기준으로, 추가 구현 대상인 **Optimal Page Replacement**, **LRU Page Replacement**, 그리고 보너스 신규 정책인 **TSA-PR(TLB Shootdown Aware Page Replacement)**의 설계·실험·분석 방법을 함께 정리합니다.

> 현재 C# Windows Forms 기반 프로젝트는 FIFO 동작을 시각화하는 구조로 구성되어 있으며, 본 README는 과제 유의사항에 맞춰 구현 대상, 입력/출력, 결과 표·그래프, 동작 순서 시각화, 신규 정책 고유 분석 항목까지 포함한 제출 문서 템플릿 역할을 합니다.

---

## 1. 과제 목표

1. 수업시간에 학습한 가상 메모리의 페이지 교체 정책을 구현한다.
2. 각각의 페이지 교체 알고리즘의 동작 결과를 분석하고 이해한다.
3. 기존 정책과 다른 신규 페이지 교체 정책을 제안하고 평가한다.

---

## 2. 구현 및 분석 대상 알고리즘

| 구분 | 정책 | 설명 | 구현/분석 목적 |
|---|---|---|---|
| 기본 | FIFO | 가장 먼저 적재된 페이지를 먼저 교체 | 기본 기준 성능 측정 |
| 추가 정책 1 | Optimal | 앞으로 가장 늦게 사용되거나 다시 사용되지 않는 페이지를 교체 | 이론적 최적 성능 비교 기준 |
| 추가 정책 2 | LRU | 가장 오랫동안 사용되지 않은 페이지를 교체 | 실제 시스템에서 자주 쓰이는 근사 정책의 기준 |
| 신규 정책 | TSA-PR | TLB shootdown, IPI, dirty writeback, refault risk를 함께 고려 | 멀티코어 환경의 총 지연 비용 최소화 |

---

## 3. 실행 환경

- Language: C#
- Framework: .NET Framework Windows Forms
- IDE 권장: Visual Studio
- Solution file: `Memory_Policy_Simulator/Memory_Policy_Simulator.sln`
- 주요 파일
  - `Memory_Policy_Simulator/Memory_Policy_Simulator/Core.cs`
  - `Memory_Policy_Simulator/Memory_Policy_Simulator/Page.cs`
  - `Memory_Policy_Simulator/Memory_Policy_Simulator/Form1.cs`

---

## 4. 기본 입력 형식

모든 정책은 공통적으로 다음 입력을 사용합니다.

| 입력 항목 | 설명 | 예시 |
|---|---|---|
| 페이지 교체 정책 | FIFO, Optimal, LRU, TSA-PR 중 선택 | `FIFO` |
| Reference String | 페이지 참조열, 각 페이지는 1 character | `70120304230321201701` 또는 `ABCADBEA` |
| Frame Size | 물리 메모리 프레임 개수 | `3` |

### 4.1 신규 정책 TSA-PR 추가 입력

TSA-PR은 멀티코어 환경의 비용을 고려하므로 다음 입력을 추가로 사용합니다.

| 추가 입력 | 설명 | 예시 |
|---|---|---|
| Core Sequence | 각 참조가 어느 CPU core에서 발생했는지 나타내는 순서열 | `0 1 2 0 3 1 2 0` |
| Dirty Sequence | 각 참조가 write 접근인지 여부 | `0 0 1 0 1 0 0 1` |
| IPI Cost | TLB shootdown을 위해 core 간 interrupt를 보낼 때의 단위 비용 | `10` |
| Writeback Cost | dirty page를 제거할 때 발생하는 디스크 기록 비용 | `30` |
| Weight α | TLB shootdown 비용 가중치 | `1.0` |
| Weight γ | refault risk 가중치 | `1.0` |
| Weight δ | writeback 비용 가중치 | `1.0` |

예시 입력은 다음과 같습니다.

```text
Policy: TSA-PR
Reference String: A B C A D B E A
Core Sequence:    0 1 2 0 3 1 2 0
Dirty Sequence:   0 0 1 0 1 0 0 1
Frame Size: 3
IPI Cost: 10
Writeback Cost: 30
α = 1.0, γ = 1.0, δ = 1.0
```

---

## 5. 기본 출력 항목

모든 알고리즘은 다음 결과를 출력합니다.

| 출력 항목 | 의미 |
|---|---|
| Hit Count | 페이지가 이미 프레임에 존재하여 hit가 발생한 횟수 |
| Page Fault Count | 요청 페이지가 프레임에 없어 fault가 발생한 횟수 |
| Page Fault Rate | 전체 참조 대비 page fault 비율 |
| Migration Count | 프레임이 가득 찬 상태에서 페이지 교체가 발생한 횟수 |
| Execution Time | 알고리즘 전체 실행 시간 |
| Page Fault Delay | page fault로 인해 발생한 추정 지연 시간 |

Page Fault Rate는 다음 식으로 계산합니다.

```text
Page Fault Rate (%) = Page Fault Count / Total Reference Count × 100
```

---

## 6. 신규 정책: TSA-PR

### 6.1 정책 이름

**TSA-PR: TLB Shootdown Aware Page Replacement**

### 6.2 핵심 아이디어

기존 FIFO, LRU, Optimal 정책은 주로 page fault 횟수를 줄이는 데 집중합니다. 하지만 실제 멀티코어 운영체제에서는 페이지를 제거할 때 다음과 같은 추가 비용이 발생할 수 있습니다.

- 해당 페이지를 TLB에 보유하고 있을 가능성이 있는 다른 CPU core에 invalidation 요청을 보내야 함
- core 간 IPI(Inter-Processor Interrupt) 발생
- page table lock 또는 동기화 비용 발생
- dirty page 제거 시 writeback 비용 발생

따라서 TSA-PR은 단순히 page fault 수를 줄이는 것이 아니라, **page fault delay와 TLB shootdown delay를 포함한 총 메모리 관리 비용을 줄이는 것**을 목표로 합니다.

---

## 7. TSA-PR의 페이지별 관리 정보

각 페이지는 다음 정보를 가집니다.

| 필드 | 의미 |
|---|---|
| Page ID | 페이지 이름 |
| Loaded Time | 메모리에 적재된 시간 |
| Last Used Time | 마지막으로 참조된 시간 |
| Access Count | 누적 접근 횟수 |
| Recent Cores | 최근 해당 페이지를 접근한 CPU core 집합 |
| Dirty Bit | dirty page 여부 |
| Process ID | 프로세스 ID |
| Region ID | VMA 또는 page table region ID |

예시는 다음과 같습니다.

| Page | Recent Cores | Last Used Time | Dirty |
|---|---|---:|---|
| A | `{0, 1, 2, 3}` | 12 | No |
| B | `{0}` | 9 | No |
| C | `{1}` | 14 | Yes |

Page A는 여러 core에서 최근 접근되었기 때문에 제거 시 TLB shootdown 비용이 클 가능성이 있습니다. 반면 Page B는 하나의 core에서만 접근되었으므로 제거 비용이 상대적으로 낮습니다.

---

## 8. TSA-PR Eviction Score

TSA-PR은 각 후보 페이지 p에 대해 다음 점수를 계산합니다.

```text
EvictionScore(p)
= α × TLB_Shootdown_Cost(p)
+ β × Page_Table_Lock_Cost(p)
+ γ × Refault_Risk(p)
+ δ × Writeback_Cost(p)
```

점수가 가장 낮은 페이지를 victim page로 선택합니다.

```text
Victim = page with minimum EvictionScore
```

### 8.1 단순 구현 버전

과제 구현에서는 복잡도를 줄이기 위해 다음 단순식을 사용할 수 있습니다.

```text
EvictionScore(p)
= α × RecentCoreCount(p)
+ γ × RefaultRisk(p)
+ δ × DirtyCost(p)
```

이 방식은 TSA-PR의 핵심인 **최근 접근 core 수 기반 TLB shootdown 비용 회피**를 유지하면서 구현 난이도를 낮출 수 있습니다.

---

## 9. TSA-PR 비용 계산 방식

### 9.1 TLB Shootdown Cost

```text
TLB_Shootdown_Cost(p)
= RecentCoreCount(p) × IPI_COST × ProcessSensitivity
```

예를 들어 Page A가 최근 4개 core에서 접근되었고 IPI 비용이 10이면 다음과 같습니다.

```text
TLB_Shootdown_Cost(A) = 4 × 10 = 40
```

### 9.2 Refault Risk

LRU 개념을 함께 반영하기 위해 최근에 사용된 페이지일수록 제거 위험도를 크게 둡니다.

```text
Refault_Risk(p) = 1 / (CurrentTime - LastUsedTime + 1)
```

최근에 사용된 페이지는 다시 사용될 가능성이 높으므로 제거 점수가 증가합니다.

### 9.3 Writeback Cost

```text
Writeback_Cost(p) = WRITEBACK_COST, if p is dirty
Writeback_Cost(p) = 0, otherwise
```

Dirty page는 제거 시 디스크에 기록해야 하므로 더 높은 비용을 가집니다.

### 9.4 Batch Eviction Cost

같은 프로세스, 같은 VMA, 같은 page table region에 속한 페이지를 묶어서 제거하면 TLB shootdown 횟수를 줄일 수 있습니다.

```text
Individual Eviction: P1, P2, P3 제거 → TLB shootdown 3회
Batch Eviction:     P1, P2, P3 묶음 제거 → TLB shootdown 1회
```

Batch eviction은 과제의 선택 구현 요소로 두며, 구현할 경우 TSA-PR의 고유 분석 항목에 포함합니다.

---

## 10. 알고리즘 동작 순서 시각화

과제 유의사항의 “다양한 표현 도구를 활용하여 시각적으로 동작되는 순서를 제시”하기 위해, 각 알고리즘은 다음과 같은 표를 제공합니다.

예시 Reference String:

```text
Reference String = A B C A D B E A
Frame Size = 3
```

### 10.1 FIFO 동작 예시

| Step | Ref | Frame 1 | Frame 2 | Frame 3 | Result | Victim |
|---:|---|---|---|---|---|---|
| 1 | A | A | - | - | Fault | - |
| 2 | B | A | B | - | Fault | - |
| 3 | C | A | B | C | Fault | - |
| 4 | A | A | B | C | Hit | - |
| 5 | D | D | B | C | Fault | A |
| 6 | B | D | B | C | Hit | - |
| 7 | E | D | E | C | Fault | B |
| 8 | A | D | E | A | Fault | C |

### 10.2 TSA-PR 동작 예시

| Step | Ref | Core | Dirty | Candidate Scores | Victim | Result |
|---:|---|---:|---:|---|---|---|
| 1 | A | 0 | 0 | - | - | Fault |
| 2 | B | 1 | 0 | - | - | Fault |
| 3 | C | 2 | 1 | - | - | Fault |
| 4 | A | 0 | 0 | - | - | Hit |
| 5 | D | 3 | 1 | A=1.50, B=1.25, C=31.33 | B | Fault |
| 6 | B | 1 | 0 | A=1.33, C=31.25, D=31.50 | A | Fault |

위 표에서는 각 page fault 시점마다 후보 페이지의 Eviction Score를 계산하고, 가장 낮은 점수를 가진 페이지를 victim으로 선택합니다.

---

## 11. 결과 분석을 위한 표

과제 유의사항의 “그래프 혹은 표를 제시하고 해당 결과에 대한 분석을 기술”하기 위해 다음 표를 사용합니다.

### 11.1 기본 성능 비교표

| Policy | Hit Count | Fault Count | Fault Rate | Migration Count | Execution Time |
|---|---:|---:|---:|---:|---:|
| FIFO | 0 | 0 | 0.00% | 0 | 0 ms |
| Optimal | 0 | 0 | 0.00% | 0 | 0 ms |
| LRU | 0 | 0 | 0.00% | 0 | 0 ms |
| TSA-PR | 0 | 0 | 0.00% | 0 | 0 ms |

분석 시에는 다음 내용을 포함합니다.

- FIFO는 구현이 단순하지만 미래 사용 여부나 최근 사용 여부를 고려하지 않으므로 fault가 증가할 수 있습니다.
- Optimal은 미래 참조열을 알고 있다고 가정하므로 fault 수가 가장 낮은 이론적 기준입니다.
- LRU는 최근 사용성을 기준으로 하므로 일반적으로 FIFO보다 안정적인 결과를 보입니다.
- TSA-PR은 fault 수가 LRU보다 약간 증가할 수 있지만, TLB shootdown과 writeback 비용을 포함한 총 비용은 감소할 수 있습니다.

### 11.2 총 비용 비교표

| Policy | Page Fault Delay | TLB Shootdown Cost | Writeback Cost | Lock Cost | Total Cost |
|---|---:|---:|---:|---:|---:|
| FIFO | 0 | 0 | 0 | 0 | 0 |
| Optimal | 0 | 0 | 0 | 0 | 0 |
| LRU | 0 | 0 | 0 | 0 | 0 |
| TSA-PR | 0 | 0 | 0 | 0 | 0 |

분석 관점은 다음과 같습니다.

```text
Total Cost
= Page Fault Delay
+ TLB Shootdown Cost
+ Writeback Cost
+ Lock Cost
```

TSA-PR은 Page Fault Count만 최소화하는 정책이 아니므로, 반드시 **Fault Count**와 **Total Cost**를 함께 비교해야 합니다.

---

## 12. 결과 분석을 위한 그래프 제안

보고서에는 다음 그래프를 포함하는 것을 권장합니다.

### 12.1 알고리즘별 Page Fault Count 막대그래프

| Policy | Fault Count |
|---|---:|
| FIFO | 0 |
| Optimal | 0 |
| LRU | 0 |
| TSA-PR | 0 |

분석 내용:

- Optimal이 가장 낮은 fault count를 보이는지 확인합니다.
- FIFO와 LRU의 차이를 통해 최근 사용성 정보의 효과를 분석합니다.
- TSA-PR이 fault count에서 반드시 최적은 아니더라도, 추가 비용을 줄이기 위한 trade-off가 있는지 분석합니다.

### 12.2 알고리즘별 Total Cost 누적 막대그래프

누적 막대그래프의 항목은 다음처럼 나눕니다.

```text
Total Cost = Page Fault Delay + TLB Shootdown Cost + Writeback Cost + Lock Cost
```

분석 내용:

- FIFO/LRU는 페이지 교체만 고려하므로 shootdown 비용이 클 수 있습니다.
- TSA-PR은 TLB shootdown 비용이 큰 페이지를 피해서 제거하므로 total cost가 낮아질 수 있습니다.
- Dirty page 비율이 높을수록 Writeback Cost의 영향이 커집니다.

### 12.3 Frame Size 변화에 따른 Fault Rate 선그래프

| Frame Size | FIFO | Optimal | LRU | TSA-PR |
|---:|---:|---:|---:|---:|
| 2 | 0.00% | 0.00% | 0.00% | 0.00% |
| 3 | 0.00% | 0.00% | 0.00% | 0.00% |
| 4 | 0.00% | 0.00% | 0.00% | 0.00% |
| 5 | 0.00% | 0.00% | 0.00% | 0.00% |

분석 내용:

- 일반적으로 frame size가 증가하면 fault rate는 감소합니다.
- FIFO에서는 Belady's anomaly가 발생할 수 있으므로 frame size 증가에도 fault가 증가하는 경우가 있는지 관찰합니다.
- TSA-PR은 frame size가 작을수록 victim 선택의 영향이 커지므로 total cost 차이가 더 뚜렷하게 나타날 수 있습니다.

---

## 13. TSA-PR 고유 분석 항목

과제 유의사항의 “특정 알고리즘에서 요구하는 고유의 분석을 위한 출력 결과”에 해당하는 항목입니다.

| TSA-PR 출력 항목 | 의미 |
|---|---|
| Total TLB Shootdown Count | 페이지 제거로 인해 발생한 TLB shootdown 횟수 |
| Estimated IPI Cost | 최근 접근 core 수와 IPI 비용으로 계산한 추정 비용 |
| Average Recent Core Count | 제거된 victim page들의 평균 최근 접근 core 수 |
| Dirty Eviction Count | dirty page가 victim으로 제거된 횟수 |
| Writeback Cost | dirty page 제거로 인한 총 비용 |
| Average Eviction Score | victim 선택 시 평균 Eviction Score |
| Batch Eviction Count | 묶음 제거를 수행한 횟수 |
| Batch Saved Shootdowns | batch eviction으로 줄인 shootdown 횟수 |

### 13.1 TSA-PR 매개변수 분석

TSA-PR은 α, γ, δ 값에 따라 결과가 달라질 수 있으므로 다음 표를 추가하면 가점을 기대할 수 있습니다.

| α | γ | δ | Fault Count | TLB Shootdown Cost | Writeback Cost | Total Cost | 분석 |
|---:|---:|---:|---:|---:|---:|---:|---|
| 0.5 | 1.0 | 1.0 | 0 | 0 | 0 | 0 | TLB 비용 영향이 작음 |
| 1.0 | 1.0 | 1.0 | 0 | 0 | 0 | 0 | 균형 설정 |
| 2.0 | 1.0 | 1.0 | 0 | 0 | 0 | 0 | TLB shootdown 회피가 강해짐 |
| 1.0 | 2.0 | 1.0 | 0 | 0 | 0 | 0 | 최근 사용 페이지 보호가 강해짐 |
| 1.0 | 1.0 | 2.0 | 0 | 0 | 0 | 0 | dirty page 제거 회피가 강해짐 |

분석 예시:

- α가 증가하면 여러 core에서 접근된 페이지를 더 강하게 보호하므로 TLB shootdown cost는 감소할 수 있습니다.
- α가 너무 크면 최근에 사용되지 않은 페이지라도 core residency가 높다는 이유로 계속 보호되어 page fault가 증가할 수 있습니다.
- γ가 증가하면 LRU 성향이 강해져 최근 사용 페이지를 덜 제거합니다.
- δ가 증가하면 dirty page 제거를 회피하므로 writeback cost가 감소할 수 있지만, clean page가 더 자주 제거되어 fault pattern이 바뀔 수 있습니다.

---

## 14. 분석 서술 예시

보고서에는 다음과 같은 형태로 결과 분석을 작성합니다.

```text
실험 결과, Optimal은 미래 참조열을 알고 있다는 가정 때문에 가장 낮은 page fault count를 보였다. LRU는 FIFO보다 최근 사용성을 반영하기 때문에 대부분의 입력에서 더 낮은 fault rate를 보였다.

TSA-PR은 일부 입력에서 LRU보다 page fault count가 높게 나타났다. 그러나 TSA-PR은 여러 CPU core에서 최근 접근된 페이지를 victim으로 선택하지 않기 때문에 TLB shootdown cost가 감소하였다. 특히 core sequence가 여러 core에 고르게 분포된 경우, FIFO와 LRU는 TLB shootdown 비용이 큰 페이지를 제거할 수 있었지만 TSA-PR은 RecentCoreCount가 낮은 페이지를 우선 제거하여 total cost를 줄였다.

따라서 TSA-PR은 page fault count만을 최소화하는 정책이 아니라, 멀티코어 환경에서 page fault delay, IPI cost, dirty writeback cost를 합산한 total memory management cost를 줄이는 정책으로 해석할 수 있다.
```

---

## 15. 제출 시 포함할 자료 체크리스트

- [ ] FIFO, Optimal, LRU, TSA-PR 실행 결과
- [ ] 각 알고리즘의 Hit Count, Fault Count, Fault Rate 표
- [ ] 알고리즘별 Page Fault Count 그래프
- [ ] 알고리즘별 Total Cost 그래프
- [ ] Frame Size 변화에 따른 Fault Rate 그래프
- [ ] 페이지 참조 순서별 프레임 상태 변화 표
- [ ] TSA-PR의 Eviction Score 계산 예시
- [ ] TSA-PR의 TLB Shootdown Count, IPI Cost, Writeback Cost 출력
- [ ] α, γ, δ 매개변수 변화에 따른 결과 비교표
- [ ] 결과에 대한 분석 서술

---

## 16. 결론

본 프로젝트는 FIFO를 기본 정책으로 하여 Optimal, LRU, TSA-PR을 비교 분석하는 것을 목표로 합니다. FIFO, Optimal, LRU는 page fault count와 fault rate를 중심으로 비교하고, TSA-PR은 멀티코어 환경에서 발생하는 TLB shootdown, IPI, dirty writeback 비용까지 포함한 total cost 관점에서 평가합니다.

TSA-PR은 page fault count만 보면 항상 최적은 아닐 수 있지만, TLB shootdown 비용이 큰 페이지를 제거하지 않도록 유도함으로써 전체 시스템 지연 시간을 줄일 수 있습니다. 따라서 TSA-PR은 단일 지표 최적화가 아닌 **멀티코어 운영체제의 실제 비용을 반영한 페이지 교체 정책**이라는 점에서 기존 정책과 차별화됩니다.
