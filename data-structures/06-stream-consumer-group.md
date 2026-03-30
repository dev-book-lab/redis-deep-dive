# Stream — Consumer Group과 PEL

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Stream의 메시지 ID `1711234567890-0`은 어떻게 구성되어 있는가?
- Radix Tree(기수 트리)가 메시지 저장에 적합한 이유는?
- Consumer Group의 `last-delivered-id`와 `pending`의 차이는?
- PEL(Pending Entry List)이 없으면 어떤 문제가 발생하는가?
- `XACK`를 빠뜨렸을 때 메시지는 어떻게 처리되는가?
- Redis Stream과 Kafka의 포지셔닝 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Stream은 Redis에서 메시지를 영구 저장하고 Consumer Group으로 처리를 보장하는 유일한 자료구조다. PEL(Pending Entry List)이 없으면 처리 중 크래시 시 메시지가 사라진다는 것을 모르면, "한 번은 반드시 처리된다"는 보장이 없는 시스템을 만들게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 비즈니스 이벤트에 Pub/Sub 사용

  코드:
    PUBLISH order:events '{"order_id": 123, "status": "paid"}'

  장애 시나리오:
    ① Consumer 서버 재배포 중 (30초 연결 끊김)
       → 이 시간의 이벤트 전부 소실 (Pub/Sub: 구독자 없으면 버림)
       → 결제된 주문 처리 안 됨 → 고객 불만

    ② Consumer가 처리 도중 서버 OOM으로 크래시
       → 처리 중이던 메시지 재처리 방법 없음 (ACK 개념 없음)
       → 주문 상태 불명확

    ③ 두 Consumer가 동일 이벤트 중복 처리
       → 동일 주문 두 번 결제 처리
       → 환불 이슈

실수 2: XACK를 빠뜨린 Consumer 코드

  코드:
    while True:
        msgs = xreadgroup(group, consumer, ">", count=10)
        for msg in msgs:
            process(msg)
        # XACK 누락!

  결과:
    PEL이 계속 쌓임 (처리됐지만 ACK 안 된 메시지)
    XAUTOCLAIM이 주기적으로 같은 메시지를 재전달
    → 중복 처리 + PEL 메모리 누수
    → 며칠 후 XPENDING 확인 시 수만 건 pending

실수 3: Stream 크기 제한 없이 XADD

  코드: XADD events '*' data {payload}  # MAXLEN 없음
  결과: Stream 메시지가 무한히 누적
        1년 후 수억 건 → 수십 GB 메모리
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
비즈니스 이벤트 처리: Pub/Sub → Stream으로 전환

  # Producer
  XADD order:events '*' order_id 123 status paid amount 29900

  # Consumer Group 생성 (한 번만)
  XGROUP CREATE order:events processors $ MKSTREAM

  # Consumer (올바른 패턴)
  while True:
      # 1. 자신의 pending 먼저 처리 (재시작 후 미처리 복구)
      pending = xreadgroup(group, consumer, "0-0", count=10)
      if pending:
          process_and_ack(pending)

      # 2. 새 메시지 처리
      new_msgs = xreadgroup(group, consumer, ">", count=10)
      if new_msgs:
          process_and_ack(new_msgs)

  def process_and_ack(msgs):
      for msg in msgs:
          try:
              process(msg)
              xack(stream, group, msg.id)  # 반드시 ACK
          except Exception:
              pass  # ACK 안 함 → PEL에 남아 재처리 가능

독소 메시지 처리 (delivery_count 높은 메시지):
  pending = xpending(stream, group, count=100)
  for msg in pending:
      if msg.delivery_count > 10:
          xadd("dead_letter", msg)  # DLQ로 이동
          xack(stream, group, msg.id)

Stream 크기 관리:
  XADD events MAXLEN ~ 100000 '*' ...  # 최대 10만 건 유지
  or XTRIM events MAXLEN 100000        # 주기적 정리
```

---

## 🔬 내부 동작 원리

### 1. Stream 기본 구조 — 메시지 ID와 저장 방식

```
Stream 메시지 ID 구조:
  <Unix timestamp ms>-<sequence>
  예: 1711234567890-0
      │             └── 같은 ms 내 순서 (0부터)
      └── Unix timestamp (밀리초)

