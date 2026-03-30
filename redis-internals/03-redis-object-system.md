# Redis 객체 시스템 — redisObject 구조체

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Redis의 모든 값(String, List, Hash...)이 내부적으로 어떤 구조체로 표현되는가?
- `type`과 `encoding`은 어떻게 다르고, 왜 이 둘을 분리했는가?
- `embstr`과 `raw` 인코딩의 경계가 44 bytes인 이유는?
- 공유 정수 객체(0~9999)가 메모리를 절약하는 원리는?
- `OBJECT ENCODING`, `OBJECT REFCOUNT`, `OBJECT IDLETIME` 명령어로 무엇을 알 수 있는가?
- 인코딩이 자동 전환될 때 메모리와 CPU에 어떤 일이 일어나는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Redis의 모든 값은 `redisObject`라는 공통 구조체로 감싸진다. 같은 "string"이어도 내부 인코딩이 `int`, `embstr`, `raw`로 나뉘고, `hash`도 `listpack`과 `hashtable`로 나뉜다. 이 인코딩 차이가 메모리 사용량과 성능을 수 배 이상 바꾼다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Hash 메모리 과대/과소 추정

  질문: "Hash 100만 개, 필드 10개, 값 평균 20 bytes → 메모리 얼마?"
  잘못된 계산: 100만 × 10 × 20 = 2 GB
  실제 결과 (listpack): ~500 MB
  실제 결과 (hashtable): ~2~4 GB

  원리 모를 때:
    → 서버 sizing을 2 GB 기준으로 잡음
    → listpack이면 남아도는 메모리 낭비 (과다 비용)
    → 필드가 130개로 늘어나면 hashtable 전환 → 메모리 폭발 (예측 실패)

실수 2: APPEND로 String 인코딩이 영구 변경되는 것을 모름

  코드:
    redis.set("msg", "")
    redis.append("msg", "hello")  # embstr → raw 강제 전환
    redis.append("msg", " world") # raw 유지

  원리 모를 때:
    → "APPEND가 += 연산자처럼 동작하겠지"라고 가정
    → 실제로는 첫 APPEND에 embstr → raw 영구 전환
    → 이후 값이 다시 짧아져도 raw 유지 (2회 malloc, 단편화 가능)
    → 수백만 개 키에 이 패턴 적용 → 메모리 예측 불가

실수 3: OBJECT ENCODING을 확인하지 않고 메모리 최적화 시도

  "Hash를 더 작게 만들면 됩니다" → 필드 이름을 한 글자로 줄임
  → 정작 hashtable 인코딩인데 listpack으로 안 돌아감
  → 효과 없음 (이미 전환됨, 역방향 없음)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
인코딩 기반 메모리 예측:
  Hash 100만 개 × 10필드 × 20 bytes 값
  → 필드 수 10 < 128 (listpack 임계값)
  → 값 크기 20 < 64 bytes
  → listpack 유지 → ~500 MB (4배 절약)

  필드가 200개로 늘 경우:
  → hashtable 전환 → ~2~4 GB
  → 설계 단계에서 필드 분할 (Hash Sharding) 결정

인코딩 제어 설계:
  hash-max-listpack-entries 128  # 이 범위 안에서 설계
  hash-max-listpack-value 64     # bytes

카운터는 INCR/DECR (int 인코딩 유지):
  SET count "0"   → int (정수 감지)
  INCR count      → int 유지 (ptr에 직접 저장, malloc 0회)
  
  APPEND count "x" 하지 말 것
  → int → raw 전환 → 이후 산술 연산 불가

진단 순서:
  1. OBJECT ENCODING key     → 현재 인코딩 확인
  2. MEMORY USAGE key        → 실제 메모리 바이트
  3. DEBUG OBJECT key        → 직렬화 크기, idle time
  → 인코딩 확인 후 최적화 방향 결정
