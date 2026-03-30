# 장애 복구 절차 — RDB/AOF로 데이터 복원

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Redis 재시작 시 AOF와 RDB 중 어느 것을 먼저 로드하는가?
- `appendonly yes`인데 RDB 파일도 있으면 어느 것이 우선인가?
- `redis-check-aof --fix`는 AOF 파일의 어떤 부분을 어떻게 수정하는가?
- `redis-check-rdb`로 발견한 오류를 어떻게 처리하는가?
- `appendfsync everysec` 환경에서 최대 데이터 손실을 실제로 계산하면 얼마인가?
- 복구 시뮬레이션으로 어떻게 복구 절차를 검증하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

장애는 항상 새벽에 온다. AOF와 RDB 중 어느 것이 최신인지, 손상된 파일을 어떻게 복구하는지, Redis가 시작할 때 어떤 기준으로 파일을 선택하는지 — 이 절차를 모르면 복구 가능한 데이터를 잃는다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: AOF와 RDB 중 어느 것이 우선인지 몰라서 혼란

  상황: appendonly yes + dump.rdb 둘 다 존재
  Redis 재시작 시: AOF로 복원 (RDB 무시)
  
  팀의 오해: "dump.rdb가 최신이니까 RDB로 복원됐겠지"
  실제: AOF가 있으면 무조건 AOF 우선 (더 오래됐어도)
  
  결과: 의도치 않게 오래된 AOF로 복원되어 최신 RDB 데이터 손실
        또는 appendonly를 켜놓고 RDB만 백업 관리 → 복원 시 혼란

실수 2: 손상된 AOF 파일로 Redis 재시작 시도

  현상: Redis 시작 시 즉시 종료
  로그: Bad file format reading the append only file
  
  잘못된 대응:
    → "AOF 파일이 손상됐으니 삭제" → appendonly.aof 삭제
    → Redis 재시작 → 빈 상태 (모든 데이터 손실!)
  
  올바른 대응:
    redis-check-aof --fix appendonly.aof
    → 손상 지점 이전까지 복구 (일부 명령어만 손실)

실수 3: 복구 가능한 시간 범위를 모름

  "얼마나 데이터가 손실됐나요?"라는 질문에 대답 못 함
  appendfsync everysec이면 최대 1초
  save 300 100이면 최대 5분
  → 이 범위를 알아야 비즈니스 임팩트를 계산할 수 있음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
재시작 전 복원 파일 선택 기준:
  appendonly yes → AOF 파일 우선 (RDB 무시)
  appendonly no  → RDB 파일만 사용
  
  → appendonly yes이면서 RDB를 쓰려면:
     appendonly no로 임시 변경 후 재시작
     또는 AOF 파일 이동 후 재시작

AOF 파일 손상 복구 절차:
  1. redis-check-aof appendonly.aof          # 손상 지점 확인
  2. cp appendonly.aof appendonly.aof.bak    # 백업
  3. redis-check-aof --fix appendonly.aof    # 손상 부분 제거
  4. redis-server redis.conf                 # 재시작

RDB 파일 손상 확인:
  redis-check-rdb dump.rdb
  → [OK] Checksum OK → 정상
  → [error] → 손상됨 → 이전 백업 사용

데이터 손실 범위 계산:
  appendfsync everysec: 최대 1초
  appendfsync always:   이론적 0
  save 300 100:         최대 5분
  
  INFO persistence | grep -E "rdb_last_save_time|aof_last_write_status"
  → rdb_last_save_time으로 마지막 저장 시각 확인

장애 대응 런북 준비:
  1. 파일 존재 확인: ls -la dump.rdb appendonly.aof
  2. 무결성 확인: redis-check-rdb, redis-check-aof
  3. 파일 선택: appendonly 설정 기준
  4. 재시작: INFO keyspace로 복원 확인
  5. 손실 범위: 타임스탬프 기반 계산
