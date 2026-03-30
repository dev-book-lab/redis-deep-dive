# 메모리 관리 — jemalloc과 maxmemory 정책

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Redis가 기본 `malloc` 대신 `jemalloc`을 사용하는 이유는?
- 메모리 단편화(fragmentation)는 왜 발생하고, `mem_fragmentation_ratio`가 높을 때 어떻게 해결하는가?
- `maxmemory`를 설정하지 않으면 Redis는 어떻게 되는가?
- `maxmemory-policy` 8가지는 각각 내부에서 어떻게 키를 선택해 제거하는가?
- `allkeys-lru`와 `volatile-lru`를 어떤 서비스에 써야 하는가?
- LRU 근사 알고리즘이 "정확한 LRU"가 아닌 이유와 그 트레이드오프는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

jemalloc, maxmemory, eviction policy — 이 세 가지는 Redis가 메모리를 얼마나 효율적으로 쓰고, 한계에 다다랐을 때 무엇을 희생하는지 결정한다. 설정 없이 운영하면 OOM으로 프로세스가 종료되거나, 중요한 데이터가 조용히 사라지는 상황을 맞게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: maxmemory 미설정으로 OOM 발생

  상황: redis.conf에 maxmemory 설정 없이 운영
  결과: Redis가 OS 메모리 전체를 점유 → 다른 프로세스 메모리 부족
        Linux OOM Killer → Redis 프로세스 강제 종료
        또는 RAM 부족 → 스왑(Swap) 발생 → 응답 시간 수백 ms 폭등

  잘못된 대응:
    → 서버 재시작 (근본 원인 미해결)
    → "Redis가 불안정하다" → 다른 솔루션 검토
    → maxmemory를 매우 크게 설정 (물리 RAM의 95%)
       → BGSAVE 시 COW로 추가 메모리 → OOM 재발

실수 2: maxmemory-policy를 역할에 맞지 않게 설정

  상황: 결제 상태를 저장하는 Redis에 allkeys-lru 설정
  결과: 캐시 데이터가 많이 쌓이면 → LRU로 오래된 결제 상태도 제거
        → 조용한 데이터 손실 (에러조차 발생 안 함)
        → 결제 완료됐는데 상태 못 찾음 → 중복 결제 시도

실수 3: mem_fragmentation_ratio < 1.0 을 "효율적"이라고 오해

  used_memory_rss < used_memory → 스왑 발생 신호
  → "단편화가 없어서 좋다"고 오해
  → 실제로는 Redis 데이터가 디스크 스왑에 있음 → 극심한 성능 저하
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
올바른 maxmemory 설정:
  persistence 사용: maxmemory = 물리 RAM × 0.5~0.6
    (BGSAVE fork + COW → 최악 2배 메모리 필요)
  캐시 전용 (persistence 없음): maxmemory = 물리 RAM × 0.7~0.8

역할별 maxmemory-policy 결정:
  캐시 (API 응답, 임시 데이터)  → allkeys-lru
  세션 (TTL 있는 키만 제거)     → volatile-lru
  결제/주문 (절대 제거 금지)     → noeviction + 알림
  접근 패턴 명확한 캐시          → allkeys-lfu

단편화 모니터링:
  INFO memory | grep mem_fragmentation_ratio
  1.0~1.5: 정상
  > 2.0: 단편화 심각 → activedefrag yes 또는 재시작
  < 1.0: 스왑 발생 중 → 즉각 대응 필요

메모리 진단 흐름:
  INFO memory              → 전체 현황
  MEMORY DOCTOR            → 자동 진단 메시지
  MEMORY USAGE key         → 개별 키 메모리
  INFO stats | grep evicted → 제거된 키 수 (예상치 못한 제거 감지)
```

---

## 🔬 내부 동작 원리

### 1. jemalloc — 왜 기본 malloc이 아닌가

```
C 표준 malloc의 문제: 메모리 단편화

  시나리오:
    malloc(100)  → 블록 A (100 bytes)
    malloc(200)  → 블록 B (200 bytes)
    malloc(100)  → 블록 C (100 bytes)
    free(블록 B) → 200 bytes 공간 반환됨

    이후 malloc(300) 요청
    → 남은 공간: 블록 A 앞/뒤 조각 + 블록 B 자리 200 bytes
    → 연속 300 bytes 없음 → 새 메모리 요청
    → 실제 사용: 200 bytes, 실제 할당: 500 bytes → 단편화!

  Redis에서 단편화 발생 원인:
    키/값 크기가 천차만별 (수 bytes ~ 수 MB)
    Set/Del 반복 → 크고 작은 빈 블록이 산재
    작은 블록이 큰 할당 요청을 충족 못 함

