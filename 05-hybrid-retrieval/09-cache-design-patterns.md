# Lesson 05-9 — RAG Cache Design Patterns: Cache-Aside, Strategy & Observer

> Previous: [Lesson 05-8 — Context Engineering & Retrieval Caching](./08-context-engineering-and-cache.md)

## 20. 무엇을 캐싱하고, 어떻게 추적할 것인가?

> ❓ **Q. Retrieval Result를 캐싱한다면 무엇을 저장할지, 어떤 캐시가 효과적이었는지를 어떤 디자인 패턴으로 관리할까?**
>
> **A. Cache-Aside + Strategy(Policy) + Observer를 조합할 수 있다.** 사용 가능 여부를 검증하는 Verifier와 사용 여부를 결정하는 Stage Gate를 별도로 둔다.

### 20.1 책임 분리

| 책임 | 접근 | 예시 |
|---|---|---|
| 조회 및 미스 처리 | Cache-Aside | Hit면 재사용, Miss면 Retrieval 후 저장 |
| 캐시 대상 선택 | Admission Policy / Strategy | 근거·출처·범위가 유효할 때만 저장 |
| 저장 공간 관리 | Eviction Policy / Strategy | TTL, LRU, LFU, 품질 기반 |
| 상태 추적 | Observer / Event | Hit, Miss, Validation Failure, Verified Success |
| 원본 변경 대응 | Invalidation Policy + Events | 버전 변경, 원본 삭제 시 무효화 |
| 정합성 검증 | Verifier | 권한·시간·문서 버전 검사 |
| 제어 흐름 | Stage Gate | Reuse / Recompute / Abstain |

**Strategy는 정책 결정을 내리고, Observer는 실행 결과를 관찰·기록한다.** Observer의 기록은 후속 Policy를 개선하는 입력이 될 수 있지만, 관측 자체가 정확성을 증명하지는 않는다.

### 20.2 캐시 레벨 분리

| Cache Type | 캐싱 대상 | 절약 가능 단계 |
|---|---|---|
| Retrieval Cache | 정규화된 Query → Top-K 문서 ID·순위 | Retrieval, Fusion |
| Context Cache | Query와 조건 → 검증된 Context | Retrieval, Context Assembly |
| Answer Cache | 동등 Query·조건 → 검증된 Answer | Retrieval, Generation |

문서 ID 캐시를 재사용할 때도 실제 원문의 존재·버전·접근권한을 확인해야 한다. Answer Cache는 답변의 시점·권한·질의 조건이 동일해야 하며 특히 위험도가 높다.

### 20.3 Cache-Aside와 검증 흐름

```text
Query
  ↓
Cache Key / Scope / Policy
  ↓
Cache-Aside Coordinator
  ├→ HIT → CacheVerifier → Valid? → Reuse
  │                         └→ Invalid → Recompute
  └→ MISS → RAG Pipeline → Verify → Admission Policy → Cache Store
                                              ↓
                                  Cache Events / Observer
```

CacheStore와 RetrievalPipeline은 교체 가능한 Connector 추상화로 구성할 수 있다. 구체 저장소 기술(Redis 등)에 상위 정책이 직접 결합하지 않도록 한다.

## 20.4 Admission / Eviction / Invalidation

- **Admission:** 어떤 데이터가 캐시에 들어갈 수 있는가? 예: 근거·출처·권한 범위·버전이 검증된 결과.
- **Eviction:** 공간이 부족할 때 어떤 항목을 제거할 것인가? 예: LRU/LFU 또는 보존 가치 점수.
- **Invalidation:** 어떤 사건이 발생하면 기존 캐시를 더는 사용하지 않을 것인가? 예: 원문 변경·권한 변경·TTL 만료.

**Validity는 재사용의 필수 조건이고, Popularity는 유효한 항목 간 보존 우선순위 신호**로 사용하는 것이 안전하다.

조회 수가 많다고 정답률이 높은 것은 아니다. Verifier가 실제 성공을 판정한 경우에만 `verified_success_count`를 증가시킨다.

```json
{
  "cache_key": "query:example",
  "cache_type": "retrieval",
  "hit_count": 125,
  "verified_success_count": 117,
  "validation_failure_count": 8,
  "source_version": "v12"
}
```

숫자는 예시다. **조회 성공(Cache Hit)과 정답 검증 성공(Verified Success)을 혼동하지 않는다.**

## 👨‍💻 Implementation — Protocol 기반 Strategy × Observer

```python
from dataclasses import dataclass
from typing import Protocol

@dataclass(frozen=True)
class CacheEvent:
    cache_key: str
    event_type: str
    verified: bool | None = None

class CachePolicy(Protocol):
    def should_cache(self, result: object) -> bool: ...

class CacheObserver(Protocol):
    def on_event(self, event: CacheEvent) -> None: ...

class CacheMetricsObserver:
    def on_event(self, event: CacheEvent) -> None:
        # 실제 구현에서는 Counter/Trace 시스템에 전달
        print(event)
```

