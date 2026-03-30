# RDB + AOF 혼합 모드 — Redis 4.0 Mixed Format

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- AOF 파일 안에 RDB 스냅샷이 들어가는 혼합 포맷의 구조는?
- 재시작 시 RDB 부분과 AOF 부분을 어떻게 구분해서 복원하는가?
- 혼합 포맷이 순수 AOF 대비 재시작 속도를 왜 크게 단축하는가?
- `aof-use-rdb-preamble` 설정을 켰을 때 BGREWRITEAOF의 동작이 어떻게 달라지는가?
- 혼합 포맷의 단점과 순수 AOF를 선호해야 하는 경우는?
- Redis 7.0의 Multi Part AOF(MP-AOF)는 혼합 포맷을 어떻게 개선했는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

순수 AOF는 재시작 복원이 느리고, RDB만 쓰면 데이터 손실이 크다. 혼합 포맷(Mixed Format)은 AOF 파일 앞부분에 RDB 스냅샷을 삽입해 두 문제를 동시에 해결한다. 이 구조를 모르면 "AOF를 쓰면 재시작이 오래 걸린다"는 오해를 그대로 믿고 최적 설정을 포기하게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: "AOF는 재시작이 느리다"는 이유로 포기

  팀 결정: "AOF 쓰면 재시작 때 수십 분 걸린다고 하던데, RDB만 쓰자"
  설정: appendonly no, save 300 100
  결과: 장애 시 최대 5분 데이터 손실
        재시작은 빠르지만 세션/주문 수분치 유실

  실제: 혼합 포맷 사용 시 재시작 1~3분 (RDB 속도와 유사)
        AOF의 내구성(~1초 손실)까지 함께 확보 가능했음

실수 2: 혼합 포맷 활성화 후 AOF 파일을 일반 텍스트 뷰어로 열기

  cat appendonly.aof | head -20
  → 파일 앞부분이 바이너리(RDB 부분) → 깨진 문자 출력
  → "AOF 파일이 손상됐다!" → 불필요한 긴급 대응

  실제: 정상 (앞부분 RDB 바이너리 + 뒷부분 RESP 텍스트)
        redis-check-aof appendonly.aof 로 검증하면 정상 확인

실수 3: 혼합 포맷에서 AOF Rewrite 빈도 과다 설정

  auto-aof-rewrite-percentage 10  (10%마다 rewrite)
  결과: 자주 Rewrite → 자주 fork() → 지연 반복
        혼합 포맷이라도 Rewrite는 비용이 있음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
혼합 포맷 활성화 (Redis 4.0+, 기본값):
  aof-use-rdb-preamble yes    # redis.conf 확인 (기본 활성)
  appendonly yes
  appendfsync everysec

재시작 복원 시간 예측:
  RDB 부분 로드: used_memory 크기에 비례 (~4 GB → ~30~60초)
  AOF 부분 재실행: Rewrite 이후 추가된 명령어 수에 비례
  → Rewrite 직후 재시작: RDB 속도와 거의 동일
  → Rewrite 오래됐으면: AOF 누적분 재실행 추가

Rewrite 주기 적절히 유지:
  auto-aof-rewrite-percentage 100  # 기본 (100% 증가 시 Rewrite)
  auto-aof-rewrite-min-size 64mb   # 기본 (64 MB 이상일 때만)
  → 너무 자주 Rewrite = fork() 비용 반복
  → 너무 드물게 Rewrite = AOF 부분 누적 → 재시작 느려짐

혼합 포맷 파일 검증:
  redis-check-aof appendonly.aof
  # [OK] AOF analyzed: filename=appendonly.aof, size=...,
  # ok_up_to_line=..., ok_up_to_byte=...
  # 앞부분 바이너리(RDB) + 뒷부분 텍스트(AOF) = 정상