```

---

## 🔬 내부 동작 원리

### 1. redisObject 구조체 — 모든 값의 공통 표현

```c
/* Redis 소스코드 server.h */
typedef struct redisObject {
    unsigned type:4;      /* 외부에서 보이는 타입 (String/List/Hash/Set/ZSet) */
    unsigned encoding:4;  /* 실제 내부 인코딩 (how this object is stored) */
    unsigned lru:24;      /* LRU 시각 또는 LFU 빈도 카운터 */
    int refcount;         /* 참조 카운수 (공유 객체 관리) */
    void *ptr;            /* 실제 데이터를 가리키는 포인터 */
} robj;
```

메모리 레이아웃:

```
redisObject (16 bytes):
┌────────────────────────────────────────────────┐
│ type(4bit) │ encoding(4bit) │ lru(24bit)       │  4 bytes
├────────────────────────────────────────────────┤
│ refcount                                       │  4 bytes
├────────────────────────────────────────────────┤
│ ptr                                            │  8 bytes (64bit 시스템)
└────────────────────────────────────────────────┘
Total: 16 bytes per object (헤더만)

ptr이 가리키는 실제 데이터는 별도 메모리에 존재
단, embstr 인코딩은 예외 (아래 설명)
```

### 2. type vs encoding — 분리한 이유

```
type: Redis 사용자가 보는 외부 타입
  OBJ_STRING  = 0  → redis-cli TYPE key → "string"
  OBJ_LIST    = 1  → "list"
  OBJ_SET     = 2  → "set"
  OBJ_ZSET    = 3  → "zset"
  OBJ_HASH    = 4  → "hash"
  OBJ_STREAM  = 6  → "stream"

encoding: 실제 내부 저장 방식
  OBJ_ENCODING_RAW        = 0   SDS (일반 동적 문자열)
  OBJ_ENCODING_INT        = 1   정수를 ptr에 직접 저장
  OBJ_ENCODING_HT         = 2   hashtable
  OBJ_ENCODING_ZIPLIST    = 5   ziplist (연속 메모리)
  OBJ_ENCODING_INTSET     = 6   정수 집합 (압축)
  OBJ_ENCODING_SKIPLIST   = 7   skip list + hashtable
  OBJ_ENCODING_EMBSTR     = 8   embstr (짧은 문자열, redisObject와 연속)
  OBJ_ENCODING_QUICKLIST  = 9   quicklist (ziplist 노드의 linked list)
  OBJ_ENCODING_STREAM     = 10  radix tree + listpack
  OBJ_ENCODING_LISTPACK   = 11  listpack (ziplist의 후계자)

분리한 이유:
  같은 "list"라도 작은 list = ziplist, 큰 list = quicklist
  같은 "hash"라도 작은 hash = ziplist, 큰 hash = hashtable
  사용자 API는 동일 (HGET, HSET) → 내부 구현만 교체
  → 인터페이스 안정성 + 구현 최적화 동시 달성

확인 방법:
  redis-cli TYPE key          → 외부 타입 (string/list/hash/set/zset)
  redis-cli OBJECT ENCODING key → 내부 인코딩 (embstr/ziplist/hashtable...)
```

### 3. String 인코딩 3가지 — int / embstr / raw

```
String은 값에 따라 3가지 인코딩 중 하나로 자동 선택:

인코딩 1: int
  조건: 저장되는 값이 long 범위의 정수 (-2^63 ~ 2^63-1)
  원리: ptr 포인터(8 bytes) 공간에 정수 값을 직접 저장
        별도 SDS 메모리 할당 없음

  메모리 레이아웃:
  ┌────────────────────────────────────────────────┐
  │ type:OBJ_STRING │ encoding:OBJ_ENCODING_INT    │
  │ refcount: 1                                    │
  │ ptr: 12345      ← 포인터 대신 정수값 직접 저장        │
  └────────────────────────────────────────────────┘

  예시:
    SET counter 12345
    OBJECT ENCODING counter → "int"
    SET counter 99999999999999999999  ← long 범위 초과
    OBJECT ENCODING counter → "embstr" 또는 "raw"