자동 생성 ID ('*' 사용):
  XADD mystream '*' field1 value1
  → Redis가 현재 타임스탬프 + 시퀀스 자동 부여
  → 단조 증가 보장 (같은 ms이면 시퀀스 증가)

수동 ID 지정:
  XADD mystream '1711234567890-5' field1 value1
  → 이전 ID보다 커야 함 (강제 단조 증가)

ID의 의미:
  시간 순서 = ID 순서 (자동 생성 시)
  XRANGE, XREVRANGE로 시간 범위 탐색 가능
  
메시지 엔트리 구조:
  각 메시지 = ID + field-value 쌍들
  XADD mystream '*' user_id 1001 action "login" timestamp "2024-01-01"
  → ID: 1711234567890-0
     fields: {user_id: "1001", action: "login", timestamp: "2024-01-01"}
```

### 2. Radix Tree (기수 트리) — 내부 저장 구조

```
Redis Stream의 내부 저장소: rax (Radix Tree)
  파일: rax.h, rax.c

Radix Tree의 특성:
  트리의 각 엣지: 문자열의 공통 접두사(prefix)
  "1711234567890-0"
  "1711234567890-1"
  "1711234567891-0"
  
  공통 접두사 "171123456789"를 공유:
  
  "171123456789"
       ├── "0-0" → 메시지 [1]
       ├── "0-1" → 메시지 [2]
       └── "1-0" → 메시지 [3]

실제 저장 최적화:
  listpack으로 메시지 그룹화:
    같은 millisecond의 메시지들 → 하나의 listpack 노드로 묶임
    listpack 노드: [메시지1 필드들][메시지2 필드들]...
  
  Radix Tree 노드 → listpack 노드 포인터

메모리 효율:
  ID의 공통 접두사 공유 → ID 저장 공간 절약
  같은 시간대 메시지를 연속 메모리에 묶음 → 캐시 친화
  listpack 내부: 필드 이름 중복 제거 (첫 메시지 이후 필드 이름 생략)

XLEN key:
  Stream의 전체 메시지 수
  O(1) — 카운터 필드 유지

XRANGE key start end [COUNT count]:
  start ID 이상, end ID 이하 메시지 반환
  O(log N + M) — N = 전체, M = 반환 메시지 수
  시작 탐색: Radix Tree → O(log N)
  순차 반환: listpack 연속 읽기 → O(M)

XREVRANGE key end start:
  역순 반환 (최신 메시지부터)
```

### 3. Consumer Group — 독립 읽기 커서

```
Consumer Group 생성:
  XGROUP CREATE mystream mygroup $ MKSTREAM
  
  $: 현재 마지막 ID (이후 추가되는 메시지만 처리)
  0: 스트림 처음부터 (기존 메시지도 처리)
  MKSTREAM: 스트림이 없으면 자동 생성

Consumer Group 내부 구조:

  mystream (Radix Tree):
    [msg1: id=100-0, data=...]
    [msg2: id=101-0, data=...]
    [msg3: id=102-0, data=...]
    [msg4: id=103-0, data=...]

  Consumer Group "orders-group":
    ┌─────────────────────────────────────────────────────┐
    │  last-delivered-id: 102-0  (마지막으로 전달된 메시지)     │
    │                                                     │
    │  PEL (Pending Entry List):                          │
    │    102-0 → consumer1, delivered_at=1711234567000    │
    │    (ACK 안 된 메시지 목록)                              │
    │                                                     │
    │  consumers:                                         │
    │    consumer1: last_seen=..., pending_count=1        │
    │    consumer2: last_seen=..., pending_count=0        │
    └─────────────────────────────────────────────────────┘

