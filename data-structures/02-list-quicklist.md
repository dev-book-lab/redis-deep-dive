# List — quicklist의 ziplist 블록 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Redis List가 ziplist → linkedlist → quicklist 순으로 진화한 이유는?
- quicklist가 "ziplist 노드의 이중 연결 리스트"라는 말의 의미는?
- `list-max-listpack-size`(구 `list-max-ziplist-size`) 설정이 청크 크기를 제어하는 방식은?
- `LPUSH`/`RPUSH`가 O(1)인 이유와 `LINDEX`가 O(N)인 이유는?
- List가 Queue로 쓰일 때 vs Stack으로 쓰일 때 성능 특성 차이는?
- listpack과 ziplist의 차이는?

---

## 🔍 왜 이 개념이 중요한가

### List 선택이 실무에서 틀리는 이유

```
흔한 실수:

  시나리오: 최근 활동 피드 (최신 100개만 유지)
  
  실수 코드:
    LPUSH activity:user:1 event1 event2 ...
    LRANGE activity:user:1 0 99    # 최신 100개 조회
    LLEN activity:user:1           # 크기 확인
    → 원소가 10만 개 쌓이면?
    
    LRANGE activity:user:1 0 99: O(100) → 여전히 빠름
    LLEN activity:user:1:        O(1) → 빠름
    하지만 메모리: 10만 개 × 원소 = 수십 MB
    
    올바른 설계:
      LPUSH + LTRIM으로 길이 유지
      redis-cli LPUSH activity:user:1 event
      redis-cli LTRIM activity:user:1 0 99  # 항상 100개만 유지
      → 크기 고정 → 메모리 예측 가능

  시나리오: 분산 작업 큐 (Producer-Consumer)
  
  실수:
    RPUSH queue task1
    LRANGE queue 0 -1  # 전체 조회해서 처리할 태스크 선택
    → O(N) 전체 스캔 + 네트워크 전송
    
  올바른 설계:
    RPUSH queue task1       # Producer
    BLPOP queue 0           # Consumer (블로킹 팝)
    → O(1) + 블로킹 대기 지원

원리를 알면 보이는 것:
  quicklist의 청크 구조 → 메모리 지역성과 포인터 오버헤드 트레이드오프
  LRANGE가 O(N)이지만 내부적으로 ziplist 청크를 연속 읽기 → 캐시 효율
  LINDEX가 O(N)인 이유 → Random Access를 쓰면 안 된다는 신호
```

---

## 🔬 내부 동작 원리

### 1. List 인코딩의 진화 역사

```
Redis 2.x: 두 개의 독립 인코딩
  ① ziplist: 원소 수 ≤ 128, 값 ≤ 64 bytes → 연속 메모리
  ② linkedlist: 임계값 초과 → 이중 연결 리스트 (포인터 per 원소)

  문제:
    linkedlist: 원소당 prev/next 포인터 2개 (각 8 bytes) = 16 bytes 오버헤드
    + listNode 구조체 8 bytes
    = 원소 하나에 최소 24 bytes 오버헤드 (실제 데이터 외)
    10만 원소 → 2.4 MB 포인터만으로 낭비
    
    ziplist 대형화: 단일 ziplist가 수MB → 수정 시 전체 realloc
    중간 삽입: O(N) 메모리 복사

Redis 3.2: quicklist 도입 (두 방식의 혼합)
  ziplist 블록들의 이중 연결 리스트
  각 노드 = 여러 원소를 담은 ziplist 블록(청크)
  → 포인터 오버헤드를 블록 단위로 분산
  → 단일 블록 수정 시 해당 블록만 realloc

Redis 7.0: listpack으로 ziplist 대체
  ziplist의 개선 버전 (하위 호환 유지)
  cascade update 문제 해결 (ziplist의 구조적 약점)
  quicklist의 내부 노드가 ziplist → listpack으로 변경
  → 현재: quicklist + listpack 조합
  (이 문서에서는 ziplist/listpack을 함께 "압축 블록"으로 설명)
```

### 2. quicklist 구조 — 세부 레이아웃