jemalloc의 해결책:
  크기 클래스(Size Class) 분리:
    8, 16, 32, 48, 64, 80, 96, 112, 128 bytes ...
    할당 요청을 위 크기 중 가장 가까운 것으로 올림(round-up)
    
    예: 100 bytes 요청 → 128 bytes 클래스로 할당
        200 bytes 요청 → 256 bytes 클래스로 할당

  Arena 기반 분리:
    스레드마다 독립 Arena 사용 (Redis는 단일 스레드이므로 Arena 1개)
    같은 크기 클래스끼리 같은 메모리 영역(slab)에서 관리
    → 동일 크기 블록의 free/alloc이 같은 영역에서 반복
    → 단편화 최소화

  Chunk 단위 관리:
    대용량 할당(4MB 이상)은 별도 처리
    중소 할당은 2MB Chunk 안에서 슬롯으로 관리

결과:
  일반 malloc 단편화율: 10~30% (워크로드에 따라 훨씬 더 심할 수 있음)
  jemalloc 단편화율: 3~8% (대부분 상황)
  → Redis가 jemalloc을 기본 할당기로 채택한 이유
```

### 2. 메모리 단편화 측정과 해소

```
단편화 확인:

  redis-cli INFO memory | grep -E "used_memory|mem_fragmentation"

  used_memory:             1073741824    # Redis가 실제 사용 중인 메모리 (1GB)
  used_memory_rss:         1610612736    # OS가 Redis 프로세스에 할당한 메모리 (1.5GB)
  mem_fragmentation_ratio: 1.50         # RSS / used_memory

  mem_fragmentation_ratio 해석:
    < 1.0   : 메모리 스왑 발생 (매우 위험 — RAM이 부족해 디스크에 스왑 중)
    1.0~1.5 : 정상 범위
    1.5~2.0 : 단편화 약간 심함, 모니터링 필요
    > 2.0   : 단편화 심각, 조치 필요

단편화 해소 방법:

  방법 1: MEMORY PURGE (Redis 4.0+)
    redis-cli MEMORY PURGE
    → jemalloc에게 미사용 메모리를 OS에 반환 요청
    → 단편화를 즉시 해소하지는 않지만 RSS를 줄임
    → 이벤트 루프를 잠시 차단하므로 트래픽 적은 시간에 실행

  방법 2: activedefrag (Redis 4.0+)
    redis-cli CONFIG SET activedefrag yes
    redis-cli CONFIG SET active-defrag-ignore-bytes 100mb  # 100MB 이상 낭비 시 작동
    redis-cli CONFIG SET active-defrag-threshold-lower 10  # 단편화율 10% 이상 시 작동
    → 백그라운드에서 단편화된 메모리를 점진적으로 재배치
    → CPU 사용량이 약간 증가하지만 이벤트 루프는 유지

  방법 3: 재시작 (가장 확실)
    → Replica → Failover → 이전 Master 재시작
    → 재시작 후 메모리가 새로 할당되어 단편화 0
    → 영속성 설정에 따라 데이터 복원

메모리 사용 상세 분석:
  redis-cli MEMORY DOCTOR
  # 출력: 메모리 상태 진단 메시지
  # "Your memory allocation library is jemalloc"
  # "High fragmentation ratio" 등

  redis-cli MEMORY MALLOC-STATS
  # jemalloc 내부 통계 (arena별 사용량 등)
```

### 3. maxmemory 설정과 동작 원리

```
maxmemory 미설정 시:

  Redis가 메모리를 무제한으로 사용
  → used_memory가 물리 RAM을 초과하면:
  
  케이스 1: 스왑(Swap) 발생
    OS가 Redis 데이터를 디스크 스왑 파티션으로 이동
    → Redis 응답 시간 수십 ms → 수 초로 폭등
    → mem_fragmentation_ratio < 1.0으로 감지 가능
  
  케이스 2: OOM Killer 발생
    Linux 커널이 메모리를 가장 많이 사용하는 프로세스 강제 종료
    Redis가 종료되면 메모리의 모든 데이터 손실 (AOF 없으면)
    /var/log/kern.log: "Out of memory: Kill process [PID] (redis-server)"

