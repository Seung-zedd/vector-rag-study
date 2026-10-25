# Lesson 03 — RAG Retrieval Pipeline

## 1. Indexing Pipeline vs Query Pipeline

RAG is easier to understand when it is divided into two pipelines.

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

This is the preparation path that makes data searchable.

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

The key distinction is:

```text
Indexing Pipeline
→ prepares searchable knowledge

Query Pipeline
→ retrieves and uses that knowledge at request time
```

---

## 2. Retrieval, Augmentation, and Generation

RAG stands for:

```text
Retrieval
Augmented
Generation
```

### Retrieval

Find relevant external evidence.

### Augmentation

Turn the retrieved evidence into context that the LLM can use.

### Generation

Produce an answer from the augmented context.

> ❓ **Q. Is Context Assembly basically the Augmentation step?**
>
> **A. Mostly yes.** Augmentation is the broader act of enriching the LLM input with retrieved external evidence, and Context Assembly is the main step where that evidence is selected, ordered, grouped, formatted, and inserted into the prompt.

A useful mental model is:

```text
LLM parametric memory
        +
retrieved external evidence
        ↓
augmented context
```

---

## 3. Query Embedding → Retrieval → Context Assembly

A semantic query path looks like:

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

Then Context Assembly transforms retrieved items into something useful for the LLM.

```text
Retrieved chunks
   ↓
selection
ordering
deduplication
formatting
source attribution
   ↓
LLM Context
```

The distinction is:

> Retrieval decides **what to bring**.  
> Context Assembly decides **how to present it to the model**.

---

## 4. RAG Failure Decomposition

A wrong RAG answer should not be treated as one undifferentiated failure.

A useful decomposition is:

```text
Retrieval Failure
→ needed evidence was not retrieved

Context Assembly Failure
→ evidence was retrieved but assembled poorly

Generation Failure
→ good evidence was provided but the LLM answered incorrectly
```

A practical debugging trace is:

```text
User Query
   ↓
What was retrieved?
   ↓
What entered the final context?
   ↓
What answer did the LLM generate?
```

---

## 🔗 Connecting the Dots — RAG Pipeline × Phase Verifier

A RAG pipeline can be debugged similarly to a phase-based system with verifiers between stages.

```text
User Query
   ↓
[Verifier]
query interpretation / embedding

Retrieval
   ↓
[Verifier]
retrieved IDs / scores / filters

Context Assembly
   ↓
[Verifier]
chunks / order / truncation / duplication

Generation
   ↓
[Verifier]
format / grounding / semantic quality
```

The general abstraction is:

> Do not validate only the final output. Validate intermediate artifacts so that failure propagation can be localized.

### Hard verifier vs soft verifier

Many retrieval and context checks are hard-verifier friendly because they can be inspected objectively.

Examples:

```text
retrieved document IDs
similarity scores
Top-K boundary
filter conditions
token count
duplicate ratio
truncation
source presence
```

Generation is different.

```text
format / required fields
→ hard verification possible

faithfulness / usefulness / semantic correctness
→ often requires soft verification
```

So RAG tends to become more soft-verifier-heavy toward the generation stage.

---

## 5. Retrieval Failure

Retrieval failure means the needed evidence exists in the system but is not successfully returned.

Common causes include:

- wrong fields selected for embedding
- poor chunking
- query-document phrasing mismatch
- Top-K too small
- threshold too strict
- filters removing the correct candidate
- ANN approximation error

A useful diagnostic order is:

```text
1. Does the correct source data exist?
2. Is the needed meaning present in the indexed representation?
3. Is the query represented correctly?
4. Did a filter remove the candidate?
5. Did Top-K or threshold cut it off?
6. Did ANN miss it?
```

---

## 6. Context Assembly Failure

Retrieval can succeed and the final answer can still fail because the context was assembled badly.

Common problems:

- a necessary chunk is truncated
- chunk order is poor
- duplicates consume context budget
- irrelevant chunks add noise
- sources are not distinguishable
- conflicting facts are mixed together
- context is technically valid but semantically weak

A larger context window does not eliminate these issues.

