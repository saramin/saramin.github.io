---
title: '통합된 개발과 배포 : Monorepo와 GitOps의 매력적인 조합'
author: 조성창
---

안녕하세요. FE개발팀 조성창 입니다.

사람인에선 서비스의 레거시 영역을 점진적으로 개선해나가고 있습니다.

그동안 FE개발팀은 긱이나 멘토링 같은 버티컬 서비스의 FE개발을 진행해왔는데,
작년부터 주요서비스의 FE분리를 시작하면서 FE 영역의 아키텍쳐에 대한 고민을 했었습니다.

그 결과 Monorepo를 적용하기로 하였고 첫번째 서비스가 배포를 앞두고 있습니다.

그 과정에서 배포 환경에 대한 고민도 같이 하게 되었고 GitOps를 적용하게 되었는데 그 과정을 소개해보려고 합니다.

**FE개발팀 파이프라인**

FE개발팀이 운영하던 서비스들은 일반적인 파이프라인의 모습을 가지고 있었습니다.

기능 개발을 위해 `feature` 브랜치를 생성하고 개발코드 커밋들을 푸시하게 되면 `review`서버에서 `next build`와 `next start`를 진행합니다.

개발이 완료된 뒤 기능 QA를 `review`서버에서 1차로 진행한 뒤 `develop` 브랜치에 `merge`를 하게 됩니다.
그럼 `dev`서버에 배포가 되고 여기서 최종 QA를 진행한 뒤 운영환경으로 배포를 결정하게 됩니다.

운영 배포를 위해 `main` 브랜치에 `merge`를 하게 되면 앞선 브랜치들과 동일하게 진행해야 하는데요 운영환경이다 보니 웹서버가 여러대이고, 이에 맞게 배포를 위해 서버 하나씩 `LB`를 내리고 `build`와 `deploy`를 진행한 뒤 `pm2 reload`, `LB`를 다시 올리는 과정을 거치게 됩니다.

**그 과정에서 아쉬움**

사람인엔 `DevOps`를 전담하는 조직이 없다보니 SRE팀에 서버를 요청한 뒤 개발자가 주도적으로 환경을 구축해야 합니다.

그렇다 보니 FE개발자들 입장에선 `local`에서 개발할때와 비슷한 방식으로 환경을 구성하게 되는데요.
`next build` > `dist` 산출물을 서버에 배포한 뒤 `next start`를 통해 `localhost`를 띄우고, `nginx`에서 `proxy`로 연결하는 방식으로요.

`dev`나 `prod` 환경은 매칭되는 브랜치가 각각 `develop`, `main`로 고정되어 있기 때문에 괜찮았는데, `feature`, `hotfix` 브랜치는 유동적으로 운영해야 하다 보니 웹서버와 브랜치별를 매칭해주기엔 어려움이 있었습니다.
그렇다보니 `review` 서버를 3개정도 구축해놓고, 필요할때마다 해당 서버에 배포해서 사용었했습니다.

결국 이런 과정을 좀더 유연하게 구성하려면 컨테이너 기반의 배포환경이 필연적이라고 판단했고, 도입을 검토하게 되었습니다.

# GitOps
> DevOps의 실천 방법 중 하나로 애플리케이션의 배포와 운영에 관련된 모든 요소들을 Git에서 관리(Operation) 한다는 뜻

때는 작년 12월 어느 추운날, IT연구소에선 TechDay를 진행하게 되었는데 팀별로 1년동안 진행했던 프로젝트중 성과있는 것들을 발표하는 시간이였습니다.
이날 SRE팀에서는 GitOps에 대해서 발표를 하였는데 전 눈이 번쩍 뜨였습니다.

"우리가 GitOps 환경을 구축했어! 드루와~"

마치 들어오라고 손짓을 하는 느낌을 받았고, 전 겁도 없이 덮썩 물었습니다.

~~도커 이미지만 우리가 빌드하면 나머진 자동으로 배포될것 같았거든요.~~
~~(중간 과정 생략.....할많하않.....)~~