```c
/* quicklist.h */
typedef struct quicklist {
    quicklistNode *head;   /* 첫 번째 노드 포인터 */
    quicklistNode *tail;   /* 마지막 노드 포인터 */
    unsigned long count;   /* 전체 원소 수 */
    unsigned long len;     /* quicklistNode 수 (블록 수) */
    int fill:QL_FILL_BITS; /* 노드당 최대 원소 수 또는 최대 크기 */
    unsigned int compress:QL_COMP_BITS; /* 양 끝 제외 압축 노드 수 */
    unsigned int bookmark_count:QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;

typedef struct quicklistNode {
    struct quicklistNode *prev; /* 이전 노드 */
    struct quicklistNode *next; /* 다음 노드 */
    unsigned char *entry;       /* listpack (또는 ziplist) 데이터 */
    size_t sz;                  /* entry 크기 (bytes) */
    unsigned int count:16;      /* 이 노드의 원소 수 */
    unsigned int encoding:2;    /* RAW=1, LZF=2 (LZF 압축 여부) */
    unsigned int container:2;   /* PLAIN=1, PACKED=2 */
    unsigned int recompress:1;  /* 임시 압축 해제 여부 */
    unsigned int attempted_compress:1;
    unsigned int extra:10;
} quicklistNode;
```

메모리 구조 시각화:

```
quicklist (수백 개 원소 가정):

head ──► [quicklistNode 1]
         │ prev: NULL
         │ next: ──────────────────────────────────────────┐
         │ entry: ──► [listpack 블록]                       │
         │            ["item1","item2","item3",...,"item8"]│
         │ count: 8                                        │
                                                           │
         [quicklistNode 2] ◄────────────────────────────── ┘
         │ prev: ◄── node1
         │ next: ──────────────────────────────────────────┐
         │ entry: ──► [listpack 블록]                       │
         │            ["item9",...,"item16"]               │
         │ count: 8                                        │
                                                           │
         [quicklistNode 3] ◄────────────────────────────── ┘
         │ ...
         │
tail ──► [quicklistNode N]
         │ prev: ◄── node(N-1)
         │ next: NULL
         │ entry: ──► [listpack 블록]
         │            ["itemX",...,"itemLast"]

노드 1개당:
  quicklistNode 구조체: ~32 bytes (포인터 2 + entry 포인터 1 + 메타데이터)
  listpack 블록: 블록 내 원소들을 연속 메모리에 저장

포인터 오버헤드 비교:
  linkedlist: 원소당 24 bytes 오버헤드
  quicklist:  블록당 32 bytes 오버헤드 (블록에 8개 원소 → 원소당 4 bytes)
  → 동일 원소 수에서 quicklist가 6배 적은 포인터 오버헤드
```

### 3. listpack (ziplist 계승자) 내부 구조

```
ziplist 구조 (레거시, Redis 7.0 이전):
┌──────────┬──────────┬──────────┬───────────┬───────────┬─────┐
│ zlbytes  │ zltail   │ zllen    │ entry 1   │ entry 2   │ END │
│ (4 bytes)│ (4 bytes)│ (2 bytes)│           │           │0xFF │
└──────────┴──────────┴──────────┴───────────┴───────────┴─────┘

각 entry 구조:
┌──────────────────┬────────────────┬───────────┐
│ prevlen          │ encoding       │ content   │
│ (이전 항목 길이)    │ (타입+길이)       │ (실제 값)  │
│ 1 or 5 bytes     │ 1 or 5 bytes   │ N bytes   │
└──────────────────┴────────────────┴───────────┘

ziplist의 cascade update 문제:
  prevlen이 1 bytes로 표현 가능한 범위: 0~253 bytes
  이전 항목이 253 bytes → prevlen에 5 bytes 필요
  → 현재 항목의 prevlen이 1→5 bytes 증가
  → 현재 항목 크기 증가 → 다음 항목의 prevlen도 증가 가능
  → 연쇄적으로 전체 재할당 발생! (최악 O(N))

listpack 구조 (Redis 7.0+, cascade update 해결):
┌─────────────┬─────────────┬──────────────┬───────────┬────────┐
│total-bytes  │ num-elements│ entry 1      │ entry 2   │  END   │
│ (4 bytes)   │ (2 bytes)   │              │           │ 0xFF   │
└─────────────┴─────────────┴──────────────┴───────────┴────────┘

각 entry 구조 (prevlen 없음!):
┌──────────────┬───────────┬──────────────┐
│ encoding-len │ content   │ backlen      │
│ (타입+길이)    │ (실제 값)   │ (entry 전체 크기, 역방향 탐색용) 
└──────────────┴───────────┴──────────────┘

변화: prevlen(앞 항목 크기) → backlen(현재 항목 크기)
  backlen으로 역방향 탐색 가능 (이전 항목 찾기)
  cascade update 없음: 항목 크기 변화가 다른 항목에 영향 없음
```

