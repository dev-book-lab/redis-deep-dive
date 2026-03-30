# String — SDS와 인코딩 전환

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- C의 `char*` 대신 SDS(Simple Dynamic String)를 만든 이유는?
- SDS의 `len` / `alloc` / `flags` / `buf` 각 필드는 무슨 역할인가?
- 사전 할당(pre-allocation) 전략이 재할당 횟수를 어떻게 줄이는가?
- `int` / `embstr` / `raw` 인코딩이 선택되는 조건과 메모리 구조 차이는?
- `APPEND` 명령어가 `embstr`을 `raw`로 전환하는 이유는?
- `SETRANGE`, `GETRANGE`가 O(N)인 이유는?

---

## 🔍 왜 이 개념이 중요한가

### String이 Redis에서 가장 기본 타입인 이유

```
Redis의 모든 것이 String 위에 있다:
  모든 키(key)        → SDS
  String 값           → SDS 또는 int
  Hash의 필드/값      → SDS
  List의 원소         → SDS
  Set의 원소          → SDS
  Sorted Set의 멤버   → SDS

"String이 느리면 Redis 전체가 느리다"

실무 영향:
  1. 메모리 최적화
     SET user:1:name "Alice"         → raw (5 bytes → sdshdr8)
     SET user:1:age 30               → int (ptr에 직접)
     SET session:token:xxxxx "data"  → embstr (짧으면) 또는 raw
     
     카운터를 INCR로 관리하면 int 인코딩 → 최소 메모리
     카운터를 SET "1", SET "2"으로 관리하면 embstr → 낭비

  2. APPEND 패턴의 함정
     redis-cli SET log ""
     for line in logs:
       redis-cli APPEND log line
     
     처음: embstr → APPEND 후: raw (한 번 수정되면 영구 raw)
     raw는 2회 malloc → embstr은 1회 malloc
     → 로그 누적에 APPEND 반복 시 메모리 단편화 누적
```

---

## 🔬 내부 동작 원리

### 1. C 문자열의 한계와 SDS의 탄생

```c
/* C 표준 문자열의 문제 */
char *s = "hello";
strlen(s);        // O(N) — 매번 '\0'을 찾아 순회
strcat(s, " world"); // 먼저 충분한 공간인지 모름 → Buffer Overflow 위험
                     // 이진 안전하지 않음 ('\0'이 있으면 거기서 종료)

/* SDS: Simple Dynamic String (sds.h) */
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;    /* 현재 문자열 길이 (1 byte) — O(1) 길이 조회 */
    uint8_t alloc;  /* 할당된 버퍼 크기 (헤더 제외) */
    unsigned char flags; /* 3 bits: SDS 타입 (sdshdr5/8/16/32/64) */
    char buf[];     /* 실제 문자열 데이터 + '\0' 터미네이터 */
};
```

SDS 타입별 헤더 크기:

```
sdshdr5:  flags(1) + buf[]   — 32 bytes 미만 문자열 (len이 flags에 내장)
sdshdr8:  len(1) + alloc(1) + flags(1) + buf[]  → 헤더 3 bytes
sdshdr16: len(2) + alloc(2) + flags(1) + buf[]  → 헤더 5 bytes
sdshdr32: len(4) + alloc(4) + flags(1) + buf[]  → 헤더 9 bytes
sdshdr64: len(8) + alloc(8) + flags(1) + buf[]  → 헤더 17 bytes

선택 기준: 문자열 길이에 따라 자동으로 최소 크기 헤더 선택
  0~31 bytes:   sdshdr5 (flags에 len 내장, alloc 필드 없음)
  32~255 bytes: sdshdr8
  256~65535 bytes: sdshdr16
  65536~4GB:    sdshdr32
  4GB+:         sdshdr64
```

메모리 레이아웃 (sdshdr8 예시: "hello"):