maxmemory 설정:
  # redis.conf
  maxmemory 2gb          # 절대값
  maxmemory 75%          # Redis 7.0+: 물리 메모리 비율
  
  권장 설정:
    캐시 전용: 물리 RAM의 60~75%
    데이터 저장소: 물리 RAM의 50% (RDB fork() 시 메모리 2배 필요)
    Cluster 노드: 물리 RAM의 50~60%

maxmemory 도달 시 동작:
  설정된 maxmemory-policy에 따라 키를 제거하거나 오류 반환
  used_memory가 maxmemory에 근접하면 매 명령어 전에 제거 검토

INFO memory 주요 지표:
  used_memory:              실제 데이터 크기 (C malloc 기준)
  used_memory_rss:          OS가 본 프로세스 크기 (실제 사용 RAM)
  used_memory_peak:         최대 사용 메모리 (피크)
  used_memory_lua:          Lua 엔진 메모리
  used_memory_scripts:      Lua 스크립트 캐시
  maxmemory:                설정된 한도 (0 = 무제한)
  mem_fragmentation_ratio:  RSS / used_memory
  mem_allocator:            jemalloc-5.x.x
```

### 4. maxmemory-policy 8가지 완전 분석

```
제거 대상 범위:
  allkeys-* : TTL 유무 상관없이 모든 키가 제거 후보
  volatile-*: TTL이 설정된 키만 제거 후보
              (TTL 없는 키는 절대 제거 안 됨)

제거 알고리즘:
  *-lru  : LRU(Least Recently Used) — 가장 오래 전에 접근된 키 제거
  *-lfu  : LFU(Least Frequently Used) — 접근 빈도가 가장 낮은 키 제거
  *-random: 무작위 제거
  volatile-ttl: TTL이 가장 짧은(곧 만료될) 키 먼저 제거
  noeviction: 제거 없음, 쓰기 명령어에 에러 반환

8가지 정책 전체:

  noeviction   : 제거 없음. 메모리 부족 시 SET/LPUSH 등 쓰기 명령 → OOM error
                 읽기 명령(GET)은 계속 동작
                 → 데이터 보존이 최우선인 서비스 (결제, 주문)

  allkeys-lru  : 전체 키 중 LRU (최근 미사용 순)
                 → 일반 캐시 서비스에 가장 많이 사용
                 → "모든 키를 캐시로 취급"

  volatile-lru : TTL 있는 키 중 LRU
                 → TTL 없는 키는 영구 보존, TTL 있는 캐시만 제거
                 → 영구 데이터 + 임시 캐시 혼재 시

  allkeys-lfu  : 전체 키 중 LFU (접근 빈도 낮은 순)
                 → Hot Key 패턴이 명확한 서비스
                 → 자주 쓰는 키는 남기고 가끔 쓰는 키 제거

  volatile-lfu : TTL 있는 키 중 LFU

  allkeys-random: 전체 키 중 무작위 제거
                  → 접근 패턴이 완전히 균일한 경우 (드묾)

  volatile-random: TTL 있는 키 중 무작위 제거

  volatile-ttl : TTL이 짧은 키 먼저 제거
                 → "곧 만료될 것을 미리 정리"
                 → TTL 설정이 중요성을 반영하는 구조에서 유용

서비스 유형별 권장:
  ┌─────────────────────────────┬─────────────────────┐
  │ 서비스 유형                    │ 권장 정책             │
  ├─────────────────────────────┼─────────────────────┤
  │ 순수 캐시 (API 결과 등)         │ allkeys-lru         │
  │ 세션 저장소                    │ volatile-lru        │
  │ 결제/주문 데이터                │ noeviction          │
  │ Hot Key가 명확한 캐시          │ allkeys-lfu         │
  │ TTL이 중요도를 반영하는 캐시      │ volatile-ttl        │
  └─────────────────────────────┴─────────────────────┘
```

### 5. LRU 근사 알고리즘 — 왜 정확한 LRU가 아닌가

```
정확한 LRU의 비용:
  이론적 LRU: 모든 키를 접근 시간순으로 정렬된 linked list 유지
  → 키 접근 시마다 리스트에서 찾아 맨 앞으로 이동: O(1)
  → 제거 시 리스트 끝에서 꺼냄: O(1)
  
  문제:
    키 1억 개면 linked list 포인터만 수 GB
    접근마다 포인터 갱신 → 캐시 미스 발생 → 느려짐
    Redis의 메모리 효율과 성능 저하 허용 불가

