# 개발자를 위한 레디스 — 4장 정리 · 캐싱 활용 사례

> 김가림 저 『개발자를 위한 레디스』 4장(자료구조 활용 사례)에 맞춰, 사이드 프로젝트(카카오 소셜 로그인)에서
> 직접 실습한 **캐싱 시나리오 두 가지**를 자료구조 관점으로 정리.
>
> - **사례 1**: OIDC 공개키 캐싱 — `String` + `@Cacheable` + RedisCacheManager
> - **사례 2**: Refresh Token 저장 — `Hash` + `@RedisHash` + 자동 인덱스 `Set`
>
> 두 사례 모두 "Redis를 캐시로 쓴다"는 점은 같지만, **자료구조 · Spring 추상화 · 캐시 성격이 모두 달라**
> 자료구조 선택이 왜 중요한지 비교해볼 수 있다.

---

## 캐시(Cache)란 무엇인가

캐시는 **"한 번 가져온 것을, 다음 번에 더 빨리 가져오기 위해 가까운 곳에 둔 사본"** 이다. 원본은 어딘가 멀리(또는 느리게) 있고, 그 원본을 매번 다시 가져오기에는 비용이 비싸기 때문에 사본을 만들어둔다.

캐시가 효과를 보려면 두 가지 조건이 맞아야 한다.

- **읽기가 쓰기보다 훨씬 잦을 것** — 한 번 저장하고 100번 읽으면 99번 이득
- **원본 데이터가 자주 변하지 않을 것** — 자주 변하면 사본이 빨리 오래된 정보(stale)가 됨

읽기·쓰기 비율이 비슷하거나 데이터가 분 단위로 바뀐다면 캐시 도입이 오히려 복잡도만 높일 수 있다. 본 정리에서 다루는 두 사례는 모두 이 조건을 만족한다.

| 사례          | 읽기/쓰기 비율                             | 원본 변경 주기       |
| ------------- | ------------------------------------------ | -------------------- |
| OIDC 공개키   | 읽기 압도적 (인증 요청마다 JWT 검증)       | 3일 (카카오 키 교체) |
| Refresh Token | 읽기 = 검증 시점, 쓰기 = 로그인 / Rotation | 14일 TTL 내 ≤ 1회    |

---

## 왜 레디스가 캐시 저장소로 적합한가

캐시 저장소의 핵심 요구사항은 다음과 같다.

- **빠른 조회** — 원본보다 충분히 빨라야 캐시의 의미가 있음
- **TTL 지원** — 오래된 사본은 자동으로 사라져야 함
- **단순한 인터페이스** — 키만 알면 값을 즉시 꺼낼 수 있어야 함
- **(선택) 원자적 연산** — 토큰 교체처럼 조회+갱신을 묶어야 할 때

레디스는 인메모리 + 단순 명령어 체계 + TTL 기본 지원 + 원자적 연산이라는 네 박자가 모두 맞아 캐시 저장소의 표준이 됐다. 게다가 String만 쓰는 단순 캐시부터 Hash로 객체를 표현하는 구조적 캐시까지, **자료구조의 선택지가 다양**하다는 점이 다른 캐시 솔루션 대비 가장 큰 장점이다.

핵심으로 알아볼 것은 다음과 같다.

> 같은 "캐싱"이라는 목적 안에서도, **데이터의 모양**에 따라 어떤 자료구조를 골라야 하는가?

---

## 두 가지 캐싱 사례 한눈에 비교

