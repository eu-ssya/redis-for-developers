# 개발자를 위한 레디스 — 2장 · 3장 통합 정리

---

# Chapter 02 · 레디스 시작하기

설치 방식 선택부터 OS 튜닝, 설정 파일, 접속까지. 운영에 투입하기 전 밟아야 할 단계를 정리했다.

## 1. 실습 환경 준비 — WSL 설치

실습 환경이 윈도우라면, 먼저 **WSL(Windows Subsystem for Linux)** 을 설치해야 한다.

- **WSL이란?**
  - Windows Subsystem for Linux의 약자
  - 윈도우 환경에서 리눅스 커널의 일부 기능을 그대로 사용할 수 있게 해주는 호환 계층
- **설치 방법**
  - PowerShell을 관리자 권한으로 실행한 뒤 `wsl --install` 실행
  - 설치 완료 후 시스템 재부팅 필요
  - 재부팅 후엔 윈도우 PowerShell에서 우분투를 사용할 수 있게 됨

```bash
# PowerShell을 관리자 권한으로 실행
wsl --install

# 재부팅 후 확인
wsl --list --verbose

# Ubuntu 진입
wsl
```

---

## 2. 레디스 설치 — 패키지 저장소 등록 후 apt로 설치

리눅스에 레디스를 설치하는 방식은 **패키지 설치**와 **소스 빌드** 두 가지가 있다. 빠르고 간편한 패키지 방식부터 살펴보자. 운영 환경에서 여러 인스턴스를 관리하거나 특정 버전을 고정해야 한다면 소스 빌드가 더 유리하다.

### 2-1. 저장소 등록

레디스 공식 GPG 키를 내려받아 apt 저장소로 등록한다.

```bash
curl -fsSL https://packages.redis.io/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] \
  https://packages.redis.io/deb $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/redis.list
```

### 2-2. 패키지 목록 갱신 후 설치

apt 캐시를 업데이트해야 방금 추가한 저장소가 반영된다. **update를 빠뜨리고 install부터 하면 실패하니 주의.**

```bash
# 패키지 목록 갱신 (필수)
sudo apt-get update

# Redis 설치 (redis-server + redis-cli 함께 설치됨)
sudo apt-get install redis
```

### 2-3. 레디스 서버 켜기

```bash
sudo systemctl --now enable redis-server
```

정상 동작했다면 다음과 같은 심볼릭 링크가 생성된다.

```
Created symlink /etc/systemd/system/redis.service
  -> /lib/systemd/system/redis-server.service.

Created symlink /etc/systemd/system/multi-user.target.wants/redis-server.service
  -> /lib/systemd/system/redis-server.service.
```

---

## 3. 서버 환경 구성 — OS 레벨 튜닝 4가지

설치만 하고 바로 실행하면 기본값이 발목을 잡을 것이다. **레디스는 싱글 스레드 · 메모리 기반**이므로 OS 수준의 작은 병목 하나가 전체 서비스 레이턴시와 가용성을 흔들 수 있기 때문이다. 운영 전 다음 4가지를 점검한다.

### 3-1. Open Files — 파일 디스크립터 한도

파일 디스크립터는 OS가 파일·네트워크 연결에 부여하는 번호다. 레디스는 **클라이언트 1명당 FD 1개**를 소비한다.

- 레디스의 기본 `maxclients` 값은 **10,000**
- 레디스 프로세스는 내부 용도로 **32개의 FD를 예약**해 사용
- 따라서 서버의 FD 한도가 **`maxclients + 32` = 최소 10,032 이상**이어야 함
- 부족하면 실행 시 maxclients가 자동으로 하향 조정됨

```
필요 FD 수  =  maxclients  +  32
```

```bash
# 현재 open files 한도 확인
ulimit -a | grep open
# 예: open files    (-n)    1024  ← 너무 작음
```

부족하다면 `/etc/security/limits.conf` 파일에 다음과 같이 추가한다.

```
*    hard    nofile    100000
*    soft    nofile    100000
```

재로그인 후 `ulimit -a`로 다시 확인한다.

### 3-2. THP 비활성화

리눅스는 메모리를 **페이지 단위(기본 4KB)** 로 관리한다. 메모리 크기가 커질수록 페이지를 관리하는 테이블(TLB)도 커지고 오버헤드가 발생하기 때문에, 페이지를 크게(2MB) 합쳐 자동 관리하는 **THP(Transparent Huge Page)** 기능이 도입됐다.

하지만 레디스 같은 DB 애플리케이션에서는 오히려 역효과다. `fork()` 시 **큰 페이지를 복사하는 비용이 증가**해 퍼포먼스가 떨어지고 레이턴시가 치솟기 때문이다. 그래서 **반드시 이를 비활성화**한다.