새로 구축된 GitOps 배포 환경을 소개해보려 합니다.


[<img src="{{site.url}}/img/monorepo/image1.png" />]({{site.url}}/img/monorepo/image1.png)  
## 1. Image build

### Dockerfile 작성하기

최초 `Dockerfile`을 생성해야 하는데, 일반적으로 프로젝트 루트에 생성합니다.

하지만 현재 진행중인 프로젝트는 `turborepo`를 사용하여 `monorepo`로 구성했고,
이 안에서 `/apps` 폴더 밑에 두기로 했습니다.

``` bash
📦monoprepo
 ┣ 📦apps
 ┃ ┣ 📦web1
 ┃ ┣ 📦web2
 ┃ ┗ 📜Dockerfile # 요기
 ┣ 📦packages
 ┃ ┣ 📦eslint-config-custom
 ┃ ┣ 📦shared
 ┃ ┣ 📦tsconfig
 ┃ ┗ 📜package.json
 ┣ 📜package.json
 ┣ 📜pnpm-workspace.yaml
 ┗ 📜turbo.json
```
이후 생성할 프로젝트들은 계속해서 유사한 스택으로 개발할 것 같았기에 같은 파일을 사용하기 위함이였습니다.

이제 Dockerfile을 작성해야 하는데, 먼저 가이드를 살펴봅니다.
https://turbo.build/repo/docs/handbook/deploying-with-docker

여긴 `yarn` 기반으로 작성되어있네요? 전 `pnpm`을 사용하고 있는데 말이죠.
때문에 첫번째 `stage`에 `pnpm` 설치 구문을 추가해줘야 합니다.
```
RUN npm install -g pnpm
# 또는
RUN corepack enable
```
그리고 빌드될 이미지는 나름의 최적화를 거쳐 빌드시간 등을 효율적으로 관리해야 하는데,
여기서 이미지 경량화를 위한 방법중 `Multi Stage`에 대해 알아보겠습니다.

> Multi Stage 란?
* 컨테이너 이미지를 만드는 과정엔 필요한 리소스이지만 최종 이미지에선 필요없는 리소스를 제거할 수록 단계를 나눠 이미지를 만드는 방법
  * 예) `node_modules`는 빌드할땐 필요하지만 최종 이미지엔 필요없다
* 배포될 이미지 용량을 줄여 전체적인 배포과정의 시간을 줄일 수 있다

그래서 아래와 같이 `stage`를 분리해보았습니다.

### Dockerfile
#### base stage
``` bash
#### alpine stage ####
FROM node:20-alpine AS alpine

#### base stage ####
FROM alpine as base
  # 필요한 구성요소 설치를 위해 `libc6-compat` 추가
  RUN apk add --no-cache libc6-compat
  RUN apk update

  # DOckerfile을 사용할때 주입할 변수
  ARG SERVICE_NAME
  ARG SERVER_ENV

  # 외부에서 주입된 값이나 새롭게 정의한 값을 위한 ENV값 정의
  ENV APP_NAME=$SERVICE_NAME
  ENV APP_ENV=$SERVER_ENV
  ENV PNPM_HOME="/pnpm"
  ENV PATH="$PNPM_HOME:$PATH"
  
  # pnpm 설치를 위해 패키지 매니저의 버전관리 도구 이용
  RUN corepack enable
  
  # turbo 설치
  RUN pnpm install turbo --global
```

#### builder stage
``` bash
FROM base AS builder
  # Set working directory
  WORKDIR /app
  # RUN pwd
  # RUN ls -alRr
  
  # Host의 파일을 WORKDIR로 복사
  COPY . .
  
  # turborepo의 강력한 기능인 prune 실행
  RUN turbo prune --scope=${APP_NAME} --docker
```
> turbo prune 이란 모노레포를 가지치기 위한 기능
* `monorepo`는 `root`에서 있는 `pnpm-lock.yaml`파일로 모든 패키지를 관리
* 특정 프로젝트에서만 필요로 하는 `package` 설치 파일을 생성하기 위함

