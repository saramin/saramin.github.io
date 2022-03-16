---
title: SPRING CLOUD GATEWAY를 이용한 API GATEWAY 구축기
---

안녕하세요.<br>
사람인에이치알 서비스인프라개발팀 이현규 입니다.<br>
이번 포스팅은 현재 사람인에이치알에 구축되어 운영되고 있는<br>
API GATEWAY에 대하여 공유하는 내용의 포스팅입니다.<br>
<br>


## API GATEWAY란?<br>
정말 간단하게 정의하자면,  API GATEWAY는 클라이언트와 백엔드 서비스 사이에 위치하는 리버스 프록시 역활을 하는 서비스 입니다.<br><br>
더불어 인증, 속도제어, 서킷브레이커, 모니터링등 여러 공동 모듈 및 관리포인트들을 추가하여 모든 API들의 관문(GATEWAY)의 역할을 하는 서비스 입니다.<br>
<br>

## 필요성?
현재 저희 사람인HR에서도 현재 마이크로서비스화가 되어가고 있고,<br>
서비스가 성장하고 변화되는 만큼 트래픽 및 API의 숫자가 증가됨에 따라, 전체적인 API 관리포인트가 필요했었습니다.<br>
그 과정에서 API GATEWAY의 필요성이 제기 되었고, 저희 팀은 API GATEWAY 구축 프로젝트를 진행하였습니다.<br>
여러 상용 솔루션 및 오픈소스들이 검토 되었으며<br>
저희 팀에서는 다음과 같은 이유로 Spring Cloud Gateway를 선정하여 사용하였습니다.<br>

- 팀 내에 이미 SPRING CLOUD 스택을 사용 중임.<br>
- SPRING CLOUD PROJECT는 SPRING 진영내에서도 가장 활발히 움직이는 프로젝트.<br>
- 다른 상용솔루션과 달리 개발자 관점에서의 강력한 제어가 가능함.<br>
- ELASTIC APM에서 JAVA AGENT를 지원하기 때문에, 이미 구축된 APM을 시스템을 기반으로 통합 모니터링이 쉽게 가능.<br>
<br>
<br>
## GATEWAY 시스템 구성
<img src="{{site.url}}/img/api-gw/GATEWAYPLATFORM구성도.png" width="700px" />{: .text-center }
<br>
구성은 크게 GATEWAY, EUREKA, FALLBACK-SERVER, ADMIN, DB, CONFIG-SERVER, GIT로 구성 되어있습니다.

- GATEWAY
	- 실제 라우팅 및 모니터링을 진행하는 GATEWAY의 본체
	
- EUREKA
	- CLIENT-SIDE 서비스 디스커버리를 구성하기 위한 구성 요소

- FALLBACK-SERVER
	- 상황에 맞는 대체 응답을 주기 위한 서버

- ADMIN
	- GATEWAY에 설정정보들을 입력하기 위한 ADMIN

- DB
	- ADMIN에서 입력된 데이터가 저장되기 위한 저장소

- CONFIG-SERVER 
	- GATEWAY에 설정정보를 동적으로 변경하기 위한 구성 요소

- GIT
	- CONFIG-SERVER가 읽을 YAML 파일이 저장되는 저장소

<br>
<br>
## SPRING CLOUD GATEWAY란?<br><br>
#### Netty를 사용<br>
Spring Cloud Gateway는 Tomcat이 아닌 Netty를 사용합니다.<br>
API GATEWAY는 모든 요청이 통과하는 곳이기 때문에 성능적인 측면이 매우 중요하며<br>
기존의 1THREAD / 1REQUEST 방식인 SPRING MVC를 사용할 경우 성능적인 이슈가 발생할 수 있습니다.<br>
Netty는 비동기 WAS이고 1THREAD / MANY REQUESTS 방식이기 때문에 기존 방식보다 더 많은 요청을 처리할 수 있습니다.<br>
<br>


#### 동작방식
<img src="{{site.url}}/img/api-gw/hanlder.png" width="800px" />{: .text-center }
<br>

출처 :[https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.1.0.RELEASE/single/spring-cloud-gateway.html](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.1.0.RELEASE/single/spring-cloud-gateway.html)
<br>

