# 단일 스레드 이벤트 루프 — 왜 빠른가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Redis는 단일 스레드인데 왜 수만 개의 동시 연결을 처리할 수 있는가?
- `epoll`/`kqueue` I/O 멀티플렉싱은 `select`와 어떻게 다르고, 왜 빠른가?
- 단일 스레드임에도 Redis가 빠른 이유는 무엇이고, 어디서 병목이 발생하는가?
- `KEYS *`, `SMEMBERS`, `LRANGE 0 -1` 같은 명령어가 위험한 이유는?
- Redis에서 "느린 명령어"와 "느린 네트워크"는 어떻게 구분하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis는 단일 스레드임에도 수만 개의 동시 연결을 처리한다. 이 구조가 어떻게 작동하는지 이해하지 못하면, 특정 명령어 하나가 전체 서비스를 멈추는 이유를 영원히 파악할 수 없다. 이벤트 루프는 Redis의 성능 상한과 병목 지점을 동시에 결정한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
상황: 운영 중 Redis 응답 시간이 갑자기 수백 ms로 치솟음

원리를 모를 때의 판단:
  "Redis가 느려졌다 → 서버 스펙 부족 → 인스턴스 업그레이드"
  또는 "캐시 히트율 문제" → 만료 정책 변경
  또는 "네트워크 문제" → 인프라팀에 전달

결과:
  인스턴스를 두 배 비싼 것으로 교체해도 간헐적 latency spike 재발
  근본 원인을 건드리지 않았기 때문

실제 원인 (나중에 발견):
  관리 스크립트가 주기적으로 KEYS * 실행
  100만 개 키에 KEYS * → 이벤트 루프 수백 ms 독점
  그 시간 동안 다른 모든 클라이언트 요청이 큐에서 대기
  → 타임아웃, 에러 폭발

또 다른 흔한 실수:
  Lua 스크립트 안에서 SMEMBERS(대형 Set) 호출
  → 스크립트 전체가 원자적으로 실행 = 이벤트 루프 독점
  MULTI/EXEC 블록 안에 O(N) 명령어 삽입
  → 블록 전체 실행 시간만큼 다른 요청 차단
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
원리를 알 때의 진단 흐름:
  "Redis가 느리다" → SLOWLOG GET 10 확인
  → KEYS * 발견 → O(N) 명령어가 이벤트 루프 독점
  → SCAN 0 MATCH * COUNT 100 으로 교체 (커서 기반 분할 처리)
  → latency spike 즉시 해소

올바른 설계 원칙:
  이벤트 루프 = 단일 차선 도로
    한 번에 하나의 명령어만 통과 가능
    O(N) 명령어는 그 시간 동안 도로 전체를 막음

  O(N) 명령어 → SCAN 계열로 대체
    KEYS *       → SCAN 0 MATCH * COUNT 100
    SMEMBERS key → SSCAN key 0 COUNT 100
    HGETALL key  → HSCAN key 0 COUNT 100

  운영 도구:
    SLOWLOG GET 20                   # 느린 명령어 목록
    INFO commandstats | grep usec    # 명령어별 평균 실행 시간
    redis-cli --latency              # 실시간 응답 시간 모니터링
    CONFIG SET slowlog-log-slower-than 1000  # 1ms 이상 기록
```

---

## 🔬 내부 동작 원리

### 1. 전통적 멀티스레드 서버 vs Redis 이벤트 루프

```
전통적 멀티스레드 서버 (예: Apache, Tomcat 기본 설정):

  클라이언트 1 → Thread 1 (소켓 읽기 대기 중...)
  클라이언트 2 → Thread 2 (소켓 읽기 대기 중...)
  클라이언트 3 → Thread 3 (소켓 읽기 대기 중...)
  ...
  클라이언트 N → Thread N (소켓 읽기 대기 중...)

  문제:
    Thread 1이 클라이언트로부터 데이터를 기다리는 동안 → Thread는 Block
    10,000 클라이언트 → 10,000 Thread
    Thread 생성 비용: 메모리(스택 1MB~8MB/Thread) + OS 스케줄링 비용
    Context Switch: Thread 간 전환 시 CPU 레지스터 저장/복원
    → 동시 연결이 늘어날수록 Thread 관리 오버헤드가 처리량을 압도

