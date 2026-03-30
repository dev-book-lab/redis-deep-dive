# 데이터 구조 선택 가이드 — 메모리 vs 성능 트레이드오프

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 같은 데이터를 String vs Hash vs List로 저장할 때 실제 메모리 차이는 얼마인가?
- `OBJECT ENCODING`, `DEBUG OBJECT`, `MEMORY USAGE`로 무엇을 알 수 있고 어떻게 다른가?
- "작게 유지하면 listpack, 크면 hashtable"이 왜 10배 메모리 차이를 만드는가?
- 유저 프로필, 세션, 랭킹, 카운터 등 서비스 패턴별로 어떤 자료구조가 최적인가?
- 인코딩을 의도적으로 제어하는 설계 방법은?

---

## 🔍 왜 이 개념이 중요한가

### 자료구조 선택이 서버 비용을 결정한다

```
실제 사례 패턴:

  서비스: 유저 10만 명의 프로필 캐싱 (이름, 나이, 도시, 가입일 등 10개 필드)

  팀 A (String 방식):
    SET user:1:name "Alice"
    SET user:1:age "30"
    SET user:1:city "Seoul"
    ... (10개 키)
    
    10만 유저 × 10 필드 = 100만 개 키
    각 키 = redisObject + SDS(키명) + redisObject + SDS(값) + dict entry
    평균 키당 약 150~200 bytes
    총: 150~200 MB
  
  팀 B (Hash + listpack 유지):
    HSET user:1 name "Alice" age "30" city "Seoul" ...
    
    10만 유저 = 10만 개 Hash (각 10필드 → listpack)
    Hash당 약 300~400 bytes (listpack: 포인터 오버헤드 없음)
    총: 30~40 MB
    
    → 팀 A 대비 5배 메모리 절약
    → EC2 r5.large(16 GB, $180/월) → r5.medium(8 GB, $90/월) 가능
    → 연 $1,080 절약 (단순 계산)

  팀 C (Hash + hashtable 전환 허용):
    같은 HSET이지만 필드 200개 → hashtable 전환
    Hash당 약 3,000~5,000 bytes
    총: 300~500 MB → 팀 A보다 오히려 2~3배 더 사용!

"자료구조 선택은 기능의 문제가 아니라 비용의 문제다"
```

---

## 🔬 내부 동작 원리

### 1. 진단 명령어 완전 가이드

```bash
# 세 가지 진단 명령어의 목적 구분

# OBJECT ENCODING: "어떤 인코딩으로 저장됐는가"
redis-cli OBJECT ENCODING mykey
# → int | embstr | raw | listpack | quicklist | hashtable | skiplist | intset

# DEBUG OBJECT: 인코딩 + 직렬화 크기 + LRU 정보
redis-cli DEBUG OBJECT mykey
# → Value at:0x7f... refcount:1 encoding:listpack
#   serializedlength:89 lru:12345678 lru_seconds_idle:2 type:hash
#   serializedlength: RDB 직렬화 크기 (압축 후, 실제 메모리 아님)
#   lru_seconds_idle: OBJECT IDLETIME과 동일

# MEMORY USAGE: 실제 OS 할당 메모리 (jemalloc 슬롯 반영)
redis-cli MEMORY USAGE mykey
redis-cli MEMORY USAGE mykey SAMPLES 5
# → bytes (키 이름 + 값 + 모든 오버헤드 포함)
# SAMPLES: 중첩 구조 샘플링 깊이 (기본 1, 중첩 Hash/List 깊이)

# 세 명령어 비교:
#   OBJECT ENCODING:   타입 파악 → 메모리 최적화 여부 판단
#   DEBUG OBJECT:      RDB 크기 파악 → 네트워크/디스크 영향 추정
#   MEMORY USAGE:      실제 메모리 파악 → 인스턴스 sizing

# 일괄 분석 스크립트
analyze_key() {
  KEY=$1
  TYPE=$(redis-cli TYPE $KEY)
  ENC=$(redis-cli OBJECT ENCODING $KEY)
  MEM=$(redis-cli MEMORY USAGE $KEY)
  TTL=$(redis-cli TTL $KEY)
  echo "KEY=$KEY TYPE=$TYPE ENCODING=$ENC MEMORY=${MEM}B TTL=${TTL}s"
}

# 상위 메모리 사용 키 탐색
redis-cli --no-auth-warning MEMORY USAGE $(
  redis-cli SCAN 0 COUNT 100 | tail -n +2 | head -20
) 2>/dev/null | paste - - | sort -k2 -nr | head -10
```