```

---

## 🔬 내부 동작 원리

### 1. Redis 재시작 시 파일 선택 로직

```
Redis 시작 시 복원 우선순위:

  if (appendonly == yes AND appendonly.aof 파일 존재):
    AOF 파일로 복원 (RDB 무시!)
    
  else if (dump.rdb 파일 존재):
    RDB 파일로 복원
    
  else:
    빈 메모리로 시작

핵심: appendonly yes이면 AOF가 항상 우선

이유:
  AOF가 활성화됐다면 RDB보다 더 최신 데이터를 담고 있음
  RDB는 시점 스냅샷, AOF는 그 이후 모든 명령어 기록
  → AOF = RDB + 이후 변경사항 → AOF가 항상 더 완전한 상태

예외 상황:
  AOF가 활성화됐지만 AOF 파일이 없음:
  → RDB로 복원 (appendonly on이지만 파일이 없는 경우)
  → 재시작 후 이후 명령어부터 AOF 기록 시작

  AOF가 손상됨 (첫 로드 실패):
  → aof-load-truncated yes: 마지막 불완전 명령어 무시하고 계속
  → aof-load-truncated no: 로드 중단 → redis-check-aof --fix 필요

Redis 로그로 복원 과정 추적:
  "* DB loaded from append only file: 12.345 seconds"  → AOF 로드 성공
  "* DB loaded from disk: 5.678 seconds"               → RDB 로드 성공
  "* The server is now ready to accept connections"    → 복원 완료
  "# Wrong signature trying to load DB from file"      → RDB 손상
  "# Bad file format reading the append only file"     → AOF 손상
```

### 2. 데이터 손실 범위 계산

```
장애 유형별 데이터 손실 범위:

유형 1: Redis 프로세스 정상 종료 (SIGTERM, SHUTDOWN)
  Redis는 SIGTERM 수신 시:
  → 현재 진행 중인 명령어 완료
  → AOF 버퍼 flush + fsync (appendfsync 정책 무관)
  → RDB 스냅샷 저장 (stop-writes-on-bgsave-error 설정에 따라)
  → 프로세스 종료
  
  데이터 손실: 0 (정상 종료)

유형 2: 프로세스 강제 종료 (SIGKILL, OOM Killer, kill -9)
  OS가 강제 종료 → AOF 버퍼 flush 없음 → 손실 가능
  
  appendfsync always:
    명령어마다 fsync → 강제 종료 직전 1ms 전 명령어까지 복구
    최대 손실: 마지막 fsync 이후 1개 명령어 (미미함)
  
  appendfsync everysec:
    1초마다 fsync → 마지막 fsync 이후 최대 1초 명령어 손실
    최대 손실: 1초치 명령어
    
    실제 계산 예시:
    - 초당 10,000 건 SET 실행 중
    - 강제 종료 시: 최대 10,000 건 손실
    - 초당 10 건 HSET 실행 중: 최대 10 건 손실
  
  appendfsync no:
    OS 판단으로 flush → Linux 기본 dirty_expire 30초
    최대 손실: 최대 30초치 명령어

유형 3: 서버 전원 차단 / 하드웨어 장애
  RAM 내용 전부 소실
  
  appendfsync always:
    디스크에 fsync된 것만 복구
    전원 차단 직전 fsync 완료 명령어까지 → 거의 완전 복구
  
  appendfsync everysec:
    마지막 fsync 이후 최대 1초 손실
    + 전원 차단 순간 진행 중이던 write() 불완전 가능
    → redis-check-aof로 불완전 명령어 제거 필요
  
  RDB만 (AOF 없음):
    마지막 BGSAVE 이후 전체 손실
    save 300 100 설정 → 최대 5분 손실

유형 4: 디스크 손상
  AOF/RDB 파일 손상 → 일부 데이터 복구 불가
  해결: 원격 백업에서 복원 (S3, 다른 서버 등)
  → 정기적 백업 없으면 복구 불가