| 항목              | 사례 1 · OIDC 공개키 캐싱          | 사례 2 · Refresh Token 저장                     |
| ----------------- | ---------------------------------- | ----------------------------------------------- |
| **자료구조**      | String (단일 JSON 값)              | Hash (필드-값 맵) + 자동 Set (인덱스)           |
| **Spring 추상화** | `@Cacheable` + `RedisCacheManager` | `@RedisHash` + `CrudRepository`                 |
| **데이터 모양**   | 키 1개 <-> JSON 통째로 1덩어리     | 키 1개 ↔ 여러 필드 (memberId, refreshToken)     |
| **읽기 패턴**     | 항상 통째로 (`GET`)                | 필드 단위 가능 (`HGET`) 또는 통째로 (`HGETALL`) |
| **쓰기 패턴**     | 통째로 덮어쓰기 (`SET`)            | 필드 단위 갱신 (`HSET`)                         |
| **TTL**           | 3일 (259,200초)                    | 14일 (1,209,600초)                              |
| **캐시 성격**     | 읽기 캐시 (Look-aside)             | 상태 저장소 겸 캐시                             |
| **원본**          | 카카오 JWKS 엔드포인트 (외부 HTTP) | 없음 (Redis가 정본)                             |

이 표가 보여주는 핵심은, **"캐시 = String"이 아니라는 점**이다. 데이터를 어떻게 읽고 쓸 것인가에 따라 자료구조가 달라진다.

---

## 사례 1 · OIDC 공개키 캐싱 — String + @Cacheable

### 문제 상황

카카오 소셜 로그인에서 받아오는 ID 토큰(JWT)의 서명을 검증하려면, **카카오의 공개키(JWKS)** 가 필요하다.

- 공개키는 카카오가 운영하는 `https://kauth.kakao.com/.well-known/jwks.json` 엔드포인트에서 받아옴
- **로그인 요청이 들어올 때마다** 이 엔드포인트를 호출하면
  - 매 로그인마다 외부 HTTP 호출 -> 레이턴시 증가
  - 카카오 측 rate limit 위험
  - 카카오 서버 장애 시 우리 로그인까지 같이 죽음

원본을 매번 부르면 안 되고, 한 번 받아온 공개키를 일정 기간 **로컬(Redis)에 캐싱**해야 한다.

### 왜 String을 선택했는가

JWKS 응답은 다음과 같은 JSON구조다.

```json
{
  "keys": [
    { "kid": "abc...", "kty": "RSA", "alg": "RS256", "n": "...", "e": "AQAB" },
    { "kid": "def...", "kty": "RSA", "alg": "RS256", "n": "...", "e": "AQAB" }
  ]
}
```

이걸 다룰 때 우리는 **항상 통째로 읽고, 항상 통째로 쓴다**는 점이 중요하다.

- 키 1번을 검증하든 키 2번을 검증하든, **JWKS 전체가 메모리에 있어야** `kid`로 매칭이 가능
- 카카오에서 갱신을 받아오면 JWKS 전체가 새 응답으로 교체됨
- "공개키 1개만 추가/수정"하는 시나리오는 **존재하지 않음** (그건 카카오가 할 일)

이런 데이터 모양이라면 굳이 Hash로 쪼개 저장할 이유가 없다. **JSON을 그대로 직렬화해서 String 1개에 담는 것이 가장 단순하고 빠르다.** Redis가 String에 어떤 의미도 부여하지 않기 때문에, JSON이든 바이너리 이미지든 똑같이 저장된다(binary-safe).

> **자료구조 선택의 일반 원칙**
> 데이터를 **항상 통째로** 다룬다면 -> **String**.
> **부분 단위로** 읽거나 쓴다면 -> **Hash**.
> "쪼개야 할 이유가 있는가?"를 먼저 묻는 것이 자료구조 선택에 있어 중요하다.

### 구현

Spring에서는 `@Cacheable` 어노테이션으로 캐싱 로직을 메서드에 끼워 넣을 수 있다. 캐시 저장소는 `RedisCacheManager`가 담당한다.

```java
// PublicKeyCacheConfig.java
@Bean
public CacheManager oidcCacheManager(RedisConnectionFactory connectionFactory) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
        .serializeKeysWith(
            RedisSerializationContext.SerializationPair.fromSerializer(
                new StringRedisSerializer()  // 키: 사람이 읽을 수 있는 문자열
            )
        )
        .serializeValuesWith(
            RedisSerializationContext.SerializationPair.fromSerializer(
                new GenericJackson2JsonRedisSerializer()  // 값: JSON 직렬화
            )
        )
        .entryTtl(Duration.ofDays(3));  // 3일 TTL

    return RedisCacheManager.builder(connectionFactory)
        .cacheDefaults(config)
        .build();
}
```