```

---

## 🔬 내부 동작 원리

### 1. 혼합 포맷 파일 구조

```
aof-use-rdb-preamble yes (기본값, Redis 4.0+):
  BGREWRITEAOF 실행 시 생성되는 AOF 파일 구조:

  appendonly.aof 파일:
  ┌─────────────────────────────────────────────────────┐
  │  [RDB 형식 섹션]                                     │
  │  "REDIS0011"                                        │
  │  AUX FIELDS (버전, 생성 시각 등)                     │
  │  SELECT DB 0                                        │
  │  [현재 메모리의 모든 키-값 쌍 — RDB 바이너리 포맷]    │
  │  EOF (0xFF)                                         │
  │  CRC64 CHECKSUM                                     │
  ├─────────────────────────────────────────────────────┤
  │  [AOF 형식 섹션]                                     │
  │  *3\r\n$3\r\nSET\r\n...  (BGREWRITEAOF 이후 명령어) │
  │  *2\r\n$4\r\nINCR\r\n... (계속 추가되는 명령어들)    │
  │  ...                                                │
  └─────────────────────────────────────────────────────┘

  경계 구분:
    RDB 섹션: 파일 처음 "REDIS" 매직으로 시작
    AOF 섹션: RDB EOF 이후 RESP 텍스트

  파일 확장:
    서버 실행 중 새 쓰기 명령어 → AOF 섹션 끝에 계속 추가
    다음 BGREWRITEAOF 전까지 AOF 섹션은 계속 커짐

aof-use-rdb-preamble no (순수 AOF):
  BGREWRITEAOF 실행 시:
  ┌─────────────────────────────────────────────────────┐
  │  [AOF 형식만]                                        │
  │  *3\r\n$3\r\nSET\r\n$6\r\nuser:1\r\n...            │
  │  (현재 메모리를 재현하는 최소 명령어 집합)              │
  │  *2\r\n$4\r\nINCR\r\n...  (이후 추가 명령어)         │
  └─────────────────────────────────────────────────────┘
  → 모두 RESP 텍스트 → 재시작 시 전체 명령어 재실행 필요
```

### 2. BGREWRITEAOF 동작 — 혼합 vs 순수 AOF

```
혼합 포맷 (aof-use-rdb-preamble yes):

  fork() → 자식 프로세스 생성
  
  자식 프로세스:
    1. 현재 메모리를 RDB 바이너리 포맷으로 직렬화
       → aof.tmp 파일에 RDB 섹션 기록
    2. RDB 섹션 완료 → EOF + CRC64
    3. aof.tmp 완료 신호
  
  부모 프로세스:
    - Rewrite 중 발생한 명령어를 aof_rewrite_buf에 버퍼링
    - 자식 완료 신호 수신
    - aof_rewrite_buf 내용을 aof.tmp 끝에 AOF 형식으로 추가
    - aof.tmp → appendonly.aof (rename)

  결과:
    [RDB 섹션: 현재 메모리 전체] + [AOF 섹션: Rewrite 중 발생 명령어]
  
순수 AOF (aof-use-rdb-preamble no):

  동일 과정이지만 자식이 RDB 대신 RESP 텍스트로 직렬화
  → SET key value, EXPIRE key seconds 등 명령어 형태로 기록
  → 파일 크기: RDB보다 3~5배 크고, 로드도 느림

성능 비교 (4 GB 메모리 기준):

               혼합 포맷         순수 AOF
  Rewrite 시간: ~60초            ~90초 (텍스트 직렬화 느림)
  Rewrite 파일: ~1.5 GB          ~3~5 GB (텍스트 크기)
  재시작 로드:  ~60~120초         ~15~30분
```

### 3. 재시작 복원 과정 — 혼합 포맷

```
Redis 재시작 시 AOF 로드:

  redis-server 시작
  → appendonly.aof 존재 확인
  → 파일 첫 5 bytes 읽기: "REDIS" 확인
  → 혼합 포맷 감지!

  1단계: RDB 섹션 로드
    RDB 파서를 사용해 바이너리 섹션 고속 로드
    dict에 모든 키-값 직접 삽입 (명령어 재실행 없음)
    → 수 GB를 수십 초 만에 복원
  
  2단계: AOF 섹션 재실행
    RDB EOF 이후 RESP 텍스트 명령어들 재실행
    이 부분은 BGREWRITEAOF 이후 발생한 명령어들
    → Rewrite 주기가 짧으면 AOF 섹션이 작음
    → 수천~수만 건 명령어 재실행 (빠름)
  
  결과: 완전한 메모리 상태 복원

  순수 AOF 로드와 비교:
    순수 AOF: 모든 명령어를 하나씩 재실행
    혼합 포맷: RDB 고속 로드 + 소량의 AOF 재실행
    → 혼합 포맷이 10~20배 빠를 수 있음

  로드 로그 예시:
    [파일 시작]: "Loading RDB preamble from AOF file..."
    [RDB 완료]:  "Reading the remaining AOF tail..."
    [완료]:      "DB loaded from append only file: 45.234 seconds"
