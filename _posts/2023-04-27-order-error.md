---
title: 사람인 결제 서버 과부하 이슈 공유
author: 최재우
tags:
- ''
- hikaricp
- gc
---

안녕하세요. 

IT연구소 서비스개발팀 빌링파트 최재우입니다.

**2022년 12월 7일** 사람인의 결제 및 상품을 담당하는 `Order API` 서버의 `CPU`, `Memory`과부하가 지속되는 이슈가 발생했었습니다. 해당 이슈를 통해 발생한 에러들과 원인이 무엇인지 분석한 것에 대해 공유해 드리고자 합니다.

# 원인이 무엇이었나

---

저희 `Order API`는 **Spring Boot 2.X**, **JPA(hibernate), Mysql(HikariCP)**을 사용 중입니다.

먼저 결론부터 말씀드리면 이번 이슈의 원인은 다음과 같습니다.

- 세금 계산서 관련 `Entity`를 예상치 못한 `ORM(JPA)` 쿼리로 인해 **약 46만건** 한 번에 호출
- 해당 쿼리로 조회한 `Entity`가 `Heap Memory`를 모두 차지하여 `Full GC`가 계속 수행되는 상황에서 새로운 요청까지 계속 유입되어 서버 재실행 전까지 `CPU`, `Memory` 과부하

이슈 발생 시점 `CPU`, `Memory` 사용량은 다음과 같습니다.

![Untitled](/img/order-error/Untitled.png)

![Untitled](/img/order-error/Untitled%201.png)

지금 보니 네트워크 사용량이 왜 눈에 들어오지 않았을까…라고 반성해봅니다

주로 같이 발생한 에러 로그는 다음과 같습니다.

- `Connection is not available. request timed out after xxxxms`
- `unable to commit(rollback) against JDBC Connection`
- `OutofMemoryError: GC overhead limit exceeded`

[<img src="{{site.url}}/img/order-error/Untitled%202.png" />](/img/order-error/Untitled%202.png)

[<img src="{{site.url}}/img/order-error/Untitled%203.png" />](/img/order-error/Untitled%203.png)

[<img src="{{site.url}}/img/order-error/Untitled%204.png" />](/img/order-error/Untitled%204.png)

# 이슈 feat. 불타는 CPU, RAM 그리고 에러 로그..

---

이슈 발생 후 저희 파트는 주어진 상황으로 원인을 파악하기 시작했지만… 심하게 느린 **슬로우 쿼리**도 없고, 갑자기 **요청 트래픽**이 늘어난 상황도 아닌데 **`CPU`와 `RAM` 사용량이 100%에서 떨어지지 않고 있는 상황**이었고, 저희 파트는 정확한 원인을 찾지 못하고 있었습니다.

급하게 과부하 걸린 서버의 `Application`을 재실행하여 상황은 정리했지만 언제 또  같은 이슈가 발생할지 알 수 없는 상황이었습니다. 

일단 `````````````````````HikariCP Max-Connection````````````````````` 증가시키고 **`HikariCP`, `Mysql Connector`**버전 문제일 가능성도 염두하여 최신 버전으로 올려 배포하고 상황을 지켜보기로 했습니다

하.지.만… 일주일 뒤 똑같은 상황이 발생했습니다. 저희 파트는 버전 문제가 아님을 깨닫고 일단 발생한 에러 로그를 재현해보면서 원인을 찾아보기로 했습니다. 삽질의 시작이었습니다…

![Untitled](/img/order-error/Untitled%205.png)

# 에러 로그를 재현해보자

---

먼저 이슈 동안 발생한 에러 로그들이 발생하는 원인을 알아보고 재현해보았습니다.

### 1. Connection is not available. request timed out after XXXXms 예외

해당 예외는 `Hikari`에서 `Connection`을 점유(`getConnection()`) 할 때 사용 가능한 커넥션이 없을 경우 얻을 때까지 대기하는 시간이 `hikari.connection-timeout` 보다 길어질 경우 발생하는 예외입니다.

쉽게 말해 `Connection`을 기다리는 시간이 설정값을 넘을 때 발생하는 예외라 이해하시면 됩니다

아래는 예외를 발생시키는 테스트 코드입니다.

> HikariCP 설정
> 
- maximum-pool-size: 1 → 최대 생성할 수 있는 `Connection` 개수
- connection-timeout: 30초 → `Connection`을 얻을때까지의 최대 대기 시간

