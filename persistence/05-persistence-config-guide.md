# 영속성 설정 실전 가이드 — 서비스 특성별 최적 조합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 캐시 전용 Redis에서 영속성을 활성화하면 어떤 불필요한 비용이 발생하는가?
- 세션 저장소는 RDB만으로 충분한가, AOF가 필요한가?
- 주 데이터 저장소로 Redis를 사용할 때 반드시 설정해야 하는 항목은?
- `maxmemory`와 영속성 정책이 어떻게 상호작용하는가?
- Replica를 사용할 때 Master와 Replica의 영속성 설정을 다르게 해야 하는 이유는?
- 영속성 설정 변경 후 검증해야 할 항목들은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

캐시, 세션, 결제 데이터는 모두 서로 다른 데이터 손실 허용 범위(RPO)를 가진다. "Redis니까 전부 같은 설정"이라는 접근은 성능 낭비와 데이터 손실을 동시에 일으킨다. 역할별로 영속성 설정을 분리하는 것이 핵심이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 모든 Redis에 동일 설정 (표준 템플릿 남용)

  회사 Redis 표준 템플릿:
    appendonly yes
    appendfsync everysec
    save 900 1

  결과 A: 캐시 전용 Redis에 적용
    → 모든 GET/SET이 AOF에 기록
    → 불필요한 디스크 I/O → 캐시 처리량 저하
    → AOF가 수십 GB → 재시작 시 수 분 대기
    → 재시작 후 캐시는 어차피 비워져야 하는데 의미 없음

  결과 B: 결제 Redis에 appendfsync everysec 적용
    → 최대 1초 손실 허용
    → "결제는 0초 손실이어야 하는데?" → 설정 기준 없이 결정

실수 2: maxmemory 없이 영속성 활성화

  설정: appendonly yes + maxmemory 미설정
  결과: Redis가 메모리를 무한정 사용
        BGSAVE/Rewrite 시 COW → OOM Killer
        → 영속성 설정했는데 오히려 불안정

실수 3: CONFIG REWRITE 없이 런타임 변경 후 재시작

  조치: redis-cli CONFIG SET appendonly yes  # 장애 중 긴급 조치
  재시작 후: redis.conf가 그대로이므로 appendonly no로 복귀
  → "설정 바꿨는데 왜 또?" → CONFIG REWRITE 몰랐음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
역할별 Redis 인스턴스 분리 + 각자 최적 설정:

  캐시 Redis (손실 허용):
    save ""
    appendonly no
    maxmemory 12gb          # 물리 RAM × 0.75
    maxmemory-policy allkeys-lru

  세션 Redis (최대 1초 손실):
    appendonly yes
    appendfsync everysec
    aof-use-rdb-preamble yes
    maxmemory 8gb           # 물리 RAM × 0.5
    maxmemory-policy volatile-lru

  결제 Redis (손실 불가):
    appendonly yes
    appendfsync always
    maxmemory 4gb           # 물리 RAM × 0.5
    maxmemory-policy noeviction

무중단 설정 변경 절차:
  redis-cli CONFIG SET appendonly yes    # 런타임 적용
  redis-cli CONFIG SET appendfsync everysec
  redis-cli CONFIG REWRITE               # redis.conf 영구 반영 (필수!)
  grep appendonly redis.conf             # 확인

설정 변경 후 검증:
  INFO persistence | grep -E "aof_enabled|aof_last_write_status|rdb_last_bgsave_status"
  # aof_enabled:1, aof_last_write_status:ok → 정상
```

---

## 🔬 내부 동작 원리

### 1. 서비스 유형별 요구사항 분석

```
서비스 유형 분류표:

┌─────────────────────┬──────────────┬──────────────┬──────────────────────┐
│ 서비스 유형          │ RPO (손실 허용)│ RTO (복구 시간)│ 특성                 │
├─────────────────────┼──────────────┼──────────────┼──────────────────────┤
│ 캐시 (API 응답 등)   │ 무한         │ 즉시 (캐시 miss│ 손실 시 DB 조회로 복구│
│                     │              │ → DB 조회)    │                      │
│ 세션 저장소          │ ~수 분 허용  │ 빠른 재시작   │ 재시작 시 재로그인 감수│
│ 메시지 큐 (비즈니스) │ ≤1초         │ ≤1분          │ 메시지 손실 불허      │
│ 실시간 랭킹          │ ~수 초 허용  │ 수 분 허용    │ 재계산 가능           │
│ 주 데이터 저장소     │ ≤1초         │ ≤5분          │ 손실 불허, 비즈니스 핵심│
│ 임시 연산 결과       │ 무한         │ 즉시          │ 재연산 가능           │
└─────────────────────┴──────────────┴──────────────┴──────────────────────┘