```
메모리 주소:  [   헤더 (3 bytes)   ] [  buf (6 bytes)  ]
             ┌───┬───┬────┬─────────────────────┐
             │ 5 │ 5 │0x01│ 'h''e''l''l''o''\0' │
             └───┴───┴────┴─────────────────────┘
              len alloc flags   buf[]

len = 5   (현재 문자열 길이)
alloc = 5 (할당된 버퍼 크기 — 아직 여유 없음)
flags = 0x01 (SDS_TYPE_8)

SDS 포인터는 buf[]의 시작을 가리킴:
  sds s = (sdshdr8*)ptr + 3;  // 헤더 건너뜀
  → C 표준 함수(printf, strcmp 등)와 호환 가능 (buf는 '\0' 종료)
  → s[-1] = flags, s[-2] = alloc, s[-3] = len 로 헤더 역참조
```

### 2. 사전 할당 전략 — 재할당 횟수 최소화

```c
/* SDS에 문자열 추가 시 (sdsMakeRoomFor) */

현재 len = 10, alloc = 10 (여유 없음)
→ APPEND로 5 bytes 추가 필요

할당 전략:
  새 필요 크기 = 10 + 5 = 15 bytes

  if (새 크기 < 1MB):
    new_alloc = 새 크기 × 2  → 15 × 2 = 30 bytes 할당
    → 30 - 15 = 15 bytes 여유 확보
  
  if (새 크기 >= 1MB):
    new_alloc = 새 크기 + 1MB  → 15MB + 1MB = 16MB 할당
    → 1MB 여유 확보

결과:
  첫 번째 APPEND: 15 bytes 필요 → 30 bytes 할당 (realloc 발생)
  두 번째 APPEND: 5 bytes → 여유 15 bytes 있음 → 재할당 없음
  세 번째 APPEND: 5 bytes → 여유 10 bytes 있음 → 재할당 없음
  네 번째 APPEND: 5 bytes → 여유 5 bytes 있음 → 재할당 없음
  다섯 번째 APPEND: 5 bytes → 여유 0 → 재할당!

C 표준 realloc 없이:
  매 APPEND마다 realloc → O(N) 복사 발생
  100번 APPEND → 100번 realloc → O(N²) 총 비용

SDS 사전 할당:
  100번 APPEND → ~log(N)회 realloc → 대폭 감소

len과 alloc의 분리:
  len: sdslen(s) → O(1) 길이 조회 (strlen의 O(N) 대체)
  alloc: sdsavail(s) = alloc - len → 여유 공간 O(1) 조회
  → 재할당 필요 여부를 즉시 판단 가능
```

### 3. int 인코딩 — 포인터 공간에 정수 저장

```
조건: 값이 long 범위 정수 (-2^63 ~ 2^63-1)

메모리 레이아웃:
┌──────────────────────────────────┐
│         redisObject (16 bytes)   │
│                                  │
│  type:    OBJ_STRING (0)         │
│  encoding: OBJ_ENCODING_INT (1)  │
│  lru:     ...                    │
│  refcount: 1                     │
│  ptr:     (void*)12345  ◄── 포인터 대신 정수값 직접! 
└──────────────────────────────────┘

장점:
  SDS 할당 없음 → malloc 0회
  총 메모리: redisObject 16 bytes만
  
  INCR, DECR 연산:
    ptr에서 정수 꺼내기 → +1 → 다시 ptr에 저장 → O(1)
    SDS 재할당 없음

공유 정수 (0~9999):
  ptr = shared.integers[N] → redisObject 자체도 공유
  → GET counter (counter=42) → 공유 객체 반환, 메모리 추가 사용 없음

int 인코딩의 APPEND 동작:
  SET count 42      → int 인코딩
  APPEND count " items"
  → 정수에 문자열 추가 → int 유지 불가
  → "42" + " items" = "42 items" → raw로 전환
  → SDS 새로 할당
```

### 4. embstr 인코딩 — 연속 메모리 최적화

