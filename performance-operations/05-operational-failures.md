# 운영 장애 패턴 — OOM / 복제 지연 / fork 지연

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- OOM 발생 시 Linux OOM Killer가 Redis를 종료하는 과정은?
- `maxmemory` 미설정 vs 잘못된 설정이 각각 어떤 OOM을 일으키는가?
- 복제 지연이 누적되는 근본 원인과 `repl-backlog-size` 조정이 해결하는 범위는?
- BGSAVE `fork()` 지연으로 인한 latency spike를 `latest_fork_usec`으로 진단하는 방법은?
- Redis Cluster `FAIL` 상태가 되는 조건과 복구 절차는?
- 장애 발생 후 원인 파악을 위한 사후 진단(post-mortem) 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

OOM, 복제 지연, fork 지연은 Redis 운영에서 가장 자주 발생하는 3대 장애 패턴이다. 이 패턴들은 각각 전혀 다른 원인과 해결책을 가진다. 장애 발생 후 로그와 INFO 지표를 어떻게 읽는지 모르면 원인 파악에 수 시간이 걸린다. 사전에 각 패턴의 신호와 진단 절차를 익혀두면 평균 복구 시간(MTTR)을 크게 단축할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: OOM Killer 로그를 Redis 버그로 오해

  /var/log/kern.log:
    Out of memory: Kill process 12345 (redis-server) score 950 or sacrifice child

  잘못된 분석: "Redis가 불안정하다" → Redis 버전 업그레이드
  실제 원인: maxmemory 미설정 → OS 메모리 전체 소진 → OOM Killer

실수 2: 복제 지연 원인을 네트워크로만 봄

  현상: Replica lag 수십 초 이상
  첫 번째 의심: "네트워크 문제"
  실제: Master 쓰기 속도 > Replica 처리 속도
        또는 Replica 디스크 I/O 병목 (AOF 쓰기)

실수 3: fork 지연을 Redis 버그나 CPU 부하로 오해

  현상: 5분마다 주기적으로 응답 500ms 이상
  첫 번째 의심: "CPU 부하", "GC 문제"
  실제: save 300 100 설정으로 5분마다 BGSAVE fork()
        8 GB Redis → fork() 350ms 블로킹
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
OOM 예방:
  maxmemory = 물리 RAM × 0.5~0.6 (persistence 사용 시)
  maxmemory-policy = 용도에 맞게 설정
  모니터링: used_memory / maxmemory > 0.85 알림

복제 지연 예방:
  repl-backlog-size = 쓰기 속도 × 최대 단절 시간 × 2
  Replica에 AOF 설정 시 디스크 성능 확인
  복제 지연 모니터링: slave lag > 30 알림

fork 지연 완화:
  Replica에서만 BGSAVE 실행 (Master 부하 없음)
  THP 비활성화
  latest_fork_usec 모니터링 > 200ms 알림

장애 발생 시 진단 순서:
  1. INFO all → 지표 스냅샷 저장
  2. SLOWLOG GET 100 → 느린 명령어 확인
  3. /var/log/kern.log → OOM Killer 확인
  4. 시스템 메트릭 → CPU, 메모리, 디스크 I/O 확인
  5. 복제 상태 → INFO replication lag 확인
```

---

## 🔬 내부 동작 원리

### 1. OOM 장애 패턴 — 두 가지 유형

```
유형 A: maxmemory 미설정 OOM

  과정:
    Redis가 메모리를 무제한 사용
    OS 전체 메모리 소진
    Linux OOM Killer 활성화
    → OOM Killer가 "oom_score"가 높은 프로세스 선택
    → Redis(대용량 메모리 사용) → oom_score 높음 → 종료 대상
    
  /var/log/kern.log 확인:
    grep "redis-server" /var/log/kern.log | grep "killed"
    또는
    dmesg | grep -i "out of memory" | grep redis

  OOM Score:
    /proc/{pid}/oom_score → Redis의 OOM 점수
    높을수록 OOM Killer의 타겟이 될 가능성 높음
    
    # OOM 회피 설정 (권장하지 않음, maxmemory 설정이 올바른 방법)
    echo -1000 > /proc/$(pgrep redis-server)/oom_score_adj

