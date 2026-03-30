# RDB 스냅샷 — BGSAVE와 fork() Copy-On-Write

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `BGSAVE`가 `fork()`로 자식 프로세스를 만들어 메모리 일관성을 유지하는 원리는?
- Copy-On-Write(COW)가 스냅샷 중 쓰기 요청이 들어올 때 어떻게 동작하는가?
- fork() 비용이 메모리 크기에 비례하는 이유는?
- `save 60 1000` 설정의 정확한 의미와 자동 스냅샷이 트리거되는 조건은?
- RDB 파일 포맷의 구조와 각 섹션이 담는 내용은?
- `BGSAVE` vs `SAVE` vs `BGSAVE SCHEDULE`의 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

BGSAVE는 `fork()`로 자식 프로세스를 만들어 Copy-On-Write 방식으로 스냅샷을 저장한다. 이 원리를 모르면 BGSAVE 중 메모리가 폭증해 Redis가 OOM으로 종료되거나, 수백 ms의 주기적 지연이 왜 발생하는지 설명할 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: maxmemory를 물리 RAM에 가깝게 설정 후 BGSAVE 중 OOM

  설정: maxmemory 14gb (물리 RAM 16 GB)
  현상: 주기적으로 Redis 프로세스가 갑자기 종료됨
  로그: kernel: Out of memory: Kill process (redis-server)

  잘못된 원인 분석:
    "Redis 버그인가?" → 버전 업그레이드
    "메모리가 부족하다" → 서버 업그레이드 (비용 낭비)

  실제 원인:
    BGSAVE 중 fork() → COW로 쓰기 발생한 페이지 복사
    used_memory 14 GB + COW 복사본 ~3 GB = 17 GB > 물리 RAM 16 GB
    → OOM Killer 개입 → Redis 종료

실수 2: 운영 환경에서 SAVE(동기) 실행

  코드/스크립트: redis-cli SAVE  # "빠르게 백업하려고"
  결과: SAVE 완료 전까지 모든 클라이언트 요청 차단
        용량에 따라 수십 초~수 분 동안 Redis 응답 없음
        → 전체 서비스 장애

실수 3: BGSAVE 중 성능 저하 원인 오해

  현상: 주기적으로 수백 ms 응답 지연 발생 (save 설정에 맞춰 반복)
  잘못된 분석: "메모리가 부족하다", "네트워크 문제"
  실제 원인: fork() 시스템 콜 자체가 메모리 크기에 비례해 블로킹
             8 GB Redis → fork() ~200~400 ms 동안 이벤트 루프 정지
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
maxmemory = 물리 RAM × 0.5 (persistence 사용 시):
  16 GB 서버 → maxmemory 8gb
  BGSAVE 중 COW 최악: +8 GB → 총 16 GB = 물리 RAM 안전

fork() 지연 측정 및 모니터링:
  INFO stats | grep latest_fork_usec
  # latest_fork_usec: 350000 → 350 ms
  # 허용 가능한 범위인지 확인 (SLA 기준)

BGSAVE 지연 완화:
  Replica에서 BGSAVE 실행 (Master 부하 없음):
    Master: CONFIG SET save ""         # RDB 없음
    Replica: CONFIG SET save "900 1"   # Replica에서만 스냅샷

  THP(Transparent Huge Pages) 비활성화:
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
    # 큰 페이지 → COW 시 더 많은 복사 → 비활성화 권장

RDB 파일 검증:
  redis-check-rdb dump.rdb
  → 손상 여부 확인 (재시작 전 필수)
