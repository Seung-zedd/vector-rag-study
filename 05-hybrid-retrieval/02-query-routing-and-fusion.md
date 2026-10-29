# Lesson 05-2 — Query Routing and Fusion

> Previous: [Lesson 05-1](01-hybrid-retrieval-basics.md) · Next: [Lesson 05-3](03-retrieval-metrics-and-verification.md)


## 7. Query Decomposition

Hybrid Retrieval의 핵심은 user query를 한 덩어리로 보지 않고 역할별 signal로 분해하는 것이다.

예:

```text
"부산 광안리에서 조용하고 야경이 좋은 카페"
```

가능한 분해:

```text
부산
→ Hard Constraint
→ Filter

광안리
→ Hard Constraint / Exact Term
→ Filter 또는 Lexical

카페
→ Category
→ Filter 또는 Lexical

조용한
→ Soft Preference
→ Semantic Search

야경이 좋은
→ Soft Preference
→ Semantic Search
```

중요:

> 품사 자체보다 query intent와 constraint 성격을 기준으로 분해한다.

같은 명사라도 역할이 달라질 수 있다.

```text
"카페만 보여줘"
→ Filter

"이름에 카페가 들어간 곳"
→ Lexical

"커피 마시기 좋은 곳"
→ Semantic
```

---

## 8. QueryRouter Confidence와 Fallback

QueryRouter는 단순 binary 분기기가 아니라 route confidence를 기반으로 retrieval path를 선택할 수 있다.

```text
User Query
   ↓
Query Analysis
   ↓
Confidence Estimation
   ├─ High Lexical Confidence
   │    → Lexical / Filter
   ├─ High Semantic Confidence
   │    → Vector Search
   └─ Ambiguous
        → Hybrid
```

예:

```text
Exact entity confidence = 0.95
Semantic intent confidence = 0.20
→ Lexical 우선

Exact entity confidence = 0.35
Semantic intent confidence = 0.82
→ Semantic 우선

Exact entity confidence = 0.58
Semantic intent confidence = 0.63
→ Hybrid
```

Fallback도 중요하다.

```text
Lexical
   ↓
결과 0건 또는 score 부족
   ↓
Semantic Fallback
```

즉 Lexical-first Cascade를 다음처럼 구체화할 수 있다.

```text
1. Exact / Filter
2. Lexical
3. Semantic
4. Hybrid / Rerank
```

---

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

---

## Glossary

### Query Decomposition
user query를 hard constraint, lexical term, semantic preference 등 역할별 signal로 분해하는 과정.


### Route Confidence
특정 query를 lexical, semantic, hybrid 등의 retrieval path로 보낼 때의 신뢰도.


### Fallback
선택한 retrieval path의 결과가 부족할 때 다른 retrieval strategy로 전환하는 동작.


### Weighted Signal Aggregation
여러 signal을 가중합 등으로 결합해 하나의 score 또는 decision value를 만드는 방식.


### Rule-based Router
명시적인 조건과 pattern으로 retrieval path를 결정하는 QueryRouter.


### Route Accuracy
QueryRouter가 적절한 retrieval path를 선택한 비율.


---

## References

### Primary Sources
- Cormack, G. V., Clarke, C. L. A. & Buettcher, S. (2009). [Reciprocal Rank Fusion Outperforms Condorcet and Individual Rank Learning Methods](https://doi.org/10.1145/1571941.1572114) — RRF 기반 순위 융합.
- Järvelin, K. & Kekäläinen, J. (2002). [Cumulated Gain-Based Evaluation of IR Techniques](https://doi.org/10.1145/582415.582418) — DCG/nDCG 평가.
- Lewis, P. et al. (2020). [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401) — RAG 검색·생성 구조의 배경.

### Official Documentation
- [Elasticsearch — Reciprocal rank fusion](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion) — RRF의 검색 엔진 적용.
- [TREC — Text REtrieval Conference](https://trec.nist.gov/) — qrels와 정보 검색 평가의 공개 벤치마크 맥락.

### Related Public Study
- [Self-Improving Agent Systems — Robust Verification](https://github.com/Seung-zedd/self-improving-agent-systems/blob/main/docs/wiki/02-robust-verification.md) — Stanford CS329A 학습 내용을 기반으로 한 Process Verification 연결.

> **구분:** QueryRouter confidence, Lexical-first Cascade, Phase별 Verifier 인터페이스 등은 학습 과정에서 제안한 설계·응용 아이디어다. 참고 논문의 원 주장과 구분한다.
