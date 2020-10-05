---
title: 'Elastic APM 적용 해보기 - #2. 장점 및 Newrelic 비교'
layout: post
comments: true
social-share: true
show-avatar: true
tag:
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
**[1. APM 소개]({{site.url}}/2020-03-24-elastic-apm-1/#1){:target="_blank"}<br/>
[2. APM 설치 및 구성]({{site.url}}/2020-03-24-elastic-apm-1/#2){:target="_blank"}<br/>
[3. Kibana 화면을 통한 APM 데이터 확인]({{site.url}}/2020-03-24-elastic-apm-1/#3){:target="_blank"}<br/>
[4. 도입시 장점 및 기대 효과](#4)<br/>
[부록. Elastic vs Newrelic의 비교](#add)<br/>**
#####  *<< Elastic APM  소개 및 설치와 관련된 내용은 [1편을 참고]({{site.url}}/2020-03-24-elastic-apm-1){:target="_blank"}해 주세요>>*
***

<a name="4"></a><br/>
## 4. 도입시 장점 및 기대 효과
APM을 도입하게 되면 Application을 운영할때 다음과 같은 장점이나 기대 효과를 가질 수 있습니다.

**1. Trasation에 대한 지연 발생 인지 가능**

|  단계 | 설명 | 예시 이미지 |
| -- | -- |--|
|1단계|Transaction Overview 화면에서 지연 인지|[<img src="{{site.url}}/img/apm/b_transaction_01.png" />]({{site.url}}/img/apm/b_transaction_01.png){:target="_blank"}|
|2단계|시간을 많이 차지하는 Span을 확인|[<img src="{{site.url}}/img/apm/b_transaction_02.png" />]({{site.url}}/img/apm/b_transaction_02.png){:target="_blank"}|
|3단계|지연 발생한 Request에 대한 상세 확인|[<img src="{{site.url}}/img/apm/b_transaction_03.png" />]({{site.url}}/img/apm/b_transaction_03.png){:target="_blank"}|
|4단계|Application내에 지연이 발생한 병목을 확인|[<img src="{{site.url}}/img/apm/b_transaction_04.png" />]({{site.url}}/img/apm/b_transaction_04.png){:target="_blank"}|
	
**2. Request 유입의 급증/급감에 대한 인지 가능**

|  단계 | 설명 | 예시 이미지 * |
| -- | -- |--|
|1단계|Request 개수의 급격한 증가를 확인|[<img src="{{site.url}}/img/apm/b_request_01.png" />]({{site.url}}/img/apm/b_request_01.png){:target="_blank"}|
|2단계|급증한 Request를 확인|[<img src="{{site.url}}/img/apm/b_request_02.png" />]({{site.url}}/img/apm/b_request_02.png){:target="_blank"}|

**3. Application에서 발생한 Error에 대한 확인 편리**

|  단계 | 설명 | 예시 이미지 |
| -- | -- |--|
|1단계|시간별 Error 발생 건수 확인|[<img src="{{site.url}}/img/apm/b_error_01.png" />]({{site.url}}/img/apm/b_error_01.png){:target="_blank"}|
|2단계|Error에 대한 상세 메시지 및 발생 위치 확인|[<img src="{{site.url}}/img/apm/b_error_02.png" />]({{site.url}}/img/apm/b_error_02.png){:target="_blank"}|

**4. 이상 접근 사용자를 인지 가능**

|  설명 | 예시 이미지 |
| -- | -- |
|Application의 Request Metadata(Header 정보)를 이용하여 Application에 요청하는 Client 정보를 확인 가능 잘못된 Client의 접근을 바로 발견|[<img src="{{site.url}}/img/apm/b_bad_access_01.png" />]({{site.url}}/img/apm/b_bad_access_01.png){:target="_blank"}|

<a name="add"></a><br/>
* * * *
### *부록. Elastic vs Newrelic의 비교*<br/>

현재 사내의 일부 서비스에 APM 솔루션 으로 Newrelic이 적용되어 있습니다.<br/>
Newrelic APM과 Elastic APM을 간략하게 비교해 보았습니다.<br/>
비교는 자체 개발한 demo application을 기준 일부 기능에 대해서만 진행하였으며, 아래의 내용은 Application의 상태나 사용기능에 따라 달라 질 수 있습니다.

**1. 지원 하는 Agent 종류**

Elastic APM과 Newrelic APM 모두 지원하는 언어의 종류는 비슷하며 Elastic에서는 Go와 JavaScript(RUM)을 추가로 지원하고 있습니다.

| Newrelic APM Agent 지원 언어	| Elastic APM 지원 언어  |
| -- | -- |
| ![]({{site.url}}/img/apm/newrelic_support.png){: width="500" hight="500"} | ![]({{site.url}}/img/apm/elastic_support.png) |

**2. 데이터 확인 지원 기능**

APM 데이터를 조회 및 분석할 수 있는 사용자 화면(UI)은 Elastic에서 일부 지원 하지 못하는 기능이 있으나 Elastic은 Kibana의 Visualization을 이용하여 사용자가 원하는 데로 Dashboard를 생성하여 사용할 수 있습니다.<br/>
따라서 부족한 부분은 raw 데이터를 이용하여 요구사항에 맞게 처리 가능 합니다. 

| 항목 | Newrelic APM 지원 |  Elastic APM 지원 |
| -- | -- |--|
| Transaction Percent 통계 | 지원 | 지원
| Throughput 통계 | 지원 | 지원|
| Transactions 정보 | 지원 | 지원 | 
| Reponse Time 통계 | 지원 | 지원|
| Error Rate 통계 | 지원 | 지원 |
| Errror Detail | 지원 | 지원 |
| Transaction 소요 시간 별 통계 | 미지원  | 지원 |
| Trace Details | 지원 | 지원|
| DB 별 통계<br/>(쿼리 시간,  Throughput)| 지원 | 미지원<br/> (kibana Visualization 기능으로 구현가능) |
| 외부 연동별 통계 | 지원 | 미지원<br/>(kibana Visualization 기능으로 구현가능) |
| Service Map| 지원 | 미지원|
| Host Metric | 지원 | 지원|
| Reports | 지원 | 미지원|
	
**3. JAVA, PHP Demo Application을 이용한 TPS, 평균 응답 시간 비교**

간단한 Demo Application(수집 Transaction/Span 5개 이내)에  Elastic과 Newrelic APM을 모두 적용하여 성능 테스트를 진행해 보았습니다.<br/>
해당 Case의 테스트에서는 Newrelic을 적용했을때 보다 Elastic을 적용했을때 좀 더 나은 성능을 보였습니다.<br/>
(Application의 평균응답시간이 2~3초 정도로 Biz Logic에서 지연이 있는 서비스인 경우에는 Elastic 과 Newrelic 적용에 따른 성능 저하가 크지 않으며, 둘간의 성능 차이도 비슷하였습니다.)<br/>

![]({{site.url}}/img/apm/elastic_vs_newrelic.png)
*Demo Application 기능 : DB에 저장된 사용자 정보(이름, 나이) 조회 API*


### 마무리
이상으로 사람인HR 서비스를 위해 Elastic APM을 조사하고  적용하며 알게된 내용들에 대한 간략한 정리를 마치고자 합니다.<br/>
마지막으로 APM 시스템을 구축 하면서 느낀 부분에 대해서 말씀드리겠습니다.<br/>

APM을 적용하여 유의미한 데이터를 뽑아 내기 위해서는 기본적으로 수집되는 Request/Response, HTTP, DB 성능 이외의 어떤 Method를 관제 해야 할지 로직을 분석하는 작업과 APM에서 수집 하는 성능 데이터 중 무의미 하다고 판단되는(ex. LB의 서버 상태 체크 기능과 같이 biz로직과 무관하거나 성능 측정의 의미 없는 로직 등)부분을 추려내고 제외 시켜 검증 하는 단계가 꼭 필요하다고 생각됩니다.<br/>

APM으로 데이터를 수집해 보니 수집되는 데이터의 저장 용량이 적지 않았습니다.(1일 기준 수십GB ~ 수백 GB 정도)<br/>
수집되는 데이터가 예상되는 저장 용량을 over 하는 경우에는 아래의 설정을 적용하면 데이터의 양을 줄이는 작업을 하는데 도움이 될 수 있습니다.

- stacktrace의 frame("stack_trace_limit" 설정)을 서비스에 맞는 적절한 수치로 조정 하여 불필요한 저수준 library의 frame을 제외
- Transaction의 rate("transaction_sample_rate" 설정) 조정을 통해 수집되는 span 데이터의 양을 줄임 
		(단, Transaction rate를 조정하더라도 Transaction에 대한 정보는 모두 수집되고 sampling되지 않은 Transaction의 span 정보만 수집 되지 않음

Application에 APM을 적용하면 약간의 성능 저하는 불가피하게 발생하겠지만, 서비스 운영시 장애 상황을 빠르게 인지 할 수 있고 서비스의 품질을 높일 수 있는 포인트(병목)를 찾는데 많은 도움이 될 수 있을것으로 생각됩니다.


*출처 및 참고 사이트*<br/>

- Elastic  APM : [https://www.elastic.co/kr/apm](https://www.elastic.co/kr/apm)
- Elastic APM Overview : [https://www.elastic.co/guide/en/apm/get-started/7.6/transactions.html](https://www.elastic.co/guide/en/apm/get-started/7.6/transactions.html)
- Elastic APM Server : [https://www.elastic.co/guide/en/apm/server/current/index.html](https://www.elastic.co/guide/en/apm/server/current/index.html)
- Elastic JAVA Agent : [https://www.elastic.co/guide/en/apm/agent/java/1.x/index.html](https://www.elastic.co/guide/en/apm/agent/java/1.x/index.html)
- Elastic PHP Agent : [https://github.com/philkra/elastic-apm-php-agent](https://github.com/philkra/elastic-apm-php-agent)
- Newrelic APM : [https://newrelic.com/products/application-monitoring](https://newrelic.com/products/application-monitoring)