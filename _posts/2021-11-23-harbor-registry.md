---
title: Private Docker Registry를 바꿔보자!
author: 윤진주
subtitle: 사람인 Docker-Registry
tags:
- Docker-Registry
- Container-Registry
- Harbor
- k8s
---

# Private Docker Registry를 바꿔보자!

## 소개

안녕하세요. 사람인HR IT연구소 SE팀 윤진주 입니다.
SE팀은 사람인에서 운영되는 전체 서비스가 보다 안정적으로 운영될 수 있도록 운영되는 모든 시스템을 관리하고 있습니다.
전체 서비스의 안정적인 운영을 위해 모든 서버들을 오픈소스를 이용하여 이중화 구축 진행하고 있으며 시스템을 보다 안정적으로 관리하기 위해 자체 개발한 모니터링 시스템들을 개선해 나가고 있습니다. 
또한 업무의 효율화를 위해 시스템 관리 자동화 등을 지속적으로 진행하고 있습니다.


## 도입목적

사람인에서 사용하고 있는 Docker Registry를 Harbor Registry로 바꿔보겠다고 팀에 이야기 했었는데 핑계아닌 핑계로 차일피일 미루고 있었습니다. 그리고 점차 컨테이너 운영 서비스는 많아지는데 기존에 구축된 Docker Registry는 싱글 서버로 구성되어 있었고, 인증절차 또한 없어서 누구나 도메인만 알게되면 접속해서 이미지를 볼수 있는 상황이었습니다. 또한 사용량이 많아지다보니 로컬 스토리지로 구축된 도커 레지스트리 용량은 종종 임계치를 넘어 이미지를 지우거나 수동으로 Garbage Collection 작업을 진행하는등 여러가지 불편한 점이 많았습니다.

이에 따라 충분한 사용량 확보 및 보안성 향상 그리고 사용자 친화 WEB UI 등 많은것들이 현재 사용하는 Docker Registry보다 향상된 Harbor Registry에 대한 니즈를 더이상 미룰수 없었습니다.

## Harbor란 무엇인가?

공식 홈페이지에서는 Harbor를 다음과 같이 설명하고 있습니다.

> Harbor is an open source registry that secures artifacts with policies and role-based access control, ensures images are scanned and free from vulnerabilities,
and signs images as trusted.Harbor, a CNCF Graduated project, delivers compliance, performance, and interoperability to help you consistently and securely manage artifacts
across cloud native compute platforms like Kubernetes and Docker.

(Harbor는 정책 및 역할 기반 액세스 제어로 아티팩트를 보호하고 이미지를 스캔하고 취약성이 없는지 확인하며 이미지를 신뢰할 수있는 것으로 서명하는 오픈 소스 레지스트리입니다.
CNCF Graduated 프로젝트인 Harbor는 규정 준수, 성능 및 상호 운용성을 제공하여 Kubernetes 및 Docker와 같은 클라우드 네이티브 컴퓨팅 플랫폼에서 아티팩트를 일관되고 안전하게
관리 할 수 있도록 지원합니다.)

### Harbor에서 제공하는 핵심 기능들
![Harbor기능](/img/harbor/harbor-function.PNG)
<center> 출처: https://goharbor.io/ </center>
<br><br>
### Harbor의 시스템 아키텍처
![Harbor아키텍처](/img/harbor/harbor-architecture.PNG)
<center> 출처: https://github.com/goharbor/harbor/wiki/Architecture-Overview-of-Harbor </center> 
<br><br>
## 구현

Harbor 를 배포하기 위해서 다양한 방법들이 존재합니다. 가장 쉽게 할수 있는 방법은 Docker Compose를 통해 할수 있습니다.
다만 사람인의 경우 Kubernetes 클러스터 구성이 되어 있기 때문에 Kubernetes 클러스터에 Pod 형태로 배포 하였습니다.

스토리지의 경우 다른 IT기업 기술 블로그 등을 확인해 보니 Dynamic Provisioning 지원이 되는 MinIO, Chef등을 연동하여 사용하고 있었는데요 사람인의 경우 NetAPP Trident를 활용하여 구축했습니다. Trident를 고려하게 된 계기는 온프레미스 환경에서 Kubernetes 인프라가 구축되어 있기도 하지만, Trident가 Cloud 환경에서도 안정적으로 지원되기 때문에 추후 확장성 등을 고려하여 결정하게 되었습니다.

### Trident 란?

Trident의 경우 NetAPP에서 유지 관리하는 오픈소스 프로젝트로써  CSI (Container Storage Interface)와 같은 업계 표준 인터 페이스를 사용하여 컨테이너형 애플리케이션의 지속성 요구를 충족할 수 있도록 처음부터 설계되었습니다. Trident는Kubernetes 클러스터에 Pod 형태로 배포하고 Kubernetes를 위한 Dynamin Provisioning을 제공합니다.

### Trident 설치

