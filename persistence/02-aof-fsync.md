# AOF — fsync 정책과 내구성 트레이드오프

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- AOF가 쓰기 명령어를 어떤 형식으로 기록하는가?
- `appendfsync always/everysec/no` 3가지 정책별 데이터 손실 범위와 성능 차이는?
- OS 커널의 page cache가 fsync와 어떤 관계인가?
- AOF Rewrite가 파일을 압축하는 원리와 `BGREWRITEAOF` 동작 과정은?
- AOF 파일이 커지면 Redis 재시작이 느려지는 이유와 해결책은?
- `auto-aof-rewrite-percentage`와 `auto-aof-rewrite-min-size` 설정의 의미는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`appendfsync` 정책 하나가 "최대 데이터 손실 범위"와 "초당 처리량"을 동시에 결정한다. 이 트레이드오프를 이해하지 못하면 결제 데이터에 `appendfsync no`를 쓰거나, 캐시에 `appendfsync always`로 성능을 낭비하는 실수를 저지른다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 중요 데이터에 appendfsync no 또는 AOF 미사용

  설정: appendonly yes / appendfsync no  (또는 appendonly no)
  
  장애: 새벽 3시 서버 전원 차단 (UPS 고장)
  결과: 마지막 OS flush(~30초) 이후 명령어 손실
        또는 마지막 BGSAVE(~15분 전) 이후 전체 손실
        → 결제 완료 됐지만 상태 미저장 → 환불 처리 지옥

  사후 분석: "appendfsync everysec 이었으면 최대 1초 손실이었다"

실수 2: 고쓰기 워크로드에 appendfsync always 설정 후 성능 저하

  설정: appendfsync always
  워크로드: 초당 50,000 SET 요청 (대용량 캐시)
  결과: 매 SET마다 fsync → 디스크 I/O 포화
        HDD: 초당 수백 건으로 처리량 급감 (fsync = 회전 대기)
        SSD: 초당 수천~수만 건 (그나마 낫지만 불필요한 부담)
        → "Redis가 느리다" → 원인을 appendfsync로 특정 못 함

실수 3: AOF Rewrite 중 디스크 공간 부족

  현상: BGREWRITEAOF 실행 후 Redis 로그에 에러 발생
        AOF 파일이 계속 커지고 Rewrite가 계속 실패
  원인: Rewrite 중 임시 파일 = 현재 AOF 크기만큼 추가 필요
        디스크 여유 공간 부족 → Rewrite 실패 → 원본 AOF 계속 증가
        → 디스크 풀 → Redis 쓰기 불가
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
fsync 정책 결정 흐름:
  "최대 손실 허용 범위가 얼마인가?"
  → 0초  (절대 손실 불가) → appendfsync always + SSD 필수
  → ~1초  (약간 허용)    → appendfsync everysec (대부분 서비스)
  → 수십초 (상당히 허용)  → appendfsync no
  → 수 분  (크게 허용)    → RDB only (appendonly no)

서비스별 추천:
  결제/주문:   appendfsync always (SSD + 초당 쓰기 수천 건 이하)
  세션:       appendfsync everysec
  캐시:       appendonly no (손실 허용)

AOF Rewrite 디스크 계획:
  최소 디스크 여유 = 현재 AOF 크기 × 2 이상
  auto-aof-rewrite-percentage 100 (기본)
  auto-aof-rewrite-min-size 64mb

성능 측정으로 검증:
  redis-benchmark -c 50 -n 100000 -t set
  appendfsync always → no 비교
  → SSD에서 always도 수만 ops/sec 가능한지 확인
```

---

## 🔬 내부 동작 원리

### 1. AOF 기록 형식 — RESP 프로토콜 텍스트

```
AOF 파일 = 쓰기 명령어들의 RESP 형식 텍스트 로그

예시: 다음 명령어 실행 시
  SET user:1 "Alice"
  EXPIRE user:1 3600
  HSET order:1 status paid amount 29900

