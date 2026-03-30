# Hash — listpack에서 hashtable로, 점진적 rehashing

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Hash가 listpack(ziplist)에서 hashtable로 전환되는 정확한 조건은?
- 점진적 rehashing이 두 개의 hashtable을 동시에 유지하는 방식은?
- rehashing 중에 `HGET`/`HSET`이 정상 동작하는 이유는?
- hashtable의 load factor와 rehashing 트리거 조건은?
- Hash를 "작게 유지"하는 것이 왜 메모리에 극적인 차이를 만드는가?
- Hash vs String 저장 방식의 메모리 차이는?

---

## 🔍 왜 이 개념이 중요한가

### "Hash가 listpack 범위를 넘는 순간 메모리가 폭발한다"

```
실무 상황: 사용자 프로필 캐싱

방법 1: 각 필드를 별도 String으로 저장
  SET user:1:name "Alice"
  SET user:1:age "30"
  SET user:1:email "alice@example.com"
  SET user:1:city "Seoul"
  → 4개 키 × redisObject(16) + SDS + 키 오버헤드 ≈ 400~600 bytes
  → 100만 유저 = 400~600 MB

방법 2: Hash로 묶어서 저장 (listpack 범위 내)
  HSET user:1 name "Alice" age "30" email "alice@example.com" city "Seoul"
  → 4 필드짜리 Hash (listpack) ≈ 100~150 bytes
  → 100만 유저 = 100~150 MB (String 방식의 25~37%)

  why? listpack: 포인터 오버헤드 없이 연속 메모리에 필드+값을 직접 저장
       키 오버헤드: Hash 키 1개 vs String 키 4개

방법 2의 함정: listpack 범위를 넘는 순간
  필드 수가 128을 초과하거나 값 크기가 64 bytes를 초과하면
  → hashtable로 전환 → 메모리가 2~4배로 증가!
  
  "작게 유지 = listpack = 메모리 절약"
  "한 번 크게 = hashtable = 다시 돌아올 수 없음"

이걸 모르면:
  Hash에 필드를 계속 추가하다가 128개를 넘는 순간
  메모리가 갑자기 2~4배 증가 → 원인 파악 불가
```

---

## 🔬 내부 동작 원리

### 1. listpack 인코딩 — 소형 Hash의 연속 메모리 표현

```
Hash의 listpack 저장 방식:
  필드와 값을 번갈아 연속 저장 (key-value-key-value...)

예시: HSET user:1 name "Alice" age "30"

listpack 메모리 레이아웃:
┌──────────┬──────────┬─────────────────────────────────────────────────┐
│ total    │ num-ele  │  entries                                        │
│ (4 bytes)│ (2 bytes)│                                                 │
│          │          │ [name 인코딩+값+backlen][Alice 인코딩+값+backlen]   │
│          │          │ [age 인코딩+값+backlen][30 인코딩+값+backlen]       │
└──────────┴──────────┴─────────────────────────────────────────────────┘

특징:
  포인터 없음: 이전/다음 필드를 포인터로 연결하지 않고 backlen으로 탐색
  연속 메모리: CPU 프리페치 효과, 캐시 미스 최소
  탐색: O(N) 선형 — 원소 수가 적을 때(≤128)는 실용적

탐색 과정 (HGET user:1 age):
  listpack 처음부터 순회
  entry[0] = "name" → 일치 않음
  entry[1] = "Alice" → 값 건너뜀
  entry[2] = "age"  → 일치!
  entry[3] = "30"   → 반환
  → O(4) 탐색 (4 entries)

임계값 (기본값):
  hash-max-listpack-entries 128  (Redis 7.0+)
  hash-max-listpack-value   64   (bytes)
  
  이전 명칭:
  hash-max-ziplist-entries 128
  hash-max-ziplist-value   64
```

### 2. hashtable 인코딩 — 대형 Hash의 O(1) 구조

