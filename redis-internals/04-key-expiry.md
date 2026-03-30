# 키 만료 메커니즘 — Lazy vs Active 만료

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `EXPIRE key 60`을 설정하면 60초 후 정확히 삭제되는가?
- Lazy 만료와 Active 만료는 어떻게 다르고, 각각 언제 실행되는가?
- TTL이 지난 키가 메모리에 남아있을 수 있는 상황은 언제 발생하는가?
- `hz` 설정이 만료 처리 속도에 어떤 영향을 미치는가?
- Replica에서 만료 키가 처리되는 방식은 Master와 어떻게 다른가?
- AOF/RDB에 만료된 키가 기록되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`EXPIRE key 60`을 설정했다고 60초 후 즉시 메모리가 회수되지는 않는다. Redis의 만료는 Lazy와 Active 두 경로로 처리되며, 각각 한계가 있다. 이 메커니즘을 모르면 메모리 부족과 Replica 데이터 불일치를 설명할 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: "TTL = 삭제 시각" 으로 오해

  설계:
    100만 개 캐시 키에 TTL 60초 설정
    → "1분 후 전부 사라져서 메모리가 확보될 것"으로 기대
    → 실제: 1분 후에도 used_memory 거의 그대로

  원인 모를 때:
    → "Redis 메모리 누수인가?" → 이슈 제기
    → 인스턴스 재시작으로 임시 해결 반복

  실제 원인:
    Active 만료는 샘플링 방식 (초당 hz × 20개)
    100만 개를 모두 처리하려면 수 분~수 시간 필요
    접근 없는 콜드 키는 Lazy 만료도 안 됨

실수 2: Keyspace Notification을 신뢰할 수 있는 이벤트 트리거로 사용

  코드:
    redis.subscribe("__keyevent@0__:expired")
    → "키 만료 시 후처리 로직 실행" (주문 상태 업데이트 등)

  문제:
    Pub/Sub 방식 → 구독자 없을 때 이벤트 유실
    서버 재배포 중 (연결 끊김) → 만료 이벤트 모두 소실
    → 처리 안 된 주문 상태가 다수 발생

실수 3: Replica에서 만료 키를 신뢰

  "Master에서 EXPIRE를 설정하면 Replica도 자동으로 처리하겠지"
  → Replica에서 직접 읽을 때 만료된 값이 반환되는 경우 발생
  → Master가 아직 만료를 처리하지 않았기 때문
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
메모리 회수 보장이 필요하다면:
  hz 증가: CONFIG SET hz 50  (CPU 증가 감수)
  dynamic-hz yes (기본): 연결 수에 따라 자동 조정
  maxmemory + allkeys-lru 조합: 만료 전에도 공간 확보 가능

TTL 대량 설정 시 "Thundering Expiry" 방지:
  동일 TTL → 동시에 수십만 키 만료 → Active 만료 부하 집중
  해결: TTL에 랜덤 지터 추가
    ttl = 3600 + random.randint(0, 300)  # ±5분 분산

Keyspace Notification 대신 Stream 패턴:
  신뢰성 필요 → Stream에 만료 예정 이벤트를 미리 기록
  Consumer Group으로 처리 + ACK → 재처리 보장
  (Pub/Sub보다 훨씬 안전)

Replica 읽기 시 TTL 직접 확인:
  val = redis.get(key)
  if val and redis.ttl(key) > 0:
      return val
  # 또는 Redis 3.2+: Replica가 만료 키를 nil 반환 (설정 확인)

만료 모니터링:
  INFO keyspace → avg_ttl, expires 키 수 확인
  INFO stats | grep expired_keys → 초당 만료 처리 속도 계산
```

---

## 🔬 내부 동작 원리

### 1. 만료 정보 저장 방식

```
Redis는 만료 정보를 별도 dict(dictionary)에 저장:

  db->dict      : 모든 키-값 저장
  db->expires   : 만료 시각(Unix timestamp, 밀리초)이 있는 키만 저장

예시:
  SET user:1 "Alice"             → dict["user:1"] = "Alice"
  EXPIRE user:1 60               → expires["user:1"] = now_ms + 60,000

  db->dict:                db->expires:
  ┌────────────────────┐   ┌─────────────────────────────────┐
  │ "user:1" → "Alice" │   │ "user:1" → 1711234567890 (ms)   │
  │ "user:2" → "Bob"   │   │                                 │
  │ "session:x" → ...  │   │ "session:x" → 1711234500000     │
  └────────────────────┘   └─────────────────────────────────┘
  (모든 키)                (TTL 있는 키만)

