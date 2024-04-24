---
title: 도커 swarm을 이용한 서비스 런칭하기
layout: post
comments: true
social-share: true
show-avatar: true
subtitle: 서비스 런칭을 위해 준비했던 것들과 경험하고 있는 것들을 공유합니다.
bigimg: "/img/swarm/docker-swarm-hero2.png"
tags:
- docker
- swarm
---

## Foreword
안녕하세요. 사람인 SM팀의 [박요안](https://yoanp.github.io/)입니다.  
3월 저희 회사에서 조용하게 런칭한 [재능마켓 서비스](https://www.otwojob.com)를 구축하며 경험한 것들을 공유하려고 합니다.

---

## 시작은
목표는 간단(?)했습니다.

* **새로운 기술을 도입하자.**
* 서비스 배포/롤백 작업을 최대한 단순하게 진행되도록 하자.
* 서비스 안정성은 최대한 확보하자.
* ~~런칭 후 내가 편하자~~

---

## 왜 Docker swarm 인가
사람인에서는 사용하지 않는 새로운 기술, 서비스 배포와 롤백을 더 편리하게 하기 위하여 컨테이너 도입을 고려하게 되었습니다.  
해당 컨테이너를 자동으로 관리하고 서비스를 유연하게 가지고 갈 수 있는 오케스트레이션 툴을 고민하게 되었는데요.  
아래와 같은 swarm의 장점들로 인하여 swarm을 사용하기로 결정하였습니다.

* docker 공식 API를 지원한다.
* etcd를 활용한 자체 DNS를 통한 service discovery 가 구현되어 있다.
* 다른 아이들보다 상대적으로 쉽고 편하다..

---

## 그럼 어떻게 구성되어 있는가

![docker_swarm1](/img/swarm/docker_swarm1.png)

* 하나의 로드밸런서(HAProxy)가 swarm 서버군을 연결하고 있습니다.  [관련링크](https://docs.docker.com/engine/swarm/ingress/#/configure-an-external-load-balancer)
* ( swarm 서버군은 3대의 마스터 노드를 갖고 있습니다.

>  docker swarm은 Raft 알고리즘을 활용한 고가용성 기능을 제공합니다. [관련링크](https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/)

* 마스터 노드에 [ dockercloud-haproxy](https://github.com/docker/dockercloud-haproxy) 서비스가 올라가 있습니다.  
해당 컨테이너는 proxy라는 이름의 ingress network에 80,443을 올리고 있으며 해당 포트로 들어올 경우 설정에 맞게 특정 도메인/호스트에 따라 패킷을 분배하여 줍니다.


```bash
docker network create -d overlay proxy
docker service create \
--name haproxy \
--network proxy \
--mount target=/var/run/docker.sock,source=/var/run/docker.sock,type=bind \
--mount source=/home/noticepage,destination=/noticepage,type=bind \ # errorfile 을 위하여 에러페이지가 있는 폴더를 바인딩합니다.
-p 80:80 \
-p 443:443 \
-p 1936:1936 \ # HAProxy stat 페이지
-e OPTION='accept-invalid-http-request' \
-e EXTRA_DEFAULT_SETTINGS='errorfile 503 /noticepage/error.html, errorfile 502 /noticepage/error.html' \
-e DEFAULT_SSL_CERT="$(awk 1 ORS='\\n' /home/cert.pem)" \
--constraint "node.role == manager" \ # 관리자 노드에서 실행되도록 합니다.
dockercloud/haproxy:1.6.3
```


* work 노드에 실제 컨테이너들이 올라가 있습니다.  
서비스 크기에 따라 work 노드는 계속 추가가 가능하며, work 노드는 관리자 노드가 관리하고 있습니다.  
work 노드에 여러 서비스들이 올라가 있습니다.

---

## [ingress network](https://docs.docker.com/engine/swarm/ingress/)

![docker_swarm3](/img/swarm/docker_swarm3.png)

ingree 가상 네트워크에 서비스가 포트를 오픈 할 경우 어떤 노드에 요청을 보내도 해당 요청을 컨테이너에 전달 해 줍니다.  
한마디로 첫번째 LB를 타고 넘어온 요청은 swarm의 ingress network의 cloudhaproxy 서비스의 컨테이너로 전달이 됩니다.  
해당 서비스는 요청에 따라서 특정 백엔드로 요청을 전달해 주게 됩니다.


---

## dockercloud/haproxy
중간에 dockercloud/haproxy 라는 서비스가 올라가있는 것을 보실 수 있습니다.  
이후 많은 마이크로 서비스들을 추가해야 할 것을 고려하여 선택한 아이로 swarm을 지원하는 하나의 리버스 프록시 서버입니다.  
Traefik, nginx 등이 있지만 제가 dockercloud/haproxy를 선택한 이유는 아래와 같습니다.  

* HAProxy는 사람인에서 자체 운영 중인 아이
* 자체 health-check 설정을 이용하여 WEB 과 WAS 간의 통신까지 확인하여 에러 최소화 (swarm에서 따로 가능합니다.)
* swarm 지원


---

## 서비스 배포/롤백하기

![docker_swarm4](/img/swarm/docker_swarm4.png)

CI/CD는 젠킨스를 이용하고 있습니다.  
배포를 진행하게 되면 아래의 작업이 순차적으로 이루어집니다.  

* 소스 빌드
* (변경 된 소스를 적용한) 도커 이미지 빌드
* 이미지 Registry에 저장
* (Registry의 이미지를 사용하여) 서비스 배포

서비스 배포는 현재 단순하고도 확실하게 쉘 스크립트를 이용하여 진행됩니다. (by JENKINS)  
롤링 업데이트가 이루어지며 현재는 WAS가 올라오는시간을 생각하여 업데이트 간격을 설정해 둔 상태입니다. (물론 2차 LB에서 WAS에 대한 HEALTH CHECK도 하고 있습니다.)

```bash
docker service update --image $REGISTRY/$TAG $SERVICE
```

```bash
root # docker service ps $SERVICE
ID            NAME       IMAGE                                     NODE                           DESIRED STATE  CURRENT STATE            ERROR  PORTS
7vlb1cu6qbd8  $SERVICE.1      $REGISTRY/$IMG:0.0.2  work_node1  Running        Running 12 seconds ago
rzcxfc41mo08   \_ $SERVICE.1  $REGISTRY/$IMG:0.0.1      work_node1  Shutdown       Shutdown 12 seconds ago
0k3ip87qcjjr  $SERVICE.2      $REGISTRY/$IMG:0.0.2  work_node2  Running        Running 9 seconds ago
tfusgp1t6jxq   \_ $SERVICE.2  $REGISTRY/$IMG:0.0.1      work_node2  Shutdown       Shutdown 10 seconds ago
```

롤백 또한 젠킨스에서 docker 명령어를 통하여 진행되도록 설정되어 있습니다.

```bash
docker service update --rollback $SERVICE
```

```bash
root # docker service ps $SERVICE
ID            NAME       IMAGE                                     NODE                           DESIRED STATE  CURRENT STATE           ERROR  PORTS
lg5beh0oisbr  $SERVICE.1      $REGISTRY/$IMG:0.0.1      work_node1  Ready          Ready 1 second ago
7vlb1cu6qbd8   \_ $SERVICE.1  $REGISTRY/$IMG:0.0.2  work_node1  Shutdown       Running 1 second ago
rzcxfc41mo08   \_ $SERVICE.1  $REGISTRY/$IMG:0.0.1      work_node1  Shutdown       Shutdown 3 minutes ago
h3fbeid2ad2l  $SERVICE.2      $REGISTRY/$IMG:0.0.1      work_node2  Running        Running 1 second ago
0k3ip87qcjjr   \_ $SERVICE.2  $REGISTRY/$IMG:0.0.2  work_node2  Shutdown       Shutdown 2 seconds ago
tfusgp1t6jxq   \_ $SERVICE.2  $REGISTRY/$IMG:0.0.1      work_node2  Shutdown       Shutdown 3 minutes ago
```

> 도커 swarm 서비스는 UPDATE를 하게 되면 이전 정보(PreviousSpec)를 저장하고 있습니다.
> 따라서 간단한 명령어를 통해 기존의 설정을 그대로 불러오게 됩니다.

```bash
root # docker service inspect $SERVICE
    [
        {
            "ID":"0bu6jys88xmtcj53m2hnvwyrdt",
            "Version":{...},
            "CreatedAt":"2017-03-17T19:24:23.716225053Z",
            "UpdatedAt":"2017-04-13T05:11:44.382149283Z",
            "Spec":{...},
            "PreviousSpec":{...},
            "Endpoint":{...},
            "UpdateStatus":{..}
        }
    ]
```

---

## 도입 후
3월 18일 런칭 후 현재 특별한 문제 없이 정상적으로 서비스를 하고 있습니다. (다행..)  
물론 구축하는 과정은 우여곡절이 많았습니다.  
컨테이너를 이용한 서비스가 처음이었기 때문에 이미지 생성부터 배포까지 기존 방식과는 조금 다르게 하지만 결과는 동일하게 만들어야하는 각각의 과정들이 말이지요.  

아직 docker API 또한 완벽하게 활용하지 못하고, 로그 분석 또한 Promethus/Grafana를 통한 1차적인 분석 밖에 못하고 있지요.  

하지만 이렇게 구축 및 서비스 런칭 이후 개발팀에서 저희팀에게 크게 문의 할 필요 없이 기본적인 시스템 작업이 가능해졌으며 확장성, 안정성을 추가로 확보 할 수 있게 되었습니다.  
또한 하나의 클러스터링 풀을 구성하였기 때문에 앞으로 여러 작은 서비스들을 운영하는데 활용 할 수 있게 되었습니다.  


---

## 후기
글을 쓴다는 것은 참으로 어렵네요..  

감사합니다.