```

---

## 🔬 내부 동작 원리

### 1. BGSAVE 전체 흐름

```
BGSAVE 명령어 수신 시:

  메인 프로세스 (계속 요청 처리)
        │
        │ fork() 호출
        ▼
  ┌─────────────────────────────────────────────────────────┐
  │                  fork() 이후 상태                        │
  │                                                         │
  │  부모 프로세스 (메인, Redis 서버)                          │
  │  → 계속 클라이언트 요청 처리                               │
  │  → 새 쓰기 명령어 처리 가능                                │
  │                                                         │
  │  자식 프로세스 (BGSAVE 전용)                              │
  │  → 메모리 스냅샷을 RDB 파일로 기록                         │
  │  → 완료 후 부모에게 SIGCHLD 전송 후 종료                   │
  └─────────────────────────────────────────────────────────┘

  단계:
  1. 부모: fork() 호출 → 자식 프로세스 생성
  2. 자식: fork() 직후의 메모리 상태를 RDB 파일로 직렬화
           dump.rdb.tmp (임시 파일)에 기록
  3. 자식: 완료 → dump.rdb.tmp → dump.rdb (atomic rename)
  4. 부모: SIGCHLD 수신 → "Background saving terminated with success" 로그

  임시 파일 후 rename의 이유:
    직접 dump.rdb에 쓰면 기록 중 Redis 재시작 시 → 불완전한 RDB
    임시 파일 완성 후 rename → 실패해도 이전 dump.rdb 보존
    rename은 OS에서 원자적 (atomic) 연산 보장

  상태 확인:
  redis-cli LASTSAVE          # 마지막 성공한 RDB 저장 Unix timestamp
  redis-cli INFO persistence  # rdb_bgsave_in_progress:1 (진행 중)
```

### 2. fork()와 Copy-On-Write의 원리

```
fork() 이전 — 부모 프로세스 메모리:

  가상 주소 공간:
  ┌──────────────────────────────────────┐
  │ 페이지 A (4 KB): key1="hello"        │→ 물리 RAM 페이지 A
  │ 페이지 B (4 KB): key2="world"        │→ 물리 RAM 페이지 B
  │ 페이지 C (4 KB): key3=42             │→ 물리 RAM 페이지 C
  │ ...                                  │
  └──────────────────────────────────────┘

fork() 직후 — 부모와 자식이 동일 물리 페이지 공유:

  부모 프로세스              자식 프로세스
  ┌──────────┐              ┌──────────┐
  │ 가상 A   │              │ 가상 A'  │
  │ 가상 B   │              │ 가상 B'  │
  │ 가상 C   │              │ 가상 C'  │
  └──┬───────┘              └──┬───────┘
     │                         │
     └──────┬──────────────────┘
            │ (같은 물리 페이지를 공유)
     ┌──────▼──────────────────┐
     │ 물리 RAM                │
     │ 페이지 A: key1="hello"  │
     │ 페이지 B: key2="world"  │
     │ 페이지 C: key3=42       │
     └─────────────────────────┘

  OS가 모든 공유 페이지를 "읽기 전용(Read-Only)"으로 마킹
  → 어느 쪽도 쓸 수 없는 상태

Copy-On-Write 발생 시 — 부모가 key1 수정:

  부모: SET key1 "modified"
  → 페이지 A에 쓰기 시도
  → OS: "읽기 전용 페이지에 쓰기!" → Page Fault 발생
  → OS: 페이지 A를 복사 → 새 물리 페이지 A' 생성
  → 부모의 가상 A → 물리 A' (새 복사본)
  → 자식의 가상 A' → 물리 A (원본, 변경 안 됨!)
  → 부모가 A'에 "modified" 기록
  
  결과:
  부모: 페이지 A'에 "modified" (최신 상태)
  자식: 페이지 A에 "hello" (fork 시점 상태 유지)
  → 자식이 "hello"를 RDB에 기록 → 스냅샷 일관성 보장!

  수정된 페이지가 많을수록:
  → 복사 페이지 ↑ → 부모의 실제 메모리 사용 ↑
  → BGSAVE 중 쓰기가 많으면 (hot write) → COW 복사 급증
  → 메모리 사용 급증 → OOM 위험
