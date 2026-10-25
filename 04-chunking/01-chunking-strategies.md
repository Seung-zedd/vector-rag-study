# Lesson 04 — Chunking

## 1. Why Chunking Exists

Chunking is the process of splitting a document into retrieval-friendly units.

A long document often contains multiple topics:

```text
history
operating hours
parking
nearby restaurants
transportation
```

If the entire document is embedded as one vector, the meaning of a specific detail can be diluted inside the overall representation.

```text
Long Document
   ↓
One Embedding
   ↓
Multiple meanings compressed together
```

Chunking instead creates smaller searchable units.

```text
Document
   ↓
Chunk 1
Chunk 2
Chunk 3
...
   ↓
Embedding per chunk
```

The core trade-off is:

```text
Context Preservation
        vs
Retrieval Precision
```

Chunks that are too small lose context.

Chunks that are too large mix unrelated meanings.

---

## 2. Chunking vs Retrieval-Role Classification

Chunking and retrieval-role classification solve different problems.

```text
Chunking
→ How large should each retrieval unit be?

Lexical / Filter / Embedding / Payload
→ What retrieval role should each field play?
```

If a JSON item is already compact and semantically coherent, chunking may be unnecessary.

Example:

```json
{
  "title": "A Place",
  "address": "Seoul ...",
  "description": "Known for a quiet walking path and night view.",
  "phone": "02-1234-5678"
}
```

A reasonable classification could be:

```text
title
→ Lexical / Filter + Payload

address
→ Lexical / Filter + Payload

description
→ Embedding + Payload

phone
→ Lexical / Filter + Payload
```

In this case, the JSON object is already an appropriate retrieval unit.

---

## 3. Field Decomposition Before Chunking

A badly shaped schema should not be fixed by blindly slicing strings.

Suppose one field contains several semantic roles:

```text
address:
"Seoul ... / beautiful night-view attraction / contact 02-1234-5678"
```

The better first step is structural decomposition.

```text
address
→ "Seoul ..."

description
→ "beautiful night-view attraction"

contact_number
→ "02-1234-5678"
```

Then assign retrieval roles.

```text
address
→ Lexical / Filter + Payload

description
→ Embedding + Payload

contact_number
→ Lexical / Filter + Payload
```

Only when a field remains long or semantically mixed after decomposition should chunking become the next tool.

A useful order is:

```text
Raw Data
   ↓
Canonicalization
   ↓
Field Decomposition
   ↓
Chunking needed?
   ↓
Retrieval-role classification
```

---

## 4. Fixed-size Chunking

Fixed-size chunking splits data by length.

Examples:

```text
500 characters
500 characters
500 characters
...
```

or:

```text
512 tokens
512 tokens
512 tokens
...
```

Advantages:

- simple
- fast
- deterministic
- easy to implement consistently

The main weakness is boundary breakage.

Example:

```text
"Operating hours are 09:00–18:00, and
the attraction is closed every Monday."
```

A hard boundary could produce:

```text
Chunk 1
"Operating hours are 09:00–18:00, and"

Chunk 2
"the attraction is closed every Monday."
```

The original meaning is now split across chunks.

---

## 5. Structure-aware Chunking

Structure-aware chunking uses document structure instead of arbitrary length.

Possible boundaries include:

```text
heading
paragraph
list
section
JSON object
JSON field
HTML node
```

Example:

```text
[Operating Information]
Hours: 09:00–18:00
Closed: Monday

[Transportation]
Subway ...
Bus ...
```

A structure-aware splitter can produce:

```text
Chunk 1
→ Operating Information

Chunk 2
→ Transportation
```

This preserves logical organization better than fixed-size splitting.

---

## 6. Semantic Chunking

Semantic chunking looks for changes in meaning rather than only structural or length boundaries.

```text
historical background
historical background
historical background
──────────── semantic boundary
operating hours
admission fee
closed days
──────────── semantic boundary
transportation
parking
```

The three major approaches can be summarized as:

```text
Fixed-size
→ length boundary

Structure-aware
→ document-structure boundary

Semantic
→ meaning boundary
```

A practical default is often:

> Start with structure-aware chunking, then add semantic splitting only when the source structure is not sufficient.