만료 시각 저장 명령어:
  EXPIRE key seconds       → 현재 시각 + seconds (초 단위)
  PEXPIRE key milliseconds → 현재 시각 + ms (밀리초 단위)
  EXPIREAT key timestamp   → 절대 Unix timestamp (초)
  PEXPIREAT key timestamp  → 절대 Unix timestamp (밀리초)

TTL 확인:
  TTL key   → 남은 시간 (초), -1 = 만료 없음, -2 = 키 없음
  PTTL key  → 남은 시간 (밀리초)

만료 해제:
  PERSIST key → expires dict에서 해당 키 제거 (영구 키로 변환)
```

### 2. Lazy 만료 (Passive Expiry)

```
동작 시점: 클라이언트가 키에 접근할 때마다 확인

흐름:
  클라이언트 → GET user:1
                   │
                   ▼
  Redis 이벤트 루프 → getCommand()
                   │
                   ▼
              lookupKeyRead(db, key) 호출
                   │
                   ▼
              db->expires에서 user:1 만료 시각 확인
                   │
                   ├─ 만료 시각 없음 (TTL 없는 키) → 값 반환
                   │
                   ├─ 만료 시각 > 현재 시각 → 아직 유효, 값 반환
                   │
                   └─ 만료 시각 ≤ 현재 시각 → 만료됨!
                            │
                            ▼
                       dbDelete(db, key) 호출
                       → dict에서 키 삭제
                       → expires에서 키 삭제
                       → 클라이언트에게 nil 반환
                            │
                            ▼
                       AOF에 DEL key 기록 (appendonly 설정 시)
                       Replica에 DEL key 전파 (복제 설정 시)

장점:
  접근할 때만 확인 → CPU 오버헤드 최소
  이벤트 루프를 별도로 차단하지 않음

단점:
  접근이 없는 만료 키는 영원히 메모리에 남음
  → Active 만료로 보완 필요
```

### 3. Active 만료 (Active Expiry)

```
동작 시점: serverCron() 함수 실행마다 (hz 설정에 따라)

hz 설정:
  hz 10  → 초당 10회 serverCron() 실행 (기본값)
  hz 100 → 초당 100회 실행 (CPU 증가, 만료 처리 빠름)
  
  동적 hz (Redis 4.0+):
  dynamic-hz yes (기본 활성) → 클라이언트 수에 따라 자동 조정
  클라이언트 많음 → hz 증가, 클라이언트 없음 → hz 감소

Active 만료 알고리즘 (activeExpireCycle):

  매 serverCron() 호출 시:
  
  1. 전체 실행 시간 중 최대 25%만 만료 처리에 사용
     (나머지 75%는 다른 serverCron 작업과 이벤트 처리)
  
  2. 각 DB(database)에 대해:
     a. expires dict에서 무작위로 20개 키 샘플링
     b. 샘플 중 만료된 키 모두 삭제
     c. "만료된 키 비율"이 25% 이상이면 같은 DB에서 반복 (최대 20회)
     d. 25% 미만이면 다음 DB로 이동
  
  실행 예시:
    expires dict: 10만 개 키
    샘플링 20개 → 만료된 것 8개 (40%)
    → 8개 삭제 후 다시 20개 샘플링 (40% ≥ 25%이므로)
    → 20개 중 4개 만료 (20%)
    → 4개 삭제 후 다음 DB로 (20% < 25%이므로)
  
  시간 제한:
    ACTIVE_EXPIRE_CYCLE_FAST: 1ms 이내 (before sleep hook)
    ACTIVE_EXPIRE_CYCLE_SLOW: hz 주기의 25% 이내

Active 만료 한계:
  만료 키가 매우 많고 새 만료 키가 계속 생성되면:
    샘플링으로 계속 25% 이상 → 반복 삭제 → CPU 점유
    시간 제한에 걸려 완전 삭제 못 함 → 만료 키 누적

  진단:
    redis-cli INFO stats | grep expired_keys
    # expired_keys: 12345678  (누적 만료 처리 건수)
    
    redis-cli INFO keyspace
    # db0:keys=100000,expires=50000,avg_ttl=25000
    # expires=50000이면 만료 대기 키가 5만 개
