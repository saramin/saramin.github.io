---
title: 헥사고날의 법칙
author: 정종현
subtitle: 헥사고날 아키텍처란
---

안녕하세요. 사람인 서비스개발팀 빌링파트 정종현입니다.


빌링파트는 사람인의 다양한 비즈니스 서비스가 확장됨에 따라, 유저들이 보다 빠르고 쉽게 결제할 수 있도록 지원하고 있습니다. 
동시에, 결제 이후의 세금계산서 발행, 현금영수증 처리, ERP 연동 등 여러 부서와 연계된 후처리 업무를 효율적으로 제공하는 서비스를 개발하고 있습니다.


이번 글에서는 다양한 비즈니스 요구사항에 유연하게 대응하고, 서비스의 확장성 (부하 분산, 기능 분할 등) 을 높이기 위해 헥사고날 아키텍처(Hexagonal Architecture) 를 도입한 경험을 공유하고자 합니다.
이 과정에서 마주한 이론과 실제 사이의 간극, 그리고 그로부터 얻은 인사이트에 대해 이야기해보려 합니다.

![](http://10.100.1.93/img/jeongjh/img01.jpg)





## 왜? 헥사고날 아키텍처인가?



어플리케이션의 핵심 도메인을 중심에 두는 비슷한 철학이나 구조를 가진 아키텍처 중에는 헥사고날 말고도 클린, 어니언 등이 존재합니다.

|      **아키텍처 스타일**      |   **별칭/기반 개념**   |         **주요 특징**          |
|:----------------------:|:----------------:|:--------------------------:|
| Hexagonal Architecture | Ports & Adapters | 입출력을 도메인 밖으로 밀어내고, 유연한 확장성 |
|   Onion Architecture   |   계층의 내부-외부 구조   |   의존성 방향을 안쪽(도메인)으로만 허용    |
|   Clean Architecture   |  Uncle Bob 아키텍처  |  계층 간 의존 규칙 + 인터페이스 중심 설계  |

> 출처 : "도메인 중심 아키텍처에 대해 알려주세요" 프롬프트. ChatGPT, GPT-3.5, OpenAI, 2025년 5월 25일, chat.openai.com.


![](http://10.100.1.93/img/jeongjh/img02.jpg)



`잘 부탁 드립니다! ChatGPT 님!`



이번에 저희가 진행하고자 했던 프로젝트는 회계나 재무등에서 사용하는 외부 시스템을 연동하고 명확하게 정의 되어있는 기능을 분리하는 것이 목표였습니다. 
당장에는 ERP, 세금계산서만을 연동하지만 신규 프로젝트이기에 추후 연동이 늘어날수 있고 기술요소 (DB, 메시징 시스템 등)가 변경될 가능성이 있습니다.



이에 가장 적합한 아키텍처는 핵사고날이라고 판단하여 진행하게 되었고 헥사고날에 대해 요밀조밀 뜯어서 설명해 보도록 하겠습니다.




## 1. 헥사고날 아키텍처란



헥사고날 아키텍처(Hexagonal Architecture)는 소프트웨어를 다음과 같은 세 가지 주요 영역으로 구분하여 설계하는 패턴입니다.

![](http://10.100.1.93/img/jeongjh/hexagonal-architecture.jpg)

**핵심 도메인(Core Domain)**

* 애플리케이션의 비즈니스 로직과 규칙을 정의하는 중심 레이어.
* 외부 환경에 대한 의존성이 없으며 독립적으로 동작.
* 예) 도메인 모델, 애플리케이션 서비스(Use Case).


**포트(Ports)**
* 핵심 도메인(Core Domain)과 외부 어댑터 간의 연결점(인터페이스) 역할하여 입력포트와 출력 포트로 나뉨
* 입력 포트(Input Ports)
    * 외부 요청을 핵심 도메인으로 전달.
* 출력 포트(Output Ports)
    * 핵심 도메인의 결과를 외부로 전달.


**어댑터(Adapters)**
* 포트를 구현하여 외부(데이터베이스, UI, 메시지 시스템 등)와 상호작용.
* 입력 어댑터(Input Adapters)
    * 외부 요청(REST API, CLI 등)을 포트로 전달.
* 출력 어댑터(Output Adapters)
    * 포트 결과를 외부 시스템에 전달 (예: DB 저장, 이벤트 발행).



**장단점**
* 장점
    * 유지보수 용이성
        * 기술 스택이나 외부 의존성이 변경되어도 비즈니스 로직에 미치는 영향을 최소화합니다.
    * 테스트 용이성
        * 비즈니스 로직을 독립적으로 테스트할 수 있어 단위 테스트 작성이 쉬워집니다.
    * 유연성
        * 새로운 인터페이스(예: REST API 대신 GraphQL 추가)를 추가하기 쉽습니다.
    * 재사용성
        * 핵심 도메인 로직은 특정 어댑터에 의존하지 않으므로, 다른 환경에서도 재사용 가능합니다.

* 단점
    * 초기 설계의 복잡성
        * 작은 프로젝트에서는 필요 이상의 복잡성을 초래할 수 있습니다.
    * 추가적인 학습 곡선
        * 포트와 어댑터, 의존성 방향성 등 아키텍처 개념을 이해해야 합니다.
    * 과잉 설계의 위험
        * 작은 규모의 프로젝트에서는 비효율적일 수 있습니다.



정리해보면 헥사고날 아키텍처는 복잡한 도메인과 다양한 외부 인터페이스를 가진 시스템에 적합하며, 장기적으로 유지보수 비용을 절감할 수 있습니다.
하지만 작은 규모의 프로젝트나 단순한 요구사항에서는 과잉 설계로 이어질 수 있으므로 신중히 선택해야 합니다.


![](http://10.100.1.93/img/jeongjh/img03.jpg)

`MVP 는 빠져 주세요`




**2. 주요 구현 요소**


다중 모듈 구성으로 각 요소를 모듈단위로 구성했습니다.
![](http://10.100.1.93/img/jeongjh/hexagram.jpg)

데이터 흐름은 다음과 같은 단계를 거칩니다.

    [사용자 요청 또는 외부 시스템 이벤트]

                   ↓

    [Adapter (입력 어댑터): 예, REST Controller, Consumer]
    
                   ↓

    [Input Port (UseCase 인터페이스)]
    
                   ↓
    
    [Application/Domain Layer (비즈니스 로직)]
    
                   ↓
    
    [Output Port (추상화된 의존 인터페이스)]
    
                   ↓
    
    [Adapter (출력 어댑터): 예, DB Repository, External API Client] 

 `출처 : "헥사고날의 데이터 흐름을 설명해주세요." 프롬프트. ChatGPT, GPT-3.5, OpenAI, 2025년 5월 25일, chat.openai.com.`



**Input Adapter (= API Module / 입력 어댑터)**

* 외부 요청 수신하는 Controller
* 사용자가 REST API를 통해 요청하면 요청은 입력 어댑터(Controller)에 전달되고 입력 어댑터는 요청 데이터를 검증하고 입력 포트를 호출.


```java
// 입력 어댑터 샘플소스
// Controller에서 get방식으로 호출을 받음
@GetMapping(value = "/if-sales/{salesType}/{salesSeq}")
  public ResponseEntity<ApiResult<ErpIfSale>> getIfSales(
  @Parameter(description = "사이트구분", required = true) @PathVariable("salesType") String salesType, 
  @Parameter(description = "채용정보SEQ", required = true) @PathVariable("salesSeq") Long salesSeq) {
    return ok(erpService.findErpIfSalesById(salesType, salesSeq));
}
```

**Input Port (= link-module / 입력 포트 구현)**

* 입력 포트를 구현한 애플리케이션 서비스
* domain의 비즈니스 로직을 호출.

```java
// 입력 포트 구현 샘플소스
// domain의 ERP정보 조회하는 로직을 호출
public ErpIfSale findErpIfSalesById(String salesType, Long salesSeq) {
  return erpIfSalePort.findErpIfSaleById(SalesType.byName(salesType), salesSeq);
}
```

**Domain Layer (= domain / 핵심 도메인)**

* 비즈니스 로직과 입력 포트가 있는 모듈로 도메인 모델과 협력하여 작업을 수행.
* 애플리케이션 서비스 또는 유스케이스를 나타내는 인터페이스.
* 작업 결과를 처리하기 위해 출력 포트를 호출.
* 출력 포트를 호출로 외부 시스템(예: 데이터베이스, 메시지 브로커)과의 통신

```java
// 핵심 도메인 샘플소스
// ERP를 조회하는 비즈니스 로직을 작성
public interface ErpIfSalePort {
  ErpIfSale findErpIfSaleById(SalesType salesType, Long salesSeq);
}
```

**Output Port (= infrastructure / 출력 포트 구현)**

* 출력 포트를 구현한 어댑터로 외부 시스템과 상호작용.
* 작업 결과가 입력 어댑터로 반환되어 사용자에게 전달.

```java
// 출력 포트 구현 샘플 소스
// DB에서 실질적으로 데이터를 조회하는 로직 구현
@Override
public ErpIfSale findErpIfSaleById(SalesType salesType, Long salesSeq) {
    return erpIfSaleRepository.findById(IfSaleId.of(salesType, salesSeq))
             .map(erpIfSaleMapper::toDomain)
             .orElse(null);
}
```

**common (공통 유틸리티)**

* 공통적으로 사용하는 유틸리티를 정의.

```java
// 공통 유틸리티 샘플 소스
// 쿼리스트링을 Map으로 변경하는 로직 구현
public static Map<String, String> getQueryParamsMapString(String str) {
  UriComponentsBuilder uriComponentsBuilder = UriComponentsBuilder.newInstance();
  return uriComponentsBuilder.query(str.trim()).build().getQueryParams().toSingleValueMap();
}
```

**Output Adapter 는 어디로??**

업체에서 제공하는 모듈을 사용하고 있어 자체적으로 구현하지 않아도 자동처리 되고 있습니다.

![](http://10.100.1.93/img/jeongjh/img04.jpg)


`모듈이 자동으로 저장~`





**3. 실제 구현에서 배운 교훈**



**비즈니스 로직 중심 설계**


프로젝트 초기에 구조를 잡고 각 레이어의 역할과 경계를 명확히 정의하는 데 많은 시간이 소요되었지만 핵심 도메인(Core Domain)을 외부 의존성으로부터 분리하면서 비즈니스 로직에만 집중할 수 있습니다.

처음에는 어렵지만, 시간이 지날수록 변경에 유연하게 대응할 수 있어 장기적으로는 훨씬 유리한 구조임라고 생각합니다.

그리고 실질 운영하면서 봐야할 부분이긴하지만 유지보수와 테스트가 용이해졌다고 생각합니다.



**유연성과 확장성**


각 구성요소의 역할과 협력을 이해하는 데 시간이 걸리고 포트와 어댑터 간 인터페이스 설계에서 혼란이 생기는 부분이 있었습니다.

하지만 외부 어댑터를 교체하거나 새로운 API나 새로운 데이터베이스 등을 추가할 때, 핵심 로직을 수정할 필요가 없어서 개발 생산성이 향상되었습니다.

특히, 다양한 입력/출력 어댑터(Web, CLI, 메시지 브로커 등)를 쉽게 통합 가능합니다.



**이상과 현실의 조화**


순수한 헥사고날 아키텍처의 이상과 현실적인 구현 사이에서 균형을 찾는 것이 중요했습니다.

때로는 실용적인 타협이 필요했지만, 그 과정에서 다음과 같은 원칙을 수립했습니다.

처음부터 완벽한 순수성을 추구하기보다 점진적으로 개선하고 비즈니스 중요도에 따른 우선순위 설정했습니다.

도메인의 복잡성과 기술적 제약을 고려한 경계를 설정하고 과도한 추상화를 지양했습니다.



**팀원의 합의와 학습**


초기에 구조를 잡고 레이어의 역할을 명확히 하는데 어려움이 있었습니다.

개발을 하는 중에 역할과 책임별로 나눠져야하는데 같은 패키지 안에 다른 기능을 하는 소스가 섞이는 일이 있었습니다.

이해나 논의가 부족해서 발생한 문제였습니다.

새로운 아키텍처를 적용할 때, 팀원 모두가 같은 이해를 가지고 접근해야 하고 충분한 학습과 논의가 필요하다고 생각되었습니다.


![](http://10.100.1.93/img/jeongjh/img05.jpg)

`빌링파트는 토론으로 승부.. 아니 단합한다.`





# 마치며


새로 도입하는 아키텍처인 만큼 관련 지식을 학습하고 실제 프로젝트에 적용하는데는 많은 노력이 필요했습니다.

헥사고날 아키텍처는 초기 진입장벽이 있었지만 기술의 유연성이나 확정성에서 프로젝트의 목적에 맞는 아키텍처를 선택했다고 생각합니다.



복잡한 애플리케이션이나 장기적인 유지보수가 중요한 프로젝트에서 강력한 도구가 될 수 있을듯 합니다.

헥사고날 아키텍처를 도입해봄으로써 프로젝트에 따른 유연하면서도 효율적인 아키텍처란 무엇인가를 생각해보는 계기가 되었습니다.



긴 글 읽어 주셔서 감사합니다!