```
조건: 문자열 길이 ≤ 44 bytes

메모리 레이아웃:
  단 1회 malloc으로 redisObject + SDS를 연속 할당:

┌────────────────────────────────────────────────────────┐
│              하나의 연속 메모리 블록 (64 bytes)              │
│                                                        │
│  ┌──────────────────┐ ┌──────────────────────────────┐ │
│  │  redisObject     │ │  sdshdr8 헤더 + buf           │ │
│  │  16 bytes        │ │  3 + len + 1 bytes           │ │
│  │  ptr ───────────►│ │  buf: "hello"                │ │
│  └──────────────────┘ └──────────────────────────────┘ │
└────────────────────────────────────────────────────────┘

계산:
  16 (redisObject) + 3 (sdshdr8 헤더) + 44 (buf max) + 1 ('\0') = 64 bytes
  → jemalloc 64 bytes 슬롯에 정확히 맞음!
  
  45 bytes 이상이면:
  16 + 3 + 45 + 1 = 65 bytes → 64 bytes 슬롯 초과
  → 128 bytes 슬롯 필요 → 비효율
  → raw로 전환 (별도 SDS 할당)

embstr의 immutable(불변) 특성:
  Redis는 embstr을 수정하는 API를 제공하지 않음
  수정 필요 시 (APPEND, SETRANGE):
    embstr → raw 변환 후 수정
    한 번 raw가 되면 다시 embstr로 되지 않음
    (strlen이 줄어도 자동 다운그레이드 없음)

embstr의 캐시 친화성:
  redisObject.ptr이 바로 다음 메모리를 가리킴
  → CPU가 redisObject 읽을 때 SDS도 같은 캐시 라인에 적재 가능
  → 별도 포인터 참조 없이 데이터 접근

44 bytes 경계는 Redis 버전마다 달랐음:
  Redis 3.x: 39 bytes (sdshdr 구조가 달랐음)
  Redis 4.x+: 44 bytes (현재)
```

### 5. raw 인코딩 — 일반 SDS

```
조건: 문자열 길이 ≥ 45 bytes, 또는 수정된 embstr

메모리 레이아웃:
  2회 malloc (redisObject 따로, SDS 따로):

  malloc 1:                  malloc 2:
  ┌──────────────────┐       ┌──────────────────────────────┐
  │  redisObject     │       │  sdshdr (len + alloc + flags)│
  │  16 bytes        │  ptr  │  + buf[] + '\0'              │
  │  ptr ────────────┼──────►│  가변 크기                     │
  └──────────────────┘       └──────────────────────────────┘

특징:
  수정 가능 (mutable): APPEND, SETRANGE 등 자유롭게 가능
  사전 할당: 빈번한 수정 시 재할당 최소화
  단편화 가능성: 두 블록이 메모리 다른 위치에 할당될 수 있음

문자열 길이별 SDS 헤더 자동 선택:
  SETEX largekey "$(python3 -c "print('x'*300)")" 300
  → sdshdr16 사용 (300 > 255이므로 sdshdr8 불가)
  → len/alloc이 각 2 bytes → 65535 bytes까지 표현
```

### 6. 인코딩 전환 정리

```
전환 조건 및 방향:

  SET key 42           → int    (정수)
    │
    ├─ APPEND key "x"  → raw    (정수+문자열)
    ├─ SET key 42.5    → embstr (부동소수점은 정수 아님)
    └─ SET key "hello" → embstr (문자열)

  SET key "hello"      → embstr (≤44 bytes)
    │
    ├─ APPEND key "!"  → raw    (수정됨 → 영구 raw)
    └─ SET key "longer_than_44_bytes_string_example_here" → raw

  방향: int or embstr → raw (역방향 없음)
  단, 새로운 SET은 처음부터 최적 인코딩 결정

명령어별 인코딩 영향:
  SET key value     : 항상 최적 인코딩으로 새로 시작
  APPEND key value  : embstr → raw, raw는 유지
  INCR/DECR key     : int 유지 (정수 범위 내)
  INCRBYFLOAT key N : 부동소수점 결과 → embstr 또는 raw
  SETRANGE key offset: embstr → raw
  GETSET key value  : 새 값으로 SET → 최적 인코딩
```

---

## 💻 실전 실험

### 실험 1: 인코딩 전환 전체 관찰

