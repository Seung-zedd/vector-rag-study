# Lesson 03 — RAG Retrieval Pipeline

## 1. Indexing Pipeline vs Query Pipeline

RAG는 크게 두 흐름으로 보면 이해하기 쉽다.

### Indexing Pipeline

```text
Raw Data
   ↓
Parse / Clean
   ↓
Chunking
   ↓
Embedding
   ↓
Vector DB
```

검색 가능한 지식을 준비하는 경로다.

### Query Pipeline

```text
User Query
   ↓
Query Analysis / Embedding
   ↓
Retrieval
   ↓
Top-K Documents
   ↓
Context Assembly
   ↓
LLM
   ↓
Answer
```

---

## 2. Retrieval / Augmentation / Generation

```text
Retrieval
→ 근거를 찾는다

Augmentation
→ 찾은 근거로 LLM input을 보강한다

Generation
→ 보강된 context를 사용해 답을 만든다
```

> ❓ **Q. Context Assembly가 사실상 Augmentation인가?**
>
> **A. 거의 그렇다.** Augmentation은 retrieval evidence로 LLM input을 보강하는 더 넓은 개념이고, Context Assembly는 그 evidence를 선택·정렬·그룹화·포맷팅해서 실제 prompt/context에 넣는 핵심 단계다.

---

## 3. Query Embedding → Retrieval → Context Assembly

Semantic path:

```text
User Query
   ↓
Embedding Model
   ↓
Query Vector
   ↓
Vector Search
   ↓
Top-K Documents
```

Context Assembly:

```text
Retrieved Chunks
   ↓
selection
ordering
deduplication
formatting
source attribution
   ↓
LLM Context
```

핵심:

> Retrieval은 **무엇을 가져올지**, Context Assembly는 **가져온 것을 모델에게 어떻게 제시할지** 결정한다.

---

## 4. RAG Failure 분해

```text
Retrieval Failure
→ 필요한 근거를 못 찾음

Context Assembly Failure
→ 근거는 찾았지만 context 구성 실패

Generation Failure
→ 좋은 근거가 있는데 LLM이 잘못 답함
```

Debugging:

```text
User Query
   ↓
무엇이 retrieval 되었나?
   ↓
최종 context에 무엇이 들어갔나?
   ↓
LLM이 무엇을 생성했나?
```

---

## 🔗 Connecting the Dots — RAG Pipeline × Phase Verifier

중간 산출물을 단계별로 검증한다.

```text
Query
→ [Verifier]
→ Retrieval
→ [Verifier]
→ Context Assembly
→ [Verifier]
→ Generation
→ [Verifier]
```

상류 단계는 hard verifier 친화적이다.

```text
document ID
similarity score
Top-K
filter condition
token count
duplicate ratio
truncation
source presence
```

하류 generation으로 갈수록:

```text
faithfulness
usefulness
semantic correctness
```

같은 soft verification 비중이 커진다.

---

## 5. Retrieval Failure

대표 원인:

- embedding할 필드 선택이 잘못됨
- chunking 문제
- query-document 표현 불일치
- Top-K가 너무 작음
- threshold가 너무 높음
- filter가 정답 후보를 제거함
- ANN approximation error

검증 순서:

```text
1. 정답 데이터가 실제로 존재하는가?
2. indexed representation에 필요한 의미가 있는가?
3. query representation이 정상인가?
4. filter가 후보를 제거했는가?
5. Top-K / threshold에서 잘렸는가?
6. ANN이 놓쳤는가?
```

---

## 6. Context Assembly Failure

대표 문제:

- 필요한 chunk truncation
- 나쁜 ordering
- duplicate
- irrelevant noise
- source 구분 불가
- conflicting evidence
- context는 문법적으로 맞지만 의미적으로 약함

> ❓ **Q. 1M context window 같은 frontier model이면 Context Assembly가 덜 중요해지는가?**
>
> **A. 아니다.** context capacity가 늘어도 noise, duplicate, latency, cost, relevance dilution 문제는 남는다.

목표:

```text
context를 많이 채우기
X

답변에 충분한 최소 고품질 evidence 제공
O
```

---

## 🔗 Connecting the Dots — Cubrain × Context Assembly

Context Assembly Failure는 두 층으로 볼 수 있다.

```text
Mechanical Failure
Semantic / Prompt-Design Failure
```

### Mechanical Failure 통제

Cubrain의 PDF context extraction:

```text
PDF page
   ↓
annotation coordinate
   ↓
target region
   ↓
surrounding context padding
   ↓
page-level grouping
   ↓
LLM input
```

3-tier context:

```text
Tier 1 — Target Content
→ 사용자가 직접 표시한 내용

Tier 2 — Underlying Text
→ target과 직접 연결된 text

Tier 3 — Surrounding Context
→ coordinate padding으로 확장한 주변 문맥
```

### Semantic / Prompt-Design Failure 통제

```text
System-level role
→ expert tutor

User-level structured context
→ page context
→ annotations
→ target language
→ generation rules
→ output format
```

핵심:

> 좋은 Context Assembly는 단순 문자열 concat이 아니라 evidence-construction layer이자 instruction-design layer다.

---

## 7. Multimodal Document Handling

```text
Document
   ↓
Type Detection
   ├─ Text → Text Parser
   ├─ Text + Image → Text Parser + OCR/Vision
   └─ Image → OCR / Vision / Image Embedding
```