**일시적 비활성화** (root 권한 필요)

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

**영구적 비활성화** — `/etc/rc.local`에 다음 구문 추가

```bash
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
```

부팅 시 자동 실행되도록 권한을 부여한다.

```bash
chmod +x /etc/rc.d/rc.local
```

### 3-3. vm.overcommit_memory = 1 — COW 메커니즘 대응

레디스는 디스크에 데이터를 저장할 때 `fork()`로 백그라운드 프로세스를 만들고, 이때 **COW(Copy-On-Write)** 메커니즘이 동작한다.

- 부모 프로세스와 자식 프로세스가 **동일한 메모리 페이지를 공유**
- 데이터가 변경될 때마다 해당 페이지를 복사
- 변경이 많을수록 메모리 사용량이 빠르게 증가 → 일시적 초과 할당 상황 발생

리눅스의 `vm.overcommit_memory` 기본값은 `0`(보수적, 초과 할당 거부)이라서 fork가 실패할 수 있다. **`1`로 바꿔야** 안정적으로 백그라운드로 저장된다.

```bash
# 즉시 적용 (재부팅 없이)
sudo sysctl vm.overcommit_memory=1
```

**영구 적용** — `/etc/sysctl.conf` 파일 말미에 추가

```
vm.overcommit_memory = 1
```

### 3-4. somaxconn · syn_backlog — TCP 백로그 큐 크기

TCP 3-way handshake 과정에서 연결은 두 개의 큐를 거친다.

- `syn_backlog` — SYN만 받고 아직 성립 중인 요청 큐
- `somaxconn` — 성립이 완료돼 `accept()` 대기 중인 요청 큐

`redis.conf`의 `tcp-backlog` 파라미터는 레디스 인스턴스가 클라이언트 연결을 받을 때 사용하는 백로그 큐 크기를 지정합니다. 기본값은 **511**이지만, OS의 `somaxconn`과 `tcp_max_syn_backlog` 값보다 클 수 없으므로, 두 OS 값을 함께 올려둬야 한다. 트래픽 스파이크 시 연결이 거부되는 것을 막기 위함이다.

```bash
# 현재 값 확인
sysctl -a | grep syn_backlog
sysctl -a | grep somaxconn
```

**영구 설정** — `/etc/sysctl.conf`에 추가

```
net.core.somaxconn = 1024
net.ipv4.tcp_max_syn_backlog = 1024
```

> 💡 `sysctl -p` 또는 `sysctl <키>=<값>` 명령으로 재부팅 없이 커널 파라미터를 즉시 반영할 수 있다.

---

## 4. redis.conf — 운영 필수 파라미터 6가지

레디스는 `redis.conf` 파일로 모든 동작을 제어한다. 운영 투입 전 반드시 확인해야 할 6가지 파라미터다.

| 파라미터                     | 기본값           | 설명                                                                                                           |
| ---------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------- |
| `port`                       | `6379`           | 클라이언트 접속을 받을 TCP 포트                                                                                |
| `bind`                       | `127.0.0.1 -::1` | 접근 허용 IP. `0.0.0.0`은 모든 IP 허용이지만 보안 위험. 운영에선 명시 IP 권장                                  |
| `protected-mode`             | `yes`            | 패스워드 미설정 시 로컬 연결만 수신. 외부 접속이 안 될 때 1순위 체크 항목                                      |
| `requirepass` / `masterauth` | (없음)           | 서버 접속 패스워드 / 리플리카→마스터 인증. 복제 환경에서는 동일값 권장                                         |
| `daemonize`                  | `no`             | `yes`로 변경 시 백그라운드 데몬 실행 + PID 파일 생성. 경로는 `pidfile`로 제어 (기본 `/var/run/redis_6379.pid`) |
| `dir`                        | `./`             | 워킹 디렉터리. 로그·RDB·AOF 저장 경로. 운영에선 명시적 절대 경로 권장                                          |

**bind 사용 예**

```
bind 192.168.12.34 127.0.0.1
```

→ `192.168.12.34`와 `127.0.0.1`로 들어오는 연결만 허용한다는 뜻.

WSL 기준으로 `redis.conf` 파일은 `/etc/redis/` 폴더에 있다. `vi` 편집기로 열어 필요한 값을 수정하면 된다.

---

## 5. 레디스 실행과 기본 명령어 실습

### 5-1. 프로세스 시작 / 종료

```bash
# 시작
sudo service redis-server start

# 종료
sudo service redis-server stop
```

