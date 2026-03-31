# 복제 일관성 트레이드오프 — WAIT 명령어와 데이터 유실 범위

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Redis가 동기 복제를 지원하지 않는 이유는?
- `WAIT numreplicas timeout`은 내부적으로 어떻게 동작하는가?
- `min-replicas-to-write`와 `min-replicas-max-lag`를 조합하면 어떤 보장이 생기는가?
- Failover 시 데이터 유실 범위를 어떻게 계산하는가?
- 최종 일관성(Eventual Consistency)을 허용하면서도 유실을 최소화하는 설계 방법은?
- Redis Cluster의 각 Shard에서 복제 일관성은 어떻게 적용되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis의 복제는 기본적으로 비동기다. 이것은 성능을 위한 의도적 선택이다. 하지만 "비동기 복제 = 데이터 유실 가능"이라는 사실을 설계 단계에서 인식하지 못하면, Failover 후 데이터가 사라지는 상황에 당황하게 된다. WAIT와 min-replicas 설정은 이 유실을 제어하는 도구지만, 각각 가용성을 일부 포기하는 트레이드오프가 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: "Master에서 OK 받았으니 Replica에도 있다"고 가정

  코드:
    redis.set("order:123", "paid")  # Master에서 OK 반환
    # 즉시 Failover 발생 (극단적 시나리오)
    result = redis.get("order:123")  # 새 Master(구 Replica)에서 조회
    # → nil 반환 가능! (비동기 복제 지연으로 미전파)

  실무 발생 조건:
    Master 장애 + 비동기 복제 지연이 겹치면
    → 최근 수십~수백 ms 이내 쓰기가 손실 가능
    → 결제, 주문 등 중요 데이터에서 발생 시 심각한 문제

실수 2: WAIT를 모든 쓰기에 적용해서 성능 저하

  코드:
    for order in orders:
        redis.set(f"order:{order.id}", order.status)
        redis.wait(1, 1000)  # 매 쓰기마다 Replica 복제 확인!

  결과:
    WAIT의 대기 시간 ~1~5 ms × 초당 10,000건 = +10~50초 추가 대기
    처리량 급락

  올바른 접근:
    결제 완료처럼 "반드시 보존되어야 하는 쓰기"에만 WAIT 사용
    일반 캐시/세션 쓰기에는 불필요

실수 3: min-replicas-to-write 설정 후 Replica 장애 시 쓰기 불가

  설정: min-replicas-to-write 1
  상황: Replica 장애 (네트워크 단절)
  결과: Master 쓰기 불가 (Replica 없어서)
        서비스 쓰기 장애!
  
  → 가용성(Availability)을 일관성(Consistency)을 위해 희생한 것
  → 서비스 요구사항에 맞는 트레이드오프 선택 필요
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
데이터 중요도에 따른 전략:

  캐시/세션 (일부 유실 허용):
    기본 비동기 복제 사용
    Failover 시 손실 = 최근 수 ms~수백 ms 쓰기
    → 허용 가능, 별도 설정 불필요

  중요 비즈니스 데이터 (유실 최소화):
    중요 쓰기에만 WAIT 추가:
    redis.set("order:123", "paid")
    acks = redis.wait(1, 500)  # 500ms 내 1개 Replica 복제 확인
    if acks == 0:
        # 복제 미확인 → 재처리 큐에 추가 or 알림

  쓰기 거부로 일관성 보장 (가용성 희생 허용):
    min-replicas-to-write 1
    min-replicas-max-lag 10
    → Replica 없거나 10초 이상 지연이면 쓰기 거부
    → 데이터 안전 but 가용성 저하

유실 범위 계산:
  Failover 직전 master_repl_offset - max_slave_repl_offset
  = 손실된 명령어 바이트 수
  
  redis-cli INFO replication | grep -E "master_repl_offset|offset="
  → 차이가 작을수록 유실 범위 작음
