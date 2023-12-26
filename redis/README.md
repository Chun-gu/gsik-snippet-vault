# Redis

## 목차

[캐싱 솔루션으로서의 레디스](#캐싱-솔루션으로서의-레디스)  
[데이터 타입 활용](#데이터-타입-활용)  
[데이터 영구 저장 방식](#데이터-영구-저장-방식)  
[아키텍처](#아키텍처)  
[운영 꿀팁 & 장애 포인트](#운영-꿀팁--장애-포인트)

## 캐싱 솔루션으로서의 레디스

### 캐싱이란

<image src="./images/chacing.png"></image>

데이터의 원래 소스보다 더 빠르고 효율적으로 액세스 할 수 있는 임시 데이터 저장소

- 캐싱 사용이 효율적인 케이스
  - 같은 데이터에 반복적으로 액세스하는 경우
  - 변하지 않는 데이터일 경우

### Redis가 캐싱 솔루션으로 좋은 이유

- 단순한 key-value 구조
- In-memory 데이터 저장소(RAM)
- 빠른 성능
  - 평균 작업속도 < 1 ms
  - 초당 수백만 건의 작업 가능

=> 지연 시간 감소, 처리량 증가

### 캐싱 전략

- 캐시 운용 방식에 따라 성능이 크게 좌우될 수 있음
- 데이터의 유형과 해당 데이터에 대한 액세스 패턴을 잘 고려해서 선택해야 함

#### 읽기 전략 - Look-Aside(Lazy Loading)

- 애플리케이션에서 데이터를 읽는 작업이 많을 때 사용하는 전략
- 레디스를 cache로 쓸 때 가장 일반적으로 사용

<div>
  <table>
    <tr>
      <td style="padding:0;">
        <image src="./images/look-aside-1.png" style="display:block;height:300px"></image>
      </td>
      <td style="padding:0;">
        <image src="./images/look-aside-2.png" style="display:block;height:300px"></image>
      </td>
    </tr>
  </table>
</div>

1. 애플리케이션은 데이터를 찾을 때 cache 먼저 확인
2. cache에 데이터가 있으면 cache에서 데이터를 가지고 오는 작업을 반복
3. 만약 찾는 데이터의 키가 레디스에 없다면 애플리케이션은 DB에 접근해서 데이터를 직접 가지고 온 뒤 다시 레디스에 저장

- 찾는 데이터가 없을 때에만 cache에 데이터가 저장되기 때문에 lazy loading이라고도 함.

- 장점

  - 레디스가 다운되더라도 바로 장애로 이어지지 않고 DB에서 데이터를 가지고 올 수 있음

- 단점
  - cache로 붙어있던 커넥션이 많이 있었다면 그 커넥션이 모두 데이터베이스로 붙기 때문에 DB에 갑자기 많은 부하가 몰릴 수 있음
  - cache가 새로 투입되었거나 DB에만 새로운 데이터를 저장했다면 처음에 캐시 미스가 많이 발생해서 성능에 저하가 올 수 있음  
    => 해결책 - cache warming: 미리 DB에서 cache로 데이터를 밀어 넣어주는 작업

#### 쓰기 전략

<image src="./images/writing-strategies.png" style="width:100%"></image>

##### Write-Around

- DB에만 데이터를 저장하는 방법
- 모든 데이터는 DB에 저장되고 cache miss가 발생한 경우에만 cache에 데이터를 끌어옴
- 단점
  - cahce 내의 데이터와 DB 내의 데이터가 다를 수 있음

##### Write-Through

- DB에 데이터를 저장할 때 cache에도 함께 저장하는 방법
- 장점
  - cache는 항상 최신 정보를 가지고 있음
- 단점
  - 저장할 때마다 두 단계 스텝을 거쳐야 하기 때문에 상대적으로 느림
  - 저장하는 데이터가 재사용되지 않을 수도 있는데 무조건 캐시에 넣어버리기 때문에 일종의 리소스 낭비
  - 몇 분 혹은 몇 시간 동안만 데이터를 보관하겠다는 의미인 expire time을 설정해 주는 것이 좋음
  - expire time 값의 관리를 어떻게 하는지가 장애 포인트가 될 수도 있음

## 데이터 타입 활용

- 레디스는 자체적으로 많은 자료구조를 제공
- 상황에 따라 자료구조들을 효율적으로 사용해야 함

### 데이터 타입 종류

<image src="./images/redis-data-types.png" style="width:100%"></image>

#### Strings

- 제일 기본적인 데이터 타입
- set 커맨드를 이용해 저장되는 데이터는 모두 string 형태

#### Bitmaps

- string의 변형이라고 볼 수 있고 bit 단위 연산 가능

#### Lists

- 데이터를 순서대로 저장하므로 큐로 사용하기 적절

#### Hashes

- 하나의 키 안에 또다시 여러 개의 필드와 밸류 쌍으로 데이터를 저장

#### Sets

- 중복되지 않은 문자열의 집합

#### Sorted Sets

- set처럼 중복되지 않은 값을 저장하지만 모든 값은 score라는 숫자 값으로 정렬
- 데이터가 저장될 때부터 score 순으로 정렬되며, 만약 score가 같을 때에는 사전 순으로 정렬되어 저장

#### HyperLogLogs

- 굉장히 많은 데이터를 다룰 때 주로 사용
- 중복되지 않는 값의 개수를 카운트할 때 사용

#### Streams

- log를 저장하기 가장 좋은 자료구조

### Best Practice

레디스의 자료구조를 사용하기 좋은 사례

#### Counting

##### Strings

- 단순 증감 연산
- INCR / INCRBY / INCRBYFLOAT / HINCRBY / HINCRBYFLOAT / ZINCRBY
- 예시  
  <image src="./images/string-count.png"></image>
  - `score:a`라는 `key`에 `SET`으로 `10` 저장
  - `INCR` 함수로 `1` 증가시켜 `11`
  - `INCRBY` 함수로 증가량을 `4`로 지정하면 `15`

##### Bits

- 데이터 저장공간 절약
- 정수로 된 데이터만 카운팅 가능

- 예시  
  <image src="./images/bit-counting.png"></image>

  - 특정 날짜(20210817)에 접속한 유저 수를 세고 싶은 경우, 날짜로 key 하나를 만들어놓고 유저 ID에 해당하는 bit를 1로 올려주는 방식
  - 한 개의 bit로 한 명의 유저를 표현하면 유저가 천만이어도 천만 개의 bit로 표현 가능하며 1.2 메가 바이트 정도밖에 차지하지 않음

- `SETBIT`으로 bit 설정
- `BITCOUNT`로 1로 설정된 값 전부 카운팅
- 단점
  - 모든 데이터를 정수로 표현할 수 있어야 사용 가능  
    => 유저 ID 같은 값이 0 이상의 정수값일 때만 카운팅 가능하며 이처럼 시퀀셜한 값이 없을 때는 사용 불가

##### HyperLogLogs

- 모든 string 데이터 값을 유니크하게 구분 가능
- set과 비슷하지만 대량의 데이터 카운팅에 훨씬 적절(오차 0.81%)
- 저장되는 데이터 개수가 몇백만, 몇천만이든 상관없이 모든 값이 `12KB`로 고정되어 저장됨
- 한번 저장된 데이터 다시 불러와서 확인할 수 없음  
  => 경우에 따라 데이터를 보호하기 위한 목적으로도 적절하게 사용 가능
- 크고 unique한 값의 계산에 적절
  - 웹사이트에 방문한 유니크한 IP의 개수 카운팅
  - 하루 종일 크롤링한 URL의 개수 카운팅
  - 검색 엔진에서 검색된 유니크한 단어의 개수 카운팅
- 예시  
  <image src="./images/hll-counting.png"></image>
  - `PFADD`: 데이터 저장
  - `PFCOUNT`: 유니크하게 저장된 값 조회
  - `PFMERGE`: key들을 머지해서 확인  
    ex) 일별로 저장한 데이터의 일주일 치를 취합해서 보고 싶은 경우

#### Messaging

##### Lists

- 자체적으로 제공하는 blocking 기능으로 불필요한 polling 프로세스 방지하는 이벤트 큐로 사용하기 적합

- 예시  
  <image src="./images/list-messaging.png"></image>

  - `Client A`가 `BRPOP`으로 `myqueue`에서 데이터를 꺼내려는데 현재 리스트 안에 데이터가 없어 대기하는 상황
  - 이때 `Client B`가 `hi`라는 값을 넣어주면 `client A`에서 바로 확인 가능

- `LPUSHX` / `RPUSHX`: 키가 있을 때만 리스트에 데이터를 저장하는 커맨드

  - key가 이미 있다는 건 예전에 사용했던 queue라는 뜻
  - 사용했던 queue에만 메시지를 저장할 수 있으므로 비효율적인 데이터의 이동 방지 가능
  - 이미 캐싱되어 있는 피드에만 신규 트윗 저장에 사용

- 예시
  <div style="display:flex;">
    <image src="./images/list-messaging-1.png"></image>
    <image src="./images/list-messaging-2.png"></image>
  </div>

  - 인스타그램, 페이스북, 트위터 등 SNS에는 각 유저별로 타임라인이 존재하고 타임라인에는 팔로우한 사람들의 데이터만 노출됨
  - 트위터는 각 유저의 타임라인에 보일 트윗을 캐싱하기 위해 레디스의 리스트를 사용하며 이때 `RPUSHX` 커맨드 사용
  - 이로 인해 트위터를 자주 이용하던 유저의 타임라인에만 새로운 데이터를 미리 캐시해 놓을 수 있음
  - 자주 사용하지 않는 유저는 caching key 자체가 존재하지 않으므로 해당 유저들을 위해 데이터를 미리 쌓아놓는 등의 비효율적인 작업 방지 가능

##### Streams

- 로그 저장에 가장 적절한 자료구조
- append-only: 서버에 로그가 쌓이듯 데이터의 추가만 가능하므로 중간에 데이터가 바뀌지 않음
- 시간 범위로 검색 / 신규 추가 데이터 수신 / 소비자별 다른 데이터 수신(소비자 그룹)
- 예시  
  <image src="./images/streams-messaging.png"></image>

  - `XADD` 커맨드로 `mystream` 키에 데이터 저장
  - `*`: ID를 의미함
    - ID 직접 지정 가능
    - 일반적으로 `*` 입력 시 레디스가 알아서 저장한 뒤 ID 값 반환
    - 반환된 ID 값은 데이터가 저장된 시간을 의미
  - 해시처럼 key-value 쌍으로 데이터 저장
    - `sensor-id`의 값으로 `1234`, `temperature`의 값으로 `19.8` 저장

- Streams의 데이터 읽어오는 방법

  - ID 값으로 시간 대역대로 저장된 값 검색
  - 실제 서버에서 로그를 읽을 때 사용하는 `tail -f`처럼 새로 들어오는 데이터만 리스닝 가능

- 소비자 그룹: 카프카처럼 원하는 소비자만 특정 데이터를 읽게 할 수 있음
- 카프카의 개념을 많이 차용
- 메세징 브로커가 필요할 때 카프카의 대체재로 간단하게 사용 가능한 자료구조([공식 문서](https://redis.io/docs/data-types/streams/))

## 데이터 영구 저장 방식

- 레디스는 In-memory 데이터 스토어
- 모든 데이터가 메모리에 저장되므로 서버나 레디스 인스턴스가 재시작되면 모든 데이터 유실
- 복제 기능을 사용해도 코드상의 버그 또는 휴먼 에러 발생 시 복제본에 똑같이 적용되므로 데이터 복원 불가
- 레디스를 캐시로만 사용한다면 백업할 필요 없음
- 캐시 이외의 용도로 사용한다면 적절한 데이터 백업 필요

### AOF - Append Only File

<image src="./images/aof.png"></image>

- 데이터를 변경하는 커맨드가 들어오면 커맨드를 그대로 모두 저장
- `key1`에 `a` 저장 후 `apple`로 변경, `key2`에 `b` 저장 후 삭제한 커맨드 전부 저장
- Append Only: 데이터가 추가되기만 해서 대부분 RDB 파일보다 커지므로 주기적인 압축 및 재작성 과정 필요
- 레디스 프로토콜 형태로 저장되기 때문에 우리가 읽을 수 없음(이미지는 예시)

#### 저장 방법

- 자동: `redis.conf` 파일에서 `auto-aof-rewrite-percentage` 옵션 설정
  - 파일 크기 기준으로 압축 시점 지정
- 수동: CLI 창에서 `BGREWRITEAOF` 커맨드로 AOF 파일 수동 재작성

#### 적합한 케이스

- 장애 상황 직전까지의 모든 데이터가 보장되어야 할 경우
  - `appendonly`: yes
  - `APPENDFSYNC` 옵션이 `everysec`(기본값)인 경우 최대 1초 사이의 데이터 유실 가능
- 강력한 내구성이 필요한 경우 RDB, AOF 동시 사용([공식 문서](https://redis.io/docs/management/persistence/#ok-so-what-should-i-use))

### RDB - Redis Database

<image src="./images/rdb.png"></image>

- 스냅샷 방식: 저장 당시의 메모리에 있는 데이터 그대로 사진 찍듯 파일로 저장
- `key1`에 `apple`이 저장된 데이터만 남아있음
- 바이너리 파일 형태로 저장되기 때문에 우리가 읽을 수 없음(이미지는 예시)

#### 저장 방법

- 자동: `redis.conf` 파일에서 `SAVE` 옵션 설정
  - 시간 기준으로 저장 시점 지정
- 수동: CLI 창에서 `BGSAVE` 커맨드로 RDB 파일 수동 저장
  - `SAVE` 커맨드 절대 사용 금지

#### 적합한 케이스

- 백업은 필요하지만 어느 정도의 데이터 손실이 발생해도 괜찮은 경우
  - `redis.conf` 파일에서 `SAVE` 옵션 적절히 사용  
    ex) `SAVE 900 1`: 900초 동안 한 개 이상의 키 변경 시 RDB 파일 재작성
- 강력한 내구성이 필요한 경우 RDB, AOF 동시 사용([공식 문서](https://redis.io/docs/management/persistence/#ok-so-what-should-i-use))

persistency 기능을 이용해서 저장된 AOF / RDB 파일은 재시작을 통해서만 복원할 수 있습니다. 이 때 redis.conf 에 지정된 'dir' 경로에 백업파일을 저장해두어야 하며, 'CONFIG GET dir' 커맨드를 이용해서도 디렉토리 경로를 확인할 수 있습니다.

## 아키텍처

### Replication

<image src="./images/replication.png"></image>

마스터와 레플리카만 존재하는 간단한 구조

#### 특성

- 단순히 복제만 연결된 상태
- `replicaof` 커맨드로 간단하게 복제 연결
- 비동기식 복제: 마스터는 레플리카로의 데이터 전달의 성공 실패 여부를 확인하거나 기다리지 않음
- HA(High Availability) 기능이 없으므로 장애 상황 발생 시 수동 복구 필요

  - `replicaof no one`: 리플리카 노드에 직접 접속해서 복제를 끊어야 함
  - 애플리케이션에서 연결 정보 변경 후 배포 필요

### Sentinel

<image src="./images/sentinel.png"></image>

마스터와 레플리카 노드 외에 일반 노드들을 모니터링하는 센티널 노드로 구성

#### 특성

- 자동 페일오버 가능한 HA 구성

  - 센티널 노드가 다른 노드 감시
  - 마스터가 비정상 상태일 시 자동으로 페일오버: 레플리카 노드가 마스터가 됨
  - 연결 정보 변경 필요 없음: 애플리케이션에서는 센티널 노드만 알고 있으면 되며 센티널이 변경된 마스터 정보로 바로 연결시켜줌
  - 센티널 노드는 항상 3대 이상의 홀수로 존재해야 함
    - 과반수 이상의 센티널이 동의해야 페일오버 진행

- NHN의 아키텍처  
  <image src="./images/nhn.png"></image>  
  두 대의 서버에는 일반 레디스와 센티널을 함께,  
  최저 사양의 다른 서버에는 센티널 노드만 올려 사용 중

### Cluster

<image src="./images/cluster.png"></image>

#### 특성

- 스케일 아웃과 HA 구성
  - 샤딩: 키를 여러 노드에 자동으로 분할해서 저장
  - 모든 노드가 서로를 감시하며 마스터가 비정상 상태일 때 자동 페일오버
  - 최소 3대의 마스터 노드가 필요하며 레플리카 노드를 하나씩 추가하는 게 일반적인 구성

### 아키텍처 선택 기준

<image src="./images/decision.png"></image>

- HA 기능: 자동 페일오버
- 샤딩 기능: 서비스 확장을 위한 스케일 아웃
- Stand-Alone: 마스터 하나만 띄운 구조

## 운영 꿀팁 & 장애 포인트

### 사용하면 안되는 커맨드

#### 레디스는 싱글 스레드

오래 걸리는 커맨드 실행 하나로 나머지 모든 요청들의 수행 불가 및 대기를 유발하며 이로 인한 빈번한 장애 발생 가능

<image src="./images/forbidden-command.png"></image>

- `keys` => `scan`

  - 모든 키를 보여주는 `key`보다 재귀적으로 키를 호출하는 `scan` 사용

- `Hash`나 `Sorted Set` 등의 자료구조
  - 키 나누기(최대 100만개)
    - 키 내부에 여러 개의 아이템을 저장할 수 있으며 아이템이 많아질수록 성능 저하
    - 하나의 키에 최대 100만 개 이상은 저장하지 않도록 키를 적절하게 분리
  - `hgetall` => `hscan`
  - `del` => `unlink`
    - 키에 많은 데이터가 들어있을 때 `del` 사용 시 해당 키를 지우는 동안 아무런 동작을 할 수 없으므로 백그라운드에서 지워주는 `unlink` 사용

### 변경하면 장애를 막을 수 있는 기본 설정값

#### STOP-WRITES-ON-BGSAVE-ERROR = NO

- `yes`(기본값): RDB 파일 저장 실패 시 레디스로 들어오는 모든 write를 차단
- 레디스 서버에 대한 모니터링을 적절히 하고 있다면 이 기능을 꺼두는 게 오히려 불필요한 장애를 막을 수 있음

#### MAXMEMORY-POLICY = ALLKEYS-LRU

- 레디스를 캐시로 사용 시 키에 대한 Expire Time 설정 권장
  - 미설정 시 금세 레디스의 MAXMEMORY까지 데이터가 가득 참
- 메모리가 가득 차면 `MAXMEMORY-POLICY`에 의해 키가 관리됨
  - `noeviction`(기본값)
    - 더 이상 새로운 키를 저장하지 않음.
    - 새로운 데이터 입력이 불가능해지므로 장애 발생
  - `volatile-lru`
    - lru(least recently used) 알고리즘에 따라 가장 오랫동안 사용되지 않은 키 삭제
    - expire time이 설정된 키만 삭제하므로 설정되지 않은 키만 남아 있다면 장애 발생
  - `allkeys-lru`
    - 모든 키를 lru 알고리즘에 따라 삭제
    - 메모리 가득 참으로 인한 장애 발생 가능성 없음

### Cache Stampede

#### TTL 값을 너무 작게 설정한 경우

- 대규모의 트래픽 환경에서 TTL 값을 너무 작게 설정한 경우 cache stampede 현상 발생 가능

<image src="./images/cache-stampede.png"></image>

- `Look-Aside` 패턴: 레디스에 데이터가 없다는 응답을 받은 애플리케이션이 직접 DB로 데이터 요청한 뒤 이를 다시 레디스에 저장
- 키가 만료되는 순간, 이 키를 보고 있던 다수의 애플리케이션에 의해 duplicate read, duplicate write 발생
  - duplicate read: 다수의 애플리케이션이 DB로부터 같은 데이터를 read
  - duplicate write: DB로부터 읽어온 데이터를 레디스에 각각 write
- 굉장히 비효율적인 상황이며 처리량과 불필요한 작업이 늘어나 장애로 이어질 수 있음
- 실제 사례
  - 티켓링크의 경우 인기 있는 공연 오픈시 하나의 공연 데이터를 읽기 위해 몇십 개의 애플리케이션 서버에서 커넥션 연결
  - 부하 발생 시점의 상황을 자세히 분석하기 위해 직접 개발 쪽의 스카우터 로그를 확인했고 원인을 파악해서 TTL 시간을 넉넉하게 늘리는 것으로 해결

### MaxMemory 값 설정

#### Persistence / 복제 사용시 MaxMemory 설정 주의

<image src="./images/copy-on-write.png"></image>

- RDB 저장 또는 AOF rewrite 시 `fork()`로 자식 프로세스 생성
  - 부모 프로세스: 일반적인 요청을 받아 데이터 처리 계속
  - 자식 프로세스: 백그라운드에서 데이터를 파일로 저장  
    => 메모리를 복사해서 사용하는 Copy-on-Write로 인해 가능
- Copy-on-Write로 인해 메모리를 두배로 사용하는 경우 발생 가능
- Persistence 또는 복제 사용 시 MaxMemory는 실제 메모리의 절반으로 설정  
  예) 4GB => 2048MB
  - 복제 연결을 처음 시도하거나, 혹은 연결이 끊겨 재시도를 할 때에 새로 RDB 파일을 저장
  - MaxMemory 값은 실제 메모리의 절반 정도 설정 권장
  - 예상치 못한 상황에 메모리가 가득 차서 장애 발생 가능성 매우 높음

### Memory 관리

레디스는 메모리를 사용하는 저장소이므로 운영에서 메모리 관리가 제일 중요

#### 물리적으로 사용되고 있는 메모리를 모니터링

모니터링 시 `used_memory` 값보다 `used_memory_rss`(Resident Set Size) 값이 더 중요

- `used_memory`: 논리적으로 레디스가 사용하는 메모리
- `used_memory_rss`: OS가 레디스에 할당한 물리적 메모리 양
- 실제 저장된 데이터는 적은데 rss 값은 큰 경우 등 차이가 클 때 fragmentation이 크다고 함
  - 주로 삭제되는 키가 많아지면 fragmentation 증가
    - 특정 시점에 피크를 찍고 다시 삭제되는 경우
    - TTL로 인한 eviction(축출)이 많이 발생하는 경우  
      <image src="./images/memory-usage.png"></image>
      - 초록색 used 그래프는 폭락했지만 노란색 rss 그래프는 아직 많이 차 있음
    - `CONFIG SET activedefrag yes`
      - `activedefrag`라는 기능을 잠시 defragmentation을 해줘서 도움 됨. 이 값을 항상 켜두기 보다 단편화가 많이 발생했을 때 켜두는 것을 권장([공식 문서 - ACTIVE DEFRAGMENTATION](https://redis.io/docs/management/config-file/) )

## 레퍼런스

- [[NHN FORWARD 2021] Redis 야무지게 사용하기](https://www.youtube.com/watch?v=92NizoBL4uA)
- [ElastiCache 운영을 위한 우아한 가이드: 초고속 메모리 분석 툴 개발기와 레디스 운영 노하우 소개](https://www.youtube.com/watch?v=JH07ABaRPWo&t=593s)(추가 작성 예정)

[⬆️ 위로 이동](#redis)
