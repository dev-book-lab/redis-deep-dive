# Pipeline과 MULTI/EXEC — 원자성과 네트워크 최적화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Pipeline이 RTT(Round Trip Time)를 줄이는 원리는?
- Pipeline은 원자적인가? MULTI/EXEC와 무엇이 다른가?
- `MULTI/EXEC`가 보장하는 원자성의 범위와 한계는?
- MULTI/EXEC가 롤백을 지원하지 않는 이유는?
- Lua 스크립트와 MULTI/EXEC의 차이는?
- Spring `RedisTemplate.executePipelined()`의 올바른 사용법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis 명령어 10개를 실행할 때 10번의 TCP 왕복이 발생한다. 이 RTT 비용이 Redis 처리 시간보다 훨씬 클 수 있다. Pipeline은 이를 1번의 왕복으로 줄인다. 하지만 Pipeline은 원자적이지 않아 MULTI/EXEC가 필요한 경우가 있고, MULTI/EXEC도 롤백이 없어 Lua 스크립트가 필요한 경우가 있다. 세 가지의 차이를 정확히 알아야 올바르게 선택할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: N개 명령어를 순차로 실행해서 RTT × N 낭비

  코드:
    for user_id in user_ids:  # 1,000개
        redis.hset(f"user:{user_id}", "last_seen", timestamp)
  
  결과:
    1,000번 RTT × 0.5 ms = 500 ms 낭비
    데이터 처리보다 네트워크 대기가 더 오래 걸림

실수 2: Pipeline이 원자적이라고 가정

  코드 (잘못된 의도):
    pipe = redis.pipeline()
    pipe.set("balance", 100)
    pipe.set("account_status", "active")
    pipe.execute()
    # "이 두 개가 원자적으로 실행되겠지?"
  
  현실:
    set("balance", 100) 성공
    set("account_status", "active") 실패 (예: 메모리 부족)
    → balance는 갱신됐지만 account_status는 아님 (불일치!)
    Pipeline은 원자적이지 않음

실수 3: MULTI/EXEC에서 롤백을 기대

  코드:
    multi = redis.multi()
    multi.set("a", 1)
    multi.hset("not_a_hash_key", "field", "value")  # 타입 오류
    multi.set("b", 2)
    multi.exec()
    # "오류 발생하면 a, b 설정이 취소되겠지?"
  
  현실:
    a = 1 설정됨 (성공)
    hset 오류 발생 (하지만 계속 실행)
    b = 2 설정됨 (성공)
    → 부분 실행, 롤백 없음!
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Pipeline으로 RTT 최소화 (원자성 불필요 시):
  pipe = redis.pipeline(transaction=False)  # MULTI/EXEC 없는 순수 Pipeline
  for user_id in user_ids:
      pipe.hset(f"user:{user_id}", "last_seen", timestamp)
  pipe.execute()  # 단일 TCP 왕복으로 전체 실행

MULTI/EXEC로 원자성 보장 (순서 보장 + 끊김 없는 실행):
  with redis.pipeline() as pipe:
      pipe.multi()  # MULTI 시작
      pipe.set("balance", new_balance)
      pipe.hincrby("stats", "transactions", 1)
      pipe.execute()  # EXEC (원자적 실행)
  # 두 명령어 사이에 다른 클라이언트 끼어들기 없음

Lua 스크립트 (Read-Modify-Write 원자성):
  # 잔액 확인 후 차감 (원자적 복합 연산)
  lua = """
  local balance = tonumber(redis.call('GET', KEYS[1]))
  if balance >= tonumber(ARGV[1]) then
      redis.call('DECRBY', KEYS[1], ARGV[1])
      return 1
  else
      return 0
  end
  """
  result = redis.eval(lua, 1, "balance:user:1", "100")
  # 조회와 수정이 원자적으로 보장됨
```

---

## 🔬 내부 동작 원리

### 1. Pipeline — RTT 감소의 원리

```
RTT(Round Trip Time) 없는 Pipeline:

일반 명령어 (N번 RTT):
  클라이언트:  SET a 1 ──────────────────────────────► 서버
                       ◄────────────────────────────── OK
  클라이언트:  SET b 2 ──────────────────────────────► 서버
                       ◄────────────────────────────── OK
  클라이언트:  SET c 3 ──────────────────────────────► 서버
                       ◄────────────────────────────── OK
  총 시간: 3 × RTT = 3 × 0.5ms = 1.5ms (RTT만)

