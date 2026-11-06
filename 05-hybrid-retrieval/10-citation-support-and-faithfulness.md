# Lesson 05-10 — Citation Support & Faithfulness Verification

> Previous: [Lesson 05-9 — RAG Cache Design Patterns](./09-cache-design-patterns.md) · Next: 예정 (Part 22)

## 21. Citation Support — 근거는 실제로 주장을 지지하는가?

> ❓ **Q. Citation Support는 Claim–Document 쌍을 평가하니, Query–Document 관련성을 기록한 qrels와 비슷한가?**
>
> **A. 관계를 판정한다는 구조는 비슷하다.** 하지만 qrels는 **검색 질의와 문서의 관련성**, Citation Support는 **생성된 주장과 인용 근거의 지지 관계**를 판단한다. 관련 있는 문서가 반드시 주장에 동의하는 것은 아니다.

| 관점 | 평가 입력 | 판단 목표 |
|---|---|---|
| Retrieval qrels | Query, Document | Relevance: 이 질의에 관련 있는 문서인가? |
| Citation Support | Claim, Cited Evidence | Entailment: 이 근거로 해당 주장을 뒷받침할 수 있는가? |

예: 질의가 입장료를 묻고 문서는 입장료 `15,000원`을 명시하지만 생성된 Claim이 `무료`라면, 문서는 **Query-Relevant**여도 해당 Claim에는 **Contradiction**이다. 모두 합성 예시다.

### 21.1 Citation Presence / Validity / Support

- **Presence:** 답변에 출처 표기가 있는가?
- **Validity:** 참조한 출처 ID가 실제 검색 결과/검증된 원문에 대응하는가?
- **Support:** 출처의 내용이 해당 Claim을 뒷받침하는가?

**Presence ≠ Validity ≠ Support**. 존재하는 출처에 잘못된 주장을 붙일 수 있다.

## 21.2 Claim-Level Verification과 NLI

생성 답변을 의미적으로 독립된 **Atomic Claim**들로 나눈다. 각 Claim의 cited evidence를 Premise로, Claim을 Hypothesis로 삼아 NLI(Natural Language Inference)를 수행할 수 있다.

| NLI 결과 | 의미 | Citation 평가 |
|---|---|---|
| Entailment | Premise가 Hypothesis를 지지 | Supported |
| Contradiction | Premise가 Hypothesis에 반함 | Contradicted |
| Neutral | Premise만으로 결론 불가 | Not established |

`Neutral`은 거짓이라는 뜻이 아니다. 그리고 NLI 분류기의 출력은 형식 논리의 완전한 증명이 아니라 **모델 추정치**다.

### 🔗 Connecting the Dots — Formal Logic

```math
P \models H
```

전제 `P`가 참인 모든 모델에서 `H`도 참이라는 **의미적 함의(Semantic Entailment)**이다. 반면:

```math
P \vdash H
```

이는 정해진 추론 규칙으로 `H`를 유도할 수 있다는 **증명 가능성(Syntactic Derivability)**이다. 자연어 NLI 모델의 분류를 곧 엄밀한 `\models` 증명으로 취급하지 않는다.

## 21.3 Faithfulness와 Answer Correctness

- **Faithfulness:** 답변의 주장들이 제공된 Context에 충실한가?
- **Answer Correctness:** 답변이 실제 정답/신뢰할 수 있는 기준에 부합하는가?

틀린 원문에 충실한 답변은 **Faithful하지만 Incorrect**할 수 있다. 검증할 Claim이 존재하고 각 Claim의 지원 여부를 판정한다고 할 때 설명용 Faithfulness 비율은:

```math
\mathrm{Faithfulness}
=\frac{\#\{\mathrm{Supported\ Claims}\}}{\#\{\mathrm{All\ Claims}\}}
```

Claims 분해 자체가 틀리거나 복수 근거의 조합을 놓치면 이 값도 왜곡된다.

---

## 21.4 🔗 Related Public Study — Retrieval Metrics 복습 후 Citation Metrics 비교

> ❓ **Q. Precision@K와 Recall@K를 잊어버리면 Citation Precision/Recall이 헷갈린다. 기존 공식을 바로 옆에 놓고 볼 수 없을까?**
>
> **A. 분수의 형태는 비슷하지만 평가 단위와 분모가 다르다.** 아래는 [Lesson 05-3 — Retrieval Metrics and Verification](./03-retrieval-metrics-and-verification.md)의 동일한 합성 예시를 수식으로 옮긴 것이다.

### Retrieval Precision@K / Recall@K

