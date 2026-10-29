# Lesson 05-4 — Evaluation and Regression

> Previous: [Lesson 05-3](03-retrieval-metrics-and-verification.md)


## 14. Evaluation Dataset과 Qrels — Training 없이 Retrieval 품질 측정하기

검색의 MRR / nDCG를 측정하려면 **어떤 query에 어떤 문서가 relevant인지**에 대한 판정값이 필요하다. 이 평가용 정답 데이터를 qrels(Query Relevance Judgments)라 한다.

### 14.1 Corpus / Evaluation Dataset / Training Dataset

| 구분 | 목적 | 모델 파라미터 업데이트? |
|---|---|---|
| Corpus | 실제 검색 대상 문서 집합 | 아니요 |
| Evaluation Dataset | 검색 결과와 정답 판정을 비교해 품질 측정 | 아니요 |
| Training Dataset | 모델을 학습·파인튜닝하기 위한 사례 | 학습 시 예 |

가상의 query와 qrels:

```json
{
  "query_id": "q001",
  "query": "부산에서 조용하게 야경을 즐길 장소",
  "judgments": [
    {"doc_id": "A", "relevance": 3},
    {"doc_id": "B", "relevance": 2},
    {"doc_id": "C", "relevance": 1},
    {"doc_id": "D", "relevance": 0}
  ]
}
```

`relevance`는 검색 결과의 vector similarity score가 아니라 **별도 기준으로 매긴 관련성 등급**이다.

### 14.2 Offline Evaluation 흐름

```text
Representative Queries ─────────────┐
         ↓                           │
Retrieval System                     │
         ↓                           │
Ranked Results                       │
         ├─────────────── Qrels ─────┘
         ↓
Offline Evaluator
         ├─ Recall@K / Precision@K
         ├─ MRR
         └─ nDCG@K
         ↓
Compare Baseline / Candidate
         ↓
Acceptance Criteria / Regression Check
```

대규모 corpus의 모든 query-document 쌍을 수작업으로 평가하지는 않는다. 대표 query와 후보 문서를 모아 관련성을 판정하는 **Pooling** 방식 등을 활용한다. 다만 미판정 문서(unjudged)를 관련 없음(irrelevant)과 동일시하면 평가가 왜곡될 수 있다.

### 14.3 Retrieval Evaluation은 Machine Learning Training이 아니다

> ❓ **Q. RAG 신뢰도를 높이려면 평가 데이터셋을 대량으로 Training해야 하고, Mac mini나 Jupyter Notebook 등 별도 학습 환경도 필요한가?**
>
> **A. Evaluation Dataset만 구축해서 MRR/nDCG를 계산하는 데 별도 모델 Training은 필요 없다.** Jupyter Notebook은 평가 결과 분석 도구일 수 있지만 필수는 아니며, 작은 평가셋은 일반적인 개발 PC와 Python 스크립트로도 측정할 수 있다. 검색 설정·RRF·Top-K 등을 바꿔 재평가하는 것은 모델 가중치를 학습시키는 것과 다르다.

> ❓ **Q. 이미 학습된 임베딩 모델로 문서 10만 건의 embedding vector를 생성하는 것은 Training인가, Inference인가?**
>
> **A. Inference다.** 모델 파라미터를 업데이트하지 않고 입력 문서에 대해 벡터를 계산한다. Training은 LLM에만 한정되지 않는다. 임베딩 모델, Reranker, Query Classifier 등의 파라미터를 최적화하는 것도 Training이다.

```text
Documents
   ↓
Pretrained Embedding Model (Inference)
   ↓
Vector Index
   ↓
Retrieval / Ranking
   ↓
Qrels-based Evaluation (No Training)
   ↓
Evidence-based Tuning
   ↓
[필요성이 검증되면] Model Fine-tuning
```

평가부터 시작하는 이유: **측정 기준이 없으면 Fine-tuning 이후 개선 여부도 검증할 수 없다.**

### 🔗 Connecting the Dots — Offline Evaluation vs Runtime Process Verification

- **Offline Evaluation:** 라벨이 있는 데이터셋을 대상으로 여러 query의 검색 품질을 통계적으로 평가한다.
- **Runtime Process Verification:** 실제 요청의 단계별 산출물이 검증 가능한 조건(invariant)을 충족하는지 확인하고 실패 지점을 찾는다.
- 둘은 상호보완적이지만 동일하지 않다. Runtime에는 보통 정답 qrels가 없으므로 매 요청마다 MRR/nDCG를 계산할 수 없다.

## ❓ Confusing Point — Embedding Score와 Recall@K / Precision@K의 계산 시점