### 2. 같은 데이터, 다른 자료구조 메모리 비교

```
시나리오: 유저 10명의 { id, name, score } 저장

방법 1: String (필드별 개별 키)
  SET user:1:id "1"
  SET user:1:name "Alice"
  SET user:1:score "1500"
  × 10명 = 30개 키

  메모리 분석 (키 1개 기준):
  ┌──────────────────────────────────┬────────────┐
  │ 구성요소                           │ 크기        │
  ├──────────────────────────────────┼────────────┤
  │ dict entry (key→value 매핑)       │ 24 bytes   │
  │ redisObject (키 SDS용)            │ 16 bytes   │
  │ SDS "user:1:name" (13 chars)     │ 3+13+1=17B │
  │ jemalloc 슬롯 (32B 슬롯)           │ 32 bytes   │
  │ redisObject (값용)                │ 16 bytes   │
  │ SDS "Alice" (5 chars, embstr)    │ 1회 malloc │
  │ jemalloc 슬롯 (embstr 64B 슬롯)    │ 64 bytes   │
  ├──────────────────────────────────┼────────────┤
  │ 키 1개 총계 (대략)                  │ ~150 bytes │
  └──────────────────────────────────┘
  30개 키 총계: ~4,500 bytes

방법 2: Hash (유저당 1개 Hash, listpack)
  HSET user:1 id "1" name "Alice" score "1500"
  × 10명 = 10개 Hash

  메모리 분석 (Hash 1개 기준, 3 필드):
  ┌──────────────────────────────────┬────────────┐
  │ 구성요소                           │ 크기        │
  ├──────────────────────────────────┼────────────┤
  │ dict entry (키→Hash 매핑)          │ 24 bytes   │
  │ redisObject (키)                  │ 16 bytes   │
  │ SDS "user:1" (6 chars)           │ ~30 bytes  │
  │ redisObject (Hash용)              │ 16 bytes   │
  │ listpack 데이터                    │            │
  │   헤더: 6 bytes                    │            │
  │   "id"+backlen+1+backlen          │ ~12 bytes  │
  │   "name"+backlen+5+backlen        │ ~18 bytes  │
  │   "score"+backlen+4+backlen       │ ~18 bytes  │
  ├───────────────────────────────────┼────────────┤
  │ Hash 1개 총계 (대략)                 │ ~180 bytes │
  └───────────────────────────────────┘
  10개 Hash 총계: ~1,800 bytes
  → String 방식 대비 2.5배 절약

방법 3: Hash (필드 수 > 128 → hashtable)
  위와 동일 데이터지만 필드가 200개인 경우:
  hashtable: dictEntry × 200 + 버킷 배열 + SDS들
  → ~15,000 bytes per Hash
  → 10개: 150,000 bytes = String 방식의 33배!

  결론: Hash 최적화의 핵심 = listpack 범위 유지
```

### 3. 서비스 패턴별 자료구조 결정 트리

```
데이터 선택 질문 흐름:

Q1: 데이터에 순서가 있는가?
  ├─ Yes → Q2
  └─ No  → Q3

Q2: 스코어(우선순위, 점수)가 있는가?
  ├─ Yes → Sorted Set
  │         ZADD ranking score member
  │         ZREVRANGE ranking 0 9  (상위 10개)
  └─ No  → List (삽입 순서 보존)
            LPUSH/RPUSH + LRANGE

Q3: 데이터가 단순 존재 여부(멤버십)인가?
  ├─ Yes → Q4
  └─ No  → Q5

Q4: 정수 ID인가?
  ├─ Yes + 512개 이하 → Set (intset)
  │  SADD permissions 1 2 3
  │  SISMEMBER permissions 2  → O(log N)
  └─ No  → Set (listpack 또는 hashtable)
            SADD tags "news" "tech"
            SISMEMBER tags "news"

Q5: 카운터인가?
  ├─ Yes, 정수 카운터 → String (int 인코딩)
  │  SET counter 0 + INCR counter
  └─ Yes, 근사 카운터 → HyperLogLog
     PFADD uv user_id + PFCOUNT uv

Q6: 구조화된 데이터(여러 필드)인가?
  ├─ Yes, 필드 128개 이하 + 각 값 64 bytes 이하
  │  → Hash (listpack)
  │  HSET user:1 name "Alice" age 30
  └─ Yes, 필드 128개 초과 or 큰 값
     → Q7

Q7: 개별 필드 접근인가, 전체 조회인가?
  ├─ 개별 필드 O(1) 접근 필요 → Hash (hashtable 허용)
  │  HGET user:1 name
  └─ 전체 조회 위주 → JSON String (직렬화)
     SET user:1 '{"name":"Alice","fields":...}'
     → 단, JSON 파싱 비용 있음
```