클라이언트는 Spring Cloud Gateway로 요청을 합니다.<br>
Gateway Handler Mapping이 Route의 조건에 일치하는 요청이라고 판단하면, 해당 요청을 로 보내줍니다.<br>
Gateway Web Handler는 요청과 관련된 Filter들을 통해 요청을 보냅니다.<br>
Filter는 요청의 기능에 따라 proxy 요청이 보내지기 전/후로 로직을 실행합니다.<br>

#### Route? Predicates? Filters?<br>

Spring Cloud Gateway에는 크게 3가지 구성요소가 존재합니다.

##### Route란?<br>
고유ID, 목적지 URI, Predicate, Filter로 구성된 구성요소.<br>
GATEWAY로 요청된 URI의 조건이 참일 경우, 매핑된 해당 경로로 매칭을 시켜줍니다.<br>


##### Predicate란?<br>
주어진 요청이 주어진 조건을 충족하는지 테스트하는 구성요소<br>
각 요청 경로에 대해 충족하게 되는 경우 하나 이상의 조건자를 정의할 수 있습니다.<br>
만약 Predicate에 매칭되지 않는다면 HTTP 404 not found를 응답합니다.<br>

##### Filter란?<br>
GATEWAY 기준으로 들어오는 요청 및 나가는 응답에 대하여 수정을 가능하게 해주는 구성요소<br>

<img src="{{site.url}}/img/api-gw/yaml3.png" width="800px" />{: .text-center }
<br>

사실 Predicate와 Filter에는 꽤 많은 설정들이 존재합니다.<br>
자세한 정보들은 공식 레퍼런스에서 확인 하실 수 있습니다.<br>
https://cloud.spring.io/spring-cloud-gateway/reference/html/<br>


## 우리는 어떻게 만들었나
SPRING CLOUD GATEWAY의 설정들은 Java DSL로 정의를 하거나 application.yml 파일과 같은 설정 파일에서 정의를 하는 2가지 방식이 있습니다.
저는 application.yml에 설정 정보를 정의 하는 방식을 선택하였습니다.<br>
허나, 모든 개발자분들이 yaml 파일을 직접 수정하는 프로세스로 API GATEWAY의 설정들을 관리하기에는 무리가 있다고 판단하였기에  ADMIN을 통한 API GATEWAY 관리기능을 제공하기로 하였습니다.


아래는 실제 SPRING CLOUD GATEWAY에서 동작하고 있는 yml파일의 설정정보들입니다.

```
spring:
  cloud:
    gateway:
      routes:
      - id: recruit-service
        uri: lb://recruit
        predicates:
        - name: Path
          args:
            patterns: /recruit/list
        - name: Method
          args:
            methods:
            - GET
        filters:
        - name: CircuitBreaker
          args:
            name: recruitlist
            fallbackUri: forward:/fallbacks/7
        - name: RewritePath
          args:
            regexp: /recruit/(?<path>.*)
            replacement: /$\{path}

```

위 설정들은 결과적으로 아래와 같은 RouteDefinition이라는 객체로 매핑이 되고, 매핑된 설정정보들로 GATEWAY가 동작하는 구조입니다.<br>

```
public class RouteDefinition {

	private String id;

	@NotEmpty
	@Valid
	private List<PredicateDefinition> predicates = new ArrayList<>();

	@Valid
	private List<FilterDefinition> filters = new ArrayList<>();

	@NotNull
	private URI uri;

	private Map<String, Object> metadata = new HashMap<>();

	private int order = 0;

	public RouteDefinition() {
	}
	
	........
	........
	
}
```

위에 보이는 소스와 같이 PredicateDefinition 객체는 Predicate에 대한 설정 정보, FilterDefinition객체는 필터에 대한 설정 정보로 이루어져 있습니다.<br>
저는 위 클래스들을 기반으로 아래와 같은 Routes라는 객체를 재구성하였습니다.<br>

```
@Data
public class Routes implements Serializable {
	
	private static final long serialVersionUID = ;

	private String id;
	
	private String uri;
	
	private Integer domainId;
    
	private String routeId;
	
	private List<PredicateAndFilter> predicates;
	
	private List<PredicateAndFilter> filters;
	
	private Map<String, Object> metadata;
	
	
}	

```

