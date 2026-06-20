# 12장 클라이언트 관리(Client Management)

> 레디스가 클라이언트 연결을 **어떻게 수립·유지·정리**하는지, 그리고 성능과 직결되는 **파이프라이닝**과 **클라이언트 사이드 캐싱**을 다룬다.

---

## 0. 개요

레디스가 클라이언트를 처리하는 방법을 살펴보자. 클라이언트의 요청을 어떻게 처리하고 연결을 어떻게 관리하는지, 클라이언트 연결을 정상으로 유지하기 위한 타이머(timeout, keepalive), 그리고 클라이언트 사이드 캐싱까지 다룬다.

핵심 주제는 다음과 같다: 클라이언트 핸들링(연결 방식·멀티플렉싱·maxclients), 클라이언트 출력 버퍼 제한, 클라이언트 이빅션, 타임아웃/TCP keepalive, 파이프라이닝, 클라이언트 사이드 캐싱.

---

## 1. 클라이언트 핸들링

### 1.1 연결 방식: TCP 포트 vs 유닉스 소켓

- 레디스는 클라이언트 연결을 수립하는 데 **TCP 포트**뿐 아니라 **유닉스 소켓(Unix domain socket)**을 사용할 수 있다.
- 일반적으로는 TCP 포트를 사용하지만, **레디스와 클라이언트가 같은 서버(머신)에 있을 때**는 유닉스 소켓을 쓰면 TCP/IP 스택을 거치지 않아 성능을 높일 수 있다.
- 유닉스 소켓을 쓰려면 `redis.conf`에서 `unixsocket`(소켓 파일 경로)과 `unixsocketperm`(소켓 파일 권한)을 설정한다.

```conf
unixsocket /tmp/redis.sock
unixsocketperm 777
```

이렇게 설정하고 레디스를 기동하면 지정한 경로에 소켓 파일이 생성된다. 클라이언트에서는 `-s` 옵션으로 접속한다.

```bash
redis-cli -s /tmp/redis.sock
```

### 1.2 멀티플렉싱과 Non-blocking I/O, TCP_NODELAY