```java
// KakaoTokenClient.java
public interface KakaoTokenClient {

    @Cacheable(cacheNames = OIDC_PUBLIC_KEYS, cacheManager = "oidcCacheManager")
    @GetMapping("/.well-known/jwks.json")
    PublicKeysDto getKakaoPublicKeys();
    // 캐시 히트 시 실제 HTTP 호출 없이 Redis에서 반환
}
```

이렇게 해두면 첫 호출에서만 카카오에 HTTP 요청을 보내고, 이후 3일간은 Redis에서 꺼내 쓴다. 메서드 본문은 그대로 두고 어노테이션만으로 캐싱이 적용된다는 점이 `@Cacheable`의 강점이다.

### 실제로 저장되는 키와 값 뜯어보기

`@Cacheable`이 동작한 뒤 Redis에 어떤 모양으로 저장되는지 직접 확인해 보자.

#### 키 구조

키 이름은 Spring Cache가 두 부분을 `::` 구분자로 이어붙여 자동으로 생성한다.

```
oidc_public_keys :: SimpleKey []
       ①                 ②
```

- **① `oidc_public_keys`** — `@Cacheable(cacheNames = OIDC_PUBLIC_KEYS)`에 지정한 캐시 이름
  - 커스텀 `oidcCacheManager`는 `RedisCacheConfiguration.defaultCacheConfig()`를 직접 생성하므로, `application.yml`의 `spring.cache.redis.key-prefix` 설정이 적용되지 않아 **접두사가 없는 형태**가 됨
- **② `SimpleKey []`** — 캐시 키
  - `getKakaoPublicKeys()`는 **파라미터가 없는 메서드**이므로 Spring이 자동으로 "파라미터 없음"을 의미하는 `SimpleKey []`를 키로 사용
  - 파라미터가 있었다면 `SimpleKey [파라미터값]` 형태가 됨

#### 값 구조 — @class 타입 힌트의 정체

`GenericJackson2JsonRedisSerializer`는 역직렬화 시 어떤 클래스로 복원할지 알아야 하므로, 값을 저장할 때 **`@class` 필드를 JSON에 자동으로 포함**한다.

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

`@class`는 Spring이 Redis에서 꺼낸 JSON을 `PublicKeysDto` 객체로 복원할 때 사용하는 **타입 힌트**다. 개발자가 직접 작성하는 것이 아니라 직렬화 시 자동으로 삽입된다.

> **`JWK`의 `@class` 불확실성**
> `GenericJackson2JsonRedisSerializer`는 `NON_FINAL` 타이핑을 사용한다. Java `record`는 묵시적으로 `final`이므로 원칙적으로 `@class`가 붙지 않아야 하지만, `JWK`가 `Serializable`을 구현하고 있으면 Jackson 버전에 따라 `@class`가 포함될 수도 있다. 단, 실제 저장 값은 애플리케이션을 실행해 `GET` 명령어로 직접 확인하는 게 좋다.

#### 자료구조 타입 확인

```bash
TYPE "oidc_public_keys::SimpleKey []"
# -> string
```

겉으로 보면 JSON처럼 보여도 Redis 입장에서는 그냥 문자열 한 덩어리다.

### 실습 명령어

```bash
# 1. 캐시 키 찾기
KEYS "oidc_public_keys*"

# 2. 캐시 값 조회 (JSON 직렬화된 PublicKeysDto)
GET "oidc_public_keys::SimpleKey []"

# 3. TTL 남은 시간 확인
TTL "oidc_public_keys::SimpleKey []" # 약 259200 (3일) 또는 그 이하

# 4. 자료구조 타입 확인
TYPE "oidc_public_keys::SimpleKey []" # string

# 5. 캐시 무효화 (강제로 공개키 재조회 유도)
DEL "oidc_public_keys::SimpleKey []"

# 6. 캐시 수동 주입 (테스트용)
SET "oidc_public_keys::SimpleKey []" '{"keys":[{"kid":"abc","kty":"RSA","alg":"RS256","use":"sig","n":"modulus","e":"AQAB"}]}'
```