AOF 파일에 기록되는 내용:
  *3\r\n          ← 3개 인자
  $3\r\n
  SET\r\n
  $6\r\n
  user:1\r\n
  $5\r\n
  Alice\r\n
  *3\r\n          ← EXPIRE 명령어
  $6\r\n
  EXPIRE\r\n
  $6\r\n
  user:1\r\n
  $4\r\n
  3600\r\n
  *6\r\n          ← HSET 명령어
  $4\r\n
  HSET\r\n
  ...

특징:
  사람이 읽을 수 있는 텍스트 → 장애 분석 용이
  모든 쓰기 명령어를 순서대로 추가(append-only)
  삭제 없음 → 파일이 계속 커짐 → AOF Rewrite 필요

기록 시점:
  명령어 실행 → 메모리 반영 → AOF 버퍼에 추가
  버퍼 → 파일 write() 호출 (OS 커널 page cache)
  fsync 정책에 따라 page cache → 디스크 강제 flush
```

### 2. 쓰기 경로와 fsync의 역할

```
데이터가 디스크에 도달하는 경로:

클라이언트 → Redis 이벤트 루프 → 명령어 실행 → AOF 버퍼
                                                   │
                                              write() 시스템 콜
                                                   │
                                            OS 커널 page cache
                                            (메모리 버퍼링)
                                                   │
                                    fsync() 호출 시 강제 flush
                                                   │
                                               디스크 (SSD/HDD)

  ① write()만 하면:
     OS page cache에만 기록됨
     Redis 정상 종료: OS가 flush → 디스크 기록 → 안전
     서버 전원 차단 / 커널 패닉: page cache 소실 → 데이터 손실!

  ② fsync() 호출:
     OS page cache → 디스크 강제 동기화
     완료 후 데이터가 디스크에 확실히 존재
     하지만 시간 비용: SSD 0.1~1 ms, HDD 수 ms

page cache의 역할:
  쓰기: 메모리 → page cache (빠름) → 나중에 디스크 (비동기)
  fsync: page cache → 디스크 즉시 동기화 (느림)
  
  OS 자동 flush 시점: dirty page가 일정 비율 이상, 또는 일정 시간 경과
  Linux 기본: dirty_expire_centisecs = 3000 (30초)
  → fsync no = 최대 30초 데이터가 page cache에만 존재
```

### 3. appendfsync 3가지 정책 완전 분석

```
정책 1: appendfsync always

  동작:
    명령어 실행 → AOF 버퍼 기록 → write() → fsync() 즉시
    (이벤트 루프 실행마다 fsync 호출)
  
  내구성:
    어떤 장애에서도 마지막으로 fsync된 명령어까지 복구
    전원 차단 직전 1ms 전에 실행한 명령어도 복구 가능
    → 데이터 손실: 이론적으로 0
  
  성능 영향:
    명령어당 1회 fsync
    SSD: fsync 비용 ~0.1~1 ms → 초당 최대 1,000~10,000 ops
    HDD: fsync 비용 ~수 ms → 초당 수백 ops
    → 일반 메모리 연산 대비 10~100배 느림
    → 처리량이 낮아도 되는 결제/감사 로그에 적합

정책 2: appendfsync everysec (기본값)

  동작:
    명령어 실행 → AOF 버퍼 기록
    별도 백그라운드 스레드가 1초마다 fsync
    (write()는 이벤트 루프에서, fsync는 별도 스레드)
  
  내구성:
    최악의 경우: 서버 전원 차단 시 최대 1초치 명령어 손실
    일반 장애(Redis 프로세스 종료): OS가 page cache flush → 손실 없음
    → 실제 데이터 손실: 전원 차단 등 극단적 장애 + 마지막 fsync 후 1초
  
  성능 영향:
    write(): 이벤트 루프에서 비동기 (빠름)
    fsync(): 백그라운드 스레드에서 1초마다 (이벤트 루프 영향 없음)
    → 대부분의 워크로드에서 거의 성능 저하 없음
    → 권장 기본값