> ❓ **Q. Recall@K와 Precision@K를 배우다가 헷갈렸다. 도대체 언제 Embedding을 수행하고, 언제 점수를 계산하는가? 외부 문서를 Retrieve할 때 Vector Search를 수행하는 것인가?**
>
> **A. Embedding, Retrieval Ranking Score, Evaluation Metric은 서로 다른 단계다.** Vector Search는 이미 저장된 문서 벡터와 사용자 질의 벡터를 비교해 검색 결과를 만들고, Recall@K / Precision@K는 그 **검색이 끝난 결과**를 정답 판정(Qrels)과 비교해 계산한다.

### 1. Indexing 시점 — Document Embedding

```text
External Documents / API / Crawling
       ↓
Parse / Normalize / (필요 시) Chunk
       ↓
Document Embedding  ← 사전 학습된 Embedding Model의 Inference
       ↓
Vector Index 저장
```

문서 벡터는 일반적으로 미리 계산한다. 매 요청마다 전체 문서를 다시 임베딩하는 것은 아니다. 외부 문서를 수집하는 Fetch/Crawling과 저장된 문서를 검색하는 Retrieval도 구분해야 한다.

### 2. Query / Retrieval 시점 — Query Embedding 및 Similarity Score

```text
User Query
   ↓
Query Embedding  ← Semantic/Vector Search를 할 때
   ↓
Query Vector
   ↓
Vector Index에서 Similarity / Distance 계산
   ↓
Search Score에 따른 Ranking
   ↓
Top-K Documents
```

- **Query Embedding:** 사용자 질의를 벡터로 변환한다.
- **Similarity / Distance Score:** query vector와 document vector의 유사도·거리를 계산한다. ANN은 이 근접한 문서들을 효율적으로 근사 탐색할 수 있다.
- **Top-K:** 검색 엔진이 계산한 순위를 기준으로 K개를 반환한다.
- **Lexical-only 경로:** 키워드 기반 검색만 수행하므로 Query Embedding이 필수는 아니다.
- **Hybrid 경로:** Lexical/Vector 결과를 RRF 등으로 Fusion할 수 있다. 이때 Fusion Score도 정답률 자체가 아니다.

### 3. Evaluation 시점 — Recall@K / Precision@K

```text
Ranked Top-K Documents ───────┐
                              ├→ Compare → Recall@K / Precision@K
Qrels (정답/관련성 판정) ──────┘
```

예시:

```text
Vector Search가 반환한 Top-5:
[A, D, B, E, F]

정답 문서 집합(Qrels):
{A, B, C}

Recall@5    = 2 / 3 ≈ 0.667
Precision@5 = 2 / 5 = 0.4
```

여기서 **Recall@5=0.667은 Embedding 모델이 반환한 유사도 점수가 아니다.** 검색이 끝난 뒤 정답 문서가 얼마나 포함되었는지 측정한 *평가 지표*다. 이 지표는 일반적으로 정답 정보가 있는 Offline Evaluation에서 계산한다.

### 🔗 Connecting the Dots — Score의 책임 분리

| 구분 | 계산 시점 | 역할 |
|---|---|---|
| Embedding Vector | 문서 Indexing / 질의 처리 | 의미 표현 생성 |
| Similarity / BM25 Score | Retrieval 실행 | 후보 문서 검색과 순위 산정 |
| RRF Fusion Score | 여러 검색 결과 결합 시 | 최종 검색 순위 결합 |
| Recall@K / Precision@K | Retrieval 이후, Qrels와 비교 | 검색 품질 평가 |
| MRR / nDCG@K | Retrieval 이후, Qrels와 비교 | 순위 품질 평가 |

> **핵심:** Retrieval Score는 **검색 결과를 만드는 점수**이고, Evaluation Metric은 **그 검색 결과가 얼마나 좋은지 측정하는 점수**다. 둘은 같은 숫자가 아니다.

---

## 15. Retrieval Regression Testing — Golden Set과 Acceptance Criteria

검색 알고리즘을 개선해도 과거에 잘 찾던 결과가 악화될 수 있다. **Retrieval Regression Testing**은 같은 평가 조건에서 기존 버전(Baseline)과 변경 버전(Candidate)을 비교해 이러한 회귀를 발견한다.

```text
Versioned Golden Set (Queries + Qrels)
              ↓
      ┌───────┴───────┐
      ↓               ↓
   Baseline        Candidate
      ↓               ↓
   Evaluation      Evaluation
      └───────┬───────┘
              ↓
      Regression Verifier
      (Acceptance Criteria)
              ↓
          PASS / FAIL
```