### 4. LPUSH / RPUSH — O(1) 삽입

```
LPUSH mylist "new_item":

quicklist 구조:
  head → [node1: "a","b","c"] → [node2: "d","e","f"] → tail

head 노드(node1)에 여유가 있으면:
  listpack에 "new_item" 앞에 삽입 (연속 메모리 내 이동)
  count: 3 → 4
  O(M) (M = 블록 내 원소 수, 최대 fill 설정값)

head 노드가 꽉 찼으면 (fill=8이고 8개 모두 찼으면):
  새 quicklistNode 생성 → head로 설정
  새 listpack에 "new_item" 삽입
  O(1) (새 노드 생성, 포인터 연결)

RPUSH는 tail 노드에 동일하게 적용

결과:
  대부분의 경우 O(fill) → fill이 작으면 사실상 O(1)
  새 노드 생성 시도 O(1)

LPOP / RPOP:
  head/tail 노드의 listpack에서 첫/마지막 항목 제거
  블록이 비면 → 해당 quicklistNode 제거 → 포인터 갱신
  O(fill)
```

### 5. LINDEX — O(N) 선형 탐색

```
LINDEX mylist 50 (인덱스 50번째 원소):

quicklist 탐색 과정:
  1. head에서 시작
  2. 각 노드의 count를 누적 합산
     node1.count = 8  → 누적 8
     node2.count = 8  → 누적 16
     node3.count = 8  → 누적 24
     ...
     node7.count = 8  → 누적 56 > 50 → node7에 있음!
  3. node7의 listpack에서 (50 - 48 = 2)번째 항목 선형 탐색
     listpack 내 O(원소 수)

총 복잡도: O(노드 수 + 블록 내 탐색) = O(N/fill + fill)
  fill=8이면: O(N/8) 노드 순회 + O(8) 블록 탐색 = O(N/8) ≈ O(N)

인덱스 방향 최적화:
  음수 인덱스(-1, -2...): tail에서 역방향 탐색
  LINDEX mylist -1 → tail 노드의 마지막 원소 → O(1)~O(fill)
  LINDEX mylist 0  → head 노드의 첫 원소 → O(1)~O(fill)
  중간 인덱스: O(N/2) 평균

결론:
  LINDEX를 자주 쓴다면 List는 적합하지 않음
  → Hash (O(1)) 또는 Sorted Set (O(log N)) 고려
```

### 6. list-max-listpack-size 설정 완전 분석

```
설정명 (버전별):
  Redis 7.0+: list-max-listpack-size  (redis.conf)
  Redis 6.x 이전: list-max-ziplist-size

값의 의미:
  양수 값 (1, 2, 3, 4, 5, ...):
    각 listpack 노드가 담을 수 있는 최대 원소 수
    list-max-listpack-size 8  → 노드당 최대 8개 원소
  
  음수 값 (-1 ~ -5):
    각 listpack 노드의 최대 크기 (bytes)
    -1: 4 KB
    -2: 8 KB  (기본값)
    -3: 16 KB
    -4: 32 KB
    -5: 64 KB

기본값 -2 (8 KB):
  각 listpack 노드가 8 KB까지 커질 수 있음
  원소 크기에 따라 담는 개수가 달라짐
  20 bytes 원소: 8192 / 20 ≈ 409개
  100 bytes 원소: 8192 / 100 ≈ 81개

설정 선택 가이드:
  ┌─────────────────────────────┬─────────────────┬────────────────────────┐
  │ 워크로드                      │ 권장 설정         │ 이유                     │
  ├─────────────────────────────┼─────────────────┼────────────────────────┤
  │ LPUSH/RPOP 위주 큐            │ -2 (기본)        │ 양 끝만 접근, 블록 크기 무관 │
  │ 작은 원소 대량 저장              │ 128 (양수)       │ 원소 수 기준, 예측 가능     │
  │ 큰 원소 (1KB+)                │ -1 (4KB)        │ 큰 블록 방지              │
  │ 자주 수정 (중간 삽입/삭제)        │ 16~32 (양수)     │ 블록 작게, realloc 최소   │
  └─────────────────────────────┴─────────────────┴────────────────────────┘

list-compress-depth 설정:
  quicklist 양 끝을 제외한 중간 노드를 LZF 압축
  0: 압축 없음 (기본)
  1: 양 끝 노드 1개씩 제외 후 나머지 압축
  2: 양 끝 노드 2개씩 제외 후 나머지 압축
  
  → LPUSH/RPOP 위주 큐에서 중간 데이터 압축으로 메모리 절약
  → 단, 중간 접근 시 압축 해제 비용 발생 (CPU)
```