Redis 이벤트 루프 (단일 스레드 + I/O 멀티플렉싱):

  단일 Thread
    │
    ▼
  ┌──────────────────────────────────────────────┐
  │            이벤트 루프 (ae.c)                   │
  │                                              │
  │  ① aeApiPoll() → epoll_wait() 호출            │
  │     "읽기/쓰기 준비된 소켓이 생길 때까지 대기"         │
  │     (CPU 사용 없이 OS에 위임)                    │
  │                                              │
  │  ② 이벤트 발생 → 준비된 소켓 목록 수신              │
  │     [소켓 42: 읽기 가능, 소켓 87: 쓰기 가능,       │
  │      소켓 113: 읽기 가능, ...]                  │
  │                                              │
  │  ③ 각 소켓에 등록된 핸들러 순서대로 호출             │
  │     readQueryFromClient(소켓 42)              │
  │     sendReplyToClient(소켓 87)                │
  │     readQueryFromClient(소켓 113)             │
  │                                              │
  │  ④ ①로 돌아가 다시 대기                          │
  └──────────────────────────────────────────────┘

  핵심: I/O 대기 시간 = 0
    소켓에 데이터가 없으면 그 소켓은 처리하지 않음
    준비된 소켓만 처리 → CPU가 항상 실제 작업에만 집중
    단일 Thread로 수만 연결을 지연 없이 처리 가능
```

### 2. epoll의 동작 원리 — select와의 결정적 차이

```
select (구식 방법):
  fd_set readfds;   // 감시할 소켓 목록 (비트맵)
  FD_SET(sock1, &readfds);
  FD_SET(sock2, &readfds);
  ...
  select(max_fd + 1, &readfds, NULL, NULL, &timeout);
  // "이 중에서 준비된 것 있어?"

  문제:
    매 호출마다 전체 fd_set을 커널로 복사 (O(N) 복사)
    커널이 반환한 fd_set을 전부 스캔해서 어느 소켓이 준비됐는지 확인 (O(N) 스캔)
    1,000개 연결 → 매 루프마다 O(1,000) 작업
    → 연결이 많을수록 select 자체가 병목

epoll (Linux, Redis가 사용):

  1단계: epoll 인스턴스 생성 (단 한 번)
    epfd = epoll_create(1);

  2단계: 감시할 소켓 등록 (소켓 추가 시 한 번씩)
    struct epoll_event ev;
    ev.events = EPOLLIN;  // 읽기 이벤트 감시
    ev.data.fd = sock_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, sock_fd, &ev);
    // 커널이 내부적으로 이 소켓을 추적 목록에 추가

  3단계: 이벤트 대기 (매 루프)
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
    // 준비된 소켓만 events 배열에 담아 반환
    // 10,000개 연결 중 3개만 준비됐으면 nfds = 3
    
    for (int i = 0; i < nfds; i++) {
        handle(events[i].data.fd);  // 준비된 것만 처리
    }

epoll의 핵심 장점:
  등록: O(1) per socket (커널 내부 red-black tree에 삽입)
  대기: O(1) (커널이 이벤트 발생 시 직접 준비 목록에 추가)
  처리: O(이벤트 수) — 전체 연결 수가 아닌 준비된 소켓 수에만 비례

  10,000 연결 중 10개만 동시에 데이터 송수신 중
  → select: O(10,000) 스캔
  → epoll:  O(10) 처리
  → 1,000배 차이!

kqueue (macOS/BSD):
  epoll과 동일한 원리, API만 다름
  kevent() 로 이벤트 등록/대기

Redis 추상화 레이어 (ae.c):
  ae_epoll.c   → Linux: epoll 사용
  ae_kqueue.c  → macOS/BSD: kqueue 사용
  ae_select.c  → 그 외: select (폴백)
  → 플랫폼 무관하게 동일한 이벤트 루프 인터페이스
```

### 3. Redis 이벤트 루프 전체 구조

```
Redis 프로세스 시작 후 main() → aeMain() 호출:

┌─────────────────────────────────────────────────────────┐
│                    aeMain() 루프                         │
│                                                         │
│  매 루프마다:                                              │
│                                                         │
│  [Before Sleep Hook]                                    │
│    → activeExpireCycle()   만료 키 주기적 삭제               │
│    → flushAppendOnlyFile() AOF 버퍼 플러시                 │
│    → handleClientsWithPendingWrites() 쓰기 대기 클라이언트   │
│                                                         │
│  [aeApiPoll — epoll_wait]                               │
│    → OS에 제어 넘김, I/O 이벤트 발생까지 대기                   │
│    → 이벤트 발생 시 즉시 깨어남                               │
│                                                         │
│  [파일 이벤트 처리]                                         │
│    읽기 이벤트: readQueryFromClient()                      │
│      → 소켓에서 RESP 프로토콜 파싱                            │
│      → 명령어 큐에 추가                                     │
│                                                         │
│  [명령어 실행]                                             │
│    processCommand()                                     │
│      → 명령어 함수 호출 (setCommand, getCommand, ...)       │
│      → 메모리에서 데이터 읽기/쓰기                             │
│      → 응답 버퍼에 결과 저장                                 │
│                                                         │
│  [쓰기 이벤트 처리]                                         │
│    sendReplyToClient()                                  │
│      → 응답 버퍼를 소켓으로 전송                              │
│                                                         │
│  [시간 이벤트 처리]                                         │
│    serverCron() (100ms마다)                              │
│      → 통계 업데이트, 만료 키 샘플링, 복제 heartbeat            │
│                                                         │
└─────────────────────────────────────────────────────────┘

중요: 모든 과정이 단일 스레드에서 순서대로 실행
  → 클라이언트 A의 명령어 실행 중 클라이언트 B는 대기
  → Race Condition, Mutex, Deadlock이 원천적으로 없음
  → 메모리 접근에 락이 필요 없음 → 오버헤드 0
```

### 4. 왜 메모리 접근은 빠른가

```
CPU ↔ 메모리 접근 시간 비교:

  L1 Cache:     ~1 ns
  L2 Cache:     ~4 ns
  L3 Cache:     ~10 ns
  RAM (DRAM):   ~100 ns   ← Redis 데이터 여기
  SSD:          ~100 μs   (RAM의 1,000배)
  HDD:          ~10 ms    (RAM의 100,000배)

Redis가 인메모리인 이유:
  GET/SET 하나의 처리 시간
    디스크 DB: 파싱 + 인덱스 탐색 + 디스크 I/O + 네트워크 = 수 ms
    Redis:     파싱 + 해시 테이블 조회 + 네트워크 = 수십 μs ~ 수백 μs

  네트워크 왕복(RTT)이 처리 시간보다 더 큰 경우도 많음
  → Redis 처리 자체: ~1 μs
  → 로컬 네트워크 RTT: ~0.1 ms = 100 μs
  → 처리가 네트워크보다 100배 빠름

단일 스레드 + 인메모리의 시너지:
  멀티스레드 서버: 락 경합 오버헤드 + Context Switch 비용 상존
  Redis: 락 없음 + Context Switch 없음 + 메모리만 접근
  → 단순하고 예측 가능한 성능
```

### 5. 병목이 발생하는 지점 — 이벤트 루프를 막는 것들

```
이벤트 루프는 단일 스레드이므로 "하나의 명령어가 오래 걸리면 다른 모든 요청이 대기"

병목 유형 1: O(N) 명령어

  KEYS pattern      → 전체 keyspace 스캔 (N = 전체 키 수)
  SMEMBERS key      → Set의 모든 원소 반환 (N = Set 크기)
  LRANGE key 0 -1   → List 전체 반환 (N = List 크기)
  HGETALL key       → Hash 전체 반환 (N = Hash 필드 수)
  SORT key          → 정렬 (N log N)
  FLUSHDB / FLUSHALL → 전체 삭제

  100만 개 키에 KEYS * 실행:
    처리 시간 수백 ms ~ 수 초
    그 동안 다른 모든 클라이언트의 요청이 Queue에서 대기
    → 타임아웃 발생 → 클라이언트 오류

병목 유형 2: 큰 Value 처리

  HGETALL로 필드 10만 개짜리 Hash 반환
  SET key 10MB_string
  → 직렬화/역직렬화 + 네트워크 전송 시간이 이벤트 루프 차지

병목 유형 3: 동기 persistence

  SAVE 명령어 (동기 RDB 저장) → 완료될 때까지 이벤트 루프 멈춤
  → BGSAVE (비동기, fork) 또는 AOF 사용 권장

진단 명령어:
  redis-cli SLOWLOG GET 10          # 느린 명령어 확인
  redis-cli LATENCY HISTORY event   # 지연 히스토리
  redis-cli --latency               # 실시간 지연 측정
  redis-cli INFO commandstats       # 명령어별 호출 횟수/누적 시간