```bash
# int 인코딩
redis-cli SET n 42
redis-cli OBJECT ENCODING n      # int
redis-cli MEMORY USAGE n         # ~52 bytes (키 오버헤드 포함)

# long 범위 정수 최대값
redis-cli SET n 9223372036854775807  # 2^63-1
redis-cli OBJECT ENCODING n          # int
redis-cli SET n 9223372036854775808  # 2^63 초과
redis-cli OBJECT ENCODING n          # embstr (long 범위 초과)

# embstr 인코딩
redis-cli SET s "hello"
redis-cli OBJECT ENCODING s      # embstr
redis-cli MEMORY USAGE s         # ~70 bytes

redis-cli SET s44 "$(python3 -c "print('a'*44)")"
redis-cli OBJECT ENCODING s44    # embstr (정확히 44 bytes)
redis-cli MEMORY USAGE s44       # 64 bytes 슬롯

redis-cli SET s45 "$(python3 -c "print('a'*45)")"
redis-cli OBJECT ENCODING s45    # raw (45 bytes → 경계 초과)
redis-cli MEMORY USAGE s45       # 더 큰 슬롯

# raw 인코딩
redis-cli SET r "$(python3 -c "print('x'*100)")"
redis-cli OBJECT ENCODING r      # raw
redis-cli DEBUG OBJECT r
# encoding:raw serializedlength:... lru:... type:string

# embstr → raw 전환
redis-cli SET e "hello"
redis-cli OBJECT ENCODING e      # embstr
redis-cli APPEND e " world"
redis-cli OBJECT ENCODING e      # raw (수정됨)
redis-cli GET e                  # "hello world"
redis-cli APPEND e ""            # 빈 APPEND도 raw 유지
redis-cli OBJECT ENCODING e      # raw (되돌아가지 않음)
```

### 실험 2: SDS 사전 할당 관찰

```bash
# APPEND 반복 시 메모리 변화 (DEBUG OBJECT의 serializedlength로 간접 확인)
redis-cli DEL growing
redis-cli SET growing ""
redis-cli DEBUG OBJECT growing    # serializedlength:0

for i in $(seq 1 10); do
  redis-cli APPEND growing "hello-world-"  > /dev/null
  redis-cli MEMORY USAGE growing
done
# 처음 몇 번은 realloc 발생, 이후 여유 공간이 있으면 재할당 없이 진행

# alloc vs len 차이 확인 (DEBUG OBJECT의 serializedlength는 압축 크기)
redis-cli SET s "hello"
redis-cli DEBUG OBJECT s
# serializedlength:5 (실제 저장 데이터)
# MEMORY USAGE: 실제 메모리 (alloc 포함)
```

### 실험 3: 인코딩별 메모리 비교

```bash
# 1000개 정수 키 - int 인코딩
redis-cli FLUSHDB
for i in $(seq 1 1000); do redis-cli SET "int:$i" $i > /dev/null; done
redis-cli INFO memory | grep used_memory:
MEMORY_INT=$(redis-cli INFO memory | grep "^used_memory:" | awk -F: '{print $2}' | tr -d ' \r')

# 1000개 짧은 문자열 키 - embstr 인코딩
redis-cli FLUSHDB
for i in $(seq 1 1000); do redis-cli SET "emb:$i" "value-$i" > /dev/null; done
redis-cli INFO memory | grep used_memory:
MEMORY_EMB=$(redis-cli INFO memory | grep "^used_memory:" | awk -F: '{print $2}' | tr -d ' \r')

# 1000개 긴 문자열 키 - raw 인코딩
redis-cli FLUSHDB
for i in $(seq 1 1000); do redis-cli SET "raw:$i" "$(python3 -c "print('value-'*10)")" > /dev/null; done
redis-cli INFO memory | grep used_memory:
MEMORY_RAW=$(redis-cli INFO memory | grep "^used_memory:" | awk -F: '{print $2}' | tr -d ' \r')

echo "int 인코딩: $MEMORY_INT bytes"
echo "embstr 인코딩: $MEMORY_EMB bytes"
echo "raw 인코딩: $MEMORY_RAW bytes"
```

### 실험 4: INCR vs SET 카운터 메모리 비교