- `R`: 평가 대상 Query에 대해 relevant한 문서의 집합
- `D_K`: Retriever가 반환한 Top-K 문서의 집합 (중복 없는 K개 결과라고 가정)
- `R \cap D_K`: Relevant이면서 실제 검색된 문서의 교집합

```math
\mathrm{Precision@K}=\frac{|R\cap D_K|}{|D_K|}
```

```math
\mathrm{Recall@K}=\frac{|R\cap D_K|}{|R|}
```

```math
R=\{A,B,C\},\qquad D_5=\{A,D,B,E,F\}
```

```math
R\cap D_5=\{A,B\},\quad |R\cap D_5|=2
```

```math
\mathrm{Precision@5}=\frac25=0.4,\qquad
\mathrm{Recall@5}=\frac23\approx0.667
```

**수직선 `|R|`의 의미에 주의:** 이 문맥의 `|R|`은 **절댓값이 아니라 집합의 기수(Cardinality), 즉 원소의 개수**다. `|R|=3`, `|D_5|=5`, `|R\cap D_5|=2`. 표기 모양은 절댓값과 동일하지만 피연산자의 종류가 다르다.

### 조건부확률 vs 베이즈 정리

유한한 문서 전체에서 균등하게 문서 하나를 뽑는 **조건부확률의 집합 비율**로 해석할 수도 있다.

```math
\mathrm{Precision@K}=P(R\mid D_K)
```

```math
\mathrm{Recall@K}=P(D_K\mid R)
```

여기서 `R`과 `D_K`를 각각 `문서가 relevant하다`, `문서가 Top-K에 있다`라는 사건으로 해석한다. 이것은 **조건부확률의 정의**이며 그 자체가 베이즈 정리는 아니다.

```math
P(R\mid D_K)=\frac{P(D_K\mid R)P(R)}{P(D_K)}
```

이 식처럼 조건부확률을 뒤집어 사전확률·우도로 연결할 때 **베이즈 정리**가 된다. 또한 이것은 고정된 질의·검색 집합에서의 비율을 확률로 재표현한 것이지, 검색 성능을 베이지안 모델로 추정했다는 뜻은 아니다.

### Citation Precision / Citation Recall (학습용 단순화)

- `C`: 평가할 생성 Claim의 집합
- `L`: 생성된 (Claim, Citation) **연결**의 집합
- `S(c,E_c)\in\{0,1\}`: Claim에 달린 근거들의 조합이 Claim을 충분히 지지하면 1
- `V(\ell)\in\{0,1\}`: 인용 연결이 적절하고 불필요한 중복이 아니면 1 (평가 정책에 의존)

```math
\mathrm{CitationRecall}=\frac{\sum_{c\in C}S(c,E_c)}{|C|}
```

```math
\mathrm{CitationPrecision}=\frac{\sum_{\ell\in L}V(\ell)}{|L|}
```

| 평가 지표 | 분자 | 분모 |
|---|---|---|
| Precision@K | Top-K에 포함된 관련 문서 수 $`\lvert R\cap D_K\rvert`$ | Top-K 문서 수 $`\lvert D_K\rvert`$ |
| Recall@K | Top-K에 포함된 관련 문서 수 $`\lvert R\cap D_K\rvert`$ | 전체 관련 문서 수 $`\lvert R\rvert`$ |
| Citation Precision | 적절한 인용 연결 수 $`\sum_{\ell\in L}V(\ell)`$ | 생성된 인용 연결 수 $`\lvert L\rvert`$ |
| Citation Recall | 근거가 충분한 Claim 수 $`\sum_{c\in C}S(c,E_c)`$ | 평가 대상 Claim 수 $`\lvert C\rvert`$ |

**핵심:** Retrieval의 Precision@K와 Recall@K는 **동일한 교집합** `R ∩ D_K`의 원소 수를 공통 분자로 사용한다. 반면 위에서 정의한 Citation Precision/Recall은 평가 단위(인용 연결 / Claim)가 서로 달라 분자도 다르다.

합성 예: 4개 Claim 중 2개에 충분한 근거가 있으면 Citation Recall = `2/4`; 4개 인용 연결 중 2개가 적절하면 Citation Precision = `2/4`. **분모는 서로 다르고, 점수가 같아도 동일한 품질 속성을 의미하지 않는다.**

**평가 정의 주의:** 위 Citation Metrics는 **Atomic Claim 기반 학습용 정의**이다. ALCE는 문장 단위 Citation Recall과 필요성/중복성을 고려한 Citation Precision을 제안한다. 단일 Citation이 혼자서는 충분하지 않아도 다른 Citation과 결합하여 지지할 수 있으므로 실제 구현에서 `V`, `S`의 판정 기준을 명시해야 한다. `|C|=0` 또는 `|L|=0`이면 0으로 나누지 않고 평가 정책상 별도 처리한다.