```

### 3. fork() 비용 — 왜 메모리 크기에 비례하는가

```
fork() 자체의 비용:

  Linux fork():
    실제 물리 메모리를 복사하지 않음 (COW 덕분에)
    하지만 페이지 테이블(Page Table)을 복사해야 함

  페이지 테이블:
    가상 주소 → 물리 주소 매핑 정보
    메모리 4 GB → 페이지 수 = 4 GB / 4 KB = 1,000,000 페이지
    페이지 테이블 엔트리: ~8 bytes × 1,000,000 = ~8 MB

  fork() 비용:
    페이지 테이블 복사: O(메모리 크기 / 페이지 크기)
    + 모든 페이지를 read-only로 마킹: O(페이지 수)
    
    실제 측정값:
    Redis 1 GB → fork(): ~10~50 ms
    Redis 4 GB → fork(): ~100~500 ms (최악 수 초)
    Redis 16 GB → fork(): ~수 초

  Transparent Huge Pages (THP)의 문제:
    Linux 기본 페이지: 4 KB
    THP: 2 MB 페이지로 묶어 페이지 테이블 크기 감소
    
    문제: COW 발생 시 2 MB 전체를 복사
    4 KB 수정 → 4 KB 복사 (THP 없을 때)
    4 KB 수정 → 2 MB 복사 (THP 있을 때) → 500배 메모리 복사!
    
    Redis 권장 설정:
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
    → THP 비활성화
    
    확인:
    cat /sys/kernel/mm/transparent_hugepage/enabled
    # [always] madvise never  → THP 활성 (Redis에 나쁨)
    # always madvise [never]  → THP 비활성 (Redis에 좋음)

fork() 성능 모니터링:
  redis-cli INFO stats | grep latest_fork_usec
  # latest_fork_usec:50234   ← 마지막 fork() 소요 시간 (마이크로초)
  # 50234 μs = 50 ms → 괜찮은 수준
  # 1,000,000 μs = 1초 이상 → 메모리 감소 또는 THP 비활성화 필요
```

### 4. save 설정과 자동 스냅샷 트리거

```
redis.conf의 save 설정:
  save <seconds> <changes>
  의미: <seconds>초 안에 <changes>개 이상의 키 변경이 발생하면 BGSAVE

기본 설정:
  save 3600 1     # 1시간 안에 1개 이상 변경 시
  save 300 100    # 5분 안에 100개 이상 변경 시
  save 60 10000   # 1분 안에 10,000개 이상 변경 시

  이 조건 중 하나라도 충족되면 자동 BGSAVE 실행

내부 동작:
  serverCron()이 주기적으로 체크:
    dirty counter: 마지막 스냅샷 이후 변경된 키 수
    lastsave: 마지막 성공한 스냅샷 Unix timestamp
    
    if (현재시각 - lastsave >= seconds) AND (dirty >= changes):
        BGSAVE 트리거

dirty counter 갱신:
  SET key value      → dirty += 1
  MSET k1 v1 k2 v2  → dirty += 2
  DEL key            → dirty += 1
  INCR key           → dirty += 1
  EXPIRE key 60      → dirty += 1  (만료 설정도 변경으로 카운트)

save 비활성화:
  save ""   # 빈 문자열로 모든 save 규칙 제거 → 자동 스냅샷 없음
  또는 redis.conf에서 save 라인 모두 삭제

수동 스냅샷:
  SAVE           # 동기 (메인 스레드 블로킹, 이벤트 루프 중단!)
  BGSAVE         # 비동기 (fork() → 자식이 저장, 메인 스레드 계속 동작)
  BGSAVE SCHEDULE # 이미 BGSAVE 진행 중이면 완료 후 다시 실행 예약
  
  SAVE는 운영 환경에서 절대 사용 금지:
  → 수 GB 데이터 → SAVE 수 초~수 분 동안 Redis 완전 정지