```c
/* dict.h — Redis hashtable 구현 */
typedef struct dict {
    dictType *type;   /* 타입별 함수 포인터 (해시 함수, 비교 함수 등) */
    void *privdata;
    dictht ht[2];     /* ht[0]: 현재 테이블, ht[1]: rehashing 중 새 테이블 */
    long rehashidx;   /* -1: rehashing 아님, 0~N: rehashing 진행 위치 */
    int16_t pauserehash; /* rehashing 일시 중지 카운터 */
} dict;

typedef struct dictht {
    dictEntry **table;    /* 버킷 배열 (포인터 배열) */
    unsigned long size;   /* 버킷 수 (항상 2의 거듭제곱) */
    unsigned long sizemask; /* size - 1 (비트마스크로 버킷 인덱스 계산) */
    unsigned long used;   /* 저장된 키-값 쌍 수 */
} dictht;

typedef struct dictEntry {
    void *key;           /* 키 포인터 */
    union {
        void *val;       /* 값 포인터 */
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; /* 해시 충돌 체이닝 */
} dictEntry;
```

메모리 레이아웃:

```
dictht (버킷 배열):
  ┌───┬───┬───┬───┬───┬───┬───┬───┐  ← table[] (8개 버킷 예시)
  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │
  └─┬─┴─┬─┴───┴───┴─┬─┴───┴───┴───┘
    │   │           │
    ▼   ▼           ▼
  [key:"name"    [key:"city"    [key:"age"
   val:"Alice"]   val:"Seoul"]   val:"30"
   next:NULL]     next:NULL]     next:NULL]

해시 충돌 시 (두 키가 같은 버킷):
  버킷[3] → [key:"email" val:"alice@..." next:────►]
                                              [key:"phone" val:"010-..." next:NULL]
  → 체이닝(chaining)으로 연결

오버헤드 분석:
  dictEntry 1개: key 포인터(8) + val 포인터(8) + next 포인터(8) = 24 bytes
  버킷 배열: size × 8 bytes (포인터)
  
  10개 필드 Hash:
    dictEntry 10개 × 24 bytes = 240 bytes
    버킷 배열 (최소 16개) × 8 bytes = 128 bytes
    키/값 SDS 별도: 10 × (key_sds + val_sds) ≈ 수백 bytes
    
  같은 10개 필드를 listpack으로:
    ≈ 100~200 bytes (포인터 없음)
  
  → hashtable: listpack 대비 3~5배 메모리 사용
```

### 3. 점진적 rehashing — 두 테이블 공존의 원리

```
rehashing이 필요한 시점:
  ht[0].used / ht[0].size (load factor) > 1.0
  → 버킷보다 엔트리가 많음 → 충돌 증가 → 탐색 O(N) 저하
  → ht[1] 크기 = ht[0].size × 2 (확장)
  
  또는 특정 조건에서 축소:
  ht[0].used / ht[0].size < 0.1
  → 공간 낭비 → ht[1] 크기 = ht[0].used 이상 최소 2의 거듭제곱

점진적 rehashing 과정:

  t=0: 전환 시작
    rehashidx = 0  (ht[0]의 버킷 0번부터 이동 시작)
    
    ht[0]:  [bucket 0: A,B] [bucket 1: C] [bucket 2: D,E,F] ... (1000 버킷)
    ht[1]:  [empty] [empty] [empty] ... [empty]  (2000 버킷, 새로 할당)
  
  t=1: 명령어 실행마다 버킷 N개 이동 (일반적으로 1개씩)
    HSET hash newfield newval → 실행 후 rehashidx=0 버킷 이동
    ht[0] bucket[0]의 A, B → ht[1]로 재해시 후 이동
    rehashidx = 1
  
  t=2: 다음 명령어
    HGET hash field → 실행 후 rehashidx=1 버킷 이동
    ht[0] bucket[1]의 C → ht[1]로 이동
    rehashidx = 2
  
  ...반복...
  
  t=N: 모든 버킷 이동 완료
    ht[0] 해제
    ht[1] → ht[0]으로 교체
    ht[1] 초기화
    rehashidx = -1  (rehashing 완료)

rehashing 중 읽기/쓰기 동작:

  HGET hash field:
    ht[0]에서 먼저 탐색
    → 없으면 ht[1]에서 탐색
    (아직 이동 안 된 키는 ht[0], 이미 이동된 키는 ht[1])
  
  HSET hash newfield newval:
    새 키는 항상 ht[1]에 추가 (ht[0]에 추가하면 나중에 또 이동해야 하므로)
  
  HDEL hash field:
    ht[0]과 ht[1] 양쪽 모두에서 삭제 시도

rehashing 중 메모리:
  ht[0] (이전 데이터) + ht[1] (새 데이터) 동시 존재
  → 최대 2배 메모리 사용? 아님!
  
  실제: 이동 완료된 키는 ht[0]에서 제거
        ht[0]과 ht[1]의 총 엔트리 수 = 전체 엔트리 수 (동일)
        다만 두 버킷 배열이 동시 존재:
          ht[0] 버킷 배열: 1000 × 8 bytes = 8 KB
          ht[1] 버킷 배열: 2000 × 8 bytes = 16 KB
          합계: 24 KB (일시적 추가 메모리)
        → 실제 데이터 복사는 없음, 포인터만 이동 → 메모리 "2배"는 과장
```