RPO (Recovery Point Objective) vs 설정:
  RPO 0초:   appendfsync always (비현실적, 성능 매우 낮음)
  RPO 1초:   appendfsync everysec (권장 균형점)
  RPO 5분:   save 300 100 (RDB만)
  RPO 허용:  save "" + appendonly no (캐시 전용)
```

### 3. redis-check-aof 복구 절차

```
AOF 파일 손상 유형:

  유형 A: 마지막 명령어 불완전 (가장 흔함)
    Redis 크래시 직전 write() 중 → 명령어 일부만 기록
    
    예: *3\r\n$3\r\nSET\r\n$5\r\n  (여기서 잘림)
    
    감지: redis-check-aof appendonly.aof
    → "AOF file ends with an unfinished MULTIBULK command"
    → "This is usually harmless. You can fix this issue running..."
    
    수정: redis-check-aof --fix appendonly.aof
    → 불완전한 마지막 명령어 제거
    → 손실: 마지막 불완전 명령어 1개

  유형 B: 중간 데이터 손상 (드물지만 치명적)
    디스크 오류 등으로 파일 중간 데이터 손상
    
    감지: redis-check-aof appendonly.aof
    → "Unexpected char in AOF at position X"
    
    수정: redis-check-aof --fix appendonly.aof
    → 손상 위치 이후 모든 명령어 제거 (!)
    → 손실: 손상 지점 이후 모든 데이터

  유형 C: 파일 완전 손상
    복구 불가 → 백업에서 복원

redis-check-aof 사용법:

  # 1단계: 손상 확인 (파일 수정 없음)
  redis-check-aof appendonly.aof
  # 출력: "AOF analyzed: size=12345678, ok_up_to=12345600, diff=78"
  # ok_up_to: 이 위치까지 정상
  # diff: 손상/불완전 bytes

  # 2단계: 백업 (필수!)
  cp appendonly.aof appendonly.aof.backup.$(date +%Y%m%d_%H%M%S)

  # 3단계: 자동 수정
  redis-check-aof --fix appendonly.aof
  # "This will cause 78 bytes of data to be lost."
  # "Continue? [y/N]: " → y 입력
  # "Successfully truncated AOF"

  # 4단계: 재확인
  redis-check-aof appendonly.aof
  # "AOF analyzed: size=12345600, ok_up_to=12345600, diff=0"
  # diff=0 → 이제 정상

  # 5단계: Redis 재시작
  redis-server /etc/redis/redis.conf
```

### 4. redis-check-rdb 사용법

```
RDB 파일 검사:

  redis-check-rdb dump.rdb

  출력 예시 (정상):
  [offset 0] Checking RDB file dump.rdb
  [offset 26] AUX FIELD redis-ver: '7.0.12'
  [offset 40] AUX FIELD redis-bits: '64'
  [offset 52] AUX FIELD ctime: '1711234567'
  [offset 67] AUX FIELD used-mem: '536870912'
  [offset 79] Selecting DB ID 0
  [offset 12345678] Checksum OK
  [info] Exiting normally

  출력 예시 (CRC 오류):
  [offset 12345678] CRC error, expected checksum ...
  
  RDB 파일 손상 복구:
    rdbchecksum yes인데 CRC 불일치 → 파일 손상
    복구 방법:
    
    방법 1: redis-server --rdbchecksum no
    → CRC 검사 없이 로드 (손상 데이터가 메모리에 들어올 수 있음)
    
    방법 2: 이전 백업 RDB에서 복원
    
    방법 3: Replica에서 데이터 복원 (복제 설정 시)

  RDB와 AOF 중 선택:
    두 파일 모두 있을 때 어느 것이 더 최신인가?
    
    # RDB 생성 시각
    redis-check-rdb dump.rdb | grep ctime
    # 또는 파일 수정 시각
    stat dump.rdb | grep Modify
    
    # AOF 마지막 수정 시각
    stat appendonly.aof | grep Modify
    
    일반적으로 AOF가 더 최신 (계속 추가되므로)
    appendonly yes이면 Redis가 자동으로 AOF 우선
