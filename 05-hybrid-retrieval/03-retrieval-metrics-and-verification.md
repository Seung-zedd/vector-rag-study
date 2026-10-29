# Lesson 05-3 — Retrieval Metrics and Verification

> Previous: [Lesson 05-2](02-query-routing-and-fusion.md) · Next: [Lesson 05-4](04-evaluation-and-regression.md)


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

---

## Glossary

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