정책 3: appendfsync no

  동작:
    명령어 실행 → AOF 버퍼 기록 → write()만 (fsync 안 함)
    OS가 자체 판단으로 page cache를 디스크에 flush
  
  내구성:
    Redis 정상 종료: OS가 flush → 데이터 보존
    서버 전원 차단: 최대 30초 (Linux dirty_expire_centisecs) 손실
    → 캐시 전용 또는 손실 허용 서비스에 적합
  
  성능 영향:
    fsync 없음 → 최대 처리량
    write()도 OS 버퍼링으로 빠름
    → 거의 메모리 속도에 근접

fsync 정책 비교표:
  정책        | 데이터 손실       | 성능     | 권장 용도
  ───────────┼─────────────────┼─────────┼────────────────────
  always      | 0               | 낮음     | 결제, 주문, 금융
  everysec    | 최대 1초        | 높음     | 세션, 일반 서비스 (기본)
  no          | 최대 ~30초      | 최고     | 캐시, 손실 허용 서비스
```

### 4. AOF 버퍼와 write() 타이밍

```
AOF 버퍼의 존재 이유:
  매 명령어마다 write() 시스템 콜은 오버헤드
  → 여러 명령어를 버퍼에 모아 한 번에 write()

Redis AOF 쓰기 흐름:

  이벤트 루프 한 사이클에서:
  1. 여러 클라이언트 명령어 처리 (메모리 반영)
  2. 각 명령어를 aof_buf(버퍼)에 추가
  3. 이벤트 루프 끝에서 flushAppendOnlyFile() 호출:
     → aof_buf의 내용을 write()로 OS page cache에 씀
     → appendfsync 정책에 따라 fsync 여부 결정

  appendfsync always일 때:
    flushAppendOnlyFile()에서 write() + fsync() 순서로 호출
    
  appendfsync everysec일 때:
    flushAppendOnlyFile()에서 write()만
    백그라운드 I/O 스레드가 1초마다 fsync()
    
    단, fsync가 2초 이상 지연되면 이벤트 루프가 잠시 대기
    (2초 이상 지연 = 디스크 I/O 문제 신호)

  appendfsync no일 때:
    flushAppendOnlyFile()에서 write()만
    fsync 없음

no-appendfsync-on-rewrite yes (기본값):
  AOF Rewrite 중에는 fsync 실행하지 않음
  이유: Rewrite 자체가 이미 많은 I/O → 추가 fsync 시 성능 저하
  영향: Rewrite 중 (수 분) 동안 appendfsync always/everysec도 실질적으로 no처럼 동작
  → Rewrite 중 장애 시 최대 Rewrite 시간만큼 손실 가능
  
  no-appendfsync-on-rewrite no:
    Rewrite 중에도 fsync → 더 안전하지만 I/O 부하 증가
```

### 5. AOF Rewrite — 파일 압축의 원리

```
AOF 파일이 계속 커지는 문제:
  INCR counter    → 1,000번 실행 → 1,000줄
  SET key value   → 10번 SET + 5번 DEL → 15줄이지만 현재 값은 마지막 SET

  실제로 필요한 것:
    "현재 메모리 상태를 재현하는 최소 명령어 집합"
    INCR counter 1,000번 → SET counter 1000 (단 1줄!)
    SET key "v1"~"v10" + DEL key 5번 → SET key "v10" (1줄) 또는 없음