Pipeline (1번 RTT):
  클라이언트:  SET a 1 │                              ► 서버
               SET b 2 │ 한 번에 전송              ► 서버
               SET c 3 │                              ► 서버
                       ◄──────────────────────────── OK
                       ◄──────────────────────────── OK
                       ◄──────────────────────────── OK
  총 시간: 1 × RTT + 처리 시간 ≈ 0.5ms

효과 계산:
  1,000개 명령어:
    일반: 1,000 × 0.5ms = 500ms
    Pipeline: 0.5ms + 처리 시간 ≈ 1~2ms
    → 250~500배 속도 향상

Pipeline의 동작 원리:
  클라이언트가 명령어들을 TCP 버퍼에 누적 후 한 번에 전송
  서버는 수신된 순서대로 실행
  응답들을 한 번에 반환

파이프라이닝 한계:
  한 번에 보내는 명령어 수가 너무 많으면:
    TCP 버퍼 초과 → 분할 전송 → 효율 감소
    서버 메모리 사용 증가 (응답 버퍼)
  권장: 1,000~10,000개 단위로 분할 Pipeline 실행
```

### 2. Pipeline vs MULTI/EXEC 원자성 비교

```
Pipeline (원자적 아님):
  명령어들이 네트워크 최적화로 묶여 전송되지만,
  각 명령어는 독립적으로 실행됨
  
  클라이언트 A (Pipeline):  SET x 1, SET y 2, SET z 3
  클라이언트 B (일반):      INCR x  ← A의 SET x와 SET y 사이에 끼어들기 가능!

  Pipeline 실행 중:
    A: SET x 1  → 완료
    B: INCR x   → x = 2  ← A와 B 사이에 실행됨
    A: SET y 2  → 완료 (x는 이미 2)
    A: SET z 3  → 완료
  
  → Pipeline 중간에 다른 클라이언트 명령어 실행 가능
  → 원자적 아님!

MULTI/EXEC (원자적 순차 실행):
  MULTI ~ EXEC 블록은 이벤트 루프에서 끊김 없이 실행
  
  클라이언트 A (MULTI/EXEC):
    MULTI
    SET x 1
    SET y 2
    SET z 3
    EXEC  ← 이 시점에 세 명령어가 연속 실행
  
  클라이언트 B: INCR x
    → A의 EXEC 전이나 후에만 실행 가능
    → A의 SET x ~ SET z 사이에 절대 끼어들기 불가

MULTI/EXEC가 보장하는 것:
  ① 실행 순서 보장: MULTI 안의 명령어들이 순서대로 실행
  ② 끊김 없는 실행: 블록 실행 중 다른 명령어 끼어들기 없음
  ③ 명령어 큐잉: MULTI 후 EXEC 전 명령어들은 즉시 실행 안 됨
  
MULTI/EXEC가 보장하지 않는 것:
  ④ 롤백 없음: 일부 명령어 실패해도 나머지는 계속 실행
  ⑤ 오류 전파 없음: 오류 발생해도 EXEC 전체 취소 안 됨
```

### 3. MULTI/EXEC 오류 처리 — 두 가지 오류 유형

```
오류 유형 1: 큐잉 시점 오류 (EXEC 전 취소)
  MULTI
  SET a 1
  WRONG_COMMAND  ← 문법 오류 (명령어 없음)
  SET b 2
  EXEC
  
  → EXEC 시 전체 취소 (이 경우는 롤백처럼 동작)
  → 문법 오류는 EXEC 전에 감지 가능

오류 유형 2: 실행 시점 오류 (부분 실행)
  MULTI
  SET a 1          ← 성공
  HSET not_hash field val  ← 타입 오류 (a는 string인데 hash로 접근)
  SET b 2          ← 성공
  EXEC
  
  결과:
    a = 1 (설정됨)
    HSET 오류 (하지만 중단 없음)
    b = 2 (설정됨)
  
  → 부분 실행, 롤백 없음!