안전한 대안:
  KEYS * → SCAN 0 MATCH * COUNT 100   (커서 기반, 분할 처리)
  SMEMBERS → SSCAN                    (Set 분할 스캔)
  HGETALL → HSCAN                     (Hash 분할 스캔)
```

### 6. RESP 프로토콜 — 클라이언트와의 통신 형식

```
Redis Serialization Protocol (RESP):
  클라이언트 → Redis: 명령어 전송
  Redis → 클라이언트: 응답 전송

예시: SET mykey "hello" 전송
  클라이언트 → Redis:
    *3\r\n          ← 3개의 인자
    $3\r\n          ← 첫 번째 인자 길이 3
    SET\r\n
    $5\r\n          ← 두 번째 인자 길이 5
    mykey\r\n
    $5\r\n          ← 세 번째 인자 길이 5
    hello\r\n

  Redis → 클라이언트:
    +OK\r\n         ← Simple String 응답

예시: GET mykey 응답
  Redis → 클라이언트:
    $5\r\n          ← Bulk String (길이 5)
    hello\r\n

RESP3 (Redis 6.0 이후):
  더 풍부한 타입 지원 (Map, Set, Double, Boolean 등)
  타입 정보를 응답에 포함 → 클라이언트 파싱 부담 감소
  Lettuce/Jedis 최신 버전에서 RESP3 지원

파싱의 단순함:
  RESP는 텍스트 기반의 단순한 프로토콜
  → readQueryFromClient()에서 파싱 비용 최소화
  → 이벤트 루프 처리 시간의 대부분은 실제 명령어 실행
```

---

## 💻 실전 실험

### 실험 1: 이벤트 루프 블로킹 재현

```bash
# 실험 환경 시작
docker run -d --name redis-test -p 6379:6379 redis:7.0

# 터미널 1: 1만 개 키 생성
redis-cli -n 0 DEBUG SLEEP 0  # Redis 연결 확인

for i in $(seq 1 10000); do
  redis-cli SET "key:$i" "value-$i" > /dev/null
done
echo "10,000개 키 생성 완료"

# 터미널 2: 지연 실시간 측정 시작
redis-cli --latency-history -i 1

# 터미널 1: KEYS * 실행 (이벤트 루프 블로킹 유발)
time redis-cli KEYS "*" | wc -l
# 터미널 2에서 latency 급등 확인

# 터미널 1: SCAN으로 동일 작업 (비블로킹)
redis-cli SCAN 0 COUNT 100  # 커서 기반, 이벤트 루프 양보하며 처리
```

### 실험 2: SLOWLOG로 느린 명령어 포착

```bash
# SLOWLOG 설정 (1ms 이상 걸린 명령어 기록)
redis-cli CONFIG SET slowlog-log-slower-than 1000   # 1000 마이크로초 = 1ms
redis-cli CONFIG SET slowlog-max-len 128

# 의도적으로 느린 명령어 실행
redis-cli KEYS "*"
redis-cli DEBUG SLEEP 0.1  # 100ms 강제 대기

# SLOWLOG 확인
redis-cli SLOWLOG GET 5
# 출력 예시:
# 1) 1) (integer) 2          ← 로그 ID
#    2) (integer) 1711234567  ← Unix timestamp
#    3) (integer) 105234      ← 실행 시간 (마이크로초)
#    4) 1) "DEBUG"            ← 실행된 명령어
#       2) "SLEEP"
#       3) "0.1"
#    5) "127.0.0.1:54321"     ← 클라이언트 주소
#    6) ""                    ← 클라이언트 이름

redis-cli SLOWLOG LEN    # 총 기록된 개수
redis-cli SLOWLOG RESET  # 초기화
```

### 실험 3: commandstats로 명령어별 성능 분석

```bash
redis-cli INFO commandstats

# 출력 예시:
# cmdstat_get:calls=15234,usec=45123,usec_per_call=2.96
# cmdstat_set:calls=8456,usec=23456,usec_per_call=2.77
# cmdstat_keys:calls=3,usec=234567,usec_per_call=78189.00  ← KEYS는 평균 78ms!

# 해석:
#   usec_per_call: 명령어 평균 실행 시간 (마이크로초)
#   GET: 평균 2.96 μs → 정상
#   KEYS: 평균 78,189 μs (= 78ms) → 위험!