```

---

## 🔬 내부 동작 원리

### 1. 비동기 복제의 구조 — 동기를 지원 않는 이유

```
Redis 복제 타임라인:

  t=0:  클라이언트 → Master: SET order:123 "paid"
  t=0:  Master: 메모리에 저장
  t=0:  Master → 클라이언트: OK  ← 즉시 반환!
  t=0:  Master: replication buffer에 명령어 추가
  t=1ms: Master → Replica TCP 스트림: SET order:123 "paid"
  t=2ms: Replica: 명령어 수신 및 적용
  t=3ms: Replica: ACK (offset 업데이트)

동기 복제를 하지 않는 이유:
  동기 복제 = "Replica ACK 받을 때까지 클라이언트에게 OK 안 보냄"
  
  Replica와의 RTT: ~0.5~5 ms (데이터센터 내부)
  Replica 2개면: 두 ACK 모두 대기 → 지연 배가
  Redis 목표 응답 시간: ~0.1~1 ms
  
  동기 복제 시: 응답 시간 최소 10배 이상 증가
  → "고성능 인메모리 DB"의 핵심 가치 상실
  
  선택: 성능 > 강한 일관성 (Redis의 철학)

비동기 복제의 허용:
  "Replica ACK 없어도 OK 반환"
  → 데이터가 Replica에 있다는 보장 없음
  → Failover 시 최근 쓰기 유실 가능
  → 서비스 요구사항에 맞게 WAIT로 보완
```

### 2. WAIT 명령어 — 특정 쓰기의 복제 확인

```c
/* replication.c */
/* WAIT numreplicas timeout */

WAIT 동작 원리:
  1. Master가 현재 replication offset(X) 기록
  2. Replica들에게 REPLCONF GETACK 전송 (각 Replica의 현재 offset 확인)
  3. timeout까지 대기하면서 각 Replica의 ACK 수신
  4. offset ≥ X인 Replica 수를 반환

예시:
  Master offset: 100,000
  Replica A offset: 99,990 (10 bytes 뒤처짐)
  Replica B offset: 100,000 (동기화됨)
  
  WAIT 2 1000 실행:
    Master가 현재 offset 100,000 기록
    Replica A, B에게 REPLCONF GETACK 전송
    Replica B: 즉시 ACK (offset 100,000 ≥ 100,000) → count=1
    Replica A: ~수 ms 후 ACK (offset 100,000에 도달) → count=2
    → WAIT 반환값: 2 (2개 Replica 모두 현재까지 복제됨)

WAIT의 보장:
  WAIT 1 1000이 1을 반환 → 해당 시점까지의 모든 쓰기가 최소 1개 Replica에 있음
  → Failover 시 이 Replica가 선택되면 해당 쓰기는 보존됨
  
  WAIT 1 1000이 0을 반환:
    → 1초 동안 어떤 Replica도 현재 offset에 도달 못함
    → 타임아웃 시나리오 (Replica 장애 or 심한 지연)
    → 이 쓰기의 복제 보장 안 됨

WAIT의 성능 특성:
  WAIT 자체: O(1) + 복제 지연 대기
  응답 시간: 복제 지연 시간 + RTT (최소 1~5 ms, 최악 timeout)
  
  최적 사용 패턴:
    중요한 여러 쓰기 → 일괄 실행 → 마지막에 WAIT 한 번
    (매 쓰기마다 WAIT 금지 — 처리량 급락)
```

### 3. min-replicas-to-write — 쓰기 거부로 일관성 보장

```
설정:
  min-replicas-to-write N   # 최소 N개 Replica가 연결 + 응답 중이어야 쓰기 허용
  min-replicas-max-lag  T   # Replica의 lag이 T초 이하여야 카운트

동작:
  현재 연결된 Replica 수 < N → 쓰기 명령어에 즉시 에러 반환
  또는 Replica lag > T초 → 해당 Replica는 카운트 안 함

예시:
  min-replicas-to-write 1
  min-replicas-max-lag 10
  
  정상 상태:
    Replica 1개 연결, lag 2초 → 카운트 1 ≥ 1 → 쓰기 허용
  
  Replica 장애:
    Replica 0개 연결 → 카운트 0 < 1 → 쓰기 거부
    에러: NOREPLICAS Not enough good slaves to write
  
  Replica 지연:
    Replica 1개 연결, lag 15초 > 10초 → 카운트 0 < 1 → 쓰기 거부