```bash
# INCR 카운터 (int 인코딩 유지)
redis-cli SET counter_incr 0
for i in $(seq 1 1000); do redis-cli INCR counter_incr > /dev/null; done
redis-cli OBJECT ENCODING counter_incr  # int
redis-cli MEMORY USAGE counter_incr     # ~52 bytes

# SET 카운터 (매번 embstr 재생성)
redis-cli SET counter_set 0
for i in $(seq 1 1000); do redis-cli SET counter_set $i > /dev/null; done
redis-cli OBJECT ENCODING counter_set   # int (결국 1000이니까)
# → 중간 과정에서 매 SET마다 새 객체 생성 → GC 부담

# 부동소수점 카운터
redis-cli SET float_counter 0
redis-cli INCRBYFLOAT float_counter 1.5
redis-cli OBJECT ENCODING float_counter  # embstr ("1.5")
```

---

## 📊 성능 비교

```
인코딩별 메모리 사용량 (키 제외):

인코딩    | 조건           | 메모리          | malloc 횟수
────────┼───────────────┼────────────────┼───────────
int     | 정수값          | 16 bytes 고정   | 0회
embstr  | ≤44 bytes     | 16+3+N+1 bytes | 1회
        |               | jemalloc 슬롯에 맞춤
raw     | ≥45 bytes     | 16 + SDS 별도   | 2회
        |               | SDS: 헤더+N+1

실제 메모리 (jemalloc 슬롯 반영):
  "hello" (5 bytes):
    embstr: 16+3+5+1=25 → jemalloc 32 bytes
    (절약: 단 1회 malloc, 캐시 친화)

  "x"×44 (44 bytes):
    embstr: 16+3+44+1=64 → jemalloc 64 bytes 정확히 맞음
    raw: 16 bytes + (3+44+1=48 bytes) = 64 bytes total (2회 malloc, 단편화 위험)

SDS 길이 조회 성능:
  C strlen:  O(N) — '\0' 찾을 때까지 순회
  SDS len:   O(1) — len 필드 직접 읽기
  → 길이 조회가 빈번한 연산에서 의미 있는 차이
```

---

## ⚖️ 트레이드오프

```
인코딩 선택 결과:

int:
  장점: 메모리 최소, O(1) 산술 연산
  단점: 정수에만 사용 가능
  → INCR/DECR 활용 시 항상 int 유지

embstr:
  장점: 1회 malloc, 캐시 친화, 16~64 bytes 슬롯 효율
  단점: 수정 불가 (APPEND 시 raw 전환)
  → 읽기 위주 짧은 문자열에 최적

raw:
  장점: 수정 가능, 크기 제한 없음, 사전 할당으로 realloc 최소화
  단점: 2회 malloc, 단편화 가능, 캐시 미스 가능성
  → 긴 문자열 또는 자주 수정되는 문자열

APPEND 패턴 주의:
  로그 누적에 APPEND 반복 → raw + 사전 할당으로 안전
  단, 처음 한 번은 embstr → raw 전환 비용 발생
  → SET "" 후 바로 APPEND 시작 → 처음부터 raw
```

---

## 📌 핵심 정리

```
Redis String 내부:

SDS 구조 (sdshdr8 예시):
  len(1) + alloc(1) + flags(1) + buf[]
  len:   O(1) 길이 조회
  alloc: 여유 공간 추적
  flags: SDS 타입 구분

사전 할당:
  새 크기 < 1MB → 새 크기 × 2 할당
  새 크기 ≥ 1MB → 새 크기 + 1MB 할당
  → APPEND 반복 시 realloc 횟수 O(log N)

인코딩 3가지:
  int:    정수 → ptr에 직접, malloc 0회
  embstr: ≤44 bytes → redisObject+SDS 연속 1회 malloc
  raw:    ≥45 bytes 또는 수정 → 별도 2회 malloc

전환:
  embstr 수정 → raw (단방향, 역방향 없음)
  SET로 새로 저장하면 다시 최적 인코딩

진단:
  OBJECT ENCODING key → 현재 인코딩
  MEMORY USAGE key    → 실제 바이트
  DEBUG OBJECT key    → 상세 정보 (serializedlength, lru_seconds_idle)
```