### TTL 3일을 정한 이유

카카오의 공개키 교체 주기를 감안한 값이다.

- **너무 짧으면**: 카카오에 자주 호출하게 됨 -> 캐싱 효과 감소
- **너무 길면**: 카카오가 키를 교체했는데 우리는 옛 키를 들고 있다가 검증 실패 -> 로그인 장애
- **3일**: 카카오의 일반적 키 교체 주기(수 주~수 개월)보다 넉넉히 짧으면서, 일상 운영에서는 캐시 히트율을 충분히 보장한다고 판단

키가 교체된 직후의 짧은 윈도우에서는 검증 실패가 발생할 수 있는데, 이 경우 클라이언트의 재시도 + 캐시 무효화(`DEL`) 로직으로 대응할 수 있다.

---

## 사례 2 · Refresh Token 저장 — Hash + @RedisHash

### 문제 상황

OAuth 2.0의 Refresh Token Rotation 패턴을 구현해야 한다.

- 로그인 성공 시 **Refresh Token을 서버에 저장** (재발급 시 검증용)
- `/token/refresh` 호출 시 클라이언트가 보낸 토큰과 서버 저장값을 비교 → 일치하면 새 토큰 발급 + **기존 토큰을 새 토큰으로 교체** (Rotation)
- 로그아웃 시 토큰 즉시 무효화
- TTL 14일 — 만료되면 사용자는 다시 로그인해야 함

이 데이터는 RDB에 저장해도 되지만, **TTL 자동 만료 + 빠른 조회 + 단순 키-값 구조**라는 특성을 보면 Redis로 훨씬 효율적으로 운용할 수 있다.

### 왜 Hash를 선택했는가

저장하려는 데이터는 다음과 같이 **여러 필드를 가진 객체** 형태다.

```
RefreshTokenEntity
├─ memberId: "12345"
└─ refreshToken: "eyJhbGciOi..."
```

이걸 String 1개로 다루려면 두 가지 선택지가 있는데 둘 다 단점이 있다.

| 방법                                                           | 단점                                           |
| -------------------------------------------------------------- | ---------------------------------------------- |
| `SET refresh:12345 "eyJhbGc..."` (값에 토큰만)                 | memberId가 키에 묻혀 데이터로 다루기 불편      |
| `SET refresh:12345 '{"memberId":..,"refreshToken":..}'` (JSON) | 토큰만 갱신해도 JSON 통째로 다시 직렬화 + 쓰기 |

반면 Hash를 쓰면:

```
RefreshTokenEntity:12345
├─ memberId      : "12345"
└─ refreshToken  : "eyJhbGc..."
```

- `HGET RefreshTokenEntity:12345 refreshToken` — **토큰만** 꺼낼 수 있음 (검증용)
- `HSET RefreshTokenEntity:12345 refreshToken <new>` — **토큰만** 갱신 (Rotation)
- 필드 추가 시 스키마 변경 없음 (예: `lastIssuedAt` 같은 메타데이터 추가)

**필드 단위 읽기/쓰기가 자주 일어나는 객체**에는 Hash가 가장 적합하다. JWKS 캐시와 정반대 케이스다.

### 구현

Spring Data Redis의 `@RedisHash`를 사용하면 JPA처럼 객체 단위로 다룰 수 있다. 내부적으로는 Hash 자료구조로 매핑된다.

```java
@RedisHash(value = "refresh_token", timeToLive = 1209600)  // 14일
@RequiredArgsConstructor
@Getter
public class RefreshTokenEntity {

    @Id
    private final String memberId;

    private final String refreshToken;
}
```