#### installer stage
``` bash
FROM base AS installer
  WORKDIR /app

  # turbo prune 으로 추출된 /out 폴더를 WORKDIR로 복사
  COPY .gitignore .gitignore
  COPY --from=builder /app/out/json/ .
  COPY --from=builder /app/out/pnpm-lock.yaml ./pnpm-lock.yaml
  COPY --from=builder /app/out/pnpm-workspace.yaml ./pnpm-workspace.yaml
  
  # 프로젝트 소스들을 WORKDIR로 복사
  COPY --from=builder /app/out/full/ .
  
  # next build
  RUN turbo run build:${APP_ENV} --filter=${APP_NAME}
```

#### runner stage
``` bash
FROM base AS runner
  WORKDIR /app
  
  # 불필요한 root 권한 획득을 위해 별도 USER 생성
  RUN addgroup --system --gid 1001 nodejs
  RUN adduser --system --uid 1001 nextjs
  USER nextjs

  COPY --from=installer /app/apps/${APP_NAME}/next.config.js .
  COPY --from=installer /app/apps/${APP_NAME}/package.json .
  
  COPY --from=installer --chown=nextjs:nodejs /app/apps/${APP_NAME}/.next/standalone ./
  COPY --from=installer --chown=nextjs:nodejs /app/apps/${APP_NAME}/.next/static ./apps/${APP_NAME}/.next/static
  COPY --from=installer --chown=nextjs:nodejs /app/apps/${APP_NAME}/public ./apps/${APP_NAME}/public

  EXPOSE 3000
  ENV PORT 3000
  
  CMD node apps/${APP_NAME}/server.js
```




### Gitlab-ci 구성
#### build 과정 설계
이제 `build`를 위한 ci스크립트 작성을 해봅니다.
우선 일반적인 파이프라인은 아래와 같습니다.
* 소스코드를 `push`하면 해당 프로젝트에 연결된 `gitlab-runner`가 동작
  * 동작 초기엔 소스코드 전체를 내려받기 때문에 추가로 `clone`을 할 필요가 없음
* `node package`를 설치하고 `next build` 진행
* `next start`를 통해 프로젝트 구동

하지만 dockerfile을 제작할때 위와 같은 과정을 작성했기 때문에 실제 ci스크립트에선 작성하지 않고 바로 컨테이너 이미지를 빌드하도록 작성합니다.

보통은 `Docker Daemon`을 통해 빌드를 진행하지만 여기선 `kaniko`를 가지고 빌드를 진행합니다.
이유는 아래와 같습니다. (인프라팀에서 추천해준 도구)

* 현재 gitlab-runner는 컨테이너 안에서 돌고 있고 이 안에서 이미지를 빌드해야 하는데, 노드가 분산된 환경에서 Darmon을 띄우는건 리소스 차원에서 비효율적
* `Docker Daemon`은 Host에 대한 ` Root Privilege`를 요구하기 떄문에 보안적으로 취약
* `Daemon`이 필요 없고 Host의 `root`권한을 요구하지 않는 도구중 `kaniko`를 선정
* `kaniko`는 `Snapshot` 기능을 통해 이미지 빌드 과정의 각 Layer를 캐싱함

`kaniko`로 빌드를 진행할땐 `pod.yaml`파일을 생성하여 선언적 방식으로 할 수 있으나 우린 `GitOps`를 구성하고 있기 때문에 `gitlab-ci.yml` 에 명령적 방식으로 소스코드를 작성해야 합니다.
해서 컨테이너 이미지가 저장될 곳을 정하고, 관련 정보를 `gitlab` 변수로 지정하기로 합니다.


#### 컨테이너 이미지 저장소 설정
>Harbor란? 컨테이너 이미지를 프라이빗하게 저장하기 위한 공간

사내 인프라인 `harbor`를 이용합니다.

