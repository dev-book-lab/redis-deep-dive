# Set과 Sorted Set — intset / skiplist 내부

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Set의 `intset`이 정수를 압축 저장하고 O(log N) 탐색을 보장하는 원리는?
- 정수 Set에 문자열 하나 추가했을 때 왜 즉시 hashtable로 전환되는가?
- Sorted Set이 `skiplist + hashtable` 두 자료구조를 동시에 유지하는 이유는?
- 스킵 리스트(skip list)가 O(log N)을 보장하는 "확률적 레이어" 구조는?
- `ZADD`, `ZRANGEBYSCORE`, `ZRANK`의 내부 동작이 어떻게 다른가?
- `ZRANGEBYLEX`가 동작하기 위한 전제 조건은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Set의 `intset`은 정수를 압축 저장하고 이진 탐색으로 O(log N)을 보장한다. Sorted Set은 `skiplist + hashtable` 두 자료구조를 동시에 유지해 범위 쿼리와 O(1) 스코어 조회를 모두 지원한다. 이 구조를 모르면 인코딩 전환 순간에 메모리가 폭발하거나 올바른 자료구조를 선택하지 못한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 정수 Set에 문자열 하나 추가 → 메모리 폭발

  초기 설계: 권한 ID를 정수로 관리
    SADD user:1:perms 1 2 3 4 5
    → intset: ~30 bytes (매우 작음)

  나중에 특별 권한 추가:
    SADD user:1:perms "superadmin"
    → 즉시 hashtable 전환!
    → ~400 bytes (13배 증가)
    → 100만 유저 × 370 bytes 추가 = 370 MB 갑자기 증가
    → OBJECT ENCODING을 확인하지 않아서 원인 파악 불가

실수 2: Sorted Set을 Set처럼 쓰는 낭비

  코드:
    ZADD tags 0 "redis"    # score=0, 순서 필요 없음
    ZADD tags 0 "python"
    ZADD tags 0 "java"

  문제:
    실제로 순서가 필요 없음 (score 모두 0)
    하지만 skiplist + hashtable 두 구조 유지
    Set 대비 2~3배 메모리 사용
    → Set을 써야 할 자리에 Sorted Set 사용

실수 3: ZSCORE를 O(log N)이라고 착각

  "Sorted Set은 skiplist라서 모든 연산이 O(log N)이겠지"
  → ZSCORE는 실제로 O(1) (내부 hashtable 조회)
  → ZRANK는 O(log N) (skiplist span 누산)
  → 두 연산의 성능이 다름을 모르고 과도한 최적화 시도
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Set 타입 통일 원칙:
  정수 ID만 쓸 것이면 → 철저히 정수로 유지 (intset 유지)
  문자열이 섞일 가능성 있으면 → 처음부터 문자열로 통일 (listpack → hashtable)

  권한 ID 예:
    정수만: SADD perms 1 2 3    → intset (압축, O(log N))
    문자열 혼용 가능성: 처음부터 문자열
    → SADD perms "read" "write"  → listpack (128개 이하) 또는 hashtable

Set vs Sorted Set 선택:
  순서/스코어 불필요 → Set (메모리 50% 절약)
  랭킹/순서 필요 → Sorted Set (score 의미 있게 사용)

Sorted Set 핵심 명령어 복잡도:
  ZSCORE: O(1)     → hashtable 조회 (score 빠르게 필요 시)
  ZRANK:  O(log N) → skiplist 탐색 (순위 계산)
  ZADD:   O(log N) → skiplist 삽입
  ZRANGEBYSCORE: O(log N + K) → 범위 쿼리

실시간 랭킹 최적 패턴:
  ZINCRBY game:score 100 "user:1"    # 점수 증가 O(log N)
  ZREVRANGE game:score 0 9 WITHSCORES  # 상위 10명 O(log N + 10)
  ZREVRANK game:score "user:1"       # 내 순위 O(log N)
