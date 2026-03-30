# Redis 6.0 Threaded I/O — 단일 스레드 모델의 진화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Redis 6.0 이전과 이후의 I/O 처리 구조는 어떻게 다른가?
- "명령어 실행은 여전히 단일 스레드"라는 말의 의미는?
- Threaded I/O가 해결하는 병목과 해결하지 못하는 병목은 무엇인가?
- `io-threads` 설정은 어떻게 해야 하고, 언제 효과가 있는가?
- Threaded I/O 환경에서도 원자성(atomicity)이 보장되는 이유는?
- Redis 7.0에서 추가된 멀티스레드 관련 변화는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis 6.0의 Threaded I/O는 "모든 것을 빠르게"가 아니라 "I/O 병목만 해결"이다. 이 차이를 모르면 io-threads를 늘려도 성능이 개선되지 않는 상황에 당황하거나, 원자성이 깨진다고 잘못 우려하게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: io-threads 늘렸는데 성능이 그대로

  상황: "Redis가 느리다" → "Redis 6.0은 멀티스레드 지원!" → io-threads 8로 설정
  결과: 처리량 변화 없음 (또는 오히려 감소)

  원인: CPU 병목이었는데 I/O를 늘림
    CPU 100% 사용 중 (복잡한 Sorted Set 연산, 대형 Lua 스크립트)
    → 명령어 실행은 여전히 단일 스레드
    → io-threads는 소켓 I/O만 병렬화
    → CPU 병목에는 효과 없음

  올바른 진단을 안 한 이유:
    병목 타입을 구분하는 방법을 몰라서 (CPU vs I/O vs 네트워크)

실수 2: "io-threads 사용 = Race Condition 가능" 우려로 사용 안 함

  팀 논의: "Threaded I/O 켜면 동시성 문제 생기지 않나요?"
  결과: io-threads=1 유지, I/O 병목 해소 못 함
  
  실제: 명령어 실행은 메인 스레드 단일 순차 실행 유지
        MULTI/EXEC, INCR 모두 원자성 완전 보장
        I/O 스레드는 소켓 바이트 읽기/쓰기만 담당

실수 3: io-threads를 CPU 코어 수보다 높게 설정

  io-threads 16 (4코어 서버)
  → 16개 스레드가 4코어 경쟁 → Context Switch 폭증
  → 오히려 성능 저하
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
병목 타입 먼저 진단:
  top -p $(pgrep redis-server)
  → %CPU ≈ 100%:  CPU 병목 → io-threads 효과 없음 → Cluster 확장
  → %CPU ≈ 50~70%: I/O 병목 가능 → io-threads 시도

io-threads가 효과적인 조건:
  ✓ Value 크기 1 KB 이상 (네트워크 전송이 병목)
  ✓ 동시 연결 수 1,000개 이상
  ✓ 네트워크 대역폭 거의 포화

io-threads 설정 가이드:
  소량 Value GET/SET 위주 → io-threads 1 (기본, 효과 없음)
  대용량 Value + 높은 처리량 → io-threads 4 (코어 수 이하)
  변경은 redis.conf + 재시작 필요 (런타임 변경 불가)

벤치마크로 효과 검증:
  redis-benchmark -c 100 -n 500000 -d 1024 -t get,set
  # 변경 전후 처리량 비교
  # I/O 병목이라면 2~3배 향상 확인 가능
```

---

## 🔬 내부 동작 원리

### 1. Redis 6.0 이전 — 완전한 단일 스레드

```
Redis 6.0 이전 요청 처리 흐름:

클라이언트 A, B, C → (수천 개의 연결)

                  메인 스레드 (단일)
                       │
              ┌────────▼────────┐
              │  epoll_wait()   │  ① 이벤트 대기
              └────────┬────────┘
                       │ 이벤트 발생 (소켓 A, C에 데이터)
              ┌────────▼────────┐
              │ 소켓 A 읽기       │  ② 소켓에서 바이트 읽기 (recv())
              │ RESP 프로토콜     │  ③ 명령어 파싱
              │ 파싱             │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │ 명령어 실행        │  ④ 메모리에서 데이터 처리
              │ (GET/SET/...)   │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │ 응답 버퍼 작성     │  ⑤ RESP 응답 직렬화
              │ 소켓 A 쓰기       │  ⑥ 응답 전송 (send())
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │ 소켓 C 읽기       │  ⑦ 다음 소켓 처리 (A 처리 후)
              │ 명령어 파싱        │
              │ 명령어 실행        │
              │ 응답 전송         │
              └────────┬────────┘
                       │
              다시 epoll_wait()로

