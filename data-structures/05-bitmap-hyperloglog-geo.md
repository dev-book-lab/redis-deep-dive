# Bitmap, HyperLogLog, Geo — 특수 자료구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Bitmap이 별도 자료구조가 아닌 String 위에서 동작하는 원리는?
- 1억 명의 DAU를 단 12.5 MB로 추적할 수 있는 이유는?
- HyperLogLog는 어떻게 12 KB로 수십억 개의 고유값 수를 추정하는가?
- HyperLogLog의 오차율 0.81%가 어디서 나오는가?
- Geo의 좌표가 Sorted Set의 스코어로 변환되는 방식(Geohash)은?
- `GEORADIUS`의 반경 탐색이 O(N+log M)으로 동작하는 원리는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Bitmap, HyperLogLog, Geo는 별도 자료구조처럼 보이지만 내부적으로는 String, Sorted Set 위에 구축된다. "어떤 문제를 얼마나 싸게 풀 수 있는가"를 결정하는 것이 이 특수 자료구조의 핵심이다. 잘못된 선택은 메모리 1,000배 차이를 만든다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: DAU 추적에 Set 사용 → 수 GB 메모리 낭비

  코드:
    SADD dau:20240101 user_id_1 user_id_2 ...
    SCARD dau:20240101  # DAU 계산

  결과:
    1억 DAU × 8 bytes ID = 800 MB~2 GB per 일
    30일치 보관 → 24~60 GB
    → 메모리 비용 폭발

실수 2: HyperLogLog로 멤버십 확인 시도

  코드:
    PFADD visited user_123
    # "이 유저가 방문했는지 확인하고 싶다"
    # SISMEMBER 대신 HLL로 확인 방법 없을까?

  문제:
    HLL은 카운트 추정만 가능 (PFCOUNT)
    특정 원소가 추가됐는지 확인 불가
    → 멤버십 확인 = Set 또는 Bitmap 필요

실수 3: Bitmap을 희소(sparse) ID에 사용 → 메모리 낭비

  유저 ID가 랜덤 UUID를 정수로 변환한 값 (예: 8,000,000,000)
  SETBIT dau 8000000000 1
  → offset 80억 → ceil(80억/8) = 1 GB 메모리 (실제 유저 1명!)
  → Bitmap은 ID가 밀집된 순차 정수일 때만 효율적
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
문제 유형별 선택:
  ┌──────────────────────────────────┬──────────────┬──────────┐
  │ 요구사항                           │ 자료구조        │ 메모리    │
  ├──────────────────────────────────┼──────────────┼──────────┤
  │ 정확한 DAU + 멤버십 확인 + 교집합      │ Set          │ ~2 GB    │
  │ 정확한 DAU + 순차 정수 ID            │ Bitmap       │ 12.5 MB  │
  │ 대략적 UV (오차 ±0.81% 허용)         │ HyperLogLog  │ 12 KB    │
  │ 위치 기반 근접 탐색                   │ Geo          │ ZSet 수준│
  └──────────────────────────────────┴──────────────┴──────────┘

Bitmap의 조건과 사용법:
  유저 ID가 순차 정수인 경우만 효율적
  SETBIT dau:20240101 {user_id} 1    # 방문 기록
  GETBIT dau:20240101 {user_id}      # 방문 여부 O(1)
  BITCOUNT dau:20240101              # DAU 수 O(N/8)
  BITOP AND both dau:01 dau:02       # 이틀 연속 방문자

HyperLogLog의 올바른 용도:
  정확도보다 메모리가 중요한 대용량 카운팅
  PFADD uv:homepage user_id         # UV 추가
  PFCOUNT uv:homepage               # UV 추정값
  PFMERGE monthly uv:01 uv:02 ...   # 월간 UV 합산

Geo 사용 조건 확인:
  TYPE places → "zset"  (Sorted Set 기반)
  정밀한 지리 쿼리(폴리곤 등) 필요 → PostGIS 검토
  빠른 근접 탐색 (반경 검색) → Geo