---

## 💻 실전 실험

### 실험 1: quicklist 구조 관찰

```bash
# 기본 List 생성
redis-cli RPUSH mylist a b c d e f g h  # 8개
redis-cli OBJECT ENCODING mylist         # listpack (적으면)
redis-cli LLEN mylist                    # 8
redis-cli DEBUG OBJECT mylist
# encoding:listpack serializedlength:... count 확인

# 원소 추가로 quicklist 전환
for i in $(seq 1 200); do
  redis-cli RPUSH mylist "item-$i" > /dev/null
done
redis-cli OBJECT ENCODING mylist         # quicklist
redis-cli DEBUG OBJECT mylist
# encoding:quicklist ql_nodes:26 ql_avg:8.00 ql_ziplist_max:-2

# ql_nodes: quicklistNode 수
# ql_avg: 노드당 평균 원소 수
# ql_ziplist_max: list-max-listpack-size 설정값
```

### 실험 2: LPUSH/RPOP vs LPOP/RPUSH 성능

```bash
# 큐 패턴 (FIFO): RPUSH + BLPOP
redis-cli DEL queue
for i in $(seq 1 10000); do redis-cli RPUSH queue "task-$i" > /dev/null; done
time redis-cli LLEN queue  # O(1)

# 10000개 팝
time (for i in $(seq 1 10000); do redis-cli LPOP queue > /dev/null; done)

# 스택 패턴 (LIFO): LPUSH + LPOP  
redis-cli DEL stack
for i in $(seq 1 10000); do redis-cli LPUSH stack "item-$i" > /dev/null; done
time (for i in $(seq 1 10000); do redis-cli LPOP stack > /dev/null; done)

# LINDEX 성능 비교 (O(N) 주의)
redis-cli DEL indexed_list
for i in $(seq 1 10000); do redis-cli RPUSH indexed_list "item-$i" > /dev/null; done

time redis-cli LINDEX indexed_list 0       # O(1) 앞쪽
time redis-cli LINDEX indexed_list 4999    # O(N/2) 중간
time redis-cli LINDEX indexed_list -1      # O(1) 뒤쪽
# 중간 인덱스가 훨씬 느림 확인
```

### 실험 3: LTRIM으로 고정 크기 List 유지

```bash
# 최근 100개 이벤트만 유지하는 패턴
redis-cli DEL activity

# 이벤트 추가 + 즉시 트림 (트랜잭션으로 원자적 실행)
for i in $(seq 1 1000); do
  redis-cli MULTI > /dev/null
  redis-cli LPUSH activity "event-$i" > /dev/null
  redis-cli LTRIM activity 0 99 > /dev/null      # 최신 100개만 유지
  redis-cli EXEC > /dev/null
done

redis-cli LLEN activity  # 100 (항상 100개)
redis-cli LRANGE activity 0 4  # 최신 5개 확인

# 메모리 확인 (고정 100개이므로 예측 가능)
redis-cli MEMORY USAGE activity
```

### 실험 4: list-max-listpack-size 조정 효과

```bash
# 기본 설정 (-2 = 8KB)
redis-cli CONFIG GET list-max-listpack-size
redis-cli DEL l_default
for i in $(seq 1 1000); do redis-cli RPUSH l_default "item-$i" > /dev/null; done
redis-cli DEBUG OBJECT l_default  # ql_nodes 확인

# 소형 블록으로 변경 (4개씩)
redis-cli CONFIG SET list-max-listpack-size 4
redis-cli DEL l_small
for i in $(seq 1 1000); do redis-cli RPUSH l_small "item-$i" > /dev/null; done
redis-cli DEBUG OBJECT l_small    # ql_nodes 증가 확인

# 메모리 비교
redis-cli MEMORY USAGE l_default
redis-cli MEMORY USAGE l_small

# 설정 복원
redis-cli CONFIG SET list-max-listpack-size -2
```

---

## 📊 성능 비교