병목 지점:
  단계 ②③⑥이 네트워크 I/O — 특히 큰 Value나 대량 응답 시
  단계 ④이 CPU — 복잡한 명령어 처리

  네트워크 병목 예시:
    1MB Response를 소켓에 쓰는 동안 (수백 μs)
    다른 클라이언트의 요청은 전혀 처리 못 함
    → 고처리량, 대용량 Value 환경에서 단일 스레드가 I/O 포화 상태
```

### 2. Redis 6.0 Threaded I/O 구조

```
Redis 6.0 이후 요청 처리 흐름:

io-threads = 4 설정 시:

  I/O Thread 1  I/O Thread 2  I/O Thread 3
       │              │              │
       │              │              │
  ─────▼──────────────▼──────────────▼──────── (소켓 분배)
  소켓 A,D,G     소켓 B,E,H     소켓 C,F,I

  ┌──────────────────────────────────────────────────┐
  │                  메인 스레드                        │
  │                                                  │
  │  ① epoll_wait() — 이벤트 감지 (메인만)               │
  │                                                  │
  │  ② 준비된 소켓을 I/O 스레드에 분배                     │
  │     Thread 1 → 소켓 A, D, G                       │
  │     Thread 2 → 소켓 B, E, H                       │
  │     Thread 3 → 소켓 C, F, I                       │
  │                                                  │
  │  ③ I/O 스레드들이 병렬로 소켓 읽기 + 파싱               │
  │     [메인 스레드는 여기서 대기]                        │
  │                                                  │
  │  ④ 모든 I/O 스레드 파싱 완료 후 동기화 (barrier)        │
  │                                                  │
  │  ⑤ 메인 스레드가 파싱된 명령어를 순서대로 실행 ← 핵심!      │
  │     Thread 1이 파싱한 소켓 A 명령어 실행               │
  │     Thread 2가 파싱한 소켓 B 명령어 실행               │
  │     ...                                          │
  │                                                  │
  │  ⑥ 응답을 I/O 스레드에 분배 → 병렬 소켓 쓰기             │
  │                                                  │
  └──────────────────────────────────────────────────┘

핵심:
  파싱(recv + RESP 파싱): 멀티스레드 병렬
  명령어 실행:            메인 스레드 단일 순차 실행 ← 원자성 유지
  응답 쓰기(send):        멀티스레드 병렬

  "Threaded I/O"라는 이름이 정확한 이유:
    멀티스레드는 I/O(소켓 읽기/쓰기)에만 적용
    CPU-intensive한 명령어 실행은 단일 스레드로 변함없음
```

### 3. 병목 분석 — Threaded I/O가 효과 있는 경우

```
병목 유형 1: 네트워크 I/O 병목

  증상:
    CPU 사용률: 50~60% (단일 코어 포화)
    네트워크 대역폭: 서버 한계에 근접
    Value 크기가 크거나 (1KB~1MB)
    처리량(ops/sec)이 기대치의 절반 이하

  Threaded I/O의 효과:
    소켓 읽기/쓰기가 여러 스레드로 분산
    단일 코어 포화 해소 → 처리량 증가
    효과: io-threads=4로 처리량 2~3배 향상 (가이드 참고)

병목 유형 2: CPU 병목 (명령어 실행)

  증상:
    CPU 사용률: 100% (단일 코어 포화)
    Value가 작고 (< 100 bytes) 처리량 높음
    복잡한 명령어: SORT, ZUNIONSTORE, Lua 스크립트

  Threaded I/O의 효과:
    → 거의 없음
    명령어 실행은 여전히 단일 스레드
    I/O를 빠르게 해도 실행이 병목이므로 전체 처리량 변화 없음

  해결책:
    여러 Redis 인스턴스 실행 (같은 서버에 포트 6379, 6380, 6381...)
    Redis Cluster로 수평 확장
    Lua 스크립트 최소화, O(N) 명령어 제거

