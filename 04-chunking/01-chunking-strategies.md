# Lesson 04 — Chunking

## 1. 왜 Chunking이 필요한가

Chunking은 긴 문서를 retrieval-friendly unit으로 나누는 과정이다.

긴 문서 하나에 여러 주제가 섞이면:

```text
역사
운영시간
주차
주변 맛집
교통
```

문서 전체를 하나의 embedding으로 만들 경우 특정 의미가 희석될 수 있다.

```text
Document
   ↓
Chunk 1
Chunk 2
Chunk 3
   ↓
Embedding per Chunk
```

핵심 trade-off:

```text
Context Preservation
        vs
Retrieval Precision
```

---

## 2. Chunking vs Retrieval-role Classification

둘은 다른 문제를 푼다.

```text
Chunking
→ 검색 단위를 얼마나 크게 나눌 것인가?

Lexical / Filter / Embedding / Payload
→ 각 field를 retrieval에서 어떤 역할로 사용할 것인가?
```

이미 JSON item이 작고 의미적으로 응집돼 있다면 chunking을 생략할 수 있다.

---

## 3. Field Decomposition이 먼저다

예:

```text
address:
"서울 ... / 야경이 아름다운 관광지 / 문의 02-1234-5678"
```

이 경우 무작정 chunking하기보다 먼저:

```text
address
description
contact_number
```

로 의미별 field decomposition을 하는 것이 낫다.

추천 순서:

```text
Raw Data
   ↓
Canonicalization
   ↓
Field Decomposition
   ↓
Chunking 필요한가?
   ↓
Retrieval-role Classification
```

---

## 4. Fixed-size Chunking

```text
500 characters
500 characters
500 characters
...
```

또는:

```text
512 tokens
512 tokens
...
```

장점:

- 단순함
- 빠름
- deterministic
- 구현 쉬움

단점:

- 의미 단위가 경계에서 잘릴 수 있음

---

## 5. Structure-aware Chunking

문서의 구조를 경계로 사용한다.

```text
heading
paragraph
list
section
JSON object
JSON field
HTML node
```

fixed-size보다 논리적 구조를 잘 보존한다.

---

## 6. Semantic Chunking

길이나 문서 구조가 아니라 **의미 변화**를 기준으로 자른다.

```text
역사 설명
역사 설명
──────── semantic boundary
운영시간
입장료
휴무일
──────── semantic boundary
교통
주차
```

비교:

```text
Fixed-size
→ 길이 기준

Structure-aware
→ 문서 구조 기준

Semantic
→ 의미 변화 기준
```

실무 기본값으로는 Structure-aware를 먼저 고려하고 필요할 때 Semantic splitting을 추가하는 것이 안정적이다.

---

## 🔗 Connecting the Dots — Paging vs Segmentation

```text
Fixed-size Chunking
↔ Paging

Structure-aware / Semantic Chunking
↔ Segmentation
```

Paging은 고정 크기 단위, Segmentation은 논리적 단위를 보존한다.

Loose analogy:

```text
Virtual Address
→ MMU
→ Page Table
→ Physical Page

User Query
→ QueryRouter
→ Retrieval Index / Metadata
→ Chunk
```

공통 추상화:

> 큰 공간을 addressable unit으로 나눈 뒤 mapping/index 구조를 통해 필요한 unit으로 이동한다.

---

## 7. Chunk Overlap

인접 chunk가 일부 token을 공유하게 하는 방식이다.

```text
Chunk 1
[1 ............... 500]

Chunk 2
          [401 ............... 900]
```

목적:

> chunk boundary에서 문맥이 끊기는 것을 완화한다.

Trade-off:

```text
Overlap 증가
→ context preservation ↑
→ duplicate storage ↑
→ embedding cost ↑
→ duplicate retrieval ↑
```

Overlap은 보험이지 클수록 좋은 값이 아니다.

---

## 🔗 Connecting the Dots — Chunk Overlap × TCP Boundary Robustness

TCP와 Chunk Overlap은 메커니즘 자체는 다르지만 공통된 engineering principle이 있다.

```text
Chunking
→ overlap으로 semantic continuity 보호

TCP
→ sequence number + ACK/SACK + receive buffering
→ reliable reconstruction 보호
```

공통점:

> 연속된 정보를 작은 단위로 나눌 때 경계 손실 비용을 줄이기 위한 추가 구조를 둔다.

---

## Lesson 04 요약

```text
Canonicalize
   ↓
Mixed Field Decomposition
   ↓
Chunking 필요 여부 판단
   ↓
전략 선택
   ├─ Fixed-size
   ├─ Structure-aware
   └─ Semantic
   ↓
필요할 때 Overlap
```

핵심:

> Chunking은 모든 데이터에 강제하는 preprocessing이 아니라 retrieval unit을 설계하는 선택이다.

---

## Glossary

### Chunk
큰 문서에서 분리한 retrieval unit.

### Chunking
문서를 더 작은 retrieval unit으로 나누는 과정.

### Fixed-size Chunking
고정 character/token 길이로 나누는 방식.

### Structure-aware Chunking
section, paragraph, JSON object 같은 source structure를 보존하는 방식.

### Semantic Chunking
topic/meaning 변화 지점에서 나누는 방식.

### Chunk Overlap
인접 chunk 사이에 일부 내용을 중복시켜 문맥을 보존하는 방식.

### Field Decomposition
서로 다른 의미가 섞인 하나의 field를 여러 semantic field로 분리하는 과정.

### Retrieval Unit
독립적으로 검색 가능한 최소 단위.

### Context Preservation
preprocessing 후에도 중요한 문맥 관계가 유지되는 정도.

### Retrieval Precision
검색 단위가 query와 관련된 정보에 얼마나 집중되어 있는지.