```

### 4. Redis 7.0 Multi Part AOF (MP-AOF)

```
혼합 포맷의 남은 문제:
  단일 파일에 RDB + AOF가 함께 → 파일 관리 복잡
  Rewrite 중 파일이 일시적으로 두 배 크기 (원본 + 임시 파일)
  Rewrite 실패 시 임시 파일 정리 필요

MP-AOF (Redis 7.0):

  파일 분리:
  appenddirname "appendonlydir"  # AOF 파일을 담는 디렉토리
  
  디렉토리 구조:
  /var/lib/redis/appendonlydir/
    appendonly.aof.1.base.rdb   # Base 파일 (RDB 포맷 스냅샷)
    appendonly.aof.1.incr.aof   # Incremental AOF (스냅샷 이후 명령어)
    appendonly.aof.2.incr.aof   # 다음 Rewrite 이후 증분
    appendonly.aof.manifest     # 파일 목록 및 순서 메타데이터

  manifest 파일:
    file appendonly.aof.1.base.rdb seq 1 type b  # b = base
    file appendonly.aof.1.incr.aof seq 1 type i  # i = incremental
    file appendonly.aof.2.incr.aof seq 2 type i

  Rewrite 과정 (MP-AOF):
    1. 새 base 파일 생성 (현재 메모리 → RDB)
    2. 새 incremental 파일 시작 (이후 명령어)
    3. 이전 파일들을 manifest에서 제거
    → 원자적 전환 (rename 대신 manifest 업데이트)
    → 실패해도 이전 파일들이 남아있어 복구 가능

  장점:
    파일 레벨 원자성: manifest 업데이트로 안전한 전환
    부분 기록 복구: 특정 incremental 파일만 손상 시 해당 부분만 처리
    Rewrite 중 추가 파일 크기 증가 최소화

설정 확인:
  redis-cli CONFIG GET appenddirname    # MP-AOF 디렉토리
  ls /var/lib/redis/appendonlydir/      # 파일 목록 확인
  cat /var/lib/redis/appendonlydir/appendonly.aof.manifest
```

### 5. 혼합 포맷 활성화 조건과 확인

```
활성화 조건:
  appendonly yes         # AOF 활성
  aof-use-rdb-preamble yes  # 혼합 포맷 활성 (기본값)

  → 두 조건 모두 충족 시 BGREWRITEAOF 실행마다 혼합 포맷 생성

비활성화 (순수 AOF):
  aof-use-rdb-preamble no  # redis.conf에서 설정 또는
  redis-cli CONFIG SET aof-use-rdb-preamble no

현재 파일이 혼합 포맷인지 확인:
  hexdump -C /var/lib/redis/appendonly.aof | head -2
  # 00000000  52 45 44 49 53 30 30 31  31 ...  "REDIS0011"
  # → 혼합 포맷 (RDB 매직으로 시작)
  
  # 또는:
  head -c 5 /var/lib/redis/appendonly.aof
  # "REDIS" → 혼합 포맷
  # "*3" 등의 RESP 문자 → 순수 AOF

INFO persistence에서 확인:
  redis-cli INFO persistence | grep aof_rewrite_base_size
  aof_using_rdb_format_for_base:1   # 혼합 포맷 사용 중 (Redis 7.0+)
```

---

## 💻 실전 실험

### 실험 1: 혼합 포맷 파일 구조 확인

```bash
# AOF + 혼합 포맷 활성화
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET aof-use-rdb-preamble yes

# 데이터 생성
for i in $(seq 1 1000); do
  redis-cli SET "mixed:$i" "value-$i" > /dev/null
done

# BGREWRITEAOF 실행
redis-cli BGREWRITEAOF
sleep 10  # 완료 대기