인코딩 2: embstr (embedded string)
  조건: 문자열 길이 44 bytes 이하
  원리: redisObject와 SDS를 하나의 연속 메모리 블록에 할당
        단 한 번의 malloc으로 헤더 + 데이터 동시 할당
        캐시 지역성(cache locality) 극대화

  메모리 레이아웃:
  ┌─────────────────────────────────────────────────────────┐
  │    하나의 연속 메모리 블록 (단일 malloc)                       │
  │  ┌──────────────────┬──────────────────────────────┐    │
  │  │  redisObject     │  sdshdr8 + buf               │    │
  │  │  (16 bytes)      │  (5 bytes + 문자열 + '\0')    │     │
  │  └──────────────────┴──────────────────────────────┘    │
  └─────────────────────────────────────────────────────────┘

  장점:
    malloc 1회 (raw는 2회)
    메모리 단편화 최소
    CPU 캐시 적중률 향상 (redisObject와 데이터가 같은 캐시 라인에 존재 가능)
  
  단점:
    immutable (수정 불가)
    APPEND 등으로 수정 시 → 즉시 raw로 변환됨
    변환 후 다시 짧아져도 embstr로 돌아가지 않음

인코딩 3: raw (SDS)
  조건: 문자열 길이 45 bytes 이상, 또는 수정된 embstr
  원리: redisObject → ptr → SDS 별도 할당 (2회 malloc)

  메모리 레이아웃:
  ┌──────────────────┐    ┌─────────────────────────┐
  │  redisObject     │    │  SDS                    │
  │  (16 bytes)      │    │  sdshdr64               │
  │  ptr ───────────►│    │  len: 8 bytes           │
  └──────────────────┘    │  alloc: 8 bytes         │
                          │  flags: 1 byte          │
                          │  buf[]: 실제 문자열        │
                          └─────────────────────────┘

44 bytes 경계의 이유:
  jemalloc 크기 클래스: 64 bytes 슬롯
  redisObject = 16 bytes
  SDS sdshdr8 헤더 = 3 bytes (len:1 + alloc:1 + flags:1)
  null terminator = 1 byte
  16 + 3 + 44 + 1 = 64 bytes → jemalloc 64 bytes 슬롯에 정확히 맞음
  45 bytes 이상이면 → 64 bytes 슬롯 초과 → 별도 할당 필요 → raw
```

### 4. 공유 정수 객체 — 0~9999 메모리 절약

```
Redis 시작 시 0~9999 정수를 위한 redisObject를 미리 생성:

  shared.integers[0]  = redisObject(type=STRING, encoding=INT, ptr=0)
  shared.integers[1]  = redisObject(type=STRING, encoding=INT, ptr=1)
  ...
  shared.integers[9999] = redisObject(type=STRING, encoding=INT, ptr=9999)

SET key1 100 → ptr이 shared.integers[100]을 가리킴
SET key2 100 → ptr이 동일한 shared.integers[100]을 가리킴
SET key3 100 → ptr이 동일한 shared.integers[100]을 가리킴

  refcount = 3 (3개 키가 참조 중)
  실제 메모리: redisObject 1개 (16 bytes) + 키 3개의 포인터

공유 없이:
  key1→ redisObject(16bytes), key2→ redisObject(16bytes), key3→ redisObject(16bytes) = 48 bytes
공유 시:
  shared.integers[100] = 16 bytes + 포인터 3개 × 8 bytes = 40 bytes
  (절약량은 작지만 수백만 개의 카운터/ID 값 저장 시 의미 있음)

확인:
  redis-cli SET a 100
  redis-cli SET b 100
  redis-cli OBJECT REFCOUNT a  → 2147483647 (공유 객체는 refcount = INT_MAX, 절대 해제 안 됨)
  redis-cli OBJECT REFCOUNT b  → 2147483647

  redis-cli SET c "hello"
  redis-cli OBJECT REFCOUNT c  → 1 (일반 객체)