XREADGROUP GROUP mygroup consumer1 COUNT 2 STREAMS mystream >:
  '>' = last-delivered-id 이후의 새 메시지
  
  1. last-delivered-id(102-0) 이후 메시지 탐색
  2. msg3(103-0) 전달 → consumer1에게
  3. PEL에 추가: 103-0 → consumer1, delivered_at=now
  4. last-delivered-id = 103-0 업데이트

  consumer2에서 XREADGROUP:
  1. last-delivered-id(103-0) 이후 메시지 탐색
  2. msg4(104-0) 전달 → consumer2에게
  3. PEL에 추가: 104-0 → consumer2
  4. last-delivered-id = 104-0 업데이트
  
  결과: 같은 그룹 내에서 메시지가 하나의 Consumer에게만 전달

같은 그룹 내 메시지 분배:
  msg3 → consumer1
  msg4 → consumer2
  msg5 → consumer1 (라운드 로빈은 아님, 먼저 XREADGROUP 호출한 Consumer)
  → 여러 Consumer가 병렬 처리 가능
```

### 4. PEL (Pending Entry List) — 처리 보장의 핵심

```
PEL의 목적:
  "전달됐지만 아직 ACK 받지 못한 메시지" 추적
  Consumer가 크래시해도 메시지 재처리 가능

PEL 엔트리 구조:
  {
    message_id: "103-0",
    consumer_name: "consumer1",
    delivery_time: 1711234567000,  (ms timestamp)
    delivery_count: 1              (몇 번 전달됐는지)
  }

ACK 처리:
  XACK mystream mygroup 103-0
  → PEL에서 103-0 제거
  → consumer1의 pending_count 감소

ACK 없이 크래시 시:
  consumer1 크래시
  103-0은 PEL에 남아있음 (delivery_time이 오래됨)
  
  다른 Consumer가 재처리:
  XPENDING mystream mygroup - + 10
  → PEL 내 pending 메시지 목록 조회
  # 1) 1) "103-0"
  #    2) "consumer1"
  #    3) (integer) 30000  ← 전달 후 경과 ms
  #    4) (integer) 1      ← 전달 횟수
  
  XCLAIM mystream mygroup consumer2 60000 103-0
  → 60초 이상 경과한 103-0을 consumer2로 재할당
  → PEL 업데이트: consumer_name=consumer2, delivery_count=2

XAUTOCLAIM (Redis 6.2+):
  자동으로 오래된 pending 메시지 재할당
  XAUTOCLAIM mystream mygroup consumer2 60000 0-0 COUNT 10
  → 60초 이상 idle인 pending 메시지 최대 10개를 consumer2에 할당
  → delivery_count 증가 (재처리 횟수 추적)
  → delivery_count가 너무 높으면 "독소 메시지"(poison pill) 처리 필요

PEL 관련 주의:
  XACK를 빠뜨리면 PEL이 계속 쌓임 → 메모리 증가
  XPENDING으로 정기적으로 확인 필요
  최대 PEL 크기 제한은 없음 → 운영 모니터링 필수
```

### 5. Stream 소비 패턴 비교

```
패턴 1: 단순 소비 (Consumer Group 없이)
  XREAD COUNT 10 STREAMS mystream 0-0
  
  → 모든 소비자가 모든 메시지를 받음 (Broadcast)
  → ACK/PEL 없음 → 처리 보장 없음
  → Pub/Sub과 유사하지만 메시지 영구 저장
  → 사용 예: 감사 로그, 이벤트 스트림 다중 소비

패턴 2: Consumer Group
  XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS mystream >
  XACK mystream mygroup message_id
  
  → 각 메시지가 하나의 Consumer에게만 전달 (Competing Consumers)
  → PEL로 처리 보장
  → 사용 예: 작업 큐, 이벤트 처리 워커

패턴 3: XREAD BLOCK (Long Polling)
  XREAD COUNT 1 BLOCK 0 STREAMS mystream $
  BLOCK 0: 새 메시지가 올 때까지 무한 대기
  BLOCK 5000: 5초 대기 후 nil 반환
  
  → Pub/Sub의 블로킹 구독과 유사하지만 메시지 저장됨
  → 단일 Consumer가 실시간 스트리밍 처리