> `sudo redis-cli shutdown`은 인증·설정에 따라 실패할 수 있으므로 `service` 명령어를 사용하는 게 안전하다.

### 5-2. 레디스 접속하기 — redis-cli

설치 시 함께 제공되는 `redis-cli`로 레디스에 접속한다. 두 가지 모드가 있다.

**대화형 모드** — IP 생략 시 기본값 `127.0.0.1`로 접속, 연결을 끊을 때까지 세션이 유지된다.

```bash
$ redis-cli
127.0.0.1:6379> PING
PONG
```

**커맨드라인(단발성) 모드** — 한 번 실행 후 즉시 종료. 스크립팅·CI에 적합.

```bash
$ redis-cli PING
PONG
```

> `requirepass`가 설정돼 있다면 접속 시 `-a <pw>` 옵션을 주거나, 접속 후 `AUTH <pw>` 커맨드로 인증한다.

### 5-3. 데이터 저장과 조회 — SET / GET

```bash
# Key-Value 저장
127.0.0.1:6379> SET hello world
OK

# Key로 Value 조회
127.0.0.1:6379> GET hello
"world"

# 등록되지 않은 Key 조회
127.0.0.1:6379> GET nothing
(nil)
```

`SET` 명령어의 두 번째 파라미터는 value이므로, 등록되지 않은 키를 `GET`하면 `(nil)`이 반환되는 게 정상 동작이다.

---

# Chapter 03 · 자료구조와 키 관리

> 사이드 프로젝트(카카오 소셜 로그인 + Refresh Token 관리)를 진행하면서 등장한 Redis 관련 내용을 정리했다.
> 레디스의 자료구조와 키를 관리하는 법 등 기본 개념을 다룬다.

---

## 1. 레디스의 자료구조

레디스는 다양한 자료구조를 제공하지만, 본 정리에서는 **사이드 프로젝트에서 실제로 사용된 String · Hash · Set** 세 가지에 집중한다.

각 자료구조는 저장 단위와 연산 특성이 다르다. 목적에 맞게 선택하는 것이 좋다.

| 자료구조   | 저장 형태               | 주요 명령어                       | 특징                                                  |
| ---------- | ----------------------- | --------------------------------- | ----------------------------------------------------- |
| **String** | Key <-> Value 1:1       | `SET` / `GET` / `DEL`             | 가장 단순. 최대 512MB 문자열·바이너리·JSON 저장       |
| **Hash**   | Key <-> Field-Value Map | `HSET` / `HGET` / `HGETALL`       | 한 키 아래 여러 필드. 객체 표현에 적합, 유연한 스키마 |
| **Set**    | Key <-> Unique Items    | `SADD` / `SMEMBERS` / `SISMEMBER` | 정렬 없는 유일 원소 집합. 교집합·합집합 연산 제공     |

---

### 1-1. String

String은 Redis의 **가장 간단한 자료구조**다. 최대 **512MB**의 문자열 데이터를 저장할 수 있다.

**저장 가능한 데이터 종류**

- 일반 문자열
- **바이너리 세이프** — 이진 데이터를 포함하는 모든 종류의 문자열 (예: JPEG 이미지)
- HTTP 응답 값 (JSON)

String은 **키와 실제 저장되는 아이템이 1:1로 연결되는 유일한 자료구조**다. String이 아닌 다른 자료구조에서는 하나의 키에 여러 개의 아이템이 저장된다.

#### 기본 명령어

```bash
SET key value          # 저장
GET key                # 조회
DEL key                # 삭제

# 캐시 무효화 (강제로 공개키 재조회 유도)
DEL "oidc_public_keys::SimpleKey []"

# 수동으로 캐시 데이터 주입 (테스트용)
SET "oidc_public_keys::SimpleKey []" '{"keys":[{"kid":"abc"}]}'
```

#### 실전 예: @Cacheable + RedisCacheManager로 OIDC 공개키 캐싱

**상황**

- 카카오 JWKS 엔드포인트에서 받아온 공개키를 Redis에 캐싱
- 캐시 이름: `oidc_public_keys`
- TTL: **3일** (259,200초) — 카카오 키 교체 주기를 감안한 캐시 유효기간

```java
// PublicKeyCacheConfig.java
RedisCacheConfiguration.defaultCacheConfig()
    .serializeKeysWith(...)   // 키: StringRedisSerializer → 사람이 읽을 수 있는 문자열
    .serializeValuesWith(...) // 값: GenericJackson2JsonRedisSerializer → JSON 직렬화
    .entryTtl(Duration.ofDays(3));
```