병목 유형 3: 메모리 대역폭 병목

  증상:
    RAM bandwidth 포화 (서버의 메모리 대역폭 한계)
    대용량 Hash/List 조회 (전체 메모리 순회)

  Threaded I/O의 효과:
    → 없음 (메모리 접근은 여전히 단일 스레드)

진단 방법:
  # CPU 사용 현황
  top -p $(pgrep redis-server)
  # 단일 코어가 100%에 가까우면 CPU 병목
  # 60% 이하이면 I/O 병목 가능성

  # 네트워크 처리량
  redis-cli INFO stats | grep -E "total_net_input|total_net_output"
  # 초당 입출력 바이트 계산으로 대역폭 확인

  # instantaneous throughput
  redis-cli INFO stats | grep instantaneous
  # instantaneous_ops_per_sec: 현재 초당 처리량
  # instantaneous_input_kbps: 현재 입력 대역폭 (kbps)
  # instantaneous_output_kbps: 현재 출력 대역폭 (kbps)
```

### 4. io-threads 설정 가이드

```
기본 설정 (redis.conf):
  io-threads 1            # 기본값: Threaded I/O 비활성 (단일 스레드)
  io-threads-do-reads yes # 읽기도 스레드 사용 (쓰기만은 기본 no)

io-threads 설정 원칙:
  io-threads 1:  Threaded I/O 비활성 (6.0 이전과 동일)
  io-threads 2~4: 일반적인 권장 범위
  io-threads N:  N은 물리 CPU 코어 수를 초과하지 말 것
                 초과하면 스레드 경합으로 오히려 성능 저하

권장 설정:
  ┌─────────────────────────────────────────────┬───────────────────┐
  │ 환경                                         │ io-threads 권장    │
  ├─────────────────────────────────────────────┼───────────────────┤
  │ 소량 Value(< 100 bytes), 단순 GET/SET 위주     │ 1 (비활성)          │
  │ 중간 Value, 혼합 워크로드                       │ 2~4               │
  │ 대용량 Value(1KB+), 높은 처리량 필요             │ 4~8 (코어 수 이하)   │
  │ CPU 병목 (복잡한 명령어)                        │ 1 (효과 없음)        │
  └─────────────────────────────────────────────┴───────────────────┘

io-threads-do-reads:
  yes: 읽기(소켓 recv + 파싱)도 I/O 스레드로 처리
  no (기본): 쓰기(소켓 send)만 I/O 스레드로 처리, 읽기는 메인

  → 일반적으로 yes가 더 효과적이지만 실험으로 확인 권장

적용 후 검증:
  redis-benchmark -c 100 -n 1000000 -t get,set -d 1024
  # -d 1024: 1KB Value로 I/O 병목 시뮬레이션
  # io-threads 전후 처리량 비교
```

### 5. 원자성 보장 — Threaded I/O에서도 안전한 이유

```
원자성이 유지되는 이유:

  모든 명령어 실행은 메인 스레드 단일 실행
  
  MULTI/EXEC 블록:
    MULTI   → 메인 스레드가 큐 시작 표시
    SET a 1 → 큐에 추가 (실행 안 함)
    SET b 2 → 큐에 추가
    EXEC    → 메인 스레드가 순서대로 실행 (다른 명령어 끼어들기 불가)

  I/O 스레드의 역할:
    소켓에서 "EXEC\r\n" 바이트를 읽고 파싱해서 명령어 객체 생성
    이 명령어 객체를 메인 스레드의 실행 큐에 전달
    메인 스레드가 순서대로 실행 → 원자성 유지

  Lua 스크립트:
    EVAL script → 메인 스레드에서 Lua 인터프리터 실행
    스크립트 전체가 단일 스레드에서 원자적으로 실행
    I/O 스레드는 스크립트 내 명령어를 볼 수 없음

경쟁 조건(Race Condition)이 없는 이유:
  I/O 스레드: 소켓 데이터 읽기 (데이터 구조 접근 없음)
  메인 스레드: 모든 Redis 데이터 구조 접근 (순차 실행)
  → 데이터 구조를 동시에 접근하는 스레드는 메인 스레드 하나뿐
  → Lock 없이도 안전