```

---

## 🔬 내부 동작 원리

### 1. Bitmap — String의 비트 조작

```
Bitmap은 별도 자료구조가 아니다:
  TYPE dau:bitmap → "string"  ← String 타입!
  
  Redis String = SDS (바이너리 안전)
  SDS의 buf[]를 비트 단위로 접근하는 것이 Bitmap

비트 레이아웃:
  SETBIT dau 0 1  → buf[0]의 최상위 비트(bit 7) = 1
  SETBIT dau 1 1  → buf[0]의 bit 6 = 1
  SETBIT dau 7 1  → buf[0]의 bit 0 = 1
  SETBIT dau 8 1  → buf[1]의 bit 7 = 1

  바이트: [bit7 bit6 bit5 bit4 bit3 bit2 bit1 bit0] [bit15 bit14...]
  오프셋:  0     1     2    3    4    5    6    7      8     9    ...

메모리 사용:
  offset 최댓값이 메모리 크기 결정
  SETBIT key 99999999 1 → offset 1억-1
  → ceil(1억 / 8) = 12,500,000 bytes = 12.5 MB

  SETBIT key 0 1 → 1 byte만 사용
  SETBIT key 100 1 → 13 bytes 사용 (ceil(101/8))

  희소(sparse) 비트맵의 문제:
  SETBIT key 0 1
  SETBIT key 999999999 1  ← 10억 번째 비트
  → 두 유저인데 125 MB 메모리!
  → 해결: 유저 ID를 해시해서 범위 압축
         또는 HyperLogLog로 대체

GETBIT, SETBIT:
  offset → byte_index = offset / 8, bit_index = 7 - (offset % 8)
  O(1) — 단순 비트 마스크 연산

BITCOUNT key [start end]:
  주어진 byte 범위 내 1인 비트 수 카운트
  전체: BITCOUNT dau → O(N) (N = 바이트 수)
  특정 날짜 범위 분석 가능

BITOP (AND, OR, XOR, NOT):
  여러 Bitmap에 비트 연산
  BITOP AND result dau:2024-01-01 dau:2024-01-02
  → 이틀 모두 접속한 유저 비트맵 → O(N)
  
  BITOP OR monthly dau:2024-01-01 dau:2024-01-02 ...
  → 월간 활성 유저 비트맵

BITPOS key bit [start end]:
  지정 bit 값의 첫 번째 위치 찾기
  O(N)

Bitmap 실전 패턴:
  # 유저 로그인 이벤트 기록
  SETBIT "login:2024-01-01" {user_id} 1
  
  # 이 유저가 오늘 접속했는가?
  GETBIT "login:2024-01-01" {user_id}  # O(1)
  
  # 오늘 DAU 수
  BITCOUNT "login:2024-01-01"  # O(N), N=bitmap 크기
  
  # 월간 활성 유저 (OR)
  BITOP OR "login:2024-01" login:2024-01-01 login:2024-01-02 ...
  BITCOUNT "login:2024-01"
```

### 2. HyperLogLog — 확률적 카디널리티 추정

```
HyperLogLog(HLL)의 목표:
  "대용량 데이터 스트림에서 중복 제거 후 고유값 수(cardinality) 추정"
  정확도보다 메모리 효율 우선

핵심 아이디어 (극단적 단순화):
  직관: 동전을 N번 던졌을 때 앞면이 k번 연속으로 나올 확률 ≈ 1/2^k
  → 연속 앞면이 k개라면, 총 던진 횟수 ≈ 2^k

  HyperLogLog 적용:
  각 원소를 해시 → 이진수 표현에서 앞에 오는 0의 수 = "연속 뒷면"
  
  예: hash("alice") = 0b00000101... → 앞에 0이 5개 → max_leading_zeros = 5
      hash("bob")   = 0b00110011... → 앞에 0이 2개 → max_leading_zeros = max(5, 2) = 5
      hash("carol") = 0b00001001... → 앞에 0이 4개 → max_leading_zeros = max(5, 4) = 5
  
  추정 카디널리티 ≈ 2^max_leading_zeros = 2^5 = 32
  실제 추가한 원소 3개 → 오차가 큼 (단순화 문제)