```java
public interface RefreshTokenRepository extends CrudRepository<RefreshTokenEntity, String> {
}
```

이렇게만 하면 `repository.save(entity)` / `repository.findById(memberId)` / `repository.deleteById(memberId)` 가 그대로 동작한다.

### @RedisHash가 만드는 두 가지 키 — Hash + Set

`@RedisHash`는 이름과 달리 **Hash 하나만 만드는 게 아니라**, 항상 **Hash + Set 두 개**를 함께 만든다.

#### 첫 번째 키 — Hash (실제 데이터)

```bash
HGETALL "RefreshTokenEntity:12345"
# 1) "memberId"
# 2) "12345"
# 3) "refreshToken"
# 4) "eyJhbGciOiJIUzI1NiJ9..."
# 5) "_class"
# 6) "...RefreshTokenEntity"
```

엔티티의 각 필드가 Hash의 필드-값으로 저장된다. `_class`는 역직렬화용 타입 힌트(JWKS 사례의 `@class`와 같은 역할).

#### 두 번째 키 — Set (인덱스)

```bash
TYPE "refresh_token" # set

SMEMBERS "refresh_token"
# 1) "1"
# 2) "12345"
# 3) "99999"
```

`@RedisHash(value = "refresh_token", ...)`의 `value` 이름으로 Set이 하나 더 만들어진다. 여기에는 **저장된 모든 엔티티의 ID 목록**이 담긴다.

#### 왜 Set이 필요한가 — findAll()의 정체

JPA의 `findAll()`을 호출했다고 생각해 보자. Redis는 키-값 저장소이므로, **"이 클래스에 속한 모든 엔티티 ID"를 알아낼 방법이 없다.** 더군다나 `KEYS RefreshTokenEntity:*` 같은 명령은 운영 환경에서 쓸 수 없다(블로킹 위험).

그래서 Spring Data Redis는 **저장 시점에 ID를 따로 모아 두는 인덱스 Set을 만든다**.

```
findAll() 호출
  → SMEMBERS "refresh_token"           (Set에서 ID 목록 조회)
  → HGETALL "refresh_token:1",
    HGETALL "refresh_token:12345",
    HGETALL "refresh_token:99999",
    ...                                 (각 Hash 조회)
```

#### 검증 — RedisKeyValueAdapter.put()의 소스

직접 Spring 소스에서 확인할 수 있다.

```
spring-projects/spring-data-redis
  → src/main/java/org/springframework/data/redis/core/
    → RedisKeyValueAdapter.java
      → put() 메서드:

        // 1. 엔티티 저장 (Hash)
        redisOps.execute(...HSET {keyspace}:{id} ...)

        // 2. 인덱스 Set에 ID 추가 (Set)
        redisOps.boundSetOps(keyspace).add(id)
        // → SADD "refresh_token" "1"
```

대충 소스를 보면 `save()` 한 번 호출이 내부적으로 **HSET + SADD 두 번의 명령**으로 실행된다는 의미라는 걸 알 수 있다.

> **이 사례가 자료구조 활용의 좋은 예인 이유**
> 한 비즈니스 요구(RefreshToken을 저장하고 ID로 전부 조회)를 만족시키기 위해 **두 가지 자료구조가**쓰인다. Hash는 객체 표현, Set은 ID 인덱스. 이 조합이 자연스러운 이유는 각자의 강점이 서로 보완적이기 때문이다.
> Hash는 객체 모양 보존에 강하지만 전체 순회가 어렵고, Set은 유일성인데다 O(1) 멤버십 검사에 강하지만 객체 정보를 담을 수 없다.

### Refresh Token Rotation 시뮬레이션

실제 인증 플로우에서 일어나는 일을 redis-cli로 따라가 보자.

