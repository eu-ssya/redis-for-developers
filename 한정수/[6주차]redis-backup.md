# 7장. 레디스 데이터 백업 방법

## 1. 레디스에서 데이터를 영구 저장하기

### 1.1 왜 백업이 필요한가
- 레디스의 모든 데이터는 **메모리**에서 관리된다.
- 인스턴스 재시작·서버 장애 시 메모리에 상주하던 데이터는 모두 손실될 가능성이 있다.
- 캐시 용도가 아닌 영구 저장소 용도로 사용한다면 디스크에 주기적으로 백업해야 안전하다.

### 1.2 복제 vs 백업: 목적이 다르다
| | 복제 (Replication) | 백업 (Backup) |
|---|---|---|
| 목적 | **가용성** 확보 | **장애 복구** |
| 동작 | 마스터 -> 복제본으로 실시간 전달 | 디스크에 주기적으로 저장 |
| 약점 | 개발자 실수로 `DEL` 실행하면 복제본에도 즉시 반영됨 | - |

**핵심**: 복제 구조만으로는 데이터를 안전하게 유지할 수 없다.

---

## 2. RDB와 AOF 개요

### 2.1 두 방식의 정의
- **AOF (Append Only File)**: 레디스가 처리한 모든 쓰기 작업을 차례대로 기록. 복원 시 파일을 재실행해 데이터셋 재구성.
- **RDB (Redis DataBase)**: 일정 시점의 메모리 데이터 전체를 저장 (snapshot 방식).

### 2.2 동작 예시로 보는 차이

다음 커맨드를 실행했다고 가정하자:
```
> SET key1 a       # OK
> SET key1 apple   # OK
> SET key2 b       # OK
> DEL key2         # (integer) 1
```

| AOF 파일 (모든 이력 기록) | RDB 파일 (최종 상태만) |
|---|---|
| `set key1 a`<br>`set key1 apple`<br>`set key2 b`<br>`del key2` | `key1 -> apple` |

- **AOF**: 전체 커맨드 이력이 RESP 프로토콜 텍스트로 저장 → 따라가면 원본 데이터에 도달 가능
- **RDB**: 메모리 그대로 내려쓴 바이너리 -> 최종적으로 메모리에 남아있던 `key1=apple`만 저장

### 2.3 장단점 비교
| | RDB | AOF |
|---|---|---|
| **장점** | 시점별 백업 다수 보관 가능, 복원 빠름, 파일 작음 | 특정 시점으로 복구 가능, 데이터 손실 적음 |
| **단점** | 마지막 저장 시점 ~ 장애 직전 데이터 손실 | 파일 크기 큼, 주기적 압축(재구성) 필요 |

### 2.4 동시 사용 권장
- 한 인스턴스에서 **RDB + AOF 동시 사용 가능**.
- 데이터 안정성을 원한다면 두 방식을 함께 쓰는 것을 공식 권장.
- 복원 시 두 파일이 모두 있으면 → **AOF가 더 내구성 있다고 판단해 AOF를 로드**한다.

---

## 3. RDB 방식의 데이터 백업

> 가장 단순한 백업 방법. 메모리 자체를 스냅숏처럼 저장한다.

### 3.1 운영 전략
- RDB 파일이 저장될 때마다 원격 저장소로 옮겨 **2차 백업**을 수행하면 데이터센터 장애에도 대응 가능
- **단점**: 저장 시점부터 장애 발생 직전까지의 데이터는 손실될 수 있음 -> 손실을 최소화해야 하는 서비스에 RDB를 단독 사용하는 것은 부적절함

### 3.2 RDB 파일 생성 방법 3가지

#### 방법 1: 특정 조건에 자동 생성 (`save` 옵션)

```conf
save <기간(초)> <기간 내 변경된 키의 개수>
dbfilename <RDB 파일 이름>
dir <RDB 파일이 저장될 경로>
```

기본 설정 예시:
```conf
save 900 1       # 900초(15분) 동안 1개 이상 변경 시
save 300 10      # 300초(5분) 동안 10개 이상 변경 시
save 60 10000    # 60초(1분) 동안 10,000개 이상 변경 시
```

- `dbfilename` 기본값: `dump.rdb`
- 비활성화: `save ""` (빈 문자열)

런타임에 변경하려면:
```
> CONFIG GET save
1) "save"
2) "900 1 300 10 60 10000"

> CONFIG SET save ""
OK

> CONFIG REWRITE  # redis.conf 파일에도 반영
OK
```

> **NOTE - CONFIG REWRITE**: 실행 중 인스턴스에서 `CONFIG SET`으로 파라미터를 수정한 뒤, `CONFIG REWRITE`로 설정 파일을 재작성하지 않으면 재시작 시 이전 설정으로 돌아간다.