```

### 5. RDB 파일 포맷

```
RDB 파일 구조:

  [MAGIC NUMBER]    "REDIS0011" (버전 11)
  [AUX FIELDS]      메타데이터 (Redis 버전, 생성 시각, 메모리 사용량 등)
  [SELECT DB n]     데이터베이스 번호 (0~15)
  [RESIZE DB]       해시 테이블 크기 힌트 (빠른 로드용)
  [KEY-VALUE PAIRS] 실제 데이터
    - 만료 시각 (있으면)
    - 값 타입
    - 키 (문자열)
    - 값 (타입별 직렬화)
  [SELECT DB n+1]   다음 DB
  ...
  [EOF]             0xFF
  [CRC64 CHECKSUM]  무결성 검증용 8 bytes

값 인코딩 (효율적 압축):
  String: 정수이면 정수 인코딩 (1~4 bytes), 아니면 LZF 압축 옵션
  List/Hash/Set: 각 원소를 RLE 또는 길이-접두사 방식으로 직렬화
  
  rdbcompression yes (기본):
    긴 String 값을 LZF로 압축
    CPU 약간 증가, 파일 크기 감소
  
  rdbchecksum yes (기본):
    CRC64로 파일 무결성 검증
    로드 시간 약 10% 증가 (CRC 계산)
    손상 감지 가능 → 비활성화 비권장

파일 크기:
  메모리 사용량 < RDB 파일 크기 (항상)
  이유: RDB = 직렬화 + 압축, 메모리 = raw 데이터 + Redis 오버헤드
  
  실제: 4 GB 메모리 → RDB 1~2 GB (데이터 특성에 따라)
        문자열 위주: 압축 효율 높음 → 더 작음
        이미 압축된 바이너리: 거의 압축 안 됨

RDB 로드 속도:
  바이너리 포맷 → AOF(텍스트 명령어 재생)보다 훨씬 빠름
  4 GB RDB 로드: ~수십 초
  4 GB AOF 로드: ~수 분 이상
```

### 6. BGSAVE 중 상태 모니터링

```
INFO persistence 주요 필드:

  rdb_changes_since_last_save:5234    # 마지막 스냅샷 이후 변경 수
  rdb_bgsave_in_progress:1            # 1 = BGSAVE 진행 중
  rdb_last_bgsave_time_sec:-1         # 마지막 BGSAVE 소요 시간 (-1 = 없음)
  rdb_current_bgsave_time_sec:12      # 현재 진행 중인 BGSAVE 경과 시간
  rdb_last_bgsave_status:ok           # 마지막 BGSAVE 결과
  rdb_last_cow_size:12345678          # 마지막 BGSAVE 중 COW로 복사된 bytes

  rdb_last_cow_size가 큰 경우:
    BGSAVE 중 쓰기가 많았음 → 많은 페이지가 COW 복사됨
    used_memory 급증 → maxmemory에 근접 위험
    → 스냅샷 주기를 트래픽 적은 시간으로 이동
    → 또는 RDB 대신 AOF 단독 사용 고려

BGSAVE 실패 시 동작:
  rdb_last_bgsave_status:err
  → Redis는 이후 모든 쓰기 명령어를 거부! (기본 동작)
  → 이유: 데이터 손실 위험 경고 + 운영자 주의 촉구
  
  설정으로 변경:
  stop-writes-on-bgsave-error no   # BGSAVE 실패해도 쓰기 계속 허용
  → 데이터 손실 감수하고 가용성 유지 시 사용
```

---

## 💻 실전 실험

### 실험 1: BGSAVE 동작 관찰

```bash
# 데이터 준비
for i in $(seq 1 10000); do redis-cli SET "key:$i" "value-$i" > /dev/null; done
redis-cli DBSIZE  # 10000

# BGSAVE 실행 및 모니터링
redis-cli BGSAVE   # "Background saving started"
redis-cli LASTSAVE # 이전 저장 시각 (Unix timestamp)

# 진행 상태 확인
watch -n 1 'redis-cli INFO persistence | grep -E "rdb_bgsave|rdb_last"'