```

### 6. Redis 7.0+ 변화 — 추가 멀티스레드 개선

```
Redis 7.0: 추가 멀티스레드 개선

  1. Fork-free RDB (개발 중): 일부 작업의 포크 의존도 감소
  2. 통계 수집 개선: 멀티스레드 환경에서 정확한 stats
  3. WAIT 명령어 개선

Redis의 멀티스레드 철학:
  "명령어 실행의 단순성과 원자성을 유지하면서
   I/O 병목만 제거한다"

  MySQL, PostgreSQL처럼 "모든 것을 멀티스레드"가 아닌:
  "I/O만 멀티스레드, 실행은 단일 스레드"

이 철학의 장점:
  기존 싱글스레드 Redis의 모든 보장이 유지됨
  MULTI/EXEC, Lua, 원자 명령어 모두 재설계 불필요
  디버깅과 동작 예측이 여전히 단순

이 철학의 한계:
  CPU 병목은 여전히 해결 못 함
  복잡한 연산의 병렬화 불가
  → Redis Cluster로 수평 확장이 근본 해결책

실무 결론:
  "io-threads 설정만으로 Redis 처리량을 2~4배 늘릴 수 있다"
  조건: 네트워크/I/O 병목인 경우
  조건: Value 크기가 크거나, 연결 수가 매우 많은 경우
  주의: CPU 병목이면 효과 없음 → 진단 먼저
```

---

## 💻 실전 실험

### 실험 1: Threaded I/O 활성화 및 성능 비교

```bash
# 기준 성능 측정 (io-threads=1, 단일 스레드)
redis-cli CONFIG SET io-threads 1

# 1KB Value로 처리량 측정 (I/O 병목 시뮬레이션)
redis-benchmark -c 50 -n 500000 -t get,set -d 1024 --csv
# GET: XXX requests/sec
# SET: XXX requests/sec

# Threaded I/O 활성화 (io-threads는 런타임 변경 불가 → redis.conf 수정 필요)
# redis.conf 수정:
cat >> /etc/redis/redis.conf << 'EOF'
io-threads 4
io-threads-do-reads yes
EOF

# Redis 재시작
docker restart redis-test

# 동일 벤치마크 재실행
redis-benchmark -c 50 -n 500000 -t get,set -d 1024 --csv
# 처리량 증가 확인 (I/O 병목이었다면 2~3배 향상)

# 소량 Value 벤치마크 (CPU 병목 시뮬레이션)
redis-benchmark -c 50 -n 500000 -t get,set -d 8 --csv
# 작은 Value에서는 io-threads 효과가 제한적
```

### 실험 2: INFO server로 스레드 설정 확인

```bash
# 현재 설정 확인
redis-cli INFO server | grep -E "redis_version|io_threads_active|io_threads"

# 실행 중인 스레드 확인 (Linux)
ps -eLf | grep redis-server | grep -v grep
# PID와 LWP(Light Weight Process) 수 확인
# io-threads=4이면 메인 스레드 + I/O 스레드 4개 + 백그라운드 스레드

# top으로 CPU 사용 확인 (H 키로 스레드별 보기)
top -H -p $(pgrep redis-server)
# io-threads 활성 시 여러 스레드가 CPU를 나눠 사용하는 것 확인
```

### 실험 3: 병목 유형 진단

```bash
# 실시간 처리량 및 대역폭 모니터링
watch -n 1 'redis-cli INFO stats | grep -E "instantaneous"'

# 출력:
# instantaneous_ops_per_sec:45000    ← 현재 초당 명령어 수
# instantaneous_input_kbps:23000     ← 현재 입력 대역폭 (kbps)
# instantaneous_output_kbps:45000    ← 현재 출력 대역폭 (kbps)

# CPU 사용률 확인
top -p $(pgrep redis-server)
# %CPU가 99~100%이면 → CPU 병목 (io-threads 효과 없음)
# %CPU가 50~70%이면 → I/O 병목 가능 (io-threads 효과 있음)

# 네트워크 대역폭 확인
sar -n DEV 1  # 또는 iftop, nethogs