```

### 4. Lazy vs Active 만료 비교

```
┌─────────────────────┬──────────────────────────┬──────────────────────────┐
│                     │ Lazy 만료                 │ Active 만료               │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 실행 시점             │ 키 접근 시 (GET, HGET...)  │ serverCron() 주기적 실행    │
│ 처리 범위             │ 접근된 키 1개               │ 무작위 샘플 20개씩           │
│ CPU 비용             │ 접근 시 추가 만료 체크        │ hz 주기마다 샘플링+삭제       │
│ 메모리 회수 보장        │ 접근 없으면 회수 안 됨        │ 주기적으로 회수              │
│ 정확성                │ 접근 즉시 정확히 만료         │ 만료 후 회수까지 지연 있음     │
└─────────────────────┴──────────────────────────┴──────────────────────────┘

두 방식의 보완 관계:
  자주 접근되는 핫한 키 → Lazy 만료가 즉시 처리
  접근이 드문 콜드 키  → Active 만료가 주기적으로 정리

만료 처리 보장 없는 경우:
  콜드 키 + Active 만료 실패 (샘플링에 안 걸림) → 지연 남아있음
  대량 만료 + hz 낮음 → 수 초 ~ 수 분 지연 가능
  → 만료 키가 많은 환경: hz 증가 또는 maxmemory 정책으로 보완
```

### 5. Replica에서의 만료 처리

```
핵심 원칙: Replica는 독자적으로 만료를 실행하지 않음

Master-Replica 만료 전파 흐름:

  키가 만료되는 두 가지 경로:
  
  경로 1: Lazy 만료 (Master에서 클라이언트 접근)
    클라이언트 → Master: GET expired_key
    Master: Lazy 만료 → dbDelete() → DEL expired_key를 AOF/Replica에 전파
    Replica: DEL expired_key 수신 → 키 삭제
  
  경로 2: Active 만료 (Master에서 주기적 실행)
    Master: activeExpireCycle() → 만료 키 발견 → dbDelete()
    → DEL expired_key를 Replica에 전파
    Replica: DEL expired_key 수신 → 키 삭제

문제: 전파 지연
  Master에서 만료됐지만 Replica에 DEL이 아직 전파되지 않은 구간:
  
    t=100: Master에서 키 만료 (Active 만료 대기 중)
    t=101: 클라이언트가 Replica에서 GET → 만료된 값 반환! ← 불일치
    t=102: Master Active 만료 실행 → DEL Replica 전파
    t=103: Replica에서 키 삭제 완료

Replica의 만료 보호 (Redis 3.2+):
  Replica가 직접 쓰기 권한 없어도 만료 여부는 자체 확인
  만료된 키에 접근 시 nil 반환 (데이터는 아직 있지만 클라이언트에는 nil)
  → Master DEL 전파 전에도 Replica가 expired 키를 노출하지 않음
  
  단, 이 동작은 읽기 경로에서만: Replica가 직접 삭제하지는 않음
  → Replica의 expires dict는 Master DEL 명령 수신 후에만 정리

AOF와 만료 키:
  AOF에는 실제 DEL 명령어가 기록됨 (만료 시각 자체가 아님)
  Redis 재시작 시 AOF 재생: DEL 명령어 실행 → 만료 키 삭제됨
  
  RDB와 만료 키:
  RDB 저장 시: 이미 만료된 키는 RDB에 포함하지 않음
  RDB 로드 시: 로드 시점에 이미 만료된 키는 무시
```

### 6. EXPIRETIME과 만료 시각 관리

```
만료 시각 정밀도:
  Redis 저장: 밀리초 단위 Unix timestamp
  TTL / PTTL: 남은 시간 조회 (정수 반내림)
  
  주의: TTL이 1초로 표시되어도 실제로는 0.5초 남을 수 있음

만료 시각 확인 (Redis 7.0+):
  EXPIRETIME key   → 만료될 Unix timestamp (초)
  PEXPIRETIME key  → 만료될 Unix timestamp (밀리초)
  → -1: 만료 없음, -2: 키 없음