### 4. 핵심 패턴별 최적 구현

```
패턴 A: 실시간 랭킹

  요구사항:
    점수 업데이트: 빈번
    상위 N명 조회: 빈번
    내 순위 조회: 빈번
  
  최적: Sorted Set
    ZADD game:score {score} {user_id}       O(log N)
    ZINCRBY game:score {delta} {user_id}    O(log N)
    ZREVRANGE game:score 0 9 WITHSCORES     O(log N + K)
    ZREVRANK game:score {user_id}           O(log N)
  
  주의:
    원소 수 > 128 or 멤버 > 64 bytes → skiplist (listpack → skiplist 전환)
    skiplist 메모리: ~200 bytes/원소 (두 자료구조 유지)
    100만 유저 → ~200 MB → 허용 가능

패턴 B: 세션 저장

  요구사항:
    세션 ID로 O(1) 조회
    여러 필드 (user_id, permissions, last_seen...)
    TTL로 자동 만료
  
  최적: Hash + TTL
    HSET session:{token} user_id 1001 permissions "admin" last_seen 1711234567
    EXPIRE session:{token} 3600  # 1시간
    
    또는 HSET + EXPIREAT로 절대 시간 설정
  
  주의:
    필드 수 ≤ 128 → listpack 유지 → 메모리 최소
    필드가 많으면 Hash를 논리적으로 분리
    세션별 TTL이 다르면 개별 EXPIRE 필요 (Hash 내 특정 필드 TTL 불가)

패턴 C: 피드/타임라인

  요구사항:
    최신 N개만 유지
    순서 보장
    빠른 최신 조회
  
  최적: List + LTRIM
    LPUSH feed:{user_id} {event_json}
    LTRIM feed:{user_id} 0 99   # 최신 100개만 유지
    LRANGE feed:{user_id} 0 9   # 최신 10개 조회
  
  대안: Sorted Set (time-ordered)
    ZADD feed:{user_id} {timestamp} {event_id}
    ZREVRANGEBYSCORE feed:{user_id} +inf -inf LIMIT 0 10
    → 시간 범위 탐색 가능 (List는 불가)

패턴 D: 방문자 통계

  요구사항 A: 정확한 MAU (중복 제거)
  요구사항 B: 특정 유저 방문 여부 확인
  요구사항 C: 이틀 모두 방문한 유저 수
  
  요구사항 A만: HyperLogLog (12 KB)
    PFADD mau:{yyyymm} {user_id}
    PFCOUNT mau:{yyyymm}
  
  요구사항 A+B+C: Bitmap (12.5 MB / 1억 유저)
    SETBIT dau:{yyyymmdd} {user_id} 1
    BITCOUNT dau:{yyyymmdd}
    GETBIT dau:{yyyymmdd} {user_id}
    BITOP AND both dau:20240101 dau:20240102
    BITCOUNT both

패턴 E: 분산 큐 (작업 처리)

  요구사항:
    순서 보장
    처리 보장 (크래시 후 재처리)
    병렬 Consumer
  
  단순 요구사항: List + BLPOP
    RPUSH queue:{name} {task_json}     # 생산
    BLPOP queue:{name} 0              # 소비 (블로킹)
    단점: 처리 중 크래시 시 메시지 손실
  
  처리 보장 필요: Stream + Consumer Group
    XADD tasks '*' data {task_json}   # 생산
    XREADGROUP GROUP workers c1 ...   # 소비
    XACK tasks workers {id}           # 완료 확인
    → PEL로 미ACK 메시지 추적 → 재처리 가능

패턴 F: 위치 기반 서비스

  요구사항:
    반경 N km 내 가게 검색
    가게 추가/삭제
    거리 계산
  
  최적: Geo (Sorted Set 기반)
    GEOADD stores {lng} {lat} {store_id}
    GEOSEARCH stores FROMLONLAT {lng} {lat} BYRADIUS 3 km ASC COUNT 20
    GEODIST stores store1 store2 km
  
  주의:
    Geo는 Sorted Set → TYPE = "zset"
    원소 수 > 128 → skiplist 전환 (대부분 skiplist 사용)
    복잡한 지리 쿼리(폴리곤 등)는 PostGIS 필요
```

### 5. 메모리 최적화 기법 — 인코딩 제어