유형 B: maxmemory 설정 but BGSAVE 중 OOM

  과정:
    maxmemory 14 GB, 물리 RAM 16 GB
    BGSAVE fork() → 자식 프로세스 생성
    쓰기 워크로드로 COW 발생 → 추가 메모리 필요
    used_memory(14 GB) + COW(최대 14 GB) > 물리 RAM(16 GB)
    → OOM!
  
  예방:
    maxmemory ≤ 물리 RAM × 0.5 (persistence 사용 시)
    
  진단:
    INFO memory | grep -E "used_memory_human|used_memory_rss_human"
    INFO stats | grep latest_fork_usec
    → fork_usec가 매우 크면 메모리 압박 가능성
```

### 2. 복제 지연 누적 — 원인별 분류

```
INFO replication으로 지연 확인:
  slave0:ip=10.0.0.2,port=6379,state=online,offset=12345670,lag=5
  master_repl_offset: 12345678
  
  lag_bytes = 12345678 - 12345670 = 8 bytes (정상)
  lag_seconds = 5 (INFO의 lag 필드, 초 단위)

복제 지연 원인 1: Master 쓰기 속도 > Replica 처리 속도

  상황: Master 초당 50 MB 쓰기
        Replica 처리 속도 30 MB/초 (CPU or 디스크 병목)
  
  결과: Replica가 점점 뒤처짐 → lag 누적
        repl-backlog 소진 → Full Resync 유발
  
  진단:
    redis-cli -h replica INFO stats | grep instantaneous_input_kbps
    → 수신 대역폭 확인
    
    redis-cli -h replica INFO cpu
    → Replica CPU 사용률 확인

복제 지연 원인 2: Replica의 AOF fsync 병목

  상황: Replica에 AOF everysec 설정
        AOF fsync가 디스크 I/O 병목 유발
        → 복제 스트림 처리 지연
  
  진단:
    iostat -x 1 (Replica 서버에서)
    → /dev/sda의 await 높음 → 디스크 I/O 병목
  
  해결:
    Replica의 AOF fsync 낮춤 (everysec → no)
    또는 SSD 사용

복제 지연 원인 3: 네트워크 대역폭 포화

  진단:
    sar -n DEV 1 | grep eth0
    → rxkB/s가 네트워크 한계에 근접
  
  해결:
    Replica of Replica 구성
    Redis 6.0+ io-threads 활용

repl-backlog-size 설정:
  기본: 1 MB
  복제 지연 × 초당 쓰기 속도 > backlog_size → Full Resync 유발
  
  설정 공식:
    repl-backlog-size = max_lag_seconds × write_rate_per_second × 2
    
    예: 최대 60초 지연 허용, 쓰기 5 MB/초
    → repl-backlog-size = 60 × 5 MB × 2 = 600 MB
```

### 3. fork() 지연 — latency spike 진단

```
fork() 지연 발생 시나리오:
  save 300 100 설정 → 5분마다 BGSAVE 자동 실행
  BGSAVE → fork() → page table 복사
  
  fork() 시간:
    1 GB Redis: ~10~50 ms
    8 GB Redis: ~100~400 ms
    32 GB Redis: ~500ms~2s

fork() 지연 측정:
  redis-cli INFO stats | grep latest_fork_usec
  # latest_fork_usec:350000  → 350ms

  LATENCY HISTORY로 시간대별 확인:
  redis-cli LATENCY HISTORY bgsave
  # event     timestamp  latency_usec
  # bgsave    1711234567  350000
  # bgsave    1711234267  345000
  # → 5분마다 350ms 지연 패턴 확인

fork() 지연 완화:

  방법 1: Replica에서만 BGSAVE (가장 효과적)
    Master redis.conf:
      save ""                    # BGSAVE 비활성
      appendonly no              # AOF도 Replica에서만
    
    Replica redis.conf:
      save "900 1"               # Replica에서 스냅샷
      appendonly yes             # Replica에서 AOF (옵션)
  
  방법 2: THP(Transparent Huge Pages) 비활성화
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
    # THP 활성 시 COW 단위가 2 MB → fork 후 더 많은 COW 복사
    
    Redis 시작 시 경고:
    # WARNING you have Transparent Huge Pages (THP) support enabled
    
  방법 3: BGSAVE 주기 조정
    save "3600 1"  # 더 긴 주기로 fork 횟수 감소
    
  방법 4: maxmemory 낮춤
    메모리 작을수록 fork() 빠름