실제 HyperLogLog (LogLog 논문):
  단순 최대값 대신 여러 레지스터의 조화 평균 사용
  m개의 레지스터(Redis: 16384개)에 원소를 분산
  각 레지스터에 해당 그룹의 max_leading_zeros 저장
  최종 추정 = 상수 × m^2 × 조화평균(2^register[i])

Redis HyperLogLog 구현:
  16384개 (2^14) 레지스터
  각 레지스터: 6 bits (0~63 범위 저장)
  메모리: 16384 × 6 bits = 98304 bits = 12 KB!
  
  희소 표현 (sparse representation):
    원소가 적을 때: 실제로 변경된 레지스터만 저장 (훨씬 작음)
    임계값 도달 시: 밀집 표현(dense = 12 KB)으로 전환
  
  오차율 계산:
    표준 오차 ≈ 1.04 / sqrt(m) = 1.04 / sqrt(16384) = 1.04 / 128 ≈ 0.0081 = 0.81%

PFADD key element [element ...]:
  각 원소를 해시 → 레지스터 인덱스와 leading zeros 계산
  해당 레지스터의 값을 max(현재값, leading_zeros)로 업데이트
  O(1) per element

PFCOUNT key [key ...]:
  현재 레지스터 값들로 카디널리티 추정
  여러 키: 레지스터를 merge (union)한 후 추정
  O(M) — M = 레지스터 수 (항상 16384, 사실상 상수)

PFMERGE destkey sourcekey [sourcekey ...]:
  여러 HLL을 하나로 합침 (각 레지스터의 최댓값 사용)
  O(M × K) — M = 레지스터, K = 키 수

HLL 합집합의 정확도:
  PFMERGE는 합집합(union) 추정
  교집합 직접 지원 없음 (포함-배제 원리로 근사 가능)
  PFCOUNT(A) + PFCOUNT(B) - PFCOUNT(MERGE(A,B)) ≈ |A ∩ B|
  → 오차가 누적될 수 있음

HLL이 적합한 경우:
  ✓ 일별/월별 고유 방문자 수 (UV)
  ✓ 검색어 고유 수
  ✓ API 고유 호출자 수
  ✓ 1% 오차 허용 + 메모리 극단적 절약 필요

HLL이 부적합한 경우:
  ✗ 특정 유저가 포함됐는지 확인 (SISMEMBER 대체 불가)
  ✗ 0.1% 이하의 정확도 필요
  ✗ 원소 목록 조회 필요
```

### 3. Geo — Sorted Set 위에 구축된 위치 자료구조

```
Geo의 내부 구조:
  TYPE my:locations → "zset"  ← Sorted Set 타입!
  
  위도(latitude), 경도(longitude)를 52비트 Geohash 정수로 변환
  → Sorted Set의 스코어로 저장
  → 위치 기반 탐색 = Sorted Set 범위 탐색

Geohash 변환 과정:
  경도 범위: -180.0 ~ 180.0
  위도 범위: -90.0 ~ 90.0
  
  이진 분할 (52비트):
  경도 127.0에서:
    전체 범위 반으로 분할: -180~180 → 두 구간
    127.0 > 0 → 오른쪽 구간 선택 → 비트 1
    0~180 범위에서 다시 분할 → 127.0 > 90 → 오른쪽 → 비트 1
    ... 26번 반복 → 경도 26 bits
  
  위도 37.0에서 동일하게 26 bits
  
  경도 비트와 위도 비트를 인터리브(interleave):
  [위도bit0][경도bit0][위도bit1][경도bit1]...[위도bit25][경도bit25]
  → 52 bits 정수 = Geohash
  → 이것이 Sorted Set 스코어로 저장!

Geohash의 공간 지역성:
  지리적으로 가까운 위치 → 비슷한 Geohash 값
  → Sorted Set에서 스코어가 가까움 → 범위 탐색으로 근접 위치 검색
  
  단, 완벽하지 않음: 경계 지점에서는 인접한 위치여도 Geohash가 크게 달라질 수 있음
  → Redis Geo가 이를 보정해 정확한 거리 계산