Redis LRU 근사(Approximated LRU):

  각 redisObject에 24bit lru 필드 저장
  → 마지막 접근 시간을 초 단위로 기록 (unix timestamp % 2^24)
  → 2^24 초 ≈ 194일 주기로 순환 (매우 긴 주기)

  제거 시 동작:
    1. maxmemory-samples(기본 5)개 키를 무작위 샘플링
    2. 샘플 중 lru 값이 가장 오래된 키 제거
    3. 반복

  예시 (maxmemory-samples=5):
    전체 키: A(last=100), B(last=50), C(last=200), D(last=30), E(last=150)
    샘플 5개 선택: A, C, D, E, B
    D가 lru=30으로 가장 오래됨 → D 제거

  samples 값이 클수록:
    정확도 ↑, CPU 비용 ↑
    maxmemory-samples 10: 정확한 LRU에 근접, CPU 약간 증가
    maxmemory-samples 3 : 빠르지만 덜 정확

  Redis 3.0에서 개선:
    제거 후보 풀(pool)을 유지
    매 제거 시 샘플링한 키 중 좋은 후보를 풀에 보관
    다음 제거 시 풀과 새 샘플을 비교
    → 샘플 수가 적어도 정확도 향상

LFU 내부 구조 (Redis 4.0+):
  lru 필드를 LFU 모드에서 다르게 사용:
    상위 16bit: 마지막 접근 시각 (분 단위)
    하위 8bit:  접근 빈도 카운터 (0~255)

  빈도 카운터 갱신:
    접근할 때마다 +1이 아님 → 확률적 감소(Morris Counter) 적용
    현재 카운터가 높을수록 +1 확률이 낮아짐
    → 255까지 빠르게 오르는 것을 방지, 범위 0~255로 다양한 빈도 표현

  시간 경과에 따른 감소:
    lfu-decay-time(분 단위, 기본 1) 동안 접근 없으면 카운터 감소
    → 과거에 자주 쓰였지만 지금은 안 쓰이는 키를 정확히 식별
```

---

## 💻 실전 실험

### 실험 1: 메모리 현황 분석

```bash
# 전체 메모리 상태
redis-cli INFO memory

# 주요 지표만 추출
redis-cli INFO memory | grep -E "^used_memory:|used_memory_rss:|used_memory_peak:|mem_fragmentation_ratio:|maxmemory:|mem_allocator:"

# 출력 예시:
# used_memory:1073741824          # 1.0 GB (실제 데이터)
# used_memory_rss:1207959552      # 1.12 GB (OS가 할당한 RSS)
# used_memory_peak:1342177280     # 1.25 GB (과거 최대)
# mem_fragmentation_ratio:1.12    # 정상 범위
# maxmemory:2147483648            # 2 GB 제한
# mem_allocator:jemalloc-5.3.0

# 단편화 비율 경고 확인
FRAG=$(redis-cli INFO memory | grep mem_fragmentation_ratio | awk -F: '{print $2}' | tr -d ' \r')
echo "단편화 비율: $FRAG"
```

### 실험 2: maxmemory-policy 동작 확인

```bash
# 테스트 환경 설정 (1MB 제한으로 빠른 실험)
redis-cli CONFIG SET maxmemory 1mb
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# 키 생성 (1MB 초과 시 LRU 제거 시작)
for i in $(seq 1 500); do
  redis-cli SET "cache:key:$i" "value-with-some-padding-data-$(date)" > /dev/null
done

# 키 수 확인 (1MB 한도로 일부만 남음)
redis-cli DBSIZE

# 어떤 키가 남았는지 확인 (최근 접근된 키가 남아야 함)
redis-cli SCAN 0 COUNT 100

# 제거 통계 확인
redis-cli INFO stats | grep evicted
# evicted_keys:247  ← 247개 키가 LRU 정책으로 제거됨

# 정책 변경 후 비교
redis-cli CONFIG SET maxmemory-policy noeviction
redis-cli SET new_key "this_will_fail"
# (error) OOM command not allowed when used memory > 'maxmemory'
```

### 실험 3: 개별 키 메모리 사용량 측정

```bash
# 키 생성
redis-cli SET small_key "hello"
redis-cli SET large_key "$(python3 -c "print('x' * 10000)")"
redis-cli HSET user:1 name "Alice" age "30" email "alice@example.com"

