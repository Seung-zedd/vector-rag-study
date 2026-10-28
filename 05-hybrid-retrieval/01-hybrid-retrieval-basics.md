# Lesson 05 — Hybrid Retrieval

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

```text
rel_i = i번째 검색 결과의 관련성 등급
예: 3=매우 관련, 2=관련, 1=약간 관련, 0=무관

DCG@K = Σ(i=1..K) (2^rel_i - 1) / log₂(i+1)
IDCG@K = 이상적인 정렬 결과의 DCG@K
nDCG@K = DCG@K / IDCG@K
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


## 🔗 Connecting the Dots — Stanford Process Verification × RAG

Stanford CS329A Robust Verification에서 다룬 **Process Verification**은 최종 결과뿐 아니라 중간 단계의 검증에 초점을 맞춘다.

> ❓ **Q. Process Verification을 RAG Pipeline의 각 단계에 Verifier로 적용하고, 공통 추상화를 오버라이딩하는 구현체로 확장하면 어떨까?**
>
> **A. 타당한 설계 방향이다.** 다만 Process Verification은 *검증 방법론*이고, `Verifier`는 그 방법론을 구현하는 *컴포넌트 인터페이스*다. 다형성이나 Strategy Pattern을 통해 단계별 검증 정책을 분리할 수 있다.

```text
QueryRouter         → RouteVerifier
                         ↓ PASS / FAIL / FALLBACK
Retriever           → RetrievalVerifier
                         ↓ PASS / FAIL / RETRY
ContextAssembler    → ContextVerifier
                         ↓ PASS / FAIL / REPAIR
Generator           → AnswerVerifier
                         ↓ PASS / FAIL / ESCALATE
```

핵심 목적은 *Failure Localization*: 최종 답변이 잘못되었을 때 어떤 단계에서 문제가 시작됐는지 추적하는 것.

**Verifier / Stage Gate / Process Verification 구분**

- Process Verification: 중간 단계의 정확성을 검사하는 검증 접근법
- Verifier: 단계별 검증 로직을 담는 컴포넌트
- Stage Gate: 검증 결과에 따라 계속 진행, 재시도, fallback, 중단을 결정하는 제어 정책
- Execution trace / trajectory: 실제 실행 순서와 검증 증거를 기록한 데이터. Process Verification과 동일어가 아님

모든 검증 실패가 곧 중단을 의미하는 것은 아니다. 검색 후보 부족은 fallback, 인용 누락은 context repair로 이어질 수 있다.

### 👨‍💻 Implementation — 공통 Verifier 계약

```python
from dataclasses import dataclass
from typing import Generic, Protocol, TypeVar

T = TypeVar("T", contravariant=True)

@dataclass(frozen=True)
class VerificationResult:
    passed: bool
    reason: str
    evidence: dict[str, object]

class Verifier(Protocol[T]):
    def verify(self, artifact: T) -> VerificationResult: ...

class RetrievalVerifier:
    def verify(self, artifact: "RetrievalResult") -> VerificationResult:
        # 실제 retrieval invariant에 맞춰 구현
        ...
```

검증 인터페이스를 통일하고, 검색 결과의 필드/출처/건수 같은 *실시간 검증 가능한 조건*은 Runtime Verifier가 담당한다. 반면 **MRR/nDCG는 보통 qrels가 필요한 Offline Evaluator**로 분리한다. 모든 phase에 클래스를 미리 만들기보다 실제 invariant가 발견된 단계부터 적용한다.

참조: [Self-Improving Agent Systems — Robust Verification](https://github.com/Seung-zedd/self-improving-agent-systems/blob/main/docs/wiki/02-robust-verification.md)



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

### Verifier
지표·로그·중간 결과를 명시한 acceptance criteria와 비교해 검증하는 절차 또는 컴포넌트.

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
