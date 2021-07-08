---
title: Customize GitLab Auto DevOps with Existing Cluster (On-Premise Kubernetes)
tags:
- kubernetes
- k8s
- on-premise
- gitlab-ci
- CI/CD
- DevOps
- GitLab
author: 엄기화
subtitle: 온-프레미스 환경에서 실행되는 쿠버네티스 클러스터에서 깃랩 자동 데브옵스 커스터마이징
meta-description: |-
  최근 (주)사람인HR 은 등용문S 프로젝트에서 GitLab Auto DevOps 기능을 활용하여 쿠버네티스 클러스터로 배포하기 시작했습니다.
  이후 새 프로젝트를 진행하는데 있어 GitLab Auto DevOps 는 소프트웨어 배포에 대한 복잡성을 줄이는데 도움이 되었고, 빠르게 시작할 수 있었습니다.
  이 게시물에서는 GitLab Auto DevOps 기능을 온-프레미스 환경에서 직접 kubespray로 설치한 쿠버네티스 클러스터와 연결하는 방법을 살펴봅니다.
  * Auto DevOps 템플릿을 통해 프로젝트 전반에 걸쳐 일관성을 제공
  * CI / CD 모범 사례  및 도구를 활용하는 Auto DevOps 템플릿에 대한 이해
  * 현대적인 소프트웨어 개발 라이프 사이클의 설정 및 실행을 단순화
  * 코드를 푸시하고 나머지는 GitLab 이 알아서 하도록 하여 생산성과 효율성 향상

  Auto DevOps 시작하기
  1. 클러스터 추가,  서비스 계정 생성
  2. Nginx 인그레스 설치
  3. Kubernetes 기본 도메인 구성
  4. GitLab Runner 설치 및 등록
  5. CI/CD 파이프라인 작성 (.gitlab-ci.yml)
  6. 헬름 차트 커스터마이징 밸류
layout: post
show-avatar: true
social-share: true
comments: true
---

## 소개
최근 (주)사람인HR IT연구소/IT 전략팀에서는 등용문S ([채용홈페이지](/2020-10-01-s/) 및 채용관리자) 프로젝트에서 GitLab Auto DevOps 기능을 활용하여 쿠버네티스 클러스터로 배포하기 시작했습니다.

이후 새 프로젝트를 진행하는데 있어 GitLab Auto DevOps 는 소프트웨어 배포에 대한 복잡성을 줄이는데 도움이 되었고, 빠르게 시작할 수 있었습니다.

이 게시물에서는 GitLab Auto DevOps 기능을 온-프레미스 환경에서 직접 [kubespray로 설치한 쿠버네티스](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubespray/) 클러스터와 연결하는 방법을 살펴봅니다.

단점으로는 외부 인터넷 연결과 관련된 문제가 있는데 다음 게시물 [GitLab Auto DevOps 기능을 사용하면서 빌드 속도를 향상시킨 커스터마이징 내용 소개](/2021-07-01-gitlab-ci-pipeline-efficiency/)를 참고해 주십시오.

이전 게시물 [K8S 인프라 CI/CD](/2020-05-01-k8s-cicd/) 과 유사한 주제를 다루지만 이 게시물의 목표는 다음과 같습니다.

