# 레디스 운영하기 — 13장 정리

# Part 01 · 프로메테우스 & 그라파나로 레디스 모니터링 구축하기

운영의 출발점은 관찰이다. **Exporter**가 메트릭을 노출하면 **Prometheus**가 끌어와(pull) 저장하고, **Grafana**가 그린다. 임계치를 넘으면 **Alertmanager**가 사람에게 알린다.

## 1. 아키텍처 — 누가 무엇을 수집하는가

`Exporter`는 대상의 상태를 실시간으로 스크랩해 메트릭으로 노출하는 프로그램이다. Prometheus는 지정한 타깃(Exporter)에 **직접 접근해 pull 방식**으로 시계열 데이터를 가져온다.

```
┌──────────────────────────┐
│ Redis 서버 1             │
│   Redis Exporter         │──┐
│   Node Exporter          │  │
├──────────────────────────┤  │   pull    ┌──────────────┐     ┌────────────┐
│ Redis 서버 2             │  ├─────────► │  Prometheus  │ ──► │  Grafana   │
│   Redis Exporter         │  │           │ 시계열·알림룰 │     │  대시보드  │
│   Node Exporter          │  │           └──────┬───────┘     └────────────┘
├──────────────────────────┤  │                  │
│ Redis 서버 3 (인스턴스 2)│  │                  ▼
│   Redis Exporter × 2     │──┘           ┌──────────────┐
│   Node Exporter × 1      │              │ Alertmanager │ ──► SMS · Email · Slack
└──────────────────────────┘              └──────────────┘
```

- **Redis Exporter** — 지정한 레디스 인스턴스의 실시간 정보(메모리·연결·복제 등) 수집. **인스턴스마다 1개** 필요.
- **Node Exporter** — 레디스가 도는 **서버의 하드웨어·OS 메트릭** 수집. 한 서버에 인스턴스가 2개여도 Node Exporter는 1개면 된다.

## 2. 구성과 포트 — 서버 두 대 실습 구성

대상 서버에 Exporter들을, 별도 모니터링 서버에 Prometheus·Grafana·Alertmanager를 올린다. Prometheus가 대상 서버의 Exporter를 pull로 긁어온다.

```
10.0.0.1 · 레디스 서버                   10.0.0.2 · 모니터링 서버
  redis           : 6379       scrape      prometheus    : 9090
  redis_exporter  : 9121    ◄───────────   grafana       : 3000
  node_exporter   : 9100                   alertmanager  : 9093
```

> **설치 순서**: `node_exporter` → `redis_exporter` → `alertmanager` → `prometheus` → `grafana` 순으로 백그라운드 실행.
> Grafana 기본 접속은 `:3000`, 초기 계정은 `admin / admin`.

## 3. Exporter 설정 & 알림 규칙

Redis Exporter는 플래그(또는 환경변수)로 **수집 대상과 인증**을 지정한다. 알림은 Prometheus의 **alert rule**로 임계치를 정의하고, 발화 시 Alertmanager가 채널로 보낸다.

**redis_exporter 주요 플래그**

| 플래그 | 설명 |
|---|---|
| `redis.addr` | 수집 대상 주소 · 기본 `redis://localhost:6379` |
| `redis.user` | ACL 사용 시 유저명 |
| `redis.password` | 접속 비밀번호 |
| `web.listen-address` | 노출 주소 · 기본 `0.0.0.0:9121` |

**Prometheus 스크랩/알림 구성**

- `scrape_configs` 에 Redis·Linux job 등록 (타깃: `:9121`, `:9100`)
- `alerting` → Alertmanager(`:9093`) 연결, `rule_files` 로 규칙 로드
- Alertmanager `route` / `receivers` 에서 Discord webhook 등 지정

**alert.rules 예시**

```yaml
- alert: RedisDown
  expr: redis_up == 0
  annotations: { summary: "Redis down (instance {{ $labels.instance }})" }

- alert: RedisMissingMaster        # 마스터가 사라짐
  expr: (count(redis_instance_info{role="master"}) or vector(0)) < 1

- alert: RedisReplicationBroken     # 복제본 수 급감
  expr: delta(redis_connected_slaves[1m]) < 0
```

## 4. Grafana 대시보드 — 두 가지 길

Prometheus를 **중간 저장소로 거치는 방식**과, Grafana의 **Redis 플러그인으로 직결하는 방식**이 있다. 전자는 과거 시점 그래프를, 후자는 Exporter 없는 실시간 조회를 준다.