공유 범위:
  0~9999: 공유 (시작 시 생성)
  10000 이상: 공유 안 됨 (매 SET마다 새 객체)
  maxmemory-policy가 LFU일 때는 공유 객체 OBJECT FREQ 확인 불가
```

### 5. 인코딩 자동 전환과 비용

```
전환 방향: 항상 "압축 → 일반" 방향으로만 전환 (역방향 없음)

String:
  int ──(값이 정수 아닌 문자열)──► embstr ──(44 bytes 초과 또는 수정)──► raw
  ✗ 역방향 없음 (raw가 작아져도 embstr로 돌아가지 않음)

Hash:
  ziplist ──(필드수 > 128 또는 값크기 > 64 bytes)──► hashtable
  ✗ hashtable에서 다시 ziplist로 전환 없음

List:
  listpack ──(원소수 > 128 또는 값크기 > 64 bytes)──► quicklist
  ✗ 역방향 없음

Set:
  intset ──(정수 아닌 원소 추가 또는 원소수 > 512)──► hashtable
  ✗ 역방향 없음

ZSet:
  ziplist ──(원소수 > 128 또는 값크기 > 64 bytes)──► skiplist+hashtable
  ✗ 역방향 없음

전환 비용:
  전환 시 전체 데이터를 새 인코딩으로 복사
  → 이벤트 루프에서 실행 (단일 스레드)
  → 원소가 매우 많으면 일시적 지연 가능

  예: 원소 100개 Hash(ziplist) → 원소 129개째 추가 → hashtable로 전환
      100개 데이터 전체를 hashtable로 복사 → 완료 후 정상 응답
      원소가 10만 개면 전환 비용 수 ms → SLOWLOG에 기록될 수 있음

전환 후 메모리 변화:
  Hash (ziplist → hashtable):
    ziplist: 필드+값을 연속 메모리에 저장, 포인터 오버헤드 없음
    hashtable: 버킷 배열 + 각 entry의 key/value 포인터 + linked list
    → 동일 데이터 기준 4~10배 메모리 증가 가능

임계값 조정으로 ziplist 유지:
  hash-max-listpack-entries 128  # 기본값 (Redis 7.0+, 이전: hash-max-ziplist-entries)
  hash-max-listpack-value 64     # bytes 기준
  
  → 임계값을 낮추면: 더 빨리 hashtable 전환, 메모리 ↑ 속도 ↑
  → 임계값을 높이면: ziplist 유지, 메모리 ↓ 하지만 O(N) 선형 탐색
```

---

## 💻 실전 실험

### 실험 1: String 인코딩 전환 관찰

```bash
# int 인코딩
redis-cli SET count 100
redis-cli OBJECT ENCODING count  # → "int"

redis-cli SET big_int 9999999999999999999
redis-cli OBJECT ENCODING big_int  # → "embstr" (long 범위 초과)

# embstr 인코딩 (44 bytes 이하)
redis-cli SET short "hello"
redis-cli OBJECT ENCODING short  # → "embstr"

redis-cli SET exact44 "$(python3 -c "print('a'*44)")"
redis-cli OBJECT ENCODING exact44  # → "embstr"

# raw 인코딩 (45 bytes 이상)
redis-cli SET long45 "$(python3 -c "print('a'*45)")"
redis-cli OBJECT ENCODING long45  # → "raw"

# APPEND로 embstr → raw 강제 전환
redis-cli SET s "hello"
redis-cli OBJECT ENCODING s  # → "embstr"
redis-cli APPEND s " world"
redis-cli OBJECT ENCODING s  # → "raw" (수정됐으므로 embstr 불가)

# 공유 정수 확인
redis-cli SET a 9999
redis-cli SET b 9999
redis-cli OBJECT REFCOUNT a  # → 2147483647 (INT_MAX, 공유 객체)

