# Lesson 05-5 — Retrieval Failure Analysis

> Previous: [Lesson 05-4 — Evaluation and Regression](./04-evaluation-and-regression.md) · Next: [Lesson 05-6 — Retrieval Optimization](./06-retrieval-optimization.md)

## 16. Retrieval Failure Analysis — 왜 검색 결과가 나빠졌는가?

**Evaluation은 검색 품질이 얼마나 좋은지 측정하고, Failure Analysis는 그 품질이 왜 나빠졌는지 원인을 추적한다.**

> ❓ **Q. Regression Testing에서 Recall@K 또는 nDCG@K가 떨어지면 즉시 Top-K를 높이거나 임베딩 모델을 바꾸면 될까?**
>
> **A. 아니다.** 먼저 정답 문서가 색인되었는지, 후보에 포함되었는지, 최종 순위에서 밀렸는지, Context에서 누락되었는지 확인해야 한다. 원인에 따라 해결책이 다르다.

### 16.1 단계별 Failure 분류

```text
Document Ingestion / Indexing
    → Indexing Failure: 정답 문서가 누락되거나 잘못 저장됨
Query Analysis / Router
    → Routing Failure: 질의 의도·필수 조건이 잘못 해석됨
Candidate Retrieval
    → Candidate Failure: 관련 문서가 검색 후보에 없음
Ranking / Fusion
    → Ranking Failure: 후보에는 있지만 순위에서 밀림
Context Assembly
    → Context Failure: 검색한 근거가 LLM 입력에서 빠짐
Generation
    → Generation Failure: 충분한 근거를 받았지만 답변 오류
```

예시 질의: "조용하고 야경이 좋은 산책 장소 추천".

검색 결과가 시끄러운 장소 위주였다면 QueryRouter가 "조용한" 선호를 놓쳤을 수도 있고, 정답 문서가 아예 색인되지 않았을 수도 있다. **최종 결과만으로는 실패 단계가 식별되지 않는다.**

### 16.2 Candidate Failure vs Ranking Failure

```text
정답 문서: A
후보 상위 10개: [B, C, D, E, F, ...]       → Candidate Failure
후보 상위 10개: [B, C, D, E, A, ...]       → Ranking/Cutoff Failure
최종 반환: Top-3                       → 두 경우 모두 A가 보이지 않음
```

두 경우의 조치가 다르다.

- **Candidate Failure:** 데이터 존재 여부, 필터, 라우팅, ANN 탐색 범위, 검색 질의 등을 조사한다.
- **Ranking Failure:** 후보 확보 후 RRF/Weighted Fusion, Reranker, 최종 Top-K 정책을 조사한다.

후보가 없는 것과 순위가 낮은 것을 구별하려면 **Fusion 이전 후보 문서 ID와 Fusion 이후 순위**를 별도로 남겨야 한다.

### 16.3 Golden Set 기반 Failure Localization

```text
Golden Query q017 — expected relevant document A
                  ↓
A가 index에 있는가?
  ├─ NO  → Ingestion / Indexing 조사
  └─ YES
       ↓
A가 candidate set에 있는가?
  ├─ NO  → Routing / Retrieval 조사
  └─ YES
       ↓
A가 final Top-K에 있는가?
  ├─ NO  → Fusion / Ranking / Cutoff 조사
  └─ YES
       ↓
A가 context에 포함되었는가?
  ├─ NO  → Context Assembly 조사
  └─ YES → Generation 조사
```

이 판단은 관련 문서의 정답 ID를 아는 **Offline Evaluation**에서 특히 강력하다. 실제 사용자 요청에서는 정답 문서 자체를 모르는 경우가 많으므로 모든 단계의 의미적 정확성을 확정할 수는 없다.

## 🔗 Connecting the Dots — Stanford Process Verification × Failure Localization

> ❓ **Q. 각 단계마다 Verifier를 만들고 로그를 남기면 Failure Analysis가 효율적이지 않을까?**
>
> **A. 그렇다.** Process Verification은 중간 산출물에 대한 검증 접근법이고, Structured Logging은 판단 근거를 추적 가능하게 만든다. 단, Verifier와 Logger의 책임은 분리한다.