```

---

## 🔬 내부 동작 원리

### 1. Set 인코딩 — intset

```c
/* intset.h */
typedef struct intset {
    uint32_t encoding;  /* 정수 인코딩 타입 (2/4/8 bytes) */
    uint32_t length;    /* 원소 수 */
    int8_t contents[];  /* 실제 정수 배열 (정렬된 상태) */
} intset;
```

메모리 레이아웃:

```
intset: {1, 3, 5, 7, 100, 200, 500}  (7개 정수, 모두 int16 범위)

┌─────────────┬──────────────┬───────────────────────────────────────┐
│ encoding    │ length       │ contents[]                            │
│ INTSET_ENC  │ 7            │ [1][3][5][7][100][200][500]           │
│ _INT16      │              │  각 2 bytes, 오름차순 정렬                │
│ (4 bytes)   │ (4 bytes)    │  총 14 bytes                           │
└─────────────┴──────────────┴───────────────────────────────────────┘
총 크기: 4 + 4 + 14 = 22 bytes (7개 정수)

정수 인코딩 자동 업그레이드:
  모든 정수가 int16 (-32768~32767) 범위 → INTSET_ENC_INT16 (2 bytes/원소)
  int32 범위 초과 → INTSET_ENC_INT32 (4 bytes/원소)
  int64 범위 초과 → INTSET_ENC_INT64 (8 bytes/원소)

업그레이드 예시:
  SADD myset 1 2 3             → int16 인코딩
  SADD myset 100000            → 여전히 int16 (100000 < 32767? 아니면 int32로)
  
  실제: 100000 > 32767 → int32로 업그레이드!
  → 기존 1,2,3도 모두 4 bytes로 재저장 (업그레이드 = 전체 재할당)
  → 다운그레이드는 없음 (int32 → int16으로 돌아가지 않음)

이진 탐색 (Binary Search):
  contents[]가 항상 정렬된 상태 유지
  SISMEMBER myset 7:
    이진 탐색 → O(log N)
    [1,3,5,7,100,200,500] 중 7 탐색:
      mid=100 → 7 < 100 → 왼쪽 절반
      mid=5   → 7 > 5   → 오른쪽 절반
      mid=7   → 일치!
  
  SADD myset 50 (새 원소 삽입):
    이진 탐색으로 삽입 위치 찾기 → O(log N)
    [1,3,5,7,100,200,500] → 50은 7과 100 사이
    → 100,200,500을 한 칸씩 뒤로 이동 → 50 삽입 → O(N) 이동
    → 삽입 전체: O(N) (탐색 O(log N) + 이동 O(N))
```

### 2. intset → hashtable 전환 조건

```
전환 조건 (OR):
  ① 원소 수 > set-max-intset-entries (기본 512)
  ② 정수가 아닌 원소(문자열) 추가

조건 ②가 중요:
  SADD myset 1 2 3 4 5     → intset
  SADD myset "hello"       → 즉시 hashtable! (문자열 하나로 전환)
  
  이유: intset은 정수만 저장하는 압축 구조
        문자열이 섞이면 일반 해시테이블로 전환 불가피

전환 후 메모리:
  intset 5개 정수 (int16): 4 + 4 + 10 = 18 bytes
  hashtable 5개 원소:
    dictEntry × 5: 24 × 5 = 120 bytes
    버킷 배열 (8개): 8 × 8 = 64 bytes
    SDS × 5: ~(3+N+1) × 5 bytes
    총: ~250 bytes
  → 문자열 하나 추가로 14배 메모리 증가!

listpack 인코딩 (소형 문자열 Set):
  조건: 원소 수 ≤ set-max-listpack-entries (기본 128)
        원소 값 크기 ≤ set-max-listpack-value (기본 64 bytes)
        원소가 정수가 아닌 경우 (정수면 intset 우선)
  
  → 소형 문자열 집합에도 listpack 사용 가능
  → 128개 이하 문자열 Set → listpack (연속 메모리)
  → 129개 이상 → hashtable 전환
```

### 3. Sorted Set — 왜 두 자료구조를 동시에 유지하는가

```
Sorted Set이 제공하는 두 가지 쿼리 패턴:

쿼리 유형 A: 스코어 기준 범위 조회
  ZRANGEBYSCORE ranking 1000 2000  # 스코어 1000~2000 멤버
  ZREVRANGE ranking 0 9            # 스코어 높은 순 상위 10명
  → 정렬된 순서로 탐색 필요 → 스킵 리스트