```

### 4. Redis Cluster FAIL 상태 — 감지와 복구

```
Cluster FAIL 상태 전환 조건:
  1. Master 장애 감지 (pfail → fail)
  2. Replica가 과반수 Master의 동의로 fail 결정
  3. Failover 실행: 해당 Shard의 Replica가 새 Master로 승격

CLUSTER INFO로 상태 확인:
  redis-cli CLUSTER INFO
  # cluster_state:ok          → 정상
  # cluster_state:fail        → 하나 이상 슬롯 미커버 (긴급!)
  # cluster_slots_assigned:16384  → 모든 슬롯 배정됨
  # cluster_slots_ok:16384    → 모든 슬롯 정상
  # cluster_slots_fail:0      → 실패 슬롯 없음
  # cluster_known_nodes:6     → 알려진 노드 수
  # cluster_size:3            → 마스터 노드 수

CLUSTER NODES로 상세 확인:
  redis-cli CLUSTER NODES
  # <id> <ip:port> <flags> ...
  # flags:
  #   master  → 정상 마스터
  #   slave   → 정상 레플리카
  #   fail    → 장애 노드 (확정)
  #   pfail   → 장애 의심 (Possible fail)
  #   noaddr  → 주소 불명

Cluster FAIL 상태 복구 절차:

  시나리오: Node C (Master) 장애 → FAIL 표시

  1단계: 장애 확인
    redis-cli CLUSTER NODES | grep fail
    # → Node C: fail 플래그 확인
    
  2단계: 자동 Failover 대기
    Cluster 자체 Failover 실행 중인지 확인
    redis-cli CLUSTER NODES | grep "node_c_replica"
    # → 레플리카가 master로 전환됐는지 확인
    
  3단계: Failover 미진행 시 수동 실행
    redis-cli -h node_c_replica CLUSTER FAILOVER FORCE
    
  4단계: 구 Master 복구 후 재연결
    redis-server --cluster-announce-ip <ip> --cluster-announce-port 6379
    redis-cli CLUSTER MEET <any-node-ip> <port>
    # → 구 Master가 Replica로 자동 재구성

  5단계: 클러스터 정상 확인
    redis-cli CLUSTER INFO | grep cluster_state
    # cluster_state:ok
    redis-cli --cluster check <any-node>:<port>
    # [OK] All 16384 slots covered
```

### 5. 사후 진단 (Post-Mortem) 방법

```
장애 발생 후 원인 파악 체크리스트:

1. INFO 스냅샷 수집 (장애 직후 즉시):
   redis-cli INFO all > /tmp/redis_post_mortem_$(date +%Y%m%d_%H%M%S).txt

2. SLOWLOG 수집:
   redis-cli SLOWLOG GET 1000 > /tmp/slowlog_dump.txt

3. 시스템 로그 확인:
   # OOM Killer 흔적
   grep -i "out of memory\|killed process" /var/log/kern.log | tail -50
   grep -i "redis" /var/log/kern.log | tail -50
   
   # Segfault 등 Redis 크래시
   grep "redis" /var/log/syslog | grep -i "error\|fault\|crash" | tail -50

4. Redis 로그 확인:
   tail -200 /var/log/redis/redis.log
   # BGSAVE/AOF 오류
   # 복제 재연결
   # OOM 경고

5. 시스템 메트릭 확인 (Grafana 또는 수동):
   # 메모리 사용 추이 (장애 전후)
   # CPU 사용률 스파이크
   # 디스크 I/O 급증
   # 네트워크 대역폭

6. Prometheus 메트릭 질의 (Grafana):
   redis_memory_used_bytes[1h]  → 메모리 증가 패턴
   rate(redis_evicted_keys_total[1m])  → eviction 발생 시점
   redis_connected_slaves  → 복제 연결 끊김 시점
```

---

## 💻 실전 실험

### 실험 1: OOM 상황 시뮬레이션

```bash
# maxmemory 낮게 설정 후 OOM 시뮬레이션
redis-cli CONFIG SET maxmemory 10mb
redis-cli CONFIG SET maxmemory-policy noeviction

