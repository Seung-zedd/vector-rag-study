# Lesson 05-6 — Retrieval Optimization & Adaptive Strategy Routing

> Previous: [Lesson 05-5 — Retrieval Failure Analysis](./05-failure-analysis.md) · Next: [Lesson 05-7 — Adaptive Retrieval](./07-adaptive-retrieval-and-early-exit.md)

## 17. Retrieval Optimization — Quality, Latency, Cost의 Trade-off

Part 16의 Failure Localization으로 문제 단계를 파악했다면, Part 17에서는 **변경할 검색 전략·파라미터를 선택하고 동일한 조건에서 평가**한다.

### 17.1 주요 Optimization Lever

| 변경 대상 | 기대 효과 | 주의점 |
|---|---|---|
| Candidate Top-N | 정답 후보 확보 범위 확대 | 후보 처리·재정렬 비용 증가 |
| Score Threshold | 낮은 유사도 후보 배제 | 중요한 문서까지 누락할 수 있음 |
| ANN Search Breadth | 탐색 Recall 개선 가능 | 지연시간·자원 사용량 증가 |
| RRF Rank Constant | 순위 기여도의 민감도 조절 | 여러 검색 결과의 융합 순위 변화 |
| Cross-Encoder Reranking | 후보 간 의미적 관련성 재평가 | 모델 추론에 따른 추가 비용 |

**Top-K를 늘린다고 Router 오류나 색인 누락이 자동으로 해결되지는 않는다.** 변경 위치가 실패 원인과 일치해야 한다.

### 17.2 Pareto Frontier와 Diminishing Returns

검색의 Quality(nDCG 등), Latency(P95 등), Cost는 서로 충돌할 수 있다.

```math
\max_{\theta}\ \mathrm{nDCG}@10(\theta)
```

subject to

```math
\mathrm{P95Latency}(\theta) \le L_{\max}
```

`θ`는 후보 수, 검색 범위, RRF 설정, Reranker 종류 등의 시스템 설정이다. 실험 예시의 250ms 같은 지연시간 제한은 **실제 SLA가 아닌 가정**이다.

어떤 구성보다 **품질은 낮고 지연시간은 높은 구성**은 두 지표만 보면 dominated configuration이다. 다른 구성으로 품질과 지연시간을 동시에 개선할 수 없는 점들은 Pareto Frontier의 후보가 된다.

> ❓ **Q. Quality–Latency 그래프가 로그함수 개형을 띠는 것 같은데?**
>
> **A. 가능하다.** 많은 시스템에서 지연시간을 더 투자할수록 추가 품질 개선 폭이 줄어드는 **Diminishing Returns**를 관측할 수 있다. 하지만 실제 품질이 항상 로그함수를 따르는 것은 아니다.

다음은 **실험 관측값이 아니라 설명용 모델**이다.

```math
Q(L)=0.60+0.07\ln\left(1+\frac{L}{50}\right)
```

```math
Q'(L)=\frac{0.07}{50+L}>0
```

```math
Q''(L)=-\frac{0.07}{(50+L)^2}<0
```

이 모델에서는 Latency가 늘면 Quality도 증가하지만 한계 개선량은 감소한다. 그러나 실제 시스템에는 이산적인 설정 변경, 지연시간 급증, 품질 하락, 모델·하드웨어 의존성이 있으므로 **하나의 로그 곡선이라고 단정하지 말고 측정한 점들의 Pareto Frontier를 분석**한다.

### 17.3 Golden Set 기반 Optimization Loop

```text
Baseline → Evaluate (fixed corpus / qrels / settings)
         → Identify Failure
         → Change One Parameter or Strategy
         → Evaluate Candidate (same conditions)
         → Compare Quality / P95 Latency / Cost
         → Accept or Roll Back
```

가능하면 초기에는 한 번에 하나씩 변경해 원인과 결과를 해석한다. 상호작용이 강한 파라미터는 후속 실험에서 함께 탐색할 수 있다. Golden Set에만 과도하게 맞추는 **Evaluation Overfitting**에 유의하고, 별도의 검증 질의도 둔다.

