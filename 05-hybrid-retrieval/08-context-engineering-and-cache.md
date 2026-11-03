# Lesson 05-8 — Context Engineering, Lost in the Middle & Retrieval Caching

> Previous: [Lesson 05-7 — Adaptive Retrieval](./07-adaptive-retrieval-and-early-exit.md) · Next: [Lesson 05-9 — RAG Cache Design Patterns](./09-cache-design-patterns.md)

## 19. Retrieval Success ≠ Answer Success

Retriever가 정답 문서를 Top-K에 포함시켜도 Context Assembly 또는 Generation에서 정답이 잘못 활용될 수 있다.

가상 질문: "2025년 A 축제의 입장료는?"
- Document A: 2024년 입장료 10,000원
- Document B: **2025년 입장료 15,000원** (정답)
- Document C: 2026년 입장료 18,000원

B가 Top-3 안에 있어도 LLM이 A의 금액으로 답하면 Retrieval Recall 성공과 Answer Correctness 실패가 동시에 발생한다.

### 19.1 Context Assembly와 Evidence Grounding

검색 결과를 무조건 이어 붙이는 것이 아니라 다음 정보를 보존·선별한다.

- **Relevance:** 질의 조건에 맞는 문서인가?
- **Temporal Relevance:** 사건 연도, 적용 시점, 수정일이 일치하는가?
- **Deduplication:** 중복·상충 문서를 어떻게 처리하는가?
- **Provenance:** document ID, 원본 URL, 제목, 발행일 등 출처 정보가 있는가?
- **Token Budget:** 관련성을 유지하며 얼마나 압축할 것인가?

```json
{
  "source_id": "doc_B",
  "event_year": 2025,
  "content": "Admission fee: 15,000 KRW",
  "source_url": "https://example.org/festival/2025"
}
```

위 JSON은 전적으로 가상 예시다. Metadata를 전달해도 모델이 언제나 올바른 연도를 선택한다는 보장은 없다.

### 19.2 Citation Presence ≠ Citation Correctness

| 항목 | 검증 질문 |
|---|---|
| Citation Presence | 답변에 출처 표시가 존재하는가? |
| Citation Validity | 참조한 document ID가 실제로 존재하는가? |
| Citation Support | 해당 문서가 해당 주장을 정말 뒷받침하는가? |

**Database Referential Integrity × Citation Validity:** 외래 키처럼 출처 ID의 존재를 검증할 수 있지만, **의미적 Support까지 보장하지는 못한다.**

```python
def valid_citation_ids(answer, retrieved_docs) -> bool:
    ids = {doc.id for doc in retrieved_docs}
    return all(c.doc_id in ids for c in answer.citations)
```

이 코드는 ID의 유효성만 검증한다. Citation Support는 별도의 평가가 필요하다.

## 19.3 Lost in the Middle — 인간 기억의 Serial Position Effect와의 유사성

**Lost in the Middle:** 긴 Context에서 관련 정보가 처음 또는 끝보다 **중간에 위치했을 때 활용 성능이 떨어질 수 있는 현상**.

인간의 Primacy/Recency Effect와 결과 형태가 닮았지만, 동일한 인지 원인이라고 단정할 수 없다.

- **Primacy Bias:** 초기 위치에 대한 학습된 편향, Attention Sink 등이 관여할 수 있다.
- **Recency Bias:** 생성 위치와 가까운 최근 문맥 활용 패턴이 관여할 수 있다.
- **Positional Attention Bias:** 위치 정보·Attention 메커니즘·학습 분포가 결합해 중간 정보 활용의 상대적 불이익을 만들 수 있다.
- 원인은 **하나로 확정되지 않았으며** 모델과 평가 과제에 따라 다르다.

### 19.4 Autoregressive Transformer, KV Cache, Attention Sink

Decoder-only 언어 모델은 앞서 나온 토큰에 조건부로 다음 토큰을 예측한다.

```math
P(x_1,\ldots,x_T)=\prod_{t=1}^{T}P(x_t\mid x_1,\ldots,x_{t-1})
```

Scaled Dot-Product Attention의 예시 식:

```math
\mathrm{Attention}(Q,K,V)
=\mathrm{softmax}\left(\frac{QK^T}{\sqrt{d_k}}+M\right)V
```