GEOADD key longitude latitude member:
  내부: Geohash 계산 → ZADD key hash member
  O(log N) (Sorted Set 삽입)

GEOPOS key member:
  내부: ZSCORE key member → Geohash → 위도/경도로 역변환
  O(log N)

GEODIST key member1 member2 [unit]:
  두 멤버의 Geohash → 위도/경도 복원 → 하버사인 공식으로 거리 계산
  O(log N) (각 ZSCORE 조회)

GEORADIUS (deprecated) / GEOSEARCH key FROMMEMBER|FROMLONLAT BYRADIUS|BYBOX:
  반경 내 탐색 알고리즘:
  1. 중심점의 Geohash 계산
  2. 반경을 커버하는 Geohash 범위들 결정 (9개 정도의 셀)
  3. 각 Geohash 범위에 대해 ZRANGEBYSCORE로 후보 추출
  4. 후보들과 중심점 간 정확한 거리 계산 (하버사인)
  5. 반경 내 포함된 것만 반환
  O(N+log M) — N = 반경 내 원소 수, M = 전체 원소 수

GEOHASH key member [member ...]:
  내부 52비트 Geohash → Base32 인코딩 → 사람이 읽기 쉬운 문자열
  예: "wyd4k9z6rf0" (geohash.org 호환)
```

---

## 💻 실전 실험

### 실험 1: Bitmap DAU 추적

```bash
# 날짜별 유저 접속 Bitmap
DATE="20240101"

# 유저 접속 기록 (유저 ID = 오프셋)
redis-cli SETBIT "dau:$DATE" 1001 1   # 유저 1001 접속
redis-cli SETBIT "dau:$DATE" 1002 1   # 유저 1002 접속
redis-cli SETBIT "dau:$DATE" 5000 1   # 유저 5000 접속
redis-cli SETBIT "dau:$DATE" 9999 1   # 유저 9999 접속

# 특정 유저 접속 여부 확인 O(1)
redis-cli GETBIT "dau:$DATE" 1001  # 1 (접속함)
redis-cli GETBIT "dau:$DATE" 1003  # 0 (미접속)

# DAU 계산
redis-cli BITCOUNT "dau:$DATE"     # 4

# 메모리 확인 (오프셋 9999 = ceil(10000/8) = 1250 bytes)
redis-cli MEMORY USAGE "dau:$DATE"  # ~1300 bytes

# 1억 유저 시뮬레이션 (대용량 Bitmap)
redis-cli SETBIT "dau_large:$DATE" 99999999 1
redis-cli MEMORY USAGE "dau_large:$DATE"  # ~12.5 MB

# 이틀 모두 접속한 유저
redis-cli SETBIT "dau:20240102" 1001 1
redis-cli SETBIT "dau:20240102" 1003 1

redis-cli BITOP AND "active_both" "dau:20240101" "dau:20240102"
redis-cli BITCOUNT "active_both"  # 1 (유저 1001만 이틀 모두)
redis-cli BITPOS "active_both" 1  # 1001 (첫 번째 1인 오프셋)
```

### 실험 2: HyperLogLog UV 카운팅

```bash
# 페이지뷰에서 고유 방문자 카운팅
redis-cli PFADD "uv:homepage:20240101" "user:1001" "user:1002" "user:1003"
redis-cli PFADD "uv:homepage:20240101" "user:1001"  # 중복 추가 (반영 안 됨)
redis-cli PFCOUNT "uv:homepage:20240101"             # 3

# 대량 데이터로 오차율 확인
redis-cli DEL hll_test
for i in $(seq 1 100000); do
  redis-cli PFADD hll_test "user:$i" > /dev/null
done
ESTIMATED=$(redis-cli PFCOUNT hll_test)
echo "실제: 100000, 추정: $ESTIMATED, 오차: $(echo "scale=4; ($ESTIMATED - 100000) * 100 / 100000" | bc)%"
# 오차 대략 ±0.81% 이내