## 17.4 RRF vs Reranker vs Cross-Encoder

| 구분 | 정체 | 입력 | 핵심 역할 |
|---|---|---|---|
| RRF | 순위 융합 알고리즘 | 둘 이상의 검색 결과 순위 | Rank만으로 여러 결과 융합 |
| Reranking | 후보 재정렬 단계·기능 | Query와 후보 목록 등 | 후보 순서를 개선 |
| Cross-Encoder | Query–Document를 결합해 처리하는 모델 구조 | Query–Document 쌍 | 관련성 점수 직접 추정 |

RRF는 원문을 읽어 의미를 다시 판단하지 않고, **검색 엔진에서 얻은 순위**를 조합한다.

```math
\mathrm{RRF}(d)=\sum_{r\in R,\ d\in r}\frac{1}{k+\mathrm{rank}_{r}(d)}
```

Cross-Encoder 기반 Reranker는 각 후보에 `(query, document)`를 결합 입력해 관련성 점수를 계산한다. 반면 Bi-Encoder는 Query와 Document를 각각 인코딩해 벡터 유사도를 계산한다. Cross-Encoder는 정밀한 상호작용을 포착할 수 있으나 후보별 추론 비용이 크므로 **검색 전체 corpus보다는 후보 집합에 적용**하는 경우가 많다.

```text
User Query
    ├→ BM25 / Lexical Search ──┐
    └→ Vector / Semantic Search┤
                               ↓
                         RRF Fusion
                               ↓
                   Candidate Top-N (e.g. 30)
                               ↓
                 Cross-Encoder Reranking
                               ↓
                      Final Top-K → LLM
```

**RRF와 Cross-Encoder는 일반적으로 상호배타적인 선택지가 아니다.** Fusion 결과를 Cross-Encoder로 다시 정렬할 수 있고, 필요하면 한쪽 단계를 생략하도록 설계할 수도 있다. Cross-Encoder는 **후보에 없는 관련 문서를 일반적으로 새로 발견하지 못한다**는 제약이 있다.

### ❓ Q&A — 검색 공간이 좁으면 Cross-Encoder가 더 빠른가?

> **Q. 검색 공간이 넓을 때 RRF, Localized Space에서는 실시간성 때문에 Cross-Encoder로 라우팅하면 되는가?**
>
> **A. 그대로 대응시키기는 어렵다.** RRF는 순위 융합, Cross-Encoder는 후보별 신경망 추론이라 비용 구조와 책임이 다르다. 적은 후보는 Cross-Encoder의 실행 비용을 줄이지만 **RRF보다 빠르다는 뜻은 아니다.** 실시간성이 중요할수록 오히려 Reranking을 생략하거나 경량 모델을 고려할 수 있다.

## 🔗 Connecting the Dots — Adaptive Strategy Routing × DIP × Constrained Optimization

**두 종류의 의사결정을 분리**하는 설계 아이디어:

```text
Query / Latency Budget / Constraints
           ↓
Retrieval Strategy Router
  → Exact | Lexical | Semantic | Hybrid
           ↓
Candidate Retrieval + optional RRF Fusion
           ↓
Reranking Strategy Router
  → Skip | Lightweight Model | Cross-Encoder
           ↓
Final Top-K / Context Assembly
```

- **Retrieval Router:** 어디에서 어떤 방식으로 후보를 가져올까?
- **Reranking Router:** 확보된 후보를 얼마나 정밀하게 재평가할까?
- **Routing Evidence:** 후보 수, Query Intent, 기대 품질, P95 Latency Budget, 모델 비용 등을 기록.
- **Verification:** Golden Set의 그룹별 성능, Latency, Cost로 전략 선택의 효과 검증.

Strategy Pattern을 사용하면 상위 실행 계층이 구체적인 재정렬 구현체 대신 인터페이스에 의존하도록 설계할 수 있다(DIP).