# 메모리 사용량 확인
redis-cli MEMORY USAGE small_key        # ~50 bytes (오버헤드 포함)
redis-cli MEMORY USAGE large_key        # ~10,090 bytes
redis-cli MEMORY USAGE user:1           # Hash 오버헤드 포함

# MEMORY USAGE 옵션
redis-cli MEMORY USAGE large_key SAMPLES 5  # 중첩 구조 샘플링 깊이

# DEBUG OBJECT로 인코딩 정보 포함 상세 확인
redis-cli DEBUG OBJECT small_key
# Value at:0x7f... refcount:1 encoding:embstr serializedlength:5 lru:12345678 lru_seconds_idle:3 type:string

# 메모리 진단
redis-cli MEMORY DOCTOR
# Your memory allocation library is jemalloc (fragmentation below threshold)
```

### 실험 4: LFU 접근 빈도 확인

```bash
redis-cli CONFIG SET maxmemory-policy allkeys-lfu

# 빈도 차이 생성
redis-cli SET hot_key "hot"
redis-cli SET cold_key "cold"

# hot_key에 1000번 접근
for i in $(seq 1 1000); do redis-cli GET hot_key > /dev/null; done
# cold_key에 1번 접근
redis-cli GET cold_key

# LFU 빈도 확인
redis-cli OBJECT FREQ hot_key   # 높은 빈도
redis-cli OBJECT FREQ cold_key  # 낮은 빈도
# → 메모리 부족 시 cold_key가 먼저 제거됨
```

---

## 📊 성능 비교

```
maxmemory-policy별 특성 비교:

정책              | 제거 대상           | 알고리즘    | CPU 비용  | 데이터 안전성
─────────────────┼───────────────────┼──────────┼──────────┼─────────────
noeviction       | 없음 (에러)         | -        | 최소       | 최고
allkeys-lru      | 전체 키             | LRU 근사  | 낮음      | 낮음 (모두 제거 가능)
volatile-lru     | TTL 있는 키만       | LRU 근사  | 낮음       | 중간
allkeys-lfu      | 전체 키             | LFU      | 중간      | 낮음
volatile-lfu     | TTL 있는 키만        | LFU      | 중간      | 중간
allkeys-random   | 전체 키             | 무작위    | 최소       | 낮음
volatile-random  | TTL 있는 키만        | 무작위    | 최소       | 중간
volatile-ttl     | TTL 있는 키 중 TTL 짧은 것 | TTL 비교 | 낮음   | 중간

maxmemory-samples 값별 LRU 정확도:
  samples=3 : 정확도 약 73%, CPU 최소
  samples=5 : 정확도 약 83% (기본값)
  samples=10: 정확도 약 95%, CPU 약간 증가
  samples=20: 정확도 약 99%, CPU 중간 증가
  → 캐시 히트율이 중요한 서비스: samples=10 권장
```

---

## ⚖️ 트레이드오프

```
메모리 관리 설계 결정:

jemalloc 크기 클래스 단위 할당:
  장점: 단편화 최소화, 동일 크기 재사용 효율
  단점: 실제 요청보다 큰 블록 할당 (내부 단편화 약 5~15%)
  예: 100 bytes 요청 → 128 bytes 할당 → 28 bytes 낭비

LRU 근사 vs 정확한 LRU:
  정확한 LRU: 최적 제거 후보, 메모리 오버헤드 크고 성능 저하
  LRU 근사:  메모리 오버헤드 최소(24bit per key), 샘플 수로 정확도 조절

allkeys vs volatile:
  allkeys: 공간 확보 확실 (TTL 없어도 제거)
           TTL 설정 없이 모든 키를 캐시로 사용 가능
  volatile: 중요 데이터 보호 (TTL 없는 키는 안전)
            TTL 있는 키만 제거 후보 → 공간 확보가 더 어려울 수 있음
            TTL 없는 키로만 가득 차면 volatile-* 정책도 OOM 에러 발생!

maxmemory 설정값:
  너무 낮음: 잦은 제거 → 캐시 히트율 저하
  너무 높음: OOM 위험, RDB fork() 시 메모리 2배 필요
  권장: 물리 RAM의 50~75% (persistence 사용 여부에 따라)
```

---

## 📌 핵심 정리

```
Redis 메모리 관리 핵심:

jemalloc:
  크기 클래스 분리로 단편화 최소화
  mem_fragmentation_ratio로 모니터링 (정상: 1.0~1.5)
  심각 단편화 시: activedefrag 또는 재시작

