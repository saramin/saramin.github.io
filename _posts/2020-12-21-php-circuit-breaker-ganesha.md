---
title: 장애 확산을 막기 위한 서킷브레이커 패턴을 PHP에서 구현해보자
author: 이현재
tags:
- PHP
- MSA
- API
- Design pattern
- Circuit breaker
titlecolor: "#49b6bb"
subtitle: PHP에 적합한 서킷브레이커 라이브러리 소개
---

```SATA
Java는 Hystrix와 Resilience4j, 그렇다면 PHP는?!
```

안녕하세요, 사람인의 빌링 서비스를 개발하는 이현재입니다.
저희 파트가 운영하는 도메인은 Spring Boot를 사용하여 MSA로 분리되어 있습니다.
MSA를 위한 여러 가지 기법이 존재하는데, 그중에서 서킷브레이커 패턴을 활용하는 방법에 대해 살펴보도록 합시다.
이번에는 오픈소스 라이브러리를 활용해볼 건데, 사실 서킷브레이커 패턴은 원리만 이해하면 구현 자체가 어렵지 않습니다.
물론 고도화를 한다면 이야기는 달라지겠죠.
그러니 이 글을 보시는 분들이 쉽게 접근할 수 있도록 라이브러리를 통한 서킷브레이커 구현을 다루어볼까 합니다.

# Java 서킷브레이커, 히스트릭스
서킷브레이커 패턴을 처음 듣는 사람이 있을지 몰라도 Netflix는 모를 수 없겠죠.
바로 Netflix OSS 중 하나인 히스트릭스가 Java의 대표적인 서킷브레이커 구현체 라이브러리입니다.

![히스트릭스 로고]({{site.url}}/img/circuit-breaker/hystrix.png)

Java를 하지 않는다면 Netflix OSS도 생소할 테고, MSA가 아닌 모노리틱 아키텍처 기반 개발자라도 마찬가지겠죠.
서킷브레이커 패턴을 구현한 히스트릭스가 가진 대표적인 성질은 다음과 같습니다.

1. 장애 대응 상황을 설정하여 문제가 발생하였을 때 규정된 루트에 따라 시스템을 유지한다.
1. 설정된 임계치를 초과할 경우 장애가 발생한 로직을 실행하지 않는다.

이 외에도 대시보드 등 부가 기능을 제공하지만, 히스트릭스는 Java 기반 라이브러리로 PHP에 도입할 수 없습니다.
또한 히스트릭스에 대한 예제는 많아 PHP를 바탕으로 준비하였습니다.
혹시라도 서킷브레이커가 생소하신 분들이 있다면 이전 글([외부 API로 빚어진 장애대응 후일담 after 1years]({{site.url}}/2020-12-18-post-api-with-circuit-breaker))을 참조해주세요.

# PHP 서킷브레이커, 가네샤
패키지스트를 살펴보면 라라벨5.6을 위한 서킷브레이커 라이브러리도 존재하고, 가네샤보다 더 많은 다운로드 수를 가진 오픈소스도 있습니다.
하지만 이 두 가지는 서비스에 도입할 정도로 적합해 보이지 않습니다.
심지어 라라벨 5.6은 이미 보안 지원이 종료된 버전입니다.
최근에 라라벨 8이 공개되었죠.

가네샤보다 많은 다운로드 수를 가진 라이브러리도 수년째 관리되지 않습니다.
코드를 직접 돌려보지는 않았으나, PHP7 혹은 PHP8과 호환되리라 믿는 건 안일한 생각이죠.
심지어 GitHub star는 가네샤가 더 많습니다.
이와 더불어 가네샤는 꾸준히 관리되는 오픈소스이며, 거즐과 연동이 용이하도록 거즐의 미들웨어를 완성된 코드로 지원합니다.
이를 토대로 현시점에 가장 적합한 PHP 서킷브레이커 구현체 라이브러리는 가네샤라는 판단을 내렸습니다.
히스트릭스처럼 대시보드를 제공하는 등의 부가기능은 약하지만, 가네샤를 통해 간단하게 서킷브레이커를 구현할 수 있습니다.
아직 시작 단계이기에 어쩔 수 없죠.
기회가 닿는다면 기여해보고 싶습니다.
마땅찮은 라이브러리가 없다면 직접 오픈소스로 만들려고 했는데, 굳이 그러지 않아도 되겠더군요.
```SATA
Guzzle
   Netflix OSS Feign과 유사한 라이브러리로 HTTP client binder 역할을 수행한다.
   HTTP 요청을 보다 쉽게 처리할 수 있도록 도와주며, 동기와 비동기 전송 모두 지원한다.
```