```
기법 1: 임계값 조정으로 listpack 유지

  # Hash: 필드 수/값 크기 임계값
  hash-max-listpack-entries 128   # 기본값
  hash-max-listpack-value   64    # bytes

  # 최적화 예시: 필드가 항상 50개 이하이고 값이 30 bytes 이하면
  # → 임계값을 낮춰 메모리를 아끼지 않아도 됨 (이미 listpack)
  # → 필드가 200개로 늘어날 수 있으면 설계 변경이 더 효과적

  # List
  list-max-listpack-size -2     # 기본: 8 KB per node

  # Set
  set-max-intset-entries 512    # 정수 Set
  set-max-listpack-entries 128  # 문자열 Set

  # Sorted Set
  zset-max-listpack-entries 128
  zset-max-listpack-value   64

기법 2: Hash Sharding으로 listpack 유지

  # 안 좋은 패턴 (200 필드 → hashtable)
  HSET user:1 field1 val1 field2 val2 ... field200 val200
  
  # 좋은 패턴 (각 Hash 50 필드 이하 → listpack)
  HSET user:1:profile  name "Alice" age 30 city "Seoul" ...  (10 필드)
  HSET user:1:stats    login_count 42 last_login "..." ...  (10 필드)
  HSET user:1:prefs    theme "dark" lang "ko" ...           (10 필드)
  HSET user:1:social   following 500 followers 1200 ...     (10 필드)
  
  → 4개 Hash, 각 10 필드 → 모두 listpack
  → Pipeline으로 4개 Hash 동시 조회 (RTT 1회)

기법 3: 키 이름 단축

  # 기존 (긴 키)
  HSET user:session:12345 user_id 1001 permissions "admin,read" last_seen 1711234567
  
  # 단축 (키 이름 오버헤드 절감)
  HSET s:12345 u 1001 p "admin,read" t 1711234567
  
  효과:
    키 이름: "user:session:12345" (19 bytes) → "s:12345" (7 bytes)
    SDS + dict entry 절감
    단, 가독성 저하 → 팀 내 문서화 필요
  
  실측:
    MEMORY USAGE "user:session:12345"  # X bytes
    MEMORY USAGE "s:12345"             # Y bytes (X > Y)

기법 4: 값 직렬화 방식 선택

  # JSON (가독성 좋지만 크기 큼)
  SET user:1 '{"name":"Alice Wonderland","age":30,"city":"Seoul"}'
  
  # MessagePack (JSON보다 20~30% 작음, 바이너리)
  SET user:1 <msgpack_bytes>
  
  # 필드별 Hash (개별 접근 가능, 직렬화 불필요)
  HSET user:1 name "Alice Wonderland" age 30 city "Seoul"
  
  비교:
    JSON String: 직렬화/역직렬화 비용, 개별 필드 업데이트 불가
    MessagePack:  JSON보다 작지만 여전히 전체 직렬화 필요
    Hash:         개별 필드 HGET/HSET 가능, 직렬화 불필요
                  필드 수 ≤ 128 + 값 ≤ 64 bytes → listpack → 최적
```

---

## 💻 실전 실험

### 실험 1: 같은 데이터, 다른 자료구조 메모리 측정

```bash
# 10만 유저 데이터 저장 (각 5개 필드)
USER_COUNT=100000

# 방법 1: String (필드당 개별 키)
redis-cli FLUSHDB
START=$(date +%s%3N)
for i in $(seq 1 $USER_COUNT); do
  redis-cli PIPELINE <<EOF
SET user:$i:name "User$i"
SET user:$i:age "$((20 + i % 50))"
SET user:$i:city "Seoul"
SET user:$i:score "$((RANDOM % 10000))"
SET user:$i:active "1"
EOF
done 2>/dev/null
END=$(date +%s%3N)
MEM_STRING=$(redis-cli INFO memory | grep "^used_memory:" | awk -F: '{print $2}' | tr -d ' \r')
KEYS_STRING=$(redis-cli DBSIZE)
echo "String 방식: ${MEM_STRING} bytes, ${KEYS_STRING}개 키, $((END-START))ms"

# 방법 2: Hash (유저당 1개 Hash, listpack 유지)
redis-cli FLUSHDB
START=$(date +%s%3N)
for i in $(seq 1 $USER_COUNT); do
  redis-cli HSET user:$i \
    name "User$i" \
    age "$((20 + i % 50))" \
    city "Seoul" \
    score "$((RANDOM % 10000))" \
    active "1" > /dev/null
done
END=$(date +%s%3N)
MEM_HASH=$(redis-cli INFO memory | grep "^used_memory:" | awk -F: '{print $2}' | tr -d ' \r')
KEYS_HASH=$(redis-cli DBSIZE)

# 인코딩 확인
redis-cli OBJECT ENCODING user:1   # listpack

echo "Hash 방식:   ${MEM_HASH} bytes, ${KEYS_HASH}개 키, $((END-START))ms"
echo ""
echo "메모리 절감: $(echo "scale=1; ($MEM_STRING - $MEM_HASH) * 100 / $MEM_STRING" | bc)%"
echo "키 수 감소:  $((KEYS_STRING - KEYS_HASH))개 감소"
```