![Untitled](/img/order-error/Untitled%206.png)

> Mysql wait_timeout 설정
> 

`wait_timeout`은 `Mysql`**에 설정된** 활동하지 않는 `Connection`을 유지하는 시간이다.
* 여기서 헷갈리지 않아야 하는 것은 해당 `Connection`은 `Hikari Connection`이 아닌 `Mysql`에서 관리하는 `Connection`입니다. `wait_timeout` 관련해서는 아래에서 자세히 설명하겠습니다.

![Untitled](/img/order-error/Untitled%207.png)

> 코드
> 

```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class HikariTestService {
    @PersistenceContext
    private final EntityManager em;

    /**
     * Hikari Connection 35초 동안 점유
     */
    @Transactional
    public void transactionLocking() {
        log.info("=======transactionLocking 수행========");

        Member member = new Member("TestMember");
        this.em.persist(member);

        //35초 sleep 대기
        try {Thread.sleep(35000);} catch (InterruptedException ignored) {}

        log.info("=======transactionLocking 종료========");
    }

    @Transactional
    public void saveMember() {
        log.info("=======saveMember 수행========");

        Member member = new Member("TestMember2");
        em.persist(member);
    }
}
```

```java
@SpringBootTest
public class HikariTests {
    @Autowired
    HikariTestService hikariTestService;
 
    /**
     * Hikari ConnectionRequestTimedOut 유발 테스트
     */
    @Test
    void connection_request_timed_out_test() {
        CompletableFuture.allOf(
                CompletableFuture.runAsync(() -> hikariTestService.transactionLocking()),
                CompletableFuture.runAsync(() -> delaySave())
        ).join();
    }

    /**
     * TransactionLocking() 먼저 실행을 위해 1초 딜레이
     */
    void delaySave() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
        }

        hikariTestService.saveMember();
    }
}
```

코드는 다음과 같이 수행됩니다.

1. `CompletableFuture`를 통해 `hikariTestService.transactionLocking()`,`hikariTestService.saveMember()` 비동기 수행
2. `transactionLocking()`이 먼저 수행되게 하기 위해 **1초** **delay**를 주고 `saveMember()` 실행
3. `transactionLocking()` 실행될 때 `@Transactional` 을 통해 `Connection` 획득
4. `Thread.Sleep`을 통해 **35초간** `Connection`을 점유
5. `Connection` 점유 동안 `saveMember()`는 `Connection`을 획득하지 못하므로(`Connection`이 1개이므로) 계속 대기
6. **30초 후** `saveMember()` 는 `Connection`을 획득하지 못했으므로 예외 발생

[<img src="{{site.url}}/img/order-error/Untitled%208.png" />](/img/order-error/Untitled%208.png)

[<img src="{{site.url}}/img/order-error/Untitled%209.png" />](/img/order-error/Untitled%209.png)

### 2. Unable to commit(rollback) against JDBC Connection

해당 예외는  `Connection`이 아무런 활동을 하지 않는 시간이 `DB`에  설정된 `wait_timeout` 시간에 도달할 때 발생할 수 있습니다.

잠깐 `HikariCP Connection` 관련하여 설명하자면

`HikariCP`가 제공하는 `Connection` 객체는 `HikariProxyConnection`으로 `Wrapping`되어있는 형태입니다! 

다시 말하면 실제로 `Mysql`과 통신하는 `Connection` 객체를 클래스 안에 `delegate`로서 가지고 있으면서 `Connection`의 실제 기능을 수행할 때는 `delegate`를 대신 호출하는 방식입니다.

아래는 `HikariProxyConnecton` `ProxyConnection` 클래스의 일부입니다.

![Untitled](/img/order-error/Untitled%2010.png)

[<img src="{{site.url}}/img/order-error/Untitled%2011.png" />](/img/order-error/Untitled%2011.png)

`ProxyConnection`을 보면 `Mysql`과 통신하는 `Jdbc` 함수(`createStatement…`)들 구현부에는 `delegate` 함수를 대신 호출해주는 것을 확인할 수 있습니다.

갑자기 `Hikari Conenction`을 왜 설명했냐 하면……

 

`wait_timeout`으로 인해 `Mysql`과 연결이 끊어진다는 것은 실제로 연결된 `Connection` 즉, `delegate`가 참조하는 `Connection`이 활동(쿼리 실행) 하지 않는 시간이 `wait_timeout`을 넘을 경우 `Connection`이 끊어집니다.