```
List 연산별 시간 복잡도:

연산              | 복잡도       | 내부 동작
─────────────────┼────────────┼──────────────────────────────
LPUSH/RPUSH      | O(1)       | 양 끝 listpack에 삽입
LPOP/RPOP        | O(1)       | 양 끝 listpack에서 제거
LLEN             | O(1)       | quicklist.count 필드 읽기
LINDEX           | O(N)       | 노드 순회 + listpack 탐색
LRANGE           | O(S+N)     | S: 시작점 탐색, N: 반환 원소 수
LINSERT          | O(N)       | 삽입 위치 탐색 + listpack 수정
LSET             | O(N)       | 인덱스 탐색 + listpack 수정
LREM             | O(N)       | 전체 순회하며 일치 항목 제거
LTRIM            | O(N)       | 제거할 노드/항목 삭제

quicklist vs 단순 linkedlist 메모리:

100만 원소, 평균 값 크기 20 bytes:
  linkedlist:
    각 노드: prev(8) + next(8) + listNode(8) + SDS(20+8) = 52 bytes
    총: 52 × 1,000,000 = 52 MB
  
  quicklist (fill=8):
    quicklistNode: 32 bytes per node
    listpack당 원소 8개 × 20 bytes + 오버헤드 ≈ 200 bytes
    노드 수: 1,000,000 / 8 = 125,000 노드
    총: 125,000 × (32 + 200) = 29 MB
    → linkedlist 대비 44% 메모리 절약
```

---

## ⚖️ 트레이드오프

```
quicklist 설계의 트레이드오프:

블록 크기가 클 때 (list-max-listpack-size -5 = 64KB):
  장점: 노드 수 적음 → 포인터 오버헤드 최소
  단점: 중간 삽입/삭제 시 최대 64KB realloc + 메모리 복사
        단일 블록 수정이 다른 원소에 영향

블록 크기가 작을 때 (list-max-listpack-size 4):
  장점: 수정 영향 범위 최소 (4개짜리 블록만 realloc)
  단점: 노드 수 많음 → 포인터 오버헤드 증가, 탐색 시 노드 순회 증가

Redis 기본값 (-2 = 8KB)의 철학:
  "LPUSH/RPOP 큐 사용 패턴에서 양 끝만 접근 → 블록 크기 무관"
  "중간 접근 패턴이라면 List 대신 다른 자료구조"

List가 적합한 패턴:
  ✓ FIFO 큐 (RPUSH + BLPOP)
  ✓ LIFO 스택 (LPUSH + LPOP)
  ✓ 최근 N개 유지 (LPUSH + LTRIM)
  ✓ 타임라인 (양 끝에서만 접근)

List가 부적합한 패턴:
  ✗ 랜덤 인덱스 접근 (LINDEX 반복) → Hash
  ✗ 중간 삽입/삭제 빈번 → Sorted Set
  ✗ 멤버십 확인 (IS IN list) → Set
  ✗ 중복 없는 순서 집합 → Sorted Set
```

---

## 📌 핵심 정리

```
Redis List 내부:

진화:
  ziplist → linkedlist → quicklist (Redis 3.2) → quicklist+listpack (7.0)

quicklist 구조:
  이중 연결 리스트 of listpack 블록
  양 끝 접근: O(1), 중간 접근: O(N)
  포인터 오버헤드를 블록 단위로 분산

listpack (ziplist 후계자):
  cascade update 문제 해결
  entry = encoding-len + content + backlen
  역방향 탐색: backlen으로 이전 항목 크기 알 수 있음

설정:
  list-max-listpack-size -2 (기본: 8KB per 노드)
  list-compress-depth 0 (기본: 압축 없음)

핵심 연산:
  LPUSH/RPUSH/LPOP/RPOP: O(1) (양 끝)
  LINDEX: O(N) (사용 자제)
  LRANGE: O(S+N) (범위 조회)
  LTRIM: O(N) (고정 크기 유지 패턴)

큐 패턴:
  RPUSH queue task + BLPOP queue 0
  → BLPOP: 데이터 없으면 블로킹 대기 (폴링 없이)
  → 분산 작업 큐에 적합
```

---

## 🤔 생각해볼 문제

**Q1.** `LRANGE mylist 0 -1`과 `LRANGE mylist 0 9999`는 List에 원소가 10,000개 있을 때 성능이 같은가? `LLEN`을 먼저 확인하지 않고 `0 -1`을 쓰는 것의 위험은?

<details>
<summary>해설 보기</summary>

