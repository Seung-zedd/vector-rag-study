# Vector DB & RAG Study

Vector Database와 Retrieval-Augmented Generation(RAG)을 기초부터 이해하기 위한 재사용 가능한 학습 레포지토리입니다.

## 학습 원칙

각 레슨은 개념을 충분히 대화하고 이해한 뒤에만 문서화합니다.

목표는 API를 외우는 것이 아니라 다음을 할 수 있게 되는 것입니다.

- 개념을 설명한다.
- 실제 시스템을 관찰한다.
- 결과를 검증한다.
- 다른 프로젝트와 연결해서 재사용한다.

## 목차

- [Lesson 01 — 임베딩, 벡터 유사도, 메타데이터, Lexical Search](./01-embedding/01-vector-embedding-and-search-basics.md)
- [Lesson 02 — ANN, Top-K, HNSW](./02-vector-search/01-ann-topk-hnsw.md)
- [Lesson 03 — RAG Retrieval Pipeline](./03-rag-pipeline/01-retrieval-context-generation.md)
- [Lesson 04 — Chunking](./04-chunking/01-chunking-strategies.md)
- [Lesson 05 — Hybrid Retrieval](./05-hybrid-retrieval/01-hybrid-retrieval-basics.md)

## 범위

이 레포에는 일반화된 학습 내용과 공개 가능한 synthetic example만 포함합니다.

의도적으로 제외하는 내용:

- 회사 내부 데이터셋
- 비공개 API 응답
- 사내 전용 시스템 설계
- credential / secret
- 프로젝트별 기밀 정보

## 공개 및 저작권 판단 원칙

이 저장소는 **개인 학습과 독자적으로 도출한 기술 아이디어를 공개 가능한 형태로 정리**하는 공간입니다.

### 공개 가능

- 공개 논문·문서·표준을 바탕으로 학습한 일반 기술 개념
- 개인적으로 고안한 설계 아이디어, 알고리즘 비교, 디자인 패턴 응용
- 서로 다른 분야를 연결한 `Connecting the Dots` 및 Q&A
- 독립적으로 작성한 synthetic example과 일반화된 코드 예시

### 공개 금지 또는 사전 확인

- 회사의 실제 데이터셋, 비공개 API 응답·엔드포인트, credential, secret
- 회사의 내부 운영 환경, 비공개 소스코드·설정·문서
- 회사 소유의 비공개 설계·구현 세부사항 및 식별 가능한 고객·사업 정보
- **개인적으로 생각해 낸 아이디어라도 업무상 개발한 결과물에 해당할 수 있는 경우:** 고용계약, 사내 규정 및 지식재산권 귀속 조건을 확인하기 전에는 공개하지 않음

### 문서화 전 점검

1. 내용이 공개 지식 또는 독립적으로 설명 가능한 개인 학습·설계 아이디어인가?
2. 실제 회사 데이터·식별자·인프라·내부 구현 세부사항이 섞이지 않았는가?
3. 업무상 저작물이나 회사의 비공개 자산에 해당할 여지는 없는가?
4. 불확실하다면 구체 내용을 제거하고 일반화하거나, 공개 가능 여부를 먼저 확인했는가?

> **핵심:** 개인의 학습 성과와 독창적 사고는 기록하되, 회사 기밀 및 제3자의 권리는 침해하지 않는다. 독창성만으로 업무상 결과물의 공개 권한이 자동 보장되지는 않는다.

---

## License

이 레포는 [MIT License](./LICENSE)를 따릅니다.