Split-Brain 완화 역할:
  네트워크 파티션 → 구 Master의 Replica가 새 Master에 연결
  → 구 Master의 Replica 0개 → 쓰기 거부
  → 구 Master에 더 이상 쓰이지 않음
  → 파티션 해소 후 데이터 충돌 없음

트레이드오프:
  일관성: Replica 없으면 쓰기 안 됨 → 데이터 유실 방지
  가용성: Replica 장애 시 Master도 쓰기 불가 → 서비스 중단 가능
  → 가용성보다 일관성이 중요한 서비스에 적합 (결제, 주문 등)
```

### 4. 데이터 유실 범위 계산

```
Failover 시 유실 범위 계산 방법:

  INFO replication에서:
    master_repl_offset: 12,345,678   ← Master 현재 offset
    slave0:...offset=12,345,670      ← Replica A offset (8 bytes 뒤)
    slave1:...offset=12,345,650      ← Replica B offset (28 bytes 뒤)
  
  최악의 경우:
    Failover 시 Replica B(offset 낮은 것)가 새 Master로 선택되면
    12,345,678 - 12,345,650 = 28 bytes의 명령어가 유실
  
  최선의 경우:
    Failover 시 Replica A(offset 높은 것)가 새 Master로 선택
    12,345,678 - 12,345,670 = 8 bytes의 명령어만 유실
  
  Sentinel/Cluster: 가장 높은 offset Replica를 새 Master로 선택
  → 자동으로 유실 최소화

유실 범위를 "시간"으로 환산:
  쓰기 속도 1 MB/초, 28 bytes 유실
  → 유실 시간 ≈ 28 / 1,000,000 = 0.000028초 = 28 μs
  
  쓰기 속도 1 MB/초, lag이 100 ms라면:
  유실 데이터 ≈ 1 MB/초 × 0.1초 = 100 KB = 수천 건 명령어 가능

실시간 유실 위험도 모니터링:
  # Replica lag 확인
  redis-cli INFO replication | grep -E "slave.*lag|master_repl_offset"
  
  # lag 알림 임계값 설정 (Prometheus + Alertmanager 예시)
  # redis_replication_lag > 5 → Warning
  # redis_replication_lag > 30 → Critical
```

### 5. Redis Cluster에서의 일관성

```
Cluster 구성 (각 Shard = Master + Replica):

  Shard 1: Master1 (슬롯 0~5460) + Replica1
  Shard 2: Master2 (슬롯 5461~10922) + Replica2
  Shard 3: Master3 (슬롯 10923~16383) + Replica3

각 Shard 내부는 단순 Master-Replica 복제와 동일:
  비동기 복제 → 각 Shard에서 유실 가능
  WAIT는 각 Shard의 Replica에 대해 동작
  min-replicas-to-write는 각 Shard 설정 가능

Cluster Failover 절차 (Sentinel 불필요, 내장):
  Master1 장애 → 과반수 노드가 pfail/fail 판정
  → Replica1이 자동으로 Master로 승격
  → 유실 범위: Master1 offset - Replica1 offset

클러스터 전체 일관성 한계:
  각 Shard가 독립적으로 복제 → 동일 트랜잭션 불가 (멀티 Shard)
  → 단일 Shard 내 MULTI/EXEC만 원자적
  → 분산 트랜잭션 필요 시: Lua 스크립트 + hash-tag로 단일 Shard 처리
     또는 애플리케이션 레벨 보상 트랜잭션(Saga 패턴) 고려
```

---

## 💻 실전 실험

### 실험 1: WAIT 동작 및 성능 측정

```bash
# WAIT 기본 동작
redis-cli -p 6379 SET important:key "value"
redis-cli -p 6379 WAIT 1 1000
# 반환: 1 (1개 Replica가 복제 확인)

