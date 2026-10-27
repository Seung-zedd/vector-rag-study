# Lesson 02 — ANN, Top-K, HNSW

## 1. Exact Search vs ANN

Vector DB에 저장된 모든 벡터와 query vector를 하나씩 비교하면 **Exact Search**다.

```text
q ↔ d1
q ↔ d2
...
q ↔ dN
```

벡터 차원이 D, 개수가 N이면 brute-force scan은 대략:

```text
O(N × D)
```

ANN(Approximate Nearest Neighbor)은 모든 벡터를 비교하지 않고 유망한 후보군만 탐색한다.

```text
Exact Search
→ 최대 정확도
→ 비용 큼

ANN
→ 훨씬 빠름
→ 약간의 recall 손실 가능
```

중요:

```text
Cosine / L2 / Dot Product
→ 거리·유사도 metric

ANN
→ 후보를 효율적으로 찾는 search strategy
```

---

## 2. Top-K

Top-K는 검색 결과 중 상위 K개만 반환하는 정책이다.

```text
d1 = 0.93
d2 = 0.88
d3 = 0.81
d4 = 0.74
```

Top-K=3이면 d1, d2, d3만 반환한다.

Trade-off:

```text
K 너무 작음
→ relevant evidence 누락

K 너무 큼
→ noise 증가
```

Top-K와 similarity threshold를 함께 사용할 수도 있다.

---

## 🔗 Connecting the Dots — Bayesian Inference × Adaptive Top-K

> ❓ **Q. Bayesian inference로 recall과 noise를 균형 있게 조절하는 adaptive Top-K를 만들 수 있을까?**
>
> **A. 가능하다.** K를 직접 예측하기보다 각 retrieved document가 relevant할 posterior probability를 추정하는 편이 자연스럽다.

```text
P(relevant | retrieval features)
```

가능한 feature:

- cosine similarity
- BM25 score
- reranker score
- metadata match
- query-document length relation

예:

```text
d1 → P(relevant) = 0.97
d2 → P(relevant) = 0.91
d3 → P(relevant) = 0.78
d4 → P(relevant) = 0.43
```

threshold를 적용하면 K가 고정값이 아니라 결과로 나타날 수 있다.

Bayesian 방식의 장점은 **uncertainty**도 표현할 수 있다는 것이다.

---

> ❓ **Q. ANN과 HNSW 모두 유망한 후보만 찾는 것처럼 보이는데 차이가 무엇인가?**
>
> **A. ANN은 큰 범주의 search strategy이고, HNSW는 ANN을 구현하는 구체적인 graph-based algorithm/index다.**

```text
ANN
= 전체를 다 보지 않고 근사적으로 nearest neighbor를 찾는 전략

HNSW
= 계층적 proximity graph를 탐색하는 ANN 구현
```

---

## 3. HNSW

HNSW = **Hierarchical Navigable Small World**

핵심:

> 모든 벡터를 비교하지 말고, 인접한 벡터들의 graph를 따라가며 query와 가까운 region으로 이동한다.

```text
Layer 2        A -------- F

Layer 1      A -- C -- F -- H

Layer 0    A-B-C-D-E-F-G-H-I-J
```

- upper layer: sparse, long jump
- lower layer: dense, local refinement

Search 흐름:

```text
upper layer 진입
   ↓
더 가까운 node로 이동
   ↓
lower layer로 내려감
   ↓
local search
   ↓
nearest candidates 반환
```

도로 비유:

```text
고속도로
→ 간선도로
→ 골목
→ 목적지
```

### HNSW는 거리 metric이 아니다

```text
Cosine / L2 / Dot Product
→ 얼마나 가까운지 측정

HNSW
→ 어떤 후보를 어떤 경로로 탐색할지 결정
```

---

## 🔗 Connecting the Dots — HNSW × Branch-Based Search × Routing

공통 추상화:

> 전체 search space를 전부 조사하지 말고 구조를 이용해 유망한 region으로 이동한다.

```text
Routing
current node
→ next hop
→ destination에 가까워짐

HNSW
current vector node
→ better neighbor
→ query region에 가까워짐
```

목적 함수는 다르지만 구조적 탐색이라는 점은 같다.

---

## Lesson 02 요약

```text
Exact Search
→ 모든 vector 비교

ANN
→ 유망 후보만 탐색

Top-K
→ 상위 K개 반환

Top-K + threshold
→ 개수와 최소 relevance를 함께 통제

HNSW
→ hierarchical graph를 사용하는 ANN
```

---

## Glossary

### Exact Search
모든 후보 벡터를 비교해 exact nearest neighbor를 보장하는 방식.

### ANN
전체 후보를 다 보지 않고 nearest neighbor를 근사적으로 찾는 검색 전략.

### Top-K
상위 K개의 검색 결과만 반환하는 정책.

### Similarity Threshold
결과로 유지하기 위한 최소 relevance/similarity 기준.

### Recall
존재하는 relevant result 중 실제로 찾아낸 비율.

### Noise
query와 관련성이 낮거나 불필요한 검색 결과.

### HNSW
계층형 graph를 사용하는 ANN index.

### Candidate Set
전체 데이터 대신 ANN이 실제로 검토하는 후보 집합.

### Posterior Probability
관측된 evidence를 반영한 이후의 Bayesian probability.

### Credible Interval
posterior distribution에서 plausible parameter range를 나타내는 구간.
