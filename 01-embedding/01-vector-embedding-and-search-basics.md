# Lesson 01 — Embedding, Vector Similarity, Metadata, and Lexical Search

> **Question that led here**
> What determines the numeric values when text is converted into an embedding?

## 1. Embedding

An embedding model converts text into a fixed-length numeric vector.

```text
"고양이"
   ↓
Embedding Model
   ↓
[0.8, 0.1, 0.7, ...]
```

The individual values are not manually assigned human-readable attributes such as:

```text
0.8 = cuteness
0.1 = animal-ness
0.7 = amount of fur
```

Instead, the model learns a high-dimensional representation from patterns and relationships in training data.

The important part is therefore not the meaning of each individual float, but the **relative position and direction of vectors in the embedding space**.

---

> **Question that led here**
> If each float is a vector element, can the embedding be treated as a point in vector space?

## 2. A vector as components and a point

For a simple 3-dimensional example:

```text
d = [0.8, 0.1, 0.7]
```

Each float is a **vector component**.

The same embedding can be interpreted as:

- a point (P = (0.8, 0.1, 0.7)) in a 3D space, or
- a vector from the origin (O = (0,0,0)) to that point.

```text
O(0,0,0) ─────→ P(0.8, 0.1, 0.7)
```

Real embedding models usually use hundreds or thousands of dimensions, so we cannot visualize the full space directly, but the mathematical idea is the same.

---

> **Question that led here**
> If two embedded texts become vectors, can cosine similarity tell us how similar their meanings or contexts are?

## 3. Cosine similarity

Suppose two texts are embedded as vectors:

```text
d1 = "고양이"
d2 = "귀여운 고양이"
```

Cosine similarity compares the **direction** of the two vectors.

```math
cos(θ) = (d1 · d2) / (||d1|| ||d2||)
```

Conceptually:

- similar direction → small angle → cosine close to 1
- different direction → lower cosine value

This lets a system estimate semantic similarity between texts.

The core idea is:

> Cosine similarity is about the angle between vectors, not the direct line segment between the two embedding points.

---

> **Question that led here**
> If vectors are drawn from the origin, is that the same thing as measuring the Euclidean distance between the embedding points?

## 4. L2 distance vs cosine similarity

These are different measurements.

### L2 / Euclidean distance

Measures the straight-line distance **between two points**.

```text
d1 ●────────● d2
```

### Cosine similarity

Measures the angle between two vectors drawn from the origin.

```text
        d1
       ↗
O ─────────→ d2
     θ
```

So:

```text
Cosine similarity → direction similarity
L2 distance        → point-to-point distance
```

Both can be used in vector search, depending on the embedding model and retrieval design.

---

> **Question that led here**
> Why should data that does not need to be embedded still be stored as metadata?

## 5. Embedding text vs metadata

Not every useful field should be embedded.

Consider synthetic data:

```json
{
  "title": "Sample Beach",
  "description": "A coastal place known for its beautiful night view.",
  "region": "Busan",
  "category": "Beach",
  "source": "PublicAPI"
}
```

A reasonable split is:

```text
Embedding text:
- title
- description

Metadata:
- region
- category
- source
```

### Embedding text

Used when the **semantic meaning** of the content matters.

Example question:

```text
"Where can I see a beautiful night view?"
```

### Metadata

Used to identify, filter, classify, sort, display, or trace data.

Examples:

```text
region = Busan
category = Beach
source = PublicAPI
```

A useful mental model is:

> Embedding answers "What does this mean?"  
> Metadata answers "What kind of record is this, where does it belong, and how should I constrain it?"

---

> **Question that led here**
> Is metadata itself an index, or is it structured information that can be indexed?

## 6. Metadata is not the same thing as an index

Metadata is structured information attached to a record.

An **index** is a data structure that makes lookup faster.

For example, in a relational database:

```sql
CREATE INDEX idx_region ON documents(region);
```

Here:

```text
region
→ metadata / structured field

idx_region
→ index used to find region values efficiently
```

So metadata may be indexed, but metadata itself is not automatically an index.

---

> **Question that led here**
> Does RAG data cleaning mean the same kind of normalization, indexing, and duplicate removal used in traditional database design?

## 7. RAG data preparation vs traditional database preparation

Traditional database preparation may include:

- normalization
- duplicate removal
- data type cleanup
- PK/FK design
- NULL handling
- indexes
- standardized codes

RAG adds another layer:

- choose which fields should become embedding text
- combine fields into a retrieval-friendly representation
- remove irrelevant HTML or noise
- split long text into chunks
- separate metadata from semantic text
- discard fields that serve no retrieval or application purpose

The distinction is:

> Database preparation optimizes data for correct storage and structured querying.  
> RAG preparation also optimizes data for natural-language retrieval.

### Important nuance: controlled duplication can be useful

A field may exist as metadata and also appear inside embedding text.

For example:

```text
metadata:
region = Busan

embedding text:
"Located in Busan, this coastal attraction is known for its night view."
```

From a normalization perspective this can look redundant, but retrieval systems sometimes intentionally denormalize a search representation to improve semantic retrieval.

---

> **Question that led here**
> Can structured fields such as a region be retrieved quickly with an inverted-index-based search engine?

## 8. Lexical search

Lexical search retrieves documents based mainly on words or tokens that occur in the query and documents.

Example document:

```text
"Sample Beach is a famous night-view destination in Busan."
```

Query:

```text
"Busan night view"
```

A lexical search engine can match tokens such as:

```text
Busan
night
view
```

Common relevance algorithms include:

- BM25
- TF-IDF

A common supporting data structure is an **inverted index**.

Conceptually:

```text
"Busan"      → [doc1, doc7, doc20]
"night"      → [doc1, doc4, doc20]
"beach"      → [doc1, doc3]
```

Instead of scanning every document, the engine can quickly find which documents contain a token.

> ❓ **Q. If lexical search uses an inverted index and its term dictionary behaves like a key-value map, is lexical search effectively O(1) and therefore always faster than semantic search?**
>
> **A. Not end-to-end.** Looking up a term in a dictionary can be close to O(1) with a hash-based structure, or O(log V) with a tree/FST-like structure, where V is the vocabulary size. But lexical search still has to read the term's posting list, combine posting lists for multiple query terms, compute relevance scores such as BM25, and often maintain or sort the top-K results.
>
> So the actual cost depends on factors such as:
>
> - the number of query terms,
> - the document frequency of each term,
> - posting-list intersections/unions,
> - scoring work,
> - and top-K ranking.
>
> Lexical search is often faster and cheaper than vector search for exact keyword or identifier retrieval, but it is not generally correct to describe the whole lexical-search operation as O(1).
>
> Vector search is also not necessarily a full O(N × D) scan in production. ANN indexes such as HNSW avoid comparing against every vector and trade a small amount of recall for much lower latency.

---

> **Question that led here**
> Elasticsearch and lexical search sound similar. What is the actual difference between them?

## 9. Elasticsearch vs lexical search

These concepts are at different levels.

```text
Elasticsearch
= a search engine / software system

Lexical Search
= a retrieval method
```

Elasticsearch is well known for lexical/full-text search using inverted indexes and scoring algorithms such as BM25.

Modern Elasticsearch can also support vector search.

So a simplified view is:

```text
Elasticsearch
├─ lexical search
│  └─ inverted index + BM25
└─ vector search
   └─ dense vectors + nearest-neighbor retrieval
```

Therefore:

> Elasticsearch is a system that can implement lexical search; lexical search itself is not a competing product.

---

## 10. Structured filtering + vector similarity

Structured conditions should usually be handled as structured conditions.

Suppose a user asks:

```text
"Recommend a beach in Busan with a beautiful night view."
```

The request contains different kinds of information:

```text
region = Busan
category = Beach
→ structured filters

"beautiful night view"
→ semantic condition
```

A retrieval pipeline can therefore combine:

```text
metadata filtering
        +
vector similarity search
```

For example:

```text
1. Filter documents where region = Busan
2. Filter documents where category = Beach
3. Rank the remaining documents by semantic similarity to "beautiful night view"
```

This is an important foundation for hybrid retrieval.

---

## 11. Production-oriented field classification

A more useful production-oriented classification is:

```text
1. Lexical / Filter
   → exact keyword, ID, region, category, code, or structured condition

2. Embedding
   → semantic or contextual retrieval

3. Payload
   → information returned to the application or shown in the final answer

4. Drop
   → data with no retrieval, filtering, display, or operational value
```

A single field may have multiple roles.

For example:

```text
phone_number
→ Lexical / Filter
→ Payload

description
→ Embedding
→ Payload

region
→ Lexical / Filter
→ may also appear in Embedding text when semantic context benefits from it
```

This is more precise than a simple "Embedding / Metadata / Drop" split because "metadata" often mixes together several different responsibilities.

A useful pipeline is:

```text
Raw source schema
        ↓
Canonical field normalization
        ↓
Retrieval-role classification
        ↓
Lexical / Filter | Embedding | Payload | Drop
```

Important:

> "Do not embed this field" does not automatically mean "store it as metadata."

Some fields belong in Lexical / Filter, some belong only in Payload, and some should simply be dropped.

---

## Lesson 01 summary

```text
Text
  ↓
Embedding Model
  ↓
High-dimensional Vector
  ↓
Compare vectors
  ├─ Cosine → direction / angle
  └─ L2     → point-to-point distance
```

For retrieval-oriented data modeling:

```text
exact identifiers / categories / regions → Lexical / Filter
semantic or contextual content          → Embedding
answer/display data                      → Payload
irrelevant data                          → Drop
```

A field may belong to more than one role.

For search:

```text
Lexical Search
→ token/term matching
→ inverted index / BM25

Vector Search
→ semantic similarity
→ embedding vectors

Structured Filter
→ exact constraints such as region/category

Hybrid Retrieval
→ combines multiple retrieval signals
```

The key lesson is that vector search does not replace traditional database or search-engine techniques. A robust retrieval system uses each technique for the kind of information it handles best.


---

## Glossary

### Embedding
A numeric vector representation of text or other data, produced by an embedding model so that semantic relationships can be represented geometrically.

### Vector Component
One numeric element of a vector.

### Embedding Space
The high-dimensional space in which embeddings are represented as points/vectors.

### Cosine Similarity
A similarity metric based on the angle between two vectors.

### L2 Distance
Euclidean straight-line distance between two points in vector space.

### Metadata
Structured information used to identify, filter, classify, sort, display, or trace a record.

### Index
A data structure designed to speed up lookup. Metadata fields may have indexes, but metadata and indexes are different concepts.

### Lexical Search
Search based primarily on words or tokens that occur in the query and documents.

### Inverted Index
A mapping from terms to the documents that contain them.

### BM25
A widely used lexical-search ranking function.

### Vector Search
Retrieval based on similarity or distance between embedding vectors.

### Structured Filter
An exact or range-based condition on structured fields such as region, category, date, or price.

### Hybrid Retrieval
Retrieval that combines multiple signals such as lexical search, vector similarity, and structured metadata filters.

### Lexical / Filter
A retrieval role for exact keywords, identifiers, regions, categories, codes, and other structured constraints.

### Payload
Data carried with a retrieved record for display, answer generation, tracing, or downstream application use.

### Canonicalization
The process of mapping source-specific field names and values into a consistent canonical representation before retrieval-role classification.

### Posting List
The list of documents or positions associated with a term in an inverted index.

### Document Frequency
The number of documents that contain a given term. It strongly affects how much work a lexical search may need to do for that term.