# BGSAVE 완료 대기
sleep 5
redis-cli LASTSAVE  # 업데이트된 저장 시각 확인

# RDB 파일 확인
ls -lh /var/lib/redis/dump.rdb  # 또는 redis-cli CONFIG GET dir
file /var/lib/redis/dump.rdb    # "Redis RDB file version 11"
```

### 실험 2: fork() 비용 측정

```bash
# 대용량 데이터 생성
redis-cli FLUSHDB
for i in $(seq 1 500000); do
  redis-cli SET "bigkey:$i" "$(python3 -c "print('x'*100)")" > /dev/null
done

# fork() 시간 측정
redis-cli BGSAVE
redis-cli INFO stats | grep latest_fork_usec

# 스냅샷 완료 후 COW 복사량 확인
redis-cli INFO persistence | grep rdb_last_cow_size

# 동시에 쓰기 실행 (COW 증가 유발)
redis-cli BGSAVE
# 동시에 쓰기 실행
for i in $(seq 1 50000); do redis-cli SET "write_during:$i" "v" > /dev/null; done
# BGSAVE 완료 후
redis-cli INFO persistence | grep rdb_last_cow_size
# COW 복사량이 증가했는지 확인
```

### 실험 3: save 설정으로 자동 스냅샷 관찰

```bash
# save 조건 설정 (5초 안에 5개 변경)
redis-cli CONFIG SET save "5 5"
redis-cli CONFIG GET save

# dirty counter 확인
redis-cli INFO persistence | grep rdb_changes_since_last_save

# 5개 키 변경
for i in $(seq 1 5); do redis-cli SET "auto:$i" "v" > /dev/null; done

# 5초 대기 후 자동 BGSAVE 확인
sleep 6
redis-cli LASTSAVE  # 자동 저장 시각 업데이트됨
redis-cli INFO persistence | grep -E "rdb_bgsave|rdb_changes"

# 기본 설정 복원
redis-cli CONFIG SET save "3600 1 300 100 60 10000"
```

### 실험 4: RDB 파일 검사

```bash
# RDB 파일 위치 확인
REDIS_DIR=$(redis-cli CONFIG GET dir | tail -1)
RDB_FILE=$(redis-cli CONFIG GET dbfilename | tail -1)
echo "RDB 파일: $REDIS_DIR/$RDB_FILE"

# 파일 정보
ls -lh "$REDIS_DIR/$RDB_FILE"
hexdump -C "$REDIS_DIR/$RDB_FILE" | head -5
# 처음 9 bytes: "REDIS0011" (MAGIC + VERSION)

# RDB 무결성 검사
redis-check-rdb "$REDIS_DIR/$RDB_FILE"
# 출력: [offset 0] Checking RDB file dump.rdb
#       [offset 26] AUX FIELD redis-ver: '7.0.x'
#       ...
#       [info] Checksum OK

# 손상 시뮬레이션 (주의: 테스트 환경에서만!)
# cp dump.rdb dump.rdb.backup
# printf '\xFF\xFF' | dd of=dump.rdb bs=1 seek=10 conv=notrunc 2>/dev/null
# redis-check-rdb dump.rdb  # "CRC error"
```

---

## 📊 성능 비교

```
SAVE vs BGSAVE vs BGSAVE SCHEDULE:

명령어           | 블로킹 여부      | 사용 조건
────────────────┼─────────────────┼──────────────────────────────
SAVE             | Yes (전체 블로킹) | 절대 운영 환경 금지
BGSAVE           | No (fork 순간만) | 수동 스냅샷 표준
BGSAVE SCHEDULE  | No              | BGSAVE 중복 실행 방지용

fork() 시간 기준 (Linux, 물리 메모리 기준):
  1 GB: ~10~30 ms
  4 GB: ~50~200 ms (THP 비활성화 시)
        ~200~2000 ms (THP 활성화 시)
  16 GB: ~200~1000 ms