```

### 5. 복구 시뮬레이션 절차

```
주기적 복구 훈련 (Game Day) 시나리오:

시나리오 1: 프로세스 강제 종료 후 복구
  1. 현재 상태 기록: redis-cli DBSIZE, redis-cli LASTSAVE
  2. 강제 종료: kill -9 $(pgrep redis-server)
  3. 재시작: redis-server /etc/redis/redis.conf
  4. 복원 확인: redis-cli DBSIZE, redis-cli GET key_name
  5. 손실 범위: INFO persistence의 aof_last_write_status

시나리오 2: AOF 손상 후 복구
  1. AOF 복사: cp appendonly.aof /tmp/aof_backup
  2. AOF 손상 시뮬레이션: truncate -s -100 appendonly.aof
  3. redis-check-aof로 진단
  4. --fix로 복구
  5. Redis 재시작 및 데이터 확인

시나리오 3: 완전 서버 재구성
  1. 다른 서버에 Redis 설치
  2. 백업에서 dump.rdb / appendonly.aof 복원
  3. redis-check-rdb / redis-check-aof로 검증
  4. Redis 시작 및 데이터 확인
  5. 복원 소요 시간 측정 → RTO 산출

시나리오 4: 특정 시점 복구 (PITR)
  1. 특정 명령어(예: FLUSHALL) 시점 특정
     cat appendonly.aof | grep -n "FLUSHALL"
  2. 해당 줄 이전까지만 복사
     head -n [FLUSHALL 줄 이전] appendonly.aof > recovery.aof
  3. recovery.aof를 appendonly.aof로 교체
  4. Redis 재시작

복구 검증 체크리스트:
  □ 주요 키가 존재하는가? (DBSIZE, KEYS, HGET 등)
  □ 데이터 샘플 값이 올바른가?
  □ TTL이 정상인가? (PTTL로 확인)
  □ 예상 손실 범위가 실제 손실과 일치하는가?
  □ Redis 성능이 정상인가? (INFO stats)
```

---

## 💻 실전 실험

### 실험 1: 장애 복구 시뮬레이션

```bash
# 환경 설정
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET appendfsync everysec
redis-cli CONFIG SET save "3600 1 300 100 60 10000"

# 중요 데이터 생성
redis-cli SET important_session "user1_data"
redis-cli HSET order:999 status paid amount 99000
redis-cli ZADD ranking 9999 "top_user"
redis-cli INCR visit_count  # 여러 번 실행

# 손실될 가능성 있는 데이터 (1초 이내)
redis-cli SET maybe_lost "latest_action"

# 강제 종료 (전원 차단 시뮬레이션)
kill -9 $(pgrep redis-server)
sleep 1

# AOF 파일 상태 확인
redis-cli CONFIG GET dir 2>/dev/null || echo "아직 연결 안 됨"
ls -la /var/lib/redis/appendonly.aof
redis-check-aof /var/lib/redis/appendonly.aof

# 필요 시 복구
redis-check-aof --fix /var/lib/redis/appendonly.aof

# 재시작 및 복원 확인
redis-server /etc/redis/redis.conf &
sleep 3

redis-cli GET important_session   # 복원됐는가?
redis-cli HGET order:999 status   # 복원됐는가?
redis-cli ZSCORE ranking "top_user"  # 복원됐는가?
redis-cli GET maybe_lost          # 복원됐는가? (1초 이내 손실 가능)
```

### 실험 2: AOF 손상 및 복구

```bash
AOF_PATH="/var/lib/redis/appendonly.aof"

# 정상 상태 기록
redis-cli SET before_corruption "safe_data"
redis-cli BGSAVE && sleep 5

# AOF 크기 확인
ls -la $AOF_PATH
wc -l $AOF_PATH

# 손상 시뮬레이션 (마지막 100 bytes 제거)
cp $AOF_PATH ${AOF_PATH}.backup
FILESIZE=$(stat -c%s $AOF_PATH)
truncate -s $(($FILESIZE - 100)) $AOF_PATH