# Redis 연결 수
redis-cli INFO clients | grep connected_clients
# 연결이 수천 개 이상이고 처리량이 기대치 이하 → I/O 병목 가능성
```

### 실험 4: 원자성 검증

```bash
# Threaded I/O 환경에서 MULTI/EXEC 원자성 확인
redis-cli CONFIG SET io-threads 4 2>/dev/null || echo "재시작 필요"

# 동시 카운터 증가 테스트 (Race Condition 없어야 함)
redis-cli SET atomic_counter 0

# 10개 클라이언트가 각각 MULTI/EXEC로 100번 증가
for i in $(seq 1 10); do
  (
    for j in $(seq 1 100); do
      redis-cli MULTI > /dev/null
      redis-cli INCR atomic_counter > /dev/null
      redis-cli EXEC > /dev/null
    done
  ) &
done
wait
redis-cli GET atomic_counter  # 반드시 1000이어야 함 (Race Condition 없음)
```

---

## 📊 성능 비교

```
공식 Redis 벤치마크 (redis.io 가이드라인 기반):

환경: 8코어 CPU, 10G 네트워크, Value 크기 1KB

io-threads| SET (ops/sec) | GET (ops/sec) | 대역폭 활용
──────────┼───────────────┼───────────────┼────────────
1 (기본)   | ~200,000      | ~200,000      | 제한적
2         | ~320,000      | ~340,000      | 향상
4         | ~450,000      | ~500,000      | 양호
8         | ~500,000      | ~550,000      | 거의 최대

참고: 실제 성능은 Value 크기, 연결 수, 서버 사양에 따라 크게 다름

소량 Value (8 bytes) 환경:
  io-threads=1 vs io-threads=4 → 차이 거의 없음
  (I/O가 병목이 아니므로)

대용량 Value (10KB) 환경:
  io-threads=1 vs io-threads=4 → 3~4배 차이 가능
  (I/O가 확실한 병목이므로)
```

---

## ⚖️ 트레이드오프

```
Threaded I/O 사용 결정:

활성화 권장:
  - Value 크기가 크거나 (1KB+)
  - 동시 연결 수가 많고 (1000개+)
  - 네트워크 대역폭이 포화 상태
  - CPU 코어가 4개 이상

활성화 비권장:
  - 소량 Value GET/SET 위주 (8~100 bytes)
  - CPU 병목인 경우 (io-threads 늘려도 효과 없음)
  - 2코어 이하 서버 (스레드 경합 위험)

주의 사항:
  - io-threads는 런타임 CONFIG SET으로 변경 불가 → redis.conf + 재시작 필요
  - Sentinel/Cluster 환경에서는 재시작 시 Failover 절차 필요
  - 스레드 수를 물리 코어 수 이하로 유지 (초과 시 성능 저하)

Threaded I/O vs Redis Cluster:
  Threaded I/O: 단일 인스턴스의 I/O 병목 해결
  Redis Cluster: CPU/메모리 병목 + 수평 확장
  → CPU 병목이 실제 문제라면 Cluster가 올바른 선택
```

---

## 📌 핵심 정리

```
Redis 6.0 Threaded I/O 핵심:

변화한 것:
  소켓 읽기(recv + RESP 파싱): 멀티스레드 병렬 실행
  소켓 쓰기(send):              멀티스레드 병렬 실행

변하지 않은 것:
  명령어 실행: 여전히 메인 스레드 단일 순차 실행
  원자성 보장: MULTI/EXEC, INCR, Lua 스크립트 모두 안전
  데이터 구조 접근: 항상 메인 스레드 하나만

설정:
  io-threads 4           # I/O 전용 스레드 4개
  io-threads-do-reads yes # 읽기도 스레드 사용

효과 있는 경우:
  ✓ 대용량 Value (1KB+)
  ✓ 높은 동시 연결 수 (1000+)
  ✓ 네트워크 대역폭 포화

효과 없는 경우:
  ✗ CPU 병목 (복잡한 명령어 실행)
  ✗ 소량 Value 단순 GET/SET
  ✗ 2코어 이하 서버

병목 진단:
  CPU ~100% → CPU 병목 → Cluster 확장
  CPU 50~70%, 처리량 낮음 → I/O 병목 → io-threads 증가
