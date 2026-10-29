# Lesson 05-2 — Query Routing, Ranking & Evaluation Metrics

> 이전: [Lesson 05-1](./01-hybrid-retrieval-basics.md) · 다음: [Lesson 05-3](./03-evaluation-and-regression.md)

## 🔗 Connecting the Dots — Route Confidence × Weighted Signal Aggregation

Route Confidence도 여러 signal의 가중합으로 표현할 수 있다.

예:

```text
L(x)
→ lexical / exact signal strength

S(x)
→ semantic intent strength
```

```text
R(x) = αL(x) + βS(x)
```

실전에서는 lexical confidence와 semantic confidence를 별도로 두는 편이 더 해석하기 쉽다.

```text
C_lexical(x)
= a1 * entity_match
+ a2 * exact_keyword_match
+ a3 * filterability
+ a4 * ID_pattern

C_semantic(x)
= b1 * intent_abstractness
+ b2 * semantic_preference
+ b3 * paraphrase_need
+ b4 * lexical_match_weakness
```

Routing decision:

```text
C_lexical >> C_semantic
→ Lexical / Filter

C_semantic >> C_lexical
→ Semantic

둘이 비슷함
→ Hybrid
```

이 구조는 Weighted Fusion과 수학적으로 유사하지만 적용 위치가 다르다.

```text
Before Retrieval
→ Route Confidence
→ 어떤 retrieval path를 탈지 결정

After Retrieval
→ Weighted Fusion
→ 여러 retrieval 결과를 어떻게 합칠지 결정
```

공통 추상화:

> 여러 signal을 가중합으로 합성해 하나의 의사결정 score를 만든다.

초기에는 fixed weight로 시작하고, query log와 evaluation dataset이 쌓이면 α, β 또는 각 feature weight를 조정할 수 있다.


## 9. Rule-based Router vs Classifier / LLM Router

가장 단순한 QueryRouter는 deterministic rule 기반이다.

```text
if ID pattern:
    → Exact / Filter

elif known entity:
    → Lexical

elif semantic preference:
    → Semantic

else:
    → Hybrid
```

장점:

- 빠름
- 저비용
- deterministic
- debugging이 쉬움
- 설명 가능성이 높음

단점:

- rule이 늘수록 복잡해짐
- 예외가 많아짐
- 새로운 표현에 약함

Classifier 기반 Router는 query feature를 보고 route를 예측한다.

```text
Query
   ↓
Classifier
   ↓
LEXICAL / SEMANTIC / HYBRID
```

LLM Router는 decomposition까지 한 번에 수행할 수 있다.

```json
{
  "route": "HYBRID",
  "filters": {},
  "lexical_terms": [],
  "semantic_query": "..."
}
```

다만 LLM Router는 latency, cost, non-determinism, prompt dependency가 커질 수 있다.

현실적인 progression:

```text
Rule-based
   ↓
Rule-based + Confidence
   ↓
Small Classifier
   ↓
LLM Router
```

핵심:

> 확실한 패턴은 deterministic rule로 처리하고, 애매한 query만 더 비싼 classifier/LLM에 넘기는 것이 안정적이다.

---

## 10. Router Misclassification Failure

QueryRouter가 잘못 분류하면 retrieval path 자체가 잘못된다.

예:

```text
"부산에서 조용한 카페"
→ Lexical only로 잘못 routing
→ "조용한" semantic preference를 충분히 반영하지 못함
```

반대로:

```text
"광안리해수욕장 운영시간"
→ Semantic only로 잘못 routing
→ exact entity lookup 대신 비슷한 후보가 섞일 수 있음
```

Failure propagation:

```text
Wrong Routing
   ↓
Wrong Candidate Space
   ↓
Poor Retrieval
   ↓
Poor Context
   ↓
Wrong Answer
```

Router도 독립적으로 평가해야 한다.

대표 지표:

```text
Route Accuracy
→ 적절한 retrieval path를 선택했는가?

Fallback Rate
→ 첫 route 실패 후 fallback이 얼마나 발생하는가?

Retrieval Recall after Routing
→ routing 이후 정답 문서를 실제로 찾았는가?

Latency / Cost
→ routing 전략이 실제 비용을 줄였는가?
```

