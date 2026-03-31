<div align="center">

# ⚡ Redis Deep Dive

**"Redis를 캐시로 쓰는 것과, 데이터가 메모리에서 어떻게 저장되고 만료되는지 아는 것은 다르다"**

<br/>

> *"`@Cacheable` 붙이면 Redis에 저장되겠지 — 와 — 직렬화 방식이 왜 성능에 영향을 주는지, `maxmemory-policy`가 데이터를 어떻게 지우는지 아는 것의 차이를 만드는 레포"*

단일 스레드 이벤트 루프가 왜 빠른지, `listpack`이 `skiplist`로 전환되는 조건이 무엇인지, `BGSAVE`의 `fork()`가 Copy-On-Write로 일관성을 보장하는 원리, Cache Stampede를 막는 PER 알고리즘까지  
**왜 이렇게 설계됐는가** 라는 질문으로 Redis 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Redis](https://img.shields.io/badge/Redis-7.0-DC382D?style=flat-square&logo=redis&logoColor=white)](https://redis.io/docs/)
[![Spring](https://img.shields.io/badge/Spring_Data_Redis-3.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://docs.spring.io/spring-data/redis/docs/current/reference/html/)
[![Docs](https://img.shields.io/badge/Docs-37개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Redis에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`@Cacheable`을 붙이면 캐싱됩니다" | `CacheAspect` → `RedisCache` → `RedisTemplate` → `SET key value EX ttl` 전 과정, 직렬화 방식이 왜 성능에 영향을 주는지 |
| "TTL을 설정하세요" | Lazy 만료 vs Active 만료(주기적 샘플링) 차이, 만료된 키가 메모리에 잠시 남아있는 이유, `hz` 설정이 만료 속도에 미치는 영향 |
| "Redis는 빠릅니다" | 단일 스레드 이벤트 루프 + `epoll`/`kqueue` I/O 멀티플렉싱 구조, 어디서 병목이 발생하고 Redis 6.0 Threaded I/O가 무엇을 개선했는지 |
| "maxmemory를 설정하세요" | `allkeys-lru` vs `volatile-lru` vs `noeviction` 8가지 정책이 내부에서 어떻게 키를 선택하는지, OOM이 발생하는 경위 |
| "Hash는 HashMap처럼 쓰면 됩니다" | `listpack` → `hashtable` 전환 조건, 점진적 rehashing 중에도 읽기/쓰기가 가능한 이유 |
| "Redis Cluster를 쓰면 됩니다" | 해시 슬롯 16384개 분산 원리, 리샤딩 중 `ASK`/`MOVED` 리다이렉션 처리, Gossip 프로토콜로 장애를 감지하는 방식 |
| 이론 나열 | 실행 가능한 `redis-cli` 실험 + `OBJECT ENCODING` / `DEBUG OBJECT` / `MEMORY USAGE` + Docker Compose 환경 + Spring 연결 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Internals](https://img.shields.io/badge/🔹_Internals-단일_스레드_이벤트_루프-DC382D?style=for-the-badge&logo=redis&logoColor=white)](./redis-internals/01-single-thread-event-loop.md)
[![DataStructures](https://img.shields.io/badge/🔹_DataStructures-String과_SDS-DC382D?style=for-the-badge&logo=redis&logoColor=white)](./data-structures/01-string-sds.md)
[![Persistence](https://img.shields.io/badge/🔹_Persistence-RDB_스냅샷_원리-DC382D?style=for-the-badge&logo=redis&logoColor=white)](./persistence/01-rdb-snapshot.md)
[![HA](https://img.shields.io/badge/🔹_Replication-Master--Replica_복제-DC382D?style=for-the-badge&logo=redis&logoColor=white)](./replication-ha/01-master-replica-psync.md)
[![Caching](https://img.shields.io/badge/🔹_Caching-캐싱_전략_비교-DC382D?style=for-the-badge&logo=redis&logoColor=white)](./caching-patterns/01-caching-strategies.md)
[![Ops](https://img.shields.io/badge/🔹_Operations-SLOWLOG와_병목_찾기-DC382D?style=for-the-badge&logo=redis&logoColor=white)](./performance-operations/01-slowlog-bottleneck.md)
[![Spring](https://img.shields.io/badge/🔹_Spring-RedisTemplate_직렬화-DC382D?style=for-the-badge&logo=redis&logoColor=white)](./spring-redis/01-redis-template-serialization.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Redis 내부 아키텍처

> **핵심 질문:** Redis는 왜 빠른가? 단일 스레드 모델에서 병목은 어디서 발생하고, 메모리는 어떻게 관리되며, 키 만료는 정확히 언제 일어나는가?

<details>
<summary><b>단일 스레드 이벤트 루프부터 Redis 6.0 Threaded I/O까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 단일 스레드 이벤트 루프 — 왜 빠른가](./redis-internals/01-single-thread-event-loop.md) | `epoll`/`kqueue` I/O 멀티플렉싱으로 수만 개의 동시 연결을 단일 스레드로 처리하는 원리, Context Switch 없이 빠른 이유, 어디서 병목이 발생하는가(O(N) 명령어, 큰 Value, KEYS) |
| [02. 메모리 관리 — jemalloc과 maxmemory 정책](./redis-internals/02-memory-management.md) | jemalloc 할당기가 메모리 단편화를 줄이는 방식, `mem_fragmentation_ratio`로 단편화 측정, `maxmemory-policy` 8가지(noeviction/allkeys-lru/volatile-lru 등) 내부 동작과 서비스 특성별 선택 기준 |
| [03. Redis 객체 시스템 — redisObject 구조체](./redis-internals/03-redis-object-system.md) | `redisObject`의 `type` / `encoding` / `lru` / `refcount` / `ptr` 5개 필드 구조, 동일 type에서 encoding이 자동 전환되는 조건(int→embstr→raw), 공유 정수 객체(0~9999)로 메모리를 절약하는 방식 |
| [04. 키 만료 메커니즘 — Lazy vs Active 만료](./redis-internals/04-key-expiry.md) | Lazy 만료(접근 시 만료 확인)와 Active 만료(주기적 샘플링)의 차이, `hz` 설정이 만료 속도에 미치는 영향, TTL이 지난 키가 메모리에 잠시 남아있을 수 있는 이유와 메모리 임팩트 |
| [05. Redis 6.0 Threaded I/O — 단일 스레드 모델의 진화](./redis-internals/05-threaded-io.md) | Redis 6.0이 I/O 스레드를 추가한 이유(네트워크 대역폭 병목), 명령어 실행은 여전히 단일 스레드인 이유, `io-threads` 설정 효과와 적합한 워크로드, Redis 6.0 이전/이후 처리량 비교 |

</details>

<br/>

### 🔹 Chapter 2: 데이터 구조 내부 구현

> **핵심 질문:** Redis의 각 데이터 구조는 메모리에서 어떻게 생겼고, 왜 인코딩이 자동으로 전환되며, 어느 상황에서 어느 구조를 써야 하는가?

<details>
<summary><b>SDS부터 Stream Consumer Group까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. String — SDS와 인코딩 전환](./data-structures/01-string-sds.md) | SDS(Simple Dynamic String)의 `len` / `alloc` / `flags` / `buf` 구조, 사전 할당으로 재할당 횟수를 줄이는 방식, int(정수) / embstr(44바이트 이하) / raw(44바이트 초과) 인코딩 전환 조건과 메모리 차이 |
| [02. List — quicklist의 ziplist 블록 구조](./data-structures/02-list-quicklist.md) | `ziplist` → `quicklist` 전환 역사와 이유, quicklist가 ziplist 노드를 청크로 묶는 구조, `list-max-ziplist-size` 설정이 청크 크기를 제어하는 방식, 양방향 연결 리스트 vs 연속 메모리 트레이드오프 |
| [03. Hash — listpack에서 hashtable로, 점진적 rehashing](./data-structures/03-hash-internals.md) | `hash-max-listpack-entries`/`hash-max-listpack-value` 임계값에서 listpack → hashtable 전환 조건, 점진적 rehashing이 두 개의 hashtable을 동시에 유지하면서 읽기/쓰기를 처리하는 방식, rehashing 중 메모리가 두 배가 되지 않는 이유 |
| [04. Set과 Sorted Set — intset / skiplist 내부](./data-structures/04-set-sortedset-internals.md) | Set의 intset이 정수를 압축 저장하고 이진 탐색으로 O(log N)을 보장하는 방식, Sorted Set의 `listpack` → `skiplist + hashtable` 전환 조건, 스킵 리스트가 `O(log N)` 삽입/삭제를 보장하는 확률적 레이어 구조 |
| [05. Bitmap, HyperLogLog, Geo — 특수 자료구조](./data-structures/05-bitmap-hyperloglog-geo.md) | Bitmap이 String 인코딩 위에서 동작하는 원리, HyperLogLog의 확률적 카디널리티 추정과 오차율(0.81%), Geo가 Sorted Set + Geohash로 위치를 저장하는 방식, 각 자료구조의 실제 메모리 사용량 비교 |
| [06. Stream — Consumer Group과 PEL](./data-structures/06-stream-consumer-group.md) | Stream의 Radix Tree 내부 저장 구조, Consumer Group이 각 소비자의 읽기 위치를 독립적으로 관리하는 방식, PEL(Pending Entry List)이 메시지 처리 확인(ACK 전)을 추적하는 원리, Kafka와의 포지셔닝 비교 |
| [07. 데이터 구조 선택 가이드 — 메모리 vs 성능 트레이드오프](./data-structures/07-data-structure-guide.md) | 같은 데이터를 String vs Hash vs List로 저장할 때 실제 메모리 사용량 비교 실험, `OBJECT ENCODING` / `DEBUG OBJECT` / `MEMORY USAGE`로 인코딩 상태를 확인하는 방법, 서비스 패턴별 최적 데이터 구조 선택 기준 |

</details>

<br/>

### 🔹 Chapter 3: 영속성 (Persistence) 완전 분해

> **핵심 질문:** RDB와 AOF는 어떻게 다르고, `fork()`의 Copy-On-Write는 어떻게 일관성을 보장하며, 장애 시 데이터 손실 허용 범위는 어떻게 계산하는가?

<details>
<summary><b>BGSAVE fork() 원리부터 서비스 특성별 영속성 설정까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. RDB 스냅샷 — BGSAVE와 fork() Copy-On-Write](./persistence/01-rdb-snapshot.md) | `BGSAVE`가 `fork()`로 자식 프로세스를 만들어 Copy-On-Write 방식으로 메모리 일관성을 유지하는 원리, fork 비용(메모리가 클수록 큰 이유), `save` 설정으로 자동 스냅샷 조건을 설정하는 방법, RDB 파일 포맷 개요 |
| [02. AOF — fsync 정책과 내구성 트레이드오프](./persistence/02-aof-fsync.md) | AOF가 모든 쓰기 명령어를 순서대로 기록하는 방식, `appendfsync always/everysec/no` 3가지 정책별 내구성(데이터 손실 범위)과 성능 트레이드오프, AOF Rewrite가 누적된 파일을 압축하는 원리, `BGREWRITEAOF` 동작 과정 |
| [03. RDB + AOF 혼합 모드 — Redis 4.0 Mixed Format](./persistence/03-rdb-aof-mixed.md) | AOF 파일의 앞부분에 RDB 스냅샷을 삽입하는 혼합 포맷의 구조, 재시작 시 RDB로 빠르게 복원 후 AOF 명령어로 차분 적용하는 복구 과정, 혼합 포맷의 장단점과 활성화 조건 |
| [04. 장애 복구 절차 — RDB/AOF로 데이터 복원](./persistence/04-disaster-recovery.md) | Redis가 시작할 때 AOF 우선 vs RDB 선택 기준, 손상된 AOF 파일을 `redis-check-aof`로 복구하는 절차, 부분 손실 허용 범위 계산(everysec 정책 → 최대 1초 데이터 손실), 복구 시뮬레이션 실험 |
| [05. 영속성 설정 실전 가이드 — 서비스 특성별 최적 조합](./persistence/05-persistence-config-guide.md) | 캐시 전용(영속성 불필요), 세션 저장소(일부 손실 허용), 주 데이터 저장소(손실 최소화) 시나리오별 `save` / `appendonly` / `appendfsync` 최적 설정, `maxmemory`와 영속성 정책의 상호작용 |

</details>

<br/>

### 🔹 Chapter 4: 복제와 고가용성

> **핵심 질문:** Replica는 어떻게 동기화되고, Sentinel은 어떻게 장애를 감지하며, Cluster는 16384개 슬롯을 어떻게 분산하는가?

<details>
<summary><b>PSYNC 프로토콜부터 클러스터 리샤딩까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Master-Replica 복제 — PSYNC 내부](./replication-ha/01-master-replica-psync.md) | Replica가 최초 연결 시 전체 재동기화(Full Resync)를 수행하는 과정, `repl_backlog_buffer`를 활용한 부분 재동기화(Partial Resync) 조건, 복제가 비동기인 이유와 데이터 유실 가능성, `INFO replication`으로 복제 상태 확인 |
| [02. Redis Sentinel — 자동 장애감지와 Failover](./replication-ha/02-sentinel-failover.md) | Sentinel이 Quorum 기반으로 Master 장애를 감지하는 방식, Failover 절차(새 Master 선출 → Replica 재연결 → 클라이언트 공지), 클라이언트가 Sentinel을 통해 새 Master 주소를 찾는 방법, Split-Brain 시나리오 |
| [03. Redis Cluster — 해시 슬롯과 Gossip 프로토콜](./replication-ha/03-cluster-hash-slot.md) | 16384개 해시 슬롯을 노드에 분산하는 원리(`CRC16(key) % 16384`), `{hash-tag}`로 같은 슬롯에 강제 배치하는 방법, 클러스터 노드 간 Gossip 프로토콜로 상태를 전파하는 방식, `CLUSTER NODES`로 슬롯 배치 확인 |
| [04. 클러스터 리샤딩 — ASK/MOVED 리다이렉션](./replication-ha/04-cluster-resharding.md) | 슬롯 이동 중 발생하는 `MOVED` 리다이렉션(슬롯이 완전히 이동됨)과 `ASK` 리다이렉션(슬롯이 이동 중임)의 차이, `CLUSTER SLOTS` 재배치 과정, Spring Redis Cluster 클라이언트가 리다이렉션을 자동 처리하는 방식 |
| [05. 복제 일관성 트레이드오프 — WAIT 명령어와 데이터 유실 범위](./replication-ha/05-replication-consistency.md) | Redis가 동기 복제를 지원하지 않는 이유, `WAIT numreplicas timeout`으로 특정 수의 Replica 복제를 확인하는 방법, `min-replicas-to-write` 설정으로 데이터 유실 범위를 제한하는 방식, 최종 일관성 허용 범위 설계 |

</details>

<br/>

### 🔹 Chapter 5: 캐싱 패턴과 고급 활용

> **핵심 질문:** Cache Stampede는 왜 발생하고 어떻게 막는가? 분산 락은 어떻게 구현하고, Redlock의 어떤 부분이 논란이 되는가?

<details>
<summary><b>Cache-Aside부터 Redlock 논쟁까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 캐싱 전략 비교 — Cache-Aside / Write-Through / Write-Back](./caching-patterns/01-caching-strategies.md) | Cache-Aside(Lazy Loading), Write-Through, Write-Back, Read-Through 각각의 구현 방식, 캐시-DB 일관성 보장 범위, 쓰기 증폭(Write Amplification) 문제, Spring `@Cacheable`이 내부적으로 구현하는 전략 |
| [02. Cache Stampede — PER과 Mutex Lock 해결법](./caching-patterns/02-cache-stampede.md) | TTL 만료 순간 수백 개 스레드가 동시에 DB를 조회하는 Thundering Herd 발생 원리, Mutex Lock(첫 번째만 DB 조회, 나머지 대기) 방식의 구현과 단점(락 대기 지연), PER(Probabilistic Early Recomputation)이 만료 전 확률적으로 갱신하는 원리 |
| [03. Hot Key 문제 — 읽기 vs 쓰기 Hot Key 해결](./caching-patterns/03-hot-key.md) | 특정 키에 트래픽이 집중될 때 단일 Redis 노드의 CPU/네트워크 병목이 발생하는 원인, 읽기 Hot Key 해결(로컬 캐시 + Redis 이중 레이어, 키 복제), 쓰기 Hot Key 해결(카운터 분산), `redis-cli --hotkeys`로 진단하는 방법 |
| [04. 분산 락 — SET NX EX와 Redlock 논쟁](./caching-patterns/04-distributed-lock.md) | `SET key value NX EX` 단일 명령으로 원자적 락을 구현하는 방식, Redlock 알고리즘(5개 노드 과반수 획득)의 설계 의도, Martin Kleppmann이 제기한 시간 가정 문제와 Antirez의 반론, 실무에서 Redlock 사용 여부 판단 기준 |
| [05. Pub/Sub vs Stream — 메시지 보관과 재처리](./caching-patterns/05-pubsub-vs-stream.md) | Pub/Sub의 Fire-and-Forget 특성(메시지 보관 없음, 구독자 없으면 소실), Stream의 영구 저장 + Consumer Group + ACK 기반 재처리 지원 구조, 각각이 적합한 사용 사례, Kafka와 Redis Stream의 포지셔닝 비교 |
| [06. Pipeline과 MULTI/EXEC — 원자성과 네트워크 최적화](./caching-patterns/06-pipeline-transaction.md) | Pipeline이 여러 명령어를 한 번의 TCP 왕복으로 처리해 RTT(Round Trip Time)를 줄이는 원리, `MULTI/EXEC`이 보장하는 원자성의 범위(명령어 실행 순서 보장이지 롤백 미보장), Lua 스크립트와의 차이, Spring `RedisTemplate.executePipelined()` |

</details>

<br/>

### 🔹 Chapter 6: 성능 튜닝과 운영

> **핵심 질문:** SLOWLOG로 어떻게 병목을 찾고, 메모리를 어떻게 최적화하며, 운영 중 OOM과 복제 지연은 어떻게 진단하는가?

<details>
<summary><b>SLOWLOG 분석부터 운영 장애 패턴까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. SLOWLOG와 병목 찾기](./performance-operations/01-slowlog-bottleneck.md) | `slowlog-log-slower-than` 설정, `SLOWLOG GET N`으로 느린 명령어 분석, O(N) 명령어(`KEYS`, `LRANGE`, `SMEMBERS`)가 단일 스레드를 블로킹하는 원리, `SCAN` 커서 기반 탐색으로 대체하는 방법 |
| [02. Lua 스크립트 — 원자적 복합 연산](./performance-operations/02-lua-scripting.md) | Lua 스크립트가 `MULTI/EXEC` 없이 원자적 복합 연산을 구현하는 원리, `EVAL` vs `EVALSHA`(스크립트 SHA1 캐싱으로 네트워크 절약), Lua 실행 중 Redis 이벤트 루프가 블로킹되는 주의사항, 분산 락/카운터 구현 예시 |
| [03. 메모리 최적화 실전 — 인코딩 임계값 튜닝](./performance-operations/03-memory-optimization.md) | `hash-max-listpack-entries` / `hash-max-listpack-value` 등 데이터 구조별 임계값 튜닝이 메모리에 미치는 실제 영향, `MEMORY USAGE key`로 정확한 메모리 측정, `redis-cli --bigkeys`로 대형 키 탐색, `OBJECT FREQ`로 접근 패턴 분석 |
| [04. Redis 모니터링 — INFO 지표와 Prometheus](./performance-operations/04-monitoring.md) | `INFO` 섹션별 핵심 지표(`used_memory`, `rdb_last_bgsave_status`, `repl_backlog_size`, `instantaneous_ops_per_sec`), `redis_exporter` + Prometheus + Grafana 대시보드 구성, 알림 기준값 설정 |
| [05. 운영 장애 패턴 — OOM / 복제 지연 / fork 지연](./performance-operations/05-operational-failures.md) | OOM 발생 → `maxmemory` 미설정 시 OS OOM Killer 동작 과정, 복제 지연 누적(`replication_lag`) 원인과 `repl-backlog-size` 조정, 클러스터 `FAIL` 상태 전환과 복구 절차, BGSAVE `fork()` 지연으로 인한 latency spike 진단 |

</details>

<br/>

### 🔹 Chapter 7: Spring과 Redis 통합

> **핵심 질문:** `RedisTemplate`의 직렬화 방식이 왜 성능에 영향을 주고, `@Cacheable`은 내부적으로 어떻게 동작하며, Spring Session을 Redis에 저장하면 무슨 일이 일어나는가?

<details>
<summary><b>직렬화 전략부터 Redisson 분산 세마포어까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Spring Data Redis — RedisTemplate 직렬화 전략](./spring-redis/01-redis-template-serialization.md) | `JdkSerializationRedisSerializer`(바이너리, 크기 큼) vs `Jackson2JsonRedisSerializer`(JSON, 가독성) vs `StringRedisSerializer`(문자열만) 비교, 직렬화 방식이 저장 크기와 역직렬화 속도에 미치는 실제 영향, `GenericJackson2JsonRedisSerializer`와 타입 정보 포함 이슈 |
| [02. Spring Cache 추상화 — @Cacheable 내부 AOP](./spring-redis/02-spring-cache-cacheable.md) | `@Cacheable` 호출 시 `CacheInterceptor` → `CacheAspectSupport` → `RedisCache` → `RedisTemplate` 전 과정, 키 생성 전략(`SimpleKeyGenerator` vs `KeyGenerator` 커스텀), `@CacheEvict` / `@CachePut`의 실행 시점과 `@Transactional`과의 조합 주의사항 |
| [03. Spring Session — HttpSession을 Redis에](./spring-redis/03-spring-session-redis.md) | `@EnableRedisHttpSession`이 `SessionRepositoryFilter`를 등록해 `HttpSession`을 Redis에 위임하는 방식, 세션 데이터 직렬화 전략 선택, TTL과 세션 만료 이벤트 처리(`SessionExpiredEvent`), Redis Cluster 환경에서 세션 키 배치(`{session}` 해시 태그) |
| [04. 분산 환경 패턴 — Redisson 분산 락과 세마포어](./spring-redis/04-redisson-distributed-patterns.md) | Redisson의 `RLock`이 Lua 스크립트로 원자적 락 획득/해제를 구현하는 방식, `RSemaphore`로 동시 처리 수를 제한하는 패턴, Pub/Sub 기반 락 대기(Busy-Waiting 없음), Spring Batch + Redis `ItemReader`로 분산 환경 배치 처리 |

</details>

---

## 🐳 실험 환경 (Docker Compose)

모든 챕터의 실험은 아래 환경에서 재현 가능합니다.

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7.0
    command: redis-server /etc/redis/redis.conf
    volumes:
      - ./redis.conf:/etc/redis/redis.conf
      - redis-data:/data
    ports:
      - "6379:6379"

  redis-replica:
    image: redis:7.0
    command: redis-server --replicaof redis 6379
    depends_on:
      - redis

  redis-sentinel:
    image: redis:7.0
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel.conf:/etc/redis/sentinel.conf
    depends_on:
      - redis
      - redis-replica

  redis-exporter:
    image: oliver006/redis_exporter
    environment:
      REDIS_ADDR: redis:6379
    ports:
      - "9121:9121"

volumes:
  redis-data:
```

```conf
# redis.conf 핵심 설정
maxmemory 256mb
maxmemory-policy allkeys-lru
save 900 1
save 300 10
appendonly yes
appendfsync everysec
slowlog-log-slower-than 10000
hz 10
```

```bash
# 실험용 공통 명령어 세트

# 인코딩 확인
redis-cli OBJECT ENCODING key_name

# 상세 내부 정보 (lru_seconds_idle, serializedlength 등)
redis-cli DEBUG OBJECT key_name

# 정확한 메모리 사용량 (바이트 단위)
redis-cli MEMORY USAGE key_name

# 만료 키 통계 및 전체 keyspace 정보
redis-cli INFO keyspace

# 메모리 전체 현황
redis-cli INFO memory

# 슬로우 쿼리 최근 10개
redis-cli SLOWLOG GET 10

# 실시간 명령 모니터링 (⚠️ 운영 환경 사용 주의)
redis-cli MONITOR

# 큰 키 분석 (샘플 기반, 원소 수 기준)
redis-cli --bigkeys

# 메모리 기준 큰 키 분석 (SCAN + MEMORY USAGE 조합)
redis-cli --no-auth-warning SCAN 0 COUNT 1000 | tail -n +2 | \
  xargs -I{} redis-cli MEMORY USAGE {} | paste - - | sort -k2 -rn | head -20

# 복제 상태 확인
redis-cli INFO replication
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — Redis를 블랙박스로 두는 접근과 그 결과 |
| ✨ **올바른 접근** | After — 원리를 알고 난 후의 설계/운영 |
| 🔬 **내부 동작 원리** | Redis 소스 레벨 분석 + 메모리 레이아웃 ASCII 구조도 |
| 💻 **실전 실험** | `redis-cli`, `OBJECT ENCODING`, `DEBUG OBJECT`, `MEMORY USAGE` |
| 📊 **성능/비용 비교** | 인코딩별 메모리, 직렬화 방식별 속도, 정책별 트레이드오프 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "TTL 만료 시 무슨 일이 일어나는지 모른다" — 캐싱 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch1-04  키 만료 메커니즘 (Lazy vs Active) → TTL 동작 원리 이해
Day 2  Ch5-01  캐싱 전략 비교 → Cache-Aside와 @Cacheable 연결
       Ch5-02  Cache Stampede → TTL 만료 시 실제 위험 이해
Day 3  Ch7-02  @Cacheable 내부 AOP → Spring Cache 동작 전 과정
```

</details>

<details>
<summary><b>🟡 "Redis 내부 구조를 제대로 이해하고 싶다" — 핵심 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch1-01  단일 스레드 이벤트 루프 → 왜 빠른가
       Ch1-03  Redis 객체 시스템 → redisObject 구조
Day 2  Ch2-01  String / SDS 인코딩 전환
       Ch2-03  Hash 점진적 rehashing
Day 3  Ch2-04  Sorted Set skiplist 구조
Day 4  Ch3-01  RDB fork() Copy-On-Write
       Ch3-02  AOF fsync 정책 트레이드오프
Day 5  Ch5-02  Cache Stampede + PER 해결법
       Ch5-04  분산 락 + Redlock 논쟁
Day 6  Ch6-01  SLOWLOG 병목 찾기 실전
       Ch6-03  메모리 최적화 임계값 튜닝
Day 7  Ch7-01  RedisTemplate 직렬화 전략 비교
       Ch7-04  Redisson 분산 락 패턴
```

</details>

<details>
<summary><b>🔴 "Redis 소스코드까지 파고들어 내부를 완전히 정복한다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — Redis 내부 아키텍처
        → epoll 이벤트 루프를 strace로 관찰, jemalloc 단편화 측정

2주차  Chapter 2 전체 — 데이터 구조 내부 구현
        → OBJECT ENCODING / DEBUG OBJECT로 인코딩 전환 경계 실험

3주차  Chapter 3 전체 — 영속성 완전 분해
        → BGSAVE 중 쓰기 발생 시 메모리 변화, AOF 파일 직접 분석

4주차  Chapter 4 전체 — 복제와 고가용성
        → Docker Compose로 Sentinel Failover 시뮬레이션, 슬롯 리샤딩 실험

5주차  Chapter 5 전체 — 캐싱 패턴과 고급 활용
        → Cache Stampede 의도적 발생 → PER/Mutex 비교 → Redlock 구현

6주차  Chapter 6 전체 — 성능 튜닝과 운영
        → SLOWLOG + MONITOR로 병목 탐색, OOM 재현 + maxmemory 정책 비교

7주차  Chapter 7 전체 — Spring과 Redis 통합
        → 직렬화별 저장 크기 측정, @Cacheable AOP 인터셉터 디버깅
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Auto-configuration, `RedisAutoConfiguration` 등록 원리 | Ch7-01(RedisTemplate 자동 구성), Ch7-02(@Cacheable Auto-configuration) |
| [spring-data-transaction](https://github.com/dev-book-lab/spring-data-transaction) | Spring Cache 추상화, `@Transactional`과 캐시 조합 | Ch7-02(@Cacheable과 @Transactional 조합 주의사항) |
| [spring-core-deep-dive](https://github.com/dev-book-lab/spring-core-deep-dive) | AOP Proxy, 인터셉터 체인 | Ch7-02(CacheInterceptor가 Proxy로 캐시 경계를 만드는 원리) |
| [database-internals](https://github.com/dev-book-lab/database-internals) | MySQL InnoDB MVCC, 트랜잭션 | Ch5-01(Redis 캐시와 DB 일관성 — Cache-Aside의 캐시-DB 동기화 문제) |

> 💡 이 레포는 **Redis 내부 동작**에 집중합니다. Spring을 모르더라도 순수 Redis 관점으로 Chapter 1~6을 학습할 수 있습니다. 단, Chapter 7은 Spring Data Redis 선행 지식이 있을 때 더 깊이 연결됩니다.

---

## 🙏 Reference

- [Redis 공식 문서](https://redis.io/docs/)
- [Redis 소스 코드 (GitHub)](https://github.com/redis/redis)
- [Redis in Action — Josiah L. Carlson](https://www.manning.com/books/redis-in-action)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/) — Redis 관련 챕터
- [Antirez 블로그](http://antirez.com) — Redis 설계 결정 이유 원문
- [Spring Data Redis Reference](https://docs.spring.io/spring-data/redis/docs/current/reference/html/)
- [Redis Persistence Demystified — Antirez](http://oldblog.antirez.com/post/redis-persistence-demystified.html)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"Redis를 캐시로 쓰는 것과, 데이터가 메모리에서 어떻게 저장되고 만료되는지 아는 것은 다르다"*

</div>