redis-cli SET a 10000
redis-cli SET b 10000
redis-cli OBJECT REFCOUNT a  # → 1 (각자 별개 객체)
redis-cli OBJECT REFCOUNT b  # → 1
```

### 실험 2: Hash 인코딩 전환과 메모리 변화

```bash
# ziplist 상태에서 메모리 측정
redis-cli DEL myhash
for i in $(seq 1 10); do
  redis-cli HSET myhash "field$i" "value$i" > /dev/null
done
redis-cli OBJECT ENCODING myhash    # → "listpack" (Redis 7.0+) 또는 "ziplist"
redis-cli MEMORY USAGE myhash       # 작은 값

# 임계값(128)에 근접
for i in $(seq 11 128); do
  redis-cli HSET myhash "field$i" "value$i" > /dev/null
done
redis-cli OBJECT ENCODING myhash    # → 여전히 "listpack"
redis-cli MEMORY USAGE myhash

# 임계값 초과 → hashtable 전환
redis-cli HSET myhash "field129" "value129"
redis-cli OBJECT ENCODING myhash    # → "hashtable"
redis-cli MEMORY USAGE myhash       # 메모리 급증 확인!

# 값 크기로 전환 (64 bytes 초과)
redis-cli DEL bighash
redis-cli HSET bighash field1 "$(python3 -c "print('x'*65)")"
redis-cli OBJECT ENCODING bighash   # → "hashtable"
```

### 실험 3: OBJECT 명령어 전체 활용

```bash
redis-cli SET mykey "hello"

# 인코딩 확인
redis-cli OBJECT ENCODING mykey  # embstr

# 참조 횟수 확인 (일반 객체: 1, 공유 정수: INT_MAX)
redis-cli OBJECT REFCOUNT mykey  # 1

# 마지막 접근 이후 경과 시간 (초)
redis-cli OBJECT IDLETIME mykey  # 0 (방금 접근)
sleep 5
redis-cli OBJECT IDLETIME mykey  # 5

# LFU 빈도 확인 (LFU 정책 사용 시)
redis-cli CONFIG SET maxmemory-policy allkeys-lfu
for i in $(seq 1 100); do redis-cli GET mykey > /dev/null; done
redis-cli OBJECT FREQ mykey     # 접근 빈도 값

# DEBUG OBJECT: 더 상세한 정보
redis-cli DEBUG OBJECT mykey
# Value at:0x7f8a4c002b70 refcount:1 encoding:embstr
# serializedlength:5 lru:12345678 lru_seconds_idle:0 type:string
# serializedlength: RDB 직렬화 시 크기 (압축 후)
```

### 실험 4: 인코딩별 메모리 사용량 비교

```bash
# String: int vs embstr vs raw 메모리 비교
redis-cli SET int_val 42
redis-cli SET embstr_val "hello_world"
redis-cli SET raw_val "$(python3 -c "print('a'*100)")"

redis-cli MEMORY USAGE int_val     # ~52 bytes (키 포함)
redis-cli MEMORY USAGE embstr_val  # ~70 bytes
redis-cli MEMORY USAGE raw_val     # ~175 bytes

# Hash: listpack vs hashtable 메모리 비교
redis-cli CONFIG SET hash-max-listpack-entries 5  # 임계값 낮춤

redis-cli DEL h_small h_large
for i in $(seq 1 5); do redis-cli HSET h_small "f$i" "v$i" > /dev/null; done
for i in $(seq 1 6); do redis-cli HSET h_large "f$i" "v$i" > /dev/null; done

redis-cli OBJECT ENCODING h_small  # listpack
redis-cli OBJECT ENCODING h_large  # hashtable
redis-cli MEMORY USAGE h_small     # 소
redis-cli MEMORY USAGE h_large     # 대 (같은 데이터인데 더 큼)