# 10 MB 이상 데이터 삽입 시도
for i in $(seq 1 1000); do
  result=$(redis-cli SET "oom:test:$i" "$(python3 -c "print('x'*10000)")" 2>&1)
  if echo "$result" | grep -q "OOM"; then
    echo "OOM 에러 발생! (i=$i)"
    break
  fi
done

# OOM 에러 메시지:
# (error) OOM command not allowed when used memory > 'maxmemory'

# INFO memory로 상태 확인
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human"

# 설정 복구
redis-cli CONFIG SET maxmemory 0
redis-cli CONFIG SET maxmemory-policy noeviction
```

### 실험 2: 복제 지연 측정

```bash
# Master-Replica 구성 확인
redis-cli INFO replication | grep -E "role:|connected_slaves:|slave0:|master_repl_offset:"

# 복제 지연 실시간 모니터링
watch -n 1 '
  MASTER_OFF=$(redis-cli -p 6379 INFO replication | grep master_repl_offset | awk -F: "{print \$2}" | tr -d " \r")
  SLAVE_OFF=$(redis-cli -p 6379 INFO replication | grep "slave0:.*offset" | grep -oP "offset=\K[0-9]+")
  LAG=$(( MASTER_OFF - SLAVE_OFF ))
  echo "$(date +%T) Master: $MASTER_OFF | Slave: $SLAVE_OFF | Lag: $LAG bytes"
'

# 쓰기 폭발로 지연 유발
for i in $(seq 1 100000); do
  redis-cli -p 6379 SET "lag:test:$i" "value-$i" > /dev/null
done
```

### 실험 3: BGSAVE fork() 지연 측정

```bash
# 테스트 데이터 생성
for i in $(seq 1 500000); do
  redis-cli SET "fork:test:$i" "$(python3 -c "print('x'*100)")" > /dev/null
done

# fork() 지연 측정
redis-cli INFO stats | grep latest_fork_usec  # BGSAVE 전

redis-cli BGSAVE
sleep 2  # fork() 완료 대기

redis-cli INFO stats | grep latest_fork_usec  # 측정값 확인

# LATENCY HISTORY 확인
redis-cli CONFIG SET latency-monitor-threshold 1  # 1ms 이상 기록
redis-cli BGSAVE
sleep 2
redis-cli LATENCY HISTORY bgsave
```

### 실험 4: 장애 후 진단 스크립트

```bash
#!/bin/bash
# post_mortem.sh
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTDIR="/tmp/redis_pm_$TIMESTAMP"
mkdir -p $OUTDIR

echo "=== Redis 사후 진단 스크립트 ===" | tee $OUTDIR/summary.txt

# 1. INFO 전체
redis-cli INFO all > $OUTDIR/info_all.txt
echo "INFO 수집 완료"

# 2. SLOWLOG
redis-cli SLOWLOG GET 1000 > $OUTDIR/slowlog.txt
echo "SLOWLOG 수집 완료 ($(redis-cli SLOWLOG LEN) 건)"

# 3. CLUSTER 상태 (클러스터인 경우)
redis-cli CLUSTER INFO > $OUTDIR/cluster_info.txt 2>/dev/null
redis-cli CLUSTER NODES > $OUTDIR/cluster_nodes.txt 2>/dev/null

# 4. 시스템 로그
grep "redis" /var/log/kern.log 2>/dev/null | tail -100 > $OUTDIR/kern_log.txt
grep "Out of memory" /var/log/kern.log 2>/dev/null | tail -20 > $OUTDIR/oom_log.txt
tail -200 /var/log/redis/redis.log 2>/dev/null > $OUTDIR/redis_log.txt

# 5. 시스템 리소스
free -h > $OUTDIR/memory.txt
vmstat 1 5 > $OUTDIR/vmstat.txt

echo "진단 파일 저장: $OUTDIR"
ls $OUTDIR
```

---

## 📊 성능/비용 비교

```
장애 유형별 복구 시간 (MTTR):