쿼리 유형 B: 특정 멤버의 스코어/순위 조회
  ZSCORE ranking "user:1"          # user:1의 스코어
  ZRANK ranking "user:1"           # user:1의 순위
  → 멤버 이름으로 O(1) 탐색 필요 → hashtable

두 패턴을 하나의 자료구조로는 불가:
  스킵 리스트만: ZSCORE에 O(N) 탐색 필요 (멤버 이름으로 검색 = 전체 순회)
  hashtable만:  ZRANGEBYSCORE에 O(N log N) 정렬 필요 (정렬 없음)

해결: skiplist + hashtable 동시 유지
  skiplist: 멤버를 스코어 순서로 저장 → 범위 쿼리 O(log N + K)
  hashtable: 멤버 → 스코어 매핑 → ZSCORE O(1)
  
  메모리 대가: 동일 데이터를 두 구조에 중복 저장
  → 멤버 SDS는 공유 (포인터로 참조)
  → 스코어는 두 구조에 각각 저장

zset 구조체:
  typedef struct zset {
      dict *dict;       /* 멤버 → 스코어 해시테이블 */
      zskiplist *zsl;   /* 스코어 순서 스킵 리스트 */
  } zset;
```

### 4. 스킵 리스트 — O(log N) 확률적 레이어 구조

```
스킵 리스트 기본 아이디어:
  정렬된 연결 리스트에서 탐색은 O(N)
  → "고속 레인"을 여러 층 쌓아 이진 탐색처럼 건너뜀

구조 (스코어: 10 20 30 40 50 60 70 80 90 100):

Level 3 (가장 드문): ────────────────────────────────────── 100
Level 2:          10 ──────────── 40 ──────── 70 ──────── 100
Level 1:          10 ──── 20 ──── 40 ── 50 ── 70 ──── 90 ── 100
Level 0 (전체):   10 - 20 - 30 - 40 - 50 - 60 - 70 - 80 - 90 - 100

각 노드는 자신이 속하는 레이어까지 포인터를 보유

탐색: 스코어 60 찾기
  Level 3: HEAD → 100 (60 < 100, 60은 왼쪽)
           → 더 내려감
  Level 2: HEAD → 10 (60 > 10) → 40 (60 > 40) → 70 (60 < 70)
           → 40 노드에서 더 내려감
  Level 1: 40 → 50 (60 > 50) → 70 (60 < 70)
           → 50 노드에서 더 내려감
  Level 0: 50 → 60 → 발견!
  
  총 이동: 1+3+2+1 = 7번 (10개 원소에서)
  이진 탐색과 유사한 O(log N) 동작

```c
/* Redis 스킵 리스트 노드 */
typedef struct zskiplistNode {
    sds ele;              /* 멤버 이름 (SDS) */
    double score;         /* 정렬 기준 스코어 */
    struct zskiplistNode *backward; /* 이전 노드 (Level 0에서 역방향) */
    struct zskiplistLevel {
        struct zskiplistNode *forward; /* 다음 노드 포인터 */
        unsigned long span;            /* 이 포인터가 건너뛰는 노드 수 */
    } level[];            /* 가변 길이 레이어 배열 */
} zskiplistNode;
```

레이어 높이 결정 (확률적):
  새 노드 삽입 시 레이어 높이를 무작위 결정
  P = 0.25 (ZSKIPLIST_P): 다음 레이어로 올라갈 확률
  
  높이 1: 항상 (100%)
  높이 2: 25% 확률
  높이 3: 6.25% 확률
  높이 4: 1.5625% 확률
  최대 높이: 32 (ZSKIPLIST_MAXLEVEL)
  
  기대 레이어 높이: 1/(1-P) = 1/0.75 = 1.33
  → 대부분 노드는 낮은 레이어 → 포인터 오버헤드 최소