# 손상 확인
redis-check-aof $AOF_PATH
# "AOF analyzed: size=XXXX, ok_up_to=YYYY, diff=100"

# 복구
redis-check-aof --fix $AOF_PATH
# y 입력

# 복구 후 확인
redis-check-aof $AOF_PATH
# diff=0 확인

# 재시작
kill $(pgrep redis-server)
redis-server /etc/redis/redis.conf &
sleep 3
redis-cli GET before_corruption  # "safe_data" 복원 확인
```

### 실험 3: RDB vs AOF 선택 확인

```bash
# 두 파일 모두 존재할 때 로드 순서 확인

# 1. appendonly yes 상태에서 데이터 생성
redis-cli CONFIG SET appendonly yes
redis-cli SET aof_data "from_aof"
redis-cli BGSAVE  # RDB도 생성
sleep 5

# RDB에는 없고 AOF에만 있는 데이터 생성
redis-cli SET only_in_aof "newer_data"

# 종료
redis-cli SHUTDOWN NOSAVE

# 파일 타임스탬프 비교
ls -la /var/lib/redis/dump.rdb /var/lib/redis/appendonly.aof

# 재시작 후 로그 확인
redis-server /etc/redis/redis.conf &
sleep 3
tail -20 /var/log/redis/redis-server.log | grep -E "loaded|Loading|AOF|RDB"

redis-cli GET aof_data    # "from_aof" (RDB에 있음)
redis-cli GET only_in_aof # "newer_data" (AOF에만 있음, AOF 우선이면 존재)

# appendonly no로 변경 후 차이 확인
redis-cli CONFIG SET appendonly no
kill $(pgrep redis-server)
redis-server /etc/redis/redis.conf &
sleep 3
redis-cli GET only_in_aof  # nil (RDB만 로드되면 없음)
```

### 실험 4: 데이터 손실 범위 측정

```bash
# everysec 정책으로 손실 범위 측정
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET appendfsync everysec

# 초당 1000건 쓰기 백그라운드 실행
(for i in $(seq 1 100000); do
  redis-cli SET "loss_test:$i" "v$i" > /dev/null
  sleep 0.001  # 1ms 간격 = 초당 1000건
done) &
WRITE_PID=$!

# 1초 대기 후 강제 종료
sleep 1
kill -9 $(pgrep redis-server)
kill $WRITE_PID 2>/dev/null

# AOF 파일의 마지막 기록된 키 확인
tail -c 1000 /var/lib/redis/appendonly.aof | grep "loss_test"
# 마지막으로 기록된 loss_test:N의 N 확인

# 재시작 후 손실 범위 계산
redis-server /etc/redis/redis.conf &
sleep 3
redis-cli KEYS "loss_test:*" | wc -l
# 예상: 900~1000개 (1초 치 중 일부 손실)
```

---

## 📊 성능 비교

```
복구 시간 비교 (데이터 4 GB, SSD 기준):

방식              | 복구 시간      | RPO (최대 손실)
─────────────────┼──────────────┼────────────────
RDB만             | ~30~60초      | 마지막 스냅샷 이후
혼합 포맷 AOF     | ~1~3분        | 최대 1초 (everysec)
순수 AOF          | ~15~30분      | 최대 1초 (everysec)
AOF (always)     | ~15~30분      | 거의 0
RDB 없음 AOF 없음 | 복구 불가     | 전체 손실

redis-check-aof 처리 속도:
  분석 (--fix 없음): ~1 GB/초 (고속 스캔)
  수정 (--fix):     손상 위치까지 스캔 + 파일 재작성
  4 GB AOF: 수십 초

데이터 손실 허용 범위별 권장 정책:
  RPO = 0      → appendfsync always (성능 저하 감수)
  RPO ≤ 1초   → appendfsync everysec (균형점)
  RPO ≤ 30초  → appendfsync no
  RPO = 분 단위 → RDB save 설정 (AOF 불필요)
  RPO = 무관   → persistence 비활성 (캐시 전용)