`harbor`에 프로젝트를 생성하고, 여기서 활동활 `bot`을 생성한 뒤 해당 정보를 `gitlab` 프로젝트 변수에 저장하였습니다.

이 변수는 컨테이너 이미지를 빌드할때 캐싱하고, 이미지를 `push`할때 사용하게 됩니다.

#### CI 스크립트 작성
사용된 변수는 아래와 같습니다.

* 사용자 정의 변수
  * REGISTRY : harbor url
  * REGISTRY_USER : username
  * REGISTRY_PASSWORD : userpassword
  * PROJECT_DIR : 저장소가 복제되고 실행되는 경로
  * SERVICE_NAME : `monorepo`에서 실제 프로젝트 위치
* Gitlab 내부 변수
  * COMMIT_REF_SLUG : 개발서버 서브도메인
  * COMMIT_SHORT_SHA : `commit message`의 `hash`값중 8자리로 이미지의 `tag`로 지정하기 위함
  * CI_COMMIT_BRANCH : commit이 발생한 브랜치 이름
  * CI_MERGE_REQUEST_SOURCE_BRANCH_NAME : `Merge Request` 대상 브랜치 이름, `MR`이 생성되었을때만 실행하기 위해 사용


``` bash
# Extend - 컨테이너 이미지 생성
.docker-build:
  image:
    # 구글에서 제공하는 kaniko 이미지 사용
    #   debug 태그의 이미지를 사용해야 파이프라인에 로그 출력
    name: gcr.io/kaniko-project/executor:debug
   
    # entrypoint를 override 하지 않으면 script들이 실행되지 않는다고 함
    entrypoint: [""]
    
  before_script:
    # harbor registry 접속 정보 저장
    - mkdir -p /kaniko/.docker   
    - echo "{\"auths\":{\"$REGISTRY/fe\":{\"username\":\"$REGISTRY_USER\",\"password\":\"$REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json

  script:
    # kaniko로 컨테이너 이미지 빌드
    #   tag: latest, hash 2개의 이미지 생성
    #   context : 프로젝트 루트로 docker파일 내에서 명령어를 실행할 위치
    #   dockerfile : Dockerfile 위치
    #   destination : 이미지가 push될 경로와 파일명
    #   --dockerfile : Dockerfile 위치
    #   --destination : 이미지가 push될 경로와 파일명
    #   --cache : 각각의 Layer들을 캐싱
    #   --build-arg SERVICE_NAME=$SERVICE_NAME : monorepo의 서비스명을 주입
    #   --build-arg SERVER_ENV=$SERVER_ENV : 환경변수 주입
    - >
      /kaniko/executor
      --context "$PROJECT_DIR"
      --dockerfile "$PROJECT_DIR/apps/Dockerfile"
      --destination $REGISTRY/fe/apps/${SERVICE_NAME}:$COMMIT_SHORT_SHA
      --destination $REGISTRY/fe/apps/${SERVICE_NAME}:latest
      --cache
      --build-arg SERVICE_NAME=${SERVICE_NAME}
      --build-arg SERVER_ENV=${SERVER_ENV}

# Extend - review deploy rules
.rules-only-develop:
  rules:
    - if: $CI_COMMIT_REF_NAME == "develop"
      changes:
        paths:
          - '.gitlab/ci/**'
          - 'apps/Dockerfile'
          - 'apps/${SERVICE_NAME}/**/*'
          - packages/shared/**/*

# Stage - 컨테이너 이미지 빌드
"build/web1":
  stage: build
  interruptible: true
  extends:
    - .docker-build
    - .rules-only-develop
  tags: [fe-dev]
  variables:
    SERVICE_NAME: web1
    SERVER_ENV: dev
```

#### Harbor Policy 설정
위와 같이 설정했을 경우 `harbor`의 저장소엔 이미지들이 계속해서 쌓이게 됩니다.
적절히 유지되도록 `policy` 정책 설정을 해줍니다.
[<img src="{{site.url}}/img/monorepo/image2.png" />]({{site.url}}/img/monorepo/image2.png) 