결정 질문:
  Q1. Redis가 죽었다 살아났을 때 데이터가 있어야 하는가?
      No  → 영속성 불필요 (캐시, 임시 연산)
      Yes → Q2

  Q2. 데이터 손실 허용 범위는?
      수 분~수십 분 허용 → RDB만으로 충분
      최대 1초 허용    → AOF (everysec)
      손실 0           → AOF (always) → 성능 저하 감수 필요

  Q3. 재시작 속도가 중요한가?
      Yes → 혼합 포맷 (aof-use-rdb-preamble yes)
      No  → 순수 AOF 가능
```

### 2. 시나리오별 최적 설정

```
시나리오 A: 순수 캐시 (API 응답, DB 쿼리 결과 캐싱)

  특성:
    Redis 손실 시 DB에서 다시 채울 수 있음
    영속성 불필요
    최대 처리량이 핵심
  
  설정:
  ─────────────────────────────────
  # 영속성 비활성화
  save ""
  appendonly no
  
  # 메모리 정책 (캐시 용도)
  maxmemory 6gb
  maxmemory-policy allkeys-lru
  
  # 재시작 빠르게
  dbfilename ""  # RDB 파일 저장 안 함
  ─────────────────────────────────
  
  장점: 최대 처리량, 최소 디스크 I/O, 빠른 재시작
  주의: Redis 재시작 = 캐시 cold start → DB 부하 급증
        → 캐시 워밍업 전략 필요 (캐시 이중화, 점진적 트래픽 전환)

시나리오 B: 세션 저장소 (로그인 세션, 임시 인증 토큰)

  특성:
    재시작 시 세션 손실 = 전체 재로그인 (UX 문제)
    수 분의 데이터 손실은 허용 가능 (세션 TTL이 있으므로)
    빠른 재시작 중요 (다운타임 최소화)
  
  설정:
  ─────────────────────────────────
  # RDB만 (빠른 재시작, 수 분 RPO)
  save 900 1
  save 300 10
  save 60 1000
  dbfilename "sessions.rdb"
  
  # AOF는 선택적 (더 높은 내구성 원할 시)
  appendonly yes
  appendfsync everysec
  aof-use-rdb-preamble yes
  
  # 메모리 (세션이 만료되므로 volatile-lru)
  maxmemory 4gb
  maxmemory-policy volatile-lru
  ─────────────────────────────────
  
  RDB만으로 충분한 경우:
    - 세션 TTL이 수 분~수십 분 (어차피 만료됨)
    - 재시작이 점검 시간에만 발생
    - 사용자 재로그인을 허용하는 서비스

  AOF 추가가 필요한 경우:
    - 세션 TTL이 길거나 (수 시간~수 일)
    - 재시작이 예고 없이 발생 가능
    - 재로그인이 비즈니스 손실인 서비스 (결제 중 로그아웃 등)

시나리오 C: 실시간 랭킹 / 카운터

  특성:
    수 초의 데이터 손실은 허용 (랭킹은 자주 바뀜)
    재시작 후 Sorted Set/카운터 재집계 가능한 경우 많음
    처리량이 매우 중요 (초당 수십만 ZADD/INCR)
  
  설정:
  ─────────────────────────────────
  # RDB 단독 (수 분 RPO, 빠른 재시작)
  save 300 1
  save 60 10000
  appendonly no
  
  # 높은 처리량
  maxmemory 8gb
  maxmemory-policy allkeys-lru
  ─────────────────────────────────
  
  또는 재집계가 불가능하면:
  appendonly yes
  appendfsync everysec  # 1초 RPO