# O(N) 명령어 탐지 스크립트
redis-cli INFO commandstats | awk -F'[=,]' '
{
    for(i=1;i<=NF;i++) {
        if($i == "usec_per_call") {
            val=$(i+1)
            if(val+0 > 1000) {  # 1ms 초과
                print $0, "← 주의: 평균", val/1000, "ms"
            }
        }
    }
}'
```

### 실험 4: 실시간 명령어 모니터링

```bash
# 터미널 1: MONITOR 실행 (⚠️ 운영 환경 절대 금지 — CPU 2배 증가)
redis-cli MONITOR

# 터미널 2: 명령어 실행
redis-cli SET foo bar
redis-cli GET foo
redis-cli HSET user:1 name "Alice" age 30

# 터미널 1 출력:
# 1711234567.123456 [0 127.0.0.1:54321] "SET" "foo" "bar"
# 1711234567.124567 [0 127.0.0.1:54321] "GET" "foo"
# 1711234567.125678 [0 127.0.0.1:54321] "HSET" "user:1" "name" "Alice" "age" "30"
```

---

## 📊 성능 비교

```
동시 연결 처리 모델 비교:

모델                  | 동시 연결 1,000개   | 동시 연결 10,000개
─────────────────────┼──────────────────┼──────────────────
멀티스레드 (1:1)        | 1,000 Thread     | 10,000 Thread
                     | 메모리: ~8GB       | OOM 가능성 높음
                     | Context Switch   | Context Switch 폭증
─────────────────────┼──────────────────┼──────────────────
epoll 이벤트 루프       | 1 Thread         | 1 Thread
(Redis 방식)          | 메모리: 최소        | 메모리: 최소
                     | Context Switch 0 | Context Switch 0

명령어 복잡도별 처리 시간 (키 100만 개 기준):

명령어           | 복잡도    | 처리 시간 (참고치)
───────────────┼──────────┼──────────────────
GET key        | O(1)     | ~1 μs
SET key value  | O(1)     | ~1 μs
HGET key field | O(1)     | ~1 μs
ZADD key score | O(log N) | ~2 μs
KEYS *         | O(N)     | ~100 ms (100만 키)
SMEMBERS key   | O(N)     | N에 비례
SORT key       | O(N logN)| N에 비례

결론:
  O(1) 명령어 → 이벤트 루프에 부담 없음
  O(N) 명령어 → 루프 1회를 N에 비례하는 시간 독점 → 전체 지연 유발
```

---

## ⚖️ 트레이드오프

```
단일 스레드 이벤트 루프의 장단점:

장점:
  ① Race Condition 없음 → 락/뮤텍스 오버헤드 0
  ② Context Switch 없음 → CPU 캐시 효율 최대
  ③ 디버깅 용이 → 명령어가 항상 순서대로 실행
  ④ 원자성 보장 → 단일 명령어는 절대 중단되지 않음

단점:
  ① CPU 코어 1개만 사용
     → 멀티코어 서버에서 나머지 코어는 Redis에 기여 못 함
     → 해결: 같은 서버에 여러 Redis 인스턴스 + Cluster
  
  ② 느린 명령어가 전체를 차단
     → O(N) 명령어, 큰 Value, Lua 스크립트
     → 해결: 명령어 선택 주의, SLOWLOG 모니터링

  ③ 대용량 Value 직렬화/전송 시 병목
     → 10MB Value를 SET/GET할 때 이벤트 루프 차지
     → 해결: Value 크기 제한, 분산 저장

Redis 6.0+ Threaded I/O:
  → 명령어 실행은 여전히 단일 스레드
  → 소켓 읽기/쓰기(I/O)만 멀티스레드
  → 네트워크 대역폭 병목을 해결하지만 CPU 병목은 해결 못 함
  (자세한 내용은 05-threaded-io.md 참고)
```

---

## 📌 핵심 정리

```
Redis 단일 스레드 이벤트 루프 핵심:

구조:
  단일 스레드 + epoll(Linux)/kqueue(macOS) I/O 멀티플렉싱
  → 준비된 소켓만 처리 → I/O 대기 시간 0
  → 수만 연결을 Thread 없이 처리

빠른 이유 3가지:
  ① 인메모리: 디스크 I/O 없음 (~100 ns vs ~100 μs)
  ② 락 없음: 단일 스레드이므로 동기화 오버헤드 0
  ③ Context Switch 없음: Thread 간 전환 비용 없음

병목이 발생하는 경우:
  ① O(N) 명령어: KEYS, SMEMBERS, HGETALL → SCAN 계열로 대체
  ② 큰 Value: 10MB+ Value 처리 시 이벤트 루프 독점
  ③ SAVE: 동기 RDB 저장 → BGSAVE 사용

