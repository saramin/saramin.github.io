---
title: 분산처리 시스템 구조개선 (1)
layout: post
comments: true
social-share: true
show-avatar: true
author: 박동형
tags:
- MSA
- Docker
- Docker Swarm
- Apache Kafka
- KStream
- Spring Cloud Stream
bigimg: "/img/distributed/big_image_v2.png"
---

**구조개선을 하려 했던 이유**와 **전체 시스템 설계에 대한 설명, 주요 기술, 빌드, 배포**에 대해 공유

```STATA
안녕하세요. 사람인HR IT연구소 서비스인프라개발팀 박동형입니다.  
이 포스트는 저희가 2년여간 운영해온 분산처리 시스템의 구조개선 프로젝트에 대해서  
시스템 구축에 사용한 기술과 경험을 공유하고자 합니다
```


<br>

## 구성  
  
1. [분산처리 시스템 구조개선 (1)]({{ site.url }}/2019-02-25-msa-cloud-system-1/){: target="_blank" }
2. [분산처리 시스템 구조개선 (2)]({{ site.url }}/2019-02-28-msa-cloud-system-2/){: target="_blank" }


---

<br>

## 왜 구조개선을 생각하게 됐는가?

![engine]({{site.url}}/img/distributed/engine.png)  
엔진 아키텍처
{: .text-center }
<br>
![old_tpas]({{site.url}}/img/distributed/old_tpas.png)  
시스템 흐름도
{: .text-center }

구조개선 하려는 시스템은 IMDG 플랫폼 [Apache Ignite](https://ignite.apache.org/index.html){: target="_blank" } 기반으로 클러스터링한 
Daemon으로 구동되는 대용량 Push 메시지 생성 시스템입니다.  
그래서 시스템의 중심인 Engine에 의해 모든 프로세스가 진행되는 상황으로 캠페인 관리, Job 분배 및 실행, 
결과에 대한 summary 처리를 스케줄 실행 시점의 마스터 노드가 관리했습니다.

성능적 이슈는 없었지만 운영 이슈 및 어려움들이 존재했고 다음과 같이 정리할 수 있습니다.

1. Engine 장애시 모든 기능 마비
2. 클러스터링의 많은 메모리 요구로 노드 전체가 약 80%의 불필요한 메모리 점유율 발생
3. memory기반의  IMDG (In-Memory Data Grid) 특성으로 시스템 장애 상황시 작업에 대한 실패(유실) 발생
4. 통합 프로젝트 버전관리에 따른 반영시 항상 전체 프로젝트에 대한 빌드, 배포, 설치가 진행되는 번거로움
5. 환경별 클러스터링 설정, jsvc 설치, ignite 관련 관제포트 처리 등 노드 추가에 따른 필요 고려사항이 많음


그동안 시스템 운영을 해오면서 좀 더 기능을 분리해서 운영이 쉽도록 개선이 필요하다고 느끼고 있었습니다.  
그래서 시스템 구조개선의 방향성을 잡기 위해 고민을 하게 되었고 몇가지로 정리되었습니다.

> 프로세스의 역할 분리를 통한 프로젝트 운영 메뉴얼 개선  
> 빌드, 배포 프로세스의 개선  
> 장애 상황의 유연한 대처  / 안정성있는 failover 처리  
> Simple of scale out   


Engine에 집중되어 있는 기능을 **도메인으로 분리해서 각각의 역할과 책임을 가지고 운영**할 수 있어야 한다고 생각했습니다.  
그리고 분리된 도메인은 **컨테이너화**를 통한 독립된 어플리케이션으로 실행되고, 도메인간 통신은 kafka를 이용한다면 
**결합성이 낮고 유연하게 운영**이 가능할 것으로 기대했습니다.

물론 Kafka의 선택은 failover 상황에서도 메시지 및 데이터 유실없이 장애대응이 가능하다는  2년간의 운영에서 얻은 신뢰가 있었습니다.

---

<br>

## 아키텍처

<img src="{{site.url}}/img/distributed/image2018-11-7_14_11_52.png" width="400px" title="아키텍처 그림" />
{: .text-center }

당연스럽게도 마이크로서비스 아키텍처를 기반으로 설계가 진행되었고, 분리된 도메인들은 Docker를 사용하여 컨테이너화 했습니다.  

|도메인 |역할|
|---|---|
| `관리자` |캠페인 정보 관리<br>스케줄 정보 설정 및 에이전트 실행시에 필요한 파라미터 정보들을 설정|
| `스케줄러` |실행 스케줄 관리, 실제 스케줄이 등록되고 관리<br>스케줄 시간이 되면 에이전트로 메시지 발행|
| `에이전트` |수신된 메시지를 기반으로 주어진 정보에 맞게 작업을 실행<br>ex) 푸시 메시지 / 메일 생성|

도메인 분리를 통해서 이제 어느 하나에 장애가 발생하더라도 다른 도메인에는 영향이 없어졌으며 빌드, 배포도 쉽게 진행 할 수 있습니다.
모든 통신은 Apache Kafka를 통해서 이뤄지기 때문에 메시지 유실에 대해서도 신뢰를 가질 수 있습니다.  