# 메모리 확인 (항상 약 12 KB)
redis-cli MEMORY USAGE hll_test  # ~12300 bytes (희소→밀집 전환 후)

# 여러 페이지의 합산 UV
redis-cli PFADD "uv:product:20240101" "user:1001" "user:2001" "user:3001"
redis-cli PFADD "uv:cart:20240101" "user:1001" "user:4001"

# 전체 사이트 UV (합집합)
redis-cli PFMERGE "uv:total:20240101" "uv:homepage:20240101" "uv:product:20240101" "uv:cart:20240101"
redis-cli PFCOUNT "uv:total:20240101"  # 약 6 (중복 제거)
```

### 실험 3: Geo 위치 기반 서비스

```bash
# 서울 주요 장소 좌표 추가
redis-cli GEOADD "places:seoul" \
  127.0276 37.4979 "gangnam" \
  126.9784 37.5665 "city_hall" \
  127.0016 37.5407 "coex" \
  126.9248 37.5519 "hongdae" \
  127.1055 37.5138 "jamsil"

# 위치 조회
redis-cli GEOPOS "places:seoul" gangnam
# 1) 1) "127.02759951353073"
#    2) "37.49789931636585"

# 두 장소 거리
redis-cli GEODIST "places:seoul" gangnam city_hall km  # km 단위 거리

# 강남역 2km 내 장소 검색 (Redis 6.2+ GEOSEARCH)
redis-cli GEOSEARCH "places:seoul" FROMMEMBER gangnam BYRADIUS 5 km ASC
# coex, gangnam (자기 자신 포함)

# 특정 좌표에서 반경 검색
redis-cli GEOSEARCH "places:seoul" FROMLONLAT 127.0276 37.4979 BYRADIUS 3 km ASC COUNT 5 WITHCOORD WITHDIST

# Geohash 확인 (내부 스코어)
redis-cli GEOHASH "places:seoul" gangnam   # Base32 Geohash
redis-cli ZSCORE "places:seoul" gangnam    # 실제 Sorted Set 스코어 (52비트 Geohash)

# Geo는 내부적으로 Sorted Set
redis-cli TYPE "places:seoul"  # zset
redis-cli OBJECT ENCODING "places:seoul"  # ziplist 또는 skiplist
```

### 실험 4: 각 자료구조의 메모리 비교

```bash
# 동일 데이터를 다른 방식으로 저장
USER_COUNT=100000

# Set 방식
redis-cli DEL set_users
for i in $(seq 1 $USER_COUNT); do redis-cli SADD set_users $i > /dev/null; done
MEM_SET=$(redis-cli MEMORY USAGE set_users)

# Bitmap 방식
redis-cli DEL bitmap_users
redis-cli SETBIT bitmap_users $((USER_COUNT - 1)) 1  # 최대 오프셋 설정
for i in $(seq 1 $USER_COUNT); do redis-cli SETBIT bitmap_users $i 1 > /dev/null; done
MEM_BITMAP=$(redis-cli MEMORY USAGE bitmap_users)

# HyperLogLog 방식
redis-cli DEL hll_users
for i in $(seq 1 $USER_COUNT); do redis-cli PFADD hll_users $i > /dev/null; done
MEM_HLL=$(redis-cli MEMORY USAGE hll_users)

echo "Set: $MEM_SET bytes"
echo "Bitmap: $MEM_BITMAP bytes"
echo "HyperLogLog: $MEM_HLL bytes"
echo "Set 대비 Bitmap: $(echo "scale=1; $MEM_SET / $MEM_BITMAP" | bc)x 절약"
echo "Set 대비 HLL: $(echo "scale=0; $MEM_SET / $MEM_HLL" | bc)x 절약"
```

---

## 📊 성능 비교

```
DAU 1억 명 기준 메모리 비교:

방법          | 메모리     | 정확도      | 지원 연산
─────────────┼──────────┼────────────┼─────────────────────────────
Set          | ~1.5 GB  | 100% 정확   | 멤버십, 카운트, 목록, 교집합
Bitmap       | 12.5 MB  | 100% 정확   | 멤버십, 카운트, 비트 연산
HyperLogLog  | 12 KB    | ±0.81% 오차 | 카운트만 (멤버십 불가)

