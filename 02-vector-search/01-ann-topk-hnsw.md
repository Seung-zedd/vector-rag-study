# Lesson 02 — ANN, Top-K, and HNSW

## 1. Exact Search vs Approximate Nearest Neighbor (ANN)

If a vector database contains only a small number of vectors, the simplest search strategy is to compare the query vector against every stored vector.

```text
q ↔ d1
q ↔ d2
q ↔ d3
...
q ↔ dN
```

This is an **exact search**.

If each vector has dimensionality `D` and there are `N` vectors, a brute-force scan is roughly:

```text
O(N × D)
```

As `N` grows, this becomes expensive.

Approximate Nearest Neighbor (ANN) search avoids checking every vector. Instead, it searches a much smaller candidate set and accepts a small possibility of missing the mathematically exact nearest neighbor.

```text
Exact Search
→ maximum accuracy
→ potentially expensive

ANN
→ much faster
→ may lose a small amount of recall
```

ANN does not replace cosine similarity, L2 distance, or dot product.

Those are **distance/similarity metrics**.

ANN is a **search strategy** for finding promising candidates efficiently.

---

## 2. Top-K

Top-K means returning only the K highest-ranked retrieval results.

Example:

```text
d1 = 0.93
d2 = 0.88
d3 = 0.81
d4 = 0.74
d5 = 0.69
```

If:

```text
Top-K = 3
```

the system returns:

```text
d1, d2, d3
```

The key trade-off is:

```text
K too small
→ relevant evidence may be missed

K too large
→ irrelevant evidence and noise may increase
```

So Top-K is a balance between **recall** and **noise**.

A practical system may combine Top-K with a similarity threshold.

```text
Top-K = 5
threshold = 0.75
```

Then:

```text
0.91 → keep
0.87 → keep
0.79 → keep
0.72 → drop
0.61 → drop
```

Top-K controls the maximum number of candidates, while the threshold controls the minimum acceptable relevance.

---

## 🔗 Connecting the Dots — Bayesian Inference × Adaptive Top-K

> ❓ **Q. Can Bayesian inference be used to adaptively balance recall and noise when selecting Top-K documents?**
>
> **A. Yes.** A more natural Bayesian formulation is not to predict K directly, but to estimate the posterior probability that each retrieved document is relevant.

For example:

```text
P(relevant | retrieval features)
```

Possible features include:

- cosine similarity
- BM25 score
- reranker score
- metadata match
- query-document length relation

A Bayesian logistic model could estimate:

```text
d1 → P(relevant) = 0.97
d2 → P(relevant) = 0.91
d3 → P(relevant) = 0.78
d4 → P(relevant) = 0.43
```

Then the system can select documents according to a posterior decision rule.

```text
P(relevant) > 0.7
```

This makes the effective value of K emerge dynamically from the model.

Bayesian inference adds something especially useful: **uncertainty**.

Two documents can have the same posterior mean but very different uncertainty:

```text
d3
mean relevance = 0.75
credible interval = [0.72, 0.78]

d4
mean relevance = 0.75
credible interval = [0.35, 0.96]
```

A stricter system might require a posterior lower bound to exceed a threshold.

This connects retrieval design with uncertainty-aware decision making.

A practical progression is:

```text
Fixed Top-K
   ↓
Top-K + threshold
   ↓
reranker
   ↓
query-dependent / adaptive K
   ↓
probabilistic relevance modeling
```

---

> ❓ **Q. ANN and HNSW both seem to search only promising candidates instead of scanning everything. What is the difference?**
>
> **A. ANN is the broader search strategy, while HNSW is one concrete algorithm/index structure used to implement ANN.**
>
> ANN says:
>
> ```text
> Do not compare against every vector.
> Approximate the nearest neighbors efficiently.
> ```
>
> HNSW says:
>
> ```text
> Build a hierarchical proximity graph.
> Navigate through promising neighbors.
> Refine the search as you descend through layers.
> ```
>
> So the relationship is:
>
> ```text
> ANN
> = the general problem-solving strategy
>
> HNSW
> = a graph-based implementation of ANN
> ```
>
> HNSW is one ANN approach among several possible methods.