```java
// KakaoTokenClient.java
@Cacheable(cacheNames = OIDC_PUBLIC_KEYS, cacheManager = "oidcCacheManager")
@GetMapping("/.well-known/jwks.json")
PublicKeysDto getKakaoPublicKeys();
// 캐시 히트 시 실제 HTTP 호출 없이 Redis에서 반환
```

#### 실제 Redis에 저장되는 키 구조

키 이름은 Spring Cache가 두 부분을 `::` 구분자로 이어붙여 자동 생성한다.

```
oidc_public_keys :: SimpleKey []
       ①                 ②
```

- **① `oidc_public_keys`** — `@Cacheable(cacheNames = OIDC_PUBLIC_KEYS)`에 지정한 캐시 이름
  - 커스텀 `oidcCacheManager`는 `RedisCacheConfiguration.defaultCacheConfig()`를 직접 생성하므로, `application.yml`의 `key-prefix` 설정이 적용되지 않아 접두사가 없음
- **② `SimpleKey []`** — 캐시 키. `getKakaoPublicKeys()`는 파라미터 없는 메서드이므로 Spring이 자동으로 "파라미터 없음"을 의미하는 `SimpleKey []`를 키로 사용
  - 파라미터가 있다면 `SimpleKey [파라미터값]` 형태가 됨

#### 실제 저장되는 값 구조 — @class 타입 힌트

`GenericJackson2JsonRedisSerializer`는 역직렬화 시 어떤 클래스로 복원할지 알아야 하므로, 값을 저장할 때 **`@class` 필드를 JSON에 자동으로 포함**시킵니다. 개발자가 직접 작성하는 것이 아니라 직렬화 시 자동 삽입되는 **타입 힌트**다.

```bash
GET "oidc_public_keys::SimpleKey []"
```

```json
{
  "@class": "brucehan.auth.infrastructure.kakao_client.dto.PublicKeysDto",
  "keys": [
    "java.util.ArrayList",
    [
      {
        "@class": "brucehan.auth.infrastructure.kakao_client.dto.PublicKeysDto$JWK",
        "kid": "3f9698038...긴 kid 값",
        "kty": "RSA",
        "alg": "RS256",
        "use": "sig",
        "n": "q8y3...긴 모듈러스 값",
        "e": "AQAB"
      }
    ]
  ]
}
```

> **`JWK`의 `@class` 불확실성**
> `GenericJackson2JsonRedisSerializer`는 `NON_FINAL` 타이핑을 사용한다. Java `record`는 묵시적으로 `final`이므로 원칙적으로 `@class`가 붙지 않아야 하지만, `JWK`가 `Serializable`을 구현하고 있으면 Jackson 버전에 따라 `@class`가 포함될 수도 있다. **실제 저장 값은 애플리케이션을 실행해 `GET` 명령어로 직접 확인이 필요하다.**

#### 실습 명령어 모음

```bash
# 캐시 키 찾기 (prefix: 없음, 캐시명: "oidc_public_keys")
KEYS "oidc_public_keys*"

# 캐시 값 조회 (JSON 직렬화된 PublicKeysDto)
GET "oidc_public_keys::SimpleKey []"

# 캐시 무효화 (강제로 공개키 재조회 유도)
DEL "oidc_public_keys::SimpleKey []"

# 캐시 수동 저장
SET "oidc_public_keys::SimpleKey []" '{"keys":[{"kid":"abc","kty":"RSA","alg":"RS256","use":"sig","n":"modulus","e":"AQAB"}]}'
```

---

### 1-2. Hash

Hash는 흔히 생각하는 자료구조 그대로, **필드-값 쌍을 가진 집합**이다. 레디스 전체에서는 데이터가 key-value 쌍으로 저장되고, 하나의 hash 자료구조 내부에서는 **필드-값 쌍으로 저장**된다.

- 필드는 하나의 hash 내에서 유일
- 필드와 값 모두 문자열 데이터로 저장

#### Hash의 특징

- **객체 표현에 적합** — RDB 테이블 ↔ Redis Hash 변환이 간편
- **동적 필드 추가 가능** — 칼럼이 고정된 RDB와 달리 유연한 개발 가능
- **각 아이템마다 다른 필드 가능** — 같은 객체 데이터를 저장하더라도 서비스 특성에 맞게 데이터 저장소를 선택하는 것이 중요

#### 주요 명령어

- `HSET` — 한 번에 여러 필드-값 저장
- `HGET` / `HMGET` — 단건 / 다건 조회
- `HGETALL` — 모든 필드-값 반환

#### 실전 예: @RedisHash로 Refresh Token 관리

**상황**

