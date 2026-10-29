# Lesson 05-1 — Hybrid Retrieval Basics

> 이전: [Lesson 04 — Chunking](../04-chunking/01-chunking-strategies.md) · 다음: [Lesson 05-2 — Query Routing](./02-query-routing-and-fusion.md)

## 1. 왜 Hybrid Retrieval이 필요한가

Lexical Search가 강한 것:

```text
고유명사
ID
지명
code
정확한 keyword
```

Semantic Search가 강한 것:

```text
"야경이 예쁜"
"아이와 가기 좋은"
"비 오는 날 실내에서 즐기기 좋은"
```

실제 query는 두 종류의 signal을 함께 포함하는 경우가 많다.

예:

```text
"부산 광안리에서 조용하고 야경이 좋은 카페"
```

```text
부산 / 광안리 / 카페
→ Lexical / Filter signal

조용하고 / 야경이 좋은
→ Semantic signal
```

Hybrid Retrieval은 서로 다른 retrieval signal을 결합해 한쪽의 약점을 다른 쪽으로 보완한다.

---

## 2. Parallel Hybrid vs Cascade Hybrid

### Parallel Hybrid

```text
User Query
   ↓
 ┌───────────────┬───────────────┐
 ↓               ↓
Lexical Engine   Vector DB
 ↓               ↓
Result A         Result B
 └───────┬───────┘
         ↓
       Fusion
         ↓
      Final Top-K
```

장점:

- recall 확보에 유리
- 서로 다른 검색 방식이 서로 보완

단점:

- 두 backend를 항상 호출
- latency/cost 증가
- fusion 필요

### Cascade Hybrid

```text
User Query
   ↓
Lexical Search
   ↓
충분한 결과?
   ├─ YES → 종료
   └─ NO  → Semantic Search
```

장점:

- exact query가 빠름
- embedding/vector search 비용 절감

단점:

- 첫 단계 판단이 틀리면 recall 손실

---

## 3. Fusion

Parallel Hybrid에서는 두 결과 집합을 하나의 ranking으로 합쳐야 한다.

```text
Lexical Results
      +
Semantic Results
      ↓
Fusion
      ↓
Final Ranking
      ↓
Top-K
```

대표 방식:

- Weighted Score
- RRF
- Reranker

이 레슨에서는 세부 알고리즘보다 **왜 fusion이 필요한가**를 이해하는 것이 우선이다.

---

## 4. Filter vs Search

> ❓ **Q. Lexical과 Filter의 차이는 무엇인가?**
>
> **A. Filter는 조건을 만족하는지 여부로 후보군을 제한하고, Lexical Search는 query와 document의 실제 word/token match를 기반으로 relevance score를 계산해 ranking한다.**

핵심:

```text
Filter
→ Boolean / Constraint
→ "조건을 만족하는가?"

Lexical Search
→ Keyword Relevance Ranking
→ "질의의 단어와 얼마나 잘 맞는가?"

Semantic Search
→ Meaning Relevance Ranking
→ "질의의 의미와 얼마나 가까운가?"
```

같은 단어도 query intent에 따라 역할이 달라질 수 있다.

예:

```text
"서울에 있는 카페만"
→ 서울 = Filter

"'서울의 밤'이라는 이름이 들어간 장소"
→ 서울 = Lexical

"도심 야경을 즐기기 좋은 곳"
→ Semantic
```

---

## 5. Pre-filter vs Post-filter

### Pre-filter

```text
전체 데이터
   ↓
Filter
   ↓
후보군 축소
   ↓
Search / Ranking
```

장점:

- 검색 공간 감소
- exact constraint 보장
- 계산량 감소 가능

단점:

- 너무 강한 filter는 recall을 잃을 수 있음

### Post-filter

```text
전체 데이터
   ↓
Search
   ↓
Top-K
   ↓
Filter
```

단점:

- Top-K 결과가 filter 후 크게 줄 수 있음
- 불필요한 검색 비용 발생 가능

핵심:

> 확실한 제약조건은 Pre-filter, 애매한 preference는 Search/Ranking에 남기는 것이 자연스럽다.

---

## 6. Hard Constraint vs Soft Preference

예:

```text
"부산 광안리에서 조용한 카페"
```

가능한 분해:

```text
부산 / 광안리
→ Hard Constraint
→ Filter

카페
→ Filter 또는 Lexical

조용한
→ Soft Preference
→ Semantic Ranking
```

이 decomposition이 QueryRouter 설계의 핵심 입력이 된다.

---


## Glossary

### Hybrid Retrieval
Lexical, Filter, Semantic 등 여러 retrieval signal을 함께 사용하는 방식.

### Parallel Hybrid
여러 retrieval backend를 동시에 실행하고 결과를 fusion하는 방식.

### Cascade Hybrid
한 retrieval path를 먼저 실행하고 부족할 때 다음 path로 내려가는 방식.

### Fusion
서로 다른 retrieval 결과를 하나의 relevance ranking으로 합치는 과정.

### Filter
조건을 만족하지 않는 후보를 제거하는 Boolean/constraint operation.

### Lexical Search
실제 term/token match를 이용해 relevance ranking을 수행하는 검색.

### Semantic Search
embedding vector의 의미적 유사도를 이용해 ranking하는 검색.

### Pre-filter
검색 전에 후보군을 제한하는 방식.

### Post-filter
검색 결과에 나중에 filter를 적용하는 방식.

### Hard Constraint
반드시 만족해야 하는 조건.

### Soft Preference
반드시 만족할 필요는 없지만 ranking에 영향을 주는 선호 조건.

---

## References

### Primary Sources
- Lewis, P. et al. (2020). [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401) — RAG의 검색·생성 구조.
- Cormack, G. V. et al. (2009). [Reciprocal Rank Fusion Outperforms Condorcet and Individual Rank Learning Methods](https://doi.org/10.1145/1571941.1572114) — Fusion의 대표 기법.

### Official Documentation
- [Elasticsearch — Reciprocal rank fusion](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion) — RRF 적용 참고.