만료 관련 이벤트 (Keyspace Notification):
  redis-cli CONFIG SET notify-keyspace-events "Ex"
  # E = Keyevent events
  # x = expired events
  
  redis-cli SUBSCRIBE __keyevent@0__:expired
  # → 키가 만료될 때마다 키 이름 수신
  
  주의: 이벤트는 "만료 처리가 실행될 때" 발생
        키가 만료 시각을 지난 순간이 아닌,
        Lazy 또는 Active 만료가 해당 키를 처리할 때 발생
        → 최대 수 초 지연 가능
```

---

## 💻 실전 실험

### 실험 1: Lazy 만료 타이밍 관찰

```bash
redis-cli SET mykey "hello" EX 3  # 3초 TTL
redis-cli TTL mykey               # 3
sleep 4                           # 4초 대기 (만료 후)
redis-cli TTL mykey               # -2 (Lazy 만료: 접근 시 처리)
redis-cli EXISTS mykey            # 0 (Lazy 만료로 삭제됨)

# TTL이 지났지만 접근 전 상태 시뮬레이션 (DEBUG로 만료 시각 조작)
redis-cli SET testkey "value" EX 1000
redis-cli DEBUG SLEEP 0          # 연결 확인
redis-cli DEBUG SET-ACTIVE-EXPIRE 0  # Active 만료 비활성화
redis-cli PEXPIREAT testkey 1    # 과거 시각으로 만료 설정 (즉시 만료)
# → testkey는 expires dict에 있지만 Active 만료 비활성화로 삭제 안 됨
redis-cli DEBUG OBJECT testkey    # 키는 존재 (Lazy 만료 안 됨)
redis-cli GET testkey             # nil (Lazy 만료 트리거 → 즉시 삭제)
redis-cli DEBUG OBJECT testkey    # ERR no such key (삭제됨)
redis-cli DEBUG SET-ACTIVE-EXPIRE 1  # Active 만료 복원
```

### 실험 2: Active 만료 속도 비교 (hz 설정)

```bash
# hz=10 (기본값)
redis-cli CONFIG SET hz 10
redis-cli CONFIG SET active-expire-enabled yes

# 1만 개 만료 키 생성 (1초 TTL)
for i in $(seq 1 10000); do
  redis-cli SET "expire:key:$i" "value" EX 1 > /dev/null
done
echo "10,000개 만료 키 생성 완료"
redis-cli DBSIZE  # 10,000

sleep 2  # TTL 만료 대기
redis-cli DBSIZE  # Active 만료로 얼마나 줄었는지 확인
redis-cli INFO stats | grep expired_keys

# hz=100으로 변경
redis-cli CONFIG SET hz 100

# 다시 1만 개 생성
for i in $(seq 1 10000); do
  redis-cli SET "expire2:key:$i" "value" EX 1 > /dev/null
done
sleep 2
redis-cli DBSIZE  # 더 빨리 줄어들었는지 비교
```

### 실험 3: Keyspace Notification으로 만료 이벤트 수신

```bash
# 터미널 1: 만료 이벤트 구독
redis-cli CONFIG SET notify-keyspace-events "Ex"
redis-cli SUBSCRIBE "__keyevent@0__:expired"

# 터미널 2: 만료 키 생성
redis-cli SET notify_test "value" EX 5
# 5초 후 터미널 1에서:
# 1) "message"
# 2) "__keyevent@0__:expired"
# 3) "notify_test"

# 이벤트 수신 지연 확인 (만료 시각과 이벤트 발생 간 차이)
redis-cli SET timed_key "v" EX 2
START=$(date +%s%N)
redis-cli SUBSCRIBE "__keyevent@0__:expired" &
SUB_PID=$!
sleep 3
RECEIVED=$(date +%s%N)
echo "지연: $(( (RECEIVED - START) / 1000000 )) ms"  # 2000ms 이상
kill $SUB_PID
```

### 실험 4: expires dict 현황 모니터링

```bash
# keyspace 정보 확인
redis-cli INFO keyspace
# db0:keys=150000,expires=80000,avg_ttl=45678
# → 전체 15만 키 중 8만 개가 TTL 있음, 평균 잔여 TTL 45.6초