왜 Redis는 롤백을 지원하지 않는가?
  Redis 공식 문서 설명:
  "Redis는 실행 중 실패를 지원하지 않음. 이것은 단순하고 빠르기 위한 의도적 설계"
  "타입 오류 같은 프로그래밍 오류는 개발/테스트 환경에서 잡아야 함"
  "실제 운영에서 HSET 타입 오류 등은 코드 버그 → 롤백보다 코드 수정이 맞음"
  
  롤백 없음의 장점:
    Redis 이벤트 루프를 단순하게 유지
    롤백 로직 없음 → 더 빠른 EXEC 처리
    RDBMS 롤백처럼 Undo 로그 유지 불필요
```

### 4. WATCH — 낙관적 잠금으로 트랜잭션 보호

```
문제: MULTI/EXEC 큐잉 중 다른 클라이언트가 변경
  t=0: Thread A: GET balance → 100
  t=1: Thread B: SET balance 50  ← 중간에 다른 변경!
  t=2: Thread A: MULTI
  t=3: Thread A: SET balance 90  ← 100 - 10이라고 가정하고 SET
  t=4: Thread A: EXEC
  
  결과: balance = 90 (B의 변경을 모름, 잘못된 값!)

WATCH로 해결:
  t=0: Thread A: WATCH balance     ← balance 변경 감시 시작
  t=1: Thread A: GET balance → 100
  t=2: Thread B: SET balance 50   ← balance 변경!
  t=3: Thread A: MULTI
  t=4: Thread A: SET balance 90
  t=5: Thread A: EXEC → nil (취소!)  ← WATCH 감시 중 변경됨
  
  WATCH 동작:
    감시 중인 키가 EXEC 전에 변경되면 → EXEC가 nil 반환 (트랜잭션 취소)
    Thread A는 재시도 로직 실행:
      UNWATCH → GET balance → MULTI → SET balance → EXEC

WATCH + MULTI/EXEC = 낙관적 잠금(Optimistic Locking):
  충돌이 적다고 가정 → 락 없이 시도 → 충돌 시 재시도
  
  차이:
    MULTI/EXEC만: 실행 순서 보장, 중간 끼어들기 없음
    WATCH + MULTI/EXEC: 큐잉 시작 시점의 값이 변경 없어야 EXEC 성공
```

### 5. Lua 스크립트 vs MULTI/EXEC

```
MULTI/EXEC의 한계:
  조건부 실행 불가:
    MULTI
    GET balance      ← 값을 읽어서 조건 확인 불가
    SET balance 90   ← GET 값이 100 이상일 때만 실행하고 싶지만 불가
    EXEC

Lua 스크립트:
  Redis가 Lua 인터프리터를 내장
  스크립트 전체가 단일 원자적 실행 (이벤트 루프 독점)
  조건문, 반복문, 복합 로직 가능

  예: 잔액 차감 (조건 + 실행 원자적):
  local balance = tonumber(redis.call('GET', KEYS[1]))
  if balance >= tonumber(ARGV[1]) then
      redis.call('DECRBY', KEYS[1], ARGV[1])
      return 1  -- 성공
  end
  return 0  -- 잔액 부족

MULTI/EXEC vs Lua 선택:
  단순 연속 명령어: MULTI/EXEC (더 단순)
  조건부 실행: Lua 스크립트 (유일한 방법)
  Read-Modify-Write: Lua 스크립트 (WATCH 재시도보다 단순)
  복잡한 비즈니스 로직: Lua (단, 스크립트 길어지면 이벤트 루프 독점 주의)
```

### 6. Spring RedisTemplate.executePipelined()

```java
// Pipeline 사용
List<Object> results = redisTemplate.executePipelined(
    new RedisCallback<Object>() {
        @Override
        public Object doInRedis(RedisConnection connection) {
            StringRedisConnection conn = (StringRedisConnection) connection;
            for (Long userId : userIds) {
                conn.hSet("user:" + userId, "last_seen", timestamp);
            }
            return null;  // 반드시 null 반환
        }
    }
);
// results: 각 명령어의 반환값 리스트

// MULTI/EXEC (SessionCallback 사용)
List<Object> txResults = redisTemplate.execute(
    new SessionCallback<List<Object>>() {
        @Override
        public List<Object> execute(RedisOperations operations) {
            operations.multi();  // MULTI 시작
            operations.opsForValue().set("balance", "100");
            operations.opsForHash().increment("stats", "count", 1);
            return operations.exec();  // EXEC
        }
    }
);