```bash
# 1. 로그인 성공 → 토큰 저장
HSET "RefreshTokenEntity:1" memberId "1" refreshToken "token_v1"
EXPIRE "RefreshTokenEntity:1" 1209600 # (Spring Data Redis는 SADD "refresh_token" "1" 도 함께 실행)

# 저장 확인
HGETALL "RefreshTokenEntity:1"
SMEMBERS "refresh_token"
# 1) "1"


#  2. /token/refresh 호출 → 검증
HGET "RefreshTokenEntity:1" refreshToken # "token_v1" 클라이언트가 보낸 토큰과 비교


# 3. 검증 통과 → 새 토큰으로 교체 (Rotation)
HSET "RefreshTokenEntity:1" refreshToken "token_v2" # 다른 필드(memberId)는 건드리지 않음 -> Hash의 필드 단를위 갱신했을 때 장점

HGET "RefreshTokenEntity:1" refreshToken # "token_v2" 갱신 확인


# 4. 로그아웃 → 토큰 무효화
DEL "RefreshTokenEntity:1" # Spring Data Redis는 SREM "refresh_token" "1" 도 함께 실행


# 5. 로그아웃 후 재발급 시도 -> 차단됨
EXISTS "RefreshTokenEntity:1" # 0 -> 키 없음. 애플리케이션은 JWT_REFRESH_NOT_FOUND_IN_REDIS 예외 던짐
```

`HGET`으로 **토큰 필드만** 검증하고, `HSET`으로 **토큰 필드만** 교체하는 흐름이 핵심이다. Hash가 아니었다면 매번 객체 전체를 직렬화/역직렬화해야 했을 것이다.

### TTL 14일을 정한 이유

- **사용자 경험** — 14일 동안은 다시 로그인하지 않고도 자동 재인증 가능
- **보안** — 너무 길면 토큰 탈취 시 피해 기간이 길어짐. 14일이 일반적인 절충선
- **자동 정리** — TTL이 있으니 별도 cleanup batch가 필요 없음. 만료된 키는 Redis가 알아서 제거

```bash
TTL "RefreshTokenEntity:1" # 1209598 (저장 직후 거의 1209600 = 14일)
```

> **TTL 갱신 시 주의**
> `HSET`만으로는 TTL이 갱신되지 않는다. `@RedisHash`의 `timeToLive`는 **새로 save()가 호출될 때만** 다시 14일로 세팅된다. Rotation 시점에 TTL을 같이 늘리고 싶다면 명시적으로 `EXPIRE`를 다시 호출하거나 save()로 객체 전체를 다시 저장해야 한다.

---

## 전체 인증 플로우에서 두 캐시가 상호작용하는 과정

두 캐시는 따로 노는 게 아니라, 한 번의 로그인 요청 안에서 함께 동작한다.

| 단계 | 시나리오                                | 사용 자료구조             | 핵심 명령어                            |
| ---- | --------------------------------------- | ------------------------- | -------------------------------------- |
| 1    | 사용자 로그인 → 카카오에서 ID 토큰 받음 | -                         | -                                      |
| 2    | ID 토큰 검증을 위해 공개키 필요         | **String** (사례 1)       | `GET "oidc_public_keys::SimpleKey []"` |
| 3    | 캐시 미스라면 카카오 호출 후 저장       | **String** (사례 1)       | `SET ...` + `EXPIRE ... 259200`        |
| 4    | 검증 통과 -> Refresh Token 저장         | **Hash** (사례 2)         | `HSET "RefreshTokenEntity:..." ...`    |
| 5    | 동시에 인덱스 Set에도 ID 추가           | **Set** (사례 2의 부산물) | `SADD "refresh_token" "..."` (자동)    |
| 6    | 이후 `/token/refresh` 호출 시 토큰 검증 | **Hash** (사례 2)         | `HGET ... refreshToken`                |
| 7    | Rotation — 토큰 필드만 교체             | **Hash** (사례 2)         | `HSET ... refreshToken <new>`          |
| 8    | 로그아웃                                | **Hash** (사례 2)         | `DEL ...` + `SREM ...` (자동)          |

이 과정에서 **세 가지 자료구조(String / Hash / Set)** 가 모두 다른 역할로 등장한다.

### redis-cli 실습

