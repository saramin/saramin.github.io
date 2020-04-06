---
title: 'Elastic APM 적용 해보기 - #1. 소개 및 설치'
layout: post
comments: true
social-share: true
show-avatar: true
tags:
- APM
- elastic
- application
- monitoring
bigimg: "/img/apm/apm_bg.png"
author: 임재호
---

사람인HR에서 Elastic APM을 적용한 경험에 대해서 공유하고자 합니다.

```SATA
안녕하세요. 사람인HR IT연구소 서비스인프라개발팀 임재호입니다.
사람인HR에서 서비스 모니터링을 위한 APM시스템을 구축 하였습니다.
시스템을 구축하며 조사한 자료에 대해서 공유 하고자 합니다.
```

목차<br/>
**[1. APM 소개](#1)<br/>
[2. APM 설치 및 구성](#2)<br/>
[3. Kibana 화면을 통한 APM 데이터 확인](#3)<br/>
[4. 도입시 장점 및 기대 효과]({{site.url}}/2020-03-24-elastic-apm-2/#4){:target="_blank"}<br/>
[부록. Elastic vs Newrelic의 비교]({{site.url}}/2020-03-24-elastic-apm-2/#add){:target="_blank"}<br/>**
#####  *<< Elasti APM 도입시의 장점 및 Newrelic과의 비교는 [2편을 참고]({{site.url}}/2020-03-24-elastic-apm-2){:target="_blank"}해 주세요>>*
***
<a name="1"></a><br/>
## 1. APM 소개
저희 팀에서는 여러 APM 솔루션 중 다음과 같은 이유로 Elastic APM을 선정하여 사용하였습니다. 
- 팀내에 이미 ELK 스택에 대한 사용 경험이 있음
- Basic License(무료 사용)만으로 APM의 전체적인 기능을 사용 가능
- 이미 Metric, Log에 대한 모니터링 시스템을 구축하여 사용하고 있음

Elastic에서 APM뿐아니라 Metric, Log에 대한 All-In-One 시스템을 제공해 주고 있어서 모니터링이 필요하신 분들을 한번쯤 사용해 보면 좋은 기회가 될것 같습니다.

### 1-1. APM 이란
APM은 Application Performance Monitoring의 약어로,  Application에 대한 **성능정보** 및 발생한 **Error정보** 그리고 Application이 동작중인 서버의 기본적인 **Metric정보**를 수집 할 수 있는 기능을 지원합니다.<br/>
또한 MicroService 환경에서 서비스를 구성하는 여러 Application간의 Request를 하나의 Trace로 묶어서 추적 할 수 있는 **분산 Tracing**(distributed tracing)에 대한 기능도 지원합니다.<br/>
APM은 위와 같이 수집된 여러 데이터를 바탕으로 하여  Application에 지연이 발생하였을때  **지연에 대한 병목 구간**을 찾아 낼 수 있는 모니터링 서비스 입니다.

### 1-2. APM 시스템 구성
APM에서 데이터를 수집하기 위해서는 Agent와 서버가 모두 필요하며 아래와 같은 시스템으로 구성됩니다.

![Elastic APM 시스템 구성도]({{site.url}}/img/apm/apm_archtecture.png)

Elastic APM은 데이터를 수집 하기 위한 APM Agent와 수집된 데이터의 가공을 위한 APM Server로 구성됩니다<br/>
APM Server를 통해 수집된 데이터는 최종적으로 Elasticsearch에 적제되어 Kibana의 APM UI를 이용하여 조회 가능 합니다.
- APM Agent
	- 실시간으로 Performance 및 error 데이터를 수집 하여 APM Server로 전달
	- APM 서버 연동 실패시 데이터 저장을 위한 memory buffer가 존재 함
- APM Server
	- APM Agent에서 수집된 데이터에 대한 유효성 체크
	- 수집된 데이터를 Elasticsearch의 Document 포멧으로 변환
- Elasticsearch
	- APM 데이터에 대한 저장, 검색, 분석을 지원
	- 성능 데이터에 대한 집계 기능 제공
- Kibana 
	- APM UI에서 데이터 필터링 및 Service, Trace, Trasaction, Error, Metric에 대한 Overview 및 상세  기능 제공
	- Dashboard/Visualization을 이용하여 사용자가 원하는 형태로 데이터 가시화 가능
	- APM Agent Configuration, Machine Learning 통합(일부 License 필요) 기능 지원 

### 1-3. Elastic 지원 APM Agent
Elastic에서는 다양한 언어로된 APM Agent를 지원하고 있습니다. <br/>
지원하는 언어들에서는 APM을 위한 별도의 Agent나 연동 모듈을 개발하지 않고 사용가능 합니다.
- [Go agent](https://www.elastic.co/guide/en/apm/agent/go/1.x/introduction.html)
- [JAVA agent](https://www.elastic.co/guide/en/apm/agent/java/1.x/intro.html)
- [.NET agent](https://www.elastic.co/guide/en/apm/agent/dotnet/1.x/intro.html)
- [Node.js agent](https://www.elastic.co/guide/en/apm/agent/nodejs/3.x/intro.html)
- [Python agent](https://www.elastic.co/guide/en/apm/agent/python/5.x/getting-started.html)
- [Ruby agent](https://www.elastic.co/guide/en/apm/agent/ruby/3.x/introduction.html)
- [JavaScript RUM(Real User Monitoring) agent](https://www.elastic.co/guide/en/apm/agent/rum-js/5.x/intro.html)
> JavaScript RUM : RUM은 Browser에서 실행되는 성능 데이터 및 metric을 수집 하는 것으로 아래의 정보를 수집함
>   - Page Load Metric
>   - 정적 데이터 Load 시간
>   - API Request(XMLHttpRequest and Fetch)

###  1-4. APM 데이터 모델
APM에서 사용되는 성능 데이터는 Transaction, Span, Error, Metric, Metadata와 같은 데이터 모델로 관리 됩니다. <br/>
아래는 데이터 모델 중  Transaction, Span의 관계를 좀더 쉽게 이해하기 위한 그림입니다.
![]({{site.url}}/img/apm/distributed_tracing.png)

**1. Transaction**

- Application내에서 측정되는 최상위 작업 (ex. Server의 Request, Batch Job, Background Job 등)
- 데이터 구성
	- Event Timestamp
	- ID, type, name
	- Event가 생성된 환경에 대한 데이터

| 환경 | 환경별 수집 데이터|
| --- | --- |
|Server|environment, framework, language, etc.|
|Host| architecture, hostname, IP, etc.|
|Process | args, PID, PPID, etc.|
|URL | full, domain, port, query, etc.|
|User | (if supplied) email, ID, username, etc.|
		
**2. Span**

- 실행을 추적하기 위한 논리 작업 단위
- 실행되는 Code의 시작 및 소요시간 정보가 있음
- span 사이의 부모/자식 관계를 가질 수 있음
- 데이터 구성
	- **transaction.id**, parent.id
	- start time, **duration**
	- name, type, stacktrace

**3. Error**

- Application에서 발생한 Error Exception정보 및 로그
- 데이터 구성
	- **Error 발생위치**
	- Stacktrace 정보
		- Error Exception
		- Error log
	- transaction.id
	- Error Event가 생성된 환경에 대한 데이터

	| 환경 | 환경별 수집 데이터|
	| --- | --- |
	|Server|environment, framework, language, etc.|
	|Host| architecture, hostname, IP, etc.|
	|Process | args, PID, PPID, etc.|
	|URL | full, domain, port, query, etc.|
	|User | (if supplied) email, ID, username, etc.|
		
**4. Metric**

- Agent Host에 대한 CPU, Memory 등의 기본 Metric 정보를 자동으로 수집 
- JAVA의 JVM Metric이나 Go의 런타임 Metric도 수집 가능

**5. Metadata**

- Event에 대한 부가적인 정보
- 종류
	- Label

	|속성|설명|
	| --- | --- |
	|Indexed| Yes|
	|Elasticsearch type| object|
	|Elasticsearch field|labels|
	|Apply |Transactions \| Spans \| Errors|

	- Custom context

	|속성|설명|
	| --- | --- |
	|Indexed| No|
	|Elasticsearch type| object|
	|Elasticsearch field|transaction.custom \| error.custom|
	|Apply |Transactions \| Errors|

	- User context

	|속성|설명|
	| --- | --- |
	|Indexed| Yes|
	|Elasticsearch type| keywords|
	|Elasticsearch field|user.email \| user.name \| user.id|
	|Apply |Transactions \| Errors|

<a name="2"></a><br/>
## 2. APM 설치 및 구성
APM 시스템 구축을 위한 서버 및 Agent(java, php)를 설치 하는 방법에 대해서 간단히 설명하도록 하겠습니다.

### 2-1. APM 서버 설치 및 설정
APM 서버는 Elasticsearch 버전과 호환되는 버전으로 설치 하면됩니다.<br/>
현재 Elasticsearch 7.3.2를 사용중 이므로 APM Server도 7.3.2를 설치 하도록 하겠습니다.<br/>
APM 서버를 설치하는 방법은 다양하지만 저희는 kubernetes 환경을 구축하여 사용중 이기 때문에 kubernetes 환경 기반의 설치를 진행하였습니다.<br/>
(참고 : [Elastic APM Server Install 방법](https://www.elastic.co/guide/en/apm/server/current/installing.html))

**1. APM 서버 설치**

APM 서버 설치를 위한 Deployment 및 NodePort Service YAML
```
apiVersion: apps/v1
kind: Deployment
metadata:
	labels:
		app.kubernetes.io/name: apmserver
	name: apm-server
spec:
	replicas: 1
	selector:
		matchLabels:
			app.kubernetes.io/name: apmserver
	template:
		metadata:
			labels:
				app.kubernetes.io/name: apmserver
		spec:
			containers:
			- name: apmserver
				image: docker.elastic.co/apm/apm-server:7.3.2
				ports:
				- containerPort: 8200
					name: http
					protocol: TCP
				resources: {}
				volumeMounts:
				- mountPath: /usr/share/apm-server/apm-server.yml
					name: apmserver-config
					readOnly: true
					subPath: apm-server.yml
			volumes:
			- configMap:
					defaultMode: 420
					name: apmserver-config
				name: apmserver-config
---
apiVersion: v1
kind: Service
metadata:
	name: apmserver-nodeport
spec:
	type: NodePort
	selector:
		app.kubernetes.io/name: apmserver
	ports:
	- name: apmserver-nodeport
		nodePort: 30082
		port: 8200
		protocol: TCP
		targetPort: 8200
```
APM 서버의 설정 파일은 별도의 Kubernetes ConfigMap을 이용하여 설정 합니다.

**2. APM 서버 설정 파일 설치**

APM 서버 설정 ConfigMap YAML
```
apiVersion: v1
kind: ConfigMap
metadata:
	labels:
		app.kubernetes.io/name: apmserver
	name: apmserver-config
data:
	apm-server.yml: |-
		apm-server:
			host: "0.0.0.0:8200"
		output.elasticsearch:
			hosts: [ "<<elasticsearch domain>>:9200" ]
			index: "apm-%{[observer.version]}-%{[processor.event]}"
```
Elasticsearch로 저장을 위한 output 설정을 해주어야 합니다.<br/>
(APM Server config에 대한 상세 정보 : [Elastic APM Server Config](https://www.elastic.co/guide/en/apm/server/current/configuring-howto-apm-server.html))

**3. APM 서버 설치 확인**

Kubernetes에 YAML을 적용하고 "http://<< kubernetes node ip >>:30082"로 접속하면 아래와 같은 결과 화면을 확인 할 수 있습니다.

![]({{site.url}}/img/apm/apmserver_result.png)

데이터가 정상적으로 조회 되면 APM Server의 설치가 완료된 것입니다.
	

### 2-2. APM Agent 설치
APM Agent는 Application의 library형태로 제공되기 때문에 Application이 개발된 언어에 맞는 Agent를 설치 해야 합니다.<br/>
Elastic에서 공식적으로 지원하는 Agent는 설치 후 별다른 설정을 하지 않아도 Request/Response, HTTP 연동, DB 연동등에 대한 성능 정보를 수집 할 수 있습니다.

#### 2-2-1. JAVA Agent에 대한 설치 및 설정 방법
Elastic APM JAVA Agent를 설치 하는 방법은 manual, automatic, programmatic 세가지 방법이 존재 하지만 해당 문서에서는 manual방법에 대해서만 설명합니다.<br/>
다른 설치 방법에 대해서는 아래 참고 내용을 확인 부탁드립니다.<br/>
(참고 : [Elastic APM JAVA Agent Install 방법](https://www.elastic.co/guide/en/apm/agent/java/1.x/setup.html))

**1. 설치를 위한 APM Agent jar 파일 다운로드**

```
## 수동으로 jar 파일을 다운로드함
##   - maven repository에서 다운로드
다운로드 링크 : https://search.maven.org/search?q=g:co.elastic.apm%20AND%20a:elastic-apm-agent
```
	
**2. Application 실행시 "-javaagent" 옵션을 설정 하여 APM Agent를 설치**

```
java -javaagent:<download>/<path>/elastic-apm-agent.jar 
		-Delastic.apm.server_urls=http://<< kubernetes node ip >>:30082
		-Delastic.apm.service_name=demoappjava-es
		-Delastic.apm.environment=dev
		-Delastic.apm.application_packages=com.example
	 -jar demoappjava.jar
```
실행시 설정한 Command Line Parameter는 아래에 정리 하였습니다.

| 항목 | 설명 | 참고 링크 |
| -- | -- |--|
| service_name | Application의 APM서비스 이름 (kibana ui에 서비스 구분자로 사용됨) | [service_name 상세](https://www.elastic.co/guide/en/apm/agent/java/1.x/config-core.html#config-service-name)|
| environment | Application의 실행 환경 정보 (kibana ui에서 필터링 항목을 사용가능) | [environment 상세](https://www.elastic.co/guide/en/apm/agent/java/1.x/config-core.html#config-environment)|
| application_packages | Application의 최상위 Package 이름(Trace하는 Frame의 in-app구분을 위해 사용, 이외는 library로 구분됨)| [application_package 상세](https://www.elastic.co/guide/en/apm/agent/java/1.x/config-stacktrace.html#config-application-packages)|
	
**3. 추가 설정 항목**

기본적인 설정 항목 이외에도 Method 단위의 성능 데이터 수집이나 APM 수집 데이터 양을 조절을 위한 다양한 설정 항목이 있습니다.<br/>
전체 설정 항목은 [Elastic APM JAVA Agent 설정 문서](https://www.elastic.co/guide/en/apm/agent/java/1.x/configuration.html)를 참고 하시면 됩니다.

| 항목 | 설명 | default값 | 참고 링크 |
| -- | -- |--| --|
| transaction_sample_rate | Transaction 데이터 수집 비율로 1.0 ~ 0.0까지 설정 가능 (샘플링에서 제외된 Transaction은 수집 되고 해당 Transaction의 Span만 수집 되지 않음) | 1.0 | [transaction_sample_rate 상세](https://www.elastic.co/guide/en/apm/agent/java/1.x/config-core.html#config-transaction-sample-rate) |
| trace_methods | Transaction 데이터 수집을 위한 method 목록("\*"를 이용하여 다중 설정 가능 )<br/> ex) \* org.example.MyClas\*#myMe\*(\*.String, int[]) | N/A | [trace_methods 상세](https://www.elastic.co/guide/en/apm/agent/java/1.x/config-core.html#config-trace-methods) |
| stack_trace_limit | StackTrace를 수집할 Frame 최대 개수 (많은 frame을 수집하면 APM 데이터 수집 성능에 영향이 있을 수 있음) | 50 |[stack_trace_limit 상세](https://www.elastic.co/guide/en/apm/agent/java/1.x/config-stacktrace.html#config-stack-trace-limit) |
| disable_instrumentations | APM으로 계측이 불필요한 모듈에 대한 설정 | incubating | [disable_instrumentations 상세](https://www.elastic.co/guide/en/apm/agent/java/1.x/config-core.html#config-disable-instrumentations) |

**4. JAVA Agent 동작 원리**

JAVA Agent는 APM 성능 데이터 수집을 위해 필요한 Code들을 [Byte Buddy](https://bytebuddy.net/#/)라는 library를 이용하여 Compile된 Application Byte Code에 직접 삽입합니다.<br/>
따라서, Agent를 Command Line Parameter로 설정만 해주면 별도의 BizLogic 수정 없이 APM 성능 데이터 수집이 가능 합니다. 
![Byte Buddy]({{site.url}}/img/apm/byte_buddy.png)

#### 2-2-2. PHP를 위한 APM Agent 설치
Elastic APM에서 공식적으로 지원하는 PHP Agent는 존재 하지 않습니다.<br/>
그러나 Elastic Engineer인 ["Philip Krauss"](https://github.com/philkra)가 개발하여 공개한 [PHP Agent library](https://github.com/philkra/elastic-apm-php-agent)가 있습니다.<br/>
해당 문서에서는 "Philip Krauss"가 개발한 Agent library를 이용하여 PHP Application에 Elastic APM을 적용하는 방법을 설명합니다.

먼저 Agent library에서 제공하는 기능에 대해서 살펴 보면 아래와 같습니다.

- APM Agent 사용을 위한 객체 생성 및 Config 설정
- Transaction 데이터 전송을 위한 코드 작성 및 데이터 설정 기능 제공
- Span 데이터 전송을 위한 코드 작성 및 데이터 설정 기능 제공
- Transaction에 Span binding 기능 제공
- Exception을 기반으로 이용하여 Error 데이터를 capture 할 수 있는 기능 제공
- 외부 연동 시 HTTP Header에 Tracing과 관련된 데이터를 생성 해주는 기능 제공 (W3C의 Trace 규격 수용)
	(단, Request Header에 삽입은 별도의 코딩 필요)
		
**1.   composer를 이용한 Agent library 설치**

```
## "php.ini" 파일에 "curl" extension 추가 필요 (아래는 개인 환경에 따라 다름)
vi ../php.ini
	extension="php_curl.so"

## composer.json 파일 수정 후 composer update
## APM Server가 7.3.2 이므로 elastic-apm-php-agent library도 호환 가능한 7.0.0-RC1 이상의 버전 설치 필요
vi ../conposer.json
	"require" : {
				...
				"philkra/elastic-apm-php-agent" : ">=7.0.0-RC1" 
		},
```
- 참고 자료 : [설치 방법 참고 링크](https://github.com/philkra/elastic-apm-php-agent/blob/master/docs/install.md), [php library 다운로드 링크](https://packagist.org/packages/philkra/elastic-apm-php-agent)


**2. Agent 생성 및 Config 설정**

```
	$this->agent = new Agent( [
	 'appName' => 'demoappphp-es',                             ## apm application의 서비스 이름 (kibana ui에 서비스 구분자로 사용됨)
	 'environment' => 'dev',                                    ## application 구동 환경
	 'serverUrl' => 'http://<< kubernetes node ip >>:30082',     ## APM Server URL
	]);
```


**3. Transaction 및 Span 생성**

PHP인 경우에는 Application의 성능을 측정 하고자 하는 위치의 Source Code에 직접 Transaction과 Span 생성을 위한 Code를 작성해 주어야 합니다.
```
public function test()
	{
			/* for APM - start*/
			$trxName = get_class($this).' - test';
			$transaction = $this->agent->startTransaction( $trxName );                       # Transaction 생성

			$span = $this->agent->factory()->newSpan('SELECT php_user', $transaction);     # span 생성
			$span->setType('db.mysql');                                                      # span의 type 지정
			$span->start();                                                                  # span 시작
			/* for APM - end*/

			.... (Application 코드)

			/* for APM - start*/
			$span->stop();                                                                   # span 종료
			$this->agent->putEvent($span);                                                    # span을 Event로 등록 
																																								#    - Transaction이 종료되는 시점에 Transaction과 Span 데이터가 한번에 APM 서버로 전송됨

			$this->agent->stopTransaction($transaction->getTransactionName(),[          # Trasaction 종료
					'result' => 'HTTP 2xx',
					'type' => 'request'
			]);
			/* for APM - end*/

			return $return_value;
	}
```


**4. Exception Capture**

Exception 정보는 captureThrowable Method를 이용하여 쉽게 수집 할 수 있습니다.
```
	try {
	.... (Application 코드)
}
catch (Exception $e) {
	 /* for APM - start*/
	 $this->agent->captureThrowable($e, [], $transaction);             # Exception Capture
	 /* for APM - end*/
}
```
	
**5. 외부 Application으로 HTTP 연동시 Trance ID를 전달**

Application 외부 연동시 분산 trace를 위한 값을 HTTP Header로 전달<br/>
([W3C의 trace 규격](https://www.w3.org/TR/trace-context/#header-name)을 따르고 있음)
```
$request = $client->request('GET', $url, [
 ## Http Header 값으로 tarce 정보를 전달
 // 'headers' => [DistributedTracing::HEADER_NAME => $span->getDistributedTracing()]
 ## TraceFlag를 00 --> 01로 전달하도록 임의로 header value 조정 (현재 php-agent library에서는 TraceFlag 변경 기능을 지원하지 않음)
 ## Elastic APM에서 분사 Tracing을 위해서는 Flag값을 01로 설정해 주어야 함
 'headers' => [DistributedTracing::HEADER_NAME => substr($span->getDistributedTracing(),0,-3).'-01']    
]);
```

<a name="3"></a><br/>
## 3. Kibana 화면을 통한 APM 데이터 확인
kibana Basic License를 설치 하면 APM에 대한 데이터를 확인 할 수 있는 APM 전용 UI Plug-in이 같이 설치가 되며, Kibana를 접속하면 아래의 화면과 같이 해당 메뉴를 확인 할 수 있습니다.
![]({{site.url}}/img/apm/kibana_menu.png)

APM 메뉴를 클릭 후 Service를 선택하면 Transaction, Error, Metric등을 확인 할 수 있는 메뉴들이 있습니다.
##### Transaction 확인
Transaction을 선택하였을 경우 Transaction에 대한 상세 내용을 확인 할수 있는 화면 예시 입니다.
[<img src="{{site.url}}/img/apm/transaction_detail.png" />]({{site.url}}/img/apm/transaction_detail.png){:target="_blank"}

##### Error 확인
Error 메뉴를 선택하였을 경우 Transaction에 대한 상세 내용을 확인 할수 있는 화면 예시 입니다.
[<img src="{{site.url}}/img/apm/error_detail.png" />]({{site.url}}/img/apm/error_detail.png){:target="_blank"}

##### Metric 확인
Metric 메뉴를 선택하였을 경우 Transaction에 대한 상세 내용을 확인 할수 있는 화면 예시 입니다.
[<img src="{{site.url}}/img/apm/metric_detail.png" />]({{site.url}}/img/apm/metric_detail.png){:target="_blank"}

<b>*[<< Elasti APM 도입시의 장점 및 Newrelic과의 비교는 2편 에서 계속 됩니다>> ]({{site.url}}/2020-03-24-elastic-apm-2)*</b>