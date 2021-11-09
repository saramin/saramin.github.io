---
title: 어느날 갑자기 배포가 되지 않았다.
author: 박금주
subtitle: 사람인 GitLab-Runner
tags:
- gitlab-runners
- group-runners
- 파이프라인
- yml
- 배포
---

# 어느날 갑자기 배포가 되지 않았다.

안녕하세요, 사람인 서비스를 개발하는 박금주 연구원입니다.

사람인은 상시배포가 아니라 정기배포로 서비스를 운영하기에 배포에 대한 구심축인 배포관리자가 필요합니다. 무분별한 배포를 막아 사이드이펙트에 대한 피로도를 줄이는 장치입니다. 지난 1년 간 배포관리자로서 겪었던 이슈 중 하나를 소개하고자 글을 남깁니다.

혹시 워크플로우에 대해 궁금하신 분들은 작년에 적었던 글을 참조해주세요.

[사람인 GIT 전환](https://saramin.github.io/2020-11-17-SSG/)

## GitLab 파이프라인이 돌지 않아요!

  예년처럼 평소와 동일하게 프로젝트 개발을 하던 중 CI/CD가 제대로 동작하지 않는다는 제보를 받았습니다. 특정 리포지토리의  파이프라인이 제대로 동작 하지 않던 겁니다. 수명업무도 중요하지만, 배포관리자로서 CI/CD 이슈를 간과할 수 없으니 원인 분석을 시작했습니다. 

   배포 서버에 문제가 발생했는지 살펴봐도, 쉘 스크립트를 뜯어봐도 그 이유를 알 수 없었습니다. 서버도, 스크립트도 모두 문제가 없었습니다. 범위를 넓혀 분석해보니 하나가 딱 손에 집혔습니다. 바로 CI/CD를 구동하기 위한 runner입니다. 누군가가 GitLab을 정비하던 중 `GitLab runner`를 건들어서 생긴 이슈였습니다. `GitLab runner`가 가진 특성을 고려하지 않아 발생한 문제입니다. CD/CD를 위한 yml에서 지정한 `runner` `tag`가 사라져 파이프라인이 실행되지 않았던 겁니다.

   원인을 찾았으니 이슈는 금방 해결했습니다. 하지만 동일한 문제가 똑같이 불거지지 않을 거란 확신을 가질 수 없었습니다. 어떻게 하면 동일한 이슈가 발생하지 않을지 고심하였습니다. 근본적인 원인을 해결하지 않으면 이번처럼 불미스러운 일이 발생하지 않을거라는 걸 확신할 수 없었습니다. 해결 방법은 멀지 않은 곳에 있었습니다. 이번 이슈는 `Shared runner` 문제입니다. 서로 runner를 공유하다가 다른 리포지토리에서 쓰는 `tag`를 지우게 되면 동일한 문제가 불거집니다. `Shared runner`는 편리하지만 연쇄적인 문제를 일으킬 수 있는 위험 부담을 감당해야 합니다.

## 해결 방법은 `Group`에 있습니다.

   이를 방지 하기 위해 `Shared runner` 대신 `Group runner`로 변경하였습니다. `Group runners`는 문제가 발생하더라도 동일한 `Group` 내에서 발생하기에 원인 파악 범위를 줄일 수 있습니다. 다른 Group에 영향을 주지 않습니다. 또한 `Group` 내 생성되는 리포지토리는 별도로 `runner`를 새롭게 설정하지 않아도 `Group runner`를 바로 사용할 수 있다는 장점을 가졌습니다. `Shared runner`와 `Specific runner`의 장점을 고루 갖습니다. `GitLab runner`를 살펴본 김에 다른 개발자 분들도 알아두었으면 유용하겠다는 생각이 들어 간단하게 정리해보았습니다. 같이 GitLab CI/CD(이하 GitLab CI)를 살펴볼까요? 

# GitLab Runners

   GitLab CI는 yml에 정의한 조건에 따라 CI를 구동합니다. `runner`는 앞서 이야기한대로 세 가지 액세스 유형이 존재합니다. 어드민 권한을 가진 계정으로 runners 메뉴에 들어가면 아래와 같이 `Shared`, `Group`, `Specific` 을 확인 가능합니다.

![<img src="/img/gitlab-runner/gitlab-admin.png" />](/img/gitlab-runner/gitlab-admin.png)

### 공유 러너`Shared runners`

   `Shared runners`는 GitLab 인스턴스의 모든 그룹과 프로젝트에서 끌어다 사용합니다. 이름 그대로 전체 공유하는 `runner`입니다. 어떤 리포지토리든 상관하지 않고 다 같이 공유해서 사용 할수 있다는 장점은 곧 단점이기도 합니다. 이번 사건처럼 `tag`가 삭제 되거나 권한에서 제외되면 해당 `Shared runner`를 사용 하는 모든 리포지토리에 영향을 끼칩니다. 지나친 의존성이 발생하여 함부로 수정하거나 삭제할 수 없는 `runner`가 되어버립니다.

### 그룹 러너`Group runners`

   GitLab은 `Group`를 생성하여 사용자를 관리하는 기능이 있습니다. `runner`도 `Group`에 등록하면 해당 그룹에서 생성하는 리포지토리와 하위 그룹만 공유합니다. `Shared runners`가 가진 지나친 의존성을 `Group`으로 한정짓게 됩니다. 이 방식으로 등록된 `runner`는 하위 리포지토리에서 별도 `runner`를 생성하지 않고 사용 가능합니다.  다른 `Group runners`와 `Shared runners`가 삭제된다고 하더라도 영향이 받지 않게 됩니다.

### 개인화 러너`Specific runners`

   `Specific runners`는 특정 리포지토리에 등록한 `runner`를 일켣습니다. `runner` 한 대당 단일 리포지토리를 관리하게 되죠. 리포지토리 하나만 관리하고자 할 때 사용하는 옵션입니다. `Shared runners`, `Group runners`가 공유 기능은 리포지토리마다 `runner`를 새롭게 등록하지 않아도 되기에 편리하지만, 그만큼 위험성이 내제됩니다. `Specific runners`는 번거롭지만, 그 책임을 명확히 가려냅니다.

## GitLab Runner 설정

   간단하게 Group runner를 등록하는 방법에 대해 살펴봅시다. 사실 셋 다 별반 다르지 않아요.

### 1. GitLab 메뉴 → Groups → Your groups

![그룹 러너 메뉴](/img/gitlab-runner/gitlab-menu.png)

### 2. 원하는 Group를 선택합시다.

![ci/cd 셋팅](/img/gitlab-runner/your-groups.png)
### 3. Settings → CI/CD
![그룹 러너](/img/gitlab-runner/gitlab-lnb.png)


CI/CD 메뉴에서 `Runners`로 접근하면 `Group runners`에 등록할 URl과 토큰 정보가 보입니다. 이 값은 서버에서 `runner`를 생성할 때 필요합니다.

![ci/cd](/img/gitlab-runner/group-runners.png)
### 4. yml 작성

   GitLab CI는 yml을 통해 구성되고 동작하므로 이를 작성해야 합니다. 이처럼 코드로 작성하면 파이프라인 또한 자연스레 버전관리가 이루어집니다. 설정에 따라 사용자 입맛에 맞는 파이프라인을 구성됩니다. 아래 yml는 사람인 리포지토리에서 사용하는 키워드를 바탕으로 만든 예시입니다. 주석으로 키워드 설명을 작성해두었어요.

```yaml
stages:         
  - build
  - test
  - deploy
  - cleanup

.create: &create
  stage: deploy
  script:
    - echo "create"
  tags:
    - shared-runners-manager-4.gitlab.com

build-job:      
  stage: build
  script:
    - echo "Compiling the code..."
    - echo "Compile complete."

test-stg2:
  stage: test
  script:
    - echo "This job tests something"

create to stg2:
  stage: deploy
  variables:
    HOST_NAME: "stg2"
    SERVER_NAME: $STAGE_2
  rules:
# 특정 조건일때 동작 하록록 한다.
# master 브랜치가 아니고 push 참인 조건에서 동작 한다.
    - if: '$CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH != "master"'
  <<: *create #.create: &create 사용한다.
  environment:
    name: deploy/stg2  # delete to stg2 name 값 동일 하게 맞춰 준다.
    url: https://www.test.co.kr # delete to stg2 url 값 동일 하게 맞춰 준다.
    on_stop: delete to stg2
  tags:
    - shared-runners-manager-4.gitlab.com

sync to stg2:   # 임의로 지어준 job 이름을 사용 한다.(ci/cd에 표현 하고 싶은 이름으로 사용)
  stage: deploy
  variables:
    HOST_NAME: "stg2" #서버 이름
  script: # runner에 의해 수행될 쉘 스크립트를 작성한다.
    - echo "Running unit tests... This will take about 60 seconds."
    - sleep 60
    - echo "Code coverage is 90%"
  only: #master 브랜치에 push가 들어 올떄 동작하는 job
    - master
  tags: 
# 특정 태그가 달린 runner에 명령을 내릴 수 있다. 
# (여기서는 shared-runners-manager-4.gitlab.com 태그를 가진 runner에 명령을 내린다.)
    - shared-runners-manager-4.gitlab.com

sync to stg3:
  stage: deploy
  when: manual # manual : 수동  , 설정 하지 않는 다면 자동 sync 된다
  variables:
    HOST_NAME: "stg3" #서버 이름
  script: # runner에 의해 수행될 쉘 스크립트를 작성한다.
    - echo "Linting code... This will take about 10 seconds."
    - sleep 10
    - echo "No lint issues found."
  only:  #master 브랜치에 push가 들어 올떄 동작하는 job
    - master
  tags:  
# 특정 태그가 달린 runner에 명령을 내릴 수 있다. 
# (여기서는 shared-runners-manager-4.gitlab.com 태그를 가진 runner에 명령을 내린다.)
    - shared-runners-manager-4.gitlab.com

sync to stg4:
  stage: deploy
  when: manual # manual : 수동  , 설정 하지 않는 다면 자동 sync 된다
  variables:
    HOST_NAME: "stg4"
  script:
    - echo "Deploying application..."
    - echo "Application successfully deployed."
  except: #작업이 생성 되지 않는 시점을 정한다  -master 브랜치 시점에서는 보이지 않는다.
    - master
  only:
    - master
  tags:
    - shared-runners-manager-4.gitlab.com

delete to stg2:
  stage: cleanup
  script:
    - echo "Linting code... This will take about 10 seconds."
  when: manual
  environment:
    name: deploy/stg2
    action: stop
  tags:
    - shared-runners-manager-4.gitlab.com
```

   이 외에도 GitLab CI는 다양한 키워드를 제공합니다. 하나하나 짚고 넘어가기엔 내용이 방대하여 자세한 건 공식 문서를 참조해주세요.

[ GitLab Build Cloud runners | GitLab ](https://docs.gitlab.com/ce/ci/runners/index.html)

### 5. 파이프라인 확인

   GitLab CI는 yml 파일에 따라 파이프라인을 자동으로 그려줍니다. 의도한 대로 파이프라인이 생성되었는지 확인합시다.

![파이프라인 기본](/img/gitlab-runner/pipeline.png)

브랜치 생성 파이프라인 예시

![git push 파이프라인 예시 ](/img/gitlab-runner/pipeline-2.png)

git push 파이프라인 예시 

## Git Lab runner 생성

   GitLab에서 제공하는 `public runner`도 존재하지만, `runner`에 대한 제약을 받지 않으려면 `GitLab runner`를 생성할 서버가 필요합니다. 해당 서버에서 아래와 같이 코드를 실행하면 가볍게 `runner`가 만들어집니다.

```bash
$ sudo gitlab-runner register 

Runtime platform arch=amd64 os=linux pid=3668 revision=943fc252 version=13.7.0 Running in 
system-mode.

# gitlab 주소 (Runner에 있는 url 을 입력 해준다).
Enter the GitLab instance URL (for example, https://gitlab.com/): 
https://gitlab.com/

# token 입력
Enter the registration token: 
(group runner 토큰값)

# runner 설명 
Enter a description for the runner: 
(간략하게 nunner 대한 설명을 넣어준다)

# runner tag 지정, tag에 따른 실행러너가 달라짐 
Enter tags for the runner (comma-separated): Registering runner... succeeded runner=yFecrSgp 
(yml 파일에 있는 tags 명을 넣어준다.)

# runner 실행 형식 
# 실행형식에 따라 입력 정보가 다름 
Enter an executor: parallels, shell, virtualbox, docker+machine, docker-ssh+machine, kubernetes, custom, docker, docker-ssh, ssh: 
shell (필자 경우에는 shell을 사용)

# 등록 성공 
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```

[Registering runners  GitLab]  (https://docs.gitlab.com/runner/register)


   등록이 끝나면 GitLab runner 메뉴에 표기됩니다. `Shared runners`, `Group runners`, `Specific runners` 전부 runner를 등록하는 방식은 동일해요.

![러너 등록 화면](/img/gitlab-runner/runner.png)

## 글을 마치며

   Git은 개발자에게 참으로 익숙한 도구입니다. 버전관리시스템 없이 개발한다는 건 상상할 수 없는 일이 되어버렸죠. 개발 환경을 구성할 때도 IDE만큼 GIT를 가장 먼저 설치합니다. 하지만 정작 Git을 활용하는 GitHub 또는 GitLab과 같은 매니징시스템에 대해 깊에 고민해보지는 않았습니다.

   너무 가깝기에 놓쳤던 건 아닐까요? 늘상 사용했으니 스스로 '이건 잘 알지.'라고 착각했겠죠. 이번 기회에 GitLab Runner에 파악한 계기가 되었습니다. 익숙하기에 익숙한 만큼 되돌아야 한 번쯤 되돌아봐야겠어요. 

글을 끝까지 읽어주셔서 감사합니다.