// WATCH + MULTI/EXEC (낙관적 잠금)
redisTemplate.execute(new SessionCallback<>() {
    @Override
    public Object execute(RedisOperations ops) {
        ops.watch("balance");  // WATCH
        String current = (String) ops.opsForValue().get("balance");
        int newBalance = Integer.parseInt(current) - 100;
        
        ops.multi();
        if (newBalance >= 0) {
            ops.opsForValue().set("balance", String.valueOf(newBalance));
        }
        List<Object> result = ops.exec();
        
        if (result == null) {
            // WATCH 충돌 → 재시도
            throw new OptimisticLockingFailureException("Retry");
        }
        return result;
    }
});
```

---

## 💻 실전 실험

### 실험 1: Pipeline vs 순차 실행 성능 비교

```bash
# 순차 실행 (1,000개 SET)
START=$(date +%s%3N)
for i in $(seq 1 1000); do
  redis-cli SET "seq:key:$i" "value-$i" > /dev/null
done
END=$(date +%s%3N)
echo "순차 실행: $((END-START))ms"

# Pipeline 실행 (1,000개 SET을 하나의 파이프로)
START=$(date +%s%3N)
{
  for i in $(seq 1 1000); do
    echo "SET pipe:key:$i value-$i"
  done
} | redis-cli --pipe
END=$(date +%s%3N)
echo "Pipeline 실행: $((END-START))ms"

# 또는 redis-cli --pipe-mode
```

### 실험 2: MULTI/EXEC 원자성 확인

```bash
# MULTI/EXEC 원자적 실행
redis-cli MULTI
redis-cli SET atomic:a 1
redis-cli SET atomic:b 2
redis-cli SET atomic:c 3
redis-cli EXEC
# 1) OK
# 2) OK
# 3) OK

# MULTI/EXEC 도중 오류 (부분 실행 확인)
redis-cli SET not_hash "string_value"
redis-cli MULTI
redis-cli SET error_test:a 1
redis-cli HSET not_hash field val  # 타입 오류!
redis-cli SET error_test:b 2
redis-cli EXEC
# 1) OK
# 2) WRONGTYPE Operation against a key holding the wrong kind of value
# 3) OK

redis-cli GET error_test:a  # 1 (설정됨)
redis-cli GET error_test:b  # 2 (설정됨)  ← 롤백 없음!
```

### 실험 3: WATCH + MULTI/EXEC 낙관적 잠금

```bash
# 잔액 100 설정
redis-cli SET balance 100

# 터미널 1: WATCH + MULTI/EXEC 시작
redis-cli WATCH balance
redis-cli MULTI
redis-cli DECRBY balance 50

# 터미널 2: WATCH 감시 중 변경 (충돌 유발)
redis-cli SET balance 200  # 다른 클라이언트가 변경

# 터미널 1 계속
redis-cli EXEC
# (nil) ← WATCH가 변경을 감지, 트랜잭션 취소!

redis-cli GET balance  # 200 (터미널 2의 변경만 적용)

# WATCH 없을 때
redis-cli UNWATCH
redis-cli MULTI
redis-cli DECRBY balance 50
redis-cli EXEC
# 1) (integer) 150  ← 정상 실행 (WATCH 없으므로)
```

### 실험 4: Lua 스크립트 원자적 Read-Modify-Write

```bash
# 잔액 차감 (원자적)
redis-cli SET balance 100

redis-cli EVAL "
local bal = tonumber(redis.call('GET', KEYS[1]))
if bal >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1
else
    return 0
end
" 1 balance 30

redis-cli GET balance  # 70

# 잔액 부족 시
redis-cli EVAL "
local bal = tonumber(redis.call('GET', KEYS[1]))
if bal >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1
else
    return 0
end
" 1 balance 100

# (integer) 0 ← 잔액 부족, 차감 안 됨
redis-cli GET balance  # 70 (변경 없음)
```

---

## 📊 성능/비용 비교

```
명령어 실행 방식별 성능:

방식           | RTT        | 원자성      | 롤백      | 조건부 실행
──────────────┼────────────┼───────────┼──────────┼────────────
순차 실행       | N × RTT    | 없음       | 없음      | 가능 (응답 보고)
Pipeline      | 1 RTT      | 없음       | 없음      | 불가
MULTI/EXEC    | 2 RTT      | 순서 보장   | 없음      | 불가
(MULTI+EXEC)  |(MULTI+EXEC)|           |          |
WATCH+MULTI   | 3+ RTT     | 낙관적 잠금  | 취소 가능  | 불가
Lua 스크립트    | 1 RTT      | 완전 원자성  | 없음      | 가능