쉽게 말해 `Connection`**이 쿼리를 날리지 않는 시간이 `wait_timeout`을 넘으면 `Mysql`에서 연결을 끊어버립니다.**

`Unable to commit(rollback) against JDBC Connection` 발생하는 이유는! `Hikari Connection`이 어떠한 쿼리를 수행하기 위해 `delegate`의 함수를 호출하지만 실제 `Mysql`과 통신을 하는 과정에서 이미 연결이 끊어졌을 경우 해당 예외가 발생할 수 있습니다.

아래는 간단하게 예외를 발생시키는 코드입니다.

> HikariCP 설정
> 

![Untitled](/img/order-error/Untitled%2012.png)

> wait_timeout
> 

![Untitled](/img/order-error/Untitled%2013.png)

> 코드
> 

```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class HikariTestService {
    @PersistenceContext
    private final EntityManager em;

    @Transactional
    public void delaySaveMember() {
        log.info("=======delaySaveMember 수행========");
        try {Thread.sleep(22000);}
        catch (InterruptedException e) {}

        Member member = new Member("TestMember2");
        em.persist(member);
    }
}
```

```java
@SpringBootTest
public class HikariTests {
    @Autowired
    HikariTestService hikariTestService;

    @Test
    void wait_timeout_test() {
        hikariTestService.delaySaveMember();
   }
}
```

코드는 다음과 같이 수행됩니다.

1. `delaySaveMember()` 호출
2. `@Transactionl`을 통해 `Hikari Connection` 획득
3. 22초(`wait_timeout`보다 긴 시간) 동안 `Connection`은 아무런 활동을 하지 않음
4. `member` 저장 시도
5. 이미 **20초(`wait_timeout`)**가 지난 시점에서 `Mysql`과의 연결은 끊어졌기에 예외 발생

[<img src="{{site.url}}/img/order-error/Untitled%2014.png" />](/img/order-error/Untitled%2014.png)

`Unable to rollback~`말고 `Unable to commit~`이 발생할 수 도 있는데 이 경우는 `commit` 수행하는 시점에 `Mysql Connection`이 끊겨있을 경우 발생할 수 있습니다.

아래는 `Unable to commit~` 유발하는 코드입니다.

```java
...
//테스트 코드는 동일

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class HikariTestService {
    @PersistenceContext
    private final EntityManager em;

    @Transactional
    public void delaySaveMember() {
        try {Thread.sleep(22000);}
        catch (InterruptedException e) {}
    }
}
```

[<img src="{{site.url}}/img/order-error/Untitled%2015.png" />](/img/order-error/Untitled%2015.png)

위 코드를 보면 `@Transactional` 로 `Connection` 획득하고 **22초** `Sleep`후에 아무 동작도 하지 않습니다. 하지만 `@Transactional` 함수가 종료될 때 자동으로 `commit` 명령어를 수행합니다.

즉, 위 코드는 **22초 후** `commit`을 날릴 때 이미 `connection`이 끊긴 상태이기 때문에 위 예외가 발생하는 것입니다.

여기서 하나 더 설명해야 할 것은 `hikariCP max-lifetime`, `wait_timeout`의 관계입니다.

`max-lifetime`은 `Hikari Connection` **생존 시간**을 설정합니다. 그리고 해당 생존 시간은 정확히 말하면 `Connection`이 사용되지 않는 시간으로 측정됩니다.

예를 들어 `max-lifetime`을 **30초**로 설정하고 `Connection Pool`에서 **28초**를 대기하다 `Connection`이 사용되고 다시 `Pool`로 돌아오면 **2초 후**에 사라집니다.

만약 `Connection`을 얻었는데 `DB`와의 연결이 끊겨 있다면 해당 `Connection`은 제거되고 다른 `Connection`을 제공합니다.

이로 인해 `**HikariCP`는 **`max-lifetime`을 `wait_timeout`보다 짧게 설정하는 것을 권장하고 있습니다.****

다음 코드는 `HikariCP getConnection()` 일부입니다.

[<img src="{{site.url}}/img/order-error/Untitled%2016.png" />](/img/order-error/Untitled%2016.png)

코드를 보면 `isConnectionAlive()` 존재하는데 해당 함수에서 `Connection`이 끊겨있는지 체크하고, 끊어졌다면  `close`하는 것을 확인할 수 있습니다.