### 실험 2: listpack → hashtable 전환 임계값 효과 측정

```bash
# 임계값을 낮춰 강제 hashtable 전환
redis-cli CONFIG SET hash-max-listpack-entries 5

redis-cli DEL small_hash big_hash

# 5개 필드 (listpack 유지)
for field in f1 f2 f3 f4 f5; do
  redis-cli HSET small_hash $field "value-$field" > /dev/null
done

# 6개 필드 (hashtable 전환)
for field in f1 f2 f3 f4 f5 f6; do
  redis-cli HSET big_hash $field "value-$field" > /dev/null
done

echo "=== 같은 데이터, 인코딩만 다름 ==="
echo "5 필드 (listpack):"
redis-cli OBJECT ENCODING small_hash
redis-cli MEMORY USAGE small_hash

echo "6 필드 (hashtable):"
redis-cli OBJECT ENCODING big_hash
redis-cli MEMORY USAGE big_hash

# 임계값 복원
redis-cli CONFIG SET hash-max-listpack-entries 128
```

### 실험 3: 카운터 저장 방식 비교

```bash
# 정수 카운터: int 인코딩 vs 문자열
redis-cli SET counter_int 0
redis-cli INCR counter_int
redis-cli OBJECT ENCODING counter_int   # int
redis-cli MEMORY USAGE counter_int      # ~52 bytes

# 동일 값을 문자열로 저장
redis-cli SET counter_str "1"
redis-cli OBJECT ENCODING counter_str   # int (Redis가 자동으로 int로)
redis-cli MEMORY USAGE counter_str      # 같음

# 부동소수점 카운터
redis-cli SET counter_float "1.5"
redis-cli OBJECT ENCODING counter_float  # embstr
redis-cli MEMORY USAGE counter_float     # 더 큼

# 대용량 카운터 비교 (100만 유저 활동 카운터)
USERS=1000000
redis-cli FLUSHDB

# intset으로 활성 유저 Set (정수 ID)
for i in $(seq 1 1000); do redis-cli SADD active_users $i > /dev/null; done
redis-cli OBJECT ENCODING active_users    # intset (1000개, 512 초과)
# 실제 intset 한도 512이므로 512까지만 테스트
redis-cli DEL active_users
for i in $(seq 1 512); do redis-cli SADD active_users $i > /dev/null; done
redis-cli OBJECT ENCODING active_users    # intset
redis-cli MEMORY USAGE active_users       # 매우 작음

# hashtable Set으로 비교 (문자열 ID)
redis-cli DEL active_users_str
for i in $(seq 1 512); do redis-cli SADD active_users_str "user:$i" > /dev/null; done
redis-cli OBJECT ENCODING active_users_str  # hashtable
redis-cli MEMORY USAGE active_users_str     # 훨씬 큼
```

### 실험 4: 진단 명령어 종합 활용

```bash
# 다양한 자료구조 생성
redis-cli SET str_small "hello"
redis-cli SET str_large "$(python3 -c "print('x'*100)")"
redis-cli SET num_val 42
redis-cli HSET small_hash f1 v1 f2 v2 f3 v3
redis-cli LPUSH mylist a b c d e
redis-cli SADD myset 1 2 3 4 5
redis-cli ZADD myzset 1.0 a 2.0 b 3.0 c
redis-cli PFADD myhll user1 user2 user3

# 일괄 분석
for key in str_small str_large num_val small_hash mylist myset myzset myhll; do
  TYPE=$(redis-cli TYPE $key)
  ENC=$(redis-cli OBJECT ENCODING $key)
  MEM=$(redis-cli MEMORY USAGE $key)
  IDLE=$(redis-cli OBJECT IDLETIME $key)
  echo "$key | type=$TYPE | encoding=$ENC | memory=${MEM}B | idle=${IDLE}s"
done

# DEBUG OBJECT로 직렬화 크기 vs 실제 메모리 비교
echo ""
echo "=== 직렬화 크기 vs 실제 메모리 ==="
for key in str_small small_hash mylist; do
  SRL=$(redis-cli DEBUG OBJECT $key | grep -o 'serializedlength:[0-9]*' | cut -d: -f2)
  MEM=$(redis-cli MEMORY USAGE $key)
  echo "$key: serialized=${SRL}B, actual=${MEM}B (비율: $(echo "scale=1; $MEM/$SRL" | bc)x)"
done
```