```

---

## 🤔 생각해볼 문제

**Q1.** `io-threads 4`를 설정했는데 `redis-benchmark`에서 처리량이 오히려 감소했다. 원인이 무엇일 수 있는가?

<details>
<summary>해설 보기</summary>

다음 원인을 확인하세요:

1. **Value가 너무 작음**: 소량 Value(8 bytes) 환경에서는 I/O가 병목이 아닙니다. 멀티스레드의 동기화 오버헤드(barrier 동기화 등)가 오히려 성능을 낮출 수 있습니다.

2. **코어 수 부족**: 2코어 서버에 io-threads=4를 설정하면 4개 스레드가 2개 코어를 경합 → Context Switch 증가 → 성능 저하.

3. **redis-benchmark 클라이언트 수 부족**: `-c 1`처럼 연결이 적으면 I/O 스레드가 빈 시간이 많아 스레드 오버헤드만 발생. `-c 50` 이상으로 테스트하세요.

4. **io-threads-do-reads no (기본값)**: 읽기는 여전히 메인 스레드. 쓰기만 스레드화 → 읽기 집약 워크로드에서 효과 제한.

해결: `redis-benchmark -c 100 -d 1024`로 충분한 연결과 큰 Value로 재테스트.

</details>

---

**Q2.** Threaded I/O가 활성화된 Redis에서 `MULTI`-`GET a`-`SET a 1`-`EXEC`를 실행할 때, 각 단계가 I/O 스레드와 메인 스레드 중 어디서 처리되는가?

<details>
<summary>해설 보기</summary>

```
I/O 스레드가 처리:
  MULTI\r\n             → 소켓에서 읽어서 RESP 파싱 → 명령어 객체 생성
  GET a\r\n             → 소켓에서 읽어서 RESP 파싱 → 명령어 객체 생성
  SET a 1\r\n           → 소켓에서 읽어서 RESP 파싱 → 명령어 객체 생성
  EXEC\r\n              → 소켓에서 읽어서 RESP 파싱 → 명령어 객체 생성

메인 스레드가 처리:
  MULTI 실행            → 클라이언트의 MULTI 플래그 설정
  GET a 실행           → 큐에 저장 (실제 실행 안 함)
  SET a 1 실행         → 큐에 저장
  EXEC 실행            → 큐의 GET a, SET a 1 순서대로 단일 스레드 실행

  응답 전송:
I/O 스레드가 처리:
  응답 버퍼 → 소켓 send()  → 클라이언트로 전송
```

I/O 스레드는 "바이트 읽기/파싱/쓰기"만 담당, 실제 `GET`/`SET` 실행은 전부 메인 스레드입니다.

</details>

---

**Q3.** 단일 Redis 인스턴스 대신 같은 서버에 Redis 인스턴스를 4개 실행하는 방식과 io-threads=4를 설정하는 방식의 차이점과 각각의 적합한 상황은?

<details>
<summary>해설 보기</summary>

**io-threads=4 (단일 인스턴스):**
- 네트워크 I/O만 4개 스레드로 분산
- 명령어 실행은 여전히 단일 스레드
- 데이터가 하나의 keyspace에 있어 모든 명령어가 원자적으로 상호작용 가능
- 적합: 데이터 일관성이 필요한 경우 (MULTI/EXEC, Lua 스크립트), I/O 병목인 경우

**4개 독립 인스턴스 (포트 6379, 6380, 6381, 6382):**
- 각 인스턴스가 독립 CPU 코어를 사용
- 명령어 실행이 실질적으로 병렬화됨 (서로 다른 프로세스이므로)
- 데이터가 분산되어 인스턴스 간 원자적 연산 불가
- 적합: CPU 병목인 경우, 데이터를 논리적으로 분리할 수 있는 경우 (예: DB 0~3)

**선택 기준:**
```
CPU 100% → 4개 인스턴스 (실질적 CPU 병렬화)
네트워크 포화, CPU 여유 → io-threads=4 (I/O 병렬화)
MULTI/EXEC나 원자성이 중요 → io-threads=4 (단일 keyspace 필요)
독립적인 서비스 데이터 → 4개 인스턴스 (격리)
```

</details>

---

<div align="center">

**[⬅️ 이전: 키 만료 메커니즘](./04-key-expiry.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — String과 SDS ➡️](../data-structures/01-string-sds.md)**

</div>