```text
RAG Phase
  ↓
Verifier[T]                ← 단계별 invariant 검사
  ↓
VerificationResult         ← phase, status, reason, evidence
  ├─→ Stage Gate           ← continue / fallback / retry / review
  └─→ Structured Logger    ← trace_id와 중간 증거 기록
```

### 16.4 유용한 Verifier와 검증 범위

| Verifier | 확인 대상 | 중요한 한계 |
|---|---|---|
| IndexVerifier | 문서 ID·색인 상태·벡터 차원·필수 필드 | 정답 문서가 무엇인지는 별도 정보 필요 |
| RouteVerifier | route 값·filter 구성·분기 근거 | 실제 최적 route 판정에는 라벨/기준 필요 |
| CandidateVerifier | 후보 건수·ID·중복·정답 포함 여부 | 정답 포함 여부는 Golden Set이 필요 |
| RankingVerifier | Fusion 전후 순위·정답의 순위 변화 | 관련성 판정에는 Qrels 필요 |
| ContextVerifier | 토큰 예산·source 연결·선택 문서 | 근거의 충분성을 기계적으로 보장하지 못함 |

**Runtime Verifier**는 스키마, ID, 필수 조건 같은 확인 가능한 불변조건에 집중한다. **Offline Evaluator / Verifier**는 Qrels를 이용해 정답 누락, MRR, nDCG 등의 품질을 측정하고 Acceptance Criteria를 검사한다.

### 16.5 Structured Logging의 목적

```text
trace_id: q017
route: semantic
indexed_doc_id_present: true
candidate_count: 20
gold_doc_rank_before_fusion: 5
gold_doc_rank_after_fusion: 12
final_top_k: 5
context_doc_ids: [B, C, D, E, F]
failure_stage: ranking
```

이 예시는 **평가 환경의 가상 데이터**다. 실제 운영에서는 Qrels의 정답 ID가 없으므로 `gold_doc_rank_*`를 일반적으로 계산할 수 없다.

기록할 정보는 trace/request ID, 선택된 route, 후보 ID/건수, 검색 엔진별 순위, Fusion 이후 순위, 검증 사유, 지연시간 등으로 최소화한다. 원문 질의나 개인·비공개 데이터의 무분별한 로깅은 피한다.

## 👨‍💻 Implementation — Python Protocol과 공통 Verifier

Python Type Hints는 정적 타입 검사기에 의해 검사되지만, 기본적으로 런타임에서 강제되지 않는다. `Protocol[T]`는 명시적 상속 없이도 계약을 충족하는 **Structural Typing**을 지원한다.

```python
from dataclasses import dataclass, field
from typing import Protocol, TypeVar
import logging

T = TypeVar("T", contravariant=True)
logger = logging.getLogger("rag.verification")

@dataclass(frozen=True)
class VerificationResult:
    phase: str
    passed: bool
    reason: str
    evidence: dict[str, object] = field(default_factory=dict)

class Verifier(Protocol[T]):
    def verify(self, artifact: T) -> VerificationResult: ...

class RetrievalVerifier:
    def verify(self, artifact: "RetrievalResult") -> VerificationResult:
        valid = all(doc.id for doc in artifact.documents)
        return VerificationResult(
            phase="retrieval",
            passed=valid,
            reason="valid_ids" if valid else "missing_document_id",
            evidence={"candidate_count": len(artifact.documents)},
        )

def run_verification(verifier: Verifier[T], artifact: T, trace_id: str):
    result = verifier.verify(artifact)
    logger.info(
        "verification_completed",
        extra={
            "trace_id": trace_id,
            "phase": result.phase,
            "passed": result.passed,
            "reason": result.reason,
            **result.evidence,
        },
    )
    return result
```

`RetrievalResult`는 구체적인 검색 결과 모델을 가리키는 예시용 타입이며, 실제 실행 코드에서는 정의하거나 import해야 한다. 구조화된 JSON 로그 출력에는 Logger의 별도 Formatter/Handler 설정이 필요하다.