그런데 여기서 저는 궁금한 게 생겼습니다.

`max-lifetime`: **30초,** `wait_timeout`: **20초**로 설정(`max-lifetime`이 `wait_timeout`보다 길게 설정) 후 `Connection Pool`에서 **18초** 대기하다 `@Transactional` 함수 진입 후 3초를 대기하면 해당 `Connection`은 **21초** 동안 활동을 안한게 되므로 끊어져야 하는게 아닌지 궁금했습니다. 

이렇게 될 경우 `wait_timeout`으로 인해 끊기기 아슬아슬한 시간차이로(Ex:19.XX초) `Connection`이 **사용**된다면 함수 수행 과정에서 `wait_timeout`으로 인해 `Connection`이 끊기는 상황이 운영에서 간간히 발생할 수 있지 않을까 생각했습니다.

그래서 아래처럼 테스트를 진행해봤습니다. **설정은 위 테스트와 동일합니다.**

```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class HikariTestService {
    @PersistenceContext
    private final EntityManager em;

    @Transactional
    public void delaySaveMember() {
        log.info("=======delaySaveMember 수행========");
        try {Thread.sleep(4000);}
        catch (InterruptedException e) {}

        Member member = new Member("TestMember2");
        em.persist(member);
    }
}
```

```java
@SpringBootTest
public class HikariTests {

    @Autowired
    HikariTestService hikariTestService;

    @Test
    void wait_timeout_test() {
        try {Thread.sleep(18000);}
        catch (InterruptedException e) {}
        hikariTestService.delaySaveMember();
    }
}
```

간단하게 `Connection` 점유 전 **18초**를 대기하고 `Connection` 점유 후 **4초** 대기 후 쿼리를 날리는 테스트입니다.

**결과는 *정상적으로* 수행됐고, Mysql log는 다음과 같습니다.**

![Untitled](/img/order-error/Untitled%2017.png)

`Connection`이 연결된 후 **18초 후**에 `SET autocommit=0` 명령어가 수행되는 것을 확인할 수 있습니다. 

즉 `@Transactional` 진입할 때 `autocommit`을 설정하는 것을 확인할 수 있었고, 이로 인해 `wait_timeout`이 초기화가 된다는 것을 알 수 있었습니다.

### 3. OutOfMemoryError: GC overhead limit exceeded

`OutOfMemoryError`는 메모리가 부족할 때 발생하는 에러입니다. `GC overhead limit exceeded`의 원인은 `JVM`이 `GC(Garbage Collection)`를 수행하는 시간의 **98%**이상을 사용했지만, `HeapMemory` **2%**미만으로 확보된 경우 발생할 수 있습니다.

즉 `Heap Memory`가 부족해 `GC`를 수행했지만 `Heap Memory` 확보가 되지 않을 때 발생할 수 있습니다.

`GC overhead limit exceeded` 가 발생할 경우 아래와 같은 로그가 출력됩니다.

```bash
[20XX-XX-XX XX:XX:XX.XXX] [ERROR] [GET /] - o.s.t.i.TransactionInterceptor           completeTransactionAfterThrowing - Application exception overridden by rollback exception
java.lang.OutOfMemoryError: GC overhead limit exceeded
```

# 에러 원인 발견

---

저희 파트는 2번째 이슈 발생 후 `Hikari Connection Leak` 로깅을 저희 파트 동료분께서 추가하여 배포했습니다.(로깅 관련 설명은 아래에서 설명하겠습니다.)

로그를 모니터링 하던중 어김없이 **3번째** 이슈가 발생했고! 저희는 드디어 문제의 원인을 발견했습니다! 

때마침 타팀에서도 **문제의 쿼리**를 저희에게 제보해주셨습니다.(여러 개발팀에서 원인 찾는 것에 도움을 많이 주셨습니다 ㅠ)

확인해보니 세금 계산서 관련 API에서 호출한 `ORM(JPA)` 쿼리였고 맨 끝에는 **IS NULL**이라는 구문이 들어있었습니다… 

해당 쿼리를 호출하는 함수를 살펴보니.. 역.시. 자바의 절친 `NULL`을 검사하지 않아 `NULL`값이 쿼리에 삽입되어 발생한 쿼리였습니다.