1,000개 명령어 성능 (RTT = 0.5ms):

순차 실행: 1,000 × 0.5ms = 500ms
Pipeline:  1 × 0.5ms = 0.5ms (1,000배 빠름)
MULTI/EXEC: 2 × 0.5ms = 1ms (블록 내 명령어 수 무관)

Pipeline 배치 크기 권장:
  너무 작음 (10개): Pipeline 효율 낮음
  적정 (100~1,000개): 효율 최대, 메모리 부담 없음
  너무 많음 (100,000개): Redis 응답 버퍼 과부하
  → 1,000개 단위로 분할 Pipeline 권장
```

---

## ⚖️ 트레이드오프

```
Pipeline:
  장점: RTT 최소화, 구현 단순
  단점: 원자성 없음, 명령어 간 의존성 처리 불가

MULTI/EXEC:
  장점: 원자적 순차 실행 (끊김 없음)
  단점: 롤백 없음, 조건부 실행 불가, 1개 실패해도 나머지 실행

Lua 스크립트:
  장점: 완전 원자성, 조건부 실행 가능, Read-Modify-Write
  단점: 스크립트 복잡도 증가, 이벤트 루프 독점 (긴 스크립트 주의)
        스크립트 디버깅 어려움

WATCH + MULTI/EXEC:
  장점: 낙관적 잠금으로 충돌 감지
  단점: 충돌 시 재시도 필요, 고경합 상황에서 재시도 폭발

선택 기준:
  단순 배치 쓰기/읽기:        Pipeline (원자성 불필요)
  연속 실행 보장 필요:        MULTI/EXEC
  조건부 복합 연산:           Lua 스크립트
  값 읽고 조건 확인 후 수정:  Lua 스크립트 (WATCH보다 단순)
  충돌 드문 값 수정:          WATCH + MULTI/EXEC
```

---

## 📌 핵심 정리

```
Pipeline, MULTI/EXEC, Lua 스크립트 핵심:

Pipeline:
  N개 명령어를 1번 RTT로 처리
  원자적 아님 (다른 클라이언트 끼어들기 가능)
  적합: 독립적인 명령어 대량 실행 (배치 업데이트)

MULTI/EXEC:
  MULTI ~ EXEC 블록이 끊김 없이 순차 실행
  롤백 없음 (실패해도 나머지 계속)
  큐잉 시 문법 오류 → 전체 취소
  실행 시 오류 → 해당 명령어만 오류, 나머지 성공
  적합: 여러 키 동시 업데이트, 순서 보장 필요

WATCH + MULTI/EXEC:
  WATCH key로 감시 → 변경 시 EXEC 취소 (nil 반환)
  낙관적 잠금 패턴 (충돌 적을 때 효율적)
  적합: 읽고 조건 확인 후 수정 (고경합 아닌 경우)

Lua 스크립트:
  단일 원자적 실행 (Read+Modify+Write 보장)
  조건부 로직 가능
  적합: Read-Modify-Write, 복합 원자적 연산

Spring:
  executePipelined() → Pipeline
  SessionCallback + multi()/exec() → MULTI/EXEC
  execute(new DefaultRedisScript()) → Lua
```

---

## 🤔 생각해볼 문제

**Q1.** Pipeline으로 1,000개의 HSET을 실행하던 중 500번째에서 오류가 발생했다. 이 경우 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

Pipeline은 원자적이지 않으므로:
- 1~499번: 성공
- 500번: 오류 발생
- 501~1,000번: **계속 실행** (오류와 무관)

`pipe.execute()`의 반환값은 각 명령어의 응답 리스트다:
```python
results = pipe.execute(raise_on_error=False)
# [OK, OK, ..., ResponseError, OK, OK, ...]
# 500번째에만 오류, 나머지는 OK
```

**실무 처리 패턴:**
```python
results = pipe.execute(raise_on_error=False)
failed = []
for i, (key, result) in enumerate(zip(keys, results)):
    if isinstance(result, Exception):
        failed.append(key)  # 실패한 키 수집