시나리오 D: 메시지 큐 / 작업 큐 (Redis Stream, List)

  특성:
    메시지 손실 = 작업 미처리 = 비즈니스 손실
    처리 보장(ACK) 사용 중
    높은 메시지 처리량 필요
  
  설정:
  ─────────────────────────────────
  # AOF (1초 RPO)
  appendonly yes
  appendfsync everysec
  aof-use-rdb-preamble yes
  
  # Rewrite 자동화
  auto-aof-rewrite-percentage 100
  auto-aof-rewrite-min-size 64mb
  
  # 메모리 (큐는 처리 후 제거되므로 conservative)
  maxmemory 4gb
  maxmemory-policy noeviction  # 큐 데이터는 절대 제거 금지!
  ─────────────────────────────────
  
  중요: maxmemory-policy noeviction → 메모리 부족 시 XADD/RPUSH 오류
        → 큐 처리 속도 > 생성 속도 보장 필요
        → 모니터링: used_memory와 maxmemory 간격

시나리오 E: 주 데이터 저장소 (Redis가 유일한 저장소)

  특성:
    데이터 손실 = 복구 불가
    가장 높은 내구성 요구
    성능은 두 번째 우선순위
  
  설정:
  ─────────────────────────────────
  # RDB + AOF 혼합
  save 3600 1
  save 300 100
  save 60 10000
  dbfilename "data.rdb"
  
  appendonly yes
  appendfsync everysec  # always는 성능 매우 낮음
  aof-use-rdb-preamble yes
  no-appendfsync-on-rewrite no  # Rewrite 중에도 fsync 유지
  
  auto-aof-rewrite-percentage 100
  auto-aof-rewrite-min-size 64mb
  
  # 메모리 (데이터 저장소)
  maxmemory 6gb  # 물리 RAM의 50% (BGSAVE fork 메모리 여유)
  maxmemory-policy noeviction  # 데이터 절대 제거 금지
  ─────────────────────────────────
  
  추가 권장:
    Replica 구성 (HA + 추가 내구성)
    정기 원격 백업 (S3, 다른 서버)
    no-appendfsync-on-rewrite no (Rewrite 중에도 안전)
```

### 3. maxmemory와 영속성의 상호작용

```
maxmemory가 영속성에 미치는 영향:

BGSAVE 중 메모리 계산:
  used_memory: 8 GB (Redis 데이터)
  BGSAVE 시작 → fork()
  COW 복사: 쓰기가 많으면 수 GB 추가
  peak: 최악 8 + 8 = 16 GB (100% 쓰기 발생 시)
  
  물리 RAM 16 GB 서버:
    maxmemory 8 GB 설정 → BGSAVE 중 OOM 가능!
    maxmemory 6 GB 설정 → 여유 10 GB → 안전
  
  권장 maxmemory:
    persistence 없음:      물리 RAM × 70~75%
    RDB만:                 물리 RAM × 50~60% (fork 메모리 여유)
    AOF (everysec):        물리 RAM × 60~70% (COW 적음)
    RDB + AOF:             물리 RAM × 50%

maxmemory-policy와 영속성:

  noeviction + AOF everysec:
    → 최고 내구성, 메모리 부족 시 쓰기 오류
    → 주 데이터 저장소에 적합
  
  allkeys-lru + AOF no:
    → 최고 처리량, 낮은 내구성
    → 순수 캐시에 적합
  
  volatile-lru + AOF everysec:
    → 세션 등 TTL 있는 캐시에 적합
    → TTL 없는 키는 보호, TTL 있는 키는 제거 대상
  
  allkeys-lru + persistence 없음:
    → 캐시 전용 최적 (메모리 부족 시 오래된 키 자동 제거)

AOF와 eviction의 상호작용:
  maxmemory-policy로 키가 evict되면 → AOF에 DEL 명령어 기록
  AOF Rewrite 후 evict된 키는 제외됨 (현재 메모리에 없으므로)
  → 문제 없음
```

### 4. Replica와 영속성 분리

```
Master-Replica 영속성 전략:

방법 1: Master에 영속성, Replica는 없음
  Master:
    appendonly yes
    appendfsync everysec
    save 3600 1 300 100 60 10000
  
  Replica:
    appendonly no
    save ""
  
  장점: Master 성능 보호 불필요 (어차피 Master에 영속성)
  단점: Master 재시작 시 복구 필요 (AOF/RDB 로드 시간)
  사용: 쓰기 처리량이 중요한 서비스