이는 타입 계약을 보여주는 스케치이며 실행 가능한 전체 Cache-Aside 시스템이 아니다. `CachePolicy`와 `CacheObserver`는 각기 다른 책임을 가진다.

### 🔗 Connecting the Dots — Observer × CQRS × DIP

```text
Cache Events (Writes)
       ↓
Counters / Verification Metrics
       ↓
Aggregated Read Model
       ↓
Admission / Retention Policy
       ↓
Cache Decision
```

이 흐름은 **관찰과 의사결정 데이터를 분리한다는 측면에서 CQRS와 닮았지만**, 반드시 CQRS 전체 구조나 별도 이벤트 저장소를 구축해야 한다는 뜻은 아니다.

- **DIP:** Cache coordinator는 구체 Redis 구현이 아니라 CacheStore 인터페이스에 의존.
- **Observer:** 이벤트를 수신해 Metrics/Trace를 축적.
- **Strategy:** 측정된 지표를 정책의 입력으로 사용.
- **Process Verification:** 캐시가 재사용 가능한지 Verifier가 판단.
- **Stage Gate:** 재사용·재계산·답변 보류를 결정.

처음에는 **Cache-Aside + 소규모 Policy + Metrics**만으로 시작해, 실제 필요가 확인되면 Observer/Event-driven 구조를 확장할 수 있다.

## 20.5 Observer Pattern과 Logger의 책임 분리

> ❓ **Q. Observer Pattern 외에도 Logger 관련 패턴이 있고, Logging Observer처럼 두 방식을 결합할 수 있지 않나?**
>
> **A. 가능하다.** Observer는 이벤트를 구독자에게 전달하는 GoF 패턴이고, Logger는 기록·출력 기능이다. **Logger Pattern**이라는 표현은 여러 로깅 설계를 가리킬 수 있지만 Observer처럼 하나의 독립된 GoF 패턴 이름은 아니다.

| 컴포넌트 | 단일 책임 | 변경 이유의 예시 |
|---|---|---|
| CacheEventPublisher | 이벤트 구독자에게 전달 | 구독·전파 방식의 변경 |
| LoggingObserver | CacheEvent를 로그 기록 요청으로 변환 | 어떤 이벤트 필드를 로그로 남길지 변경 |
| Logger / Handler / Formatter | 로그를 출력·전달·형식화 | 출력 대상·포맷·로그 레벨 변경 |
| MetricsObserver | 이벤트별 Counter/Histogram 관리 | 측정 지표나 집계 방식 변경 |

```text
CacheEventPublisher
        ↓  CacheObserver.on_event
        ├── LoggingObserver ──→ Logger ──→ Handler / Formatter
        └── MetricsObserver ──→ Metrics Backend
```

Observer는 Logger를 반드시 사용해야 하는 것이 아니고, Logger는 Observer 없이 직접 사용할 수도 있다. **LoggingObserver는 이벤트와 Logger 사이를 연결하는 Adapter 유사 역할**을 한다.

### 👨‍💻 Python — Protocol 기반 LoggingObserver / MetricsObserver

```python
import logging
from typing import Protocol

# CacheEvent는 위 Implementation 절의 dataclass를 재사용한다.
class CacheObserver(Protocol):
    def on_event(self, event: CacheEvent) -> None: ...

class LoggingObserver:
    def __init__(self, logger: logging.Logger):
        self.logger = logger

    def on_event(self, event: CacheEvent) -> None:
        self.logger.info(
            "cache_event",
            extra={
                "cache_key": event.cache_key,
                "event_type": event.event_type,
                "verified": event.verified,
            },
        )

class MetricsObserver:
    def __init__(self) -> None:
        self.counts: dict[str, int] = {}

    def on_event(self, event: CacheEvent) -> None:
        self.counts[event.event_type] = (
            self.counts.get(event.event_type, 0) + 1
        )

class CacheEventPublisher:
    def __init__(self, observers: list[CacheObserver]):
        self.observers = observers

    def publish(self, event: CacheEvent) -> None:
        for observer in self.observers:
            observer.on_event(event)
```

이 예시는 **동기 이벤트 전달**이다. 운영에서는 Observer 예외가 원래 Cache 동작에 미치는 영향, 비동기화·전달 보장, 민감한 Cache Key의 로깅 여부를 별도로 설계해야 한다. Python `logging`의 여러 출력 대상만 필요한 경우에는 별도 Observer 계층 대신 **Handler**만 사용하는 편이 단순하다.

## 20.6 SRP × DIP × OCP — Protocol은 누구의 계약인가?

> ❓ **Q. CacheObserver Protocol이 LoggingObserver와 MetricsObserver를 추상화하고, 인터페이스도 다른 인터페이스에 의존할 수 있으니 DIP는 단순히 구현체가 인터페이스를 상속하는 의미가 아니겠네?**
>
> **A. 맞다.** 단, LoggingObserver와 MetricsObserver는 이 예시에서 *인터페이스가 아니라 구체 구현체*다. CacheObserver는 두 구현체가 충족하는 **공통 구조적 계약**을 정의한다.