HyperLogLog 오차율 이론:
  m = 16384 레지스터
  표준 오차 = 1.04 / sqrt(16384) = 0.8125%
  99.7% 신뢰 구간 (3σ): ±2.44%

Geo GEOSEARCH 복잡도:
  반경 내 원소 수 N, 전체 원소 수 M
  GEOSEARCH BYRADIUS: O(N + log M)
  → 전체 원소 수보다 반경 내 원소 수가 성능 결정 요소
  → 밀집 지역의 좁은 반경 탐색: N이 작으면 O(log M)에 수렴

Bitmap BITCOUNT 성능:
  최적화된 Hamming Weight 알고리즘 사용 (popcount 명령어)
  64bit씩 처리 → 8바이트당 ~1 CPU 명령어
  1억 비트(12.5 MB) BITCOUNT: ~수 ms
  → O(N) but 매우 빠른 상수
```

---

## ⚖️ 트레이드오프

```
Bitmap 사용 결정:
  적합: 유저 ID가 순차 정수이고 범위가 좁을 때 (1~1억)
  부적합: 희소(sparse) ID (ID가 1, 1000000처럼 띄어있으면 낭비)
          문자열 ID (UUID 등) → Bitmap 적용 불가
  
  희소 Bitmap 해결:
    UUID → 정수 해시 → 해시 충돌 허용하고 Bitmap 사용
    (HyperLogLog와 같은 확률적 접근)

HyperLogLog 오차 허용 여부:
  ±0.81% 오차가 의미 없는 경우: DAU 대략적 추세 분석
  ±0.81% 오차가 치명적인 경우: 과금 목적의 정확한 API 호출 수
  → 비즈니스 요구사항 먼저 확인

Geo vs 직접 좌표 저장 (PostgreSQL PostGIS):
  Redis Geo:
    메모리 내 처리 → 빠른 응답 (ms 단위)
    Geohash 52비트 → 위도 ±0.0001°, 경도 ±0.000001° 오차
    복잡한 지리 쿼리 불가 (폴리곤 검색, 경로 탐색 등)
  
  PostGIS:
    디스크 기반 → 대용량 데이터
    정밀한 기하학 연산
    공간 인덱스 (R-tree)로 복잡한 폴리곤 쿼리
  
  → 빠른 근접 검색: Redis Geo
  → 정밀 지리 분석: PostGIS
  → 두 가지를 계층적으로 조합도 가능 (Redis로 1차 필터링 → DB로 정밀 계산)
```

---

## 📌 핵심 정리

```
Bitmap:
  내부: String (SDS의 buf[]를 비트 단위 조작)
  TYPE → "string"
  메모리: ceil(max_offset / 8) bytes
  적합: 순차 정수 ID 멤버십 추적
  명령어: SETBIT, GETBIT, BITCOUNT, BITOP, BITPOS

HyperLogLog:
  목적: 카디널리티(고유값 수) 추정 (정확도 희생)
  메모리: 항상 최대 12 KB (16384개 레지스터 × 6 bits)
  오차율: ±0.81% (표준 편차)
  합집합: PFMERGE
  주의: 멤버십 확인 불가 (PFADD 후 특정 원소 포함 여부 조회 안 됨)

Geo:
  내부: Sorted Set (TYPE → "zset")
  좌표 → 52비트 Geohash → Sorted Set 스코어
  명령어: GEOADD, GEOPOS, GEODIST, GEOSEARCH, GEOHASH
  탐색: O(N + log M), N = 반경 내 원소, M = 전체 원소
  정밀도: 위도 ±0.0001°, 경도 ±0.000001°

선택 기준:
  "정확한 멤버십 + 집합 연산" → Set
  "정수 ID 멤버십, 메모리 최소" → Bitmap
  "대략적 카운트, 극단적 메모리 절약" → HyperLogLog
  "위치 기반 근접 검색" → Geo