AOF Rewrite 과정:

  1. BGREWRITEAOF 명령어 수신 (또는 자동 트리거)
  
  2. fork() → 자식 프로세스 생성
  
  3. 자식: 현재 메모리 상태를 "재현하는 명령어"로 직렬화
     → 새 AOF 파일(aof.tmp) 작성
     이 시점의 메모리 상태만 보고 현재 최솟값으로 기록
     예: counter = 1000이면 → "SET counter 1000" 1줄
  
  4. 자식이 aof.tmp 작성 중 → 부모(메인)는 계속 요청 처리
     → 새로운 쓰기 명령어 발생
     → aof_rewrite_buf (별도 버퍼)에 추가로 기록
  
  5. 자식: aof.tmp 작성 완료
     → 부모에게 신호 전송
  
  6. 부모: aof_rewrite_buf의 명령어들을 aof.tmp 끝에 추가
     → aof.tmp → appendonly.aof (atomic rename)
  
  결과:
    이전 AOF: 수 GB (수백만 줄의 누적 명령어)
    새 AOF:   수 MB (현재 상태 재현 명령어 + Rewrite 중 발생 명령어)

  자동 트리거 조건:
  auto-aof-rewrite-percentage 100  # 파일 크기가 기준 대비 100% 증가 시
  auto-aof-rewrite-min-size 64mb   # 최소 파일 크기 (64 MB 이상일 때만)
  
  예: 기준 크기 100 MB → 200 MB 도달 시 Rewrite 시작
      단, 64 MB 미만이면 Rewrite 안 함

Redis 7.0 Multi Part AOF (MP-AOF):
  이전: 단일 AOF 파일
  7.0+: Base AOF + Incremental AOF
    Base AOF: RDB 포맷의 스냅샷 (현재 상태)
    Incremental AOF: Rewrite 이후 추가된 명령어들
  → 여러 파일로 분리해 Rewrite 오버헤드 분산
```

### 6. AOF 파일 크기와 재시작 속도

```
AOF 재시작 복구 과정:
  redis-server 시작
  → appendonly.aof 파일 읽기
  → 각 명령어를 순서대로 재실행
  → 메모리에 최종 상태 복원

재실행 시간:
  AOF 100 MB: 수십 초
  AOF 1 GB:   수 분
  AOF 10 GB:  수십 분 ~ 1시간
  → 대형 파일 = 재시작 느림 = BGREWRITEAOF로 관리 필수

재실행 최적화 (aof-use-rdb-preamble yes):
  이 설정이 활성화되면 AOF Rewrite 시 RDB 포맷으로 Base 작성
  → 재시작 시 RDB 부분은 바이너리 로드 (빠름)
  → 이후 Incremental 부분만 명령어 재실행 (짧음)
  → 전체 재시작 시간 대폭 감소
  (이것이 Mixed Format = 다음 문서 주제)

AOF 파일 관리:
  redis-cli INFO persistence | grep aof
  aof_enabled:1                     # AOF 활성 여부
  aof_rewrite_in_progress:0         # Rewrite 진행 중 여부
  aof_rewrite_scheduled:0           # Rewrite 예약 여부
  aof_last_rewrite_time_sec:45      # 마지막 Rewrite 소요 시간 (초)
  aof_current_size:12345678         # 현재 AOF 파일 크기 (bytes)
  aof_base_size:6000000             # 기준 크기 (마지막 Rewrite 시)
  aof_pending_rewrite:0

  aof_current_size / aof_base_size 비율이 커지면 Rewrite 임박
```

---

## 💻 실전 실험

### 실험 1: AOF 파일 내용 직접 관찰

```bash
# AOF 활성화
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG GET appendonly
redis-cli CONFIG GET dir

# 명령어 실행
redis-cli SET user:1 "Alice"
redis-cli EXPIRE user:1 3600
redis-cli HSET order:1 status paid amount 29900
redis-cli INCR page_views

# AOF 파일 내용 확인
AOF_DIR=$(redis-cli CONFIG GET dir | tail -1)
cat "$AOF_DIR/appendonly.aof" | head -50
# RESP 형식으로 명령어가 기록됨 확인

# 실시간 AOF 기록 관찰
tail -f "$AOF_DIR/appendonly.aof" &
TAIL_PID=$!
redis-cli SET live_key "live_value"
redis-cli DEL live_key
kill $TAIL_PID
```

### 실험 2: appendfsync 정책별 성능 측정

```bash
# always 정책
redis-cli CONFIG SET appendfsync always
redis-cli CONFIG SET appendonly yes
redis-benchmark -t set -n 10000 -c 50 --csv
# 낮은 ops/sec 확인

