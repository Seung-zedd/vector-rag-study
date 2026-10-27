# Lesson 01 — 임베딩, 벡터 유사도, 메타데이터, Lexical Search

> ❓ **Q. 텍스트가 embedding vector로 변환될 때 각 숫자는 무엇을 의미하는가?**
>
> **A.** 각 숫자를 사람이 직접 "귀여움=0.8"처럼 의미 붙이는 것이 아니다. 임베딩 모델이 학습 데이터의 패턴과 관계를 통해 고차원 표현을 학습한다. 중요한 것은 개별 float의 뜻보다 벡터들이 embedding space에서 갖는 상대적 위치와 방향이다.

## 1. Embedding

```text
"고양이"
   ↓
Embedding Model
   ↓
[0.8, 0.1, 0.7, ...]
```

각 값은 **vector component**다.

3차원 예시라면:

```text
d = [0.8, 0.1, 0.7]
```

이는 점 (P=(0.8,0.1,0.7))로 볼 수도 있고 원점에서 그 점을 향하는 벡터로 볼 수도 있다.

---

## 2. Cosine Similarity

Cosine similarity는 두 벡터의 **방향**을 비교한다.

```math
cos(θ) = (d1 · d2) / (||d1|| ||d2||)
```

- 방향이 비슷함 → 각도가 작음 → cosine 값이 1에 가까움
- 방향이 다름 → cosine 값이 낮아짐

핵심:

> Cosine similarity는 embedding point 사이의 직선거리가 아니라 벡터 사이의 각도를 본다.

---

## 3. L2 Distance vs Cosine Similarity

```text
Cosine Similarity
→ 방향 / 각도 비교

L2 Distance
→ 두 점 사이의 직선거리
```

둘 다 vector search에 사용할 수 있지만 의미는 다르다.

---

## 4. Embedding Text vs Metadata

모든 필드를 임베딩할 필요는 없다.

예:

```json
{
  "title": "Sample Beach",
  "description": "야경으로 유명한 해변",
  "region": "Busan",
  "category": "Beach",
  "source": "PublicAPI"
}
```

가능한 역할:

```text
title / description
→ semantic meaning이 중요
→ Embedding

region / category / source
→ 식별, 필터, 분류, 표시
→ Metadata / structured field
```

중요한 차이:

> Embedding은 "이 내용이 무엇을 의미하는가?"에 가깝고, Metadata는 "이 레코드는 어떤 속성을 가지는가?"에 가깝다.

---

## 5. Metadata와 Index는 다르다

Metadata는 구조화된 정보이고, Index는 조회를 빠르게 하는 자료구조다.

```sql
CREATE INDEX idx_region ON documents(region);
```

```text
region
→ metadata field

idx_region
→ lookup을 빠르게 하는 index
```

---

## 6. RAG 데이터 준비와 전통 DB 전처리

전통 DB 전처리:

- normalization
- duplicate 제거
- type 정리
- PK/FK
- NULL 처리
- index
- code 표준화

RAG는 여기에 추가로:

- 어떤 필드를 embedding할지 결정
- retrieval-friendly text 구성
- HTML/noise 제거
- 긴 텍스트 chunking
- semantic text와 structured field 분리
- retrieval에 필요 없는 필드 제거

중요:

> RAG에서는 검색 성능을 위해 의도적인 denormalization이 유용할 수 있다.

---

## 7. Lexical Search

Lexical Search는 query와 document에 실제로 등장하는 **word/token**을 중심으로 검색한다.

대표 요소:

- BM25
- TF-IDF
- inverted index

```text
"Busan" → [doc1, doc7, doc20]
"night" → [doc1, doc4, doc20]
```

> ❓ **Q. Inverted Index의 term lookup이 빠르면 Lexical Search 전체도 O(1)인가?**
>
> **A. 아니다.** term dictionary lookup 자체는 매우 빠를 수 있지만, posting list 조회, 여러 term의 intersection/union, BM25 scoring, Top-K 유지 작업이 추가된다. 따라서 전체 lexical search를 O(1)이라고 보는 것은 잘못이다.

Vector search도 production에서는 항상 O(N×D) full scan을 하는 것이 아니라 HNSW 같은 ANN을 사용한다.

---

## 8. Elasticsearch vs Lexical Search

```text
Elasticsearch
= 검색 엔진 / 제품

Lexical Search
= 검색 방식
```

Elasticsearch는 inverted index + BM25 기반 lexical/full-text search로 유명하지만 vector search도 지원할 수 있다.

---

## 9. Structured Filter + Vector Similarity

예:

```text
"부산에 있는 야경 좋은 해변 추천"
```

분해:

```text
region = 부산
category = 해변
→ Filter

"야경 좋은"
→ Semantic Search
```

즉:

```text
Structured Filter
        +
Vector Similarity
```

를 결합할 수 있다.

---

## 10. 실전 Retrieval Role 분류

보다 실전적인 분류:

```text
1. Lexical / Filter
→ exact keyword, ID, 지명, category, code, structured condition

2. Embedding
→ semantic / contextual retrieval

3. Payload
→ 최종 답변이나 애플리케이션에 전달할 데이터

4. Drop
→ 검색/표시/운영 어디에도 필요 없는 데이터
```

하나의 필드가 여러 역할을 동시에 가질 수 있다.

```text
phone_number
→ Lexical / Filter
→ Payload

description
→ Embedding
→ Payload

region
→ Lexical / Filter
→ 필요하면 Embedding text에도 포함
```

추천 흐름:

```text
Raw Source Schema
        ↓
Canonicalization
        ↓
Retrieval-role Classification
        ↓
Lexical / Filter | Embedding | Payload | Drop
```

---

## Lesson 01 요약

```text
Exact identifiers / category / region
→ Lexical / Filter

Semantic / contextual content
→ Embedding

Answer/display data
→ Payload

Irrelevant data
→ Drop
```

Vector Search가 기존 검색 기술을 대체하는 것이 아니라, 각 검색 방식이 잘하는 문제를 나눠 맡는 것이 핵심이다.

---

## Glossary

### Embedding
텍스트 등의 데이터를 의미 관계가 공간적으로 드러나도록 숫자 벡터로 표현한 것.

### Vector Component
벡터를 구성하는 하나의 숫자 값.

### Embedding Space
임베딩 벡터들이 위치하는 고차원 공간.

### Cosine Similarity
두 벡터 사이의 각도를 기반으로 한 유사도.

### L2 Distance
두 점 사이의 유클리드 직선거리.

### Metadata
레코드를 식별·필터·분류·표시하기 위한 구조화 정보.

### Index
조회 속도를 높이기 위한 자료구조.

### Lexical Search
query/document에 실제 등장하는 단어·토큰을 중심으로 검색하는 방식.

### Inverted Index
term에서 해당 term을 포함하는 document로 연결하는 index.

### BM25
대표적인 lexical relevance ranking 함수.

### Vector Search
embedding vector 사이의 유사도/거리로 검색하는 방식.

### Structured Filter
region, category, date 등의 구조화 필드에 exact/range constraint를 적용하는 것.

### Payload
검색 결과와 함께 downstream application 또는 최종 답변에 전달되는 데이터.

### Canonicalization
source마다 다른 필드명/값을 일관된 표현으로 정규화하는 과정.

### Posting List
특정 term이 등장하는 문서 또는 위치 목록.

### Document Frequency
특정 term을 포함하는 문서 수.