# 만료 처리 통계
redis-cli INFO stats | grep -E "expired_keys|evicted_keys"
# expired_keys:456789   ← 누적 만료 처리 건수
# evicted_keys:0        ← maxmemory-policy에 의한 제거 건수

# 실시간 만료 처리 속도 측정 (1초 간격)
PREV=$(redis-cli INFO stats | grep expired_keys | awk -F: '{print $2}' | tr -d ' \r')
sleep 1
CURR=$(redis-cli INFO stats | grep expired_keys | awk -F: '{print $2}' | tr -d ' \r')
echo "초당 만료 처리: $((CURR - PREV)) 건"
```

---

## 📊 성능 비교

```
만료 처리 방식별 특성:

                    Lazy 만료            Active 만료
─────────────────────────────────────────────────────
실행 조건         키 접근 시            hz 주기마다
CPU 비용          접근 빈도 비례        hz × 샘플링 비용
메모리 회수       즉시 (접근 시)        지연 (다음 주기)
정밀도            높음 (접근 즉시)      낮음 (샘플링 기반)
콜드 키 처리      불가 (접근 없으면)   가능 (샘플링)

hz 값별 만료 처리 속도:

hz=10:
  초당 10회 × 샘플 20개 = 초당 최대 200개 키 검사
  만료 키 비율 25% 가정: 초당 최대 50개 확실 삭제
  → 100만 만료 키: ~5.5시간 소요 (최악의 경우)

hz=100:
  초당 100회 × 샘플 20개 = 초당 최대 2000개 키 검사
  → 100만 만료 키: ~33분 소요

권장:
  대부분: hz=10 (기본값)
  만료 키가 많고 빠른 회수 필요: hz=20~100 (CPU 증가 감수)
  dynamic-hz=yes: 연결 수에 따라 자동 조정 (기본 활성)
```

---

## ⚖️ 트레이드오프

```
Lazy 만료 우선의 장단점:
  장점: CPU 오버헤드 없음 (접근 없으면 비용 없음)
  단점: 접근 없는 만료 키의 메모리 회수 불보장

Active 만료 강화 (hz 증가):
  장점: 빠른 메모리 회수, 만료 키 누적 방지
  단점: CPU 증가 (hz=100은 기본값의 10배 CPU)
        이벤트 루프 처리 시간 25% 할당 고정

만료 보장이 필요한 서비스:
  보안: 세션 토큰 만료 → Lazy 만료 충분 (접근 시 즉시 처리)
  메모리: 캐시 데이터 회수 → Active 만료 중요 (접근 없어도 회수)
  정확성: 정확한 만료 시각 보장 필요 → Keyspace Notification 활용

주의: EXPIRY 시간 = "이 이후에는 절대 읽히지 않음" 보장
      EXPIRY 시간 = "이 시각에 메모리에서 삭제됨" 보장이 아님
```

---

## 📌 핵심 정리

```
Redis 키 만료 메커니즘:

만료 정보 저장:
  db->expires dict에 Unix timestamp(ms)로 저장
  TTL 없는 키는 expires에 없음

Lazy 만료:
  키 접근(GET, HGET 등) 시마다 만료 확인
  만료됐으면 즉시 삭제 후 nil 반환
  접근 없는 키는 Lazy 만료 안 됨

Active 만료:
  serverCron()에서 주기적으로 샘플 20개씩 검사
  hz 설정으로 초당 실행 횟수 조정 (기본: 10)
  만료 키 비율 25% 이상이면 반복 샘플링

Replica 만료:
  Replica는 독자적으로 삭제 안 함
  Master의 DEL 전파를 기다림
  Redis 3.2+: Replica 읽기 시 만료 키는 nil 반환 (DEL 전파 전에도)

AOF/RDB:
  RDB: 이미 만료된 키는 저장하지 않음
  AOF: DEL 명령어로 기록 (만료 처리 시점에)

진단:
  INFO keyspace    → expires 키 수, avg_ttl
  INFO stats       → expired_keys (누적 만료 건수)
  Keyspace Notification → 만료 이벤트 실시간 수신