다음 포스트에서 설명 드리겠지만 에이전트들 간의 데이터 전송은 **Spring Cloud Stream 설정을 통한 kafka 통신**이므로
여러 컨테이너 중 하나가 장애가 나더라도 데이터 유실없이 전송이 가능합니다.  
또한 kstream을 이용한 reduce 처리를 통해 결과에 대한 summary 데이터를 저장했습니다.

시스템 구축에 사용된 주요 기술 스택은 다음과 같습니다.

### 마이크로서비스 아키텍처
<img src="{{site.url}}/img/distributed/uOLtvuo9wxHXyETP_c085A.png" width="240px" title="마이크로서비스 아키텍처" />  
독립적인 작은 단위의 서비스로 분리하는 설계 방식  
분리된 서비스들 간의 역할 분담이 되어 있기 때문에 서비스 간의 결합성 및 영향도를 낮출수 있으며, 반대로 서비스가 많아 질수록
복잡도가 증가하기 때문에 데이터 정합성을 맞추기 위한 고려사항들이 늘어나게 됨

### [Docker](https://docs.docker.com/install/){: target="_blank" }
<img src="{{site.url}}/img/distributed/docker-whale-home-logo.png" width="240px" title="Docker" />  

LXC(Linux Container)를 활용한 격리된 프로세스를 사용한 기술  
가볍고 빠르며, Host OS의 커널을 공유하는 독립적인 실행 환경이기 때문에 자원의 효율성이 장점

### [Spring Cloud Stream](https://docs.spring.io/spring-cloud-stream/docs/current/reference/htmlsingle/){: target="_blank" }
<img src="{{site.url}}/img/distributed/GvuCOnQi.jpg"  width="240px" title="Spring Cloud Stream" style="margin:-24px 0 -48px;" />  
MSA 구축을 위해 필요한 라이브러리 집합인 Spring Cloud에서 메시지 기반 시스템을 구축하기 위한 경량 프레임워크  
RabbitMQ, Apache Kafka 등과 쉽게 연결하여 사용 가능

### [Apache Kafka / Kafka Streams](https://kafka.apache.org/){: target="_blank" }
<img src="{{site.url}}/img/distributed/kafka-logo-no-text.png" width="140px" title="apache kafka" />  
메시지 큐, 메시지 시스템과 유사한 분산 스트리밍 플랫폼  
내결함성으로 메시지 전송의 안정성을 보장  
실시간 스트리밍 데이터 파이프라인 및 실시간 데이터 처리하는 스트리밍 애플리케이션 구축이 가능

### [Portainer](https://www.portainer.io/){: target="_blank" }
<img src="{{site.url}}/img/distributed/portainer_main_v2.png" title="portainer" />  
Docker Image 및 Container 등 관련 기능을 간편하게 사용 할 수 있게 제공하는 UI Tool  
오픈소스이며 docker로 설치 가능, Docker swarm 노드들의 제어를 위해선 portainer agent를 추가로 설치하면 관리 가능합니다.

---

<br>

## Docker Image 빌드, 배포
<img src="{{site.url}}/img/distributed/image2019-1-17_14_2_57.png" title="docker deploy" />  
Docker 이미지 빌드, 배포 흐름도
{: .text-center }
<br>
[<img src="{{site.url}}/img/distributed/docker_image_build_v3.png" title="jenkins docker deploy" />]({{site.url}}/docker_image_build_v3.png){: target="_blank" }  
Jenkins Plugin을 이용한 Docker 이미지 빌드 및 registry push
{: .text-center }

소스 및 docker 이미지 빌드, 배포는 Jenkins에서 진행했으며 Docker 플러그인을 추가하면 간단히 진행이 가능합니다.  
위 이미지는 저희 개발서버 배포 위한 jenkins Job 구성입니다.

---

<br>

## 서비스 배포
[<img src="{{site.url}}/img/distributed/portainer_service_add_1_v2.png" title="docker service1" />]({{site.url}}/portainer_service_add_1_v2.png){: target="_blank" } 
[<img src="{{site.url}}/img/distributed/portainer_service_add_2.png" title="docker service2" />]({{site.url}}/portainer_service_add_2.png){: target="_blank" }  
구축된 registry로 이미지 배포가 끝나면 portainer 를 이용해 필요한 정보를 입력 후 서비스 구동을 진행할 수 있어서 
기존 시스템처럼 설치 과정이 생략되므로 서비스 배포까지의 시간이 상당히 축소되었습니다.  
배포된 서비스에 대한 반영때는 업데이트 한번으로 재배포 진행이 가능합니다.

---

<br>
## 마무리
새로운 기술 도입은 언제나 즐거우면서도 조심스러운 부분입니다.   
다행히 미리 검증하고 R&D한 기술들을 사용한 부분이 있어서 진행이 수월했던 것 같습니다.   
다음 포스트에선 도메인별 프로세스 설명 및 개선된 점 등의 후기를 전해드리도록 하겠습니다.

감사합니다.

<br><br>