span 필드의 활용:
  각 포인터의 span = 건너뛰는 Level 0 노드 수
  ZRANK 계산: HEAD에서 목표 노드까지 span 합산 → O(log N)
  
  예: HEAD→Level2→40 노드의 span = 3 (10,20,30,40 중 3 건너뜀)
      HEAD→Level2→70 노드의 span = 3 (40,50,60,70 중 3 건너뜀)
      ZRANK "40" = 3 (0-indexed: 순위 3)

삽입 과정 (ZADD ranking 55 "user:new"):
  1. 삽입 위치 탐색 (스킵 리스트 탐색) → O(log N)
  2. 새 노드 레이어 높이 무작위 결정
  3. 각 레이어에서 포인터 연결 → O(레이어 수) = O(log N)
  4. span 업데이트
  5. hashtable에 (멤버 → 스코어) 추가 → O(1)
  총: O(log N)

삭제 과정 (ZREM ranking "user:new"):
  1. 스킵 리스트에서 노드 탐색 + 제거 → O(log N)
  2. hashtable에서 제거 → O(1)
  총: O(log N)
```

### 5. Sorted Set 주요 명령어 내부 동작

```
ZADD key score member:
  hashtable: member → score 추가/갱신 O(1)
  skiplist: score 기준 위치 탐색 후 삽입 O(log N)
  총: O(log N)

ZSCORE key member:
  hashtable에서 member → score 직접 조회 O(1)
  skiplist 탐색 없음

ZRANK key member:
  skiplist를 HEAD부터 member까지 탐색하며 span 합산
  O(log N)

ZREVRANK key member:
  zset.length - 1 - ZRANK(member)
  O(log N)

ZRANGE key start stop:
  start 위치까지 O(log N) 탐색
  start~stop 범위 원소 순서대로 반환 O(K)
  총: O(log N + K), K = 반환 원소 수

ZRANGEBYSCORE key min max:
  min 스코어 위치까지 O(log N) 탐색
  max 스코어까지 순서대로 반환 O(K)
  총: O(log N + K)

ZRANGEBYLEX key min max:
  전제 조건: 모든 멤버의 스코어가 동일해야 함
  (스코어가 다르면 알파벳 순서를 보장할 수 없음)
  스코어 동일 시 → skiplist가 멤버 이름으로 세컨더리 정렬
  → 알파벳 범위 탐색 O(log N + K)
  
  예: 스코어 모두 0 → ZRANGEBYLEX dict "[a" "[z" → 알파벳 사전

ZUNIONSTORE, ZINTERSTORE:
  O(N × K + M log M) — N = 입력 집합 수, K = 최대 크기, M = 결과 크기
  → 대형 Sorted Set 간 집합 연산은 비용이 큼
```

### 6. listpack 인코딩 (소형 Sorted Set)

```
임계값 이하의 Sorted Set은 listpack 사용:
  zset-max-listpack-entries 128 (기본값)
  zset-max-listpack-value   64  (bytes, 멤버 크기)

listpack 내 Sorted Set 저장 방식:
  [멤버1][스코어1][멤버2][스코어2]... (쌍으로 저장)
  스코어 기준 정렬 유지

탐색: O(N) 선형 (원소 수 적을 때 실용적)
전환: 원소 수 > 128 또는 멤버 크기 > 64 bytes → skiplist+hashtable

메모리 비교 (10개 멤버):
  listpack: ~500 bytes
  skiplist+hashtable: ~1200~2000 bytes
  → listpack이 2~4배 메모리 효율
```

---

## 💻 실전 실험

### 실험 1: intset 인코딩과 전환 관찰

```bash
# intset 생성
redis-cli DEL myset
redis-cli SADD myset 1 2 3 4 5
redis-cli OBJECT ENCODING myset      # intset
redis-cli MEMORY USAGE myset

# 정수 범위 업그레이드
redis-cli SADD myset 100000          # int32 범위
redis-cli OBJECT ENCODING myset      # intset (인코딩은 그대로, 내부 bytes 확장)
redis-cli DEBUG OBJECT myset         # encoding:intset

# 문자열 추가 → 즉시 hashtable
redis-cli SADD myset "hello"
redis-cli OBJECT ENCODING myset      # hashtable
redis-cli MEMORY USAGE myset         # 급증 확인!