방법 2: Replica에만 영속성, Master는 없음 (권장)
  Master:
    appendonly no
    save ""
  
  Replica:
    appendonly yes
    appendfsync everysec
    aof-use-rdb-preamble yes
  
  장점: Master 최대 처리량, Replica에서 영속성 부담
  단점: Master가 죽으면 Replica의 데이터가 더 오래될 수 있음
  주의: Master 재시작 후 Replica에서 전체 동기화
        → Master의 데이터가 비어있으면 Replica도 비워짐!
        → 반드시 Replica failover 후 재시작
  
  Master 재시작 안전 절차:
    1. Replica를 Master로 승격 (REPLICAOF NO ONE)
    2. 이전 Master 재시작 (이제 Replica로 연결)
    3. 새 Replica가 데이터 동기화

방법 3: Master + Replica 모두 영속성
  Master:
    appendonly yes
    appendfsync everysec
  
  Replica:
    appendonly yes
    appendfsync everysec
    # 또는 save만 (Replica는 쓰기가 적으므로)
  
  장점: 이중 보호
  단점: 이중 I/O 부하

Redis Sentinel / Cluster 환경:
  Sentinel이 자동 Failover 처리
  각 노드의 영속성 설정을 독립적으로 구성
  Sentinel 권장: 적어도 하나의 Replica에 영속성 활성화
```

### 5. 영속성 설정 변경 절차

```
운영 중 안전한 설정 변경:

1. AOF 활성화 (기존 RDB만 → RDB+AOF):
   # CONFIG SET으로 런타임 변경
   redis-cli CONFIG SET appendonly yes
   # 이후 즉시 BGREWRITEAOF 실행 (현재 RDB 데이터를 AOF에 포함)
   redis-cli BGREWRITEAOF
   # 완료 확인
   redis-cli INFO persistence | grep aof_rewrite_in_progress

2. AOF 비활성화 (AOF → RDB만):
   # 먼저 RDB 저장
   redis-cli BGSAVE && sleep 30
   redis-cli CONFIG SET appendonly no
   # 재시작 없이 즉시 적용

3. appendfsync 변경:
   redis-cli CONFIG SET appendfsync everysec  # 런타임 변경 가능
   # 즉시 적용

4. save 설정 변경:
   redis-cli CONFIG SET save "3600 1 300 100 60 10000"
   # 기존 설정 교체
   redis-cli CONFIG SET save ""  # 비활성화

5. 변경사항 영구 저장 (redis.conf 반영):
   redis-cli CONFIG REWRITE
   # redis.conf에 현재 CONFIG 상태 저장
   # 재시작 후에도 유지됨

변경 후 검증 체크리스트:
  □ INFO persistence로 설정 반영 확인
  □ redis-check-aof로 AOF 무결성 확인
  □ 실제 파일 크기 변화 모니터링
  □ LASTSAVE로 RDB 저장 시각 확인
  □ 부하 테스트로 성능 영향 측정
```

---

## 💻 실전 실험

### 실험 1: 시나리오별 설정 적용 및 성능 측정

```bash
# 시나리오 A: 순수 캐시 설정
redis-cli CONFIG SET save ""
redis-cli CONFIG SET appendonly no
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG SET maxmemory 1gb

# 처리량 벤치마크
redis-benchmark -t set,get -n 100000 -c 100 --csv
# 기준값으로 기록

# 시나리오 B: 세션 저장소 (RDB + AOF everysec)
redis-cli CONFIG SET save "900 1 300 10 60 1000"
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET appendfsync everysec
redis-cli CONFIG SET maxmemory-policy volatile-lru

# 동일 벤치마크
redis-benchmark -t set,get -n 100000 -c 100 --csv
# 성능 차이 확인

# 시나리오 E: 주 데이터 저장소 (최고 내구성)
redis-cli CONFIG SET appendfsync always
redis-benchmark -t set -n 10000 -c 10 --csv
# always fsync의 성능 저하 확인
```

### 실험 2: maxmemory와 BGSAVE 상호작용

```bash
# 낮은 maxmemory로 BGSAVE 중 메모리 관찰
redis-cli CONFIG SET maxmemory 500mb
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# 데이터 채우기 (maxmemory 근처까지)
for i in $(seq 1 50000); do
  redis-cli SET "data:$i" "$(python3 -c "print('x'*1000)")" > /dev/null