# 가네샤와 거즐을 통한 MSA 서킷브레이커 예제
본격적으로 PHP 코드를 작성해볼까요?
당연한 이야기지만, 가네샤와 거즐을 사용하기 위해서는 컴포저를 통하여 의존성을 추가해야 합니다.
```console
composer require guzzlehttp/guzzle
composer require ackintosh/ganesha
```
앞에서 이야기한 것처럼 가네샤는 거즐과 연동하기 쉽도록 `GuzzleMiddleware`를 제공합니다.
물론 미들웨어를 사용하지 않더라도 서킷브레이커를 연동할 수 있으나, 굳이 번거로운 과정을 거칠 이유는 없으니까요!
서킷브레이커를 직접 연동하고 싶다면 말리지는 않겠습니다.
서킷브레이커 자체는 어렵지 않으니 한 번 쯤 직접 해보는 것도 나쁘지 않거든요.
이 예제는 MSA 기반 서비스에서 보다 간편하게 서킷브레이커를 도입할 수 있도록 거즐 미들웨어를 통한 가네샤 연동으로 이루어져 있습니다.

## 거즐과 가네샤로 랩핑한 코어 클라이언트
```php
abstract class CircuitBreakerClient extends \GuzzleHttp\Client
{
    abstract public static function of(array $config = []): self;

    protected static function getClientConfig(String $clientName, array $config = []): array
    {
        $config += [
            'base_uri' => 'http://localhost',
            'handler' => self::guzzleHandlerWithGanesha(),
            'ganesha.service_name' => $clientName,
        ];

        return $config;
    }

    private static function guzzleHandlerWithGanesha(): \GuzzleHttp\HandlerStack
    {
        $redis = new \Redis();
        $redis->connect('localhost');

        $ganesha = \Ackintosh\Ganesha\Builder::withRateStrategy()
            ->timeWindow(30)
            ->minumumRequests(10)
            ->failureRateThreshold(50)
            ->intervalToHalfOpen(5)
            ->adapter(new \Ackintosh\Ganesha\Storage\Adapter\Redis($redis))
            ->build();

        $handlers = \GuzzleHttp\HandlerStack::create();
        $handlers->push(new \Ackintosh\Ganesha\GuzzleMiddleware($ganesha));

        return $handlers;
    }
}
```
거즐과 가네샤의 연동을 관리하는 추상클래스`CircuitBreakerClient`입니다.
MSA라면 도메인에 따라 다양한 클라이언트가 생길 수밖에 없습니다.
이를 위하여 공통된 설정을 한 곳에서 관리할 추상클래스를 만들어 보았습니다.
메소드는 크게 거즐 설정을 위한 `getClientConfig()`와 가네샤를 위한 핸들러`guzzleHandlerWithGanesha()`로 나누었습니다.

거즐을 위한 글이 아니므로 거즐만을 위한 설정은 간략하게 표기하였습니다.
`getClientConfig()` 메소드를 보면 `handler`와 `ganesha.service_name`를 볼 수 있는데,  `handler`는 가네샤와 연동할 `GuzzleMiddleware`를 위한 설정입니다.
`ganesha.service_name`는 가네샤에서 서버를 구분하기 위한 유니크한 명칭으로 적으면 됩니다.
큰 의미는 없겠지만, 만약 메소드 별로 구분하고 싶다면 그에 걸맞는 유니크 값을 지정하면 됩니다.
이 설정은 `GuzzleMiddleware`에 지정된 `default name`으로 `key name`은 변경이 가능합니다.
또한 굳이 이 방식이 아니더라도 HTTP 헤더나 다른 방식을 통해 서비스명을 구분할 수 있는 값을 넘길 수 있습니다.

다음으로 `guzzleHandlerWithGanesha()` 메소드를 보면 레디스를 발견할 수 있습니다.
서킷브레이커 상태를 저장하기 위한 스토리지입니다.
가네샤는 어댑터 패턴을 사용하여 다음과 같은 스토리지를 지원합니다.
기본적으로 제공하지 않는 스토리지를 사용하고자 할 경우 어댑터에 맞추어 구현하면 됩니다.