# 파일 시작 부분 확인
AOF_DIR=$(redis-cli CONFIG GET dir | tail -1)
AOF_FILE=$(redis-cli CONFIG GET appendfilename | tail -1)
hexdump -C "$AOF_DIR/$AOF_FILE" | head -5
# "REDIS0011" 확인 → 혼합 포맷

# 파일 구조 분석
file "$AOF_DIR/$AOF_FILE"
# "Redis RDB file..." 또는 일반 파일

# RDB 섹션과 AOF 섹션 경계 찾기
python3 - <<'EOF'
with open('/var/lib/redis/appendonly.aof', 'rb') as f:
    data = f.read()

# RDB EOF 마커 (0xFF) 위치 찾기
rdb_end = data.find(b'\xff')
if rdb_end != -1:
    # CRC64 (8 bytes) 이후가 AOF 섹션
    aof_start = rdb_end + 1 + 8
    print(f"RDB 섹션: 0 ~ {aof_start} bytes ({aof_start/1024:.1f} KB)")
    print(f"AOF 섹션 시작: {aof_start}")
    print(f"AOF 첫 10 bytes: {data[aof_start:aof_start+10]}")
EOF
```

### 실험 2: 재시작 속도 비교

```bash
# 대용량 데이터 생성
for i in $(seq 1 100000); do
  redis-cli SET "speed_test:$i" "$(python3 -c "print('x'*100)")" > /dev/null
done
redis-cli DBSIZE  # 100,000

# 혼합 포맷으로 BGREWRITEAOF
redis-cli CONFIG SET aof-use-rdb-preamble yes
redis-cli BGREWRITEAOF
sleep 30  # 완료 대기
ls -lh /var/lib/redis/appendonly.aof

# 혼합 포맷으로 재시작 시간 측정
redis-cli SHUTDOWN NOSAVE  # Redis 종료 (RDB 저장 없이)
time redis-server /etc/redis/redis.conf &
# 로그에서 "DB loaded from append only file: X seconds" 확인

# 순수 AOF로 비교
redis-cli CONFIG SET aof-use-rdb-preamble no
redis-cli BGREWRITEAOF
sleep 30
redis-cli SHUTDOWN NOSAVE
time redis-server /etc/redis/redis.conf &
# 더 오래 걸리는 것 확인
```

### 실험 3: AOF 로드 로그 분석

```bash
# Redis 로그 파일 확인
redis-cli CONFIG GET logfile
# /var/log/redis/redis-server.log (설정에 따라 다름)

tail -50 /var/log/redis/redis-server.log | grep -E "Loading|loaded|RDB|AOF|seconds"

# 예상 출력:
# [1234] * Loading RDB preamble from AOF file...
# [1234] * Loading RDB produced by Redis 7.0.x, ...
# [1234] * RDB age 45 seconds
# [1234] * RDB memory usage when created: 512.00 Mb
# [1234] * Reading the remaining AOF tail...
# [1234] * DB loaded from append only file: 12.345 seconds
```

### 실험 4: Redis 7.0 MP-AOF 구조 확인

```bash
# Redis 7.0+ 에서 MP-AOF 확인
redis-cli INFO server | grep redis_version  # 7.0 이상인지 확인
redis-cli CONFIG GET appenddirname

# MP-AOF 디렉토리 내용
AOF_DIR=$(redis-cli CONFIG GET appenddirname | tail -1)
ls -la "$AOF_DIR/"
# appendonly.aof.1.base.rdb
# appendonly.aof.1.incr.aof
# appendonly.aof.manifest

# manifest 파일 내용 확인
cat "$AOF_DIR/appendonly.aof.manifest"

# base 파일 (RDB 포맷)
file "$AOF_DIR/appendonly.aof.1.base.rdb"
# "Redis RDB file version 11"

# incremental AOF 파일
head -20 "$AOF_DIR/appendonly.aof.1.incr.aof"
# RESP 형식 명령어들
```

---

## 📊 성능 비교

```
재시작 복구 시간 비교 (메모리 4 GB, SSD 기준):