결과적으로 Spring Cloud Gateway에서 제공하는 RouteDefinition과 비슷한 구조를 가진 Routes라는 객체를 생성하였고,<br>
이 객체를 가지고 YAML파일을 작성하는 Generator 클래스를 작성하였습니다.<br>


아래 화면은 설명드렸던 Routes에 해당하는 정보들을 입력하는 화면입니다.
<img src="{{site.url}}/img/api-gw/routes2.png" width="800px" />{: .text-center }
<br>

맨 위에 시스템 구성도와 같이 시스템은 크게 GATEWAY,  CONFIG-SERVER, ADMIN, DB, GIT등으로 구성되어 있습니다.<br>
흐름도를 보면 다음과 같습니다.<br>
1. ADMIN에서 사용자가 게이트웨이 설정정보를 입력하여 DB로 저장하고<br>
2. ADMIN에서 반영버튼을 눌렀을 때, ADMIN은 GATEWAY 설정정보 YAML파일을 생성하여 GIT으로 PUSH 합니다.<br>
3. 마지막으로 ADMIN에서 적용 버튼을 눌렀을 떄, 게이트웨이는 CONFIG-SERVER에서 제공되는 YAML 설정 정보를 반영하는 구조로 되어 있습니다.<br>

<br><br>
#### CircuitBreaker
<img src="{{site.url}}/img/api-gw/circuit.png" width="800px" />{: .text-center }<br>

서킷브레이커에 대한 설명을 간단하게 표현한 그림입니다.<br>
정상적인 상태는 CLOSED 상태, 장애상황이 발생되는 경우는 OPEN 상태입니다.<br>
장애상황이 발생되어 OPEN 상태일 경우,<br>
일정 시간이 지나고 CircuitBreaker의 상태를 CLOSED로 바꾸기 위하여 판단해야 하는 상태를 HALF_OPEN 상태라고 합니다.<br>
HALF_OPEN 상태에서 정상적인 상태라고 판단되면 CLOSED, 정상적인 상태라고 판단이 되지 않으면 OPEN상태로 다시 변경되게 됩니다.<br>
CircuitBreaker는 HTTP statuscode 기반으로 동작하지 않습니다.<br>
Gateway 입장에서 400 series / 500 series 응답은 정상적인 응답일 뿐 CircuitBreaker의 동작에 영향을 끼치지 않습니다.<br>
문제가 생긴 API로 더이상 부하가 들어가지 않도록 사전에 차단(FAIL-FAST)해서 다른 서비스에 전파되지 않도록 막는 것이 주 목적입니다.<br>

 
#### 관제
<img src="{{site.url}}/img/api-gw/dashboard.png" width="800px" />{: .text-center }<br>

위 그림은 현재 KIBANA DASHBOARD에서 제공되는 APM DASHBOARD 화면의 일부분입니다.<br>
APM 데이터는 ELASTIC APM을 기반으로 적재되고 있는 [사내 시스템]({{ site.url }}/2020-04-01-heimdall/)에 적재하고 있습니다.<br>
API GATEWAY를 거쳐가는 데이터에 대한 전반적인 트래픽에 대한 정보, 트랜잭션에 대한 정보들을 전체적으로 볼 수 있습니다.<br>



### 마치며
처음에 API GATEWAY 프로젝트를 하게 되었을 때가 기억이 납니다.<br>
사람인 서비스에 적용된 모든 API를 다룰수 있는 시스템을 맡게 된다는 것에 대한 부담감이 작지 않았습니다.<br>
하지만, 개발된 API GATEWAY가 하나둘씩 사람인 서비스에 아무런 문제없이 적용이 되고 있을 때, 성취감을 느낄 수 있었습니다.<br>
지속적으로 사람인에서 서비스되고 있는 API들은 API GATEWAY를 통해 서비스 될 예정입니다.<br>

프로젝트 진행하며 발생된 문제들을 큰 어려움 없이 해결할 수 있게 도와주신 서비스인프라개발팀 팀원분들 정말 감사드립니다.<br>


마지막으로, 긴 글 읽어주신 모든분들께 감사드립니다.


## Reference
[https://spring.io/](https://spring.io/)<br>
[https:/cloud.spring.io/](https:/cloud.spring.io/)<br>