**가네샤, 스토리지 지원 목록**
- 레디스
- 멤캐시드
- 몽고디비

이 외에 가네샤로 설정이 가능한 부분은 다음과 같습니다.

|속성|기본값|단위|설명|
|------|----|---|---------|
|timeWindow|30|초|임계치를 평가하는 시간 범위|
|failureRateThreshold|50|퍼센트|서킷브레이커 상태를 OPEN으로 변경하는 실패 임계치|
|minimumRequests|10|횟수|실패를 감지하기 위한 최소 요청|
|intervalToHalfOpen|5|초|서킷브레이커 상태를 OPEN에서 HALF_OPEN으로 변경하는 간격|

> failureRateThreshold를 초과하더라도 최소 요청에 미치지 않는다면 서킷브레이커는 CLOSE로 유지한다.

## 일반 클라이언트
```php
class MemberClient extends \CircuitBreakerClient
{
    const CLIENT_NAME = 'memberClient';

    private function __construct(array $config = [])
    {
        parent::__construct($config);
    }

    public static function of(array $config = []): self
    {
        return new self(self::getClientConfig(
            self::CLIENT_NAME
        ));
    }

    public function getMember(string $memberId): \Member
    {
        $url = "/members/{$memberId}";

        try {
            return new \Member($this->get($url)->getBody());
        } catch (\Ackintosh\Ganesha\Exception\RejectedException $e) {
            ... // todo opened circuit-breaker process
        } catch (\Exception $e) {
            ... // todo closed circuit-breaker process
        }
    }
}
```

`MemberClient`에서 도메인 서버를 구분할 수 있는 `CLIENT_NAME`을 지정하였습니다.
이후 `getMember()` 메소드를 보면 `RejectedException`을 살펴볼 수 있는데, 서킷브레이커 상태가 `OPEN`일 때 미들웨어에서 이 에러를 던지게 되어 있습니다.
이 부분을 붙잡아 서버를 호출할 수 없을 때를 대비한 예외 처리를 심어놓으면 됩니다.
어렵지 않죠? 물론 이렇게 매번 `try/catch`를 이용하면 번거로울뿐더러 중복 코드가 발생하므로 이를 감싼 메소드를 만들어서 사용하기를 권합니다.
예시 코드라 직관적으로 볼 수 있게 만든 거지, 실제 운영 코드를 이렇게 사용하면 반복되는 코드가 많아 피곤해집니다.

## 클라이언트 호출
```php
$member = \MemberClient::of()->getMember('devellany');
print_r($member->info());
// ['user_name' => 'devellany', 'jobs' => 'back-end developer', 'blog' => 'https://dico.me']
```

클라이언트 호출은 간단합니다.
서킷브레이커가 `OPEN` 상태라면 별도로 지정한 프로세스가 실행될 것이며, `CLOSE`라면 문제없이 데이터를 반환합니다.
만약 이보다 앞서서 서킷브레이커 상태를 확인하고 싶다면 다음과 같이 활용할 수도 있습니다.
해당 API에 직접적인 의존 관계를 가지는 페이지라면 이처럼 불필요한 호출을 전부 차단할 수 있습니다.

```php
if (!$ganesha->isAvailable('memberClient')) {
    die('memberClient 서비스에 접근할 수 없습니다.');
}

$member = \MemberClient::of()->getMember('devellany');
print_r($member->info());
```

만약 이런 식으로 컨트롤러에서 제어하고 싶다면 가네샤를 사용하기 용이하도록 팩토리 패턴으로 구현해두는 게 좋겠죠?

```php
if (GaneshaFactory::of()->isUnavailable(\ClientServer::MEMBER_SERVICE)) {
    die(\ClientServer::MEMBER_SERVICE . '에 접근할 수 없습니다.');
}
```

## 상태 변경 시 이벤트 호출
서킷브레이커를 사용하는 이유는 다른 도메인에 장애가 확산되는 것을 막기 위해서이기도 하지만, 장애가 발생하였을 때 빠르게 인지하여 대응할 시간을 벌기 위함입니다.
이에 따라 서킷브레이커 상태가 언제 바뀌었는지 인지할 수 있어야 합니다.
가네샤는 이를 위하여 상태가 변경되었을 때 이벤트를 호출합니다.
가네샤를 설정할 때 `subscribe()`를 지정하면 이벤트가 발생하였을 때 전달받을 수 있습니다.
이를 통해 로깅`logging`을 할 수도 있고, 사내 공유를 신속하게 이루어낼 수 있습니다.

