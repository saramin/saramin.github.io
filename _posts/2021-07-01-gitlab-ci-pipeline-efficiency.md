---
title: Boost GitLab CI / CD Pipeline Efficiency
tags:
- gitlab-ci
- pipeline_efficiency
- customizing-auto-devops
- proxy-repository-for-docker
- dockerfile_best-practices
- nexus
- registry-mirrors
- Dockerfile
- BuildKit
author: 엄기화
show-avatar: true
social-share: true
layout: post
subtitle: 온-프렘 쿠버네티스 클러스터에서 GitLab Auto DevOps 기능을 사용하면서 빌드 속도를 향상시킨  커스터마이징 내용을 소개
comments: true
meta-description: |-
  온-프렘 쿠버네티스 클러스터에서 GitLab Auto DevOps 기능을 사용하면서 빌드 속도를 향상시킨  커스터마이징 내용을 소개
  파이프라인 구성 모범 사례는 효율성을 즉시 개선하고 더 빠른 소프트웨어 개발 수명주기를 더 일찍 확보하는 파이프 라인 기능을 사용하는 것입니다.
  파이프라인을 보다 효율적으로 만들면 개발자 시간을 절약할 수 있습니다.
  docker 데몬/서비스가 registry-mirrors 로 nexus 를 사용하도록 설정하여 네트워크 대기 시간(latency) 개선
  CI/CD 작업에서 docker 명령을 가능하도록 소켓 바인딩 사용 (도커 레이어가 캐시됩니다.)
  커스텀 빌드팩 및 빌드 환경으로 CI/CD Variable 전달하여 라이브러리 다운로드 경로를 사내 네트워크로 변경시켜 네트워크 지연 개선 (nexsus raw (proxy) repository)
  커스텀 `Dockerfile` 빌드 캐시 마운트로 종속성(dependencies) 등이 재사용 되어 빌드 속도 향상
---

## CI / CD 파이프라인 효율성 향상시키기
새로운 프로젝트를 시작하면서 기본 구성에서 시작하여 시행 착오를 통해 시간이 지남에 따라 파이프라인 구성을 개선하는 것이 일반적입니다.

더 나은 프로세스는 효율성을 즉시 개선하고 더 빠른 소프트웨어 개발 수명주기를 더 일찍 확보하는 파이프 라인 기능을 사용하는 것입니다.

파이프라인을 보다 효율적으로 만들면 개발자 시간을 절약할 수 있습니다.

[온-프렘 쿠버네티스 클러스터에서 GitLab Auto DevOps  커스터마이징](/2021-06-30-customize-gitlab-auto-devops-with-existing-cluster-on-premise-kubernetes)으로 빌드 속도를 향상시킨 내용을 소개하고자 합니다.


## 요약

* docker 서비스가 registry-mirrors 로 nexus 를 사용하도록 설정하여 네트워크 대기 시간(latency) 개선
* 빌드 작업 전용 GitLab Runner 서버 구성
* CI/CD 작업에서 docker 명령을 가능하도록 소켓 바인딩 사용 (도커 레이어가 캐시됩니다.)
* 커스텀 빌드팩 및 빌드 환경으로 CI/CD Variable 전달하여 라이브러리 다운로드 경로를 사내 네트워크로 변경시켜 네트워크 지연 개선
* 커스텀 `Dockerfile` 빌드 캐시 마운트로 종속성(dependencies) 등이 재사용 되어 빌드 속도 향상







## 병목 현상 및 일반적인 오류

비효율적인 파이프라인은 작업의 실행 시간으로 확인됩니다. 
* 2분 걸리던 작업이 10여분 걸린다거나
* 1-2시간이 지나 타임아웃 되는 경우

GitLab Runner 와 관련된 사항:
* 러너와 이들이 프로비저닝 된 리소스의 가용성.
* 빌드 종속성 및 설치 시간.
* 컨테이너 이미지 크기 - 이미지나 레이어 캐시가 되지 않는 경우
* **네트워크 지연 및 느린 연결.** - 빌드 팩에서 다운로드가 느리거나 타임아웃 되는 경우,

무작위로 실패하거나 신뢰할 수 없는 테스트 결과를 생성하는 비정상적인 단위 테스트는 `TEST_DISABLED` 로 그룹 환경 변수로 비활성 하였습니다. 
또는 수동, 실패해도 넘어가도록 설정하는 할 수도 있습니다.