- 레디스는 **멀티플렉싱(Multiplexing)** 방식을 사용한다. 하나의 스레드가 **Non-blocking I/O**로 여러 클라이언트의 요청을 동시에 처리하므로, 클라이언트가 많아도 효율적으로 처리할 수 있다.
- 레디스는 클라이언트 연결을 생성할 때 **`TCP_NODELAY`** 옵션을 사용한다. 이 옵션은 네이글 알고리즘(Nagle's algorithm)을 비활성화해, 작은 데이터라도 버퍼에 모으지 않고 **즉시 전송**하게 한다. 이를 통해 작은 응답의 지연(latency)을 줄인다.

레디스 소스(`redis/src/networking.c`)의 클라이언트 생성 부분을 보면 이 동작을 확인할 수 있다.

```c
client *createClient(connection *conn) {
    client *c = zmalloc(sizeof(client));

    if (conn) {
        connEnableTcpNoDelay(conn);
        if (server.tcpkeepalive)
            connKeepAlive(conn, server.tcpkeepalive);
        connSetReadHandler(conn, readQueryFromClient);
        connSetPrivateData(conn, c);
    }
    ...
}
```

### 1.3 maxclients

- `maxclients`로 레디스에 동시에 연결할 수 있는 **클라이언트 수를 제한**한다.
- 레디스 버전 **2.6 이후 기본값은 10000**이다.
- 레디스는 자기 자신이 사용할 파일 디스크립터로 **32개**를 예약하므로, 실제로 필요한 파일 디스크립터는 `maxclients + 32`이다. OS의 파일 디스크립터 한계가 낮으면 maxclients가 자동으로 줄어들 수 있다.
- `redis.conf` 또는 `CONFIG SET`으로 재설정할 수 있다.

> **repl-disable-tcp-nodelay (복제 연결의 TCP_NODELAY)**
> 레디스 노드 간 **복제 연결** 통신에서는 다음 설정으로 `TCP_NODELAY` 동작을 해제할 수 있다.
> ```conf
> repl-disable-tcp-nodelay yes
> ```
> - `yes`: 복제본에 데이터를 보낼 때 **더 작은 TCP 패킷과 더 적은 대역폭**을 사용한다. 대신 데이터가 복제본에 나타나기까지 **최대 약 40ms 지연**이 생길 수 있다.
> - `no`(기본값): 데이터 지연을 최소화하지만 복제에 **더 많은 대역폭**을 사용한다.
> - 대역폭 대비 트래픽이 매우 높은 환경이라면 `yes`가 도움이 될 수 있다.

---

## 2. 클라이언트 출력 버퍼 제한 (client-output-buffer-limit)

**결론**: 레디스는 클라이언트마다 **출력 버퍼(client output buffer)**를 두는데, 이 버퍼가 무한정 커지면 메모리 부족으로 이어질 수 있으므로 제한을 둔다.

- 레디스는 클라이언트에 반환할 데이터를 저장하기 위해 클라이언트마다 출력 버퍼를 가진다. 1,000개의 클라이언트가 연결되면 출력 버퍼도 1,000개가 생긴다.
- 출력 버퍼가 일정 크기 이상 커지면 레디스 메모리를 계속 점유해 **메모리 부족 현상**이 생길 수 있다.
- 특히 **pub/sub**에서 **발행 속도 > 구독자의 소비 속도**이면 데이터가 빠르게 쌓여 인스턴스 응답이 느려지거나 메모리 부족이 발생할 위험이 크다.
- 따라서 레디스는 출력 버퍼 크기 제한을 두고, **제한을 넘으면 해당 클라이언트 연결을 끊는다.**

**클라이언트 종류별 기본 제한**

| 종류(class) | 하드 제한 | 소프트 제한 |
|-------------|-----------|-------------|
| `normal`(일반) | 제한 없음 | 제한 없음 |
| `slave`(replica, 복제) | 256MB | 60초 동안 64MB |
| `pubsub` | 32MB | 60초 동안 8MB |

- **하드 제한(hard limit)**: 버퍼가 이 값을 넘으면 **즉시** 연결을 끊는다.
- **소프트 제한(soft limit)**: 버퍼가 이 값을 **지정한 시간(duration)** 동안 계속 초과하면 연결을 끊는다.
- 복제본 제한은 복제본이 마스터로부터 **너무 뒤처지지 않게** 하기 위해, pub/sub 제한은 구독자가 메시지를 **제때 가져가게** 하기 위해 중요하다.

**커맨드로 설정**

```text
CONFIG SET client-output-buffer-limit <class> <hard-limit> <soft-limit> <soft-limit-duration>
```
- `<class>`: `normal`, `slave`(replica), `pubsub` 중 하나
- `<hard-limit>`, `<soft-limit>`: 각각 하드/소프트 제한 값(**바이트** 단위), `0`이면 제한 없음
- `<soft-limit-duration>`: 소프트 제한이 적용되는 시간 간격(**초** 단위)

```redis
> CONFIG SET client-output-buffer-limit "slave 0 0 0"
OK
```
위 명령은 복제본 클라이언트의 출력 버퍼 제한을 해제한다(모두 0).

> 참고: 출력 버퍼 외에 클라이언트가 보낸 쿼리를 담는 입력 버퍼는 `client-query-buffer-limit` 설정으로 조정할 수 있다.

---

## 3. 클라이언트 이빅션 (maxmemory-clients, Redis 7.0이상)

클라이언트 연결이 너무 많아 클라이언트용 메모리가 과도해지면 데이터 이빅션이나 OOM이 발생할 수 있다. **레디스 7.0부터** `maxmemory-clients`로 **모든 클라이언트가 사용하는 메모리 총량**을 제한할 수 있다.

- 기존에는 클라이언트 메모리 사용량이 커지면 데이터 저장용 메모리를 압박해 `maxmemory-policy`에 따라 데이터가 이빅션될 수 있었다.
- `maxmemory-clients`는 `redis.conf` 또는 `CONFIG SET`으로 설정한다.
- 값은 **절대값** 또는 **비율(%)**로 지정할 수 있고, **`0`이면 기능 비활성화**(제한 없음)다.

```conf
maxmemory-clients 1g    # 절대값: 클라이언트 메모리 총량 1GB로 제한
maxmemory-clients 5%    # 비율: maxmemory의 5%로 제한
```

- 제한을 넘으면 레디스는 **메모리를 가장 많이 사용하는 클라이언트부터** 연결을 끊어 메모리를 회수한다(client eviction).
- pub/sub 클라이언트 등도 이빅션 대상이 될 수 있다.

### CLIENT NO-EVICT — 특정 클라이언트 보호

특정 클라이언트를 이빅션 대상에서 **제외**하고 싶으면 해당 클라이언트에서 다음을 실행한다.

```redis
> CLIENT NO-EVICT on
OK
```
이 클라이언트는 클라이언트 이빅션 발생 시에도 강제로 연결이 끊기지 않는다.

---

## 4. Timeout과 TCP keepalive

### 4.1 timeout — 유휴(idle) 연결 정리

- 레디스는 클라이언트가 오래 아무 명령도 보내지 않아도 기본적으로 연결을 유지한다. 유용하지만, 사용하지 않는 연결까지 계속 유지하는 단점이 있다.
- `timeout`을 설정하면 클라이언트가 **일정 시간 동안 유휴 상태이면 연결을 해제**한다. `redis.conf` 또는 `CONFIG SET`으로 설정한다.

```redis
> CONFIG SET timeout 600
OK
```
- **기본값은 `0`**이라 기본적으로는 유휴 연결을 끊지 않는다.
- 이 설정은 **pub/sub 클라이언트나 복제 연결의 타이머에는 영향을 주지 않는다.**

### 4.2 tcp-keepalive — 죽은 연결 감지

- `tcp-keepalive`는 레디스가 클라이언트에게 **주기적으로 TCP ACK(keepalive 패킷)**를 보내고, 응답이 있으면 연결을 유지, 없으면 연결을 끊는다. 이를 통해 비정상적으로 끊긴(죽은) 연결을 정리한다.
- 레디스 버전 **3.2.0부터 기본값은 300초**다(매 300초마다 keepalive 확인).
- 네트워크 장애 등으로 응답하지 않는 죽은 피어를 감지하는 데 유용하다.

---

## 5. 파이프라이닝 (Pipelining)

### 5.1 개념과 등장 이유 (RTT)

- 클라이언트가 명령을 보내고 응답을 받기까지의 왕복 시간을 **RTT(Round Trip Time)** 라 한다.
- 명령을 **하나씩 보내고 응답을 기다리면** 매 명령마다 RTT만큼 지연이 생긴다. 레디스 자체 처리 속도가 아무리 빨라도(예: 초당 수만~수십만 건) **네트워크 왕복**이 처리량을 제한한다. 예컨대 RTT가 250ms면 직렬 방식으로는 초당 4건 수준밖에 처리하지 못한다.
- **파이프라이닝**은 여러 명령을 **한 번에 묶어 전송**하고 응답도 한 번에 받아, RTT 지연을 크게 줄인다.

### 5.2 nc로 확인하는 파이프라이닝

```bash
$ (printf "PING\r\nPING\r\nPING\r\n"; sleep 1) | nc localhost 6379
+PONG
+PONG
+PONG
```
3개의 `PING`을 한 번에 보내면 레디스가 차례로 처리해 응답 3개를 돌려준다.

### 5.3 redis-py 파이프라인 예제

```python
import redis

# Redis 서버에 연결
r = redis.Redis(host='localhost', port=6379)

# Pipeline 시작
pipeline = r.pipeline()

# 여러 개의 명령을 Pipeline에 추가
pipeline.set('name', 'Redi')
pipeline.incr('counter')
pipeline.get('name')

# Pipeline 실행
results = pipeline.execute()
```

### 5.4 왜 빨라지는가 & 성능 비교

파이프라이닝이 빠른 이유는 **시스템 콜 횟수**도 줄이기 때문이다. 명령을 하나씩 처리하면 매번 `read()`/`write()` 시스템 콜이 호출되지만, 여러 명령을 묶으면 한 번의 `read()`/`write()`로 처리할 수 있어 시스템 콜 오버헤드와 네트워크 왕복이 모두 줄어든다.

**예제**: 약 100만 개 이상의 키를 `SCAN`으로 모두 가져온 뒤, 타입별 개수와 평균 메모리 사용량을 출력하는 코드의 두 버전을 비교한다.

**(1) 파이프라이닝 미사용**

```python
import redis

# Redis 서버에 연결
r = redis.Redis(host='레디스 주소', port=6379, db=0)

keys = []                 # 키 이름을 저장할 리스트
type_counts = {}          # 각 데이터 타입의 키 수를 저장하는 딕셔너리
total_memory_usage = 0    # 모든 키의 메모리 사용량 합계를 저장하는 변수
cursor = 1                # SCAN 명령을 사용해 키를 가져오기 위한 커서 값

# 모든 키 수집
while cursor != 0:
    cursor, partial_keys = r.scan(cursor=cursor, count=10000)
    keys.extend(partial_keys)

for key in keys:
    # 키의 데이터 타입 조회
    key_type = r.type(key).decode('utf-8')
    # 키의 메모리 사용량 조회
    memory_usage = r.memory_usage(key)

    total_memory_usage += memory_usage
    if key_type in type_counts:
        type_counts[key_type] += 1
    else:
        type_counts[key_type] = 1

for key_type, count in type_counts.items():
    print(f'Type: {key_type}, Count: {count}')

average_memory_usage = total_memory_usage / len(keys) if len(keys) > 0 else 0
print(f'Average Memory Usage: {average_memory_usage} bytes')
```

실행 결과:
```text
Type: 'string', Count: 6197600
Type: 'set', Count: 36922
Type: 'zset', Count: 48
Type: 'hash', Count: 1
Type: 'list', Count: 1
Average Memory Usage: 754.1511653085408 bytes

real    265m2.968s
user    15m12.969s
sys     6m36.587s
```

**(2) 파이프라이닝 사용**

```python
import redis

# Redis 서버에 연결
r = redis.Redis(host='레디스 주소', port=6379, db=0)

keys = []                 # 키 이름을 저장할 리스트
type_counts = {}          # 각 데이터 타입의 키 수를 저장하는 딕셔너리
total_memory_usage = 0    # 모든 키의 메모리 사용량 합계를 저장하는 변수
cursor = 1                # SCAN 명령을 사용해 키를 가져오기 위한 커서 값
batch_size = 50000        # 한 번에 가져올 키의 수

# 모든 키 수집
while cursor != 0:
    cursor, partial_keys = r.scan(cursor=cursor, count=batch_size)
    keys.extend(partial_keys)

pipeline = r.pipeline()   # 파이프라인 시작
for i, key in enumerate(keys):
    pipeline.type(key)            # 데이터 타입 조회
    pipeline.memory_usage(key)    # 메모리 사용량 조회

    # 매 batch_size 번째 또는 마지막 키에 대해 파이프라인 실행
    if (i + 1) % batch_size == 0 or i == len(keys) - 1:
        responses = pipeline.execute()   # 파이프라인 실행
        for j in range(0, len(responses), 2):
            key_type = responses[j]
            memory_usage = responses[j + 1]
            total_memory_usage += memory_usage
            if key_type in type_counts:
                type_counts[key_type] += 1
            else:
                type_counts[key_type] = 1

for key_type, count in type_counts.items():
    print(f'Type: {key_type}, Count: {count}')

average_memory_usage = total_memory_usage / len(keys) if len(keys) > 0 else 0
print(f'Average Memory Usage: {average_memory_usage} bytes')
```

실행 결과:
```text
Type: 'string', Count: 6197600
Type: 'set', Count: 36922
Type: 'zset', Count: 48
Type: 'hash', Count: 1
Type: 'list', Count: 1
Average Memory Usage: 754.1511653085408 bytes

real    3m33.937s
user    3m25.115s
sys     0m1.669s
```

동일 작업이 **약 4시간 25분(real 265m2.968s)** 에서 **약 3분 34초(real 3m33.937s)** 로 단축됐다(real time 기준 약 74배). 특히 `sys` 시간이 6분대 → 1초대로 급감한 데서, 시스템 콜/네트워크 왕복 절감 효과가 크다는 것을 알 수 있다.

### 5.5 파이프라이닝 사용 시 주의사항

- **배치 크기(batch size)** 를 적절히 정해야 한다. 한 번에 너무 많은 명령을 묶으면 클라이언트와 레디스 양쪽에서 응답을 모으는 동안 **메모리 사용량이 커지고**, 레디스가 그 배치를 처리하는 동안 다른 클라이언트의 요청이 지연될 수 있다(레디스는 단일 스레드로 명령을 처리).
- 너무 작으면 파이프라이닝 효과가 줄어든다. 일반적으로 **수만 개 단위**(예: 10,000개 등)로 묶어 메모리/응답성과 처리량의 균형을 맞춘다.
- 네트워크 I/O를 효율적으로 쓰는 만큼, **메모리, 응답 지연, CPU 사용량**을 함께 고려해 배치 크기를 결정하는 것이 좋다.

---

## 6. 클라이언트 사이드 캐싱 (Client-Side Caching, Redis 6+)

레디스 버전6에서 추가된 기능으로, **클라이언트가 조회한 데이터를 자신의 로컬 캐시에 저장**해 두고 재사용함으로써 응답 시간을 단축하고 레디스 서버 부하를 줄인다.

### 6.1 동작 방식

1. 클라이언트가 `GET user:1234`로 값을 조회하면 레디스가 `"Alice"`를 반환한다.
2. 클라이언트는 이 값을 **로컬 캐시**에 저장한다(`user:1234 -> Alice`).
3. 이후 같은 키를 다시 조회하면 레디스까지 가지 않고 **로컬 캐시에서 바로** 가져온다.

문제는 **로컬 캐시의 값이 레디스 원본과 달라질 수 있다**는 점이다(다른 클라이언트가 값을 바꾸면 로컬 캐시는 오래된 stale 값이 됨). 그래서 레디스는 **클라이언트가 캐싱한 키가 변경되면 그 클라이언트에게 무효화(invalidation) 메시지**를 보내, 로컬 캐시를 갱신(또는 삭제)하도록 한다. 이를 위해 레디스는 어떤 클라이언트가 어떤 키를 캐싱했는지 **추적(tracking)** 한다.

### 6.2 두 가지 추적 모드

#### Default Mode (기본 모드) — Invalidation Table
- 레디스가 **각 클라이언트가 어떤 키를 조회·캐싱했는지 정확히** 추적한다.
- **Invalidation Table**에 `키 → 그 키를 캐싱한 클라이언트 ID 목록`을 저장한다. (**키 이름 자체가 인덱스**)

  | Key | Client ID |
  |------|-----------|
  | `ae:98` | 134, 5, 2, 7 |
  | `a:123` | 2, 7 |
  | `b:y12` | 8418, 13 |

- 키가 변경되면 **그 키를 실제로 캐싱한 클라이언트에게만** 무효화 메시지를 보낸다.
- 장점: 정확하다(불필요한 무효화가 거의 없음). 단점: 추적할 키가 많아질수록 **레디스의 메모리 사용량이 늘어난다.**

#### Broadcasting Mode (브로드캐스팅 모드) — Prefix Table
- 레디스가 **키 이름의 프리픽스(앞부분) 단위**로 추적한다.
- **Prefix Table**에 `프리픽스 -> 클라이언트 ID 목록`을 저장한다. (**키 이름의 앞 글자 단위**)

  | Prefix | Client ID |
  |--------|-----------|
  | `name:` | 2, 6 |
  | `page:` | 2123, 317 |
  | `idx:` | 71 |

  예: `name:amazon`, `name:google` -> 둘 다 `name:` 프리픽스로 묶임.
- 클라이언트는 관심 있는 **프리픽스를 구독**하고, 그 프리픽스로 시작하는 키가 변경되면 무효화 메시지를 받는다.
- 장점: 레디스는 **프리픽스만 저장**하면 되므로 메모리를 적게 쓴다. 단점: 클라이언트가 **실제로 캐싱하지 않은 키**의 변경에도 무효화 메시지를 받을 수 있다(불필요한 무효화 발생 가능).

### 6.3 권장 사용법

- 클라이언트 사이드 캐싱은 **자주 조회되지만 변경은 드물게 발생하는 키**에 가장 효과적이다. 이런 키를 캐싱하면 레디스 서버의 **CPU 자원을 절약**할 수 있다.
- 반대로 **자주 변경되는 데이터**를 캐싱하면 매번 무효화 메시지를 받게 되어 캐싱의 이점이 줄고 오버헤드만 커진다.