성능은 같습니다. `-1`은 마지막 인덱스이므로 `LRANGE mylist 0 -1`은 전체 반환이고, 10,000개가 있을 때 `LRANGE mylist 0 9999`와 동일합니다. 둘 다 O(N)입니다.

`0 -1` 사용의 위험: List가 **무한정 커질 수 있는 경우** `LRANGE mylist 0 -1`은 모든 원소를 반환합니다. 100만 원소 List에서 실행하면 수백 MB를 네트워크로 전송하게 됩니다. 이벤트 루프가 이 응답을 보내는 동안 다른 요청이 대기합니다.

안전한 패턴:
```bash
# 크기 확인 후 조건부 조회
LEN=$(redis-cli LLEN mylist)
if [ $LEN -lt 1000 ]; then
  redis-cli LRANGE mylist 0 -1
else
  redis-cli LRANGE mylist 0 999  # 최대 1000개만
fi
```

`LTRIM`으로 List 크기를 상한선으로 유지하는 것이 근본 해결책입니다.

</details>

---

**Q2.** `list-compress-depth 1`을 설정하면 어떤 List 접근 패턴에서 성능 저하가 발생하는가?

<details>
<summary>해설 보기</summary>

`list-compress-depth 1`은 head와 tail의 quicklistNode 각 1개를 제외하고 나머지 중간 노드를 LZF로 압축합니다.

**성능 저하 패턴: 중간 인덱스 접근**
```bash
LINDEX mylist 500  # 중간 원소 접근
```
해당 노드가 압축되어 있으면 → **LZF 압축 해제** → 데이터 접근 → (옵션: 재압축). 이 과정이 CPU를 추가로 사용합니다.

**영향 없는 패턴:**
- `LPUSH/RPOP/LPOP/RPUSH`: 항상 head/tail 노드 접근 → 압축 안 된 노드 → 즉시 접근

**권장 사용 조건:**
- LPUSH + RPOP (또는 RPUSH + LPOP) 위주 큐 패턴
- `LRANGE 0 99` 같이 앞에서 N개만 읽는 패턴
- 중간 인덱스 접근이 없는 경우

`list-compress-depth`를 설정하면 메모리는 절약되지만, 중간 인덱스 접근이 있는 서비스에서는 CPU 비용이 증가합니다. 먼저 접근 패턴을 분석하고 적용하세요.

</details>

---

**Q3.** Redis List를 이용해 분산 작업 큐를 구현할 때 `LPOP`과 `BLPOP`의 차이는? Worker가 10개일 때 `BLPOP`이 적합한 이유는?

<details>
<summary>해설 보기</summary>

**LPOP**: 큐가 비어 있으면 `nil` 즉시 반환. 비어있으면 Worker가 폴링(polling)해야 합니다.
```python
# LPOP 기반 폴링
while True:
    task = redis.lpop("queue")
    if task is None:
        time.sleep(0.1)  # 100ms마다 확인 → 불필요한 Redis 호출
    else:
        process(task)
```

**BLPOP**: 큐가 비어 있으면 데이터가 올 때까지 블로킹. 데이터 도착 즉시 깨어납니다.
```python
# BLPOP 기반 (효율적)
while True:
    result = redis.blpop("queue", timeout=0)  # 0 = 무한 대기
    if result:
        task = result[1]
        process(task)
```

**Worker 10개 + `BLPOP`이 적합한 이유:**
1. **폴링 없음**: 10개 Worker가 각각 `BLPOP`으로 대기 → Redis 연결만 유지, CPU 사용 없음
2. **즉시 처리**: RPUSH로 데이터 추가 시 블로킹 중인 Worker 중 하나가 즉시 깨어남
3. **공정 분배**: Redis가 FIFO로 Worker에게 하나씩 분배 (두 Worker가 같은 태스크를 가져가지 않음)
4. **타임아웃 지원**: `BLPOP queue 30` → 30초 후 nil 반환, 주기적 health check 가능

단, BLPOP은 연결을 유지하므로 Worker 10개 = Redis 연결 10개 상시 유지. 연결 수가 매우 많다면 커넥션 풀 관리가 필요합니다.

</details>

---

<div align="center">

**[⬅️ 이전: String과 SDS](./01-string-sds.md)** | **[홈으로 🏠](../README.md)** | **[다음: Hash 내부 구조 ➡️](./03-hash-internals.md)**

</div>