```python
from typing import Protocol

class RerankingStrategy(Protocol):
    def rank(self, query: str, candidates: list[str]) -> list[str]: ...

class NoReranking:
    def rank(self, query: str, candidates: list[str]) -> list[str]:
        return candidates

class CrossEncoderReranking:
    def rank(self, query: str, candidates: list[str]) -> list[str]:
        # 실제 프로젝트에서는 모델을 주입하고 Query–Document 점수로 재정렬
        raise NotImplementedError
```

위 코드는 **설계 계약을 보여주는 스케치**이며 완성된 실행 구현이 아니다. 처음부터 모든 Router와 Strategy를 구현하기보다, 측정 결과에서 전략 분기 필요성이 확인될 때 책임을 나눠 확장한다.

### 🔗 Connecting the Dots — 기존 기술 면접 노트 복습

개인 공개 학습 자료인 [prep-tech-interview-with-gamification / INTERVIEW_QUEST.md](https://github.com/Seung-zedd/prep-tech-interview-with-gamification/blob/main/companies/_archive/com_and_human/INTERVIEW_QUEST.md)에는 **BM25 + Vector → RRF → Cross-Encoder (후보 Top-30~50에서 Final Top-K)** 개념이 정리되어 있다.

이번 학습에서는 그 **고정 2-stage Pipeline**을 확장하여 **Query Intent × Candidate Set × Latency Budget**에 따라 Reranking을 동적으로 선택하는 *Adaptive Strategy Routing* 아이디어로 연결했다. 단, 성능 수치나 효과는 실측으로 검증해야 하며 가정만으로 보장되지 않는다.

## Lesson 05-6 요약

- **Failure Analysis → Optimization → Regression Test** 순서로 품질 개선을 검증한다.
- RRF는 **여러 순위를 융합**, Cross-Encoder는 **후보의 의미적 관련성을 재평가**한다.
- 최적화는 Quality뿐 아니라 Latency·Cost·제약조건을 함께 다룬다.
- Strategy Routing은 **검색 공간 크기 하나가 아닌 여러 관측·요구사항**에 근거해야 한다.

---

## Glossary

### Pareto Frontier
다른 구성으로 모든 평가 목적을 동시에 개선할 수 없는 비지배 구성의 집합.

### Diminishing Returns
투입량이 늘어날수록 추가 투입 대비 개선 폭이 줄어드는 현상.

### Cross-Encoder
Query와 Document를 결합 입력해 관련성을 추론하는 모델 구조.

### Bi-Encoder
Query와 Document를 분리 인코딩하여 벡터 표현을 비교하는 모델 구조.

### Multi-stage Retrieval
빠른 초기 검색으로 후보를 확보하고 후속 단계에서 재정렬·정제하는 검색 파이프라인.

### Adaptive Strategy Routing
관측 정보와 예산·품질 조건에 따라 검색·재정렬 전략을 동적으로 선택하는 설계.

---

## References

### Primary Sources
- Cormack, Clarke & Buettcher (2009). [Reciprocal Rank Fusion Outperforms Condorcet and Individual Rank Learning Methods](https://doi.org/10.1145/1571941.1572114).
- Nogueira & Cho (2019). [Passage Re-ranking with BERT](https://arxiv.org/abs/1901.04085) — Cross-Encoder 기반 재정렬의 대표 연구.
- Reimers & Gurevych (2019). [Sentence-BERT](https://arxiv.org/abs/1908.10084) — 문장 임베딩 기반 Bi-Encoder 관련 배경.

### Official Documentation
- [Sentence Transformers — Cross-Encoder](https://sbert.net/examples/cross_encoder/README.html) — Cross-Encoder와 Reranking 설명.
- [Elasticsearch — Reciprocal Rank Fusion](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion) — RRF 적용 사례.

### Related Public Study
- [Technical Interview Notes — RRF & Cross-Encoder](https://github.com/Seung-zedd/prep-tech-interview-with-gamification/blob/main/companies/_archive/com_and_human/INTERVIEW_QUEST.md).

> **구분:** Adaptive Strategy Routing 구성은 이 학습에서 도출한 설계 아이디어이며, 인용된 연구의 직접적인 제안이라고 주장하지 않는다.