해당 쿼리로 **약46만건**이 한 번에 조회됐고, **250MB** 트래픽이 한 번에 발생한 것이 원인이었습니다.
(~~너무 기쁘지만 아주 살짝 허무한 이 느낌…~~) 

![Untitled](/img/order-error/Untitled%2018.png)

개발 서버에서 테스트를 진행했을 때 **16만 건**도 아래처럼 `Heap Memory`를  거의 **70%**를 사용하고 `Entity`만 **142MB**정도를 차지하고 있었습니다. 운영이었으면 단순 산술만 해도 `Entity`만 약 **400MB**… (운영 서버 `Max-Heap-Memory` **2GB** 할당)

[<img src="{{site.url}}/img/order-error/Untitled%2019.png" />](/img/order-error/Untitled%2019.png)

[<img src="{{site.url}}/img/order-error/Untitled%2020.png" />](/img/order-error/Untitled%2020.png)

이후 바로 수정 후 배포했고, **현재까지 같은 이슈는 발생하지 않고있습니다.**

이슈는 해결했지만 결론을 트래픽이 높은 쿼리로 인해 `CPU`, `Heap Memory`가 과부하가 났다고 마무리하기에는 찝찝했던 저는.. 직접 과부하 테스트를 하여 이슈를 재현해보았습니다.

최대한 비슷한 환경을 위해 저희 `Order API` 서버에서 특정 시간 동안 데이터를 적재해보는 테스트를 진행했습니다.

`-XX:+PrintGC` 옵션으로 `GC Log`와 `Intellij Live Charts`를 통해 `CPU`, `Heap Memory` 사용량을 체크해봤습니다.

```java
@RestController
public class MemberController {
    @GetMapping("/memory-test")
    public void memoryTest() {
        Map<Integer, String> map = new HashMap<>();
        Random r = new Random();
        long l = System.currentTimeMillis();
        long time = 180000;   //180초

			  //180초 동안 데이터 적재
        while(System.currentTimeMillis() - l < time) {
            //데이터 적재
            map.put(r.nextInt(), "Test Data");
        }
    }
}
```

아래는 데이터를 적재하는 동안의 `CPU`, `Heap Memory` 그래프입니다.

![Untitled](/img/order-error/Untitled%2021.png)

![Untitled](/img/order-error/Untitled%2022.png)

![Untitled](/img/order-error/Untitled%2023.png)

![Untitled](/img/order-error/Untitled%2024.png)

- `Heap Memory`가 감소하는 구간에서 `CPU` 사용량이 증가 → `Full GC` 수행으로 인한 `CPU` 사용
- `Full GC`를 수행해도 확보한 메모리가 점점 줄어듦
- 시간이 지날수록 `Full GC` 시도 빈도가 많아짐

로그를 확인해보면 `Heap Memory`가 부족한 상태이기에 지속해서 `Full GC`가 호출되는 것을 확인할 수 있습니다. `Full GC`는 수행되는 동안에는 `Stop The World(STW)`가 발생하는데 모든 `Thread`가 중지상태에 들어가고 `GC`만 수행하게 됩니다.

이제 이 상태에서 `Jmeter`로 아래 API로 부하 테스트를 진행했습니다.

```java
@Transactional
public Long testFindbyOrdNo(Long ordNo) {
	log.info("=============testFind 호출============");
	//무거운 로직이라 가정하기 위해 1초 Sleep
	try{Thread.sleep(1000);}
	catch (InterruptedException e) {throw new RuntimeException(e);}
	Order order = this.orderRepository.findByOrdNo(ordNo);
	return order.getOrdNo;
}
```

`Jmeter`는 아래와 같이 30번 동시 호출을 하도록 설정했습니다.

![Untitled](/img/order-error/Untitled%2025.png)

**테스트 결과**는 아래와 같이 이슈 상황과 유사하게 재현되었습니다!

[<img src="{{site.url}}/img/order-error/Untitled%2026.png" />](/img/order-error/Untitled%2026.png)

[<img src="{{site.url}}/img/order-error/Untitled%2027.png" />](/img/order-error/Untitled%2027.png)

[<img src="{{site.url}}/img/order-error/Untitled%2028.png" />](/img/order-error/Untitled%2028.png)

- `Heap Memory` 데이터 적재로 인해 `Full GC` 빈번히 발생하여 대부분 요청이 처리되지 못함
    - 이슈 상황과 동일하게 에러 발생
        - `Connection is not available. request timed out after xxxxms`
        - `unable to commit(rollback) against JDBC Connection`
        - `OutofMemoryError: GC overhead limit exceeded`