# Replica 없을 때 WAIT
redis-cli -p 6379 CONFIG SET min-replicas-to-write 0  # 테스트용
docker stop redis-replica  # Replica 중단

redis-cli -p 6379 SET test:key "value"
redis-cli -p 6379 WAIT 1 500  # 500ms 대기
# 반환: 0 (Replica 없음, timeout)

# WAIT 성능 영향 측정
echo "=== WAIT 없는 SET 처리량 ===" && redis-benchmark -p 6379 -t set -n 100000 | grep "requests per second"

# WAIT 포함 시 처리량 (Lua 스크립트로 SET+WAIT 원자적 실행)
cat > /tmp/setwait.lua << 'EOF'
redis.call('SET', KEYS[1], ARGV[1])
return redis.call('WAIT', 1, 100)
EOF

echo "=== WAIT 포함 SET 처리량 ===" && redis-benchmark -p 6379 --eval /tmp/setwait.lua key , value -n 10000 | grep "requests per second"
```

### 실험 2: min-replicas-to-write 동작 확인

```bash
# Replica 연결 상태에서 설정
redis-cli -p 6379 CONFIG SET min-replicas-to-write 1
redis-cli -p 6379 CONFIG SET min-replicas-max-lag 10

# 정상 상태: 쓰기 허용
redis-cli -p 6379 SET test "ok"  # OK

# Replica 중단
docker stop redis-replica

# 약 10초 후 쓰기 거부 확인
sleep 12
redis-cli -p 6379 SET test "fail"
# (error) NOREPLICAS Not enough good slaves to write

# INFO replication 확인
redis-cli -p 6379 INFO replication | grep -E "connected_slaves|min_slaves"

# 설정 복원
redis-cli -p 6379 CONFIG SET min-replicas-to-write 0
docker start redis-replica
```

### 실험 3: 복제 lag 모니터링

```bash
# 실시간 복제 lag 모니터링
while true; do
  MASTER_OFF=$(redis-cli -p 6379 INFO replication | grep "^master_repl_offset" | awk -F: '{print $2}' | tr -d ' \r')
  SLAVE_OFF=$(redis-cli -p 6379 INFO replication | grep "slave0:.*offset" | grep -oP "offset=\K[0-9]+")
  LAG_BYTES=$((MASTER_OFF - SLAVE_OFF))
  SLAVE_LAG=$(redis-cli -p 6379 INFO replication | grep "slave0:.*lag" | grep -oP "lag=\K[0-9]+")
  echo "$(date +%T) | Master: $MASTER_OFF | Slave: $SLAVE_OFF | Lag bytes: $LAG_BYTES | Lag sec: $SLAVE_LAG"
  sleep 1
done &

# 대량 쓰기로 lag 발생
for i in $(seq 1 100000); do
  redis-cli -p 6379 SET "lag:test:$i" "value-$i" > /dev/null
done
```

### 실험 4: WAIT를 활용한 중요 쓰기 패턴

```bash
# 일괄 쓰기 + 마지막에 WAIT (효율적 패턴)
redis-cli -p 6379 MULTI
redis-cli -p 6379 SET order:1 "paid"
redis-cli -p 6379 SET order:2 "paid"
redis-cli -p 6379 SET order:3 "paid"
redis-cli -p 6379 EXEC

# 일괄 쓰기 완료 후 WAIT 한 번
ACKS=$(redis-cli -p 6379 WAIT 1 1000)
echo "복제된 Replica 수: $ACKS"

if [ "$ACKS" -ge 1 ]; then
  echo "복제 성공 — 데이터 안전"
else
  echo "복제 미확인 — 재처리 큐에 추가 필요"
fi
```

---

## 📊 성능/비용 비교

```
복제 일관성 설정별 특성:

설정                       | 데이터 유실 범위 | 쓰기 성능  | 가용성
──────────────────────────┼──────────────┼──────────┼────────────
기본 비동기 복제              | Failover 시   | 최대      | 최고
                          | 수 ms~수백 ms  |          |