## 21.5 End-to-End 검증 설계

```text
Query → Retriever → qrels 기반 Retrieval Evaluation
                         ├── Precision@K / Recall@K
                         └── MRR / nDCG
      → Retrieved Context → LLM Answer
      → Claim Extraction → Citation Presence / Validity
      → Citation Support (NLI) → Citation Precision / Recall
      → Stage Gate: Accept / Repair / Expand / Abstain
```

**높은 Retrieval Recall@K ≠ 높은 Citation Recall.** 필요한 문서를 잘 찾았어도 생성 모델이 잘못 인용할 수 있다. 또한 **높은 Citation Recall ≠ Answer Completeness**: 생성된 일부 주장만 정확하게 인용하고 중요한 질문 요소를 누락할 수 있다.

### 👨‍💻 Implementation — 역할 분리 스케치

```python
from dataclasses import dataclass
from enum import Enum
from typing import Protocol

class NliVerdict(Enum):
    ENTAILMENT = "entailment"
    CONTRADICTION = "contradiction"
    NEUTRAL = "neutral"

class NliModel(Protocol):
    def predict(self, premise: str, hypothesis: str) -> NliVerdict: ...

@dataclass(frozen=True)
class CitationCheck:
    claim_id: str
    document_id: str
    verdict: NliVerdict

class CitationSupportVerifier:
    def __init__(self, model: NliModel) -> None:
        self.model = model

    def verify(
        self, claim_id: str, claim: str,
        document_id: str, evidence_text: str
    ) -> CitationCheck:
        return CitationCheck(
            claim_id=claim_id,
            document_id=document_id,
            verdict=self.model.predict(evidence_text, claim),
        )
```

이것은 **단일 Claim–Evidence 쌍**의 검증 계약 예시이며, Claim 분해·출처 ID 검사·복수 출처 결합·모델 신뢰도·정책 임계값·운영 관측은 포함하지 않는다. **Verifier는 판정, Stage Gate는 조치 선택**을 담당한다.

---

## Glossary

- **Citation Support:** 인용 근거가 답변의 개별 주장을 뒷받침하는지의 판단.
- **Atomic Claim:** 답변에서 추출한 독립적인 사실 주장 단위.
- **NLI:** Premise와 Hypothesis의 Entailment / Contradiction / Neutral 관계를 분류하는 작업.
- **Faithfulness:** 생성된 답변의 주장이 제공된 근거에 충실한 정도.
- **Answer Correctness:** 답변이 외부의 올바른 기준 답변/사실과 부합하는 정도.
- **qrels:** Query–Document 관련성 판단 레이블 집합.
- **Cardinality:** 집합의 원소 개수. 집합 `R`의 `|R|`은 절댓값이 아니라 기수.
- **Conditional Probability:** 두 사건의 교집합을 조건 사건으로 나누는 확률.
- **Bayes' Theorem:** 조건부확률 관계를 이용해 `P(A|B)`와 `P(B|A)`를 연결하는 공식.

## References

### Primary Sources
- Gao et al. (2023). [Enabling Large Language Models to Generate Text with Citations (ALCE)](https://arxiv.org/abs/2305.14627) — Citation Recall / Precision 평가의 연구적 맥락.
- Dagan, Glickman & Magnini (2006). [The PASCAL Recognising Textual Entailment Challenge](https://link.springer.com/chapter/10.1007/11736790_9) — Textual Entailment의 배경.

### Official Documentation
- [Ragas — Faithfulness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) — Claim 분해와 근거 지지 기반 평가 개념.
- [TREC — Text REtrieval Conference](https://trec.nist.gov/) — qrels와 Retrieval 평가.

### Related Public Study
- [Lesson 05-3 — Retrieval Metrics and Verification](./03-retrieval-metrics-and-verification.md) — Recall@K / Precision@K / MRR / nDCG와 qrels.
- [Lesson 05-8 — Context Engineering & Retrieval Caching](./08-context-engineering-and-cache.md) — Citation Presence / Validity / Support의 개념적 출발점.
- [Lesson 05-9 — Cache Design Patterns](./09-cache-design-patterns.md) — Protocol / Verifier / Stage Gate 책임 분리.

> **구분:** Citation Precision/Recall의 Atomic Claim 기반 이진식, 컴포넌트 추상화 및 Stage Gate는 학습을 위한 설계·정리이다. ALCE 논문의 평가 구현과 동일하다고 단정하지 않는다.