# everysec 정책  
redis-cli CONFIG SET appendfsync everysec
redis-benchmark -t set -n 10000 -c 50 --csv
# 높은 ops/sec 확인

# no 정책
redis-cli CONFIG SET appendfsync no
redis-benchmark -t set -n 10000 -c 50 --csv
# 최고 ops/sec 확인

# 비교: AOF 없이 (RDB만)
redis-cli CONFIG SET appendonly no
redis-benchmark -t set -n 10000 -c 50 --csv
# RDB만 사용 시 기준값

# fsync 정책별 지연 측정
redis-cli CONFIG SET appendfsync always
redis-cli --latency -i 1  # 실시간 지연 (ms)
```

### 실험 3: BGREWRITEAOF 관찰

```bash
# 많은 쓰기로 AOF 파일 부풀리기
redis-cli CONFIG SET appendonly yes
for i in $(seq 1 10000); do
  redis-cli SET "temp:$i" "value-$i" > /dev/null
done
for i in $(seq 1 5000); do
  redis-cli DEL "temp:$i" > /dev/null
done
for i in $(seq 1 10000); do
  redis-cli INCR counter > /dev/null
done

AOF_DIR=$(redis-cli CONFIG GET dir | tail -1)
ls -lh "$AOF_DIR/appendonly.aof"  # 큰 파일 확인

# Rewrite 실행
redis-cli BGREWRITEAOF
watch -n 1 'redis-cli INFO persistence | grep -E "aof_rewrite|aof_current"'

# Rewrite 완료 후 파일 크기 비교
ls -lh "$AOF_DIR/appendonly.aof"  # 대폭 축소 확인

# 파일 내용 확인 (RDB preamble + 차분 AOF)
redis-cli CONFIG GET aof-use-rdb-preamble
hexdump -C "$AOF_DIR/appendonly.aof" | head -5
# "REDIS0011" 프리앰블 확인 (mixed format 활성 시)
```

### 실험 4: AOF 데이터 손실 시뮬레이션

```bash
# everysec 정책으로 설정
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET appendfsync everysec

# 데이터 기록
redis-cli SET important_key "critical_data"
redis-cli HSET session:1 user_id 1001 expires 99999999

# 1초 이내에 추가 데이터 기록 + 강제 종료 (전원 차단 시뮬레이션)
redis-cli SET maybe_lost "this_might_be_lost"
pkill -9 redis-server  # SIGKILL (graceful shutdown 없이 즉시 종료)

# Redis 재시작
redis-server /etc/redis/redis.conf &
sleep 2

# 데이터 복구 확인
redis-cli GET important_key   # "critical_data" (복구됨)
redis-cli GET maybe_lost      # "this_might_be_lost" 또는 nil (1초 이내 손실 가능)
```

---

## 📊 성능 비교

```
appendfsync 정책별 처리량 벤치마크 (SSD 환경, 50 concurrent):

정책        | SET ops/sec  | 지연(avg) | 지연(p99)
───────────┼─────────────┼──────────┼──────────
always      | ~8,000       | ~6 ms    | ~15 ms
everysec    | ~80,000      | ~0.6 ms  | ~2 ms
no          | ~100,000     | ~0.5 ms  | ~1 ms
AOF 없음     | ~110,000     | ~0.45 ms | ~0.9 ms

HDD 환경에서 always 정책:
  HDD fsync 비용: 5~10 ms
  → ops/sec: ~100~200 (매우 낮음)
  → SSD 전환 없이 always 정책 = 성능 한계