| | 방식 A · Prometheus 경유 | 방식 B · Redis 플러그인 직결 |
|---|---|---|
| 데이터소스 | Prometheus(`http://10.0.0.2:9090`) | 레디스 주소 직접 (Standalone·Cluster·Sentinel) |
| 대시보드 | 공식 import — **node exporter full = ID 1860** | RedisGrafana 플러그인(Redis / Redis Application) |
| 특징 | 외부 보관 → **과거 시점 그래프** 조회 가능 | **Exporter 불필요**, `redis-cli` 패널로 직접 커맨드 |
| 환경 | — | 온프레미스·클라우드 모두 실시간 |

---

# Part 02 · 레디스 버전 업그레이드

레디스는 릴리스 주기가 빠르고 EOL이 짧다. 버그·보안 취약점을 피하려면 EOL 이전 버전을 유지하도록 주기적으로 올리는 편이 좋다.

## 1. 두 가지 방법 — 다운타임 vs 접속 정보 변경

고가용성(Sentinel·Cluster) 구성이면 **복제본부터 하나씩 올려 무중단**이 가능하고, 싱글 구성이면 둘 중 하나의 비용을 감수해야 한다.

| 방법 | 내용 | 트레이드오프 |
|---|---|---|
| ① 신규 서버 설치 + 데이터 복제 | 새 버전을 새 서버에 올리고 기존 데이터 복제 | **다운타임 없음**, 단 **접속 정보 변경 필요** |
| ② 인스턴스 중지 후 신규 소스로 재실행 | 실행 중 인스턴스 멈추고 새 바이너리로 재기동 | **접속 정보 유지**, 단 싱글이면 **다운타임 발생** |

> **버전 확인**: 실행 전후로 `redis-server --version` 또는 접속 후 `INFO server`(`redis_version`)로 확인.

## 2. Sentinel 구성 — 센티널 → 복제본 → 페일오버 → 마스터

핵심 원칙: **상태 없는 센티널을 먼저, 복제본을 그다음, 마스터는 페일오버 뒤 마지막**. 전 과정 무중단.

1. **[Sentinel]** 센티널 3대를 신규 바이너리로 교체
   → 3대 모두 종단 → 기존 `sentinel.conf` 복사 → 새 바이너리로 시작. 센티널은 상태가 없어 안전.
2. **[Replica]** 복제본 인스턴스를 먼저 교체
   → `config rewrite`로 설정을 파일에 반영 → `shutdown` → 기존 `redis.conf` 복사 → 새 바이너리로 시작.
3. **[Failover]** 수동 페일오버로 역할 교대
   → `sentinel failover mymaster` 실행. 기존 마스터에서 `INFO replication` → `role:slave` 확인.
4. **[Master]** 이제 복제본이 된 기존 마스터를 교체
   → 2단계와 동일(중단·설정 복사·재실행). 필요하면 페일오버를 한 번 더 수행해 원래 마스터로 **페일백**.

## 3. Cluster 구성 — 복제본부터 올리고, 페일오버로 번갈아

클러스터는 **센티널 인스턴스를 고려할 필요가 없어 더 간단**하다. 마스터 A·B·C와 그 복제본 D·E·F가 있을 때:

```
A ↔ D     B ↔ E     C ↔ F
(master ↔ replica 쌍)
```

1. 복제본 **D·E·F** 각각 버전 업그레이드
2. **D에서 페일오버** → D가 마스터, A가 복제본으로
3. **A** 업그레이드
4. **E에서 페일오버**
5. **B** 업그레이드
6. **F에서 페일오버**
7. **C** 업그레이드

원칙: 복제본 먼저 → 페일오버로 역할 교대 → 기존 마스터 순.

---

# Part 03 · 장애·성능 저하를 부르는 설정 항목

기본값이 항상 안전하지는 않다. 메모리 정책, 백업 실패 처리, 자동 백업 빈도 — 운영 전에 의도에 맞게 손봐야 한다.

## 1. maxmemory-policy — 메모리가 가득 차면 무엇을 버릴까

기본값 `noeviction`은 메모리 한계에 닿으면 **어떤 키도 버리지 않고 쓰기를 거부(에러)**한다. 데이터 유실은 막지만, 더 이상 입력을 못 받아 애플리케이션 장애로 이어질 수 있다.

```
메모리 사용량 → maxmemory 도달
        │
        ├─ noeviction  (기본) → 쓰기 거부 · 에러 반환        
        └─ allkeys-lru (권장) → 오래된 키 제거 · 입력 계속   
```