> ❓ **Q. If frontier models support very large context windows, is Context Assembly still important?**
>
> **A. Yes.** Large context windows increase capacity, but they do not eliminate retrieval noise, duplication, latency, cost, relevance dilution, or the risk that important evidence becomes harder to use effectively.

The goal is not:

```text
fill as much context as possible
```

The better goal is:

```text
provide the smallest high-quality evidence set
that is sufficient to answer the query
```

---

## 🔗 Connecting the Dots — Cubrain × Context Assembly

Cubrain provides a useful concrete example of how Context Assembly can fail in two different ways:

```text
Context Assembly Failure
├─ Mechanical Failure
└─ Semantic / Prompt-Design Failure
```

### 1. Mechanical Failure Control

Cubrain does not treat a PDF as one flat block of text.

Instead, it extracts information through physically constrained regions and local context expansion.

A generalized version of its pipeline is:

```text
PDF page
   ↓
annotation coordinate detection
   ↓
target-region extraction
   ↓
surrounding-context padding
   ↓
page-level grouping
   ↓
LLM input
```

Its context structure can be understood as a three-tier evidence model:

```text
Tier 1 — Target Content
→ the user-selected or visually marked content itself

Tier 2 — Underlying Text
→ text directly beneath or associated with the target region

Tier 3 — Surrounding Context
→ nearby paragraph-level context added through coordinate padding
```

This directly reduces mechanical Context Assembly failures such as:

- target text being cut off
- losing the local semantic neighborhood
- mixing unrelated parts of the page
- passing excessive irrelevant text
- failing to associate an image or annotation with its nearby text

This is conceptually similar to RAG systems that recover neighboring chunks or parent context around a retrieved chunk.

### 2. Semantic / Prompt-Design Failure Control

Cubrain also controls the semantic interpretation of assembled context through explicit prompt design.

Its generation layer separates:

```text
System-level role
→ "expert tutor"

User-level structured context
→ page context
→ annotations
→ target language
→ generation rules
→ output format
```

The prompt further defines how different evidence types should be interpreted.

For example, a generalized version is:

```text
Highlight
→ conceptual question

Underline
→ factual question

Ink
→ visual content + surrounding text

FreeText
→ user note + surrounding context
```

This addresses a different class of failure.

Even if the mechanically assembled evidence is correct, the model can still fail if:

- the evidence hierarchy is unclear
- source roles are ambiguous
- the model does not know how to interpret each annotation type
- conflicting instructions exist
- the output contract is underspecified
- the model is not told how to synthesize multimodal evidence

So Context Assembly quality depends on both:

```text
Mechanical assembly
→ What evidence is physically included?

Semantic assembly
→ How is that evidence explained and constrained for the model?
```

The transferable lesson is:

> A good Context Assembly layer is not only a string concatenator. It is both an evidence-construction layer and an instruction-design layer.

### Shared abstraction

```text
large source
   ↓
select task-relevant evidence
   ↓
restore necessary local context
   ↓
group related evidence
   ↓
define evidence roles and instructions
   ↓
assemble final model input
```

Useful transferable ideas include:

1. Do not pass the entire source when only a small region matters.
2. Add nearby context when the selected region is too narrow.
3. Batch related evidence when coherence matters.
4. Reduce duplication before generation.
5. Combine multiple evidence types when necessary.
6. Explicitly define how the model should interpret each evidence type.
7. Treat output format and reasoning constraints as part of Context Assembly.

---

## 7. Multimodal Document Handling

A retrieval system may ingest several kinds of unstructured documents.

```text
Document
   ↓
Type Detection
   ├─ Text
   │   → Text Parser
   │
   ├─ Text + Image
   │   ├─ Text Parser
   │   └─ OCR / Vision
   │
   └─ Image only
       ├─ OCR
       └─ Vision / Image Embedding
```

OCR is appropriate when the information to retrieve is text embedded inside an image.

Vision or image embeddings are more appropriate when the visual content itself carries meaning.

A robust contextualized representation may combine:

```text
page title
+
surrounding text
+
alt text / caption
+
OCR result
+
vision-derived description
```

---

## 👨‍💻 Implementation — Document Type Detection and Parsing

One possible implementation idea is to separate document classification from parsing.

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

A simple closed set can be represented as an enum:

```text
TEXT
TEXT_IMAGE
IMAGE
```