### 실험 5: 실시간 메모리 모니터링

```bash
# 키별 메모리 상위 10개 탐색
redis-cli --no-auth-warning eval "
  local keys = redis.call('SCAN', ARGV[1], 'COUNT', 100)
  local result = {}
  for i, key in ipairs(keys[2]) do
    local mem = redis.call('MEMORY', 'USAGE', key)
    if mem then
      table.insert(result, key .. ':' .. mem)
    end
  end
  return result
" 0 0

# 인코딩별 키 분포 분석
redis-cli SCAN 0 COUNT 1000 | tail -n +2 | while read key; do
  ENC=$(redis-cli OBJECT ENCODING "$key" 2>/dev/null)
  echo $ENC
done | sort | uniq -c | sort -rn

# 메모리 단편화 현황
redis-cli INFO memory | grep -E "mem_fragmentation_ratio|used_memory_human|used_memory_rss_human"
```

---

## 📊 성능 비교

```
데이터 구조별 메모리 밀도 비교 (동일 데이터 기준):

시나리오: 10개 필드 (키 평균 8B, 값 평균 20B) 저장

┌──────────────────────┬──────────────┬───────────────┬──────────────────┐
│ 방법                  │ 인코딩         │ 메모리(근사)     │ 필드 조회 복잡도     │
├──────────────────────┼──────────────┼───────────────┼──────────────────┤
│ String (개별 키)       │ embstr/raw   │ ~1,500 bytes  │ O(1) - 직접 GET   │
│ Hash (listpack)      │ listpack     │ ~500 bytes    │ O(N) - 선형 탐색   │
│ Hash (hashtable)     │ hashtable    │ ~2,000 bytes  │ O(1) - 해시 탐색   │
│ JSON String          │ raw          │ ~400 bytes    │ 파싱 필요          │
└──────────────────────┴──────────────┴───────────────┴──────────────────┘

Hash listpack이 String보다 3배 효율적인 이유:
  String: 키당 dict entry(24) + 키 SDS(~30) + 값 SDS(~30) = ~84 bytes × 10 = 840
  Hash:   dict entry(24) + 키 SDS(~20) + listpack(~400) = ~444 bytes (1개 Hash)
  → 844 vs 444, 약 2배 절약 (실제 jemalloc 슬롯 반영 시 더 큰 차이)

자료구조 선택 결정 매트릭스:

┌─────────────────┬──────┬──────┬────────┬────────┬──────────┬──────┬───────┐
│ 요구사항          │ Str  │ List │ Hash   │ Set    │ SortedSet│ HLL  │ Geo   │
├─────────────────┼──────┼──────┼────────┼────────┼──────────┼──────┼───────┤
│ 단순 값 저장       │ ✓✓   │      │        │        │          │      │       │
│ 카운터            │ ✓✓   │      │        │        │          │      │       │
│ 순서 유지 목록     │      │ ✓✓   │        │        │ ✓        │      │       │
│ 구조화 데이터      │      │      │ ✓✓     │        │          │      │       │
│ 멤버십 체크       │      │      │        │ ✓✓     │          │      │       │
│ 랭킹/정렬         │      │      │        │        │ ✓✓       │      │       │
│ 시간 범위 탐색     │      │      │        │        │ ✓✓       │      │       │
│ 카디널리티 추정    │      │      │        │        │          │ ✓✓   │       │
│ 위치 기반 탐색     │      │      │        │        │          │      │ ✓✓    │
│ 메시지 큐         │      │ ✓    │        │        │          │      │       │
│ 처리 보장 큐       │ ※    │      │        │        │          │      │       │
└─────────────────┴──────┴──────┴────────┴────────┴──────────┴──────┴───────┘
※ Stream 사용 (별도 자료구조)
✓✓ = 최적, ✓ = 가능
```

---

## ⚖️ 트레이드오프