중요:

> Router Accuracy와 Retrieval Quality는 같은 것이 아니다.

```text
Query
   ↓
[Route Evaluation]
   ↓
Retriever
   ↓
[Retrieval Evaluation]
```

Routing correctness와 retrieval correctness는 서로 다른 책임 계층이므로 따로 측정해야 한다.

---

## 11. Hybrid Retrieval Evaluation — Recall@K / Precision@K

검색 품질은 감이 아니라 metric으로 측정해야 한다.

### Recall@K

정답 문서가:

```text
A, B, C
```

이고 Top-5 결과가:

```text
[A, D, B, E, F]
```

라면 정답 3개 중 2개를 찾았으므로:

```text
Recall@5 = 2 / 3 = 0.67
```

의미:

> 존재하는 relevant document 중 Top-K 안에 얼마나 많이 포함시켰는가.

### Precision@K

같은 Top-5 결과에서 relevant result는 A, B 두 개이므로:

```text
Precision@5 = 2 / 5 = 0.4
```

의미:

> 가져온 Top-K 결과 중 실제 relevant result의 비율이 얼마나 되는가.

Trade-off:

```text
Recall 높이기
→ 후보군 넓힘
→ noise 증가 가능

Precision 높이기
→ 후보군 엄격하게 제한
→ 정답 누락 가능
```

Hybrid Retrieval 비교 예:

```text
Lexical only
Recall@5 = 0.62

Semantic only
Recall@5 = 0.71

Hybrid
Recall@5 = 0.84
```

이렇게 비교하면 "Hybrid가 좋아 보인다"가 아니라 실제 retrieval quality improvement를 수치로 설명할 수 있다.

다만 Recall@K와 Precision@K는 ranking position 자체의 품질은 충분히 반영하지 못한다.

예:

```text
System A
[정답, 정답, 오답, 오답, 오답]

System B
[오답, 오답, 오답, 정답, 정답]
```

둘의 Precision@5 / Recall@5는 같을 수 있지만 사용자 경험은 System A가 더 좋다.

이 문제는 이후 MRR, nDCG 같은 ranking metric에서 다룬다.


## 12. MRR (Mean Reciprocal Rank)

Recall@K와 Precision@K는 관련 문서의 비율을 보여주지만 첫 정답의 순위를 충분히 구분하지 못한다.

**RR (Reciprocal Rank)**은 첫 번째 relevant 문서가 등장한 순위의 역수다.

```text
RR = 1 / rank(first relevant)

첫 정답 1위 → RR=1
첫 정답 3위 → RR≈0.333
첫 정답 5위 → RR=0.2
정답 없음 → RR=0
```

**MRR**은 여러 query에서 RR의 평균이다.

```text
MRR = (1/N) × Σ RR(query_i)

첫 정답 순위가 1, 2, 4라면
MRR = (1 + 1/2 + 1/4) / 3 ≈ 0.583
```

한계: 첫 번째 relevant 문서 외의 추가 정답에는 직접 보상하지 않는다.

> ❓ **Q. 첫 정답 이후의 문서는 MRR 보상이 없으니 모두 불필요한 retrieval cost인가?**
>
> **A. 아니다.** MRR은 ranking *평가 지표*이지 검색 종료 알고리즘이 아니다. 여러 근거가 필요한 query에는 추가 문서가 유용할 수 있다. 첫 정답 이후의 문서는 MRR 계산에 반영되지 않을 뿐이다.

---

## 13. nDCG (Normalized Discounted Cumulative Gain)

nDCG는 **여러 문서의 관련성 등급과 순위**를 함께 반영한다.

$rel_i$는 검색 결과의 $i$번째 문서에 부여한 관련성 등급이다.

- $rel_i = 3$: 매우 관련 있음
- $rel_i = 2$: 관련 있음
- $rel_i = 1$: 약간 관련 있음
- $rel_i = 0$: 관련 없음