#### 방법 2: 수동 생성 (`SAVE` / `BGSAVE`)

| 커맨드 | 동작 방식 | 사용 권장 |
|--------|----------|----------|
| `SAVE` | **동기** 방식. 저장 완료까지 다른 모든 클라이언트 명령 차단함 | 운영 환경에는 권장하지 않음 |
| `BGSAVE` | `fork()`로 자식 프로세스 생성 후 백그라운드 저장. 부모 프로세스는 정상 동작함 | O |

- `BGSAVE` 도중 다시 호출하면 에러 발생 -> `BGSAVE SCHEDULE` 옵션으로 예약 가능 (현재 진행 중인 백업 끝나면 자동 실행)
- `LASTSAVE` 커맨드: RDB가 정상 저장된 마지막 시점을 유닉스 타임스탬프로 반환

#### 방법 3: 복제 사용 시 자동 생성
- 복제본이 `REPLICAOF`로 복제 요청 → 마스터가 RDB 파일을 새로 생성해 복제본에 전달
- 네트워크 이슈로 끊겼다가 재연결될 때도 마찬가지로 RDB 전송
- 즉, **마스터는 상황에 따라 언제든지 RDB 파일을 재생성**할 수 있다.

---

## 4. AOF 방식의 데이터 백업

### 4.1 기본 동작과 설정

```conf
appendonly yes
appendfilename "appendonly.aof"
appenddirname "appendonlydir"
```

- `appendonly yes`로 활성화 시 AOF 파일에 주기적으로 데이터 저장
- 버전 7.0+ 부터 AOF는 **여러 개의 파일**로 저장되며 `appenddirname`에 지정된 디렉터리 하위에 저장된다.
- `appenddirname`에는 **경로가 아닌 디렉터리 이름만** 지정 가능 (`dir` 하위에 생성)
- 실수로 `FLUSHALL` 실행해도 AOF 파일을 열어 `FLUSHALL` 라인만 삭제 후 재시작하면 직전까지 데이터 복구 가능

### 4.2 AOF 저장 형식 (RESP 프로토콜)

```
> SET key1 apple
OK
> SET key1 beer
OK
> DEL key1
(integer) 1
> DEL non_existing_key
(integer) 0
```

위 4개 커맨드 실행 시 AOF에는 다음과 같이 기록:
```
*3
$3
set
$4
key1
$5
apple
*3
$3
set
$4
key1
$4
beer
*2
$3
del
$4
key1
```

- `*N`: 인자 개수
- `$N`: 다음 인자의 바이트 길이
- **메모리에 영향을 주지 않는 작업은 기록되지 않음** → 마지막 `DEL non_existing_key`는 AOF에 안 남음ㄴ

### 4.3 커맨드 변환 규칙: AOF가 항상 그대로 저장하는 건 아니다

#### (1) BRPOP -> RPOP
```
> RPUSH mylist a b c d e   # (integer) 5
> BRPOP mylist 1           # 1) "mylist"  2) "e"
```
- 블로킹 기능은 AOF에서 굳이 명시할 필요가 없으므로 `BRPOP` -> `RPOP`로 기록.

#### (2) INCRBYFLOAT -> SET
```
> SET counter 100
OK
> INCRBYFLOAT counter 50
"150"
```
- 아키텍처에 따라 부동소수점 처리 방식이 달라질 수 있으므로, AOF에는 **증분 후의 값을 직접 SET하는 커맨드로 변환되어 저장**:
```
SET counter 100
SET counter 150
```

### 4.4 AOF 파일 재구성 (Rewrite)

#### 왜 필요한가
- AOF는 이름 그대로 **append-only**로 동작 -> 시간에 비례해 파일 크기가 계속 증가
- 예: `INCR counter`를 100번 실행하면 메모리에는 단일 값만 있지만, AOF에는 100번의 실행 내역이 그대로 남음
- 따라서 **주기적으로 압축**시키는 재구성(rewrite) 작업이 필요하다.

#### 재구성 동작 원리
- 기존 디스크의 AOF 파일을 사용하는 게 아니라, **메모리에 있는 데이터를 읽어 새로운 파일로 저장**
- `aof-use-rdb-preamble yes` (기본값) → RDB 파일 형태(바이너리)로 저장
- `aof-use-rdb-preamble no` → `*.base.aof` (RESP 텍스트)로 저장
- 재구성도 `fork()`로 자식 프로세스를 만들어 처리

### 4.5 버전 7 이전 vs 이후 구조 변화

#### 버전 7 이전: 단일 파일
```
┌─────────────────────────┐
│   RDB file (binary)     │  <- 고정 영역
├─────────────────────────┤
│   AOF tail              │
│   (RESP 프로토콜)         │  <- 증분 영역
└─────────────────────────┘
```