```
"최소 메모리" vs "최고 성능"의 선택:

listpack 유지 (메모리 최소):
  Hash HGET: O(N) 선형 (N ≤ 128)
  Set SISMEMBER: O(log N) 이진탐색 (intset)
  Sorted Set ZADD: O(N) 삽입 (listpack)
  
  허용 가능한 경우:
    트래픽이 적은 서비스
    배치 처리 위주
    조회보다 쓰기가 더 많은 경우

O(1) 구조 (성능 최우선):
  Hash hashtable: O(1) HGET/HSET
  Set hashtable: O(1) SISMEMBER
  Sorted Set skiplist: O(log N) ZADD/ZRANK
  
  필요한 경우:
    초당 수십만 건의 조회
    Hot Key (특정 Hash에 집중)
    응답 시간 SLA가 엄격한 서비스

중간 지점:
  임계값을 조정해 listpack을 더 오래 유지
  or 데이터 구조를 Sharding해서 각 조각을 listpack 범위 내로
  → 메모리 절약 + 성능 허용 범위 유지

"잘못된 트레이드오프"의 징후:
  SLOWLOG에 HGET이 자주 등장 → listpack 임계값이 너무 높음
  used_memory가 예상보다 훨씬 큼 → hashtable 전환이 예상보다 빨리 발생
  → OBJECT ENCODING으로 현재 상태 확인 후 결정
```

---

## 📌 핵심 정리

```
데이터 구조 선택 핵심 원칙:

원칙 1: 인코딩을 먼저 확인하라
  OBJECT ENCODING key → listpack인가, hashtable인가?
  listpack: 메모리 최적, 선형 탐색
  hashtable: 메모리 2~5배, O(1) 탐색

원칙 2: listpack 범위를 의도적으로 유지하라
  Hash: 필드 ≤ 128, 값 ≤ 64 bytes
  List: 원소 ≤ 128 (list-max-listpack-size)
  Set:  정수 ≤ 512 (intset), 문자열 ≤ 128 (listpack)
  ZSet: 원소 ≤ 128, 멤버 ≤ 64 bytes

원칙 3: 패턴별 최적 구조
  카운터 → String int (INCR)
  순위/정렬 → Sorted Set
  멤버십 → Set (정수: intset, 문자열: listpack or hashtable)
  구조화 데이터 → Hash (listpack 유지 설계)
  큐 → List + BLPOP (단순) or Stream (처리 보장)
  근사 통계 → HyperLogLog
  위치 → Geo (Sorted Set 기반)

원칙 4: 메모리 측정 없는 최적화는 없다
  MEMORY USAGE key  → 개별 키
  INFO memory       → 전체 사용량
  DEBUG OBJECT key  → 직렬화 크기 (네트워크/디스크)
  INFO keyspace     → DB별 키 수, expires 수

진단 흐름:
  1. INFO memory → 전체 메모리 파악
  2. SCAN + MEMORY USAGE → 상위 사용 키 찾기
  3. OBJECT ENCODING → listpack인가 확인
  4. 임계값 조정 or 구조 변경 결정
  5. 변경 후 MEMORY USAGE 재측정 → 효과 검증
```

---

## 🤔 생각해볼 문제

**Q1.** 100만 명 유저의 팔로잉/팔로워 관계를 저장하는 시스템을 설계한다. 팔로잉 수가 평균 200명이고 최대 10만 명인 경우, 어떤 자료구조를 선택하겠는가?

<details>
<summary>해설 보기</summary>

팔로잉/팔로워 시스템의 핵심 쿼리:
- "A가 B를 팔로잉하는가?" → 멤버십 체크 O(1)
- "A의 팔로잉 목록 전체" → 목록 조회
- "A와 B의 공통 팔로잉" → 교집합

**권장: Set (사용자당 1개)**

```bash
# A가 B를 팔로우
SADD following:A B

# A가 B를 팔로잉하는가?
SISMEMBER following:A B   # O(1)

# A의 팔로잉 전체
SMEMBERS following:A      # O(N) - N = 팔로잉 수

# A와 B의 공통 팔로잉
SINTERSTORE common:A:B following:A following:B
```

**인코딩 고려:**
- 팔로잉 ID가 정수 → 200명이면 intset (512 이하) → 작은 메모리
- 문자열 ID면 listpack (128 이하) 또는 hashtable
- 팔로잉 200명 → 128 초과 → hashtable 전환 가능

**문제: 팔로잉 10만 명인 사용자 (인플루언서)**
- Set에 10만 원소 → hashtable → 수 MB per user
- 해결책 1: 팔로잉 수 상한 설정 (5000명 등)
- 해결책 2: 인플루언서 계정은 별도 처리 (Sorted Set으로 최근 팔로우 순 정렬)

**실제 서비스 (Twitter/X 방식):**
- 적은 팔로잉: Redis Set으로 빠른 교집합
- 많은 팔로잉: 데이터베이스로 오프로드 + Redis 캐시
- 팔로잉 목록은 DB, 공통 팔로잉은 Redis로 캐싱