### ❓ Q&A — Overloading과 Overriding은 어떻게 다른가?

> **Q. 서로 다른 Verifier 구현체를 만들려면 함수 Overloading을 사용하면 되는가?**
>
> **A. 여기서는 Overriding 또는 Structural Polymorphism이 적절하다.** Overloading은 같은 이름의 여러 호출 시그니처를 제공하는 것이고, Overriding은 상위 계약의 메서드를 구현체별로 재정의하는 것이다. `Protocol`은 직접 상속하지 않아도 같은 메서드 계약을 충족하면 호환될 수 있다.

- **Overloading:** 동일 이름의 호출 시그니처를 타입·매개변수 형태에 따라 구분. Python의 `@overload`는 주로 정적 타입 선언이며 여러 런타임 구현을 만들지 않는다.
- **Overriding:** 상위 메서드를 하위 클래스가 재정의.
- **Polymorphism:** 같은 추상 계약으로 서로 다른 구현체를 사용.
- **DIP:** 상위 실행 로직이 구체 Verifier가 아니라 `Verifier[T]`라는 추상화에 의존.

> ❓ **Q. TypeScript처럼 Python도 엄격한 타입 검사가 가능한가?**
>
> **A.** `mypy`, `Pyright` 등을 통해 정적 검사할 수 있다. 다만 Python의 Type Hints는 기본적으로 런타임 강제 규칙이 아니다.

검증 기준이 없는 단계까지 미리 클래스를 만드는 것은 과설계가 될 수 있다. 실제로 필요한 invariant부터 검증하고, 문제를 찾는 데 필요한 중간 결과를 기록한다.

## Lesson 05-5 요약

```text
Evaluate
   ↓
Failure Localization
   ↓
Investigate Index / Route / Candidate / Rank / Context
   ↓
Fix
   ↓
Re-evaluate (same Golden Set version)
```

**평가 점수는 실패 정도를 보여주고, 단계별 검증과 관찰 기록은 실패 위치를 설명한다.**

---

## Glossary

### Retrieval Failure Analysis
검색 품질이 떨어진 원인을 단계별로 분해하고 식별하는 과정.

### Candidate Failure
관련 문서가 Ranking 이전 후보군에 포함되지 않은 상태.

### Ranking Failure
관련 문서가 후보에 있지만 최종 순위나 Top-K 제한 때문에 전달되지 않는 상태.

### Failure Localization
시스템 실패가 시작된 단계나 컴포넌트를 특정하는 작업.

### Process Verification
최종 결과뿐 아니라 중간 과정의 산출물도 검증하는 접근법.

### Structured Logging
추적·분석 가능한 필드 구조로 실행 증거를 기록하는 방식.

### Protocol / Structural Typing
필요한 멤버의 구조가 맞는지에 따라 타입 호환성을 판단하는 방식.

### Overloading / Overriding
하나의 함수 이름에 여러 호출 시그니처를 제공하는 것 / 상위 메서드를 하위 구현체에서 재정의하는 것.

---

## References

### Primary Sources
- [Lewis et al. (2020), Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401) — Retrieval과 Generation의 구분.
- [Lightman et al. (2023), Let's Verify Step by Step](https://arxiv.org/abs/2305.20050) — 중간 단계 검증의 연구 배경.

### Official Documentation
- [Python — typing.Protocol](https://docs.python.org/3/library/typing.html#typing.Protocol) — Python 구조적 타이핑과 타입 힌트.
- [Python — logging](https://docs.python.org/3/library/logging.html) — 로깅 구현 참고.

### Related Public Study
- [Self-Improving Agent Systems — Robust Verification](https://github.com/Seung-zedd/self-improving-agent-systems/blob/main/docs/wiki/02-robust-verification.md) — Stanford CS329A 학습 내용과 Process Verification 연결.

> **구분:** RAG 단계별 Verifier 구조와 Failure Localization 적용은 공개 연구 개념을 바탕으로 한 개인적인 설계 응용이며, 인용한 논문의 원래 아키텍처라고 단정하지 않는다.