방식                   | 파일 크기    | 재시작 시간   | 데이터 손실
──────────────────────┼─────────────┼─────────────┼────────────
RDB만                  | ~1~2 GB     | ~30~60초     | 마지막 스냅샷 이후
순수 AOF               | ~3~5 GB     | ~15~30분     | 최대 1초 (everysec)
혼합 포맷 AOF          | ~1.5~2.5 GB | ~1~3분       | 최대 1초 (everysec)
RDB + AOF (두 파일)    | ~1 + ~0.5 GB| ~60~120초    | 최대 1초 (everysec)

파일 크기 비교 (4 GB 메모리, Rewrite 직후):
  RDB:         1.2 GB (바이너리, LZF 압축)
  혼합 포맷:   1.3 GB (RDB + 소량 AOF)
  순수 AOF:    3.8 GB (텍스트 RESP)

BGREWRITEAOF 소요 시간:
  혼합 포맷 Rewrite: ~45초 (RDB 직렬화)
  순수 AOF Rewrite:  ~90초 (RESP 텍스트 직렬화)
  → 혼합 포맷이 Rewrite도 빠름 (바이너리 쓰기)
```

---

## ⚖️ 트레이드오프

```
혼합 포맷의 장점:
  ① 빠른 재시작: RDB 바이너리 로드 + 소량 AOF 재실행
  ② 작은 파일 크기: 순수 AOF 대비 30~60% 작음
  ③ 빠른 Rewrite: 바이너리 직렬화가 텍스트보다 빠름
  ④ AOF 내구성 유지: 여전히 everysec fsync로 세밀한 보호

혼합 포맷의 단점:
  ① 사람이 읽기 어려움: RDB 섹션은 바이너리 → 장애 분석 어려움
  ② 구버전 호환성: Redis 4.0 이전에서는 혼합 포맷 로드 불가
  ③ 부분 분석 불가: 순수 AOF는 특정 시점 이전 명령어 편집 가능
                   혼합 포맷의 RDB 섹션은 편집 어려움

순수 AOF를 선호해야 하는 경우:
  ① 장애 분석 중요: AOF 파일 직접 읽어 문제 명령어 찾기
  ② Redis 4.0 미만 환경과 혼재
  ③ PITR(Point-In-Time Recovery): 특정 시각까지의 명령어만 재실행
     예: "2시간 전 실수로 FLUSHALL한 시각 이전으로 복원"
     → 순수 AOF라면 FLUSHALL 이전 줄에서 잘라서 재실행 가능
     → 혼합 포맷이면 RDB 시점 이후 AOF만 편집 가능 (RDB 시점 이전 PITR 불가)

실무 권장:
  대부분의 서비스 → 혼합 포맷 (기본값 유지)
  장애 분석/PITR 중요 → 순수 AOF
  재시작 속도 최우선 → RDB만 (손실 허용 범위 확인 필수)
```

---

## 📌 핵심 정리

```
RDB+AOF 혼합 포맷 핵심:

파일 구조:
  [RDB 바이너리 섹션] + [AOF RESP 텍스트 섹션]
  RDB 섹션: BGREWRITEAOF 시점의 메모리 스냅샷 (바이너리)
  AOF 섹션: Rewrite 이후 발생한 쓰기 명령어 (RESP 텍스트)

재시작 복원:
  파일 시작이 "REDIS" → 혼합 포맷 감지
  1단계: RDB 섹션 바이너리 고속 로드
  2단계: AOF 섹션 명령어 재실행 (소량)
  → 순수 AOF 대비 10~20배 빠른 복원

BGREWRITEAOF:
  aof-use-rdb-preamble yes → RDB 바이너리로 base 생성
  aof-use-rdb-preamble no  → RESP 텍스트로 base 생성

Redis 7.0 MP-AOF:
  단일 파일 → 디렉토리(base.rdb + incr.aof + manifest)
  원자적 전환, 부분 실패 복구 가능

설정:
  aof-use-rdb-preamble yes  # 기본값 (활성화 권장)
  appenddirname "..."        # Redis 7.0+ MP-AOF 디렉토리

진단:
  hexdump | head → "REDIS0011"이면 혼합 포맷
  INFO persistence → aof_using_rdb_format_for_base (7.0+)