장애 유형         | 원인 모를 때   | 원인 알 때   | 자동화 가능
────────────────┼─────────────┼────────────┼───────────
OOM Killer      | 30분~수 시간  | 5~15분      | 알림으로 예방
복제 지연 누적      | 1~2시간     | 10~30분     | 알림 + backlog 조정
fork() latency  | 수 시간      | 15~30분     | Replica로 BGSAVE 이전
Cluster FAIL    | 1~3시간      | 10~20분     | Sentinel/Cluster 자동

장애 예방 효과:

설정 변경               | 예방 효과
──────────────────────┼────────────────────────────────
maxmemory 적절 설정     | OOM Killer 예방 (필수)
repl-backlog 충분히    | Full Resync 예방 (복제 안정)
THP 비활성화            | fork() 지연 50% 감소
Replica에서 BGSAVE     | Master fork() 지연 0으로 만듦
activedefrag yes      | 단편화 방지 (메모리 회수)
```

---

## ⚖️ 트레이드오프

```
OOM 예방 설정:
  maxmemory 낮게 설정 → 더 안전하지만 활용 가능한 메모리 감소
  maxmemory 높게 설정 → 더 많은 데이터, but OOM 위험 증가

repl-backlog 크게 설정:
  장점: Partial Resync 범위 확대 → Full Resync 감소
  단점: 메모리 사용 증가 (backlog는 메모리 점유)
  → maxmemory 계산 시 backlog 크기 포함 고려

Replica에서 BGSAVE:
  장점: Master fork() 지연 제거 → 응답 시간 안정
  단점: Replica에 부하 증가, RDB 파일이 최신 아닐 수 있음 (Replica의 데이터 기준)

Cluster FAIL 자동 복구:
  cluster-node-timeout이 짧을수록: 빠른 Failover, 네트워크 순간 지연에 오탐
  cluster-node-timeout이 길수록: 안정적, 실제 장애 시 복구 늦음
  → 5~15초 권장 (환경에 따라)
```

---

## 📌 핵심 정리

```
3대 운영 장애 패턴:

OOM:
  원인: maxmemory 미설정 or BGSAVE COW 초과
  예방: maxmemory = 물리 RAM × 0.5~0.6 (persistence 사용 시)
  진단: /var/log/kern.log | grep "Out of memory"
  복구: maxmemory 조정 + maxmemory-policy 설정

복제 지연:
  원인: 쓰기 속도 > Replica 처리 속도, backlog 소진
  예방: repl-backlog-size = 쓰기 속도 × 단절 시간 × 2
  진단: INFO replication | grep "slave0.*lag"
  복구: backlog 크기 조정, Replica I/O 병목 해소

fork() 지연:
  원인: BGSAVE/AOF rewrite 시 page table 복사 지연
  예방: Replica에서 BGSAVE 실행, THP 비활성화
  진단: INFO stats | grep latest_fork_usec
  복구: BGSAVE를 Replica로 이전, maxmemory 낮춤

Cluster FAIL:
  진단: CLUSTER INFO | grep cluster_state
         CLUSTER NODES | grep fail
  복구: 자동 Failover 확인 → 수동 CLUSTER FAILOVER FORCE

Post-Mortem:
  INFO all + SLOWLOG + kern.log + redis.log
  → 원인 파악 → 재발 방지 설정 변경
```

---

## 🤔 생각해볼 문제

**Q1.** `maxmemory 8gb`를 설정했는데 `used_memory`가 7.8 GB에 도달했다. `maxmemory-policy allkeys-lru`이면 자동으로 메모리가 확보되므로 OOM이 발생하지 않는다. 하지만 여전히 문제가 있을 수 있다. 무엇인가?

<details>
<summary>해설 보기</summary>

**BGSAVE fork() 시 OOM 가능성:**

`allkeys-lru`가 `used_memory < maxmemory`를 유지해주지만, BGSAVE 실행 시:
- fork() 후 COW로 인해 `used_memory_rss` 증가
- used_memory(7.8 GB) + COW(최대 7.8 GB) = 최대 15.6 GB
- 물리 RAM이 16 GB라면 → 가용 메모리 0.4 GB → 다른 프로세스 OOM!

**추가 문제: eviction으로 인한 성능 저하**

used_memory가 maxmemory에 근접하면 매 쓰기 명령어 전에 LRU eviction이 실행된다. 이 overhead가 응답 시간을 늘린다.

**올바른 설계:**
```bash
# BGSAVE 사용 시 maxmemory를 물리 RAM의 50%로
# 16 GB 서버 → maxmemory 8gb (이미 맞지만 BGSAVE 고려 안 됨)
# → maxmemory 6gb가 더 안전