```text
CacheEventPublisher (high-level policy)
               ↓ depends on
       CacheObserver (Protocol)
          ↑          ↑
   conforms to    conforms to
          │          │
  LoggingObserver  MetricsObserver
```

도표의 아래쪽 적합성 관계는 **개념적 타입 호환성**이다. Python `Protocol`은 Structural Typing을 사용하므로 구현체가 Protocol을 명시적으로 상속해야 하는 것은 아니다.

- **SRP (Single Responsibility Principle):** 하나의 모듈은 하나의 변경 이유를 갖도록 한다. Logger와 LoggingObserver의 책임을 개념적으로 구분하되, 클래스 수를 무조건 늘리지는 않는다.
- **DIP (Dependency Inversion Principle):** 상위 정책은 하위 구현 세부사항에 직접 의존하지 않고 **둘 다 추상화에 의존**하도록 설계한다. 구현체의 인터페이스 상속 여부만을 뜻하지 않는다.
- **OCP (Open/Closed Principle):** 공통 계약으로 새로운 Observer를 주입하여 Publisher의 핵심 동작을 변경하지 않고 확장할 수 있다.
- **DI (Dependency Injection):** 필요 객체를 외부에서 전달하는 구현 기법. DIP와 서로 다른 개념이다.

인터페이스가 다른 인터페이스를 확장하거나 메서드 시그니처에서 다른 인터페이스를 참조하는 것도 가능하다. **추상화 간 의존은 허용되며, 구체 구현에 대한 불필요한 결합을 줄이는 것이 핵심**이다.

예를 들어 LoggingObserver가 이벤트 직렬화와 출력에 관한 두 계약에 의존하도록 구성할 수도 있다.

```python
class EventSerializer(Protocol):
    def serialize(self, event: CacheEvent) -> str: ...

class EventLogger(Protocol):
    def write(self, message: str) -> None: ...

class LoggingObserver:
    def __init__(
        self, serializer: EventSerializer, logger: EventLogger
    ) -> None:
        self.serializer = serializer
        self.logger = logger

    def on_event(self, event: CacheEvent) -> None:
        self.logger.write(self.serializer.serialize(event))
```

이는 교체 가능성을 보여주는 별도 구현 예시다. 작은 시스템이라면 표준 `logging.Logger`를 직접 주입하는 편이 더 단순할 수 있다. **추상화는 변경 경계가 실제로 필요할 때 도입한다.**

---

## Lesson 05-9 요약

> **무엇을 캐싱할지는 Admission Policy가, 무엇을 유지할지는 Eviction Policy가, 언제 무효화할지는 Invalidation Policy가, 무엇이 일어났는지는 Observer가 담당한다.**

그리고 검증되지 않은 Cache Hit은 올바른 답변의 증거가 아니다.

---

## Glossary

- **Cache-Aside:** 애플리케이션이 캐시 조회·Miss 처리·저장을 관리하는 패턴.
- **Observer:** 관찰 대상의 이벤트를 수신하는 독립 컴포넌트들의 패턴.
- **Strategy:** 동일한 계약 아래 교체 가능한 정책·알고리즘을 구현하는 패턴.
- **Admission Policy:** 캐시에 새 항목을 저장할지 결정하는 규칙.
- **Eviction Policy:** 저장 공간 확보를 위해 제거할 항목을 결정하는 규칙.
- **Invalidation Policy:** 캐시 항목을 더는 유효하지 않다고 판정하는 규칙.
- **LoggingObserver:** 도메인 이벤트를 로그 기록으로 변환하는 Observer 구현체.
- **SRP / DIP / OCP:** 변경 이유의 단일화 / 추상화 중심 의존 / 기존 코드 수정 없이 확장하는 설계 원칙.
- **Structural Typing:** 상속 선언이 아니라 필요한 메서드 구조에 따라 타입 적합성을 판단하는 방식.

## References

### Primary Sources
- Gamma, Helm, Johnson & Vlissides (1994). *Design Patterns: Elements of Reusable Object-Oriented Software* — Observer와 Strategy Pattern.
- Nishtala et al. (2013). [Scaling Memcache at Facebook](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala) — 실서비스 캐시 운영·정합성의 대표적 사례.

### Official Documentation
- [Microsoft Azure Architecture Center — Cache-Aside Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside).
- [Redis — Key Eviction](https://redis.io/docs/latest/develop/reference/eviction/).
- [Python — typing.Protocol](https://docs.python.org/3/library/typing.html#typing.Protocol).
- [Python — logging](https://docs.python.org/3/library/logging.html) — Logger / Handler / Formatter 구성.

### Related Public Study
- [Lesson 05-8 — Context Engineering & Retrieval Caching](./08-context-engineering-and-cache.md).

> **구분:** RAG의 Verifier / Stage Gate와 Observer / Policy를 통합한 구조는 학습용 설계 응용이며 각 참고문헌의 원래 아키텍처라고 단정하지 않는다.