If format and content modality become separate concerns, keep them separate.

```text
DocumentFormat
→ PDF / HTML / DOCX

ContentType
→ TEXT / TEXT_IMAGE / IMAGE
```

This avoids mixing file format with semantic content type.

---

## 8. Generation Failure

Generation failure happens when retrieval and context construction are correct but the LLM still produces a wrong answer.

Typical cases include:

- ignoring the supplied context
- misinterpreting the context
- combining sources incorrectly
- inventing unsupported facts
- confusing stale and current information
- mismatching claims and sources

A central concept is **faithfulness**.

> Faithfulness measures how well the generated answer stays supported by the supplied context.

Example:

```text
Context:
Admission fee is 5,000 KRW.

Answer:
Admission is 5,000 KRW and parking is free.
```

The parking claim is unsupported, so faithfulness is low.

---

## 9. Lexical / Filter vs Semantic Retrieval

Not every query should be routed through vector search.

> ❓ **Q. If a query contains a clear entity name or exact identifier, would Lexical / Filter retrieval produce a more faithful answer than semantic search?**
>
> **A. Often yes.** Exact identifiers, entity names, categories, dates, codes, and structured attributes are usually better handled by exact or lexical retrieval than by semantic similarity.

Example:

```text
"A 관광지는 월요일에 갈 수 있어?"
```

The entity is already explicit.

```text
entity = "A 관광지"
```

A more precise path may be:

```text
entity match
   ↓
exact / lexical lookup
   ↓
structured fields or payload
   ↓
answer
```

Lexical retrieval does not guarantee faithfulness by itself, but it can reduce ambiguity and improve the quality of the evidence passed to generation.

---

## 10. Query Analysis and Routing

> ❓ **Q. Should every user query be embedded immediately?**
>
> **A. No.** Query structure can be analyzed first so that exact, lexical, semantic, and hybrid parts are handled differently.

A simple routing model is:

```text
User Query
   ↓
Query Analysis
   ├─ Entity / ID / exact keyword detected
   │    → Lexical / Filter
   │
   ├─ Semantic intent dominant
   │    → Vector Search
   │
   └─ Both are present
        → Hybrid Retrieval
```

Examples:

```text
"contentId 12345"
→ exact lookup

"A 관광지 운영시간"
→ entity match + structured lookup

"아이와 가기 좋은 조용한 실내 장소"
→ semantic retrieval

"부산 광안리에서 야경 좋은 곳"
→ lexical/filter + semantic retrieval
```

The key idea is:

> Query embedding should be a deliberate routing decision, not an automatic first step for every request.

---

## 👨‍💻 Implementation — QueryRouter and Separation of Read vs Write Paths

A possible implementation abstraction is a high-level query router.

```text
User Query
   ↓
QueryRouter
   ├─ Exact / Lexical path
   │    → LexicalSearchService
   │
   ├─ Semantic path
   │    → RetrievalService
   │         ↓
   │      VectorStore
   │
   └─ Hybrid path
        → HybridRetrievalService
             ├─ LexicalSearchService
             └─ RetrievalService
```

This should remain separate from ingestion.

```text
Write Path
→ IngestionService
→ parse / normalize / chunk / embed / store

Read Path
→ QueryRouter
→ lexical / semantic / hybrid retrieval
```

This avoids embedding and vector search for queries that can be answered more efficiently with exact or structured retrieval.

---


## 🔗 Connecting the Dots — Query as Highlight × Anchor-First Retrieval

A useful way to reinterpret a user query is to treat it like a virtual highlight.

In a PDF annotation system, a highlight already provides a precise anchor:

```text
Highlight
   ↓
Target Region
   ↓
Underlying Text
   ↓
Surrounding Context
```

In retrieval, the query does not initially have a physical location, so the first task is to discover an anchor inside the indexed data.

```text
User Query
   ↓
Anchor Detection
   ↓
Matched Content Region
   ↓
Tiered Context Expansion
```

A practical three-tier mapping is:

```text
Tier 1
→ directly matched entity / field / phrase

Tier 2
→ the containing object or closely related fields

Tier 3
→ broader parent, category, source, or neighboring context
```

This creates a useful bridge between annotation-based context extraction and retrieval-based context construction.