```bash
# Step 1 · OIDC 공개키 캐시 (TTL 3일)
SET "oidc_public_keys::SimpleKey []" '{"keys":[...]}'
EXPIRE "oidc_public_keys::SimpleKey []" 259200

# Step 2 · 로그인 성공 → Refresh Token 저장
HSET "RefreshTokenEntity:member_1" memberId "member_1" refreshToken "refresh_abc123"
EXPIRE "RefreshTokenEntity:member_1" 1209600

# Step 3 · /token/refresh 호출
HGETALL "RefreshTokenEntity:member_1"             # 존재 확인
HGET "RefreshTokenEntity:member_1" refreshToken   # "refresh_abc123" 검증

# 검증 통과 -> 새 토큰 발급 -> 교체 (Rotation)
HSET "RefreshTokenEntity:member_1" refreshToken "refresh_xyz789"
EXPIRE "RefreshTokenEntity:member_1" 1209600      # TTL 다시 14일

# Step 4 · 로그아웃
DEL "RefreshTokenEntity:member_1"

# Step 5 · 로그아웃 후 재발급 시도 -> 차단
EXISTS "RefreshTokenEntity:member_1" # 0 -> JWT_REFRESH_NOT_FOUND_IN_REDIS 예외
```

---

## 캐시 사용 시 주의사항

OIDC 인증과 Refresh Token 재발급을 거치며 마주칠 수 있는 고민거리는 아래와 같다.

### 1. 캐시 무효화 (Cache Invalidation)

첫번째는 **원본이 바뀌었는데 사본이 그대로 남아 있는 상황**을 어떻게 처리할 것인가다.

- 사례 1(JWKS): 카카오가 키를 교체했는데 우리는 옛 키를 들고 있음 -> 검증 실패 발생 시 `DEL`로 캐시 무효화 후 재시도
- 사례 2(RefreshToken): 원본이 곧 Redis 자체이므로 무효화 이슈 없음

### 2. TTL 갱신 누락

`HSET`은 TTL을 건드리지 않는다. 그런데 만약 Refresh Token Rotation에서 토큰만 갱신하고 EXPIRE를 깜빡하면, **새 토큰의 만료 시각이 옛 토큰의 만료 시각을 따라가게 된다**. Rotation 후 14일이 더 늘어나야 한다면 `EXPIRE`를 명시적으로 다시 호출해야 한다.

### 3. 자료구조 타입 검증

```bash
TYPE "RefreshTokenEntity:1" # hash

TYPE "oidc_public_keys::SimpleKey []" # string

TYPE "refresh_token" # set -> 직접 만든 적 없는데 존재함 (인덱스 Set)
```

`refresh_token`이라는 이름의 Set이 있는 것을 보고 "내가 만든 게 아닌데?" 하고 당황할 수 있는데, 이는 `@RedisHash`가 자동으로 만든 인덱스다. **직접 SADD/SREM 하지 말자**. Spring Data Redis의 일관성이 깨질 수 있다.

### 4. 운영에서는 KEYS 금지

캐시 키 목록을 보고 싶다고 무작정 `KEYS *`를 입력하면 안 된다.

```bash
# 절대 금지 (블로킹)
KEYS "RefreshTokenEntity:*"

# 안전하게 조회
SCAN 0 MATCH "RefreshTokenEntity:*" COUNT 100
```

레디스는 싱글 스레드이므로 KEYS 명령이 끝날 때까지 다른 모든 요청이 대기하기 때문이다.

### 5. 대용량 키 삭제는 UNLINK

인덱스 Set이 커진 상태(예: 활성 사용자 100만명)에서 `DEL refresh_token`을 치면 동기 삭제로 메인 스레드가 점유된다. 이런 경우는 `UNLINK`로 비동기 삭제하거나 `lazyfree-lazy-user-del = yes` 설정을 활용하면 된다.

## 출처

- 김가림, 『**개발자를 위한 레디스**』 — 4장 자료구조 활용 사례
- https://github.com/f-lab-edu/cyber-monday