COW 메모리 증가:
  BGSAVE 중 쓰기 없음: COW 0 → 메모리 거의 증가 없음
  BGSAVE 중 10% 키 수정: ~10% 메모리 추가 (수정된 페이지 복사)
  BGSAVE 중 50% 키 수정: ~50% 메모리 추가 → 위험

RDB vs AOF 복구 속도:
  RDB 4 GB 로드: ~30~60초
  AOF 4 GB 로드: ~5~15분 (명령어 재실행)
  → 빠른 재시작이 필요하면 RDB 필수
```

---

## ⚖️ 트레이드오프

```
RDB 스냅샷의 장단점:

장점:
  ① 빠른 재시작: 바이너리 포맷 → AOF보다 10~20배 빠른 로드
  ② 컴팩트한 파일: LZF 압축으로 메모리 사용량보다 작음
  ③ 특정 시점 백업: 시간별 스냅샷 보관 가능
  ④ 단순한 구조: 하나의 파일로 완전한 상태 표현

단점:
  ① 데이터 손실: 마지막 스냅샷 이후 변경사항 손실
     save 60 10000 → 최대 1분 데이터 손실 (10,000개 변경 미만 시 더 길 수 있음)
  ② fork() 비용: 메모리 클수록 fork 지연
  ③ COW 메모리: 스냅샷 중 쓰기 많으면 메모리 급증

데이터 손실 허용 범위 계산:
  save 3600 1 + save 300 100 + save 60 10000 설정 시:
    최악의 경우: 초당 9,999개 변경이 1시간 동안 발생
    → 60초 조건 (10,000개) 미충족
    → 300초 조건 (100개) 충족되어 5분마다 저장
    → 최대 손실: 5분치 데이터
    
  중요 데이터: save + AOF 함께 (다음 문서 참고)
  캐시 전용: save 비활성화 가능
```

---

## 📌 핵심 정리

```
RDB 스냅샷 핵심:

BGSAVE 동작:
  fork() → 자식 프로세스 → RDB 파일 기록 → 완료 → rename
  부모는 fork 후 즉시 요청 처리 재개 (이벤트 루프 중단 없음)

Copy-On-Write:
  fork 후 부모+자식이 물리 페이지 공유 (read-only)
  부모가 페이지 수정 시 OS가 해당 페이지를 복사 → 자식은 원본 유지
  → 스냅샷 일관성 보장 (fork 시점의 메모리 상태 기록)

fork() 비용:
  페이지 테이블 복사: O(메모리 크기)
  THP 활성 시 큰 문제 → echo never > .../transparent_hugepage/enabled
  latest_fork_usec로 모니터링

save 설정:
  save <seconds> <changes> → 조건 충족 시 자동 BGSAVE
  save "" → 자동 스냅샷 비활성화

데이터 손실:
  마지막 BGSAVE 이후 변경사항 손실 가능
  손실 허용 불가 → AOF 추가 (다음 문서)

진단:
  INFO persistence         → BGSAVE 상태, COW 크기, 마지막 저장 시각
  INFO stats               → latest_fork_usec
  redis-cli LASTSAVE       → 마지막 성공 저장 Unix timestamp
  redis-check-rdb file     → RDB 파일 무결성 검사