설치는 공식 문서를 확인해 보니 의외로 간단했습니다.

```bash
# 설치
 helm install <name> trident-operator-version
 
 # 업그레이드
 helm upgrade <name> trident-operator-version
 
 # 확인
 root # kubectl get pod -n trident-system
NAME                                READY   STATUS    RESTARTS   AGE
trident-csi-6c9fc4766d-krjxl        6/6     Running   6          68d
trident-csi-92dnh                   2/2     Running   2          68d
trident-csi-nb7k9                   2/2     Running   2          68d
trident-csi-v47zc                   2/2     Running   2          68d
trident-operator-6598486bf7-6lmjq   1/1     Running   1          68d
```
<br>
### Harbor 설치

위에서 언급한데로 Helm 으로 쉽게 설치를 진행했습니다.

#### 1. Harbor Namespace 추가
```bash
root # kubectl create ns harbor-system
```

#### 2. Harbor Repo 추가 및 업데이트
```bash
root # helm repo add harbor https://helm.goharbor.io
"harbor" has been added to your repositories
 
root # helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "harbor" chart repository
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "rancher-latest" chart repository
Update Complete. ⎈Happy Helming!⎈
```

#### 3. SSL 적용을 위한 시크릿 생성
```bash
kubectl -n harbor-system create secret tls tls-harbor-ingress \
  --key=domain.key \
  --cert=domain.crt
 ```

#### 4. 배포
```bash
root# helm install harbor harbor/harbor \
    --namespace harbor-system \
    --set expose.type=ingress \
    --set expose.tls.enabled=ture \
    --set expose.tls.certSource=secret \
    --set expose.tls.secret.secretName=tls-harbor-ingress \
    --set expose.tls.secret.notarySecretName=tls-harbor-ingress \
    --set expose.ingress.hosts.core=sri-registry.saraminhr.co.kr \
    --set expose.ingress.hosts.notary=sri-notary.saraminhr.co.kr \
    --set portal.replicas=2 \
    --set core.replicas=2 \
    --set jobservice.replicas=2 \
    --set registry.replicas=2 \
    --set chartmuseum.replicas=2 \
    --set notary.server.replicas=2 \
    --set notary.signer.replicas=2 \
    --set persistence.enabled=true \
    --set persistence.resourcePolicy=keep \
    --set persistence.persistentVolumeClaim.registry.storageClass=StorageClassName \
    --set persistence.persistentVolumeClaim.registry.accessMode=ReadWriteMany \
    --set persistence.persistentVolumeClaim.registry.size=1Ti \
    --set persistence.persistentVolumeClaim.chartmuseum.storageClass=StorageClassName \
    --set persistence.persistentVolumeClaim.chartmuseum.accessMode=ReadWriteMany \
    --set persistence.persistentVolumeClaim.chartmuseum.size=10Gi \
    --set persistence.persistentVolumeClaim.jobservice.storageClass=StorageClassName \
    --set persistence.persistentVolumeClaim.jobservice.accessMode=ReadWriteMany \
    --set persistence.persistentVolumeClaim.jobservice.size=50Gi \
    --set persistence.persistentVolumeClaim.database.storageClass=StorageClassName \
    --set persistence.persistentVolumeClaim.database.accessMode=ReadWriteMany \
    --set persistence.persistentVolumeClaim.database.size=50Gi \
    --set persistence.persistentVolumeClaim.redis.storageClass=StorageClassName \
    --set persistence.persistentVolumeClaim.redis.accessMode=ReadWriteMany \
    --set persistence.persistentVolumeClaim.redis.size=10Gi \
    --set persistence.persistentVolumeClaim.trivy.storageClass=StorageClassName \
    --set persistence.persistentVolumeClaim.trivy.accessMode=ReadWriteMany \
    --set persistence.persistentVolumeClaim.trivy.size=10Gi \
    --set externalURL=https://DOMAIN \
    --set harborAdminPassword='패스워드'
```

#### 5. 배포확인
```bash
root # kubectl get pod -n harbor-system
NAME                                           READY   STATUS    RESTARTS   AGE
harbor-harbor-chartmuseum-8459fbbdcb-7bvdj     1/1     Running   0          5h58m
harbor-harbor-chartmuseum-8459fbbdcb-9vqsq     1/1     Running   0          5h58m
harbor-harbor-core-54484d9ff7-dl4xt            1/1     Running   0          5h58m
harbor-harbor-core-54484d9ff7-wxtgf            1/1     Running   0          5h58m
harbor-harbor-database-0                       1/1     Running   0          6h9m
harbor-harbor-jobservice-598d8bbbd4-j722t      1/1     Running   0          5h57m
harbor-harbor-jobservice-598d8bbbd4-q8zv2      1/1     Running   0          5h58m
harbor-harbor-notary-server-74f8f87f97-2d9hr   1/1     Running   0          5h58m
harbor-harbor-notary-server-74f8f87f97-lg8xl   1/1     Running   0          5h58m
harbor-harbor-notary-signer-75f6545cd9-6hxwh   1/1     Running   0          5h58m
harbor-harbor-notary-signer-75f6545cd9-gwtbh   1/1     Running   0          5h58m
harbor-harbor-portal-5b4dbcc4c8-nk4zl          1/1     Running   0          6h10m
harbor-harbor-portal-5b4dbcc4c8-xvmtr          1/1     Running   0          6h9m
harbor-harbor-redis-0                          1/1     Running   0          6h9m
harbor-harbor-registry-67f45587d7-44vcq        2/2     Running   0          5h58m
harbor-harbor-registry-67f45587d7-6gvpg        2/2     Running   0          5h58m
harbor-harbor-trivy-0                          1/1     Running   0          5h57m
```
<br>
정상적으로 배포가 완료되면 아래와 같이 로그인 화면을 확인할 수 있습니다.