# 다시 문자열 제거해도 intset으로 돌아가지 않음
redis-cli SREM myset "hello"
redis-cli OBJECT ENCODING myset      # 여전히 hashtable

# 원소 수 임계값 테스트
redis-cli DEL int_set
for i in $(seq 1 512); do redis-cli SADD int_set $i > /dev/null; done
redis-cli OBJECT ENCODING int_set    # intset (512개, 임계값)
redis-cli SADD int_set 513
redis-cli OBJECT ENCODING int_set    # hashtable (513개 초과)
```

### 실험 2: Sorted Set 내부 구조 확인

```bash
# 소형 (listpack)
redis-cli DEL ranking
redis-cli ZADD ranking 1500 "alice" 2300 "bob" 1800 "carol"
redis-cli OBJECT ENCODING ranking    # listpack
redis-cli MEMORY USAGE ranking

# 대형 (skiplist)
for i in $(seq 1 129); do
  redis-cli ZADD ranking $((RANDOM % 10000)) "user:$i" > /dev/null
done
redis-cli OBJECT ENCODING ranking    # skiplist
redis-cli MEMORY USAGE ranking

# skiplist 구조 간접 확인
redis-cli ZCARD ranking              # 멤버 수
redis-cli ZSCORE ranking "alice"     # O(1) — hashtable 조회
redis-cli ZRANK ranking "alice"      # O(log N) — skiplist 탐색
redis-cli ZREVRANK ranking "alice"   # O(log N)

# 범위 쿼리
redis-cli ZRANGE ranking 0 9 WITHSCORES          # 하위 10개
redis-cli ZREVRANGE ranking 0 9 WITHSCORES       # 상위 10개
redis-cli ZRANGEBYSCORE ranking 1000 2000 WITHSCORES  # 스코어 범위
redis-cli ZRANGEBYSCORE ranking -inf +inf LIMIT 0 10  # 전체 중 페이징
```

### 실험 3: 랭킹 시스템 구현

```bash
# 실시간 게임 랭킹
redis-cli DEL game:score

# 유저 점수 추가
redis-cli ZADD game:score 1500 "alice"
redis-cli ZADD game:score 2300 "bob"
redis-cli ZADD game:score 1800 "carol"
redis-cli ZADD game:score 3100 "dave"
redis-cli ZADD game:score 900 "eve"

# 상위 3명 조회 (ZREVRANGE)
redis-cli ZREVRANGE game:score 0 2 WITHSCORES
# 1) "dave" 2) "3100"
# 3) "bob"  4) "2300"
# 5) "carol" 6) "1800"

# alice의 순위 조회 (0-indexed, ZREVRANK = 내림차순)
redis-cli ZREVRANK game:score "alice"  # 2 (3등)
redis-cli ZINCRBY game:score 500 "alice"  # 점수 증가
redis-cli ZREVRANK game:score "alice"  # 순위 변화 확인

# 특정 점수 구간 유저 수
redis-cli ZCOUNT game:score 1500 2500  # 1500~2500점 유저 수

# 하위 n% 제거 (일정 기간 후 랭킹 정리)
redis-cli ZREMRANGEBYSCORE game:score -inf 999  # 1000점 미만 제거
```

### 실험 4: ZRANGEBYLEX로 사전 탐색

```bash
# 스코어 동일 (사전식 정렬 활용)
redis-cli DEL dict_set
for word in apple banana cherry date elderberry fig grape; do
  redis-cli ZADD dict_set 0 "$word" > /dev/null
done
redis-cli OBJECT ENCODING dict_set   # listpack (원소 적으면)

# 알파벳 범위 탐색
redis-cli ZRANGEBYLEX dict_set "[b" "[e"
# banana, cherry, date (b 이상 e 미만)

redis-cli ZRANGEBYLEX dict_set "[c" "+"
# cherry, date, elderberry, fig, grape (c 이상 모두)