```

---

## ⚖️ 트레이드오프

```
복구 절차 설계의 트레이드오프:

복구 속도 vs 내구성:
  always fsync: 가장 안전, 가장 느린 복구 (AOF 파일 큼)
  RDB: 빠른 복구, 낮은 내구성
  혼합 포맷: 중간 지점 (권장)

단일 파일 vs 다중 파일:
  AOF만: 단순하지만 파일 손상 = 전체 손실
  RDB + AOF: 복잡하지만 AOF 손상 시 RDB 폴백 가능
  → appendonly yes + save 설정 병행 권장

로컬 vs 원격 백업:
  로컬 AOF/RDB: 서버 장애 시 같이 손실 가능
  원격 백업 (S3, NFS): 서버 전체 손실에도 복구 가능
  → 중요 데이터: 정기 원격 백업 필수

복구 훈련의 중요성:
  "장애 시 처음으로 복구 절차 실행" → 실패 가능성 높음
  정기적 Game Day로 절차 검증
  RTO(Recovery Time Objective) 측정 → 서비스 SLA 결정에 활용
```

---

## 📌 핵심 정리

```
장애 복구 핵심:

파일 선택 우선순위:
  appendonly yes + AOF 파일 존재 → AOF 우선 (RDB 무시)
  appendonly no 또는 AOF 없음 → RDB 로드
  둘 다 없음 → 빈 메모리로 시작

데이터 손실 범위:
  정상 종료 (SHUTDOWN): 손실 0
  강제 종료 (kill -9):  fsync 정책에 따라 최대 0~30초
  전원 차단:            fsync 정책에 따라 최대 0~30초
  RDB만 사용:           마지막 BGSAVE 이후 전체

AOF 복구:
  redis-check-aof appendonly.aof       # 손상 진단
  redis-check-aof --fix appendonly.aof # 자동 수정 (백업 먼저!)
  aof-load-truncated yes               # 자동으로 불완전 끝부분 무시

RDB 복구:
  redis-check-rdb dump.rdb             # 무결성 검사
  손상 시: 백업에서 복원 또는 Replica 활용

복구 절차:
  1. 파일 상태 확인 (redis-check-aof/rdb)
  2. 손상 시 --fix 적용 (백업 먼저)
  3. Redis 재시작 (로그 모니터링)
  4. 주요 데이터 검증
  5. 손실 범위 확인 및 문서화

정기 훈련:
  월 1회 복구 시뮬레이션
  RTO/RPO 측정 → SLA 갱신
```

---

## 🤔 생각해볼 문제

**Q1.** Redis 서버가 시작하면서 "Bad file format reading the append only file: make a backup of your file and run redis-check-aof --fix"라는 로그가 나왔다. 어떤 순서로 대응해야 하는가?

<details>
<summary>해설 보기</summary>

**단계별 대응:**

```bash
# 1단계: 파일 백업 (필수, 복구 실패 대비)
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
cp /var/lib/redis/appendonly.aof /backup/appendonly.aof.${TIMESTAMP}

# 2단계: 손상 진단
redis-check-aof /var/lib/redis/appendonly.aof
# ok_up_to와 diff 확인 → 손실 데이터 범위 파악

# 3단계: 손실 가능한 데이터 비즈니스 영향 평가
# diff bytes가 어떤 데이터인지 확인 (가능하다면)
tail -c [diff_bytes+1000] /var/lib/redis/appendonly.aof | strings | head -50

# 4단계: 복구 실행
redis-check-aof --fix /var/lib/redis/appendonly.aof
# y 입력하여 수정

# 5단계: 재검증
redis-check-aof /var/lib/redis/appendonly.aof
# diff=0 확인

# 6단계: Redis 재시작
redis-server /etc/redis/redis.conf