# 설정 복원
redis-cli CONFIG SET hash-max-listpack-entries 128
```

---

## 📊 성능 비교

```
String 인코딩별 메모리:

  int:     16 bytes (redisObject만, ptr에 값 직접 저장)
  embstr:  16 + 3 + N + 1 bytes (N = 문자열 길이)
           예) "hello"(5 bytes): 16+3+5+1 = 25 bytes → jemalloc 32 bytes 슬롯
  raw:     16 bytes + 별도 SDS 할당 (2회 malloc)
           SDS 헤더: 8~25 bytes (문자열 길이에 따라 sdshdr8/16/32/64)

Hash 인코딩별 특성:

               listpack(ziplist)      hashtable
  ─────────────────────────────────────────────────
  저장 방식    연속 메모리             포인터 기반
  메모리 효율  높음 (포인터 오버헤드 없음) 낮음 (버킷+포인터)
  탐색 복잡도  O(N) 선형 스캔          O(1) 해시 탐색
  삽입 복잡도  O(N) (중간 삽입 시 복사)  O(1) 평균
  적합 크기   소형 (< 128 fields)     대형 (128+ fields)

  100개 fields, 값 평균 20 bytes 기준:
    listpack: ~3,000 bytes
    hashtable: ~12,000 bytes (4배 차이)
```

---

## ⚖️ 트레이드오프

```
인코딩 설계의 트레이드오프:

압축 인코딩 (listpack, intset):
  장점: 메모리 효율 최대, 연속 메모리로 캐시 친화적
  단점: 탐색이 O(N), 원소가 많아지면 느림
  적합: 소량의 데이터 (기본 임계값 128개 이하)

일반 인코딩 (hashtable, skiplist):
  장점: O(1) ~ O(log N) 빠른 탐색/삽입
  단점: 포인터 오버헤드로 메모리 증가, 캐시 미스 증가
  적합: 대량의 데이터

임계값 튜닝:
  임계값 높임 (예: hash-max-listpack-entries 256):
    listpack 유지 범위 확대 → 메모리 절약
    탐색이 O(256) 선형 → 성능 허용 범위인지 확인 필요
    단일 키에 데이터가 몰리는 Hot Key 패턴에서 위험

  임계값 낮춤 (예: hash-max-listpack-entries 64):
    빠르게 hashtable 전환 → 성능 우선
    메모리 사용 증가

임계값 조정 시 권장:
  변경 전 MEMORY USAGE로 현재 상태 측정
  변경 후 변화 측정
  프로덕션 적용 전 SLOWLOG로 성능 영향 확인
```

---

## 📌 핵심 정리

```
Redis 객체 시스템 핵심:

redisObject (16 bytes):
  type(4bit):      사용자가 보는 타입 (string/list/hash/set/zset)
  encoding(4bit):  내부 저장 방식 (int/embstr/raw/listpack/hashtable...)
  lru(24bit):      마지막 접근 시각 또는 LFU 빈도
  refcount(4bytes):참조 카운트 (공유 객체 관리)
  ptr(8bytes):     실제 데이터 포인터

String 인코딩 전환:
  int     → 정수값을 ptr에 직접 저장 (별도 메모리 없음)
  embstr  → 44 bytes 이하 문자열, redisObject와 연속 메모리 (1회 malloc)
  raw     → 45 bytes 이상 또는 수정된 문자열 (2회 malloc)

공유 정수 (0~9999):
  시작 시 미리 생성, 모든 참조가 같은 객체 공유
  OBJECT REFCOUNT → 2147483647 (INT_MAX)로 확인

인코딩 전환:
  항상 압축 → 일반 방향 (역방향 없음)
  전환 시 전체 데이터 복사 비용 발생
  임계값: hash-max-listpack-entries(128), hash-max-listpack-value(64)

진단 명령어:
  OBJECT ENCODING key   → 현재 인코딩
  OBJECT REFCOUNT key   → 참조 카운트
  OBJECT IDLETIME key   → 마지막 접근 후 경과 시간
  MEMORY USAGE key      → 실제 메모리 사용량 (bytes)
  DEBUG OBJECT key      → 상세 내부 정보