WAIT 1 1000 (선택적 적용)    | ~1초 이내      | 중간      | 높음 (WAIT 타임아웃 포함)
min-replicas-to-write 1   | 거의 없음       | 기본과 동일| 낮음 (Replica 없으면 쓰기 불가)
WAIT 1 0 + min-replicas 1 | 없음           | 가장 낮음  | 매우 낮음

WAIT 응답 시간 영향:
  Replica lag 1 ms: WAIT 응답 시간 +1~3 ms
  Replica lag 10 ms: WAIT 응답 시간 +10~15 ms
  Replica 장애: WAIT 응답 시간 = timeout (1000 ms)
  → 중요 쓰기에만 WAIT 사용 권장

min-replicas-max-lag 효과:
  lag 10초 설정:
    lag ≤ 10초인 Replica만 카운트
    10초 이상 지연 Replica는 "없는 것"으로 취급
    → Replica가 있어도 지연이 심하면 쓰기 거부 가능
```

---

## ⚖️ 트레이드오프

```
CAP 이론 관점:
  Redis 기본: AP (Availability + Partition Tolerance)
    네트워크 파티션 시 가용성 유지, 일관성 희생
    → 데이터 유실 가능하지만 서비스 지속

  min-replicas-to-write 설정 시: CP 쪽으로 이동
    Replica 없으면 쓰기 거부 → 가용성 희생, 일관성 보장
    → 데이터 유실 없음, 하지만 쓰기 장애 가능

  "Redis를 Strong Consistency 스토리지로 쓸 수 있는가?"
  → 가능하지만: WAIT + min-replicas 설정 필요
                응답 시간 증가 + 가용성 저하 감수
  → 권장: 강한 일관성이 필요하면 MySQL/PostgreSQL (RDBMS) 우선 고려
          Redis는 캐시, 세션, 빠른 조회 레이어에 사용

WAIT의 "보장" 한계:
  WAIT 1 1000이 1을 반환해도:
    Replica에 데이터 있음은 보장
    But Failover 시 이 Replica가 새 Master로 선택된다는 보장은 없음
    → 다른 Replica(더 낮은 offset)가 선택되면 여전히 유실 가능
  완벽한 보장 = WAIT + min-replicas-to-write 조합
```

---

## 📌 핵심 정리

```
복제 일관성 핵심:

비동기 복제 (기본):
  성능 최우선 → 클라이언트에게 즉시 OK
  Failover 시 최근 쓰기 소수 유실 가능
  유실 범위: master_offset - slave_offset bytes

WAIT numreplicas timeout:
  특정 시점까지의 쓰기가 N개 Replica에 복제됐는지 확인
  반환값: 실제로 복제 확인된 Replica 수
  0 반환: timeout 내 복제 미확인 (Replica 없거나 지연)
  중요한 쓰기에만 선택적 사용 (매 쓰기마다 사용 금지)

min-replicas-to-write N:
min-replicas-max-lag T:
  N개 Replica가 T초 이하 lag으로 연결돼야 쓰기 허용
  Split-Brain 방지 + 유실 감소
  단, Replica 부족/지연 시 쓰기 거부 (가용성 희생)

데이터 유실 모니터링:
  INFO replication | grep offset → lag bytes 계산
  Sentinel/Cluster: 최고 offset Replica를 새 Master로 선택
  
선택 기준:
  캐시/세션: 기본 비동기 (유실 허용)
  중요 비즈니스 데이터: WAIT 선택적 사용
  절대 유실 불가: min-replicas + WAIT (가용성 일부 포기)
```

---

## 🤔 생각해볼 문제

**Q1.** `WAIT 2 5000`을 실행했더니 1을 반환했다. 이것은 어떤 의미이며, 데이터가 안전한가?

<details>
<summary>해설 보기</summary>

`WAIT 2 5000`의 반환값 1은: "5초(5000ms) 안에 2개 Replica 중 **1개만** 현재 시점까지 복제를 완료했다"는 의미다.

**상황:**
- Replica A: 복제 완료 (offset = Master offset)
- Replica B: 5초 안에 복제 미완료 (지연 또는 장애)

**데이터 안전성:**
- Replica A에는 데이터가 있다 → Failover 시 **Replica A가 새 Master로 선택되면** 데이터 보존
- 하지만 Failover 시 Replica B가 선택될 수도 있다 → 데이터 유실 가능

**완전한 보장을 원한다면:**
```python
acks = redis.wait(2, 5000)
if acks < 2:
    # Replica 2개 모두 복제 확인 못함 → 재처리 큐에 추가
    retry_queue.add(current_operation)
    raise ReplicationWarning(f"Only {acks}/2 replicas confirmed")