재구성 과정 (4단계):
1. `fork`로 자식 프로세스 생성 -> 메모리 데이터를 임시 파일에 저장
2. 동시에 변경 내역은 **기존 AOF 파일과 인메모리 버퍼에 동시 기록** (이중 저장)
3. 재구성 완료 시 인메모리 버퍼 내용을 임시 파일 마지막에 추가
4. 임시 파일로 기존 AOF 파일 덮어쓰기

**단점**: (2)에서 데이터 이중 저장, 하나의 파일에 바이너리와 텍스트가 혼재되어있음 -> 수동 관리가 복잡함

#### 버전 7 이후: 분리된 다중 파일 + manifest
```
appenddirname/
├── appendonly.aof.15.base.rdb   <- 고정 영역 (스냅숏)
├── appendonly.aof.15.incr.aof   <- 증분 영역 (RESP)
└── appendonly.aof.manifest      <- 어떤 파일을 사용 중인지 명시
```

manifest 파일 내용 예:
```
file appendonly.aof.15.base.rdb seq 15 type b
file appendonly.aof.15.incr.aof seq 15 type i
```

재구성될 때마다 파일명 번호와 manifest의 `seq` 값이 1씩 증가된다.

재구성 과정 (개선됨):
1. `fork`로 자식 프로세스 생성 -> 메모리 데이터를 신규 임시 파일에 저장
2. 변경 내역은 **신규 AOF 파일에만** 저장 (이중 저장 제거)
3. 임시 manifest 파일 생성 -> 변경된 버전으로 업데이트
4. 임시 manifest로 기존 manifest 덮어쓴 뒤 이전 AOF, RDB 파일 삭제

**중요한 설계 철학**
> AOF 재구성 과정은 모두 **순차 입출력(sequential I/O)** 만 사용함 -> 복원 시에도 순차적으로 데이터를 로드하므로 랜덤 I/O를 전혀 고려하지 않아 디스크 접근이 매우 효율적이라고 볼 수 있다.

### 4.6 자동 / 수동 AOF 재구성

#### 자동 재구성 옵션
```conf
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

- `auto-aof-rewrite-percentage`: 마지막으로 재구성된 AOF 크기 대비 현재 AOF가 지정 % 만큼 커지면 재구성 시도
- `auto-aof-rewrite-min-size`: 재구성된 이후 AOF의 최소 크기. 이 값보다 작으면 재구성 안 함

확인 방법:
```
> INFO Persistence
# Persistence
...
aof_current_size:186830
aof_base_size:145802
...
```

- `aof_base_size=145802`, `percentage=100` -> `aof_current_size`가 100% 커진 **291604**가 되면 자동 재구성 시도
- `min_size`가 없으면 데이터가 1KB로 줄어든 직후마다 재구성을 트리거해 매우 비효율적 -> 최소 크기 지정이 필수

#### 수동 재구성
```
> BGREWRITEAOF
```
- 원하는 시점에 직접 AOF 파일을 재구성. 자동 재구성과 동일하게 동작.

### 4.7 AOF 타임스탬프 및 시점 복원 (Point-in-Time Recovery)

#### v7+ 타임스탬프 활성화
```conf
aof-timestamp-enabled no  # 기본값
```

활성화 시 AOF에 타임스탬프가 함께 기록됨:
```
#TS:1669532240
*2
$6
SELECT
$1
0
*3
$3
set
$1
a
$1
b
```

#### 시점 복원 예시: 실수로 FLUSHALL 실행했을 때
```bash
$ src/redis-check-aof --truncate-to-timestamp 1669532844 appendonlydir/appendonly.aof.manifest
Start checking Multi Part AOF
Start to check BASE AOF (RDB format).
[offset 0] Checking RDB file appendonly.aof.15.base.rdb
...
RDB preamble is OK, proceeding with AOF tail...
Truncate nothing in AOF appendonly.aof.15.base.rdb to timestamp 1669532844
BASE AOF appendonly.aof.15.base.rdb is valid
Start to check INCR files.
Successfully truncated AOF appendonly.aof.15.incr.aof to timestamp 1669532844
All AOF files and manifest are valid
```

> **주의**: `--truncate-to-timestamp`는 **원본 파일을 직접 변경**한다. 작업 전 반드시 원본 파일을 다른 위치에 복사 백업해야 한다. 또한 타임스탬프 기능은 v7+ 에서만 지원되며 이전 버전과 호환되지 않음을 주의하자.

### 4.8 AOF 파일 복원 (손상 시)

서버 장애로 AOF 작성 중 중단됐을 수 있다.
```bash
$ src/redis-check-aof appendonlydir/appendonly.aof.manifest
...
0x              120f7: Expected to read 18 bytes, got 9 bytes
AOF analyzed: filename=appendonly.aof.15.incr.aof, size=73984, ok_up_to=73957, ok_up_to_line=8756, diff=27
AOF appendonly.aof.15.incr.aof is not valid. Use the --fix option to try fixing it.
```

`--fix` 옵션으로 손상 부분을 잘라내 복구:
```bash
$ src/redis-check-aof --fix appendonlydir/appendonly.aof.manifest
...
This will shrink the AOF appendonly.aof.15.incr.aof from 73984 bytes, with 27 bytes, to 73957 bytes
Continue? [y/N]: y
```

> **주의**: `--fix`도 원본 파일을 변경하므로 안전한 작업을 위해 사전에 백업을 해줘야 한다.

---

## 5. AOF 파일의 안전성

### 5.1 OS 시스템 콜과 디스크 쓰기 메커니즘

```
┌─────────────────────────┐
│      애플리케이션          │ <- 레디스
└──────────┬──────────────┘
           │ WRITE  ↕  success