패턴 4: XLEN + XRANGE 배치 처리
  while True:
    messages = XRANGE mystream lastId '+' COUNT 100
    if messages:
      process(messages)
      lastId = messages[-1].id
    sleep(1)
  
  → 1초마다 최대 100개씩 배치 처리
  → BLPOP 없이도 가능하지만 폴링 오버헤드 있음
```

### 6. Stream 관리 — 크기 제한과 XADD

```
Stream 크기 관리:
  MAXLEN 옵션:
    XADD mystream MAXLEN 1000 '*' field value
    → 메시지 추가 후 1000개 초과 시 오래된 것 제거
    → O(1) amortized (listpack 단위로 제거)
  
  MAXLEN ~ (근사값):
    XADD mystream MAXLEN ~ 1000 '*' field value
    → 정확히 1000개가 아닌 근사값 허용
    → listpack 단위로 제거 (더 효율적)
    → 실제: 약 1000~1100개 유지

  XDEL key id:
    특정 메시지 삭제
    listpack에서 논리적 삭제 (즉시 메모리 회수 아님)
    XTRIM으로 실제 정리

  XTRIM key MAXLEN count:
    현재 Stream을 지정 크기로 정리

Stream 영속성:
  다른 Redis 자료구조와 동일
  AOF/RDB에 포함
  복제 시 Replica에도 복제
  
  Consumer Group 상태도 RDB/AOF에 포함:
  last-delivered-id, PEL 정보 모두 저장
  → 재시작 후에도 처리 상태 복원
```

### 7. Redis Stream vs Apache Kafka

```
┌─────────────────────────┬────────────────────────┬────────────────────────┐
│ 특성                     │ Redis Stream           │ Apache Kafka           │
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ 저장 방식                 │ 메모리 (AOF/RDB 백업)     │ 디스크 (파티션 로그)        │
│ 처리량                   │ 초당 수십만 건 (메모리)      │ 초당 수백만 건 (디스크)     │
│ 메시지 보존               │ 메모리 한도 내             │ 설정된 기간/크기           │
│ 컨슈머 그룹               │ 지원 (단순)               │ 지원 (파티션별 병렬)       │
│ 파티셔닝                  │ 키 샤딩 (Cluster)        │ 파티션 내장              │
│ 메시지 재처리              │ XCLAIM, XAUTOCLAIM     │ offset reset           │
│ 멱등성 보장                │ 없음 (직접 구현)          │ Exactly-Once (트랜잭션)  │
│ 운영 복잡도                │ 낮음                    │ 높음 (ZooKeeper/KRaft)  │
│ 적합한 메시지 수            │ 수백만 건 이하            │ 수십억 건 이상             │
└─────────────────────────┴────────────────────────┴────────────────────────┘

Redis Stream 적합:
  ✓ 적은 메시지 수 (수백만 건)
  ✓ 빠른 응답 우선 (메모리 처리)
  ✓ Redis를 이미 사용 중
  ✓ 간단한 이벤트 파이프라인
  ✓ 소규모 팀 (운영 부담 최소)

Kafka 적합:
  ✓ 대용량 이벤트 (수십억 건)
  ✓ 장기 메시지 보존 (수 일~수 달)
  ✓ 다양한 Consumer 그룹이 독립 오프셋으로 처리
  ✓ Exactly-Once 의미론 필요
  ✓ 이벤트 소싱, 감사 로그
```

---

## 💻 실전 실험

### 실험 1: Stream 기본 사용

```bash
# Stream 생성 및 메시지 추가
redis-cli XADD mystream '*' event login user_id 1001 timestamp 1711234567000
redis-cli XADD mystream '*' event purchase user_id 1002 amount 29900
redis-cli XADD mystream '*' event logout user_id 1001

# 메시지 조회
redis-cli XLEN mystream                    # 3
redis-cli XRANGE mystream - +             # 전체
redis-cli XRANGE mystream - + COUNT 2     # 최대 2개
redis-cli XREVRANGE mystream + - COUNT 1  # 최신 1개

# 특정 ID 이후
FIRST_ID=$(redis-cli XRANGE mystream - + COUNT 1 | head -1)
redis-cli XRANGE mystream "$FIRST_ID" + COUNT 2  # 첫 메시지 이후 2개