> **권장**: 일부 데이터가 삭제되더라도 계속 새 입력을 받아야 한다면 `allkeys-lru`로 설정.

## 2. stop-writes-on-bgsave-error — 백업이 실패하면 쓰기를 멈출까

기본값 `yes`는 RDB 스냅숏 저장이 실패하면 **모든 쓰기를 중지**한다. 최신 백업 실패를 인지시켜 더 큰 장애를 막기 위함이다.

| 값 | 동작 | 언제 |
|---|---|---|
| `yes` (기본) | 쓰기 중지 | 백업 실패 = 서버 문제임을 즉시 알림. 안전 우선. |
| `no` | 쓰기 계속 | 이미 다른 모니터링이 충분하고, 쓰기가 끊기면 안 되는 경우. |

## 3. 자동 백업 옵션 — RDB·AOF는 의도한 시간에만

백업(RDB/AOF)은 운영 인스턴스에 부하를 준다. 백그라운드 백업은 **COW(Copy-On-Write)**로 동작해 메모리가 최대 `maxmemory`의 **2배**까지 늘 수 있어, 적절한 `maxmemory` 없이는 최악의 경우 OOM을 부른다.

**RDB · save** — 기본값 `3600 1 / 300 100 / 60 10000` = 1시간 1회 · 5분 100회 · 1분 10000회 변경 시 자동 생성.

```bash
> CONFIG SET save ""   # 자동 RDB 끄기 (의도한 시간에만 수동 백업)
OK
```

**AOF · rewrite** — `auto-aof-rewrite-percentage 100`(기존보다 100% 증가 시 재작성) · `auto-aof-rewrite-min-size` 64MB(`67108864`).

```bash
> CONFIG SET auto-aof-rewrite-percentage 0   # 자동 재작성 막기
OK
```

> **COW 메모리 폭증**: 포그라운드 백업은 메인 스레드를 막아 성능에 영향을 미치며, 백그라운드 백업은 메모리를 최대 2배까지 사용한다. 백업은 의도한 시간·인스턴스(가능하면 복제본)에서 수행하도록 설정한다.

---

# Part 04 · 레디스 운영 & 성능 최적화

레디스는 커맨드를 실행하는 동안 **싱글 스레드 이벤트 루프**로 동작한다.

## 1. 싱글 스레드의 함정 — O(N) 이상의 커맨드를 지양한다

한 번에 하나의 커맨드만 처리하므로, **실행이 길어지면 다른 모든 클라이언트가 대기**한다. 흔한 오해는 `KEYS`·`FLUSHALL`만 느리다는 생각인데, **set·list·hash 같은 컬렉션 커맨드도 아이템 수에 비례**해 느려진다.

> 지연을 막는 가장 단순한 규칙: **하나의 키 안에 아이템을 너무 많이 담지 않는다.** 시간 복잡도의 N은 대개 "그 자료 구조 안의 아이템 개수"다.

## 2. 커맨드 복잡도 레퍼런스 — 피해야 할 O(N) 커맨드 지도

| 그룹 | 커맨드 | 복잡도 | 비고 |
|---|---|---|---|
| **키스페이스** (N=전체 키) | `KEYS` | O(N) | 모든 키 순회 → `SCAN`으로 대체 |
| | `FLUSHALL` / `FLUSHDB` | O(N) | 전체 삭제. `ASYNC` 옵션이면 백그라운드 수행 |
| **공통** (N=내부 아이템) | `DEL` | O(1) / O(N) | string은 O(1), 컬렉션은 O(N)·포그라운드 → 크면 `UNLINK` |
| | `SORT` / `SORT_RO` | O(N+M·logM) | list·set·zset 정렬. `SORT_RO`는 읽기 전용(복제본 가능) |
| **set** | `SDIFF` · `SUNION` (+STORE) | O(N) | 차집합 · 합집합 |
| | `SINTER` · `SINTERCARD` | O(N·M) | 교집합. `SINTERCARD`(v7)는 카디널리티만, `LIMIT` 도달 시 종료 |
| **list** (인덱스 주의) | `LINDEX` · `LINSERT` · `LSET` · `LPOS` | O(N) | 양 끝 제외 인덱스 접근은 도달까지 순회(최악 전체) |
| **hash** | `HGETALL` · `HKEYS` · `HVALS` | O(N) | 전체 필드/값 반환 |
| **sorted set** | 기본 접근 | O(logN) | 입력 시 자동 정렬 → 인덱스·스코어 접근 효율적 |
| | `ZUNION` (+STORE) | O(N)+O(M·logM) | 합집합 |
| | `ZINTER` · `ZINTERCARD` | O(N·K)+O(M·logM) | 교집합 (v7 `ZINTERCARD`: 카디널리티 + `LIMIT`) |
| | `ZDIFF` (+STORE) | O(L+(N-K)·logN) | 차집합 |