---

## 🤔 생각해볼 문제

**Q1.** `INCRBYFLOAT counter 1.5`를 반복 실행하면 인코딩이 어떻게 유지되는가? 그리고 이것이 `INCR`보다 비효율적인 이유는?

<details>
<summary>해설 보기</summary>

`INCRBYFLOAT`는 부동소수점 결과를 문자열로 저장합니다. 예를 들어, `1.5`, `3.0`, `4.5` 같은 값은 정수가 아니므로 **embstr 또는 raw 인코딩**이 됩니다. (만약 결과가 정수처럼 보여도 → `3.0`은 문자열 "3" 이 아닌 "3"으로 저장될 수 있음)

비효율 이유:
1. **매 연산마다 SDS 재할당**: 부동소수점 문자열 표현이 달라질 수 있어 매번 새 SDS 생성 가능
2. **파싱 비용**: GET 시 문자열 → 부동소수점 파싱 필요 (INCR의 정수 파싱보다 비쌈)
3. **메모리**: int 인코딩(16 bytes)과 달리 SDS 별도 할당 필요

카운터가 정수라면 **항상 `INCR`/`DECR`을 사용**하세요. 부동소수점이 반드시 필요한 경우에만 `INCRBYFLOAT`를 사용하되, 빈도를 고려하세요.

</details>

---

**Q2.** SDS의 `alloc`이 `len`보다 클 때 `DEBUG OBJECT key`의 `serializedlength`는 `alloc`과 `len` 중 무엇을 반영하는가?

<details>
<summary>해설 보기</summary>

`serializedlength`는 **`len`** (실제 문자열 길이) 기준으로 RDB 직렬화했을 때의 크기입니다. 사전 할당으로 인해 `alloc > len`이어도 직렬화 시에는 실제 데이터(`len` 바이트)만 기록합니다.

```bash
redis-cli SET s "hello"
redis-cli APPEND s " world extra extra"  # 총 23 bytes
# SDS 내부: len=23, alloc=46 (사전 할당으로 2배)
redis-cli DEBUG OBJECT s
# serializedlength:23  ← alloc(46)이 아닌 len(23) 기준
```

`MEMORY USAGE`는 실제 OS에 할당된 메모리(alloc + 헤더 등)를 반영하므로 `serializedlength`와 다를 수 있습니다. 실제 메모리는 `MEMORY USAGE`, 직렬화 크기는 `DEBUG OBJECT`의 `serializedlength`로 구분해서 확인하세요.

</details>

---

**Q3.** `SET key "hello"` 후 `SETRANGE key 0 "H"`를 실행하면 결과 인코딩은? 그리고 `SETRANGE key 100 "x"`처럼 현재 길이를 초과한 오프셋에 쓰면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

1. **`SETRANGE key 0 "H"`**: embstr인 "hello"를 수정 → **raw로 변환** 후 "Hello"로 변경. embstr은 immutable이므로 SETRANGE는 항상 raw 전환을 트리거합니다.

2. **`SETRANGE key 100 "x"`** (현재 길이 5): 현재 길이 5 → 오프셋 100 → 사이의 바이트(5~99, 총 95 bytes)는 **null bytes(`\0`)로 패딩**됩니다. 최종 길이 = 101 bytes. SDS의 이진 안전성 덕분에 중간의 null bytes가 문자열을 종료시키지 않습니다.

```bash
redis-cli SET key "hello"
redis-cli SETRANGE key 100 "x"
redis-cli STRLEN key          # 101
redis-cli GETRANGE key 5 100  # 95개의 \x00 + "x"
redis-cli OBJECT ENCODING key # raw
redis-cli MEMORY USAGE key    # 101 bytes 이상
```

이 패턴은 Bitmap 구현에도 활용됩니다 (`SETBIT` 명령어 내부에서 SETRANGE와 유사하게 동작).

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: List — quicklist ➡️](./02-list-quicklist.md)**

</div>