if failed:
    # 실패한 키만 재시도
    retry_pipe = redis.pipeline()
    for key in failed:
        retry_pipe.hset(key, ...)
    retry_pipe.execute()
```

**롤백이 필요하다면:** MULTI/EXEC를 사용해야 하지만, MULTI/EXEC도 실행 시점 오류는 롤백하지 않는다. 완전한 롤백이 필요하면 Lua 스크립트에서 오류 처리를 직접 구현하거나, 멱등성을 보장하고 전체를 재시도하는 방식을 사용한다.

</details>

---

**Q2.** MULTI/EXEC 블록 안에서 `GET key`로 값을 읽어 조건에 따라 다른 명령어를 실행하고 싶다. 이것이 불가능한 이유와 대안은?

<details>
<summary>해설 보기</summary>

**불가능한 이유:**
MULTI/EXEC에서 명령어들은 큐에 저장됐다가 EXEC 시 실행된다. MULTI ~ EXEC 사이의 `GET` 명령어는 실제로 실행되지 않고 큐에만 들어간다. 클라이언트 코드는 EXEC 전에 GET의 결과를 알 수 없다.

```python
pipe.multi()
pipe.get("balance")     # ← 실제 실행 안 됨, 큐에만 들어감
# balance_value가 여기서 얻을 수 없음!
if balance_value >= 100:  # ← 불가
    pipe.decrby("balance", 100)
pipe.exec()
```

**대안 1: Lua 스크립트 (가장 권장)**
```lua
local balance = tonumber(redis.call('GET', KEYS[1]))
if balance >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1
end
return 0
```

**대안 2: WATCH + MULTI/EXEC**
```python
while True:
    redis.watch("balance")
    balance = int(redis.get("balance"))
    if balance < 100:
        redis.unwatch()
        return False
    
    pipe = redis.pipeline(True)
    try:
        pipe.multi()
        pipe.decrby("balance", 100)
        result = pipe.execute()
        return True
    except WatchError:
        continue  # 재시도
```

대부분의 경우 Lua 스크립트가 더 단순하고 효율적이다.

</details>

---

**Q3.** Lua 스크립트가 이벤트 루프를 독점하는 것이 위험한 이유와, 긴 스크립트를 어떻게 분할해야 하는가?

<details>
<summary>해설 보기</summary>

**이벤트 루프 독점의 위험:**
Lua 스크립트는 Redis 이벤트 루프에서 중단 없이 실행된다. 스크립트가 실행되는 동안 다른 모든 클라이언트의 요청이 대기한다.

예를 들어:
```lua
-- 위험한 스크립트: O(N) 루프
for i = 1, 100000 do
    redis.call('INCR', 'counter')
end
```
이 스크립트가 100ms 걸린다면 → 100ms 동안 Redis가 응답 없음 → 타임아웃 폭발.

**권장 Lua 스크립트 지침:**
1. **스크립트는 짧게 (< 1ms 목표):** 복잡한 루프, 대량 KEYS 처리 금지
2. **O(N) 연산 금지:** SCAN, SORT, SUNION 등 대용량 처리 스크립트 금지
3. **lua-time-limit 설정 (기본 5000ms):**
   ```conf
   lua-time-limit 5000  # 5초 초과 시 다른 클라이언트가 SCRIPT KILL 가능
   ```

**긴 로직 분할 방법:**
```python
# 잘못된 방법: 단일 Lua로 10,000개 처리
redis.eval("""
for i, key in ipairs(KEYS) do
    redis.call('DEL', key)
end
""", len(keys), *keys)  # keys = 10,000개 → 오래 걸림

# 올바른 방법: Python에서 배치 분할
batch_size = 100
for i in range(0, len(keys), batch_size):
    batch = keys[i:i+batch_size]
    pipe = redis.pipeline()
    for key in batch:
        pipe.delete(key)
    pipe.execute()  # Pipeline으로 배치 처리
```

원자성이 반드시 필요한 연산만 Lua로, 대량 처리는 Pipeline으로 분리하는 것이 핵심이다.

</details>

---

<div align="center">

**[⬅️ 이전: Pub/Sub vs Stream](./05-pubsub-vs-stream.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — 성능과 운영 ➡️](../performance-operations/01-slowlog-bottleneck.md)**

</div>