- `SINTERCARD`/`ZINTERCARD`(v7): 결과 집합 전체가 아니라 **카디널리티만** 반환. `LIMIT` 값에 도달하면 연산 종료.
- `SORT_RO`(v7): `STORE` 옵션 불가 → 순수 읽기 전용 → **복제본에서도 수행 가능**.

## 3. Lazy Freeing — DEL은 동기, UNLINK는 비동기

`DEL`은 메인 스레드에서 동기로 메모리를 해제한다. 아이템이 수백만인 큰 키는 **비동기로 해제하는 `UNLINK`**가 낫다. 삭제는 사용자 호출 외에 **eviction·만료·RENAME·복제 FLUSH** 같은 시스템 상황에서도 일어난다.

```
메인 스레드 · DEL  ──(조건 판단)──►  bio 스레드 · UNLINK
동기 해제 (큰 키면 차단)            비동기 해제 (차단 없음)
```

**freeObjAsync — `redis/src/lazyfree.c`**

```c
#define LAZYFREE_THRESHOLD 64

if (free_effort > LAZYFREE_THRESHOLD && obj->refcount == 1) {
    bioCreateLazyFreeJob(...);   // 비동기 해제
} else {
    decrRefCount(obj);           // 동기 해제 (DEL)
}
```

즉, **해제 노력(free_effort)이 64를 초과하고 참조 카운트가 1**일 때만 비동기로 해제하고, 그 외에는 동기로 삭제한다.

> ℹ️ **lazyfree 설정 5종 (기본 모두 `no`)**: `lazyfree-lazy-eviction` · `lazyfree-lazy-expire` · `lazyfree-lazy-server-del` · `replica-lazy-flush` · `lazyfree-lazy-user-del`.
> `yes`로 켜면 연결 해제 후 메모리에 잠시 남아 사용량이 늘 수 있어 모니터링이 필요하다.

## 4. 트랜잭션 vs 루아 스크립트

둘 다 여러 커맨드를 **원자적**으로 실행해 중간에 다른 클라이언트의 영향을 차단한다. 결정적 차이는 **롤백**이다.

| | MULTI / EXEC | Lua Script |
|---|---|---|
| 실행 | `MULTI` → 명령들 `QUEUED` → `EXEC` 원자 실행 | 서버에서 원자적으로 실행, 1회 왕복 |
| 오류 시 | **전체 롤백** | **롤백 없음** (다음 명령으로 진행) |
| 장점 | 일관성 보장 | 복잡한 로직 + 네트워크 절감 |
| 재사용 | — | `SCRIPT LOAD` → hash → `EVALSHA`로 **재전송 없이 재실행** |

```redis
> MULTI
OK
(TX)> INCRBY account_balance 100
QUEUED
(TX)> RPUSH transaction_log "입금 100"
QUEUED
(TX)> EXEC
1) (integer) 100
2) (integer) 1
```

> **블로킹 커맨드 금지**: 실행 중 다른 클라이언트가 모두 대기하므로 길이를 짧게. 트랜잭션·스크립트 내부의 `BLPOP`·`BRPOP`는 무한 대기를 막기 위해 강제 차단된다.
> → `(error) EXECABORT Transaction discarded ...` / `(error) ERR This Redis command is not allowed from script ...`

## 5. has-get / has-del 안티패턴

`EXISTS`로 존재를 확인한 뒤 `GET`/`DEL` 하는 패턴은 **왕복 2회 + 원자성 붕괴**를 부른다. 확인과 처리 사이에 다른 클라이언트가 키를 바꿀 수 있고, 애초에 `GET`은 없으면 `nil`, `DEL`은 없어도 에러가 없다.

```
- has-get (2 round trips)   EXISTS key → GET key   (사이에 변경 가능)
- 직접    (1 round trip)    GET key                (없으면 nil)
```

**벤치마크 · 10만 키 조회**

```
exists + get   ████████████████████████████████  223.43초
get 직접       ████████████████                   110.71초   (≈ 2배 개선)
```

