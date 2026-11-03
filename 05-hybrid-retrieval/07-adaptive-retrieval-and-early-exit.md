# Lesson 05-7 — Adaptive Retrieval, Early Exit & Abstention

> Previous: [Lesson 05-6 — Retrieval Optimization](./06-retrieval-optimization.md) · Next: [Lesson 05-8 — Context Engineering & Caching](./08-context-engineering-and-cache.md)

## 18. Adaptive Retrieval — Query마다 같은 검색 비용을 지불해야 할까?

Part 17에서는 품질·지연시간·비용을 고려해 검색 설정을 최적화했다. Part 18은 **중간 검색 증거를 보고 다음 Stage를 실행하거나 Early Exit하는 의사결정**에 집중한다.

### 18.1 Adaptive Retrieval과 Cascaded Search

질의 "ABC-123 출시일"은 Exact/Filter로 확인할 수 있지만, "아이들과 가기 좋고 조용하며 대중교통이 편한 장소"는 복합 조건과 Semantic Retrieval을 요구할 수 있다.

- **Strategy Routing:** 어떤 검색 경로를 사용할지 결정한다.
- **Adaptive Retrieval:** 수행 중 수집한 증거에 따라 검색 깊이, 추가 실행, 중단 여부를 조정한다.
- **Early Exit:** 근거가 충분하다면 나머지 검색 비용을 생략한다.

```text
User Query
  ↓
Stage 1 — Exact / Lexical Retrieval
  ↓
Evidence Sufficiency Verifier
  ├─ Sufficient → ACCEPT / Early Exit
  └─ Insufficient → EXPAND
                        ↓
                 Vector Search
                        ↓
                 Optional RRF / Reranking
                        ↓
                 Reverify Evidence
                        ↓
                 ACCEPT / EXPAND / ABSTAIN
```

이것은 **Lexical-first Cascade 예시**이며 항상 Lexical부터 시작할 필요는 없다. 앞선 검색 단계가 높은 점수를 받았다는 사실만으로 정답의 충분성이 보장되지는 않는다.

### 18.2 Early Exit에 필요한 Evidence

예를 들어 "ABC-123 출시일"의 Exact 검색 결과가 ID 일치, 필수 필드 존재, 출처 유효성, 충돌 기록 없음 등을 검증했다면 조기 종료를 고려할 수 있다.

> **Q. Lexical Top-1 점수가 높아도 실제 정답이 Vector Search에서만 나올 수 있지 않나?**
>
> **A. 맞다.** 검색 점수와 Evidence Sufficiency는 다르다. Query의 모든 중요한 조건을 뒷받침할 근거가 없으면 확장 검색을 고려해야 한다. 평가 환경에서는 Golden Set으로 Early Exit의 오판을 측정한다.

### 18.3 Query Decomposition ≠ Document Chunking

복합 질의: "서울에서 11월에 열리는 축제 중 대중교통이 편리하고 아이들과 즐기기 좋은 곳 추천해 줘."

| Query Signal | 분류 예시 | Search Mapping |
|---|---|---|
| 서울 | Hard Constraint | Region Filter |
| 11월 | Hard Constraint | Date Range Filter |
| 축제 | Content Type | Filter / Lexical |
| 대중교통 편리 | Preference 또는 강제 조건 | Semantic / Structured Accessibility |
| 아이들과 즐기기 좋음 | Soft Preference | Semantic / Reranking |

**Query Decomposition**은 자연어를 Retrieval 신호·조건으로 분해하는 작업이고, **Document Chunking**은 문서를 저장·검색 가능한 단위로 나누는 작업이다.

Hard Constraint는 정확한 근거 필드가 존재할 때 필터링으로 강제할 수 있고, Soft Preference는 순위 평가에 반영할 수 있다. "반드시 대중교통이 편해야 한다"처럼 명시되면 접근성도 Hard Constraint가 될 수 있다.

중요한 점은 **Query Decomposition은 검색 누락을 줄이고, Evidence Sufficiency Verification은 Early Exit 판단을 돕는 서로 다른 책임**이라는 것이다.

### 🔗 Connecting the Dots — Compiler Parsing과 Retrieval Planning

```text
Natural Language Query
  → Parsing / Signal Decomposition
  → Constraint & Intent Analysis
  → Retrieval Plan
  → Execution
  → Evidence Verification
```

컴파일러의 Lexical·Syntax·Semantic Analysis를 연상시키지만 자연어는 프로그래밍 언어보다 모호하므로 동일한 결정론적 파싱을 가정할 수 없다.

## 18.4 Sequential Decision Making & Optimal Stopping

추가 검색을 할지 판단하는 **개념적 모델**:

```math
\mathbb{E}[\Delta Q\mid E] > \lambda\,\mathbb{E}[\Delta C\mid E]
```

- E: 지금까지 관측한 결과와 검증 근거
- ΔQ: 다음 검색 단계의 기대 품질 개선량
- ΔC: 추가 비용·지연시간
- λ: 품질과 비용을 비교하는 가중치

정답 없는 실시간 상황에서는 ΔQ를 직접 알 수 없으므로 Golden Set에 대한 실험이나 보정된 추정기를 사용해야 한다. 이 식은 **학습용 추상화**이며 보편적인 종료 규칙이나 안전성 보증은 아니다.

## 18.5 ACCEPT / EXPAND / ABSTAIN

> **Q. Abstain이 무슨 뜻이야?**
>
> **A. 기권·판단 유보.** 근거가 불충분할 때 답을 확정하지 않는 정상적인 결정이며, 실행 오류(FAIL)와 다르다.

| Decision | 조건 | 행동 |
|---|---|---|
| ACCEPT | 근거 충분 | 검색 종료 후 답변 진행 |
| EXPAND | 근거 부족, 추가 검색 가능 | 다음 Retrieval Stage 실행 |
| ABSTAIN | 근거 부족, 추가 검색 불가·부적절 | 답변 확정 보류, 불확실성 안내 |

```python
from enum import Enum

class Decision(Enum):
    ACCEPT = "accept"
    EXPAND = "expand"
    ABSTAIN = "abstain"

def decide_next_stage(evidence) -> Decision:
    if (
        evidence.exact_match
        and evidence.required_fields_present
        and evidence.source_valid
        and not evidence.has_conflict
    ):
        return Decision.ACCEPT
    if evidence.remaining_budget <= 0:
        return Decision.ABSTAIN
    return Decision.EXPAND
```

코드는 책임 분리를 보여주는 예시다. **Verifier는 근거를 판정하고, Stage Gate는 행동을 선택**한다. 실제 Evidence 타입과 검증 정책은 별도 설계가 필요하다.

### 🔗 Connecting the Dots — Selective Prediction / Reject Option

모델이 모든 질의에 무조건 답하는 대신, 확신이 충분하지 않으면 보류하는 접근을 **Selective Prediction**과 연결할 수 있다.

- **Coverage:** 전체 요청 중 답변한 비율.
- **Selective Risk:** 답변한 요청들에서 발생하는 조건부 오류 위험.

더 엄격한 보류 기준은 Coverage를 떨어뜨릴 수 있으며, 실제 오류 위험 감소 여부는 신뢰도 추정·평가 결과에 달려 있다. **ABSTAIN ≠ FAIL**이며 무조건 많은 Abstain이 좋은 것도 아니다.

## Lesson 05-7 요약

1. Adaptive Retrieval은 실행 중간 증거를 보고 검색을 확장하거나 조기 종료한다.
2. Query Decomposition과 Evidence Sufficiency Verification을 구분한다.
3. ACCEPT / EXPAND / ABSTAIN은 Stage Gate가 선택하는 정상적인 결과다.
4. 품질, Latency, Coverage 및 Selective Risk를 평가 데이터로 검증해야 한다.

---

## Glossary

- **Adaptive Retrieval:** 질의 및 중간 관측에 따라 검색 전략과 비용을 조절.
- **Early Exit:** 근거 충분성을 판단해 추가 검색을 생략.
- **Query Decomposition:** 복합 자연어 질의의 신호·제약을 분해.
- **Evidence Sufficiency:** 특정 응답을 뒷받침하는 근거의 충분성.
- **Abstain:** 확정 가능한 근거가 부족하여 판단을 유보.
- **Coverage:** 응답한 요청의 비율.
- **Selective Risk:** 응답한 요청에 대한 조건부 오류 위험.

## References

### Primary Sources
- [Jeong et al. (2024), Adaptive-RAG](https://arxiv.org/abs/2403.14403) — 질의 복잡도에 따른 검색 적응.
- [Geifman & El-Yaniv (2017), Selective Classification for Deep Neural Networks](https://arxiv.org/abs/1705.08500) — 선택적 예측의 Risk-Coverage.
- [Lewis et al. (2020), Retrieval-Augmented Generation](https://arxiv.org/abs/2005.11401) — RAG 검색·생성 파이프라인.

### Related Public Study
- [Lesson 05-5 — Failure Analysis](./05-failure-analysis.md)
- [Lesson 05-6 — Retrieval Optimization](./06-retrieval-optimization.md)

> **구분:** Query Decomposition + Evidence Verifier + 세 갈래 Stage Gate는 학습 과정에서 제안한 일반화된 설계 응용이며 인용 논문의 원래 아키텍처와 동일하다고 주장하지 않는다.