AOF Rewrite 중 성능:
  Rewrite 동안: fork() + COW 복사 → CPU/메모리 일시 증가
  no-appendfsync-on-rewrite yes: Rewrite 중 fsync 없음
    → always/everysec 정책도 Rewrite 중에는 성능 저하 없음
    → 단, 이 기간 데이터 손실 위험 증가

파일 크기 비교 (동일 데이터):
  RDB: ~1 GB (바이너리, 압축)
  AOF (Rewrite 후): ~3~5 GB (텍스트, RESP 형식)
  AOF (Rewrite 전): 수십 GB (누적 모든 명령어)
  Mixed format AOF: ~1.5~2 GB (RDB preamble + 차분 AOF)
```

---

## ⚖️ 트레이드오프

```
AOF의 장단점:

장점:
  ① 세밀한 내구성 제어: always/everysec/no로 RPO 설정
  ② 읽기 가능한 형식: 장애 분석 시 명령어 확인 가능
  ③ 점진적 기록: 스냅샷과 달리 fork() 없음 (everysec/no)
  ④ 완전한 명령어 이력: 특정 시점 이전으로 롤백 가능

단점:
  ① 파일이 커짐: Rewrite 없이 방치 시 수십~수백 GB
  ② 느린 재시작: AOF 명령어 재실행 (RDB 대비 10~20배)
  ③ 파일 손상 가능: 쓰기 도중 크래시 → 마지막 명령어 불완전
     redis-check-aof로 복구 필요 (다음 문서)
  ④ 동일 데이터에 더 큰 파일 (RDB 대비)

RDB vs AOF 선택:
  데이터 손실 허용 없음 → AOF (everysec 이상)
  빠른 재시작 필요 → RDB 또는 Mixed format
  백업/복원 용도 → RDB (컴팩트한 파일)
  → 실제 운영: RDB + AOF 함께 사용 (혼합 전략 = 다음 문서)
```

---

## 📌 핵심 정리

```
AOF 핵심:

기록 형식:
  쓰기 명령어를 RESP 텍스트로 순서대로 Append
  텍스트 포맷 → 사람이 읽기 가능, RDB보다 파일 큼

fsync 정책:
  always:   명령어마다 fsync → 손실 0, 성능 낮음
  everysec: 1초마다 fsync → 최대 1초 손실, 성능 좋음 (기본값)
  no:       OS 판단 flush → 최대 30초 손실, 성능 최고

AOF Rewrite:
  fork() → 자식이 현재 메모리를 최소 명령어로 직렬화
  Rewrite 중 발생한 명령어 → aof_rewrite_buf에 버퍼링 → 완료 후 추가
  auto-aof-rewrite-percentage 100  # 기준 대비 2배 시 자동 Rewrite
  auto-aof-rewrite-min-size 64mb   # 최소 크기 조건

성능 영향:
  always: SSD ~8,000 ops/sec, HDD ~200 ops/sec
  everysec: ~80,000 ops/sec (대부분 서비스에 적합)
  no: ~100,000 ops/sec

진단:
  INFO persistence | grep aof     → AOF 상태
  aof_current_size                → 현재 파일 크기
  aof_last_rewrite_time_sec       → 마지막 Rewrite 시간
  redis-cli BGREWRITEAOF          → 수동 Rewrite 실행
```

---

## 🤔 생각해볼 문제

**Q1.** `appendfsync everysec`으로 설정된 Redis에서 디스크가 매우 느려 fsync가 2초 이상 걸리는 상황이다. 이벤트 루프는 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

Redis는 fsync가 2초 이상 지연되면 이벤트 루프를 **잠시 블로킹**하여 fsync 완료를 기다립니다.

`flushAppendOnlyFile()`의 내부 로직:
```c
if (server.aof_fsync == AOF_FSYNC_EVERYSEC) {
    if (sync_in_progress) {
        if (last_fsync_time elapsed > 2 seconds) {
            // 2초 이상 fsync 지연 → 이벤트 루프 블로킹하며 대기
            aof_background_fsync(server.aof_fd);
        }
    }
}
```

즉, 백그라운드 fsync가 2초 넘게 끝나지 않으면 이벤트 루프가 대기하면서 클라이언트 응답이 지연됩니다. 이것이 `SLOWLOG`에 갑자기 높은 지연이 나타나는 원인 중 하나입니다.

**진단:**
```bash
redis-cli INFO persistence | grep aof_last_write_status
# ok: 정상
# err: 쓰기 오류 발생