불필요한 패턴을 없앤 것만으로 약 2배 성능 개선. 대규모 트래픽에서는 1초 차이도 누적된다.

## 6. 클라이언트 출력 버퍼 (client-output-buffer-limit)

**client output buffer**는 서버 응답을 클라이언트로 보내기 전 임시 저장한다. 대용량 데이터나 복제 시에는 키워야 한다.

설정 형식: `client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>`

| 클래스 | hard | soft | soft seconds | 의미 |
|---|---|---|---|---|
| `normal` | `0` | `0` | `0` | 제한 없음 (일반 클라이언트는 무제한 수신) |
| `replica` | `256mb` | `64mb` | `60` | 복제본 — 큰 데이터 복제 대비 |
| `pubsub` | `32mb` | `8mb` | `60` | pub/sub 구독자 |

- **hard limit** 초과 → 즉시 연결 종료.
- **soft limit** 초과가 **soft seconds(60초)** 동안 지속 → 연결 종료.

> `maxmemory`·`maxclients`를 키울 때 **복제 버퍼도 함께** 조정. 부족하면 부분 동기화(partial sync) 실패·데이터 손실 위험.

## 7. 키스페이스 알림 (notify-keyspace-events)

**keyspace notification**은 내부 pub/sub 채널로 키 변경 사항을 구독하게 해준다. `notify-keyspace-events`에 **채널(K/E)** 과 **이벤트 타입** 문자를 조합해 지정한다.

**채널 클래스**

- `K` = keyspace · `__keyspace@<db>__` 접두사
- `E` = keyevent · `__keyevent@<db>__` 접두사

**이벤트 타입 문자**

| 문자 | 의미 | | 문자 | 의미 |
|---|---|---|---|---|
| `g` | generic (DEL·EXPIRE·RENAME 등) | | `x` | expired (만료) |
| `$` | string | | `e` | evicted (이빅션) |
| `l` | list | | `m` | key miss (없는 키 접근) |
| `s` | set | | `n` | new key (신규 키) |
| `h` | hash | | `A` | `g$lshztxed` 별칭 (**m·n 제외 전체**) |
| `z` | sorted set | | | |
| `t` | stream | | | |

```redis
> CONFIG SET notify-keyspace-events Ex      # E=keyevent, x=expired
OK
> SUBSCRIBE __keyevent@0__:expired
1) "message"
2) "__keyevent@0__:expired"
3) "my_key"
```

> **유실 주의 — pub/sub은 fire-and-forget**: 많은 키가 동시에 만료되면 메시지가 pub/sub 버퍼를 초과해 **유실**될 수 있다. 한 번 발행된 이벤트는 재확인 불가, 연결이 끊긴 동안의 이벤트도 놓친다. 안정성이 필요하면 버퍼를 키우거나 여러 인스턴스로 분산.

## 8. 특정 프리픽스 키 삭제 — SCAN+DEL 반복 vs 루아 한 번

레디스는 프리픽스 일괄 삭제를 기본 제공하지 않는다(마스터에서만 가능). `SCAN`으로 찾아 삭제하는데, 진짜 비용은 연산이 아니라 **네트워크 왕복**이다. 1000만 키 중 100만 개가 매치되면 **100만 번의 추가 왕복**이 든다.

```
기본 방식
  A: SCAN 실행 (cursor 0까지)   +   B: DEL key 실행 (매 키마다)
  → 매치 수만큼 왕복 누적

루아 방식
  EVALSHA 실행  ── 스크립트 내부에서 SCAN + DEL 함께 수행
  → DEL 호출 생략 → A 횟수만큼만 왕복 → 네트워크 I/O 절감
```

**기본 방식 (Python)**

```python
pattern = 'prefix:*'
count, cursor, keys = 100, 0, []

# SCAN으로 특정 프리픽스 키 검색
while True:
    cursor, partial_keys = r.scan(cursor, match=pattern)
    keys.extend(partial_keys)
    if cursor == 0:
        break

# 검색된 키 삭제
for key in keys:
    r.delete(key)
```

**루아 방식 (스크립트 내부에서 SCAN+DEL)**

```lua
local cursor  = ARGV[1]
local pattern = ARGV[2]
local count   = ARGV[3]

local keys    = redis.call("SCAN", cursor, "MATCH", pattern, "COUNT", count)
local cursor  = keys[1]
local keyList = keys[2]

for _, key in ipairs(keyList) do
    redis.call("DEL", key)
end

return {cursor, #keyList}
```