```php
// @see CircuitBreakerClient::guzzleHandlerWithGanesha()
$ganesha->subscribe(function ($event, $service, $message) {
    (new \MonitoringSystem())->record($event, $service, $message);
});
```
**가네샤, 이벤트 종류**
- Ganesha::EVENT_TRIPPED: 서킷브레이커 상태가 `OPEN`으로 변경될 때 호출
- Ganesha::EVENT_CALMED_DOWN: 서킷브레이커 상태가 `CLOSE`로 변경될 때 호출
- Ganesha::EVENT_STORAGE_ERROR: 스토리지 에러 발생 시 호출

# MSA 안정성을 위한 필수적인 선택
분명 서킷브레이커를 도입한다면 서버 자원을 소모할 수밖에 없습니다.
다른 서버에 존재하는 API를 실행할 때마다 매번 서킷브레이커 상태를 판단해야 하며, 서킷브레이커 상태에 대한 기록을 꾸준히 남겨야 합니다.
혹시라도 서버 자원이 그 무엇보다 중요하다면 MSA를 도입하면 안 됩니다.
서버 자원을 기회비용으로 지출하면서까지 업무 효율을 높일 수 있는 서비스만 MSA를 도입해야 합니다.
MSA를 해야만 하는 규모가 아니지만 모노리스가 싫다면 모듈형 모노리스를 권해드립니다.
MSA는 하나의 아키텍처일 뿐이니 이를 고집할 이유는 없습니다. 흐름에 따라 무작정 도입하면 독이 될 뿐입니다.

그럼에도 불구하고 MSA가 적합하여 이를 도입하고자 했다면 도메인 서비스 간 발생할 수 있는 서버 통신 장애에 대한 안전장치를 심어 두어야 합니다.
이 안전장치가 바로 지금껏 소개한 서킷브레이커입니다.
서킷브레이커를 통해 특정 도메인의 장애가 전체 서비스에 퍼져 나가지 않도록 막을 수 있습니다.
서킷브레이커가 존재해야 비로소 MSA에서 장점이라 이야기하는 자유로운 배포도 가능해진다는 이야기입니다.
안전장치가 없이 서비스에 장애가 발생한다면 롤백하는 시간 동안 혹은 서버를 다시 살리는 시간 동안 사용자는 아무런 행동을 취할 수 없게 될지도 모릅니다.
대비가 되어있지 않으면 특정 도메인의 장애가 어떤 식으로 퍼져나갈지 알 수 없습니다.
심할 경우 데이터 정합성이 깨지겠죠.
그러니 사용자를 위한 서비스는 이런 상황을 피할 수 있게 다양한 대비 수단을 도입하여 장애허용시스템을 구축해야 합니다.

분명 MSA를 도입하는 곳이라면 이미 작지 않은 규모일 것이라 예상합니다.
그 정도까지 명성을 쌓아 올렸다면 사용자의 신뢰를 잃지 않도록 장애 상황을 대비해놓아야 합니다.
사람인은 생명을 책임지는 의료 서비스가 아니지만, 입사 지원 여부에 따라 개인의 인생에 커다란 파문을 일으킬 수 있는 채용 플랫폼을 운영하고 있습니다.
다른 서비스도 무언가의 변화를 이끌어낼 수 있는 서비스라 봅니다.
오너십을 지닌 개발자라고 말하려면 자신의 손으로 만든 결과물에 대한 안정성을 보장해야 하지 않을까요?
**서킷브레이커가 당신이 짊어질 책임을 덜어줄 것입니다.**

# 참고 자료
- [PHP 서킷브레이커 가네샤](https://github.com/ackintosh/ganesha)
- [거즐 라이브러리](https://docs.guzzlephp.org/en/stable)
- [패키지스트에 등록된 서킷브레이커 구현체들](https://packagist.org/?query=Circuit%20Breaker)
- [Netflix OSS 히스트릭스](https://github.com/Netflix/Hystrix)