```

Sentinel이 "가장 높은 offset을 가진 Replica"를 우선 선택하므로, Replica A(복제 완료)가 새 Master가 될 가능성이 높다. 하지만 100% 보장은 아니다.

</details>

---

**Q2.** `min-replicas-to-write 1`을 설정했는데, Replica가 1개뿐인 환경에서 Replica가 재배포를 위해 1분간 재시작됐다. 이 시간 동안 발생하는 쓰기는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`min-replicas-to-write 1`이 설정된 상태에서 Replica가 연결 해제되면:
- `connected_slaves: 0` → min-replicas 조건 미달
- Master의 모든 쓰기 명령어에 `NOREPLICAS Not enough good slaves to write` 오류 반환

**1분간 모든 쓰기 실패.**

**해결 방법:**

1. **재배포 전 min-replicas-to-write 임시 해제:**
   ```bash
   redis-cli CONFIG SET min-replicas-to-write 0  # 재배포 전
   # Replica 재배포 완료 후
   redis-cli CONFIG SET min-replicas-to-write 1  # 복구
   ```

2. **Replica 2개 + min-replicas-to-write 1:**
   Replica 1개 재시작 중에도 남은 Replica 1개가 조건 충족 → 쓰기 계속 가능
   단, 1개 Replica로 운영 중인 짧은 시간 동안 Failover 시 유실 가능성 증가

3. **Rolling restart (점진적 재시작):**
   Replica 재시작 완료 확인 후 다음 재시작 → 항상 min-replicas 조건 유지

이것이 `min-replicas-to-write`를 설정할 때 Replica 개수와 배포 절차를 함께 설계해야 하는 이유다.

</details>

---

**Q3.** Redis Cluster에서 Shard A의 Master가 장애났고 `master_repl_offset = 1,000,000`, Replica A1의 `offset = 999,980`, Replica A2의 `offset = 999,950`이다. Sentinel 없이 Cluster 자체 Failover 시 어느 Replica가 선택되고, 유실되는 데이터는 얼마인가?

<details>
<summary>해설 보기</summary>

**선택되는 Replica: Replica A1 (offset 999,980)**

Redis Cluster 자체 Failover는 Sentinel의 Failover 알고리즘과 유사하게, **replication offset이 가장 높은 Replica를 우선 선택**한다. A1(999,980) > A2(999,950)이므로 A1이 선택된다.

**유실되는 데이터:**
```
Master offset:     1,000,000
Replica A1 offset:   999,980
──────────────────────────
유실 bytes:               20 bytes
```

20 bytes의 명령어가 유실된다. 이것이 `SET order:123 "paid"` 하나(~30 bytes)보다 작을 수도 있으므로 실제로는 0~1건의 명령어가 손실될 수 있다.

**Replica A2의 운명:**
A1이 새 Master가 되면 A2는 A1을 따르는 Replica로 재구성된다. A2의 offset(999,950)보다 A1의 현재 offset(999,980)이 높으므로, A2는 A1에서 Partial Resync로 20 bytes + 이후 데이터를 받아 동기화한다.

**모니터링으로 사전 확인:**
```bash
redis-cli -p 7001 INFO replication
# master_repl_offset:1000000
# slave0:...,offset=999980,lag=0
# slave1:...,offset=999950,lag=0
```

</details>

---

<div align="center">

**[⬅️ 이전: 클러스터 리샤딩](./04-cluster-resharding.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — 캐싱 패턴 ➡️](../caching-patterns/01-caching-strategies.md)**

</div>