- `memberId -> refreshToken` 저장
- TTL: **14일** (1,209,600초) — 장기 세션 유지와 보안을 위한 만료 설정
- 로직: 로그인 시 저장 -> 재발급 시 검증 후 교체 (Refresh Token Rotation)

```java
@RedisHash(value = "refresh_token", timeToLive = 1209600)
@RequiredArgsConstructor
@Getter
public class RefreshTokenEntity {
    @Id
    private final String memberId;
    private final String refreshToken;
}
```

`@RedisHash`는 내부적으로 Redis의 Hash 자료구조를 사용한다.

#### Refresh Token Rotation 시뮬레이션

```bash
# 저장된 RefreshToken 키 찾기
# (Spring Data Redis 네이밍 컨벤션: 클래스명:id)
KEYS RefreshTokenEntity:*

# 특정 memberId의 refresh token 해시 조회
HGETALL "RefreshTokenEntity:12345"
# 결과:
# 1) "memberId"
# 2) "12345"
# 3) "refreshToken"
# 4) "eyJhbGciOiJIUzI1NiJ9..."

# 특정 필드만 조회
HGET "RefreshTokenEntity:12345" "refreshToken"

# 1. 로그인 → 토큰 저장
HSET "RefreshTokenEntity:1" memberId "1" refreshToken "token_v1"

# 2. /token/refresh 호출 → 기존 토큰 검증 후 새 토큰으로 교체
HGET "RefreshTokenEntity:1" refreshToken  # "token_v1" 확인
HSET "RefreshTokenEntity:1" refreshToken "token_v2"

# 로그아웃 시 토큰 삭제
DEL "RefreshTokenEntity:1"
```

---

### 1-3. Set

Set은 **정렬되지 않은 문자열의 집합**이다.

- 하나의 set 자료구조 내에서 **아이템은 중복해서 저장되지 않음**
- 교집합·합집합·차집합 등의 집합 연산 제공
- 객체 간의 관계를 계산하거나 유일한 원소를 구해야 할 때 적합
- `SISMEMBER`로 **O(1) 존재 여부 조회** 가능

`SADD` 커맨드를 사용하면 set에 아이템을 저장할 수 있으며, 한 번에 여러 개의 아이템을 저장할 수 있다. 정렬은 없지만 `SISMEMBER`로 O(1) 조회가 가능하다.

#### 찍먹 — Spring Data Redis 내부 인덱싱

개발자가 직접 쓰지는 않지만, `@RedisHash`를 사용하면 **Spring Data Redis가 조회 인덱스를 위해 자동으로 Set을 생성**한다.

```bash
# Spring Data Redis가 자동 생성한 인덱스 Set
SMEMBERS "refresh_token"
# 1) "1"
# 2) "2"
# 3) "99999"
# → 저장된 모든 memberId 목록 (findAll() 구현을 위한 인덱스)
```

JPA의 `findAll()`을 호출하면 내부적으로 **이 Set에서 모든 ID를 가져온 뒤 각 Hash를 읽는 방식**으로 동작한다 (MyBatis도 동일).

```
findAll() 호출
  → SMEMBERS "refresh_token"     (Set에서 ID 목록 조회)
  → HGETALL "refresh_token:1",
    HGETALL "refresh_token:2",
    ...                          (각 Hash 조회)
```

#### 참고: RedisKeyValueAdapter.put()

```
spring-projects/spring-data-redis
  → src/main/java/org/springframework/data/redis/core/
    → RedisKeyValueAdapter.java
      → put() 메서드 안에 SADD 호출 로직

put() 메서드 안에서:
  // 엔티티 저장 (Hash)
  redisOps.execute(...HSET {keyspace}:{id} ...)

  // 인덱스 Set에 ID 추가 (Set)
  redisOps.boundSetOps(keyspace).add(id)
  // → SADD "refresh_token" "1"
```

JPA의 `save()` 호출 시 내부적으로 `RedisKeyValueAdapter` 클래스를 통해 `SADD`가 실행된다는 것을 직접 확인할 수 있다.

---

## 2. 레디스에서 키를 관리하는 법

키 관리 명령어는 용도에 따라 4가지 카테고리로 나뉜다.

| 카테고리      | 대표 명령어                            | 용도                                |
| ------------- | -------------------------------------- | ----------------------------------- |
| **만료 관리** | `EXPIRE`, `TTL`, `PERSIST`, `EXPIREAT` | 키에 생명 주기를 부여하고 확인·해제 |
| **키 탐색**   | `KEYS`, `SCAN`, `EXISTS`               | 저장된 키를 찾거나 존재 여부 확인   |
| **키 삭제**   | `DEL`, `UNLINK`                        | 동기 vs 비동기 삭제의 영향 차이     |
| **타입 확인** | `TYPE`                                 | 키에 저장된 자료구조 타입 반환      |