# 7단계: 주요 데이터 검증
redis-cli DBSIZE
redis-cli GET [key_name]
redis-cli INFO persistence | grep aof_last_write_status
```

주의: 손상이 파일 중간에 발생했다면 손상 지점 이후의 모든 데이터가 손실됩니다. 이 경우 백업에서 복원을 검토하세요.

</details>

---

**Q2.** `appendfsync everysec` 환경에서 서버가 1년에 한 번 전원이 차단된다. 이 사건으로 인한 평균 데이터 손실은 몇 초인가?

<details>
<summary>해설 보기</summary>

직관적으로는 "최대 1초 손실이니 평균 0.5초"라고 생각할 수 있지만, 실제로는 더 복잡합니다.

**분석:**
- fsync는 정확히 1초마다 실행되지 않습니다. 백그라운드 스레드가 1초 간격으로 실행하지만, 디스크 지연, 부하에 따라 약간 달라질 수 있습니다.
- 전원 차단 시점이 마지막 fsync 직후라면 손실 ≈ 0초
- 전원 차단 시점이 다음 fsync 직전이라면 손실 ≈ 1초
- 균등 분포를 가정하면 **평균 0.5초** 손실

**추가 고려사항:**
- write()는 즉시 실행 (매 명령어마다) → OS page cache에는 바로 반영
- 전원 차단 = page cache도 소실 (RAM이므로)
- 즉, write()는 됐지만 fsync 안 된 구간이 최대 1초

**실제 운영 관점:**
- 1년에 1번 전원 차단 → 기대 손실 = 연간 0.5초치 명령어
- 초당 10,000건 처리 서비스라면 → 최대 10,000건 손실 (연간)
- 비즈니스가 이 손실을 허용할 수 있는지 판단이 핵심

</details>

---

**Q3.** RDB 파일은 있지만 AOF 파일이 없는 서버에 `appendonly yes`를 설정하고 Redis를 재시작했다. 무슨 일이 발생하는가?

<details>
<summary>해설 보기</summary>

**Redis는 RDB 파일을 로드한 후 AOF를 시작합니다.**

동작 순서:
1. `appendonly yes` 설정 확인
2. `appendonly.aof` 파일 검색 → 없음
3. 폴백: `dump.rdb` 로드 (RDB가 있으므로)
4. RDB 로드 완료 후 `appendonly.aof` 파일 생성 시작
5. 이후 쓰기 명령어부터 AOF에 기록

```
로그 예시:
  "* DB loaded from disk: 5.678 seconds"  ← RDB 로드
  "* Ready to accept connections"
  → 이후부터 appendonly.aof에 명령어 기록 시작
```

**주의사항:**
이 시점에서 AOF는 비어있지만, RDB 데이터는 메모리에 있습니다. 이후 명령어부터 AOF에 기록됩니다. 재시작 시:
- `appendonly.aof`가 존재하면 → AOF 로드 (RDB 무시)
- `appendonly.aof`가 비어있으면 → 빈 AOF 로드 → **RDB 데이터 손실!**

따라서 AOF가 없는 상태에서 `appendonly yes` 전환 후 재시작하면:
- 첫 재시작: RDB 로드 + 빈 AOF 생성 → 데이터 있음
- 두 번째 재시작: 빈 AOF 로드 → **데이터 없음!**

**안전한 전환 절차:**
```bash
# 1. RDB 저장
redis-cli BGSAVE && sleep 10

# 2. AOF 활성화 + rewrite (RDB를 AOF에 포함)
redis-cli CONFIG SET appendonly yes
redis-cli BGREWRITEAOF
sleep 60  # Rewrite 완료 대기

# 3. 이제 AOF에 RDB 데이터가 포함됨
redis-cli INFO persistence | grep aof_current_size  # 비어있지 않음 확인
```

</details>

---

<div align="center">

**[⬅️ 이전: RDB+AOF 혼합 모드](./03-rdb-aof-mixed.md)** | **[홈으로 🏠](../README.md)** | **[다음: 영속성 설정 실전 가이드 ➡️](./05-persistence-config-guide.md)**

</div>