# Stream 정보
redis-cli XINFO STREAM mystream
redis-cli XINFO STREAM mystream FULL  # Consumer Group 포함 상세
```

### 실험 2: Consumer Group과 PEL

```bash
# Consumer Group 생성 (스트림 처음부터)
redis-cli XGROUP CREATE mystream workers 0 MKSTREAM

# Consumer 1 읽기
MSG1=$(redis-cli XREADGROUP GROUP workers consumer1 COUNT 1 STREAMS mystream ">")
echo "Consumer1 받은 메시지: $MSG1"

# Consumer 2 읽기
MSG2=$(redis-cli XREADGROUP GROUP workers consumer2 COUNT 1 STREAMS mystream ">")
echo "Consumer2 받은 메시지: $MSG2"

# Pending 확인 (ACK 안 된 메시지)
redis-cli XPENDING mystream workers - + 10
# 출력: 각 메시지 ID, consumer 이름, idle time, delivery count

# Consumer1 ACK
ID1=$(echo $MSG1 | awk '{print $2}')  # 첫 번째 메시지 ID 파싱
redis-cli XACK mystream workers $ID1

# ACK 후 Pending 재확인
redis-cli XPENDING mystream workers - + 10  # consumer1 메시지 사라짐

# Consumer Group 정보
redis-cli XINFO GROUPS mystream
```

### 실험 3: XAUTOCLAIM으로 오래된 메시지 재처리

```bash
# 메시지 추가
for i in $(seq 1 5); do
  redis-cli XADD tasks '*' task_id $i priority high > /dev/null
done

# Consumer Group 생성
redis-cli XGROUP CREATE tasks task-workers 0

# Consumer1이 모든 메시지 읽고 ACK 안 함 (크래시 시뮬레이션)
redis-cli XREADGROUP GROUP task-workers crashed-consumer COUNT 10 STREAMS tasks ">" > /dev/null

# Pending 확인
redis-cli XPENDING tasks task-workers - + 10

# 잠시 대기 (idle time 측정을 위해)
sleep 2

# Consumer2가 1000ms 이상 idle인 메시지 재할당
redis-cli XAUTOCLAIM tasks task-workers recovery-consumer 1000 0-0 COUNT 10

# recovery-consumer가 받은 메시지 확인
redis-cli XPENDING tasks task-workers - + 10
# delivery_count가 2로 증가한 것 확인

# 재처리 후 ACK
redis-cli XRANGE tasks 0-0 + COUNT 10 | awk 'NR%2==1{print $1}' | while read id; do
  redis-cli XACK tasks task-workers $id > /dev/null
done
redis-cli XPENDING tasks task-workers - + 10  # 비어있어야 함
```

### 실험 4: MAXLEN으로 크기 제한

```bash
# MAXLEN으로 메시지 추가 (정확한 크기)
for i in $(seq 1 200); do
  redis-cli XADD capped_stream MAXLEN 100 '*' index $i > /dev/null
done
redis-cli XLEN capped_stream  # 100 (초과분 제거)

# 근사 MAXLEN (더 효율적)
for i in $(seq 1 2000); do
  redis-cli XADD approx_stream "MAXLEN" "~" 1000 '*' index $i > /dev/null
done
redis-cli XLEN approx_stream  # 약 1000~1100개

# Stream 메모리 분석
redis-cli MEMORY USAGE capped_stream
redis-cli MEMORY USAGE approx_stream
redis-cli XINFO STREAM capped_stream  # radix-tree-keys, radix-tree-nodes 확인
```

---

## 📊 성능 비교

```
Stream 연산 복잡도:

연산                         | 복잡도
────────────────────────────┼──────────────────────────
XADD                        | O(1)
XLEN                        | O(1)
XRANGE start end COUNT n    | O(log N + M)
XREAD COUNT n               | O(log N + M)
XREADGROUP COUNT n          | O(log N + M + P) (P=PEL 업데이트)
XACK id                     | O(1)
XPENDING                    | O(log N + M)
XCLAIM                      | O(log N)
XAUTOCLAIM                  | O(log N + M)
XTRIM MAXLEN n              | O(M) (M = 제거 메시지 수)
XDEL id                     | O(1) 논리 삭제