- `GC`와 요청으로 인해 `CPU` **과부하** 유지

### 지금까지 이슈를 총정리 해보자면..

> 이슈 해결 과정
> 
- **12월 7일** 1차 이슈 발생 → `CPU`, `RAM` 과부하 및 에러 로그 다량 발생
- `HikariCP`, `Mysql Connector` 업그레이드 및 `Hikari Max-Pool` 설정 변경
- 2차 이슈 발생
- `Hikari Connection Leak` 로그 설정 및 에러 로그 재현 및 분석
- 3차 이슈 발생
- `Hikari Connection Leak` 로그로 인해 원인 발견
- 이슈 해결

> 이슈 원인
> 
- 과도한 트래픽의 데이터로 인한 `Heap Memory` 과부하 및 빈번한 `Full GC`로 인해 `CPU` 과부하
    - `GC` 수행 시간이 늘어남에 따라 요청 처리 시간이 부족해 `GC overhead limit exceeded` 발생
- `Stop The World(STW)`으로 인해 요청 처리 `Thread`가 멈췄기에 `Connection` 획득 시간 초과 또는 `wait_timeout` 초과로 인한 `Connection Close`

# 커넥션 누수 로깅

---

이번 이슈 원인 찾는데 결정적 도움이 됐던 `HikariCP Connection Leak Logging` 설정에 대해 설명하고자 합니다. 

해당 `Logging`은 `HikariCP`에서 `Connection Pool`상태 및 `Connection leak`(커넥션 누수)가 의심될시 `Log`로 출력해 주는 설정입니다.

### HikariCP Logging

`HikariCP`의 현 상태에 대한 설정 방법은 아래와 같습니다.

```yaml
logging:
  level:
    com.zaxxer.hikari.HikariConfig: DEBUG
    com.zaxxer.hikari: TRACE
```

위의 설정을 추가한다면, 현재 `HikariCP`가 관리하는 `Pool`의 상태와 `Connection` 활성화 여부를 출력하게 됩니다.

```html
2022-12-22 01:18:07.315 DEBUG 22340 --- [connection adder] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection conn5: url=jdbc:h2:mem:testdb user=SA
2022-12-22 01:18:08.080 DEBUG 22340 --- [connection closer] com.zaxxer.hikari.pool.PoolBase          : HikariPool-1 - Closing connection conn2: url=jdbc:h2:mem:testdb user=SA: (connection has passed maxLifetime)
2022-12-22 01:18:08.156 DEBUG 22340 --- [pool-1 housekeeper] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Pool stats (total=5, active=0, idle=5, waiting=0)
```

해당 로그를 통해 다음과 같은 상태를 확인할 수 있습니다. 

- **Total** : 전체 `Connection`
- **Active** : 사용중인 `Connection`
- **Idle** : 유휴 `Connection`
- **Waiting** : 대기중인 요청

아래는 `Connection`이 끊어지는 상태는 다섯 가지로, 해당 로그를 통해 `Connection`이 정상 종료되었는지, 오류가 발생하여 연결이 끊겼는지 등을 확인 할 수 있습니다.

1. **연결 확인에 실패할 경우** : 연결이 만료후 교체되며, `Failed to validate connection…` 로그가 출력됩니다.
2. **오랫동안 연결이 유휴상태 일 경우** : `idleTimeout` 설정에 의해 유휴 상태의 `Connection`이 만료후 교체되며, `Closing … (connection has passed idleTimeout)` 로그가 출력됩니다.
3. **연결이 maxLifeTime 설정 시간까지 도달한 경우** : `maxLifetime` 설정에 의해 `Connection`이 만료되어 종료 후 `Closing … (connection has passed maxLifetime)` 로그가 출력됩니다. 
만약 시간이 만기됨에도 `Connection`이 사용 중일 경우 사용 후 `(connection is evicted or dead)` 로그가 출력됩니다.
4. **사용자가 수동으로 연결을 제거한 경우** : `Connection`이 만료되고 교체되며 `closing … (connection evicted by user)` 의 로그가 출력됩니다.
5. **JDBC 호출에서 복구가 불가능한 SQLException인 경우** : `JDBC`에서 `Connection`을 복구 할 수 없는 `SQLException`가 발생했을 경우를 의미합니다. `(connection is broken)` 로 끝나는 로그가 출력됩니다.