</details>

---

**Q2.** `MEMORY USAGE key`가 1 MB를 반환하는 대형 Hash를 발견했다. 이 Hash의 메모리를 줄이기 위한 단계별 접근법은?

<details>
<summary>해설 보기</summary>

**단계별 진단 및 최적화:**

```bash
# 1단계: 현황 파악
OBJECT ENCODING big_hash    # hashtable (대부분)
HLEN big_hash               # 필드 수
MEMORY USAGE big_hash       # 1 MB

# 2단계: 필드 크기 분포 파악
redis-cli HGETALL big_hash | awk '
  NR%2==1 {key=$0}
  NR%2==0 {print length(key), length($0), key}
' | sort -n | tail -20  # 가장 큰 필드들

# 3단계: 원인 파악
# Case A: 필드 수가 많음 (>128)
HLEN big_hash  # 만약 500이면 → hashtable 오버헤드
# 해결: Hash Sharding (big_hash:1, big_hash:2...)

# Case B: 값 크기가 큼 (>64 bytes)
# 해결: 값을 별도 String으로 분리
#   HSET big_hash large_field "REF:large_field:1"
#   SET large_field:1 <actual_value>

# Case C: 필드 이름이 김
# 해결: 필드 이름 단축 (name→n, value→v 등)
```

**구체적 최적화 전략:**

1. **필드 수 축소**: 128개 이하로 Hash를 논리적 그룹으로 분리
2. **값 외부화**: 64 bytes 초과 값은 별도 String 키로 분리, Hash에는 참조만 저장
3. **필드명 단축**: 긴 필드명을 짧게 (코드에 매핑 테이블 관리)
4. **직렬화**: 여러 작은 필드를 JSON으로 묶어 단일 필드로 저장 (단, 개별 업데이트 불가)

**변경 효과 측정:**
```bash
# 변경 전후 비교
BEFORE=$(redis-cli MEMORY USAGE big_hash)
# ... 변경 작업 ...
AFTER=$(redis-cli MEMORY USAGE big_hash_new)
echo "절약: $(( (BEFORE - AFTER) * 100 / BEFORE ))%"
```

</details>

---

**Q3.** 애플리케이션에서 Redis의 특정 키가 `embstr`로 저장되길 기대하고 코드를 작성했는데, 운영 중 갑자기 메모리가 늘었다. `OBJECT ENCODING`으로 확인하니 `raw`로 변경됐다. 어떻게 이 상황이 발생했는가?

<details>
<summary>해설 보기</summary>

`embstr → raw` 전환이 발생하는 조건:

**1. 값의 길이가 44 bytes를 초과한 경우**
```bash
# 처음엔 짧은 값 (embstr)
SET user:name "Alice"          # 5 bytes → embstr

# 나중에 긴 값으로 SET
SET user:name "Alice Wonderland Johnson III"  # 34 bytes → 여전히 embstr
SET user:name "Alice Wonderland Johnson III Extended"  # 45 bytes → raw!
```

**2. APPEND로 값이 수정된 경우**
```bash
SET user:name "Alice"    # embstr
APPEND user:name " Jr."  # embstr → raw (수정됐으므로)
# 이후 "Alice Jr."는 9 bytes임에도 raw 유지
```

**3. SETRANGE로 수정된 경우**
```bash
SET user:name "Alice"
SETRANGE user:name 0 "a"  # embstr → raw
```

**운영 중 자연 발생 시나리오:**
- 값의 내용이 점진적으로 길어지는 경우 (예: 유저 이름 + 추가 정보 합치기)
- 로그나 설명 필드를 `APPEND`로 누적하는 패턴
- 외부 데이터(API 응답)를 그대로 캐싱하는데 응답 크기가 달라진 경우

**해결:**
- 값이 길어질 수 있다면 처음부터 `raw`를 예상하고 메모리 계획
- 44 bytes 경계를 넘지 않도록 값 크기 제한
- `APPEND` 대신 `GET` + `SET` (새 값으로 교체 → 다시 최적 인코딩 결정)

```bash
# APPEND (embstr → raw, 되돌아오지 않음)
APPEND key "suffix"

# GET+SET 대안 (새 값으로 SET → 인코딩 재결정)
VALUE=$(redis-cli GET key)
redis-cli SET key "${VALUE}suffix"  # 길이에 따라 embstr 또는 raw
```

</details>

---

<div align="center">

**[⬅️ 이전: Stream Consumer Group](./06-stream-consumer-group.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — Persistence ➡️](../persistence/01-rdb-snapshot.md)**

</div>