┌──────────▼──────────────┐
│  커널 영역 (OS BUFFER)    │ <- 메모리
└──────────┬──────────────┘
           │ FSYNC  ↕  success
┌──────────▼──────────────┐
│   디스크 (AOF 파일)        │
└─────────────────────────┘
```

- 애플리케이션이 `WRITE` 시스템 콜로 파일에 저장하려 해도 **곧바로 디스크에 쓰이지 않는다**. 데이터는 일단 커널 영역의 OS 버퍼에 저장됨.
- OS는 커널에 여유가 있거나 **최대 지연 시간인 30초**에 도달하면 그제야 디스크에 내려쓴다.
- `FSYNC`는 OS 버퍼 내용을 **강제로 디스크에 플러시**하도록 강제하는 시스템 콜

### 5.2 APPENDFSYNC 옵션: 성능 vs 안정성 트레이드오프

| 옵션 | 동작 | 성능 | 최대 데이터 손실 |
|------|------|------|----------------|
| `APPENDFSYNC no` | WRITE만 호출. OS가 알아서 FSYNC | **가장 빠름** | 최대 **30초** |
| `APPENDFSYNC always` | 매 쓰기마다 WRITE + FSYNC | **가장 느림** | 거의 0 |
| `APPENDFSYNC everysec` | WRITE는 매번, FSYNC는 **1초에 한 번** | no와 거의 동급 | 최대 **1초** |

- **기본값은 `everysec`** -> 속도와 안정성의 균형점.
- `always`: 데이터가 매우 느려질 수 있음
- `no`: 장애 시 최대 30초 데이터 유실 위험이 있음
- **일반적인 경우 `everysec` 권장**.

---

## 6. 백업 사용 시 주의할 점

### 6.1 Copy-On-Write (COW)와 메모리 2배 문제

- `BGSAVE`로 RDB 저장하거나 AOF 재구성 시 레디스는 `fork()`로 자식 프로세스를 만든다.
   - 자식: 메모리를 파일에 저장 / 부모: 기존처럼 클라이언트 요청 처리
   - 이때 레디스는 **Copy-On-Write 방식**으로 메모리를 복사해 백업과 서비스를 동시에 진행한다.

```
Process 1 ──> ┌────────────┐ <── Process 2
              │  page A    │
              │  page B    │
              │  page C    │
              │ Copy of C  │  <- 부모가 page C 수정 시 복사본 생성
              └────────────┘
              Physical Memory
```

- 부모가 page를 수정하면 그 페이지만 복사 -> 수정할 게 많을수록 메모리 사용량이 증가
   - **최악의 경우 기존 메모리 용량의 2배 사용**할 수 있음
- `maxmemory`를 너무 크게 설정하면 COW로 OS 메모리가 가득 차고, **OOM(Out Of Memory)** 으로 서버가 다운될 수 있음

### 6.2 maxmemory 권장 값 (서버 RAM 대비)

| RAM | Maxmemory | 비율 |
|-----|-----------|------|
| 2GB | 638MB | 33% |
| 4GB | 2048MB | 50% |
| 8GB | 4779MB | 58% |
| 16GB | 10240MB | 63% |
| 32GB | 21163MB | 65% |
| 64GB | 43008MB | 66% |

> 서버 RAM이 클수록 비율을 높게 잡아도 되지만 최대 **66% 정도**가 안전선이다.

### 6.3 동시 실행 제약
- **RDB 스냅숏 저장 중**에는 AOF 재구성 기능을 사용할 수 없다.
- **AOF 재구성 중**에는 `BGSAVE`를 실행할 수 없다.
   - fork와 디스크 쓰기 자원이 겹치기 때문