# 또는 Replica에서만 BGSAVE
redis-cli CONFIG SET save ""  # Master: BGSAVE 없음
```

eviction이 시작되면 Prometheus의 `redis_evicted_keys_total`이 증가하므로 이를 알림으로 설정해 사전에 대응한다.

</details>

---

**Q2.** Replica의 복제 지연(lag)이 갑자기 100초로 급등했다. Replica 서버의 CPU는 정상인데 디스크 I/O만 높다. 원인과 해결책은?

<details>
<summary>해설 보기</summary>

**원인: Replica의 AOF fsync 병목**

Replica에 `appendonly yes` + `appendfsync everysec`이 설정된 경우, Master에서 전달된 복제 스트림을 처리하면서 동시에 AOF에 fsync한다. 디스크가 느리면(HDD) fsync가 병목이 되어 복제 스트림 처리가 지연된다.

진단:
```bash
# Replica 서버에서
iostat -x 1  # /dev/sda의 await 확인
             # await > 50ms = 디스크 I/O 병목

# Redis Replica에서
redis-cli -h replica INFO persistence | grep aof_last_write_status
# → ok이지만 slow (응답 지연)
```

**해결책:**

1. **AOF fsync 정책 완화 (즉각 효과):**
```bash
redis-cli -h replica CONFIG SET appendfsync no
# 또는
redis-cli -h replica CONFIG SET appendfsync everysec  # 기본값
```

2. **SSD로 업그레이드:** 근본적인 해결

3. **Replica의 AOF 비활성화 (가장 간단):**
```bash
redis-cli -h replica CONFIG SET appendonly no
```
Master에 AOF가 있으면 Replica의 AOF는 선택적. Replica의 역할이 읽기 부하 분산이라면 AOF 없이도 복제된 데이터는 안전하다.

4. **별도 디스크로 AOF 분리:**
```conf
# redis.conf
dir /data_fast  # NVMe SSD 마운트 경로
```

</details>

---

**Q3.** `INFO stats | grep latest_fork_usec`로 매번 0이 반환된다. Redis에 BGSAVE가 실행된 적이 없다는 의미인가?

<details>
<summary>해설 보기</summary>

**항상 그렇지는 않다.** `latest_fork_usec: 0`은 다음 두 가지 상황을 의미한다:

1. **정말 BGSAVE(또는 AOF Rewrite)가 한 번도 실행되지 않은 경우**
   - 서버 시작 후 첫 번째 BGSAVE가 아직 없음
   - `save ""` 설정으로 자동 BGSAVE 비활성화

2. **지원하지 않는 플랫폼 (Windows 등):**
   - Windows 버전 Redis는 fork() 대신 다른 메커니즘 사용

**확인 방법:**
```bash
# BGSAVE 실행 이력 확인
redis-cli INFO persistence | grep -E "rdb_last_bgsave_time|rdb_last_save_time|rdb_bgsave_in_progress"

# 강제 BGSAVE 실행 후 latest_fork_usec 확인
redis-cli BGSAVE
sleep 5
redis-cli INFO stats | grep latest_fork_usec
# → 0이 아닌 값이 나오면 fork() 정상 동작
# → 계속 0이면 BGSAVE가 실행되지 않음 (save "" 또는 오류)

# 실행 이력 없을 때 redis.log 확인
tail -50 /var/log/redis/redis.log | grep -i "bgsave\|rdb"
```

만약 `save "900 1"` 설정이 있는데도 `latest_fork_usec: 0`이면, 아직 900초 동안 1개 이상의 변경이 없었거나, 서버가 최근에 시작됐을 가능성이 높다.

</details>

---

<div align="center">

**[⬅️ 이전: Redis 모니터링](./04-monitoring.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — Spring Redis 통합 ➡️](../spring-redis/01-redis-template-serialization.md)**

</div>