Harbor > 프로젝트 내에 Policy 탭에 들어가면 `ADD RULE` 버튼을 눌러 정책들을 추가해줄 수 있는데, 아래와 같이 설정했습니다.
* /web1/develop : 가장 최근 push된 이미지 10개
* /web1/feature*
  * 최근 push된 이미지 5개
  * 지난 7일 이내 push된 이미지
  
## 2. Chart override

> 선언적 vs 명령적
생성할 리소스를 미리 정의할 것인가, 아님 즉각적인 명령을 통해 진행할 것인가

개인적으론 추후 여러가지의 애플리케이션 배포를 위해선 선언적 방식으로 진행하는게 관리하기 용이할것 같다는 판단을 하였습니다.

### Kubernetes Manifest Template
리소스를 선언적 방식으로 생성하기 위해선 템플릿을 미리 만들어야 합니다.
인프라팀에선 `kustomize`와 `Helm chart` 두가지 방식을 추천해주었고, 난 `Helm chart` 방식을 선택하였습니다.

템플릿 방식이냐 상속방식이냐 차이인데, 템플릿을 만들어 놓고 root의 value 파일을 통해 여러가지 환경이나 애플리케이션을 대응할 수 있는 방식이 맘에 들어 선택하였습니다. 물론 러닝커브는 더 높다고 합니다.

### Helm Chart Template
```
📦helm
 ┣ 📂sample
 ┃ ┗ 📜values.yaml
 ┣ 📂templates
 ┃ ┣ 📜deployment.yaml
 ┃ ┣ 📜hpa.yaml
 ┃ ┣ 📜ingress.yaml
 ┃ ┣ 📜secrets.yaml
 ┃ ┣ 📜service.yaml
 ┃ ┣ 📜serviceaccount.yaml
 ┃ ┗ 📜_helpers.tpl
 ┣ 📜Chart.yaml
 ┗ 📜values.yaml
```
위는 현재 구성된 `Helm chart` 템플릿의 모습입니다.

최초 `helm create mychart` 명령어를 통해 템플릿을 생성한 뒤에 내 입맛에 맞게 조금 변형을 하였습니다.

변형한 부분은 다음과 같습니다.
```yaml
# values.yaml
#  컨테이너별 리소스 정의
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi 
#  FE개발 노드로만 배포되도록 label 정의
affinity:
  nodeAffinity:
      # required
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node
            operator: In
            values:
            - sri-fe
            
# deployment.yaml
#  컨테이너 헬스체크, 비정상이라고 판단되면 재시작
#  initialDelaySeconds: 컨테이너 기동 이후 설정값 만큼 대기
#  periodSeconds: 주기
livenessProbe:
  initialDelaySeconds: 15
  failureThreshold : 3
  periodSeconds : 10
  tcpSocket:
    port: 3000
#  컨테이너 헬스체크, 비정상이라고 판단되면 서비스에서 제외
readinessProbe:
  initialDelaySeconds: 15
  failureThreshold : 3
  periodSeconds : 10
  tcpSocket:
    port: 3000
```