### 4. rehashing 속도 조절 — 능동적 rehashing

```
수동적 rehashing (명령어 실행 시마다):
  매 HGET, HSET, HDEL 등 실행 시 rehashidx 버킷 1개 이동
  → 명령어가 없으면 rehashing도 멈춤

능동적 rehashing (serverCron):
  서버가 유휴 상태여도 rehashing 진행
  dictRehashMilliseconds(): 1ms 동안 최대한 rehashing
  → 이벤트 루프 Before Sleep hook에서 호출

rehash 가속 (고부하 시):
  activerehashing yes (기본값)
  → 이벤트 루프가 짧은 유휴 시간에도 rehashing 진행
  
  activerehashing no:
  → 명령어 실행 시에만 rehashing
  → 지연 시간 더 일정하지만 rehashing 완료까지 오래 걸릴 수 있음

load factor 임계값:
  일반: load_factor ≥ 1.0 → rehashing 시작 (확장)
  BGSAVE/BGREWRITEAOF 중: load_factor ≥ 5.0 → rehashing 시작
  이유: fork() 중 Copy-On-Write 사용 중
       rehashing = 버킷 이동 = 메모리 쓰기
       → COW 페이지 복사 발생 → 메모리 사용 급증
       → persistence 작업 중에는 rehashing 임계를 높여 메모리 부담 완화
```

### 5. Hash를 "작게" 유지하는 설계 패턴

```
패턴: 큰 Hash 하나 vs 작은 Hash 여러 개

유저 데이터 100만 명 저장 시:

방법 A: 유저당 하나의 큰 Hash
  HSET user:1 name "Alice" age "30" ... (200개 필드)
  → 200개 필드 > hash-max-listpack-entries(128) → hashtable!
  → 100만 Hash × hashtable 오버헤드 = 수십 GB

방법 B: 필드를 그룹으로 분할 (Sharding)
  HSET user:1:basic name "Alice" age "30" city "Seoul"  (3 필드)
  HSET user:1:stats login_count 42 last_login "2024-01"  (2 필드)
  HSET user:1:prefs theme "dark" lang "ko"  (2 필드)
  → 각 Hash가 listpack 유지 → 메모리 효율 최대
  → 조회 시 여러 명령어 필요 (Pipeline으로 해결)

방법 C: 필드 이름 단축
  HSET user:1 n "Alice" a "30" c "Seoul"  (3 필드, 단축 이름)
  → 필드 이름도 64 bytes 이하 유지 (hash-max-listpack-value)
  → 코드 가독성 vs 메모리 트레이드오프

임계값 조정으로 listpack 범위 확장:
  hash-max-listpack-entries 256  (기본 128 → 256)
  → 256개 필드까지 listpack 유지
  → HGET이 O(256) 선형 탐색 → 허용 가능한지 측정 필요
  → 256개 필드 Hash에 HGET을 초당 수천 번 하면 문제 될 수 있음

실험으로 확인:
  redis-cli HSET test_hash $(python3 -c "print(' '.join([f'field{i} value{i}' for i in range(128)]))")
  redis-cli OBJECT ENCODING test_hash  # listpack
  redis-cli HSET test_hash field128 value128
  redis-cli OBJECT ENCODING test_hash  # hashtable (전환!)
  redis-cli MEMORY USAGE test_hash     # 메모리 급증 확인
```

---

## 💻 실전 실험

### 실험 1: listpack → hashtable 전환 관찰