여기서 `M`은 Causal Mask 등을 나타낸다. Causal Mask는 **미래 토큰을 가릴 뿐, 가까운 토큰에 높은 점수를 직접 부여하지 않는다.**

KV Cache는 이미 처리한 토큰의 Key/Value 표현을 재사용한다. 일반적인 정확 KV Cache에서는 **토큰이 오래되었다는 이유만으로 자동으로 열화하거나 LRU처럼 제거되지 않는다.** 오래된 정보가 저장되어 있다는 것과 모델이 그 정보를 효과적으로 활용한다는 것은 다르다.

Causal Attention 구조에서 초기 토큰은 많은 후속 위치에서 참조 가능하다. 일부 Head에서 첫 토큰에 상대적으로 높은 Attention을 부여하는 **Attention Sink**가 관측되며, 이는 초기 위치 편향을 이해하는 단서가 된다. 단, **높은 Attention = 의미적 기억/정답 활용 성공**은 아니다.

### 19.5 Context Bloat / Context Rot와의 구분

| 현상 | 초점 |
|---|---|
| Context Bloat | 불필요한·중복된 입력이 문맥을 과도하게 차지함 |
| Context Rot | 문맥 길이·복잡성 증가 시 활용 성능 저하 또는 불안정성 |
| Lost in the Middle | 관련 정보의 **위치 변화**에 따른 활용 성능 차이 |

이를 구분하려면 Context 길이를 고정하고 정답 문서의 위치를 바꾸는 실험과, 정답 위치를 통제한 채 문맥 길이를 늘리는 실험을 별도로 수행한다.

### 🔗 Connecting the Dots — Operating System LRU vs KV Cache

- **OS Cache ↔ KV Cache:** 이미 계산된 정보를 보관하고 재사용한다는 추상적인 공통점.
- **LRU:** 최근 접근 이력을 토대로 캐시 공간을 관리·교체하는 정책.
- **KV Cache:** Attention 재계산을 줄이는 저장 구조이며, 본질적으로 LRU 정책이 아니다.
- **Lost in the Middle:** 데이터 퇴출보다 **저장된 정보의 접근·활용 편향**과 연관성이 깊다.

## 19.6 Retrieval Result Caching — 검색을 다시 하지 않아도 될까?

> **Q. Top-K 관련성 점수와 성공 조회 Counter를 기록하고 Redis 같은 캐시에 우선순위대로 보관하면 RAG Retrieval을 건너뛸 수 있지 않을까?**
>
> **A. 가능하다.** 다만 Cache Hit의 의미와 정답 검증을 구분해야 한다. 단순 조회 횟수는 인기도이지 정확도는 아니다.

세 가지 관련 접근:

1. **Exact Retrieval Cache:** 정규화된 동등 질의+제약에 대한 후보·결과 재사용.
2. **Semantic Cache:** 의미적으로 유사한 질의의 이전 결과를 재사용하되 동등성 오판 주의.
3. **Popularity/Quality-aware Eviction:** 사용량, 검증 성공률, 비용 등을 캐시 보존 정책에 반영.

```text
User Query
  → Cache Lookup (Exact / Semantic)
    ├─ Valid Hit → Freshness / Scope / Source Validation
    │               → Cached Evidence → Generation
    └─ Miss/Invalid → Retrieval → Fusion/Rerank → Verification
                                    ↓
                               Cache Update
```

**Top-K 문서 ID 캐시**는 검색 단계를 절약하지만 Generation이 필요할 수 있고, **검증된 답변 캐시**는 제한된 조건에서 Generation도 생략할 수 있다.

#### Counter와 Confidence는 다르다

```json
{
  "query_hash": "sha256:example",
  "document_ids": ["A", "B", "C"],
  "hit_count": 125,
  "verified_success_count": 117,
  "corpus_version": "v12"
}
```

- `hit_count`: 재사용 빈도. 높은 값이 정답률을 보증하지 않는다.
- `verified_success_count`: 실제 성공 판정이 있을 때만 증가. 어떤 Verifier가 무엇을 판정했는지 정의가 필요하다.
- `retrieval_score`: BM25 / Vector Similarity / Fusion Score 등으로 해석이 다르며 기본적으로 정답 확률이 아니다.
- **Counter DESC**는 LFU 유사한 보존 정책에 활용할 수 있지만 문서 관련성 순위의 유일한 기준으로 삼지 않는다.