done

redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human"

# BGSAVE 실행 중 메모리 모니터링
redis-cli BGSAVE
watch -n 1 'redis-cli INFO memory | grep -E "used_memory_human|mem_fragmentation"'

# COW 복사량 확인
redis-cli INFO persistence | grep rdb_last_cow_size
```

### 실험 3: Replica 영속성 분리 설정

```bash
# Master 설정 (영속성 없음, 최대 처리량)
redis-cli -p 6379 CONFIG SET appendonly no
redis-cli -p 6379 CONFIG SET save ""

# Replica 설정 (AOF + RDB)
redis-cli -p 6380 CONFIG SET appendonly yes
redis-cli -p 6380 CONFIG SET appendfsync everysec
redis-cli -p 6380 CONFIG SET aof-use-rdb-preamble yes
redis-cli -p 6380 CONFIG SET save "3600 1"

# 각 인스턴스 INFO persistence 비교
echo "=== Master ===" && redis-cli -p 6379 INFO persistence
echo "=== Replica ===" && redis-cli -p 6380 INFO persistence

# Master 처리량 vs Replica 처리량 비교
redis-benchmark -p 6379 -t set -n 100000 --csv
redis-benchmark -p 6380 -t set -n 100000 --csv  # 읽기는 Replica도 가능
```

### 실험 4: 영속성 설정 변경 및 검증

```bash
# 런타임 AOF 활성화
redis-cli CONFIG SET appendonly yes
redis-cli INFO persistence | grep aof_enabled  # 1

# AOF 파일 생성 확인
ls -la $(redis-cli CONFIG GET dir | tail -1)/appendonly.aof

# BGREWRITEAOF로 현재 데이터 AOF에 포함
redis-cli BGREWRITEAOF
redis-cli INFO persistence | grep aof_rewrite_in_progress

# 완료 대기 및 확인
until [ "$(redis-cli INFO persistence | grep aof_rewrite_in_progress | awk -F: '{print $2}' | tr -d ' \r')" = "0" ]; do
  sleep 1
done
echo "BGREWRITEAOF 완료"

# 파일 무결성 확인
redis-check-aof $(redis-cli CONFIG GET dir | tail -1)/appendonly.aof

# redis.conf에 반영
redis-cli CONFIG REWRITE
echo "설정이 redis.conf에 저장됨"
```

---

## 📊 성능 비교

```
영속성 설정별 처리량 및 지연 비교 (50 concurrent, SET 명령어, SSD):

설정 조합                    | ops/sec    | p50 지연 | p99 지연
───────────────────────────┼───────────┼─────────┼─────────
persistence 없음             | ~120,000   | 0.4 ms  | 0.9 ms
RDB만 (save 60 10000)       | ~115,000   | 0.4 ms  | 1.0 ms
AOF everysec                | ~95,000    | 0.5 ms  | 1.5 ms
AOF everysec + RDB          | ~90,000    | 0.5 ms  | 1.5 ms
AOF always                  | ~8,000     | 6.0 ms  | 15 ms

서비스 유형별 권장 RPO, RTO, 설정:

서비스          | RPO      | 설정                          | 재시작 시간
───────────────┼──────────┼───────────────────────────── ┼───────────
순수 캐시       | 무관     | save "" + appendonly no       | 즉시
세션 (경량)     | ~5분     | save 300 100 + appendonly no  | ~수십 초
세션 (중요)     | ~1초     | appendonly yes + everysec     | ~수 분
랭킹/카운터     | ~1분     | save 60 10000                 | ~수십 초
메시지 큐       | ~1초     | appendonly yes + everysec     | ~수 분
주 저장소       | ~1초     | RDB + AOF + everysec          | ~수 분
결제/금융       | ~0초     | AOF + always + Replica        | ~수 분 (Replica로 Failover)
```

---

## ⚖️ 트레이드오프

```
영속성 설정의 핵심 트레이드오프:

내구성 vs 성능:
  always fsync:   최고 내구성, 최저 성능 (~8,000 ops/sec)
  everysec fsync: 균형 (~95,000 ops/sec, 1초 RPO)
  no/RDB:         최고 성능, 낮은 내구성

