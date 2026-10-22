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

## 11. Three-way field classification

A practical first-pass classification for source fields is:

```text
1. Embedding text
   → semantic retrieval needs the content

2. Metadata / structured field
   → filtering, classification, display, sorting, or tracing needs it

3. Discard
   → neither retrieval nor the application needs it
```

Important:

> "Do not embed this field" does not automatically mean "store it as metadata."

Some fields can simply be discarded.

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
semantic content → embedding text
structured attributes → metadata
irrelevant attributes → discard
```

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