---

### 2-1. 만료 관리 — EXPIRE · TTL · PERSIST · EXPIREAT

#### EXPIRE

```
EXPIRE key seconds [ NX | XX | GT | LT ]
```

키가 만료될 시간을 초 단위로 정의하며, 옵션을 함께 쓸 수 있다.

| 옵션 | 의미                                                  |
| ---- | ----------------------------------------------------- |
| `NX` | 해당 키에 만료 시간이 정의돼 있지 **않을 때만** 수행  |
| `XX` | 해당 키에 만료 시간이 정의돼 **있을 때만** 수행       |
| `GT` | 현재 키의 만료 시간보다 **새 값이 더 클 때만** 수행   |
| `LT` | 현재 키의 만료 시간보다 **새 값이 더 작을 때만** 수행 |

#### TTL

```
TTL key
```

키가 몇 초 뒤에 만료되는지를 알려준다.

```
TTL 상태값:
 -2 → 키 자체가 존재하지 않음
 -1 → 키는 있지만 만료 설정이 없음 (영구 저장)
  N → N초 후 만료
```

#### EXPIRETIME

```
EXPIRETIME key
```

키가 삭제되는 유닉스 타임스탬프를 초 단위로 반환한다. TTL과 동일한 규칙으로 -1, -2를 반환한다.

#### PERSIST

만료 시간(TTL)이 설정된 키에서 제한 시간을 제거하여 키가 영구적으로 유지되도록 만든다.

#### EXPIREAT

```
EXPIREAT key unix-time-seconds [ NX | XX | GT | LT ]
```

키가 특정 유닉스 타임스탬프에 만료되도록 만료 시간을 직접 지정한다. 옵션은 EXPIRE와 동일.

#### 실습

```bash
# TTL 설정
SET testkey "hello"
EXPIRE testkey 10        # 10초 후 만료
PEXPIRE testkey 10000    # 밀리초 단위 (10,000ms = 10초)

# TTL 확인
TTL testkey         # 남은 초
PTTL testkey        # 남은 밀리초
EXPIRETIME testkey  # 만료될 unix timestamp (절대값)

# 만료 제거 → 영구 키로 전환
PERSIST testkey
TTL testkey         # -1

# 특정 시각에 만료 (절대 시간 지정)
EXPIREAT testkey 1800000000
```

#### 사이드 프로젝트의 TTL 전략

| 데이터        | TTL                | 이유                              |
| ------------- | ------------------ | --------------------------------- |
| Refresh Token | 14일 (1,209,600초) | 장기 세션 유지, 보안을 위한 만료  |
| OIDC 공개키   | 3일 (259,200초)    | 카카오 키 교체 주기를 감안한 캐시 |

---

### 2-2. 키 탐색 — KEYS · SCAN · EXISTS

#### KEYS

```
KEYS pattern
```

레디스에 저장된 모든 키 중 **패턴에 매칭되는 모든 키의 list**를 반환한다. 패턴은 글롭(glob) 스타일로 동작한다.

| 패턴       | 매칭 예                                     |
| ---------- | ------------------------------------------- |
| `h?llo`    | `hrllo`, `hillo` (한 글자 와일드카드)       |
| `h*llo`    | `hllo`, `haeillo` (여러 글자 와일드카드)    |
| `h[ae]llo` | `hello`, `hallo` 매칭. `hyllo`는 매칭 안 됨 |
| `h[^e]llo` | `hallo`, `hkllo` 매칭. `hello`는 매칭 안 됨 |
| `h[yz]llo` | `hyllo`, `hzllo`만 매칭                     |

> **KEYS 사용 시 절대 주의할 점**
>
> 레디스에 100만 개의 키가 저장돼 있다면 모든 키의 정보를 반환한다. **레디스는 싱글 스레드**로 동작하기 때문에 실행 시간이 오래 걸리는 커맨드 수행 중에는 다른 클라이언트의 모든 커맨드가 차단된다.
>
> KEYS 명령을 수행하면 메모리에 저장된 모든 키를 읽어오는 동안 다른 클라이언트가 수행하는 모든 set, get 커맨드는 대기한다. 메모리에서 모든 데이터를 읽어오는 작업은 얼마나 걸릴지 예상할 수 없으며, 그동안 마스터는 아무 동작을 할 수 없다.
>
> 다른 클라이언트가 데이터를 저장할 수 없어 대기열이 늘어나고, 모니터링 도구의 health check에 응답할 수 없어 **의도하지 않은 페일오버가 발생**할 수도 있다.