Pub/Sub vs Stream 비교:
  Pub/Sub PUBLISH:   O(N), N = 구독자 수 (브로드캐스트)
  Stream XADD:       O(1) (Consumer 수 무관)
  
  Pub/Sub 지연:   수십 μs
  Stream XADD:    수십~수백 μs (PEL 업데이트 포함)
  
  Stream의 오버헤드:
    PEL 업데이트: 딕셔너리 쓰기
    last-delivered-id 업데이트
    → Pub/Sub보다 약간 느리지만 처리 보장과의 트레이드오프
```

---

## ⚖️ 트레이드오프

```
Stream의 장단점:

장점:
  ① 메시지 영구 저장 (Pub/Sub과 달리 소비자 없어도 보존)
  ② Consumer Group으로 분산 처리 + 단일 전달
  ③ PEL로 처리 보장 (크래시 후 재처리)
  ④ 시간 기반 메시지 ID로 타임라인 탐색
  ⑤ XRANGE로 과거 메시지 재조회

단점:
  ① 메모리 기반 → 대용량 장기 보존에 부적합
  ② Kafka의 파티셔닝 같은 내장 병렬화 없음
  ③ PEL 관리 필요 (XACK 빠뜨리면 메모리 누수)
  ④ Exactly-Once 보장 없음 (At-Least-Once)

PEL 관리의 중요성:
  XREADGROUP 후 XACK를 빠뜨리는 코드:
  while True:
    msgs = xreadgroup(...)
    process(msgs)
    # XACK 누락!
  
  → PEL이 계속 증가
  → XPENDING으로 확인 시 수만 개 pending
  → 메모리 증가 + XAUTOCLAIM이 계속 같은 메시지를 재전달
  → 중복 처리 발생
  
  올바른 패턴:
  while True:
    msgs = xreadgroup(...)
    try:
      process(msgs)
      for msg in msgs:
        xack(stream, group, msg.id)
    except:
      # ACK 안 함 → PEL에 남아 재처리 가능
      log_error(...)
```

---

## 📌 핵심 정리

```
Redis Stream:

내부 구조:
  Radix Tree (rax) + listpack
  메시지 ID = Unix timestamp ms - sequence (단조 증가)
  같은 ms 메시지는 하나의 listpack 노드에 묶임

Consumer Group:
  last-delivered-id: 마지막으로 전달된 메시지 ID
  여러 Consumer가 같은 Group → 각 메시지 하나의 Consumer에게만 전달

PEL (Pending Entry List):
  전달됐지만 ACK 안 된 메시지 추적
  message_id → consumer_name, delivery_time, delivery_count
  XACK → PEL에서 제거
  XCLAIM/XAUTOCLAIM → 오래된 메시지 다른 Consumer로 재할당

핵심 명령어:
  XADD:         메시지 추가 O(1)
  XRANGE:       범위 조회 O(log N + M)
  XREADGROUP:   Consumer Group 읽기
  XACK:         처리 완료 확인
  XPENDING:     ACK 안 된 메시지 조회
  XAUTOCLAIM:   오래된 pending 자동 재할당

Kafka 비교:
  Stream: 메모리, 적은 메시지, 단순 운영
  Kafka: 디스크, 대용량, 장기 보존, 높은 처리량
```

---

## 🤔 생각해볼 문제

**Q1.** Consumer Group에서 `XREADGROUP GROUP workers consumer1 STREAMS mystream ">"` 대신 `XREADGROUP GROUP workers consumer1 STREAMS mystream "0-0"`을 사용하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`>`(greater than)는 "새 메시지" 즉 `last-delivered-id` 이후 메시지를 요청합니다.

`0-0`은 "ID 0 이상인 메시지"를 요청하는데, **Consumer Group에서는 PEL에 있는 메시지(이 Consumer에게 전달됐지만 ACK 안 된 것)만 반환**합니다. 새 메시지는 반환하지 않습니다.

즉:
- `>` → Stream에서 아직 전달되지 않은 새 메시지
- `0-0` (또는 특정 ID) → 이 Consumer의 PEL에서 해당 ID 이후 pending 메시지

**실용적 용도:**
```bash
# 크래시 후 재시작: 자신의 pending 메시지부터 먼저 처리
msgs = XREADGROUP GROUP workers consumer1 COUNT 10 STREAMS mystream 0-0
if msgs:
    process(msgs)
    xack(msgs)