재시작 속도 vs 파일 관리:
  RDB만: 빠른 재시작, 스냅샷 간격만큼 손실 가능
  혼합 포맷: 빠른 재시작 + 1초 RPO → 최적
  순수 AOF: 느린 재시작, 1초 RPO

운영 복잡도:
  영속성 없음: 가장 단순
  RDB만: 단순 (파일 1개)
  AOF만: 파일 관리 필요 (Rewrite)
  RDB + AOF: 복잡 (두 파일 + Rewrite)
  Replica + 영속성: 가장 복잡 but 최고 가용성

비용:
  영속성 → 디스크 I/O → 디스크 비용
  BGSAVE → 순간 메모리 2배 → 메모리 비용
  AOF 파일 → 저장 공간 → 스토리지 비용

결론:
  모든 Redis 인스턴스에 동일 설정 = 잘못된 접근
  서비스 특성 → RPO/RTO 요구사항 → 최소한의 영속성 선택
  더 많은 영속성 = 더 안전 (but 더 많은 비용과 복잡성)
```

---

## 📌 핵심 정리

```
서비스별 영속성 설정 요약:

캐시 전용:
  save "" + appendonly no + allkeys-lru
  → 영속성 없음, 최대 처리량

세션 (낮은 내구성):
  save 300 10 + appendonly no + volatile-lru
  → 5분 RPO, 빠른 재시작

세션/큐 (높은 내구성):
  appendonly yes + everysec + aof-use-rdb-preamble yes
  → 1초 RPO, 중간 재시작 속도

주 데이터 저장소:
  save + appendonly yes + everysec + noeviction
  + 원격 백업 + Replica
  → 1초 RPO, 최고 내구성

maxmemory 권장:
  persistence 없음: RAM × 75%
  RDB만:           RAM × 50% (fork 여유)
  RDB + AOF:       RAM × 50%

Replica 분리:
  Master: 영속성 없음 (최대 처리량)
  Replica: AOF + everysec (영속성 담당)
  주의: Master 재시작 전 반드시 Failover 먼저!

검증:
  INFO persistence   → 현재 설정 및 상태
  redis-check-aof    → AOF 무결성
  redis-check-rdb    → RDB 무결성
  redis-benchmark    → 성능 측정
  복구 훈련 (월 1회) → RTO 측정
```

---

## 🤔 생각해볼 문제

**Q1.** 캐시 전용 Redis에 `appendonly yes`를 실수로 설정했다. 서비스에 미치는 영향은 무엇이고, 어떻게 안전하게 제거하는가?

<details>
<summary>해설 보기</summary>

**영향:**
1. 모든 SET/DEL/EXPIRE 등 쓰기 명령어가 AOF 파일에 기록 → 불필요한 디스크 I/O
2. appendfsync everysec이면 백그라운드 fsync 스레드가 1초마다 실행 → CPU/I/O 부하
3. AOF 파일이 계속 커짐 → 디스크 사용량 증가
4. 재시작 시 AOF 로드 → 불필요한 지연 (캐시는 재시작 후 비워도 됨)
5. BGREWRITEAOF 자동 실행 → fork() → 순간 메모리 2배

**안전한 제거 절차:**
```bash
# 1. appendonly 비활성화 (런타임)
redis-cli CONFIG SET appendonly no
# → 즉시 적용, 기존 AOF 파일은 유지됨

# 2. 불필요한 AOF 파일 제거 (선택, 캐시이므로 필요 없음)
AOF_DIR=$(redis-cli CONFIG GET dir | tail -1)
AOF_FILE=$(redis-cli CONFIG GET appendfilename | tail -1)
rm "$AOF_DIR/$AOF_FILE"  # 삭제해도 안전 (캐시이므로 데이터 불필요)

# 3. redis.conf에 반영
redis-cli CONFIG REWRITE