#### EXISTS

```
EXISTS key [key ..]
```

키가 존재하는지 확인한다. 존재하면 `1`, 없으면 `0` 반환.

```bash
# 로그아웃 후 재발급 시도를 막는 로직의 근거
EXISTS "refresh_token:1"
# → 0 반환 → JWT_REFRESH_NOT_FOUND_IN_REDIS 예외
```

#### SCAN — KEYS의 안전한 대체재

```
SCAN cursor [MATCH pattern] [COUNT count] [TYPE type]
```

SCAN은 KEYS를 대체해 키를 조회할 때 사용한다. KEYS는 한 번에 모든 키를 반환하지만, SCAN은 **특정 범위의 키만 조회**할 수 있어 비교적 안전하다.

```bash
> SCAN 0
1) "17"
2)  1) "key:12"
    2) "key:8"
    ...
    11) "key:1"

> SCAN 17
1) "0"
2)  1) "key:5"
    2) "key:18"
    ...
    9) "key:11"
```

**처음 SCAN을 사용할 때는 커서에 `0`을 입력**한다. 첫 번째 반환값은 다음 SCAN 호출 시 사용할 **커서 위치**, 두 번째 반환값은 키 list다. 클라이언트는 **커서가 0이 될 때까지 SCAN을 반복**해 모든 키를 확인할 수 있다.

기본적으로 한 번에 반환되는 키 개수는 약 10개이며, `COUNT` 옵션으로 조정한다. 다만 정확히 지정한 개수만큼 출력되지는 않고, 레디스가 메모리를 스캔하며 데이터 형상에 따라 1~2개를 더 읽기도 한다.

> COUNT 옵션을 너무 크게 설정해 한 번에 반환되는 값이 많아져 출력 시간이 길어진다면 KEYS와 동일하게 서비스에 영향을 줄 수 있다.

#### MATCH 옵션의 동작 특성

**SCAN은 먼저 데이터를 필터링 없이 스캔한 다음, 반환하기 직전에 필터링**한다.

레디스에 `key:0` ~ `key:200`까지 저장돼 있고, `11`이라는 문자열이 포함된 키를 찾는 경우를 비교해보자.

```bash
# KEYS는 한 번에 다 반환
> KEYS *11*
 1) "key:118"
 2) "key:110"
 3) "key:111"
 ...
11) "key:113"
```

```bash
# SCAN + MATCH는 매칭 결과가 커서 순회 동안 분산되어 나타남
> SCAN 0 match *11*
1) "48"
2) 1) "key:115"

> SCAN 48 match *11*
1) "52"
2) (empty array)

> SCAN 52 match *11*
1) "190"
2) 1) "key:111"
```

따라서 한 번의 SCAN 호출에서 원하는 값이 안 나오거나 빈 배열이 반환될 수 있다. **커서가 0이 될 때까지 반복해야 완전한 결과**를 얻는다.

`TYPE` 옵션도 동일하게 반환 직전에 필터되므로, 원하는 타입을 찾기까지 오래 걸릴 수 있다.

#### SSCAN, HSCAN, ZSCAN

각각 set, hash, sorted set에서 아이템을 조회하기 위해 사용된다. `SMEMBERS`, `HGETALL`, `ZRANGE WITHSCORE`를 대체해 **서버에 최대한 영향을 끼치지 않고 반복 호출할 수 있도록** 사용하는 커맨드다.

#### 실습

```bash
# 패턴으로 키 검색 (전체 스캔이라 블로킹 위험 있음)
KEYS *
KEYS "refresh_token:*"
KEYS "oidc_public_keys*"

# 운영 환경에서는 KEYS 대신 SCAN 사용 (논블로킹 커서 방식)
SCAN 0 MATCH "refresh_token:*" COUNT 100

# 키 존재 여부 확인 (0 또는 1 반환)
EXISTS "refresh_token:1"
```

---

### 2-3. 키 삭제 — DEL · UNLINK

#### DEL

```
DEL key [key ...]
```

키와 키에 저장된 모든 아이템을 삭제하는 커맨드이며, **기본적으로 동기적으로 작동**한다.

#### UNLINK

```
UNLINK key [key ...]
```

DEL과 비슷하게 키와 데이터를 삭제하지만, **백그라운드에서 다른 스레드에 의해 처리**되며 우선 **키와 연결된 데이터의 연결을 끊는다**.

#### 왜 UNLINK가 필요한가