### 19.7 Cache Freshness / Invalidation

캐시를 재사용하려면 다음을 함께 확인한다.

- **Query Equivalence:** 동일한 의미뿐 아니라 연도·날짜·지역·조건도 같은가?
- **Authorization Scope:** 사용자·조직별 접근권한에 차이가 있는가?
- **Corpus / Source Version:** 검색에 사용한 원본 집합이 변경되었는가?
- **Source Freshness:** 원문이 여전히 최신·유효한가?
- **Pipeline Version:** Retrieval 모델·필터·Reranking 변경으로 이전 결과의 사용 조건이 달라졌는가?

**Content Hash(SHA-256 등)**로 원문 변경을 감지할 수 있지만 해시가 같다고 현실 세계의 최신성이 보장되는 것은 아니다. TTL·변경 이벤트 기반 무효화·버전 검증 등을 적절히 조합한다.

**LSH(Locality-Sensitive Hashing)**는 유사 벡터 탐색을 위한 기술이며 Content Hash와는 역할이 다르다. 모든 Vector DB가 LSH 기반인 것도 아니다.

### 🔗 Connecting the Dots — OS LFU × RAG Cache Invalidation × Stage Gate

```text
Cache Hit
   ↓
Equivalence + Authorization + Freshness Verifier
   ├─ PASS → Reuse Evidence / Answer (if permitted)
   └─ FAIL → Recompute Retrieval
```

**Cache Hit ≠ Verified Correct Answer.** Cache-aware RAG에서도 Verifier와 Stage Gate의 분리를 유지한다.

## Lesson 05-8 요약

- **Retrieval Recall → Context Coverage → Citation Support → Answer Correctness**는 서로 다른 평가 단계다.
- Lost in the Middle은 Positional Attention Bias 등이 관여할 수 있으나 원인이 완전히 확정된 것은 아니다.
- KV Cache는 LRU와 다르며 **물리적인 정보 보관과 효과적인 정보 사용을 구분**해야 한다.
- Retrieval Cache는 성능을 개선할 수 있지만 **동등성·권한·신선성·버전 검증** 없이는 안전하지 않다.

---

## Glossary

- **Context Assembly:** 검색 근거를 LLM 입력에 맞게 선별·배치하는 단계.
- **Evidence Grounding:** 주장과 실제 근거를 연결해 검증하는 접근.
- **Lost in the Middle:** 긴 Context의 중간 위치에 있는 관련 정보를 상대적으로 잘 활용하지 못하는 경향.
- **Attention Sink:** 일부 Attention Head가 초기 토큰에 내용과 무관한 Attention을 집중하는 현상.
- **KV Cache:** 이전 토큰의 Attention용 Key/Value 표현 저장.
- **Context Rot / Bloat:** 긴 문맥의 활용 성능 악화 / 불필요한 문맥의 과다 누적.
- **Semantic Caching:** 유사 질의의 기존 결과를 재활용하는 캐시 접근.
- **Cache Invalidation:** 원본 변경·권한·버전 불일치 등의 이유로 캐시를 무효화하는 과정.

## References

### Primary Sources
- [Liu et al. — Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)
- [Xiao et al. — Efficient Streaming Language Models with Attention Sinks](https://arxiv.org/abs/2309.17453)
- [Found in the Middle: Calibrating Positional Attention Bias Improves Long Context Utilization](https://arxiv.org/abs/2406.16008)
- [Vaswani et al. — Attention Is All You Need](https://arxiv.org/abs/1706.03762)

### Official Documentation
- [Redis — Cache Eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis — Semantic Caching](https://redis.io/docs/latest/develop/ai/redisvl/user_guide/llmcache/)

### Related Public Study
- [Lesson 05-5 — Retrieval Failure Analysis](./05-failure-analysis.md)
- [Lesson 05-7 — Adaptive Retrieval](./07-adaptive-retrieval-and-early-exit.md)

> **구분:** Cached Retrieval과 Stage Gate를 연결한 설계는 학습 중 도출한 일반화된 응용 아이디어이며, 실제 성능은 별도의 평가가 필요하다.