# pending 다 처리한 후 새 메시지 처리
msgs = XREADGROUP GROUP workers consumer1 COUNT 10 STREAMS mystream >
```

이 패턴으로 재시작 시 이전에 처리 못 한 메시지를 먼저 완료할 수 있습니다.

</details>

---

**Q2.** 동일한 Consumer Group에서 `delivery_count`가 10 이상인 메시지는 어떻게 처리해야 하는가? (독소 메시지/Poison Pill 문제)

<details>
<summary>해설 보기</summary>

`delivery_count`가 높다는 것은 해당 메시지가 **반복적으로 처리 실패**하고 있다는 신호입니다 (독소 메시지).

**처리 전략:**

```python
while True:
    # pending 메시지 먼저 처리
    pending = xpending(stream, group, count=10)
    for msg in pending:
        if msg.delivery_count > 10:
            # 독소 메시지 처리
            move_to_dead_letter_queue(msg)  # DLQ로 이동
            xack(stream, group, msg.id)     # PEL에서 제거
            xdel(stream, msg.id)            # 스트림에서도 제거 (선택)
        else:
            reprocess(msg)
    
    # 새 메시지 처리
    new_msgs = xreadgroup(group, consumer, ">", count=10)
    ...
```

**DLQ(Dead Letter Queue) 패턴:**
```bash
# 실패 메시지를 별도 Stream으로 이동
XADD dead_letter '*' original_stream mystream original_id "103-0" error "..."
XACK mystream workers 103-0
```

`XAUTOCLAIM`에서 반환되는 메시지에도 `delivery_count`가 포함되므로, 재할당 전 확인:
```bash
XAUTOCLAIM mystream workers consumer2 60000 0-0 COUNT 10
# 결과에서 delivery_count > threshold인 메시지는 DLQ로
```

</details>

---

**Q3.** Redis Stream에서 Consumer Group의 `last-delivered-id`와 Kafka의 Consumer Group `offset`의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

**Redis Stream `last-delivered-id`:**
- Consumer Group 단위로 하나의 `last-delivered-id` 존재
- 메시지가 어느 Consumer에게 전달됐는지 PEL로 추적
- Consumer 수에 상관없이 Group 전체가 하나의 커서 공유
- Consumer들이 경쟁적으로 메시지를 가져감 (Competing Consumers 패턴)

**Kafka `offset`:**
- 파티션별 Consumer Group의 offset 저장
- 같은 그룹의 Consumer들이 각자 담당 파티션의 offset 독립 관리
- Consumer 수 = 파티션 수 = 이상적인 병렬 처리 단위
- 순서 보장: 같은 파티션은 순서대로 처리

**핵심 차이:**

| | Redis Stream | Kafka |
|---|---|---|
| 순서 보장 | 전체 스트림 순서 보장 | 파티션 내 순서 보장 |
| 병렬화 | Consumer 수 제한 없지만 파티셔닝 없음 | 파티션 수가 병렬화 한계 |
| offset 관리 | Redis가 last-delivered-id 자동 관리 | 커밋 주기 설정 필요 |
| 재처리 | XCLAIM으로 특정 메시지 재할당 | offset reset으로 구간 재처리 |

Redis Stream은 단순하고 빠르지만, Kafka의 파티션 기반 병렬화가 없습니다. 처리량이 매우 높은 서비스에서는 Redis Cluster로 여러 Stream 키를 만들어 파티셔닝을 직접 구현해야 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Bitmap, HyperLogLog, Geo](./05-bitmap-hyperloglog-geo.md)** | **[홈으로 🏠](../README.md)** | **[다음: 데이터 구조 선택 가이드 ➡️](./07-data-structure-guide.md)**

</div>