기본적인 부분은 [가이드문서](https://helm.sh/ko/docs/chart_template_guide/getting_started/)를 참고하면 될것 같고,

전 환경별, 서비스별로 분리된 values 파일을 생성하기 위해 `sample/` 폴더를 생성해서 초기 `value.yaml` 파일을 위치해 두었습니다.

이후 파이프라인이 동작할때 해당 파일을 기준으로 변형해서 배포하도록 구성하였습니다.

### Gitlab-ci 구성
#### 설계
위에서 작성된 `helm chart`를 어떻게 사용하여 컨테이너 이미지를 `k8s` 에 배포할 것인지에 대한 부분입니다.

우선 `Helm chart`는 별도의 repo로 분리를 하였습니다.

`ArgoCD`는 구독중인 `k8s Manifest yaml`의 변화를 통해 작동하게 되는데 프로젝트 소스코드의 변화엔 작동하지 않게 하기 위함이였다. 물론 `repo`안에서 폴더를 구분하여 관리할 순 있겠으나 실제 `value.yaml`을 까보면 브랜치와 HEAD 위치까지 설정해줘야 하기 때문에 불필요한 정보는 거를 필요가 있었습니다.

파이프라인이 동작하면 `helmchart` repo를 내려받고 그 안에서 `value.yaml` 파일을 가공한 뒤 이걸 `ArgoCD`에서 감지할 수 있게 하려고 합니다.

#### 스크립트 작성
```bash
# Anchor - helmcahrt values 생성(업그레이드)
.upgrade-yaml: &upgrade-yaml
  - cp ./sample/values-${ENV_NAME}.yaml ./sample/$VALUES_PATH
  - >
    yq -i '
    .nameOverride = "'${SERVICE_NAME}-$CI_COMMIT_REF_SLUG'" |
    .fullnameOverride = "'${SERVICE_NAME}-$CI_COMMIT_REF_SLUG'" |
    .image.repository = "'$CI_REGISTRY/fe/${APP_DIR}/${SERVICE_NAME}/$CI_COMMIT_REF_SLUG'" |
    .image.tag = "'$CI_COMMIT_SHORT_SHA'" |
    .imagePullSecrets[0].name = "'$CI_COMMIT_REF_SLUG'" |
    .secretesName = "'$CI_COMMIT_REF_SLUG'" |
    .imageCredentials.registry = "'${CI_REGISTRY}'" |
    .imageCredentials.username = "'${CI_REGISTRY_USER}'" |
    .imageCredentials.password = "'${CI_REGISTRY_PASSWORD}'" |
    .imageCredentials.email = "'${GITLAB_USER_EMAIL}'" |
    .ingress.hosts[0].host = "'${SERVICE_URL}'"
    ' ./sample/$VALUES_PATH
  - cat ./sample/$VALUES_PATH
  - mv ./sample/$VALUES_PATH ./$VALUES_PATH
  - git add .
  - git commit -m "[skip ci] $CI_COMMIT_MESSAGE"
  - git push -u origin main
```
위 코드는 파이프라인 스크립트중 `chart override`를 하기 위한 Anchor 부분으로 내용은 아래와 같습니다.
- `./sample` 폴더의 values 파일을 목적에 맞는 파일명으로 복사한 뒤 yaml 파일을 알맞게 수정
- `.image.repository`와 `.image.tag`를 새로 생성된 컨테이너 정보에 맞게 변경
- `.secretesName`에는 repository에 접근하기 위한 인증정보의 이름을 정의
- `.imageCredentials.*`은 인증정보에 필요한 값을 주입
- 변경된 내역을 chart repo에 push
- (주의) 최초 value 파일을 root로 복사한뒤 수정하지 않고 원 위치에서 복사&수정 한 뒤 복사하는 이유는 해당파일을 `ArgoCD`가 구독하고 있기 때문이고, 하나의 구문으로 `create`와 `upsert`를 같이 하기 위함

위와 같이 작성한 구문을 공통화시켜서 review 서버를 생성했을때도 사용할 수 있도록 하였습니다.

## 3. App create & update
자 이제 `ArgoCD`에 `application`을 생성할 차례입니다.

기본적으로 `application`은 `ArgoCD` 대시보드에서 생성&관리 등이 가능합니다.
그렇지만 `GitOps`로 구성하고 있기 때문에 `cli`를 통해 생성해야 합니다.

`application`, `application set` 2가지 방식으로 생성할 수 있지만, 현재는 front 단독 서비스만 배포하려 하기 때문에 `application` 생성방식으로 진행했습니다.

### application.yaml
해당 파일은 `application`을 선언적인 방식으로 배포하기 위해 생성하였고 `HelmChart` 와 같은 repo에 위치하였습니다.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: # 기본 정보
  name: "web1-develop"
  namespace: ""
  labels: # 필요시 생성, 여기선 환경별 구분을 식별하기 위해 생성
    branch: ""
spec:
  # project : ArgoCD에서 생성한 프로젝트 이름
  project: frontend
  
  # source : application manifests, helmchart repo 정보
  source:
    repoURL: '${helm chart repo url}'
    path: .
    targetRevision: HEAD
    helm:
      valueFiles:
        - ""

  # Destination cluster and namespace to deploy the application
  destination:
    server: '${kube cluster url}'
    namespace: ""
  
  # Sync policy
  syncPolicy:
    automated:
      prune: true  # git에서 리소스를 삭제하면 kube에서도 삭제되도록 
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
```

### Gitlab-ci 구성
#### 설계
`applicatioin.yaml` 파일은 위 `helm chart` 배포할때와 유사한 과정으로 진행합니다.

파이프라인이 동작하면 앞에서 내려받은 소스들에서 `applicatioin.yaml` 파일을 사용하여 `app create`를 진행합니다.

#### 스크립트 작성
``` yaml
# Anchor - argocd application 생성(업그레이드)
.create-argocd-app: &create-argocd-app
  - >
    argocd app create web1-${ENV_NAME}
    --server $ARGOCD_SERVER
    --grpc-web
    --auth-token $ARGOCD_TOKEN
    --file ./argocd/application-${ENV_NAME}.yaml
    --name ${SERVICE_NAME}-$CI_COMMIT_REF_SLUG
    --label branch=$CI_COMMIT_REF_SLUG
    --values $VALUES_PATH
    --dest-namespace ${SERVICE_NAME}
    --upsert
    --loglevel debug
```
위 코드는 ArgoCD에 app create & update를 하는 부분이다.
flags 값들을 살펴보면 아래와 같습니다
- application.yaml 파일을 사용하기 위해 `app create` 명령어 뒤에 `app.name`을 명시해주고 `--file` 에 경로를 추가
- `--server`, `-auth-token` 정보를 넣어줘서 ArgoCD 접근을 허용
- `--name`, `--label` 값을 통해 환경에 맞는 `app.name`을 재정의
- `--values` 는 yaml 파일명
- `--dest-namespace`를 추가하여 사용한 클러스터가 생성한 서비스명을 명시해주고
- `--upsert` 명령어를 통해 생성된 app 이라면 update를 진행토록 정의

여기까지 진행하면 git의 역할은 끝났습니다.

이후는 ArgoCD 에서 진행상황을 볼 수 있고, 실제 컨테이너가 적용되는 시점은 ArgoCD 내부에 정의된 주기(기본 3min)마다 `value`파일을 체크해서 컨테이너 이미지의 `tag`가 변경되면 새로 이미지를 받아서 `pod`를 재구성하면서 `deploy`를 진행하게 됩니다.

### 끝맺으며
지금까지 GitOps를 기반에 두고 있는 배포환경을 구성해봤습니다.

개인적으론 막연하게 알고 있던 docker, k8s, horbor..
그리고 이번에 처음 접한 kaniko, ArgoCD 등에 대해서 조금은 알게 된것 같습니다.

이 포스팅에 디테일하게 기술하진 못했지만 처음 한번만 구성해놓으면 이후 서비스가 추가 될때마다 여기 연동만 시켜도 배포될 수 있도록 모듈화 작업도 같이 진행을 했습니다.

gitlab-ci를 통해 일반적인 파이프라인을 구성해본건 햇수로 7년째가 되었는데, 모듈화 과정을 진행하려다 보니 이렇게까지 디테일하게 파고든건 또 처음이였던 같습니다.

이젠 어느 누가 담당해도 잘 운영될 수 있도록 환경을 만드는 숙제가 남아있습니다.
전반적인 내용을 팀원들에게 리뷰하고 메뉴얼 잘 만들어 놓으면 되지 않을까 싶습니다.

지금까지 읽어주셔서 감사합니다.


**스페셜 땡스 진주, 형규**