> **차단 시간은 count로 조절**: 스크립트 실행 중 다른 클라이언트는 차단된다. 적절한 배치 크기로 `count`(예: 1,000)를 조절해 한 번에 처리하는 키 수와 차단 시간을 최소화한다. (SCAN+DEL 자체는 매우 빠르므로 주 고려 대상은 네트워크 통신 시간.)

---

# Part 05 · 운영 모니터링 지표

안정 운영의 마지막 조각은 "무엇을 볼 것인가"다. 느린 커맨드를 기록하는 SlowLog와, 컴퓨팅 자원을 보는 다섯 가지 그래프 지표를 챙긴다.

## 1. 슬로우 로그 — 느린 커맨드를 메모리에 기록

**SlowLog**는 실행이 느린 커맨드를 기록한다. 주기적으로 검토하면 예상치 못한/악의적 명령을 식별해 성능과 보안 모두에 도움이 된다.

```redis
> SLOWLOG GET
1) 1) (integer) 1923          # ① 고유 ID
   2) (integer) 1696344048    # ② 실행 시점 (unix timestamp)
   3) (integer) 35233         # ③ 실행 소요 시간 (마이크로초!)
   4) 1) "SCAN"               # ④ 느리게 수행된 커맨드
      2) "10179327"
      3) "COUNT"
      4) "50000"
```

레코드 구성: `[① id, ② timestamp, ③ 실행 시간, ④ command...]` (+ Redis 4.0 이상은 client addr / name 추가).

| 설정 | 설명 |
|---|---|
| `slowlog-log-slower-than` | 기록 임계 시간. **마이크로초** 단위, 기본 `10000` = **10ms** |
| `slowlog-max-len` | 유지 레코드 수. 기본 `128`개, 초과 시 오래된 레코드부터 대체 |

> **정확도 메모 — 책 본문 보정**
> 책은 "기본은 10,000ms, 즉 10초"라고 적었지만, `slowlog-log-slower-than`과 SLOWLOG 레코드의 ③ 실행 시간은 **밀리초(ms)가 아니라 마이크로초(μs)** 단위다. 따라서 기본값 `10000`은 **10초가 아니라 10ms**다. (Redis 공식 문서 기준 확인)

## 2. 그래프 지표 — 다섯 가지 운영 대시보드 지표

자원의 **과다 사용**은 높은 지연을, **과소 사용**은 낭비를 뜻한다. CPU·메모리·네트워크·커넥션·복제를 꾸준히 지켜봐야 한다.

### CPU
- 커맨드 실행은 싱글 스레드, **백업·UNLINK는 다른 CPU** 활용.
- O(N) 커맨드·높은 카디널리티 = 부하 증가
- 읽기 부하면 **복제본 참조**로, 백업은 **복제본에서** 수행.

### 메모리
- `used_memory`(논리) vs `used_memory_rss`(운영체제가 할당한 물리).
- 둘의 차 = **단편화(fragmentation)** → `activedefrag`(4.0이상, `CONFIG SET activedefrag yes`)로 관리. 단편화 문제 없으면 켤 필요 없음.
- `DatabaseMemoryUsagePercentage` 100% → maxmemory 정책 작동 / eviction. **과도한 eviction은 CPU 증가**를 부름.

### 네트워크
- 네트워크 I/O 모니터 — VM·도커·k8s의 **대역폭 한계** 주의.
- 한계에 걸리면 처리량이 더 안 오르는 병목 발생.
- 읽기면 복제본 추가, 쓰기면 마이그레이션·**클러스터**로 분산.

### 커넥션
- 활성·신규 연결 수 추이 주시 (급증 = 누수 의심).
- `tcp-keepalive`(기본 300초)로 유휴 연결 정리.
- 연결 설정 비용이 큼 → **커넥션 풀링**, TLS 핸드셰이크는 더 많은 시간·CPU 소모.

### 복제
- 마스터는 복제본이 있으면 명령 스트림을 계속 전송.
- **복제 지연(replication lag)**의 급증은 복제본이 마스터 속도를 못 따라간다는 신호.
- 지연이 커지면 복제본이 **전체 동기화**를 요청할 수 있고, 이때 마스터가 스냅숏을 만들며 성능 저하.
- 원인: 마스터의 과도한 쓰기, 네트워크 대역폭 고갈, 복제 출력 버퍼 크기 문제 → 다른 메트릭과 함께 점검.