# 자동완성 패턴
redis-cli ZADD autocomplete 0 "apple" 0 "application" 0 "apply" 0 "banana"
redis-cli ZRANGEBYLEX autocomplete "[app" "[app\xff"
# apple, application, apply (app로 시작하는 모든 것)
```

---

## 📊 성능 비교

```
Set 인코딩별 특성:

               intset           listpack         hashtable
  ──────────────────────────────────────────────────────────
  조건          정수만           문자열, ≤128     그 외
  탐색 SISMEMBER O(log N)        O(N)             O(1)
  삽입 SADD      O(N) 정렬유지   O(N) 정렬유지    O(1)
  메모리         최소            소               대
  연속 메모리    Yes              Yes              No

  intset 10개: ~30 bytes
  listpack 10개: ~200 bytes
  hashtable 10개: ~400 bytes

Sorted Set 인코딩별 복잡도:

               listpack              skiplist+hashtable
  ──────────────────────────────────────────────────────────
  ZADD           O(N)                 O(log N)
  ZSCORE         O(N)                 O(1) (hashtable)
  ZRANK          O(N)                 O(log N) (skiplist span)
  ZRANGE         O(N)                 O(log N + K)
  ZRANGEBYSCORE  O(N)                 O(log N + K)
  메모리          소 (연속)            대 (포인터 × 2)

  listpack 128개: ~5 KB
  skiplist 128개: ~15~20 KB
```

---

## ⚖️ 트레이드오프

```
intset 유지의 조건:
  ✓ 모든 원소가 정수 (또는 정수로 표현 가능)
  ✓ 원소 수 ≤ 512
  → 권한 ID, 카테고리 ID, 상태 코드 집합에 적합

skiplist의 설계 선택:
  "왜 AVL Tree나 Red-Black Tree 대신 Skip List인가?"
  
  AVL/RBT:
    삽입/삭제 시 트리 재균형 필요 (복잡한 회전 연산)
    구현 복잡도 높음
    캐시 비친화적 (트리 포인터 산재)
  
  Skip List:
    삽입/삭제가 확률적이라 재균형 없음
    구현 단순 (Redis 소스 zskiplist 약 400 라인)
    ZRANGE 같은 범위 쿼리에서 연결 포인터로 순회 효율적
    
  Antirez의 설명: "Skip list is a probabilistic data structure that
  allows for O(log N) average time complexity for search, insert, and
  delete. More importantly for Redis, it's much simpler to implement
  than a balanced BST."

Sorted Set의 메모리 비용:
  동일 멤버를 skiplist + hashtable 양쪽에 저장
  → 멤버 SDS는 공유 (하나의 포인터)
  → 스코어(double 8 bytes)가 두 곳에 저장
  → 범위 쿼리 성능을 위한 메모리 트레이드오프
```

---

## 📌 핵심 정리

```
Set 인코딩:
  intset: 정수만, ≤512개 → 정렬된 배열, 이진탐색 O(log N)
  listpack: 문자열, ≤128개 → 연속 메모리, 선형탐색 O(N)
  hashtable: 그 외 → O(1) 탐색
  
  정수 Set에 문자열 추가 → 즉시 hashtable (역전환 없음!)

Sorted Set 인코딩:
  listpack: ≤128개, 멤버≤64 bytes → 연속 메모리
  skiplist+hashtable: 그 외
    skiplist: 스코어 순 범위 쿼리 O(log N + K)
    hashtable: 멤버→스코어 O(1) 조회

스킵 리스트:
  확률적 레이어 (P=0.25)로 이진 탐색 근사
  span 필드로 ZRANK O(log N) 계산
  삽입/삭제: O(log N) (재균형 없음)

핵심 명령어:
  ZSCORE: O(1) (hashtable)
  ZRANK:  O(log N) (skiplist span)
  ZADD:   O(log N) (skiplist 삽입)
  ZRANGEBYSCORE: O(log N + K)
  ZRANGEBYLEX: 스코어 동일 전제 → 사전식 범위

진단:
  OBJECT ENCODING key     → intset/listpack/hashtable/skiplist
  MEMORY USAGE key        → 실제 메모리
  ZCARD key               → 원소 수