```bash
[2022-12-20 18:27:43.452] [WARN ] [] - c.zaxxer.hikari.pool.ProxyConnection     checkException       - ord-pool - Connection com.mysql.jdbc.JDBC4ReplicationMySQLConnection@d8e45d2 marked as broken because of SQLSTATE(08007), ErrorCode(0)
com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException: Communications link failure during rollback(). Transaction resolution unknown.
```

### leak-detection-threshold 옵션

`HikariCP`에서 관리하는 `Connection`에 대한 `leak`(누수)가 의심될 경우 `WARN`등급의 `logging`을 출력해주는 설정입니다.

다시 말해 누수가 의심될 경우를 말하는 것이기 때문에, 무조건 적인 누수는 아니지만 `Connection` **반환이 늦는 슬로우 쿼리 유무를 판별할 수 있습니다.**

설정은 아래와 같이 할 수 있습니다.

```yaml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 2000 # 누수 판별 기준값
```

설정 값은 최소 값은 **2000ms(2s)** 최대 값은 `max-lifetime`(default 30분)설정 값보다는 작게 설정해주어야 합니다. 

해당 설정을 추가하고, 설정 값보다 `Connection` 반환이 늦어진다면 아래와 같은 로그가 출력 됩니다.

```bash
2022-12-22 01:52:53.888  WARN 18620 --- [l-1 housekeeper] com.zaxxer.hikari.pool.ProxyLeakTask     : Connection leak detection triggered for conn0: url=jdbc:h2:mem:testdb user=SA on thread http-nio-8080-exec-7, stack trace follows

java.lang.Exception: Apparent connection leak detected
	at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:128) ~[HikariCP-4.0.3.jar:na]
	at org.hibernate.engine.jdbc.connections.internal.DatasourceConnectionProviderImpl.getConnection(DatasourceConnectionProviderImpl.java:122) ~[hibernate-core-5.6.7.Final.jar:5.6.7.Final]
	...
	at com.tax.refund.domain.user.service.UserService$$EnhancerBySpringCGLIB$$317f9a0.login(<generated>) ~[main/:na]
	at com.tax.refund.domain.user.api.UserController.login(UserController.java:41) ~[main/:na]
```

만약 누수로 의심되었던 `Connection`이 정상적으로 반환 될 경우는 다시 `leak`에 대한 정정 로그가 출력됩니다.

```bash
2022-12-22 01:56:21.205  INFO 17092 --- [nio-8080-exec-3] com.zaxxer.hikari.pool.ProxyLeakTask     : Previously reported leaked connection conn0: url=jdbc:h2:mem:testdb user=SA on thread http-nio-8080-exec-3 was returned to the pool (unleaked)ㅂ
```

반면에 실제 누수가 발생 된다면 서버 로그에서 `SQLException`과 로그와 함께 아래의 로그도 함께 출력되게 됩니다.

```bash
[XXXX-XX-XX XX:XX:XX.XXX] [WARN ] [/GET] - c.zaxxer.hikari.pool.ProxyConnection     checkException       : HikariPool-1 - Connection conn0: url=jdbc:h2:mem:testdb user=SA marked as broken because of SQLSTATE(08007), ErrorCode(0)
```

실제 위의 슬로우 쿼리 상황을 재현 한 뒤 `Connection leak`에 대한 로그를 확인 할 때는 아래와 같은 로그가 출력됨 을 확인할 수 있었습니다.

```bash
[2022-12-20 18:27:43.452] [WARN ] [] - c.zaxxer.hikari.pool.ProxyConnection     checkException       - ord-pool - Connection com.mysql.jdbc.JDBC4ReplicationMySQLConnection@d8e45d2 marked as broken because of SQLSTATE(08007), ErrorCode(0)
com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException: Communications link failure during rollback(). Transaction resolution unknown.
```

# 마무리

---

이번 이슈를 해결하는 과정을 거치면서 `HikariCP`, `GC`등 평소에 개념만 알고 있던 것들을 자세히 분석하고 알아볼 수 있어서 배운 것이 참 많았던 것 같습니다. **저희 IT연구소 서비스 개발팀은 지금도 사람인의 원활한 서비스를 위해 항상 고민하고 노력하고 있습니다.** 

긴 글 읽어주셔서 감사합니다.