# 4. 검증
redis-cli CONFIG GET appendonly  # "no" 확인
redis-cli INFO persistence | grep aof_enabled  # 0 확인
```

캐시 전용이라면 AOF 파일을 삭제해도 전혀 문제 없습니다. 다음 재시작 시 캐시 cold start가 발생하지만, 이는 캐시의 정상 동작입니다.

</details>

---

**Q2.** 세션 저장소 Redis에 `maxmemory-policy allkeys-lru`와 `volatile-lru` 중 어느 것을 선택해야 하는가? TTL이 없는 관리자 세션과 TTL이 있는 일반 사용자 세션이 혼재하는 상황에서.

<details>
<summary>해설 보기</summary>

`volatile-lru`를 선택해야 합니다.

**이유:**
- `allkeys-lru`: 메모리 부족 시 **TTL 있는 것과 없는 것 모두** 제거 대상
  - 관리자 세션 (TTL 없음)이 갑자기 제거될 수 있음 → 관리자 강제 로그아웃
  - 중요한 데이터가 메모리 부족으로 사라짐
  
- `volatile-lru`: 메모리 부족 시 **TTL 있는 세션만** 제거 대상
  - 관리자 세션 (TTL 없음) 보호
  - 일반 사용자 세션 (TTL 있음) 중 오래된 것 먼저 제거
  - 더 중요한 세션이 살아남음

**단, 주의사항:**
- TTL 있는 세션이 모두 제거돼도 메모리가 부족하면 → `volatile-lru`도 더 이상 제거할 게 없어 OOM 오류 발생
- 해결: `maxmemory`를 여유 있게 설정 + 세션 수 모니터링 + TTL 없는 관리자 세션의 수를 제한

```bash
# 혼합 세션 저장소 설정
redis-cli CONFIG SET maxmemory-policy volatile-lru
redis-cli CONFIG SET maxmemory 4gb

# 관리자 세션 (TTL 없이 저장)
redis-cli SET session:admin:1 "admin_data"  # TTL 없음 → volatile-lru 보호

# 일반 세션 (TTL 있게 저장)
redis-cli SET session:user:1001 "user_data" EX 3600  # TTL 있음 → evict 대상
```

</details>

---

**Q3.** Replica에만 영속성을 설정하고 Master는 `save "" + appendonly no`로 운영 중이다. Master 서버에서 디스크 장애가 발생해 Master를 새 서버로 교체해야 한다. 올바른 순서는?

<details>
<summary>해설 보기</summary>

**잘못된 순서 (데이터 손실 발생):**
```
1. Master 새 서버에 Redis 시작 (빈 데이터로)
2. Replica를 새 Master에 연결 (REPLICAOF new_master_ip 6379)
→ 새 Master가 빈 데이터 → Replica도 빈 데이터로 동기화됨!
→ Replica의 AOF/RDB 모두 덮어씌워짐 → 데이터 전체 손실!
```

**올바른 순서:**
```bash
# 1. Replica를 독립 Master로 승격 (현재 데이터 보존)
redis-cli -p 6380 -h replica_host REPLICAOF NO ONE
redis-cli -p 6380 -h replica_host INFO replication | grep role
# role:master 확인

# 2. 새 서버에 Redis 시작 (아직 Replica로 연결 안 함)
# 새 서버: redis-server /etc/redis/redis.conf (초기 빈 상태)

# 3. Replica(현재 Master)에서 RDB 덤프 생성
redis-cli -p 6380 -h replica_host BGSAVE
sleep 30

# 4. RDB 파일을 새 서버로 복사
scp replica_host:/var/lib/redis/dump.rdb new_server:/var/lib/redis/dump.rdb

# 5. 새 서버 Redis 재시작 (RDB 로드)
redis-server /etc/redis/redis.conf

# 6. 새 서버가 Replica(현재 Master)에 복제 연결
redis-cli -p 6379 -h new_server REPLICAOF replica_host 6380

# 7. 동기화 확인
redis-cli -p 6379 -h new_server INFO replication | grep master_link_status
# master_link_status:up 확인

# 8. 역할 교체 (새 서버를 Master로)
redis-cli -p 6380 -h replica_host REPLICAOF new_server 6379
redis-cli -p 6380 -h replica_host INFO replication | grep role
# role:slave 확인

# 9. Sentinel/앱 설정 업데이트 (새 Master IP로 변경)
```

**핵심 원칙**: 데이터가 있는 Replica를 먼저 독립 Master로 승격한 후, 새 서버를 Replica로 연결해 데이터를 동기화합니다. 빈 서버를 Master로 만들면 절대 안 됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: 장애 복구 절차](./04-disaster-recovery.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — 복제와 HA ➡️](../replication-ha/01-replication-internals.md)**

</div>