```bash
# 기본 설정 확인
redis-cli CONFIG GET hash-max-listpack-entries
redis-cli CONFIG GET hash-max-listpack-value

# 127개 필드 (listpack 유지)
redis-cli DEL test_hash
for i in $(seq 1 127); do
  redis-cli HSET test_hash "field$i" "value$i" > /dev/null
done
redis-cli OBJECT ENCODING test_hash   # listpack
redis-cli MEMORY USAGE test_hash
MEM_LIST=$(redis-cli MEMORY USAGE test_hash)

# 128번째 필드 추가 → 전환 트리거
redis-cli HSET test_hash "field128" "value128"
redis-cli OBJECT ENCODING test_hash   # hashtable!
redis-cli MEMORY USAGE test_hash
MEM_HASH=$(redis-cli MEMORY USAGE test_hash)

echo "listpack: $MEM_LIST bytes"
echo "hashtable: $MEM_HASH bytes"
echo "증가 배율: $(echo "scale=2; $MEM_HASH / $MEM_LIST" | bc)x"

# 값 크기로 전환 (64 bytes 초과)
redis-cli DEL value_test
redis-cli HSET value_test field1 "$(python3 -c "print('x'*64)")"
redis-cli OBJECT ENCODING value_test  # listpack (정확히 64 bytes)
redis-cli HSET value_test field2 "$(python3 -c "print('x'*65)")"
redis-cli OBJECT ENCODING value_test  # hashtable (65 bytes 초과)
```

### 실험 2: 점진적 rehashing 관찰

```bash
# 임계값 낮춰서 자주 rehashing 발생시키기
redis-cli CONFIG SET hash-max-listpack-entries 5

redis-cli DEL rh_test
for i in $(seq 1 5); do redis-cli HSET rh_test "f$i" "v$i" > /dev/null; done
redis-cli OBJECT ENCODING rh_test  # listpack

redis-cli HSET rh_test f6 v6       # 6번째 → hashtable 전환
redis-cli OBJECT ENCODING rh_test  # hashtable

# DEBUG RELOAD로 rehashing 강제 트리거 (내부 상태 확인)
redis-cli DEBUG RELOAD
redis-cli DEBUG OBJECT rh_test
# encoding:hashtable serializedlength:...

# INFO stats로 rehashing 통계
redis-cli INFO stats | grep rehash

# 임계값 복원
redis-cli CONFIG SET hash-max-listpack-entries 128
```

### 실험 3: Hash vs String 메모리 비교

```bash
# 방법 1: 각 필드를 별도 String으로
redis-cli FLUSHDB
for i in $(seq 1 10000); do
  redis-cli SET "user:$i:name" "User$i" > /dev/null
  redis-cli SET "user:$i:age" "$((20 + i % 50))" > /dev/null
  redis-cli SET "user:$i:city" "Seoul" > /dev/null
done
MEM_STRING=$(redis-cli INFO memory | grep "^used_memory:" | awk -F: '{print $2}' | tr -d ' \r')
echo "String 방식: $MEM_STRING bytes"

# 방법 2: Hash로 묶기 (listpack 범위 내)
redis-cli FLUSHDB
for i in $(seq 1 10000); do
  redis-cli HSET "user:$i" name "User$i" age "$((20 + i % 50))" city "Seoul" > /dev/null
done
MEM_HASH=$(redis-cli INFO memory | grep "^used_memory:" | awk -F: '{print $2}' | tr -d ' \r')
echo "Hash 방식 (listpack): $MEM_HASH bytes"
echo "절약: $(echo "scale=1; ($MEM_STRING - $MEM_HASH) * 100 / $MEM_STRING" | bc)%"
```

### 실험 4: HSCAN으로 안전한 대형 Hash 순회

```bash
# 대형 Hash 생성
redis-cli DEL large_hash
for i in $(seq 1 10000); do
  redis-cli HSET large_hash "field$i" "value$i" > /dev/null
done
redis-cli OBJECT ENCODING large_hash  # hashtable

# HGETALL은 O(N) — 대형 Hash에서 위험
# time redis-cli HGETALL large_hash | wc -l  # 이벤트 루프 블로킹!

# HSCAN으로 안전하게 순회 (커서 기반)
CURSOR=0
COUNT=0
while true; do
  RESULT=$(redis-cli HSCAN large_hash $CURSOR COUNT 100)
  CURSOR=$(echo "$RESULT" | head -1)
  COUNT=$((COUNT + $(echo "$RESULT" | tail -n +2 | wc -l) / 2))
  if [ "$CURSOR" = "0" ]; then break; fi
done
echo "총 필드 수: $COUNT"  # 10000
```

---

## 📊 성능 비교