```

---

## 🤔 생각해볼 문제

**Q1.** Redis 메모리 사용량이 8 GB이고 서버의 물리 RAM이 16 GB인 상황에서 BGSAVE를 실행했더니 OOM Killer가 Redis를 종료했다. 왜 8 + 8 = 16 GB로 충분할 것 같았는데 OOM이 발생했는가?

<details>
<summary>해설 보기</summary>

세 가지 메모리 소비 요인이 동시에 작용했습니다:

**1. OS와 다른 프로세스 메모리**: 물리 RAM 16 GB 전부가 Redis에 쓸 수 있는 게 아닙니다. OS 커널, sshd, 모니터링 에이전트 등이 수백 MB ~ 수 GB를 사용합니다.

**2. COW 복사 비용**: BGSAVE 중 쓰기가 발생하면 수정된 페이지가 복사됩니다. 쓰기가 많은 시간대에 BGSAVE가 실행되면 수 GB의 COW 복사가 추가됩니다.

**3. AOF rewrite 동시 실행 가능성**: AOF도 활성화되어 있다면 BGSAVE와 AOF rewrite가 동시에 fork()할 수 있습니다 (일반적으로 Redis는 하나씩 처리하지만, 조건에 따라 겹칠 수 있습니다).

**해결책:**
```bash
# maxmemory를 물리 RAM의 50%로 설정
redis-cli CONFIG SET maxmemory 6gb  # 16GB의 약 37%~50%

# THP 비활성화 (COW 복사량 감소)
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# BGSAVE 시간을 트래픽 최저 시간대로 이동
# 또는 Replica에서 BGSAVE 실행 (Master는 BGSAVE 없이)
```

</details>

---

**Q2.** `save 60 10000` 설정에서 59초 동안 9,999개의 키를 변경했다가 60초째에 1개를 더 변경했다. BGSAVE가 즉시 실행되는가?

<details>
<summary>해설 보기</summary>

즉시 실행되지 않습니다. `serverCron()`이 `hz` 설정에 따라 주기적으로 조건을 체크합니다 (기본 hz=10 → 100ms마다 체크).

60초째에 10,000번째 변경 → 다음 serverCron() 실행 시 조건 확인 → BGSAVE 시작. 따라서 실제 BGSAVE 시작 시각은 60초 이후 최대 100ms (= 1/hz) 지연이 있을 수 있습니다.

또한 이미 BGSAVE가 진행 중이라면 해당 BGSAVE가 완료된 후에 다음 조건 체크가 의미를 가집니다. Redis는 동시에 두 개의 BGSAVE를 실행하지 않습니다.

```bash
# dirty counter 실시간 확인
redis-cli INFO persistence | grep rdb_changes_since_last_save
# 조건 충족 후 serverCron 주기(100ms~1s)내에 BGSAVE 시작
```

</details>

---

**Q3.** 자식 프로세스가 RDB를 쓰는 동안 부모(Redis 메인 프로세스)가 FLUSHALL 명령어를 받았다. RDB 파일에는 무엇이 기록되는가?

<details>
<summary>해설 보기</summary>

RDB 파일에는 **BGSAVE 시점(fork 시점)의 완전한 스냅샷**이 기록됩니다. FLUSHALL의 영향을 받지 않습니다.

이유: fork() 후 자식은 자신의 독립적인 메모리 뷰(fork 시점의 스냅샷)를 가집니다. 부모가 FLUSHALL로 모든 키를 삭제해도:
- 자식의 페이지 테이블은 여전히 원본 물리 페이지를 가리킵니다
- FLUSHALL은 부모의 dict에서 키를 제거하고 값 객체의 참조 카운트를 줄이지만, 자식이 참조 중인 페이지의 물리 데이터는 유지됩니다 (COW 보호)
- 자식은 fork 시점의 모든 데이터를 RDB에 기록합니다

결과적으로:
- RDB 파일: fork 시점의 모든 데이터 포함 (FLUSHALL 이전 상태)
- 현재 Redis 메모리: 비어있음 (FLUSHALL 실행됨)

이것이 RDB를 사용하는 재해 복구 시 주의점입니다. "가장 최근 RDB"가 현재 상태보다 많은 데이터를 가질 수 있습니다.

</details>

---

<div align="center">

**[⬅️ Chapter 2: 데이터 구조 선택 가이드](../data-structures/07-data-structure-guide.md)** | **[홈으로 🏠](../README.md)** | **[다음: AOF — fsync 정책과 내구성 ➡️](./02-aof-fsync.md)**

</div>