---

## 3. HNSW

HNSW stands for **Hierarchical Navigable Small World**.

Its core idea is:

> Do not compare the query against every vector. Navigate through a graph of nearby vectors and move toward promising regions.

A simplified graph may look like:

```text
d1 ─── d7 ─── d20
       │       │
      d9 ─── d31
```

HNSW adds multiple layers.

```text
Layer 2        A -------- F

Layer 1      A -- C -- F -- H

Layer 0    A-B-C-D-E-F-G-H-I-J
```

The upper layers are sparse and allow long jumps.

Lower layers are denser and allow finer local search.

A simplified search path is:

```text
enter at upper layer
        ↓
move toward a closer node
        ↓
descend to a lower layer
        ↓
search more locally
        ↓
return nearest candidates
```

A useful analogy is road navigation:

```text
highway
→ major road
→ local road
→ destination neighborhood
```

Instead of checking every possible road, the system uses structure to reach the promising region quickly.

### HNSW is not a distance metric

```text
Cosine / L2 / Dot Product
→ measure vector closeness

HNSW
→ controls how candidate vectors are explored
```

So the conceptual separation is:

```text
Similarity metric
- cosine
- L2
- dot product

Search strategy / index
- exact scan
- ANN
- HNSW
```

---

## 🔗 Connecting the Dots — HNSW × Branch-Based Search × Routing

HNSW follows a general optimization pattern that also appears in tree search, graph search, and routing.

### Brute force vs structured navigation

```text
Brute force
→ inspect every candidate

Structured search
→ follow promising branches or neighbors
```

HNSW is not literally a tree, but it uses the same high-level idea:

> Do not inspect the entire search space. Use structure to navigate toward promising regions.

It also resembles routing.

```text
Routing
current node
→ choose next hop
→ move closer to destination
```

```text
HNSW
current vector node
→ choose a better neighbor
→ move closer to the query region
```

The objective differs:

```text
Network routing
→ latency, hop count, cost, reliability

HNSW
→ vector similarity / distance
```

But the shared abstraction is efficient navigation through a structured search space.

---

## Lesson 02 Summary

```text
Exact Search
→ compare against every vector

ANN
→ search only promising candidates

Top-K
→ return only the highest-ranked K results

Top-K + threshold
→ cap the count and reject weak matches

HNSW
→ hierarchical graph navigation for ANN
```

The important conceptual separation is:

```text
metric
→ how closeness is measured

search strategy
→ how candidate vectors are found

ranking policy
→ how many results are returned
```

---

## Glossary

### Exact Search
A search strategy that compares the query against all candidate vectors to guarantee exact nearest-neighbor results.

### ANN
Approximate Nearest Neighbor. A search strategy that reduces search cost by exploring only a subset of candidates while accepting a small possible loss in recall.

### Top-K
The maximum number of highest-ranked retrieval results returned by a search.

### Similarity Threshold
A minimum similarity or relevance score required for a result to be kept.

### Recall
The proportion of relevant results successfully retrieved from all relevant results that exist.

### Noise
Retrieved results that are irrelevant or weakly related to the query.

### HNSW
Hierarchical Navigable Small World. A graph-based ANN index that uses multiple layers to move quickly through the search space.

### Neighbor
A graph node connected to another node because it is considered nearby according to the index construction process.

### Search Layer
One hierarchical level in HNSW. Upper layers are sparse and support long-range navigation; lower layers are denser and support local refinement.

### Candidate Set
The subset of vectors considered during ANN search instead of scanning the entire dataset.

### Posterior Probability
In Bayesian inference, the updated probability of a hypothesis after observing evidence.

### Credible Interval
A Bayesian interval that describes a range of plausible parameter values under the posterior distribution.