```

---

## 🤔 생각해볼 문제

**Q1.** HyperLogLog에 동일한 원소를 `PFADD`로 100번 추가해도 `PFCOUNT`는 1을 반환한다. 내부적으로 어떻게 중복을 처리하는가?

<details>
<summary>해설 보기</summary>

HyperLogLog는 원소 목록을 저장하지 않습니다. 각 원소를 해시해서 **레지스터 값을 업데이트**하는 방식입니다.

"alice"를 100번 추가할 때:
1. hash("alice") → 항상 동일한 해시값 (결정적 해시)
2. 해시값 → 동일한 레지스터 인덱스, 동일한 leading zeros
3. `register[idx] = max(register[idx], leading_zeros)` → 이미 최댓값이면 변화 없음

결과: 레지스터 상태가 변하지 않음 → PFCOUNT 결과도 변하지 않음

이것이 HLL의 핵심 성질입니다. **같은 원소는 항상 같은 레지스터에 같은 영향을 미치므로** 중복을 "자동으로" 처리합니다. 별도의 중복 체크 없이 Set과 유사한 카디널리티 추정이 가능한 이유입니다.

단, 이 특성 때문에 "이 원소가 추가됐는지" 나중에 확인할 수 없습니다.

</details>

---

**Q2.** `BITOP AND result key1 key2`를 실행할 때 `key1`이 100 bytes, `key2`가 1 MB라면 result의 크기는 얼마인가? key1이 짧은 게 문제가 되는 경우는?

<details>
<summary>해설 보기</summary>

`result`의 크기는 **가장 긴 키의 크기**인 1 MB입니다. 짧은 키(key1)의 나머지 부분은 **0으로 패딩**된 것으로 처리됩니다.

즉, `AND` 연산에서 key1의 100 bytes 이후 비트는 모두 0 (key1이 짧으므로 패딩 0 AND key2 = 0). 따라서 result의 101 bytes 이후는 모두 0이 됩니다.

**이것이 문제가 되는 경우:**
- key1에 100번째 오프셋까지만 기록하고 key2에 1백만번째 오프셋까지 기록
- BITOP AND → 100 bytes 이후는 모두 AND 0 = 0
- "이틀 모두 접속한 유저"를 구하려 했는데, key2에서 100번 이후 오프셋 유저는 모두 0이 됨

**해결**: BITOP 대상 키들의 크기를 균일하게 유지하거나, `BITOP OR`로 큰 쪽에 맞춰 확장한 후 AND 수행. 또는 같은 기간에 생성된 Bitmap은 동일한 최대 오프셋을 유지하도록 설계합니다.

</details>

---

**Q3.** Geo에서 두 장소의 거리를 구할 때 내부적으로 `GEODIST`는 Sorted Set의 스코어(Geohash)를 직접 비교하는가, 아니면 좌표를 복원해 거리를 계산하는가?

<details>
<summary>해설 보기</summary>

**좌표를 복원해 하버사인(Haversine) 공식으로 거리를 계산합니다.**

과정:
1. `ZSCORE` 로 각 멤버의 Geohash(52비트 정수) 조회 → O(1)
2. 52비트 Geohash → 위도/경도 역변환
3. 하버사인 공식으로 구면 거리 계산

왜 Geohash를 직접 비교하지 않는가:
- Geohash 차이가 거리에 비례하지 않습니다. 경계 지점에서 인접한 두 위치의 Geohash가 크게 다를 수 있습니다.
- Geohash는 "지역 분할을 위한 인코딩"이지 "거리 척도"가 아닙니다.

`GEOSEARCH`의 반경 탐색이 정확한 이유:
1. Geohash 범위로 후보를 빠르게 필터링 (Sorted Set 범위 탐색)
2. 후보들을 하버사인으로 정밀 계산 → 반경 초과 후보 제거

Geohash의 정밀도 (52비트):
- 위도: ±0.000003° ≈ ±0.3m
- 경도: ±0.000001° ≈ ±0.1m
충분히 정밀한 위치 표현이 가능합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Set과 Sorted Set](./04-set-sortedset-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Stream Consumer Group ➡️](./06-stream-consumer-group.md)**

</div>