진단 명령어:
  SLOWLOG GET 10         # 느린 명령어 확인
  INFO commandstats      # 명령어별 평균 실행 시간
  redis-cli --latency    # 실시간 지연 측정
```

---

## 🤔 생각해볼 문제

**Q1.** Redis가 단일 스레드임에도 `GET`과 `SET`이 원자적이라는 것의 의미는? 그렇다면 `INCR`은 왜 Race Condition이 없는가?

<details>
<summary>해설 보기</summary>

단일 스레드 이벤트 루프에서는 **한 번에 하나의 명령어만 실행**됩니다. `INCR key`는 내부적으로 "GET → 1 더하기 → SET" 세 단계지만, 이 전체가 단일 스레드에서 끊김 없이 실행됩니다. 다른 클라이언트의 명령어가 중간에 끼어들 수 없습니다.

멀티스레드 언어에서의 `count++`는 "읽기 → 더하기 → 쓰기" 사이에 다른 스레드가 끼어들 수 있어 Race Condition이 발생합니다. Redis의 `INCR`은 이 세 단계가 이벤트 루프 안에서 분리될 수 없으므로 완벽히 원자적입니다.

```bash
# 10개 클라이언트가 동시에 INCR 1000번 실행해도 최종값은 항상 10,000
redis-cli SET counter 0
for i in $(seq 1 10); do
  (for j in $(seq 1 1000); do redis-cli INCR counter > /dev/null; done) &
done
wait
redis-cli GET counter  # 항상 10000
```

</details>

---

**Q2.** `redis-cli --latency`로 측정한 지연이 0.1ms인데 애플리케이션에서 Redis 호출이 10ms씩 걸린다. 병목은 Redis인가, 네트워크인가, 아니면 다른 곳인가?

<details>
<summary>해설 보기</summary>

`redis-cli --latency`는 Redis 서버 자체의 처리 지연(이벤트 루프 내 처리 시간)을 측정합니다. 0.1ms가 나왔다면 **Redis 서버 자체는 정상**입니다.

애플리케이션에서 10ms가 걸리는 원인 후보:

1. **네트워크 RTT**: Redis 서버가 다른 AZ(Availability Zone)에 있으면 RTT 자체가 1~5ms. `ping redis-host`로 확인
2. **커넥션 풀 고갈**: 모든 커넥션이 사용 중 → 커넥션 대기 시간이 10ms 발생. `INFO clients`의 `connected_clients` 확인
3. **직렬화 비용**: 애플리케이션에서 Java 객체 → JSON → Redis 직렬화에 시간 소요
4. **O(N) 명령어**: 애플리케이션이 보내는 특정 명령어가 느림. SLOWLOG 확인

진단 순서: SLOWLOG → INFO clients → ping RTT 측정 → 직렬화 코드 프로파일링

</details>

---

**Q3.** `MULTI/EXEC` 트랜잭션 블록 안에 `KEYS *`가 있으면 어떤 문제가 생기는가? `WATCH`를 함께 사용할 때 이벤트 루프 관점에서 무슨 일이 일어나는가?

<details>
<summary>해설 보기</summary>

`MULTI/EXEC` 블록은 `EXEC` 명령어 도달 시 큐에 쌓인 명령어들을 **순서대로 한 번에 실행**합니다. 이 실행 과정은 이벤트 루프에서 **중단 없이 연속 실행**되므로, 블록 안에 `KEYS *` 같은 O(N) 명령어가 있으면 다른 클라이언트의 요청이 전체 블록 실행 시간만큼 대기합니다.

`WATCH key`는 `EXEC` 전에 해당 키가 변경되면 트랜잭션을 취소(nil 반환)합니다. 이벤트 루프 관점에서는:
1. `WATCH` 등록 → 키의 dirty 플래그 감시 시작
2. 다른 클라이언트가 해당 키를 수정 → dirty 플래그 set
3. `EXEC` 도달 → dirty 플래그 확인 → 변경됐으면 전체 트랜잭션 취소

`WATCH`는 **낙관적 잠금(Optimistic Locking)**으로, 락을 걸지 않으므로 이벤트 루프를 차단하지 않습니다. 단, 경합이 심하면 재시도가 많아집니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 메모리 관리 — jemalloc과 maxmemory 정책 ➡️](./02-memory-management.md)**

</div>