maxmemory:
  반드시 설정할 것 (미설정 = 무제한 = OOM 위험)
  물리 RAM의 60~75% 권장 (persistence 있으면 더 낮게)

maxmemory-policy 선택:
  순수 캐시      → allkeys-lru (가장 일반적)
  중요 데이터 혼재 → volatile-lru (TTL 있는 것만 제거)
  제거 금지      → noeviction + 별도 알림
  Hot Key 패턴   → allkeys-lfu

LRU 근사:
  redisObject의 24bit lru 필드 활용
  maxmemory-samples(기본 5)개 샘플링으로 제거 후보 선택
  정확도 향상 필요 시 samples 10으로 증가

진단 명령어:
  INFO memory                    # 전체 메모리 현황
  redis-cli MEMORY DOCTOR        # 메모리 상태 진단
  redis-cli MEMORY USAGE key     # 개별 키 메모리
  INFO stats | grep evicted      # 제거된 키 수
```

---

## 🤔 생각해볼 문제

**Q1.** `maxmemory-policy`를 `volatile-lru`로 설정했는데, TTL이 설정된 키가 하나도 없는 상태에서 메모리가 꽉 찼다. 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`volatile-lru`는 TTL이 있는 키만 제거 대상으로 삼습니다. TTL이 있는 키가 없다면 제거할 후보가 없으므로 **`noeviction`과 동일하게 동작**합니다. 즉, SET 같은 쓰기 명령어에 `OOM command not allowed when used memory > 'maxmemory'` 오류를 반환합니다.

이것이 `volatile-*` 정책의 함정입니다. 영구 데이터(TTL 없는 키)만 있는 상황에서는 메모리를 확보하지 못합니다. `volatile-lru`를 사용할 때는 **반드시 캐시 키에 TTL을 설정**해야 효과가 있습니다.

</details>

---

**Q2.** Redis에 `maxmemory 4gb`를 설정하고 `appendonly yes`(AOF)와 `save 60 1`(RDB)을 모두 활성화했다. 서버 물리 RAM이 8GB라면 실제로 안전한가?

<details>
<summary>해설 보기</summary>

위험할 수 있습니다. `BGSAVE`(RDB) 시 `fork()`를 호출하면 Copy-On-Write 방식으로 **순간적으로 used_memory만큼 추가 메모리**가 필요합니다. used_memory가 4GB에 가까운 상황에서 BGSAVE 실행 시:

- 부모 프로세스: 4GB (계속 쓰기 발생하면 더 증가)
- 자식 프로세스: 변경된 페이지만 추가 할당 (최악 4GB)
- 합계: 최대 8GB 필요

물리 RAM 8GB 중 OS, 다른 프로세스, AOF rewrite를 위한 메모리를 빼면 실제로는 부족합니다.

권장 설정: 물리 RAM 8GB이고 persistence를 사용한다면 `maxmemory`를 **3~3.5GB**로 낮추는 것이 안전합니다. (자세한 내용은 03-rdb-snapshot.md 참고)

</details>

---

**Q3.** `mem_fragmentation_ratio`가 0.7이다. 단편화가 없어서 효율적인 것인가?

<details>
<summary>해설 보기</summary>

아니오. `mem_fragmentation_ratio = used_memory_rss / used_memory`이므로, 이 값이 1보다 작다는 것은 **OS가 Redis 프로세스에 할당한 RSS가 Redis가 스스로 사용 중이라고 생각하는 메모리보다 적다**는 의미입니다.

이는 **메모리 스왑이 발생 중**임을 나타냅니다. Redis의 일부 데이터가 RAM에서 디스크 스왑 파티션으로 내려가서, `used_memory`는 전체 데이터 크기를 반영하지만 `used_memory_rss`(실제 RAM에 있는 양)는 줄어든 상태입니다.

스왑 중인 Redis는 응답 시간이 수십 ms → 수 초로 급격히 저하됩니다. 즉각 대응이 필요합니다:
```bash
# 스왑 사용량 확인
cat /proc/$(pgrep redis-server)/status | grep VmSwap
# → maxmemory 조정 또는 서버 메모리 증설
```

</details>

---

<div align="center">

**[⬅️ 이전: 단일 스레드 이벤트 루프](./01-single-thread-event-loop.md)** | **[홈으로 🏠](../README.md)** | **[다음: Redis 객체 시스템 ➡️](./03-redis-object-system.md)**

</div>