---

## 🔗 Connecting the Dots — Paging vs Segmentation

Chunking has a useful analogy with operating-system memory management.

```text
Fixed-size Chunking
↔ Paging

Structure-aware / Semantic Chunking
↔ Segmentation
```

Paging divides memory into fixed-size blocks.

Segmentation preserves logical units such as code, data, or stack regions.

The analogy is not exact, but the shared idea is useful:

```text
Paging
→ regular physical-sized units

Segmentation
→ logical units

Chunking
→ retrieval units
```

In memory management, the OS manages page or segment tables, while the MMU uses those mappings for address translation.

A loose retrieval analogy is:

```text
Virtual Address
→ MMU
→ Page Table
→ Physical Page
```

and:

```text
User Query
→ QueryRouter
→ Retrieval Index / Metadata
→ Chunk
```

The shared abstraction is:

> Split a large space into addressable units, then use an index or mapping structure to reach the appropriate unit efficiently.

---

## 7. Chunk Overlap

Chunk overlap repeats part of one chunk inside the next chunk.

Example:

```text
Chunk 1
[1 ............... 500]

Chunk 2
          [401 ............... 900]

Chunk 3
                    [801 ............... 1300]
```

The purpose is to reduce information loss at boundaries.

Without overlap:

```text
Chunk 1
"This attraction has beautiful night lighting"

Chunk 2
"and is popular for evening walks."
```

With overlap, both chunks retain enough shared context to preserve the relationship.

The trade-off is:

```text
More overlap
→ context preservation ↑
→ duplicate storage ↑
→ embedding cost ↑
→ duplicate retrieval ↑
```

So overlap is a boundary-safety mechanism, not something that should be maximized.

Structure-aware chunking can often reduce the need for large overlaps because meaningful boundaries are already preserved.

---

## 🔗 Connecting the Dots — Chunk Overlap × TCP Boundary Robustness

Chunk overlap can also be connected conceptually to reliable network transport.

TCP treats application data as a byte stream and splits it into segments for transmission.

Modern TCP combines mechanisms such as:

- cumulative acknowledgments
- selective acknowledgment support
- out-of-order receive buffering
- retransmission

The exact mechanisms differ from chunk overlap, but they share one broad engineering principle:

> When a continuous stream is divided into smaller units, extra structure or redundancy is often added to reduce the cost of boundary loss or missing information.

For chunking:

```text
overlap
→ protects semantic continuity across chunk boundaries
```

For TCP:

```text
sequence numbers + ACK/SACK + buffering
→ protect reliable reconstruction across packet boundaries and loss
```

The domains are different, but both trade additional overhead for robustness.

---

## Lesson 04 Summary

```text
Canonicalize
   ↓
Decompose mixed fields
   ↓
Decide whether chunking is necessary
   ↓
Choose chunking strategy
   ├─ Fixed-size
   ├─ Structure-aware
   └─ Semantic
   ↓
Add overlap only when boundary protection is needed
```

The key lesson is:

> Chunking is not a mandatory preprocessing step for every dataset. It is a retrieval-unit design decision that should be applied only when the existing structure is too large or semantically mixed.

---

## Glossary

### Chunk
A retrieval-sized unit created from a larger document or field.

### Chunking
The process of splitting a document into smaller retrieval units.

### Fixed-size Chunking
A chunking strategy that splits data using a fixed character or token length.

### Structure-aware Chunking
A chunking strategy that respects source structure such as sections, paragraphs, JSON objects, or HTML nodes.

### Semantic Chunking
A chunking strategy that splits when the meaning or topic changes.

### Chunk Overlap
The intentional repetition of content between adjacent chunks to preserve context across boundaries.

### Canonicalization
The process of transforming heterogeneous source fields into a consistent normalized schema.

### Field Decomposition
The process of separating one mixed field into multiple fields with distinct semantic roles.

### Retrieval Unit
The smallest independently searchable unit used by a retrieval system.

### Context Preservation
The degree to which important relationships and surrounding meaning survive preprocessing and chunking.

### Retrieval Precision
The degree to which a retrieved unit focuses on the specific information relevant to the query.