```

---

## 🤔 생각해볼 문제

**Q1.** `ZADD leaderboard 100 "alice" 200 "bob" 300 "carol"`로 생성한 Sorted Set에서 `ZSCORE leaderboard "alice"`와 `ZRANK leaderboard "alice"`의 내부 경로는 어떻게 다른가?

<details>
<summary>해설 보기</summary>

**ZSCORE leaderboard "alice":**
- `zset.dict` (hashtable)에서 `"alice"` 키로 직접 조회
- O(1) — 스킵 리스트 탐색 없음
- 반환: `100.0` (double)

**ZRANK leaderboard "alice":**
- `zset.zsl` (skiplist)에서 HEAD부터 `"alice"` 노드까지 탐색
- 탐색하면서 각 포인터의 `span` 값을 누산
- HEAD→...→"alice" 까지 span 합 = 순위 (0-indexed)
- O(log N)

실제로 `ZRANK`는 내부적으로 스코어를 먼저 hashtable에서 조회한 후, 그 스코어로 skiplist를 탐색하는 방식이 아닙니다. skiplist를 HEAD부터 직접 멤버 이름으로 탐색하며 span을 누산합니다. Redis의 skiplist는 노드에 멤버 이름(`sds ele`)이 있어 이름으로도 탐색 가능합니다.

</details>

---

**Q2.** `ZADD key 0 "apple" 0 "banana" 0 "cherry"`로 스코어가 모두 0인 Sorted Set을 만들었다. `ZRANGEBYSCORE key 0 0`과 `ZRANGEBYLEX key "[a" "[z"`의 반환 결과는 같은가?

<details>
<summary>해설 보기</summary>

**같은 원소를 반환하지만, 정렬 방식이 다를 수 있습니다.**

스코어가 모두 0으로 동일할 때 Redis skiplist는 **멤버 이름의 사전식(lexicographic) 순서**를 세컨더리 정렬 기준으로 사용합니다. 따라서:

- `ZRANGEBYSCORE key 0 0` → 스코어 0인 모든 원소를 사전식 순서로 반환 → `apple, banana, cherry`
- `ZRANGEBYLEX key "[a" "[z"` → a~z 범위 원소를 사전식 순서로 반환 → `apple, banana, cherry`

두 결과가 실제로 같습니다. 단, `ZRANGEBYSCORE`는 스코어가 다른 원소가 섞여 있어도 동작하고, `ZRANGEBYLEX`는 **모든 원소의 스코어가 동일해야만 의미 있는 결과**를 보장합니다. 스코어가 혼재하면 `ZRANGEBYLEX`의 결과가 예측 불가능합니다.

자동완성 기능을 구현할 때 `ZADD autocomplete 0 "apple" 0 "app"...`처럼 스코어를 0으로 통일하고 `ZRANGEBYLEX`를 사용하는 패턴이 일반적입니다.

</details>

---

**Q3.** 10만 명의 유저가 있는 실시간 랭킹에서 유저 한 명의 점수를 `ZINCRBY`로 업데이트할 때 내부에서 어떤 일이 일어나는가? 이 연산이 O(log N)인 이유는?

<details>
<summary>해설 보기</summary>

`ZINCRBY ranking 100 "alice"` 내부 동작:

1. **hashtable에서 현재 스코어 조회**: `"alice"` → 1500 (O(1))
2. **새 스코어 계산**: 1500 + 100 = 1600
3. **skiplist에서 "alice" 노드 제거**:
   - skiplist를 탐색해 "alice" 노드 위치 찾기: O(log N)
   - 해당 노드의 모든 레이어 포인터 제거
   - span 업데이트
4. **skiplist에 새 스코어로 재삽입**:
   - 스코어 1600의 올바른 위치 탐색: O(log N)
   - 새 레이어 높이 결정 (확률적)
   - 포인터 연결 및 span 업데이트
5. **hashtable 스코어 업데이트**: `"alice"` → 1600 (O(1))

총: O(log N) (skiplist 제거 + 재삽입이 지배)

10만 명 랭킹에서 log₂(100,000) ≈ 17 → 약 17단계의 포인터 이동으로 위치 결정. 이것이 실시간 랭킹에 Sorted Set이 적합한 이유입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Hash 내부 구조](./03-hash-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Bitmap, HyperLogLog, Geo ➡️](./05-bitmap-hyperloglog-geo.md)**

</div>