```

---

## 🤔 생각해볼 문제

**Q1.** `SET key value EX 60`으로 설정한 키가 정확히 60초 후에 삭제될 것이라 가정하고 설계를 했다. 이 가정이 틀릴 수 있는 조건은?

<details>
<summary>해설 보기</summary>

다음 조건에서 60초 이후에도 키가 메모리에 남아있을 수 있습니다:

1. **Active 만료 지연**: hz가 낮거나, 만료 키가 많아 샘플링에 해당 키가 포함되지 않으면 삭제가 지연됩니다. `MEMORY USAGE`로 만료된 키를 확인하면 여전히 메모리를 점유 중일 수 있습니다.

2. **접근 없는 경우**: 해당 키에 아무도 접근하지 않으면 Lazy 만료도 동작하지 않습니다.

3. **Redis 부하 높은 상황**: Active 만료는 CPU 시간의 25%만 사용하도록 제한되어 있어, Redis가 바쁘면 만료 처리 우선순위가 낮아집니다.

**결론**: TTL 만료가 "이 이후로는 읽히지 않음"을 보장하지만, "정확히 이 시각에 메모리에서 삭제됨"은 보장하지 않습니다. 메모리 기반 설계보다는 **TTL 이후 접근 시 nil 반환** 을 신뢰하세요.

</details>

---

**Q2.** 1000만 개의 키에 1시간 TTL을 설정하고, 1시간 후 메모리가 회수되어야 하는 서비스를 설계한다면 어떤 설정을 권장하는가?

<details>
<summary>해설 보기</summary>

```bash
# 1. hz 증가 (Active 만료 빈도 향상)
redis-cli CONFIG SET hz 50  # 기본 10 → 50

# 2. dynamic-hz 활성화 (이미 기본값)
redis-cli CONFIG SET dynamic-hz yes

# 3. maxmemory + allkeys-lru 조합
#    만료 처리보다 maxmemory 정책이 메모리 확보를 먼저 할 수 있음
redis-cli CONFIG SET maxmemory 4gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# 4. Keyspace Notification으로 만료 모니터링
redis-cli CONFIG SET notify-keyspace-events "Ex"
# → expired_keys 속도 실시간 확인
```

추가로, **만료 키가 1000만 개에 달하는 설계 자체를 재검토**하는 것이 좋습니다. 이렇게 많은 만료 키가 동시에 생기는 패턴은 "Thundering Expiry"를 유발합니다. TTL을 일정 범위 내 랜덤하게 분산 (`60분 ± 5분 random`) 하면 Active 만료 부하가 균등하게 분배됩니다.

```java
// Spring: TTL에 랜덤 지터 추가
long ttlSeconds = 3600 + (long)(Math.random() * 300);  // 3600~3900초
redisTemplate.opsForValue().set(key, value, ttlSeconds, TimeUnit.SECONDS);
```

</details>

---

**Q3.** Keyspace Notification에서 `expired` 이벤트를 수신하여 "키 만료 시 후처리" 로직을 구현했다. 이 설계의 신뢰성 문제는 무엇인가?

<details>
<summary>해설 보기</summary>

두 가지 핵심 문제가 있습니다:

**문제 1: 이벤트 전달 보장 없음**
Keyspace Notification은 Pub/Sub 방식으로 동작합니다. Pub/Sub은 "구독자가 없으면 메시지 소실"입니다. 애플리케이션이 Redis에 연결되지 않은 순간(재시작, 네트워크 단절)에 발생한 만료 이벤트는 영원히 손실됩니다.

**문제 2: 이벤트 지연**
만료 시각과 이벤트 수신 사이에 Lazy/Active 만료 실행까지의 지연이 있습니다. "정확히 X초에 후처리" 보장이 불가능합니다.

**신뢰성 있는 대안:**
- **Polling**: 주기적으로 만료된 것으로 예상되는 키를 직접 확인
- **Redis Stream**: 만료 예정 이벤트를 Stream에 기록해 Consumer Group으로 처리 (영구 저장, 재처리 가능)
- **별도 스케줄러**: 만료 시각을 DB에 저장하고 스케줄러가 주기적으로 처리

Keyspace Notification은 "로깅, 모니터링" 용도로는 적합하지만, 비즈니스 로직의 신뢰할 수 있는 트리거로는 사용하지 마세요.

</details>

---

<div align="center">

**[⬅️ 이전: Redis 객체 시스템](./03-redis-object-system.md)** | **[홈으로 🏠](../README.md)** | **[다음: Redis 6.0 Threaded I/O ➡️](./05-threaded-io.md)**

</div>