```

---

## 🤔 생각해볼 문제

**Q1.** 혼합 포맷 AOF 파일에서 "2시간 전 FLUSHALL 이전 상태로 복원"하려 한다. 가능한가? 어떤 조건에서 가능하고, 어떤 조건에서 불가능한가?

<details>
<summary>해설 보기</summary>

혼합 포맷에서 PITR(Point-In-Time Recovery)은 **조건부로 가능**합니다.

**가능한 경우:**
BGREWRITEAOF가 FLUSHALL 이후에 실행되지 않았다면:
- AOF 파일 = [이전 RDB 스냅샷] + [FLUSHALL 포함 AOF 섹션]
- AOF 섹션에서 FLUSHALL 줄을 찾아 해당 줄 이전까지만 재실행
- RDB 시점 이후, FLUSHALL 직전까지의 상태로 복원 가능

**불가능한 경우:**
FLUSHALL 이전에 BGREWRITEAOF가 실행됐다면:
- AOF 파일 = [FLUSHALL 이후 데이터로 만든 RDB] + [이후 AOF]
- RDB 섹션 자체가 이미 FLUSHALL 이후 상태 (=빈 데이터)
- FLUSHALL 이전으로 돌아갈 방법 없음

**실무 교훈:**
- 정기적으로 RDB 파일을 오프사이트 백업 (S3 등)
- FLUSHALL은 복구 어려운 치명적 명령어 → 운영 환경 접근 제한
- 복구 가능한 범위를 미리 문서화하고 정기적으로 복구 훈련 실시

</details>

---

**Q2.** `aof-use-rdb-preamble yes`인데 RDB가 비활성화(`save ""`)된 경우, BGREWRITEAOF는 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

**혼합 포맷으로 동작합니다.** `aof-use-rdb-preamble`과 `save` 설정은 독립적입니다.

- `save ""`: 자동 BGSAVE(정기 스냅샷)를 비활성화
- `aof-use-rdb-preamble yes`: BGREWRITEAOF 실행 시 AOF 내부에 RDB 포맷 사용

두 설정은 서로 다른 기능을 제어합니다:
- `save`: 독립 RDB 파일(dump.rdb)로의 주기적 저장
- `aof-use-rdb-preamble`: AOF Rewrite 내부 포맷 선택

따라서 `save ""` + `aof-use-rdb-preamble yes` + `appendonly yes` 조합은:
- dump.rdb 파일 생성 안 함 (save 없음)
- appendonly.aof에는 혼합 포맷 사용 (BGREWRITEAOF 시)
- AOF만으로 영속성 관리하되 빠른 재시작을 위해 혼합 포맷 사용

이 조합이 많은 실무 환경에서 권장되는 설정입니다.

</details>

---

**Q3.** 혼합 포맷 AOF 파일이 손상됐다면 `redis-check-aof`로 복구 시 어떤 한계가 있는가?

<details>
<summary>해설 보기</summary>

`redis-check-aof`는 두 섹션을 다르게 처리합니다:

**RDB 섹션 손상:**
```bash
redis-check-aof appendonly.aof
# "RDB preamble is not valid, cannot proceed"
```
RDB 섹션이 손상되면 `redis-check-aof --fix`로 복구가 **매우 어렵거나 불가능**합니다. RDB는 바이너리 포맷이라 중간 데이터가 손상되면 이후 구조를 파악하기 어렵습니다.

→ 이 경우 이전 정상 RDB 스냅샷으로 복구 후 AOF 섹션 부분만 수동 적용

**AOF 섹션 손상 (마지막 명령어 불완전):**
```bash
redis-check-aof --fix appendonly.aof
# RDB 섹션은 건드리지 않고 AOF 섹션의 불완전 명령어 제거
```
→ 성공 가능. `aof-load-truncated yes` 설정이라면 자동으로도 처리됨

**실무 대응:**
- 혼합 포맷 사용 시 dump.rdb 파일도 함께 백업 유지
- 손상 시나리오별 복구 절차를 미리 문서화
- 정기적으로 `redis-check-rdb` + `redis-check-aof`로 파일 무결성 검사

</details>

---

<div align="center">

**[⬅️ 이전: AOF fsync 정책](./02-aof-fsync.md)** | **[홈으로 🏠](../README.md)** | **[다음: 장애 복구 절차 ➡️](./04-disaster-recovery.md)**

</div>