```
Hash 인코딩별 메모리 비교 (10개 필드, 키 평균 10 bytes, 값 평균 20 bytes):

listpack:
  헤더: 6 bytes
  각 entry: encoding(1~5) + content + backlen(1~5)
  10 fields: ~(15 + 30) × 10 = ~450 bytes + 오버헤드
  실제: ~500~600 bytes (패딩 포함)

hashtable:
  dictEntry × 10: 24 × 10 = 240 bytes
  버킷 배열 (최소 16): 16 × 8 = 128 bytes
  키 SDS × 10: (3 + 10 + 1) × 10 = 140 bytes
  값 SDS × 10: (3 + 20 + 1) × 10 = 240 bytes
  redisObject × 20 (키+값): 16 × 20 = 320 bytes
  총: 240 + 128 + 140 + 240 + 320 = ~1,068 bytes

listpack 대비 hashtable: ~2배 메모리

연산 복잡도:
              listpack        hashtable
  ─────────────────────────────────────────
  HGET        O(N) 선형       O(1) 평균
  HSET        O(N) 탐색+삽입  O(1) 평균
  HDEL        O(N)            O(1)
  HLEN        O(1)            O(1)
  HGETALL     O(N)            O(N)
  HSCAN       O(N/iter)       O(N/iter)

N = 필드 수
listpack O(N): N ≤ 128이므로 최대 O(128) — 실용적
hashtable O(1): 모든 N에서 일정
```

---

## ⚖️ 트레이드오프

```
listpack 유지 (필드 ≤ 128, 값 ≤ 64 bytes):
  장점: 메모리 최소 (hashtable 대비 40~60% 절약)
  장점: 연속 메모리 → CPU 캐시 효율
  단점: HGET O(N) — 최대 O(128) 선형 탐색
  → 소형 Hash에 최적 (128개 이하 필드)

hashtable 사용 (필드 > 128 또는 값 > 64 bytes):
  장점: O(1) 모든 연산
  단점: 메모리 2~4배 증가
  단점: 포인터 산재 → 캐시 미스 가능성
  → 대형 Hash (필드 수백 개 이상)

점진적 rehashing의 장점:
  전체 잠금 없이 (Lock-free) 진행
  → rehashing 중에도 정상 응답
  → 단, 두 테이블 동시 탐색 → HGET이 두 번 탐색할 수 있음

임계값 확장 (hash-max-listpack-entries 256):
  listpack O(256) 탐색이 허용 가능한지 벤치마크로 확인 필요
  일반적으로 HGET 위주 + 필드 수 < 200 → 확장 허용
  HGETALL 위주 → N이 커져도 어차피 O(N)이라 임계값 확장 효과 미미
```

---

## 📌 핵심 정리

```
Redis Hash 내부:

listpack (소형):
  필드와 값을 번갈아 연속 메모리 저장
  탐색: O(N) 선형, 메모리 효율 최대
  전환 조건: 필드 수 > 128 또는 값 크기 > 64 bytes

hashtable (대형):
  dict 구조체 + 버킷 배열 + dictEntry 체이닝
  ht[0](현재) + ht[1](rehashing 중) 두 테이블
  O(1) 탐색, 메모리 효율 낮음

점진적 rehashing:
  load_factor ≥ 1.0 (또는 BGSAVE 중 ≥ 5.0) 시 시작
  매 명령어 실행마다 버킷 1개씩 이동
  rehashing 중 읽기: ht[0] → ht[1] 순 탐색
  rehashing 중 쓰기: 항상 ht[1]에 추가

메모리 최적화:
  Hash를 listpack 범위 내로 유지 → 2~4배 메모리 절약
  필드 수 분할 (Sharding): HSET user:1:basic vs HSET user:1

진단:
  OBJECT ENCODING hash     → listpack or hashtable
  MEMORY USAGE hash        → 실제 메모리
  HSCAN hash 0 COUNT 100   → 안전한 대형 Hash 순회 (HGETALL 대신)
```

---

## 🤔 생각해볼 문제

**Q1.** `hash-max-listpack-entries`를 128에서 512로 늘리면 HGET 성능이 어떻게 변하는가? listpack에서 O(512) 탐색이 실제로 문제가 될 수 있는 상황은?

<details>
<summary>해설 보기</summary>