### DCG@K

```math
\mathrm{DCG}@K = \sum_{i=1}^{K} \frac{2^{rel_i}-1}{\log_2(i+1)}
```

### IDCG@K

```math
\mathrm{IDCG}@K = \max_{\pi}\sum_{i=1}^{K} \frac{2^{rel_{\pi(i)}}-1}{\log_2(i+1)}
```

$\pi$는 평가 대상 문서들의 순열이며, $\mathrm{IDCG}@K$는 관련성이 높은 문서가 앞에 오도록 이상적으로 정렬했을 때의 DCG다.

### nDCG@K

```math
\boxed{\mathrm{nDCG}@K = \frac{\mathrm{DCG}@K}{\mathrm{IDCG}@K}}
```

일반적으로 nDCG는 0~1이고 1이면 이상적인 순위다. IDCG=0인 query의 처리 규칙은 평가 프로토콜에서 별도 정의해야 한다.

```text
MRR
→ 첫 정답이 얼마나 앞에 있는가?

nDCG
→ 관련성 높은 문서들이 전체적으로 얼마나 잘 정렬됐는가?
```

nDCG 계산에는 query-document 별 관련성 판정값인 **qrels (relevance judgments)**가 필요하다.

---

## 🔗 Connecting the Dots — Ranking Strategy × Evaluation Metric × Verifier

> ❓ **Q. Strategy는 RRF, Qualification은 MRR, Verifier는 nDCG라고 보면 되는가?**
>
> **A. 역할 구분은 좋지만 MRR과 nDCG는 모두 Evaluation Metric이다.** RRF는 ranking을 만드는 전략이고, MRR/nDCG는 ranking의 품질을 측정한다. Verifier는 계산된 지표를 명시된 합격 기준과 비교하는 검증 절차다.

```text
Retrieval Results
   ↓
Fusion Strategy (RRF / Weighted Fusion)
   ↓
Final Ranking
   ↓
Evaluation Metrics
   ├─ Recall@K / Precision@K
   ├─ MRR
   └─ nDCG@K
   ↓
Verifier (Acceptance Criteria)
   ↓
PASS / FAIL / Investigation
```

예시로 `nDCG@10 ≥ 0.85` 같은 기준을 둘 수 있지만, 실제 threshold는 평가 데이터셋과 제품 요구사항으로 정해야 한다.

핵심: **Ranking algorithm / Evaluation metric / Verification policy는 서로 다른 책임이다.**
---

## Glossary

### Weighted Signal Aggregation
여러 signal을 가중합 등으로 결합해 하나의 score 또는 decision value를 만드는 방식.

### Rule-based Router
명시적인 조건과 pattern으로 retrieval path를 결정하는 QueryRouter.

### Route Accuracy
QueryRouter가 적절한 retrieval path를 선택한 비율.

### Recall@K
존재하는 relevant document 중 Top-K 안에 포함된 비율.

### Precision@K
Top-K 결과 중 relevant document가 차지하는 비율.

### Reciprocal Rank (RR)
첫 번째 relevant 결과의 순위에 역수를 취한 값.

### MRR
복수 query의 Reciprocal Rank 평균. 첫 정답의 빠른 등장에 초점을 둔 ranking metric.

### DCG / nDCG
각 결과의 관련성 등급과 순위를 할인 누적하고 이상적인 정렬 결과로 정규화하는 ranking metric.

### Qrels
Query-document 쌍에 대한 사람이 정의하거나 검증한 관련성 판정값.

## References

### Primary Sources
- Cormack, G. V. et al. (2009). [Reciprocal Rank Fusion Outperforms Condorcet and Individual Rank Learning Methods](https://doi.org/10.1145/1571941.1572114) — 순위 융합.
- Järvelin, K. & Kekäläinen, J. (2002). [Cumulated Gain-Based Evaluation of IR Techniques](https://doi.org/10.1145/582415.582418) — nDCG 평가.

### Official Documentation
- [TREC — Text REtrieval Conference](https://trec.nist.gov/) — 검색 평가.