redis-cli LATENCY HISTORY event  # 지연 이벤트 히스토리
```

**해결:**
- 느린 HDD → SSD 전환
- 전용 디스크로 AOF 분리 (OS 디스크와 별도)
- `appendfsync no`로 변경 (손실 허용 시)

</details>

---

**Q2.** AOF 파일을 직접 열어 보니 마지막에 불완전한 명령어가 있다 (중간에 잘림). Redis 재시작 시 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

Redis 재시작 시 불완전한 AOF 명령어 처리는 `aof-load-truncated` 설정에 따라 다릅니다:

**`aof-load-truncated yes` (기본값):**
- 불완전한 마지막 명령어를 무시하고 나머지를 정상 로드
- 로그: "AOF file ends with an unfinished MULTIBULK command... truncated"
- 손실: 마지막 불완전 명령어 1개

**`aof-load-truncated no`:**
- 불완전한 명령어 발견 시 로드 중단 + 오류 종료
- 수동으로 `redis-check-aof --fix` 실행 후 재시작 필요

**수동 복구 절차:**
```bash
# 1. 손상 확인
redis-check-aof appendonly.aof
# "Checking Redis AOF file appendonly.aof... AOF analyzed: size=1234567, ok_up_to=1234500, diff=67"

# 2. 자동 수정 (잘린 부분 제거)
redis-check-aof --fix appendonly.aof
# "This will cause 67 bytes of data to be lost..."
# y 입력 → 수정됨

# 3. 재시작
redis-server
```

불완전한 AOF는 대부분 서버 크래시(SIGKILL, 전원 차단) 시 발생합니다. `appendfsync always`를 사용해도 write()와 fsync() 사이에 크래시가 나면 발생할 수 있습니다.

</details>

---

**Q3.** `auto-aof-rewrite-percentage 100`으로 설정했을 때, AOF 파일이 10 MB에서 시작했다가 Rewrite 후 5 MB가 됐다. 다음 Rewrite는 언제 발생하는가?

<details>
<summary>해설 보기</summary>

Rewrite 완료 후 **새 기준 크기(aof_base_size)는 Rewrite 완료 시점의 파일 크기**로 설정됩니다.

Rewrite 완료 후: `aof_base_size = 5 MB`

다음 Rewrite 트리거 조건:
```
현재 파일 크기 ≥ aof_base_size × (1 + auto-aof-rewrite-percentage/100)
현재 파일 크기 ≥ 5 MB × (1 + 100/100)
현재 파일 크기 ≥ 5 MB × 2
현재 파일 크기 ≥ 10 MB
```

그리고 동시에 `auto-aof-rewrite-min-size 64mb` 조건도 충족해야 합니다.
- 파일이 10 MB에 도달하더라도 `min-size 64 MB` 조건을 못 충족 → Rewrite 안 됨!

따라서 실제로:
- 파일이 10 MB 도달 → percentage 조건 충족
- 하지만 64 MB 미만 → min-size 조건 미충족 → Rewrite 안 함
- 파일이 64 MB 도달 → 두 조건 모두 충족 → Rewrite 실행

이것이 `auto-aof-rewrite-min-size`가 있는 이유입니다. 작은 파일을 불필요하게 자주 Rewrite하는 것을 방지합니다.

</details>

---

<div align="center">

**[⬅️ 이전: RDB 스냅샷](./01-rdb-snapshot.md)** | **[홈으로 🏠](../README.md)** | **[다음: RDB+AOF 혼합 모드 ➡️](./03-rdb-aof-mixed.md)**

</div>