HGET 탐색이 최악 O(512) 선형 스캔으로 늘어납니다. 초당 10만 번 HGET을 실행하는 서비스에서 각 HGET이 512번 비교한다면:
- 총 비교: 10만 × 512 = 5120만 번/초
- CPU 부담 증가 (단일 스레드 이벤트 루프에서 실행)

**문제가 될 수 있는 상황:**
1. 특정 Hash에 트래픽이 집중되는 Hot Key 패턴
2. HGET 빈도가 매우 높은 경우 (초당 수십만 회)
3. 512개 모두 탐색해야 하는 경우 (찾는 필드가 끝에 있을 때)

**문제가 안 되는 상황:**
- HGET 빈도가 낮거나 (초당 수백 회)
- 필드가 항상 앞쪽에 있어 조기 종료되는 경우
- HGETALL 위주 (어차피 O(N) 전체 순회)

**권장**: 임계값 변경 전 `redis-benchmark`로 HGET 응답 시간을 측정하고, SLOWLOG로 영향을 확인하세요.

</details>

---

**Q2.** 1000만 개의 Hash를 저장하는데 각 Hash의 필드 수가 정확히 128개다. `hash-max-listpack-entries`의 기본값이 128이라면 이 데이터는 listpack으로 저장되는가?

<details>
<summary>해설 보기</summary>

**아니오, hashtable로 저장됩니다.**

임계값 조건은 `>` (초과)가 아닌 `≥` (이상)으로 전환이 일어납니다. 128번째 필드를 추가할 때 (used = 128), `hash-max-listpack-entries = 128`이므로:

`used(128) > hash-max-listpack-entries(128)` 조건은 거짓이지만,

실제 Redis 소스코드(`t_hash.c`)는:
```c
if (hashTypeLength(o) > server.hash_max_listpack_entries)
    hashTypeConvert(o, OBJ_ENCODING_HT);
```
`>` 연산자를 사용하므로, 129번째 필드가 추가될 때 전환됩니다.

즉, **정확히 128개 필드라면 listpack 유지**됩니다. 128번째 필드를 추가하는 `HSET` 실행 후 `OBJECT ENCODING`을 확인하면 여전히 `listpack`입니다.

```bash
redis-cli DEL test
for i in $(seq 1 128); do redis-cli HSET test "f$i" "v$i" > /dev/null; done
redis-cli OBJECT ENCODING test  # listpack (128개 = 임계값, 전환 안 됨)
redis-cli HSET test f129 v129
redis-cli OBJECT ENCODING test  # hashtable (129개 > 128, 전환!)
```

</details>

---

**Q3.** BGSAVE(RDB 스냅샷)가 진행 중일 때 Redis에 대량의 HSET 요청이 쏟아지면 메모리가 급격히 증가하는 이유는? (rehashing과 fork Copy-On-Write의 상호작용)

<details>
<summary>해설 보기</summary>

두 가지 메커니즘이 겹쳐서 메모리를 증가시킵니다:

**1. Fork Copy-On-Write:**
BGSAVE가 `fork()`를 호출하면 자식 프로세스는 부모의 메모리 페이지를 공유합니다. 부모(Redis)가 데이터를 수정하면 해당 페이지만 복사(COW)됩니다. HSET이 대량으로 들어오면 → 많은 메모리 페이지가 쓰기 → 많은 페이지가 COW로 복사 → 메모리 증가

**2. rehashing + COW 상호작용:**
BGSAVE 중에 load_factor 임계값이 5.0으로 높아지는 이유가 바로 이것입니다:
- rehashing = 버킷 배열 수정 = 메모리 페이지 쓰기
- 쓰기 = COW 복사 발생
- 새로운 hashtable(ht[1]) 버킷 배열이 추가 메모리를 요구
- rehashing 중 ht[0] + ht[1] 두 버킷 배열이 동시 존재

즉, **BGSAVE 중 rehashing 발생 → 이중 메모리 증가**:
- COW로 인한 데이터 페이지 복사
- rehashing을 위한 새 hashtable 버킷 배열 할당

`maxmemory`를 설정할 때 이 시나리오를 고려해 물리 RAM의 50~60%로 낮게 설정하는 것이 권장되는 이유입니다.

</details>

---

<div align="center">

**[⬅️ 이전: List quicklist](./02-list-quicklist.md)** | **[홈으로 🏠](../README.md)** | **[다음: Set과 Sorted Set ➡️](./04-set-sortedset-internals.md)**

</div>