![Harbor로그인](/img/harbor/harbor-login.PNG)
<br><br>
### Harbor LDAP 연동

Harbor를 도입하려고 한 이유중 가장 중요한 이유가 LDAP 연동을 통한 RBAC (Role Based Access Control) 를 하기 위함이었습니다.
기존 Docker Registry의 경우 계정 연동이 되어 있지 않고, URL만 입력하면 바로 레지스트리 이미지들이 노출되었기 때문에
보안성 측면에서 취약한 부분이 있었기 때문에 이를 개선하기 위해 LDAP 연동을 통해 RBAC 설정을 해보고자 합니다.

#### 1. Harbor RBAC는?
> 자세한 사항은 [Harbor GitHub](https://github.com/goharbor/harbor/blob/26905baca2df158437792010274d2c257b69a47f/docs/permissions.md) 참고 부탁드립니다.
 * ProjectAdmin: Master
 * Master: Image Clone, Project Clone
 * Developer: Project Read/Write
 * Guest: Project Read Only, Image Pull Only
 * Limited Guest : 프로젝트에 대한 권한 X, Image Pull Only

![HarborRBAC](/img/harbor/harbor-rbac.PNG)
<center> 출처 : https://goharbor.io/docs/2.2.0/administration/managing-users </center>
<br>
#### 2. LDAP 설정

아래와 같이 설정하면 LDAP 계정으로 로그인이 가능하며, 로그인시 유저가 자동으로 생성됩니다.
그룹의 경우는 수동으로 생성을 해줘야 하며, 필요에 따라서 설정하면 될것 같습니다.

![HarborLDAP](/img/harbor/harbor-ldap.PNG)
<br>
### Harbor 업그레이드

Helm 차트로 배포했기 때문에 업그레이드 명령으로 쉽게 버전 업그레이드가 가능했습니다.
위에 설치에서 자세한 내용을 기재했기 때문에 업그레이드 내용은 별도 기재하지 않겠습니다.


### 기존 레지스트리 데이터를 어떻게 이관했을까?

개발자들이 기존 도커 레지스트리를 사용하고 있었기 때문에 어떻게 데이터를 이관해야 할지 기능적인 부분으로 확인했었는데  Harbor는 리플리케이션이라는 아주 편리한 기능을 제공하고 있습니다. 그래서 기존 레지스트리 데이터를 하루 2회정도 스케쥴을 통해  데이타 전체  동기화를 진행했습니다.

그리고 전체 프로젝트를 한번에 이관하기에는 리스크가 존재하기  때문에  IT전략팀, 채용시스템개발팀, 서비스인프라개발팀과 협업하여 먼저 개발/스테이징 전환 테스트를 순차적으로 진행하였으며, 프로덕션의 경우 사용하는  프로젝트를 순차적으로 신규 레지스트리로 이관하는 작업을 진행했습니다.

이런 방식으로 진행했기 때문에 별도의 서비스 다운없이 안정적으로 레지스트리 이관을 진행할 수 있었습니다.

## 글을 마치며

올해 상반기 파일럿을 진행하고 도입하는 과정에서 여러가지 시행착오도 있었고, 하반기 이관을 진행하기 위해 사람인 시스템 환경에서 어떻게 하면 좀더 효율적이고 편하게 사용할 수 있을까 많은 고민들을 했습니다. 또한 도입하는 과정에서 각 개발자들과 소통을 하며, 많은 도움 또한 받았습니다.

초기 파일럿 시점부터  테스트 및 실제 도입까지 많은 문제를 해결 할 수 있도록 도와주고 작업을 지원해주신 IT전략팀, 채용시스템개발팀, 서비스인프라개발팀 개발자 분들께 감사드립니다.

그리고 글을 끝까지 읽어주신 모든 분들께 감사드립니다.


## 참고자료
- https://goharbor.io/docs/2.2.0/administration/
- https://github.com/goharbor/harbor
- https://netapp-trident.readthedocs.io/en/stable-v21.07/#
