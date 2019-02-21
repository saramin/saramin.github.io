---
title: 도커 swarm을 이용한 서비스 런칭하기
layout: post
comments: true
social-share: true
show-avatar: true
subtitle: 서비스 런칭을 위해 준비했던 것들과 경험하고 있는 것들을 공유합니다.
bigimg: "/img/docker-swarm-hero2.png"
tags:
- docker
- swarm
---

## Foreword

안녕하세요. 사람인 SM팀의 박요안입니다.  
3월 저희 회사에서 조용하게 런칭한 재능마켓 서비스를 구축하며 경험한 것들을 공유하려고 합니다.

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

![docker_swarm1](/img/docker_swarm1.png)