## Registry Mirror 사용
### Nexus 를 Docker Hub 미러로 사용(registry mirror)
성능 향상을 위해 [레지스트리 미러](https://docs.docker.com/registry/recipes/mirror/)를 구성 하고,  Docker Hub 속도 제한에 도달하지 않도록 합니다.

#### Docker 프락시 저장소 생성
* **docker.io** – Proxy Registry for Docker Hub – http://sri-nexus.example.com/#browse/browse:docker.io
	* Enable Docker V1 API Support
	* Allow anonymous docker pull ( Docker Bearer Token Realm required )
* **registry.gitlab.com** – Proxy Registry for GitLab Repository
	* Allow anonymous docker pull ( Docker Bearer Token Realm required )

#### Docker 저장소 그룹핑
* Name: registry-mirrors
* Repository Connectors: HTTP: 8082
* v Allow anonymous docker pull ( Docker Bearer Token Realm required )
* Enable Docker V1 API Support
* Group / member repositories
	* docker.io
	* registry.gitlab.com

#### Docker Bearer Token Realm 활성화
Nexus Admin / Security / Realms

Anonymous docker pull 을 허용하기 위해서 **Docker Bearer Token Realm** 을 활성화

![Active realms Dialog](/img/gitlab-auto-devops/nexus-realms.png)


docker daemon 서비스에 registry-mirrors 를 설정하면 도커 허브의 이미지는 Nexus 에 캐시됩니다.
`registry.gitlab.com` 에 호스팅된 이미지 `registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image:v2.0.2` 의 경우,  아래와 같이 호스트명을 생략해야 합니다.
~~~
docker pull gitlab-org/cluster-integration/auto-deploy-image:v2.0.2
~~~

Nexus 에 저장된 도커이미지를 확인해 볼 수 있습니다.

![Nexus 에 저장된 도커이미지](/img/gitlab-auto-devops/nexus-docker.png)


## 빌드 전용 GitLab Runner 설치 및 소켓 바인딩 사용
쿠버네티스 클러스터내에서 yarn 설치 및 빌드 작업이 동시에 수행될 경우, 다른 컨테이너에 영향을 미쳐 빌드 작업만 수행하는 러너를 VM 에 설치 하였습니다.

build 단계 작업에서 `docker` 명령을 실행 할 수 있도록 `docker` executor 를 사용 합니다.
Docker in Docker (dind) 는 매번 새 환경에서 실행되어, 동시 처리에는 좋으나 레이어의 캐시가 되지 않아 느립니다. 
도커 소켓 `/var/run/docker.sock` 바인딩을 사용하였으며, 레지스트리 미러, 이미지, 빌드 캐시, 레이어 캐시가 재사용됩니다.

[공식 GitLab Runner Repository](https://docs.gitlab.com/runner/install/linux-repository.html) 를 추가하고, 최신 버전으로 설치 합니다. 

~~~bash
# For Debian/Ubuntu/Mint
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo -E apt-get install gitlab-runner

# For RHEL/CentOS/Fedora
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
sudo -E yum install gitlab-runner
~~~

[GitLab Runner 등록](https://docs.gitlab.com/runner/register/index.html#linux) - `docker` executor,  `/var/run/docker.sock` 공유

~~~bash
sudo gitlab-runner register -n \
  --url http://sri-git.example.com/ \
  --registration-token REGISTRATION_TOKEN \
  --executor docker \
  --description "My Docker Runner" \
  --docker-image "docker:20.10.6" \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock
  --tag-list sri-runner-dev
~~~
* `sri-runner-dev` build 작업을 특정 러너가 실행하도록 태그를 작성합니다. [.gitlab-ciyml](/2021-06-30-customize-gitlab-auto-devops-with-existing-cluster-on-premise-kubernetes/#gitlab-ciyml) 참고


GitLab Runner 구성 `/etc/gitlab-runner/config.toml`

~~~toml
concurrent = 4
check_interval = 3
[session_server]
  session_timeout = 1800
[[runners]]
  name = "sri-runner-dev"
  url = "http://sri-git.example.com/"
  token = "REGISTRATION_TOKEN"
  executor = "docker"
	pre_build_script = "mkdir -p $HOME/.docker && echo $DOCKER_AUTH_CONFIG > $HOME/.docker/config.json "
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "docker:20.10.6"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/DATA/cache:/cache:rw", 
		                           "/var/run/docker.sock:/var/run/docker.sock", 
															 "/etc/docker/daemon.json:/etc/docker/daemon.json"]
    shm_size = 0
~~~
* pre_build_script: 사내 컨테이너 레지스트리에 인증하기 위해 `DOCKER_AUTH_CONFIG` 환경 변수를 사용할 수 있습니다.
* volumes: 캐시 경로, 소켓 바인딩 사용, 레지스트리 미러 설정 공유

GitLab Auto DevOps 템플릿을 사용하면 빌드 작업에 auto-build-image 를 사용하나 소켓 바인딩을 하면 `docker:19.03.8-dind` service 를 포함하지 않아도 됩니다.
[Build.gitlab-ci.yml](https://gitlab.com/gitlab-org/gitlab-foss/-/blob/v12.10.10/lib/gitlab/ci/templates/Jobs/Build.gitlab-ci.yml)

`.gitlab-ci.yml` 에서 `services: []` 로 `docker:dind` 서비스를 실행하지 않도록 하였습니다.
~~~
build:
  image: "gitlab-org/cluster-integration/auto-build-image:${AUTO_BUILD_IMAGE_VERSION}"
  services: []
  tags:
    - sri-runner-dev
~~~
* registry.gitlab.com 는 nexus 에 캐시된 이미지를 사용하기 위해 호스트명은 생략


build 작업의 일부분을 발췌한 내용입니다.  `CACHED` 를 확인할 수 있습니다.
~~~
 Running with gitlab-runner 13.12.0 (7a6612da)
   on sri-runner-dev zqfghrkc
Preparing the "docker" executor
 Using Docker executor with image gitlab-org/cluster-integration/auto-build-image:v0.6.0 ...

...

Using docker image sha256:ddb36cc31fecf898ebf818ab4705b727c0ceaa53ea0f2dd3fc0ef34d7a673518 for gitlab-org/cluster-integration/auto-build-image:v0.6.0 with digest gitlab-org/cluster-integration/auto-build-image@sha256:e87b75eceb24c29a2c22f64d82100ce18a42938d78d04b1942d162f2e8269890 ...
 $ mkdir -p $HOME/.docker && echo $DOCKER_AUTH_CONFIG > $HOME/.docker/config.json
 $ if [[ -z "$CI_COMMIT_TAG" ]]; then # collapsed multi-line command
 $ /build/build.sh
 Building Dockerfile-based application...
 Activating Docker BuildKit to forward CI variables with --secret
 Attempting to pull a previously built image for use with --cache-from...
 sri-registry.example.com/ea/auth/authorization-server/v1-x:a1250f9afff708bd5d761b24a19638105e8800f6
 #1 [internal] load build definition from Dockerfile
 #1 sha256:1ee382716da9ed10e2b82ad5f9e8e110806f9748512cd7592fa1bef3080c6307
 #1 transferring dockerfile: 1.34kB done
 #1 DONE 0.0s
 #2 [internal] load .dockerignore
 #2 sha256:65a3a5ee3ae86bbc99f98e758738cfd2f125276f74aad5bd67ab8fbb1e0071fe
 #2 transferring context: 2B done
 #2 DONE 0.0s
 #3 resolve image config for docker.io/docker/dockerfile:1.2
 #3 sha256:b239a20f31d7f1e5744984df3d652780f1a82c37554dd73e1ad47c8eb05b0d69
 #3 DONE 1.9s
 #4 docker-image://docker.io/docker/dockerfile:1.2@sha256:e2a8561e419ab1ba6b2fe6cbdf49fd92b95912df1cf7d313c3e2230a333fdbcc
 #4 sha256:37e0c519b0431ef5446f4dd0a4588ba695f961e9b0e800cd8c7f5ba6165af727
 #4 CACHED
 #7 [auth] dym/storage-damo:pull token for sri-registry.saraminhr.co.kr
 #7 sha256:675dca529670b92b3be120f7016985363a4f21c19afd0b2f12431a8002eef83a
 #7 DONE 0.0s
 #5 [internal] load metadata for docker.io/gliderlabs/herokuish:latest
 #5 sha256:30741a0974d06da05a4fc23a61172b4936dc9515939e3b39265007bd2daedea0
 #5 DONE 1.1s
 #9 importing cache manifest from sri-registry.example.com/ea/auth/authorization-server/v1-x:latest
 #9 sha256:35c289f085f50678ecdc2b54ab39273211973a1979e61f645ff16aef69bb4590
 #9 DONE 0.0s
 #8 importing cache manifest from sri-registry.example.com/ea/auth/authorization-server/v1-x:a1250f9afff708bd5d761b24a19638105e8800f6
 #8 sha256:3db95872f922aaf30d60293af87681e5f089f590d03b51697ac93e57bb18367b
 #8 DONE 0.0s
 #21 [internal] load build context
 #21 sha256:22fe1ec45b146664bc044f5bb4802589db90d669be093285deb7f5effbc6c797
 #21 transferring context: 1.05MB 0.3s done
 #21 DONE 0.3s
 #10 [builder 1/3] FROM docker.io/gliderlabs/herokuish@sha256:e650f2d588dce2bcc024f13504d807640be0427bc34701885cce420164881536
 #10 sha256:e9979b775d9defce6c036f43ed9ba433b014a05b44a892ed117521f150d3696b
 #10 CACHED
~~~



### `docker:dind` 서비스를 위해 [레지스트리 미러](https://docs.docker.com/registry/recipes/mirror/) 사용

GitLab Runner 관리자는 모든 `dind` 서비스에 미러를 사용할 수 있습니다 . [configuration(/etc/gitlab-runner/config.toml)](https://docs.gitlab.com/runner/configuration/advanced-configuration.html) 을 업데이트하여 [볼륨 마운트](https://docs.gitlab.com/runner/configuration/advanced-configuration.html#volumes-in-the-runnersdocker-section) 를 지정합니다 .

~~~yaml
{
    "storage-driver": "overlay2",
    "insecure-registries": [
        "sri-nexus.example.com:8082"
    ],
    "registry-mirrors" : ["http://sri-nexus.example.com:8082"]
}
~~~

configuration 을 업데이트 한 뒤 reload 하고, 적용되는 configuration 을 확인 할 수 있습니다.

~~~bash
systemctl reload docker && journalctl -f -u docker
~~~

configuration 을 되돌리려는 경우,  `"registry-mirrors" : []` 와 같이 변경하고 reload 해야 합니다.



## 빌드 작업에서 네트워크 지연 개선
* Nexus Raw (proxy) format Repository 생성
* 빌드백 URL 을 사내 리파지터리로 설정
* 외부 연결에 문제가 있는 경우, 사내 서버를 이용하도록 하던가 다운로드

### Create Repository : raw (proxy)
Nexus 에서 raw (proxy) format 으로 리파지터리를 생성합니다.

예를 들어, 다음과 같은 내용으로 Raw Repository 를 생성하면 사내 네트워크 URL 로 접속하여 캐시된 컨텐츠를 사용할 수 있습니다.

* Name: lang-common.s3.amazonaws.com
* Remote Storage: https://lang-common.s3.amazonaws.com
* URL: `http://sri-nexus.example.com/repository/lang-common.s3.amazonaws.com/`

`BUILDPACK_STDLIB_URL=http://sri-nexus.example.com/repository/lang-common.s3.amazonaws.com/buildpack-stdlib/v7/stdlib.sh` 과 같이 정의하여 리파지터리에 캐시된 URL 에 대한 네트워크 지연을 개선 할 수 있습니다.

https://github.com/heroku/heroku-buildpack-gradle 의 경우, 아래의 remote storage 로 3개의 Raw (proxy) Repositories 를 생성합니다.
* https://lang-common.s3.amazonaws.com
* https://buildpack-registry.s3.amazonaws.com
* https://lang-jvm.s3.amazonaws.com


![Components in Nexus Raw Repositories](/img/gitlab-auto-devops/nexus-raw-repository.png)


#### CI / CD Variable 
외부 인터넷 연결로 네트워크 지연을 개선하기 위해 빌드팩에서 사용할 그룹/프로젝트 환경변수
~~~
AUTO_DEVOPS_BUILD_IMAGE_FORWARDED_CI_VARIABLES=GRADLE_OPTS,CI_COMMIT_SHA,GRADLE_TASK,DOCKER_AUTH_CONFIG,JVM_COMMON_BUILDPACK,JVM_BUILDPACK_ASSETS_BASE_URL,BUILDPACK_STDLIB_URL	 	 
BUILDPACK_STDLIB_URL=http://sri-nexus.example.com/repository/lang-common.s3.amazonaws.com/buildpack-stdlib/v7/stdlib.sh	 	 
JVM_COMMON_BUILDPACK=http://sri-nexus.example.com/repository/buildpack-registry.s3.amazonaws.com/buildpacks/heroku/jvm.tgz
JVM_BUILDPACK_ASSETS_BASE_UR=http://sri-nexus.example.com/repository/lang-jvm.s3.amazonaws.com	 	 
~~~




### 커스텀 빌드 팩 URL 제공
 
Spring Boot (Gradle)로 개발한 프로젝트는 빌드 작업에서 [gliderlabs/heokuish](https://github.com/gliderlabs/herokuish/tree/master/buildpacks) 에서 자동으로 감지하여 `BUILDPACK_URL` 은 https://github.com/heroku/heroku-buildpack-gradle 이 됩니다. 

[lib/common.sh#L3](https://github.com/saramin/heroku-buildpack-gradle/blob/main/lib/common.sh#L3)
~~~
 export BUILDPACK_STDLIB_URL="https://lang-common.s3.amazonaws.com/buildpack-stdlib/v7/stdlib.sh"
~~~
 
빌드 팩 실행을 위해서 표준라이브러리를 다운로드 하는데 네트워크 지연이나 연결에 문제가 있어 heroku buildpack 을 fork 하여 환경 변수로 URL을 변경할 수 있도록 코드를 약간 변경하였습니다.

[Customizable BUILDPACK_STDLIB_URL](https://github.com/saramin/heroku-buildpack-gradle/commit/54fefdcfc8e668a5de26ddd299af19a9bc2126f9)

~~~diff
- export BUILDPACK_STDLIB_URL="https://lang-common.s3.amazonaws.com/buildpack-stdlib/v7/stdlib.sh"
+ export BUILDPACK_STDLIB_URL=${BUILDPACK_STDLIB_URL:-"https://lang-common.s3.amazonaws.com/buildpack-stdlib/v7/stdlib.sh"}
~~~
  
커스텀 빌드 팩을 사용하도록 GitLab CI Variable 로 `BUILDPACK_URL=https://sri-git.exsample.com/saramin/heroku-buildpack-gradle` 를 설정했고,
빌드팩 주소도 사내 저장소(git repository)로 연결되고, Nexus Raw (proxy) Repository 에 캐시된 stdlib.sh 를 사용하여 네트워크 연결  문제로 빌드 작업이 실패하는 문제를 해결 하였습니다.

### 빌드 환경으로 CI Variable 전달

그리고 또 네트워크 지연이 생긴 부분이 있는데
`heroku-buildpack-gradle` 은 `heroku-buildpack-jvm-common` 라이브러리를 사용합니다.
[common.sh install_jdk()](https://github.com/heroku/heroku-buildpack-gradle/blob/main/lib/common.sh#L92) 함수내에서 `JVM_COMMON_BUILDPACK` 값이 없으면 `https://buildpack-registry.s3.amazonaws.com/buildpacks/heroku/jvm.tgz` 경로를 기본값으로 사용하는 이 경우에는 빌드 환경으로 `JVM_COMMON_BUILDPACK` 을 사내 서버로 변경하여 해결할 수도 있습니다. 

~~~
JVM_COMMON_BUILDPACK=${JVM_COMMON_BUILDPACK:-https://buildpack-registry.s3.amazonaws.com/buildpacks/heroku/jvm.tgz}
~~~

[jvm.sh](https://github.com/heroku/heroku-buildpack-jvm-common/blob/main/lib/jvm.sh#L28) 에서 아래와 같은 부분도 있어 `JVM_BUILDPACK_ASSETS_BASE_URL`을 사내 서버로 변경하여 네트워크 지연 및 연결 문제를 해결 할 수 있습니다.
~~~
JVM_BUILDPACK_ASSETS_BASE_URL="${JVM_BUILDPACK_ASSETS_BASE_URL:-"https://lang-jvm.s3.amazonaws.com"}"
JDK_BASE_URL="${JVM_BUILDPACK_ASSETS_BASE_URL%/}/jdk/${STACK:-"heroku-18"}"
~~~

`AUTO_DEVOPS_BUILD_IMAGE_FORWARDED_CI_VARIABLES` 를 사용해서 [빌드 환경으로 CI/CD Variable 을 전달](https://docs.gitlab.com/ee/topics/autodevops/customize.html#forward-cicd-variables-to-the-build-environment) 할 수 있습니다.  빌드 환경에서 사용할 변수이름을 쉼표로 구분합니다.

~~~
AUTO_DEVOPS_BUILD_IMAGE_FORWARDED_CI_VARIABLES=GRADLE_OPTS,CI_COMMIT_SHA,GRADLE_TASK,DOCKER_AUTH_CONFIG,JVM_BUILDPACK_ASSETS_BASE_URL,JVM_COMMON_BUILDPACK
~~~
* 빌드 팩을  사용할 때 전달된 변수는 자동으로 환경변수로 사용됩니다.
* 커스텀 Dockerfile 을 사용하는 경우, 상단에 도커 [빌드 킷을 사용](https://docs.docker.com/develop/develop-images/build_enhancements/)하도록 구문을 활성화 해야 합니다.
~~~
# syntax=docker/dockerfile:1.2
~~~
* `Dockerfile` 에서 secret 를 가능하게 하려면 `RUN $COMMAND` 를 실행하기 전에 secret 을 마운트하고, 실행합니다.
~~~
RUN --mount=type=secret,id=auto-devops-build-secrets . /run/secrets/auto-devops-build-secrets && $COMMAND
~~~


### docker build 명령에 `--build-arg`전달
`AUTO_DEVOPS_BUILD_IMAGE_EXTRA_ARGS` 를 사용해서 `docker build` 명령에 전달할 수 있습니다. 예를 들어  기본값인 `ruby:latest` 가 아닌 `ruby:alpine` 으로 도커 이미지가 빌드 됩니다.

group/project CI / CD Variable: 
~~~
AUTO_DEVOPS_BUILD_IMAGE_EXTRA_ARGS="--build-arg=RUBY_VERSION=alpine"
~~~

Custom Dockerfile: 
~~~
ARG RUBY_VERSION=latest
FROM ruby:$RUBY_VERSION
~~~

줄 바꿈 및 공백과 같은 복잡한 값을 전달해야하는 경우 Base64 인코딩을 사용합니다. Base64 인코딩되지 않은 복잡한 값은 Auto DevOps가 인수를 사용하는 방식으로 인해 이스케이프 문제를 일으킬 수 있습니다.

비밀번호와 같은 것은 이미지에 저장될 수 있으므로  `--build-arg` 로 전달하지 않도록 합니다.


### Build Mounts `RUN --mount=type=cache,target=/tmp/cache`

[빌드 킷](https://docs.docker.com/develop/develop-images/build_enhancements/)으로 이미지를 빌드하며, 빌드 속도를 높이기 위해 `/tmp/cache` 위치를 캐시로 마운트 합니다.
이전 빌드에서 다운로드한 라이브러리가 재사용 됩니다. 

~~~
# syntax=docker/dockerfile:1.2

FROM gliderlabs/herokuish as builder
COPY . /tmp/app
ENV USER=herokuishuser

# https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/syntax.md
RUN --mount=type=cache,target=/tmp/cache --mount=type=secret,id=auto-devops-build-secrets . /run/secrets/auto-devops-build-secrets && /bin/herokuish buildpack build

FROM gliderlabs/herokuish

COPY --chown=herokuishuser:herokuishuser --from=builder /app /app
ENV PORT=5000
ENV USER=herokuishuser
EXPOSE 5000
CMD ["/bin/herokuish", "procfile", "start", "web"]
~~~
 * [auto-build-image 의 Dockerfile 기본값에 캐시 마운트](https://gitlab.com/gitlab-org/cluster-integration/auto-build-image/-/blob/master/src/Dockerfile.erb)

<p></p>
####  빌드 캐시 사용량 확인 및 불필요한 이미지 및 빌드 캐시 제거
`docker system df` 
~~~
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          77        0         25.35GB   25.35GB (100%)
Containers      0         0         0B        0B
Local Volumes   6         0         1.65MB    1.65MB (100%)
Build Cache     530       0         25.99GB   25.99GB
~~~

`docker system prune`
~~~
WARNING! This will remove:
  - all stopped containers
  - all networks not used by at least one container
  - all dangling images
  - all dangling build cache

Are you sure you want to continue? [y/N] y
Deleted Images:
untagged: sri-docker-registry.example.com:5000/ea/auth/apigateway/master@sha256:330f9063468a74d03403e6ce56f76a4624cb44c1bf22ac83b28698cf8b9779b5
deleted: sha256:8a0b137a7734c7ca1b22884720a3de1a39dbbe5431064857cecf45e1c4cb36ae
untagged: sri-docker-registry.example.com:5000/ea/auth/apigateway/master@sha256:770889d539c499329701cb67ea61f14221665c88ed2951bc4e7f2c8e24e8f0ce
deleted: sha256:8e739443be1e2ae2c38e51c2eeb18e87534947ded05ce1b5ecd1b121dcbd37bd
untagged: sri-docker-registry.example.com:5000/ea/auth/apigateway/master@sha256:820af985bafcc66d4dfb0afb700be9855d97c162521dd6686c1bed75a5646ef8
deleted: sha256:c34337c8c1a419b1d70d5d38af62add686f256cd07a94905a77b01d9e9018512
untagged: sri-registry.example.com/ea/auth/apigateway/1-enable-staging@sha256:2e93e79a940494b740544518fe64903875f8255fa6ab4be5b3f5b59a10c41142
deleted: sha256:25248d03e8ee1b7006d3f19ab1b779b7ce9023121bc05587411828ac3a310113
...
~~~

`docker system df`
~~~
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          73        0         24.5GB    24.5GB (100%)
Containers      0         0         0B        0B
Local Volumes   6         0         1.65MB    1.65MB (100%)
Build Cache     115       0         0B        0B
~~~

## 파이프라인 분석 및 모니터링
파이프 라인의 성능을 분석하여 효율성을 개선하는 방법을 찾을 수 있습니다.  CI / CD 인프라에서 차단 요소를 식별하는 데 도움이 될 수 있습니다.

* 작업 워크로드.
* 실행 시간의 병목 현상.
* 전체 파이프 라인 아키텍처.

파이프 라인 워크 플로를 이해하고 문서화하고 가능한 작업 및 변경 사항에 대해 논의하는 것이 중요합니다. 파이프 라인 개선은 DevSecOps 라이프 사이클에서 팀 간의 신중한 상호 작용이 필요할 수 있습니다.

docker build 작업 멈춰있는 경우, 보통 빌드 팩에서 라이브러리 다운로드중 네트워키 지연이 발생하곤 하는데, 컨테이너 내에서 실행되는 docker:dind 파드에서 `ps -ef` 와 같은 명령으로 어떤 작업에서 병목이 생기는지 확인해 볼 수 있습니다.


### 파이프라인 인사이트
[파이프 라인의 성공과 지속 시간 차트](https://docs.gitlab.com/ee/ci/pipelines/index.html#pipeline-success-and-duration-charts)는 파이프 라인의 실행과 실패한 작업 카운트에 대한 정보를 제공합니다.

![Pipeline Insight Screenshot](/img/gitlab-auto-devops/pipeline-insight.png)




## 파이프라인 구성
작업 실행 속도를 높이고 리소스 사용량을 줄이는 방향을 구성

### 작업 실행 빈도 줄이기
[interruptible](https://docs.gitlab.com/ee/ci/yaml/index.html#interruptible) 키워드를 사용하여 새 파이프라인에 의해 이전(old) 파이프라인을 취소할 수 있습니다.

~~~
build:
    interruptible: true
~~~

[rules](https://docs.gitlab.com/ee/ci/yaml/index.html#rules): 예를 들어, package.json yarn.lock 소스 코드가 변경되었을 때만 yarn 설치 및 빌드

~~~
docker build:
  script: docker build -t my-image:$CI_COMMIT_REF_SLUG .
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - Dockerfile
      when: manual
      allow_failure: true
~~~

불필요한 [예약된 작업(scheduled pipeline)](https://docs.gitlab.com/ee/ci/pipelines/schedules.html) 은 빈번하게 실행되지 않도록 합니다.



### Fail Fast
파이프라인 초기에 오류가 감지되도록 하여 나머지 파이프라인이 실행되지 않도록 합니다.
* 구분, 스타일 Linting, Git 커밋 메시지 확인 및 유사한 작업


### 캐싱 
종속성 캐시, NodeJs/node_module 이나 PHP Composer/vendors
`cache:when: 'always'` 로 작업이 실패한 경우에도 다운로드된 종속성을 캐시합니다.

도커 빌드 킷을 활성하고 빌드 캐시를 적용할 수 도 있습니다.

### Docker 이미지 최적화

큰 Docker 이미지는 많은 공간을 사용하고 느린 연결 속도로 다운로드하는 데 오랜 시간이 걸립니다.
가능하면 모든 작업에 하나의 큰 이미지를 사용하지 않고. 다운로드 및 실행 속도가 더 빠른 특정 작업에 대해 각각 여러 개의 작은 이미지를 사용합니다.

소프트웨어가 사전 설치된 사용자 지정 Docker 이미지를 사용해보십시오. 일반적으로 일반적인 이미지를 사용하고 매번 소프트웨어를 설치하는 것보다 미리 구성된 더 큰 이미지를 다운로드하는 것이 훨씬 빠릅니다. [Dockerfiles 작성 모범 사례](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) 에는 효율적인 Docker 이미지 빌드에 대한 자세한 정보가 있습니다.

Docker 이미지 크기를 줄이는 방법 :

* `.dockerignore` 빌드 불필요한 파일을 추가
* 작은 기본 이미지를 사용합니다. 예) `debian-slim`.
* 꼭 필요하지 않은 경우 vim, curl 등과 같은 편의 도구를 설치하지 않습니다.
* 전용 개발 이미지를 만듭니다.
* 공간을 절약하기 위해 패키지에 의해 설치된 매뉴얼 페이지 및 문서를 비활성화합니다.
* RUN계층을 줄이고 소프트웨어 설치 단계를 결합하십시오.
* [멀티 스테이지 빌드](https://blog.alexellis.io/mutli-stage-docker-builds/) 이미지 크기를 줄일 수있는 하나의 Dockerfile에 빌더 패턴을 사용하는 여러 Dockerfiles을 병합 할 수 있습니다.
* `apt` 를 사용하는 경우 불필요한 패키지를 피하기 위해   `--no-install-recommends` 추가
* 마지막에 더 이상 필요하지 않은 캐시와 파일을 정리합니다. 
예를 들어 Debian 및 Ubuntu 의 경우,  `rm -rf /var/lib/apt/lists/*` 또는 RHEL 및 CentOS의 경우,  `yum clean all`.
* [dive](https://github.com/wagoodman/dive) 또는 [DockerSlim](https://github.com/docker-slim/docker-slim) 과 같은 도구를 사용 하여 이미지를 분석하고 축소합니다.

도커 이미지 관리를 단순화하기 위해, 전용 그룹을 만들 수 있습니다. CI / CD 파이프 라인으로  도커 이미지 및 테스트, 빌드 및 게시



### Test, document, and learn
파이프 라인 개선은 반복적 인 프로세스입니다. 약간 변경하고 효과를 모니터링 한 다음 다시 반복하십시오. 작은 개선 사항이 많으면 파이프 라인 효율성이 크게 증가 할 수 있습니다.

파이프 라인 설계 및 아키텍처를 문서화하는 데 도움이 될 수 있습니다. GitLab 저장소에서 직접 Markdown의 Mermaid 차트로 이를 수행 할 수 있습니다 .

수행 된 연구 및 발견 된 솔루션을 포함하여 문제의 CI / CD 파이프 라인 문제 및 사고를 문서화합니다. 이를 통해 새로운 팀원을 온 보딩하고 CI 파이프 라인 효율성으로 반복되는 문제를 식별 할 수 있습니다.






## 참조
1. [Pipeline efficiency](https://docs.gitlab.com/ee/ci/pipelines/pipeline_efficiency.html) - GitLab's 파이프라인 효율성
2. [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
3. [Nexus Repository Manager 3: Proxy Repository for Docker](https://help.sonatype.com/repomanager3/formats/docker-registry/proxy-repository-for-docker)
4. [Customizing Auto DevOps](https://docs.gitlab.com/ee/topics/autodevops/customize.html#customizing-auto-devops)