set, sorted set처럼 **하나의 키에 여러 개의 아이템이 저장된 자료구조**의 경우, 1개의 키를 삭제하는 DEL 커맨드 수행이 인스턴스에 큰 영향을 끼칠 수 있다.

100만 개의 아이템이 저장된 sorted set 키를 DEL로 삭제하는 것은, 전체 키가 100만 개 있는 레디스에서 **동기적인 방식으로 FLUSHALL을 수행하는 것과 같다**. 수행되는 시간 동안 다른 클라이언트는 아무런 커맨드를 사용할 수 없다.

따라서 **키에 저장된 아이템이 많은 경우 DEL이 아니라 UNLINK를 사용**해 데이터를 삭제하는 것이 좋다.

#### DEL vs UNLINK 비교

| 항목      | `DEL`                   | `UNLINK`                       |
| --------- | ----------------------- | ------------------------------ |
| 처리 방식 | 동기 (메인 스레드 점유) | 비동기 (별도 스레드)           |
| 동작      | 키·데이터 즉시 삭제     | 키 연결 해제 + 백그라운드 정리 |
| 대용량 키 | **블로킹 위험**         | 안전                           |
| 권장 상황 | 작은 키의 단순 삭제     | 다량 아이템 Hash/Set/ZSet      |

> `lazyfree-lazy-user-del = yes` 설정 시 모든 DEL이 UNLINK처럼 백그라운드로 동작한다. 버전 7 기준 **기본값은 `no`**.

#### 실습

```bash
# 단일 삭제 (동기)
DEL "refresh_token:1"

# 다중 삭제
DEL "refresh_token:1" "refresh_token:2" "refresh_token:3"

# UNLINK: 비동기 삭제 (대용량 키에 유리)
UNLINK "refresh_token:1"
```

---

### 2-4. 타입 확인 — TYPE

```
TYPE key
```

지정한 키의 자료구조 타입을 반환한다.

```bash
TYPE "refresh_token:1"                  # → hash
TYPE "oidc_public_keys::SimpleKey []"   # → string
TYPE "refresh_token"                    # → set (인덱스 컬렉션)
```

---

## 3. 실전 활용 — 카카오 소셜 로그인 인증 플로우

지금까지 다룬 자료구조와 명령어가 실제 서비스에서 어떻게 조합되는지, 카카오 소셜 로그인의 5단계 흐름으로 정리했다.

| 단계 | 시나리오            | 핵심 명령어                 | Redis 동작                          |
| ---- | ------------------- | --------------------------- | ----------------------------------- |
| 1    | 로그인 성공         | `HSET` + `EXPIRE`           | Refresh Token 저장 · TTL 14일       |
| 2    | OIDC 공개키 캐시    | `SET` + `EXPIRE`            | 최초 조회 후 3일간 재사용           |
| 3    | 토큰 재발급 요청    | `HGETALL` → `HGET` → `HSET` | 기존 검증 · 새 토큰 교체 (Rotation) |
| 4    | 로그아웃            | `DEL`                       | Refresh Token 키 즉시 무효화        |
| 5    | 만료 후 재발급 시도 | `EXISTS` → `0` → 예외       | `JWT_REFRESH_NOT_FOUND_IN_REDIS`    |

### redis-cli로 재현하는 인증 플로우

```bash
# Step 1 · 로그인 성공 → Refresh Token 저장
HSET "RefreshTokenEntity:member_1" \
     memberId "member_1" \
     refreshToken "refresh_abc123"
EXPIRE "RefreshTokenEntity:member_1" 1209600

# Step 2 · OIDC 공개키 캐시 (TTL 3일)
SET "oidc_public_keys::SimpleKey []" '{"keys":[...]}'
EXPIRE "oidc_public_keys::SimpleKey []" 259200

# Step 3 · /token/refresh 호출
HGETALL "RefreshTokenEntity:member_1"           # 존재 확인
HGET "RefreshTokenEntity:member_1" refreshToken # "refresh_abc123" 검증

# 검증 통과 → 새 토큰 발급 → 교체 (Rotation)
HSET "RefreshTokenEntity:member_1" refreshToken "refresh_xyz789"
EXPIRE "RefreshTokenEntity:member_1" 1209600

# Step 4 · 로그아웃 (토큰 무효화)
DEL "RefreshTokenEntity:member_1"

# Step 5 · 로그아웃 후 재발급 시도 → 없는 키
EXISTS "RefreshTokenEntity:member_1"
# → 0 반환 → JWT_REFRESH_NOT_FOUND_IN_REDIS 예외
```

---

## 출처

- 김가림, 『**개발자를 위한 레디스**』