OCR은 이미지 속 **문자**를 뽑을 때 적합하고, Vision/Image Embedding은 이미지 자체의 시각 의미가 중요할 때 적합하다.

---

## 👨‍💻 Implementation — Document Type Detection

```text
DocumentIngestionService
        ↓
DocumentDetector
        ↓
DocumentType
        ↓
DocumentParser
   ├─ TextParser
   ├─ MultimodalParser
   └─ ImageParser
```

format과 content modality는 섞지 않는 편이 좋다.

```text
DocumentFormat
→ PDF / HTML / DOCX

ContentType
→ TEXT / TEXT_IMAGE / IMAGE
```

---

## 8. Generation Failure

Retrieval과 Context Assembly가 성공했는데 LLM이 잘못 답하는 경우다.

대표 사례:

- context 무시
- context 오해
- source 잘못 결합
- unsupported fact 생성
- 최신/과거 정보 혼동
- claim-source mismatch

핵심 개념은 **Faithfulness**다.

> 생성된 답변이 주어진 context에 얼마나 충실한가.

---

## 9. Lexical / Filter vs Semantic Retrieval

> ❓ **Q. 고유명사나 exact identifier가 명확하면 Lexical / Filter가 semantic search보다 적합한가?**
>
> **A. 대체로 그렇다.** exact identifier, entity name, category, date, code 같은 정보는 semantic similarity보다 exact/lexical retrieval로 처리하는 편이 안정적이다.

---

## 10. Query Analysis & Routing

> ❓ **Q. 모든 user query를 무조건 embedding해야 하는가?**
>
> **A. 아니다.** query structure를 먼저 분석하고 exact / lexical / semantic / hybrid path로 나눌 수 있다.

```text
User Query
   ↓
Query Analysis
   ├─ Entity / ID / exact keyword
   │    → Lexical / Filter
   ├─ Semantic intent
   │    → Vector Search
   └─ 둘 다 존재
        → Hybrid Retrieval
```

---

## 👨‍💻 Implementation — QueryRouter

```text
User Query
   ↓
QueryRouter
   ├─ Exact / Lexical
   │    → LexicalSearchService
   ├─ Semantic
   │    → RetrievalService → VectorStore
   └─ Hybrid
        → HybridRetrievalService
```

Write path와 read path도 분리한다.

```text
Write
→ IngestionService

Read
→ QueryRouter → Retrieval
```

---

## 🔗 Connecting the Dots — Query as Highlight × Anchor-First Retrieval

Query를 가상의 Highlight처럼 볼 수 있다.

```text
User Query
   ↓
Anchor Detection
   ↓
Matched Content Region
   ↓
Tiered Context Expansion
```

3-tier mapping:

```text
Tier 1
→ 직접 매칭된 entity / field / phrase

Tier 2
→ 같은 object의 밀접한 field

Tier 3
→ parent / category / source / nearby context
```

### Lexical-first Cascade

```text
Exact / Entity / Lexical Anchor Search
   ↓
High-confidence anchor?
   ├─ YES → Local Context Expansion
   └─ NO  → Semantic Retrieval → Vector Search
```

핵심:

> deterministic evidence로 충분하면 먼저 사용하고, 표현 차이 때문에 lexical이 부족할 때 semantic retrieval로 내려간다.

---

## 👨‍💻 Implementation — JSON Logical Region

JSON 중심 시스템에서는 억지로 physical coordinate를 만들 필요가 없다.

JSON에는 이미:

```text
JSONPath
field name
object boundary
array index
parent-child relationship
```

이 존재한다.

```text
$.items[42].title
→ anchor

$.items[42]
→ local region

parent/category/source
→ broader context
```

핵심 추상화:

```text
Anchor
→ Region
→ Context Expansion
```

---

## Lesson 03 요약

```text
Indexing
→ 검색 가능한 지식 준비

Query Analysis
→ 어떤 retrieval path를 사용할지 결정

Retrieval
→ 근거 찾기

Context Assembly
→ LLM이 쓰기 좋은 근거 구성

Generation
→ 답변 생성

Verification
→ 각 단계의 상태 검증
```

---

## Glossary

### Indexing Pipeline
검색 전 source data를 retrieval 가능하게 준비하는 과정.

### Query Pipeline
runtime에서 query를 처리하고 근거를 찾고 답변을 만드는 과정.

### Context Assembly
retrieved evidence를 선택·정렬·그룹화·포맷팅해 LLM input으로 만드는 과정.

### Retrieval Failure
관련 근거가 존재하지만 retrieval하지 못한 실패.

### Context Assembly Failure
근거는 찾았지만 context를 잘못 구성한 실패.

### Generation Failure
충분한 context가 있지만 LLM이 잘못 생성한 실패.

### Faithfulness
답변이 supplied context에 얼마나 충실한지를 나타내는 개념.

### Hard Verifier
deterministic/objective rule로 판정 가능한 verifier.

### Soft Verifier
semantic quality처럼 규칙만으로 판정하기 어려운 verifier.

### Query Routing
query를 exact/lexical/semantic/hybrid path 중 어디로 보낼지 결정하는 과정.

### Anchor
context expansion의 시작점이 되는 정확한 lexical/structural/semantic 지점.

### Logical Region
JSON object/field group처럼 구조적으로 경계가 있는 데이터 영역.

### Lexical-first Cascade
exact/lexical search를 먼저 시도하고 부족할 때 semantic retrieval로 fallback하는 전략.