```

---

## 🤔 생각해볼 문제

**Q1.** `SET counter 0`을 실행한 후 `APPEND counter " items"`를 실행했다. `OBJECT ENCODING counter`는 어떻게 바뀌는가? 그리고 이후 `SET counter 0`으로 다시 설정하면?

<details>
<summary>해설 보기</summary>

1. `SET counter 0` → 인코딩: **int** (0은 정수)
2. `APPEND counter " items"` → 인코딩: **raw** (수정되어 embstr 불가, "0 items" = 7 bytes이므로 embstr 가능한 크기이지만 이미 수정되었으므로 raw)
3. `SET counter 0` → 인코딩: **int** (새로운 SET은 처음부터 인코딩을 결정)

핵심: `APPEND`는 기존 객체를 수정하므로 embstr → raw 전환이 발생합니다. 그러나 `SET`은 완전히 새 객체를 생성하므로 다시 최적 인코딩으로 시작합니다.

이것이 `INCR`이 권장되는 이유입니다. `INCR counter`는 int 인코딩을 유지하지만, `SET counter $(expr $(redis-cli GET counter) + 1)` 패턴은 불필요한 변환을 유발할 수 있습니다.

</details>

---

**Q2.** `hash-max-listpack-entries`를 기본값 128에서 512로 늘리면 어떤 이점이 있고 어떤 위험이 있는가?

<details>
<summary>해설 보기</summary>

**이점:**
- 필드 수 512개까지 listpack 유지 → 메모리 효율 유지 (hashtable 대비 1/4~1/8 메모리)
- 작은 Hash 수백만 개를 저장할 때 메모리 절약 효과가 큼

**위험:**
- listpack 탐색은 O(N) 선형 스캔 → 필드 512개 Hash에서 `HGET`이 최대 512번 비교
- 512개 Hash에 `HSET`으로 중간 삽입 시 메모리 이동 비용 발생
- **Hot Key 위험**: 특정 Hash에 모든 요청이 집중되면 O(512) 탐색이 이벤트 루프 차지

**권장 사용 조건:**
- Hash의 필드 수가 임계값에 근접하지 않는 경우 (예: 항상 50개 이하)
- `HGET`/`HSET`이 아닌 `HGETALL`을 주로 사용하는 경우 (어차피 전체 스캔)
- 트래픽이 분산되어 특정 Hash에 집중되지 않는 경우

변경 전 반드시 `SLOWLOG`로 `HGET`/`HSET` 응답 시간을 모니터링하세요.

</details>

---

**Q3.** `OBJECT REFCOUNT mykey`가 2147483647(INT_MAX)를 반환했다. 이 키를 `DEL mykey`하면 실제로 메모리가 해제되는가?

<details>
<summary>해설 보기</summary>

아니오. `refcount = INT_MAX`는 **공유 정수 객체(0~9999)** 임을 나타냅니다. 이 객체들은 Redis가 시작할 때 생성되어 프로세스 전체 생명주기 동안 유지되도록 설계되어 있습니다.

`DEL mykey`를 실행하면:
1. 키-값 매핑(dict entry)이 삭제됨 → 키 메모리 해제
2. 공유 정수 객체(`shared.integers[N]`)는 그대로 유지 (refcount 감소 없음)

공유 객체의 refcount가 INT_MAX인 이유는, 참조 카운트가 0이 되면 Redis가 자동으로 메모리를 해제하는데, 공유 객체는 절대 해제되면 안 되므로 감소하지 않는 특수 값(INT_MAX)으로 설정해 "절대 해제 금지" 플래그처럼 사용합니다.

```bash
redis-cli SET a 100
redis-cli DEL a
# shared.integers[100]은 여전히 존재, 다음 SET b 100에서 재사용됨
```

</details>

---

<div align="center">

**[⬅️ 이전: 메모리 관리](./02-memory-management.md)** | **[홈으로 🏠](../README.md)** | **[다음: 키 만료 메커니즘 ➡️](./04-key-expiry.md)**

</div>