* Auto DevOps 템플릿을 통해 GitLab은 프로젝트 전반에 걸쳐 일관성을 제공
* [CI / CD 모범 사례 ](https://docs.gitlab.com/ee/ci/pipelines/pipeline_efficiency.html)및 도구를 활용하는 Auto DevOps 템플릿에 대한 이해
* 현대적인 소프트웨어 개발 라이프 사이클의 설정 및 실행을 단순화
* 코드를 푸시하고 나머지는 GitLab 이 알아서 하도록 하여 생산성과 효율성 향상


## Auto DevOps 시작하기
1. 클러스터 추가, [서비스 계정 생성](#service-token)
2. [Nginx 인그레스 설치](#인그레스-설치)
3. [기본 도메인 구성](#기본-도메인-구성)
4. [GitLab Runner 설치 및 등록](#GitLab-Runner-설치)
5. [CI/CD 파이프라인 작성](#cicd-파이프라인)
6. [헬름 차트 커스터마이징 밸류](#커스터마이징-밸류)
7. [결론](#결론) (파이프라인 화면 캡처)

<br/>

### 기존 클러스터 추가
Google Kubernetes Engine(GKE) 이나 아마존 EKS 클러스터, 온 프레미스 및 다른 제공업체와 함께 클러스터를 호스팅 할 수 있습니다.


1. 그룹에서 Kubernetes 메뉴로 이동
2. 그룹 클러스터를 추가합니다.  `Add Kubernetes cluster`
3. `Add existing cluster` 탭을 클릭하고 세부 정보를 입력

![Add existing cluster](/img/gitlab-auto-devops/add-existing-cluster.png)


클러스터 추가 입력 양식에서 **More information**{: style="color: #008AFF"} 을 눌러 설명대로 명령을 복사해서 사용하면 됩니다.
#### API URL
명령을 실행하여 API URL을 가져옵니다.
~~~bash
kubectl cluster-info | grep -E 'Kubernetes master|Kubernetes control plane' | awk '/http/ {print $NF}'
~~~
<br/>
#### CA 인증서 
~~~bash
kubectl get secrets
kubectl get secret <secret name> -o jsonpath="{['data']['ca\.crt']}" | base64 --decode
~~~
기본으로 생성된 인증서를 사용합니다. `default-token-xxxxx`

명령이 전체 인증서 체인을 반환하는 경우 루트 CA 인증서와 모든 중간 인증서를 체인 맨 아래에 복사해야합니다. 체인 파일의 구조는 다음과 같습니다.
~~~
 -----BEGIN MY CERTIFICATE-----
 -----END MY CERTIFICATE-----
 -----BEGIN INTERMEDIATE CERTIFICATE-----
 -----END INTERMEDIATE CERTIFICATE-----
 -----BEGIN INTERMEDIATE CERTIFICATE-----
 -----END INTERMEDIATE CERTIFICATE-----
 -----BEGIN ROOT CERTIFICATE-----
 -----END ROOT CERTIFICATE-----
~~~ 
<br/>
#### Service Token

GitLab 은 서비스 토큰(JWT)으로 인증하며 `cluster-admin` 권한이 있어야 합니다.

##### 서비스 계정 생성

`gitlab-admin-service-account.yaml` 파일을 만듭니다 .

~~~yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: gitlab
    namespace: kube-system
~~~

서비스 계정 및 클러스터 역할 바인딩을 클러스터에 적용하고 토큰을 조회합니다.

~~~bash
kubectl apply -f gitlab-admin-service-account.yaml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep gitlab | awk '{print $1}')
~~~

출력 에서 `<authentication_token>` 값을 복사합니다 .
인증이 제대로 되지 않는 경우, 토큰을 복사할 때 줄바꿈 문자가 복사되진 않았는지 확인해 보세요.

~~~
Name:         gitlab-token-b5zv4
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=gitlab
             kubernetes.io/service-account.uid=bcfe66ac-39be-11e8-97e8-026dce96b6e8

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      <authentication_token>
~~~

{% raw %}
{% endraw %}




### 인그레스 설치
클러스터가 실행 된 후 인터넷에서 애플리케이션으로 트래픽을 라우팅하려면 NGINX Ingress Controller를 로드 밸런서로 설치해야합니다.
~~~
kubectl create ns gitlab-managed-apps
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm install ingress stable/nginx-ingress -n gitlab-managed-apps

# Check that the ingress controller is installed successfully
kubectl get service ingress-nginx-ingress-controller -n gitlab-managed-apps
~~~

### 기본 도메인 구성
Auto Review App. 브랜치 별로 개발환경을 만들어 주기 위해 필요합니다.

Auto DevOps에는 기본 도메인과 일치하는 와일드카드 DNS A 레코드가 필요합니다.  기본 도메인이 `example.com`인 경우, 다음과 같은 DNS 항목(entry)이 필요합니다.

~~~
*.example.com   3600     A     1.2.3.4
~~~

`<IP address>.nip.io`와 같은 도메인을 Base 도메인으로 사용할 수도 있습니다.



### GitLab Runner 설치
러너는 일반적 으로 권한 모드가 활성화된(privilleged) Docker 또는 Kubernetes 실행기로 Docker 를 실행하도록 구성되어야 합니다 . 러너는 Kubernetes 클러스터에 설치할 필요가 없지만 Kubernetes 실행기는 사용하기 쉽고 자동으로 확장됩니다. 

[GitLab Runner Helm Chart](https://docs.gitlab.com/runner/install/kubernetes.html)

~~~bash
helm repo add gitlab https://charts.gitlab.io

# For Helm 3
helm install --namespace <NAMESPACE> gitlab-runner -f <CONFIG_VALUES_FILE> gitlab/gitlab-runner
helm upgrade --namespace <NAMESPACE> <RELEASE-NAME> -f <CONFIG_VALUES_FILE> gitlab/gitlab-runner
~~~

#### CONFIG_VALUES_FILE 
- 동시에 4개의 작업을 수행하도록 구성
- docker socket binding 사용 ( `/var/run/docker.sock` 마운트) 

~~~
checkInterval: 3
concurrent: 4
imagePullPolicy: IfNotPresent
metrics:
  enabled: false
rbac:
  clusterWideAccess: false
  create: true
  podSecurityPolicy:
    enabled: false
    resourceNames:
      - gitlab-runner
  rules: []
resources: {}
runners:
  builds: {}
  cache: {}
  config: |
    [[runners]]
      [runners.kubernetes]
        image = "ubuntu:18.04"
        privileged = true
        [[runners.kubernetes.volumes.host_path]]
          name = "docker-sock"
          mount_path = "/var/run/docker.sock"
  helpers: {}
  services: {}
  runUntagged: true
  tags: 'cluster,kubernetes,sri-runner-dev'
secrets: []
securityContext:
  fsGroup: 65533
  runAsUser: 100
terminationGracePeriodSeconds: 3600
tolerations: []
gitlabUrl: 'http://sri-git.saraminhr.co.kr/'
preEntrypointScript: |
  cat /etc/gitlab-runner/config.toml
  echo ---
  cat /home/gitlab-runner/.gitlab-runner/config.toml
runnerRegistrationToken: <registration token>
~~~

gitlab-runner 컨테이너에서 잘 구성되었는지 확인해 볼 수 있습니다.
~~~
root # cat /etc/gitlab-runner/config.toml

concurrent = 4
check_interval = 3
[session_server]
  session_timeout = 1800
[[runners]]
  name = "sri-runner-dev"
  url = "http://sri-git.saraminhr.co.kr/"
  token = "<registration token>"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "ubuntu:18.04"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
~~~


### 로그 및 매트릭 수집
[Heimdall - 로그 수집 관제 시스템](/2020-04-01-heimdall/)을 참고해 주십시오.


## CI/CD 파이프라인

### DevOps 관련 파일 및 디렉터리 구조
<img alt="Ci/CD Related File and Directory Structure" src="/img/gitlab-auto-devops/structure-gitlab-ci.png" style="float: right">

~~~
.gitlab
    ci/
        global.gitlab-ci.yml # 전역 CI/CD Variable
        Laravel.gitlab-ci.yml # node_module 설치 및 빌드
    auto-deploy-values.yaml # 헬름차트 값
.gitlab-ci.yml
Dockerfile # herokuish; use multi-stage build
Procfile # web, release 명령 정의
system.properties # JVM 버전 정의
~~~


* GitLab Community Edition [12.10.10](https://gitlab.com/gitlab-org/gitlab-foss/-/tags/v12.10.10)
* Kubernetes Version v1.19.9 [Kubernetes 1.16+](https://docs.gitlab.com/ee/topics/autodevops/stages.html#kubernetes-116)

#### .gitlab-ci.yml
~~~yaml
stages:
  - pre-build
  # Auto-DevOps Stages
  - build
  - test
  - deploy  # dummy stage to follow the template guidelines
  - review
  - dast
  - staging
  - canary
  - production
  - incremental rollout 10%
  - incremental rollout 25%
  - incremental rollout 50%
  - incremental rollout 100%
  - performance
  - cleanup
  # end of Auto-DevOps Stages

include:
  - template: Auto-DevOps.gitlab-ci.yml
  - local: .gitlab/ci/global.gitlab-ci.yml
  - local: .gitlab/ci/Laravel.gitlab-ci.yml
	
build:
  image: "gitlab-org/cluster-integration/auto-build-image:${AUTO_BUILD_IMAGE_VERSION}"
  tags:
    - sri-runner-dev
production_manual:
  environment:
    name: production
    url: https://demos.applyin.co.kr
~~~

* [Auto-DevOps.gitlab-ci.yml](https://gitlab.com/gitlab-org/gitlab-foss/-/blob/v12.10.10/lib/gitlab/ci/templates/Auto-DevOps.gitlab-ci.yml) 을 기본 템플릿으로 사용하고, production_manual 작업의 경우, `environment.url` 을 변경할 수 있습니다.
	* [Jobs/Build.gitlab-ci.yml](https://gitlab.com/gitlab-org/gitlab-foss/-/blob/v12.10.10/lib/gitlab/ci/templates/Jobs/Build.gitlab-ci.yml) 나 [Jobs/Deploy.gitlab-ci.yml](https://gitlab.com/gitlab-org/gitlab-foss/-/blob/v12.10.10/lib/gitlab/ci/templates/Jobs/Deploy.gitlab-ci.yml)  만 템플릿으로 사용할 수도 있습니다.
* 빌드 전용 gitlab-runner 사용 `sri-runner-dev`
* [auto-build-image:v0.6.0 ](https://gitlab.com/gitlab-org/cluster-integration/auto-build-image/-/tree/v0.6.0)사용 
* [auto-deploy-image:v2.0.2](https://gitlab.com/gitlab-org/cluster-integration/auto-deploy-image/-/tree/v2.0.2) 사용 
* [v2 auto-deploy-image 로 배포 업그레이드](https://docs.gitlab.com/ee/topics/autodevops/upgrading_auto_deploy_dependencies.html) 호환성 참고 (Helm v3)

#### .gitlab-ci.yml
~~~yaml
.auto-deploy:
  image: "gitlab-org/cluster-integration/auto-deploy-image:v2.0.2"
  dependencies: [ ]

variables:
  AUTO_BUILD_IMAGE_VERSION: v0.6.0
~~~

#### Laravel.gitlab-ci.yml

`pre-build` 단계에서 패키지를 설치하고 빌드합니다.  artifacts 로 다음 build 단계에서 Docker 이미지에 포함됩니다.

~~~yaml
alpine yarn build:
  extends:
    - .default-retry
  tags:
    - dyms-build
  image: "node:12.18.3-alpine3.12"
  stage: pre-build
  variables:
    YARN_CACHE_FOLDER: "${CI_PROJECT_DIR}/.cache/yarn"
  script:
    - printenv
    - yarn install
    - yarn prod
  artifacts:
    paths:
      - public/css
      - public/fonts
      - public/js
      - public/vendors~js
      - public/mix-manifest.json
    expire_in: 1 week
  cache:
    key: node-modules-cache
    paths:
      - .cache/yarn
  only:
    - branches
    - tags
~~~


#### Dockerfile
[Dockerfile 기본값](https://gitlab.com/gitlab-org/cluster-integration/auto-build-image/-/blob/master/src/Dockerfile.erb ) 을 참고하여 작성하고 아래와 같은 커스터마이징이 필요할 수 있습니다.

* 쓰기권한 부여
* Self Singed 루트 인증서 설치
* 암복호화 모듈 설치
* 로컬 타임과 타임존 변경
* BuildKit 을 활성화하고 빌드 캐시 사용

~~~dockerfile
# syntax=docker/dockerfile:1.2
FROM gliderlabs/herokuish as builder
COPY . /tmp/app
ARG BUILDPACK_URL
ARG CACHE_PATH
ENV USER=herokuishuser

# https://stackoverflow.com/questions/59017965/how-can-i-store-composer-cache-in-a-volume-during-when-building-a-docker
RUN --mount=type=cache,target=/tmp/cache /bin/herokuish buildpack build

FROM gliderlabs/herokuish

# Self Singed SSL 인증서 설치
ADD ca-cert.crt /usr/local/share/ca-certificates/ca-cert.crt
RUN chmod 644 /usr/local/share/ca-certificates/ca-cert.crt 
RUN apt-get update -y && apt-get install -y ca-certificates-java
RUN update-ca-certificates

COPY --chown=herokuishuser:herokuishuser --from=builder /app /app
RUN chmod o+w /app/storage/framework/*
RUN ln -snf /usr/share/zoneinfo/Asia/Seoul /etc/localtime && echo "Asia/Seoul" > /etc/timezone
ENV PORT=5000
ENV USER=herokuishuser
EXPOSE 5000

CMD ["/bin/herokuish", "procfile", "start", "web"]
~~~

참고로 Auto DevOps 를 활용하면 이렇게 복잡해 지지 않습니다.

- https://github.com/lorisleiva/laravel-docker/blob/master/7.3/Dockerfile

### Procfile

laravel
~~~
web: vendor/bin/heroku-php-nginx -C .gitlab/nginx_dyms-admin.conf -F .gitlab/fpm_custom.conf public/
queue: php artisan queue:work
stop_queue: php artisan queue:restart
schedule: php artisan schedule:work
~~~
* nginx 및 php-fpm 설정을 추가할 수 있습니다.
* 라라벨 스케줄러와 큐를 워커로 실행시킬 수 있습니다.


Spring Application
~~~
web: java -Dserver.port=$PORT $JAVA_OPTS -jar build/libs/*.jar
~~~

## K8S_SECRET_* 및 환경별 CI / CD  Variables
### .env
라라벨(PHP) 의 경우, dotenv 를 사용하는데 환경별 변수를 사용하여 `.env.review` 를 사용하게 할 수 있습니다.

| Type | Key | Value | Protected | Masked | Environments |
| -------- | -------- | -------- |-------- |-------- |-------- |
| Var	| K8S_SECRET_APP_ENV | review	| x	| x	| review/* |
| Var	| K8S_SECRET_APP_ENV | stg	| x	| x	| staging |
| Var	| K8S_SECRET_APP_ENV | prod | x	| x	| production |


### 빌드와 배포에 관련된 환경 변수
추가적인 호스트명이나 운영환경에서 실행될 Pods 수를 정의합니다.

| Type | Key | Value | Protected | Masked | Environments |
| -------- | -------- | -------- |-------- |-------- |-------- |
| Var	| CI_DEBUG_TRACE | true	| x	| x	| All (default) |
| Var	| ADDITIONAL_HOSTS | `*.dev.apply.co.kr`	| x	| x	| review/* |
| Var	| PRODCTION_ADDITIONAL_HOSTS | `*.apply.co.kr`	| x	| x	| All (default) |
| Var	| PRODUCTION_REPLICAS | 6	| x	| x	| All (default) |






### GitLab 커뮤니티 버전에서 여러 클러스터 사용
프로젝트 **Settings > CI/CD > Variables** 에서 , `KUBE_INGRESS_BASE_DOMAIN`, `KUBECONFIG` , 환경별 CI Variable 로 추가하여 `production` 환경인 경우, 운영 클러스터를 사용할 수 있습니다.

* 리뷰로 시작하는 모든 환경은 는 개발 클러스터 사용
* 스테이징 까지 개발 환경 클러스터
* 운영은 운영 클러스터 사용. 증분 롤아웃




| Type | Key | Value | Protected | Masked | Environments |
| -------- | -------- | -------- |-------- |-------- |-------- |
| File	| KUBECONFIG | apiVersion: 1 ...	| v	| x	| production |
| Var	| KUBE_INGRESS_BASE_DOMAIN | dyms-kube.saraminhr.co.kr	| v | 	x	| production

* KUBECONFIG 는 랜처(Rancher)의 Kubeconfig File 을 사용했습니다.
* production 환경인 경우에만 쿠버네티스 클러스터 URL 과 토큰(`KUBECONFIG`)을 변경하여 운영 클러스터를 사용


`K8S_SECRET_*` 은 비밀번호나 토큰을 위해 사용되는 것이 좋을 것 같습니다.

환경별로 변경해야 될 URL 이 많아지다 보니 이렇게 되었습니다.

<img src="/img/gitlab-auto-devops/cicd-variables.png" alt="파이프라인">

## 커스텀 빌드 팩
멀티팩으로 node 와 php 를 빌드할 수도 있으나, 전용 빌드 서버를 따로 구축하였고,  node 에서 빌드한 산출물은 artifacts 로 저장하고, 웹 서버를 실행하는 php 빌드 팩을 사용

* 멀티 팩은 테스트를 지원하지 않습니다.
* 외부 인터넷에 접속되지 않는 상황을 줄이기 위해 github 에서 heroku/heroku-buildpack-php 로 부터  fork 한 [공개 저장소](https://github.com/saramin/heroku-buildpack-php-laravel)를 사내 GitLab 으로 import


## 커스텀 헬름 차트
* 기본 차트 - https://gitlab.com/gitlab-org/cluster-integration/auto-deploy-image/-/blob/v2.0.2/assets/auto-deploy-app/Chart.yaml
* **번들 차트** -  ./chart/Chart.yaml 파일이 존재하면 [기본 차트](https://gitlab.com/gitlab-org/cluster-integration/auto-deploy-image/-/blob/v2.0.2/assets/auto-deploy-app/Chart.yaml)  대신 사용됩니다.
* **Project variable** - `AUTO_DEVOPS_CHART` 값(URL)으로 사용자 차트를 지정할 수 있습니다. 또는 `AUTO_DEVOPS_CHART_REPOSITORY`, `AUTO_DEVOPS_CHART` 로 호스트와 패스로 설정할 수도 있습니다.



### 커스터마이징 밸류
기본 헬름 챠트의 value.yaml 파일을 .gitlab/auto-deploy-values.yaml 로 override 할 수 있습니다.
- https://gitlab.com/gitlab-org/cluster-integration/auto-deploy-image/-/blob/v2.0.2/assets/auto-deploy-app/values.yaml


~~~yaml
# Kubernetes 1.16+
deploymentApiVersion: apps/v1
image:
  pullPolicy: IfNotPresent
ingress:
  tls:
    enabled: false
  modSecurity:
    enabled: true
livenessProbe:
  path: /health
readinessProbe:
  path: /health
postgresql:
  enabled: false
workers:
  worker:
    replicaCount: 1
    terminationGracePeriodSeconds: 60
    command:
      - /start
      - queue
    preStopCommand:
      - /start
      - stop_queue
    livenessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 15
      probeType: "exec"
      command:
        - date
    readinessProbe:
      initialDelaySeconds: 5
      timeoutSeconds: 3
      probeType: "exec"
      command:
        - date
  schedule:
    replicaCount: 1
    terminationGracePeriodSeconds: 60
    command:
      - /start
      - schedule
    livenessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 15
      probeType: "exec"
      command:
        - date
    readinessProbe:
      initialDelaySeconds: 5
      timeoutSeconds: 3
      probeType: "exec"
      command:
        - date
~~~
* workers 는 [Procfile](#Procfile) 에 정의한 작업 입니다.
* HAProxy 를 사용하며, TLS 종료 처리


Spring Boot

~~~yaml
# Kubernetes 1.16+
deploymentApiVersion: apps/v1
image:
  pullPolicy: IfNotPresent
ingress:
  tls:
    enabled: false
livenessProbe:
  path: /actuator/health/liveness
readinessProbe:
  path: /actuator/health/readiness
postgresql:
  enabled: false
~~~
* HAProxy 를 사용하며, TLS 종료 처리
* 헬스 체크 경로 변경



CI/CD 환경 변수 HELM_UPGRADE_VALUES_FILE  로 다른 경로나 이름을 지정할 수 있습니다.

| Type | Key | Value | Protected | Masked | Environments |
| -------- | -------- | -------- |-------- |-------- |-------- |
| File	| AUTO_DEVOPS_CHART | ./chart/Chart.yaml	| v	| x	| All (default) |
| Var	| KUBE_INGRESS_BASE_DOMAIN | dyms-kube.saraminhr.co.kr	| v | 	x	| production




## 결론
GitLab 으로 푸시하면 파이프라인이 트리거 됩니다.
CI / CD 메뉴에서 작업을 확인할 수 있습니다.

토픽 브랜치에서 트리거된 파이프라인은 리뷰 환경 배포 됩니다.

<img src="/img/gitlab-auto-devops/pipeline-review.PNG" alt="파이프라인">

* Clean up / stop_review 가 트리거되면 해당 컨테이너가 내려가고 정상적인 경우 네임스페이스도 삭제됩니다.


Operations > environment 에서 환경별로 이동할 수도 있고, 롤백, 재배포도 가능합니다. 실행중인 컨테이너에 터미널로 연결 할 수도 있습니다.

<img src="/img/gitlab-auto-devops/operations-environments.PNG" alt="등용문S 채용관리자">


마스터로 머지되어 트리거된 파이프라인은 `STAGING_ENABLED` 로 스테이징에 배포되고, 운영 환경에는 수동으로 배포 합니다.
<img src="/img/gitlab-auto-devops/pipeline-master.PNG" alt="파이프라인">

<img src="/img/gitlab-auto-devops/operations-environments-auth.PNG" alt="인증 서버">








## 문제 해결

### 빌드 팩을 선택할 수 없습니다.
Laravel 애플리케이션의 경우, PHP 와 NodeJS 가 같은 프로젝에 사용되는데 package.json 파일의 존재로 PHP 가 구동되지 않음

환경변수 BUILDPACK_URL: http://sri-git.saraminhr.co.kr/ea/buildpacks/heroku-buildpack-php-laravel 로 설정하고
node / webpack 빌드는 pre-build 단계에서 작업을 추가하고 artifacts 로 저장 하여 해결함



### UPGRADE FAILED: "review-14-sprint-l3t9mg" has no deployed releases
정상적으로 릴리즈가 배포에 성공하지 못한 경우, 해당 네임스페이스를 삭제하고 클러스터 캐시를 삭제한뒤 파이프라인을 실행시킵니다.






## 참조

* [GitLab Auto DevOps](https://docs.gitlab.com/ee/topics/autodevops/)
* [Validate .gitlab-ci.yml syntax with the CI Lint tool](https://docs.gitlab.com/ee/ci/lint.html)
* [Kubespray로 쿠버네티스 설치하기](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubespray/)
* [CI/CD variables](https://docs.gitlab.com/14.0/ee/topics/autodevops/customize.html)