같은 Corpus snapshot, 평가용 Query/Qrels, Top-K 설정 등 가능한 평가 조건을 고정해 비교한다. 전체 평균이 개선되어도 특정 query 유형의 품질은 악화될 수 있으므로 질의 그룹별 성능도 살핀다.

### Golden Set / Invariant / Acceptance Criteria의 차이

> ❓ **Q. Acceptance Criteria를 정하기 위해 고정하는 Invariant Dataset을 Golden Set이라고 부르는가?**
>
> **A. 비슷한 방향이지만 서로 구분한다.** Golden Set(Gold Dataset)은 평가용 정답이 포함된 신뢰 가능한 기준 데이터이고, Invariant는 변경 후에도 깨지면 안 되는 시스템 조건이며, Acceptance Criteria는 평가 점수나 조건의 합격 기준이다.

| 개념 | 질문 | 예시 |
|---|---|---|
| Golden Set | 무엇이 관련 문서인가? | Query + Relevant Document IDs / Qrels |
| Invariant | 무엇을 반드시 지켜야 하나? | 반환된 문서 ID가 유효하며 필수 필터를 준수 |
| Acceptance Criteria | 얼마나 잘해야 합격인가? | Recall@5 ≥ 0.90, nDCG@5 ≥ 0.85 |

수치 기준은 설명용 예시이며 실제 제품의 기준을 의미하지 않는다.

```json
{
  "query_id": "q001",
  "query": "가상의 박물관 운영시간",
  "judgments": [{"doc_id": "doc_A", "relevance": 3}]
}
```

### Golden Set은 점진적으로 확장할 수 있다

> ❓ **Q. Golden Set도 incrementally 업데이트할 수 있는가?**
>
> **A. 그렇다.** 새 질의 유형이나 실패 사례를 발견하면 평가용 Query와 Qrels를 검토 후 추가할 수 있다. 정답이 바뀌거나 잘못된 라벨이 발견되면 수정할 수도 있다. 단, **버전을 기록하고 같은 버전끼리 비교**해야 의미 있는 회귀 평가가 가능하다.

```text
golden-set-v1
  → 대표 Query 30개

golden-set-v2
  → 신규 실패 사례 10개 추가 / 라벨 검증

golden-set-v3
  → 실제 정보 변경에 맞춰 정답 갱신
```

평가 데이터셋이 변경되면 점수 차이가 검색 전략 때문인지 평가셋 변화 때문인지 구분해야 한다. 따라서 Baseline과 Candidate를 **동일한 Golden Set 버전**으로 재평가하고 변경 이력을 보존한다.

### 🔗 Connecting the Dots — Stanford Process Verification

- **Evaluator:** Recall@K / MRR / nDCG를 계산한다.
- **Verifier:** 계산 결과와 Acceptance Criteria를 대조한다.
- **Stage Gate:** 통과·보류·실패 시의 진행 경로를 결정한다.

코드의 단위 테스트가 통과하더라도 검색 품질의 회귀가 발생할 수 있다. 즉, **TEST PASS ≠ ENGINEERING DONE**.

---

## Lesson 05 현재 요약

```text
User Query
   ↓
Query Analysis
   ├─ Hard Constraint → Filter
   ├─ Exact Keyword → Lexical Search
   └─ Soft Semantic Preference → Vector Search
        ↓
Parallel or Cascade
        ↓
Fusion
        ↓
Final Top-K
```

---

---

## Glossary

### Corpus
실제 검색 대상 문서의 전체 집합.


### Evaluation Dataset
대표 query와 정답 판단 정보를 이용해 시스템 품질을 측정하는 데이터. 모델 학습과 구분한다.


### Training Dataset
모델 파라미터 학습이나 파인튜닝에 쓰는 데이터.


### Pooling
여러 검색 방식에서 나온 후보 결과를 모아 관련성을 판정하는 평가셋 구축 방법.


### Inference
학습된 모델의 파라미터를 변경하지 않고 새 입력에 대한 출력을 계산하는 작업.


### Fine-tuning
이미 학습된 모델의 파라미터를 특정 목적의 데이터로 추가 학습하는 과정.

---


### Golden Set (Gold Dataset)
대표 Query와 관련성 판정(Qrels)을 담아 검색 품질 평가의 기준으로 사용하는 데이터셋. 버전 관리하며 확장·수정할 수 있다.


### Regression Testing
시스템 변경 뒤 기존 동작이나 품질이 악화되지 않았는지 기준 버전과 비교하는 검증.


### Baseline / Candidate
비교 기준이 되는 기존 버전 / 새로 평가하는 변경 버전.


### Acceptance Criteria
평가 결과의 통과 여부를 판단하기 위해 명시한 기준.


### Invariant
시스템 동작 중 반드시 유지해야 하는 조건.

---


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