### Lexical-first Cascade

Exact and lexical signals are usually more deterministic than semantic similarity.

A good default routing strategy is therefore:

```text
User Query
   ↓
Exact / Entity / Lexical Anchor Search
   ↓
High-confidence anchor?
   ├─ YES
   │    → Local Context Expansion
   │    → Answer / downstream generation
   │
   └─ NO
        → Semantic Retrieval
        → Vector Search
```

This does not mean lexical retrieval is universally better than embedding.

Instead:

> Use deterministic evidence first when it is sufficient, and use semantic retrieval when lexical evidence cannot express the user's intent.

This can reduce unnecessary embedding calls, vector-search latency, and ambiguity for exact queries.

---

## 👨‍💻 Implementation — JSON Logical Regions Instead of Artificial Coordinates

For JSON-centric crawling pipelines, introducing artificial physical coordinates is usually unnecessary.

A JSON document already contains structural location information such as:

```text
JSONPath
field name
object boundary
array index
parent-child relationship
```

Therefore, rather than pretending that JSON items are PDF pages with synthetic x/y coordinates, it is simpler to use logical regions.

Example:

```text
$.items[42].title
→ anchor

$.items[42]
→ local object region

parent/category/source metadata
→ broader context
```

A compact architecture can be:

```text
QueryRouter
   ↓
Lexical / Filter Anchor Detection
   ↓
JsonRegionResolver
   ↓
3-tier Context Expansion
   ↓
Semantic Retrieval fallback when needed
```

This preserves the useful abstraction from coordinate-based extraction without introducing unnecessary geometry.

The important abstraction is not:

```text
physical clipping box
```

but:

```text
Anchor
→ Region
→ Context Expansion
```

When the ingestion domain is predominantly JSON-based, a single JSON-oriented resolver is usually preferable to prematurely building multiple format-specific resolvers.

## Lesson 03 Summary

```text
Indexing
→ prepare searchable knowledge

Query Analysis
→ decide how the request should be routed

Retrieval
→ find evidence

Context Assembly
→ construct model-ready evidence

Generation
→ produce the answer

Verification
→ inspect each stage independently
```

The key lesson is:

> RAG quality is not determined by vector search alone. It depends on routing, retrieval, context construction, generation, and verification working together.

---

## Glossary

### Indexing Pipeline
The offline or pre-query process that prepares source data for retrieval.

### Query Pipeline
The runtime process that handles a user request, retrieves evidence, assembles context, and generates an answer.

### Augmentation
The process of enriching an LLM input with externally retrieved evidence.

### Context Assembly
The process of selecting, ordering, grouping, formatting, and inserting retrieved evidence into the model input.

### Retrieval Failure
A failure where relevant evidence exists but is not successfully retrieved.

### Context Assembly Failure
A failure where evidence is retrieved but poorly constructed into the final model context.

### Generation Failure
A failure where sufficient context is provided but the LLM still produces an incorrect or unsupported answer.

### Faithfulness
The degree to which a generated answer is supported by the supplied context.

### Hard Verifier
A verifier that can evaluate a condition using deterministic or objectively measurable rules.

### Soft Verifier
A verifier that evaluates semantic quality, usefulness, or correctness where deterministic pass/fail rules are insufficient.

### Query Routing
The process of deciding whether a query should use exact, lexical, semantic, or hybrid retrieval.

### OCR
Optical Character Recognition. A technique for extracting textual content from images.

### Multimodal
A setting in which multiple information types such as text and images are processed together.

### Mechanical Context Assembly
The deterministic construction of model input through selection, ordering, grouping, truncation control, deduplication, region extraction, and token-budget management.

### Semantic Context Assembly
The instruction-level design that tells the model how to interpret, prioritize, and synthesize the evidence placed into context.

### Evidence Hierarchy
An explicit ordering or role assignment that distinguishes primary evidence from supporting or surrounding context.

### Anchor
A precise lexical, structural, or semantic point used as the starting location for context expansion.

### Logical Region
A structurally bounded area of data, such as a JSON object or field group, used instead of physical coordinates.

### Lexical-first Cascade
A routing strategy that attempts exact or lexical retrieval first and falls back to semantic retrieval when deterministic matching is insufficient.
