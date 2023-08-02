---
title: 우리는 이렇게 코드리뷰 합니다
author: 선아리
---

# 우리는 이렇게 코드리뷰 합니다

안녕하세요.

이번 글에서는 현재 사람인개발팀에서 진행하고 있는 코드리뷰 문화에 대해서 얘기해보려고 합니다.
코드리뷰를 검색하면 수많은 검색결과가 노출되듯, 이미 많은 기업에서 진행하고 있어요. 코드리뷰의 목적은 비슷하면서도 그 방식은 조직마다 다를텐데요.
코드리뷰 문화를 계속해서 개선하기 위한 노력은 진행중이지만, 그 경험을 공유해보고자 하니 가벼운 마음으로 읽어주세요!

그 전에 코드리뷰에서 자주 사용되는 용어에 대해 간단히 정의하겠습니다.

```SATA
💡 리뷰어 : 리뷰하는 사람
   리뷰이 : 리뷰를 받고자 하는 사람
```
<br><br>

### 이전의 코드리뷰 방식

사람인개발팀에서는 이미 예전부터 코드리뷰를 진행하고 있었습니다.
때는 바야흐로 SVN을 사용하던 시절에는 오프라인 코드리뷰를 진행하고 있었죠.
리뷰이가 코드를 직접 설명하면 리뷰어가 해당 내용에 대해서 의견을 내고 얘기한 내용을 리뷰이가 수기로 정리하여 코드에 반영한 이후에 다시 공유하는 방식으로 진행하였죠.

그리고 리뷰어는 `제비뽑기`를 통해서 지정하였습니다. 왠 제비뽑기냐? 리뷰어를 제비뽑기로 정하는게 맞느냐? 이런 얘기가 나올 수 있겠습니다만!
팀의 성격 중 하나가 서비스의 담당자를 지정하기보다는 다양한 서비스를 다루는 경험이 더 중요하다고 생각하는 점이 있어 이런 리뷰어 뽑기가 가능하였습니다. 또한 특정인에게 코드리뷰가 몰릴 수 있어서 이 부분을 해소하기 위함이기도 하였습니다.

이렇게 적고 나니 이미 단점이 눈에 보이는데요.
오프라인 코드리뷰의 경우 장소 선정을 하고 리뷰어와 시간을 맞춰서 진행해야 하기 때문에 그에 따른 제약이 있었습니다. 그리고 리뷰어의 의견을 수기로 정리했기 때문에 제대로 기록이 되지 않았습니다.

단점을 안고 코드리뷰를 진행하면서 그렇게 시간이 흘렀고, git 으로 변경한 이후에는 **`gitlab merge request`** 를 통한 온라인 코드리뷰를 진행하였습니다.
온라인으로 변경되고 나니까 단점이 모두 해소된 것 같았어요. 굳이 시간을 모두 맞춰서 회의를 잡지 않아도 되고, 오프라인으로 일일히 설명하던걸 하지 않아도 되었습니다.
<br><br>
### 문화 개선의 시작

그러나, 한해를 마무리하면서 다음 한해를 어떻게 보낼지에 대해 진행된 워크샵에서 팀원들이 코드리뷰에 대해 문제점을 제기하였습니다. 

제기한 문제점들은 어떤게 있었을까요? 크게 아래와 같았어요.

- 리뷰어로 지정한 사람 외에 참여도가 낮다.
- 기획서 확인과 코드 만으로는 다양한 관점에서 이슈 파악을 하기가 어렵다.
- 개발 완료 후 코드리뷰 진행으로는 범위가 너무 커서 리뷰가 어렵다.

우리는 하나씩 해결해나가보기로 하였습니다.

아! 그 이전에 코드 리뷰의 목적을 명확하게 정의하기로 하였습니다. 그에 따라서 방향이 조금씩 달라질 수 있기 때문입니다. 사람인개발팀의 코드리뷰 목적은 프로젝트 배포 이후 발생할 수 있는 문제를 최소화하는데에 초점을 맞추었습니다. 코드 리뷰를 통해 배포 이전에 잠재적인 문제를 사전에 파악하고 수정함으로써 실제 배포 후 발생하는 이슈를 최소화하고, 이를 통해 사용자 경험과 서비스 품질을 향상시키는 것을 최우선 목표로 하였죠.
<br>
자 이제 다시 문제점들을 보겠습니다.

**1. 참여도가 낮다.**

리뷰어 외에는 참여도가 낮다는 의견이였습니다.
사실 모두가 프로젝트를 참여하면서 다른 사람의 코드리뷰까지 확인하기는 쉽지 않습니다.

그렇다면 어떻게 해야할까를 고민했고, 결론적으로는 우리 모두 **`리뷰 메이트`**를 만들어보자! 까지 왔습니다.

**`리뷰 메이트`**란 무엇이냐. 1대1로 각자 서로의 코드를 확인하자는 개념입니다.
사람인개발팀은 현재 자리 배정이 시니어와 주니어의 조합으로 되어있습니다. 이 또한 팀원들의 의견을 통해서 정해졌습니다. 그래서 자리 기준으로 서로의 리뷰 메이트가 되기로 하였습니다.


 **2. 코드만으로는 이슈 파악이 어렵다.**

코드만으로는 이슈를 확인하기 힘들다는 얘기가 있었습니다. 코드의 오류 뿐 아니라 서비스의 정책도 확인해야 하는 부분이 있었죠. 그래서 리뷰 메이트가 기획 리뷰까지 참여하기로 하였습니다. 이렇게 보니 리뷰 메이트가 할일이 많네요.


**3. 범위가 큰 경우 리뷰가 어렵다.**

리뷰 메이트가 코드리뷰를 요청할때 뿐만이 아니라 수시로 커뮤니케이션을 하면서 프로젝트를 진행할거라 생각되어 이부분은 자연스럽게 해소되지 않을까 기대하였습니다.
<br><br>
### 현재의 코드리뷰 방식

위의 문제점들에서 나온 결론들을 종합해보면 사실상 리뷰메이트가 찰싹 붙어서 서로의 프로젝트를 봐줘야하는데, 일정 등의 현실적인 문제에 부딪히게 되었습니다. 본인의 프로젝트도 진행하면서 리뷰도 참여한다는건 쉬운 일이 아니었죠. 이런 문제들을 서로 보완하며 나온 진행 방향은 아래와 같아요.

- 한달 이상 소요되는 프로젝트 진행시 직책자가 리뷰어를 지정한다.
- 리뷰어가 지정되지 않은 경우는 서비스 도메인 전문가를 리뷰어로 지정하여 진행한다.
- 리뷰어는 기획 리뷰에 참여한다. (리뷰어의 일정에 따라)
- 혼자 프로젝트를 진행하는 경우 리뷰어가 지정되지 않았다면, 리뷰 메이트가 코드리뷰를 진행한다.
- 코드리뷰 문화에 대한 회고를 분기별로 진행한다.
- Pn룰을 도입한다.
<br>
<br>

```SATA
💡 P1: 꼭 반영해 주세요 (Request changes)
   P2: 적극적으로 고려해 주세요 (Request changes)
   P3: 웬만하면 반영해 주세요 (Comment)
   P4: 반영해도 좋고 넘어가도 좋습니다 (Approve)
   P5: 그냥 사소한 의견입니다 (Approve)
```
<br>

<img src="/img/codereview_comment.png" width="800px" />

물론 리뷰어만 코드리뷰를 하진 않습니다. 모두에게 코드리뷰 요청을 하되 리뷰어는 꼭 참여하자라는 의미로 설정합니다.

아! 그리고 배포 후 발생한 문제점을 수정하여 재배포 한 목록을 정리하기 시작하였습니다. 문제점의 원인 등을 정리하여 코드리뷰로 해결 가능한 부분인지 확인하고, 개선할 방향을 찾아가기 위해서요.
<br><br>
### 진행중 그리고 앞으로

이렇게 방향을 설정한 이후로 반년 정도 지난 지금, 한번의 회고를 진행하였고 얘기를 나누다 보니 리뷰메이트가 리뷰어로 설정될 일이 많지는 않았다는걸 알았어요. 대부분의 프로젝트가 두명 이상으로 진행되다보니 자연스럽게 프로젝트 담당자끼리 리뷰를 진행하게 되었죠. 물론 코드리뷰를 요청하기도 했는데, 보통 서비스 도메인 전문가가 리뷰어로 설정되었죠. 좀 더 리뷰메이트를 활성화 시킬수 있는 방안에 대해 고민해보기로 했어요. 그리고 꾸준히 회고를 진행하면서 코드리뷰 방향에 대해 문제점을 논의하고 해결할 수 있는 방안들을 만들어가려고 해요.

프로젝트마다 코드리뷰를 진행해야 하는 일은 너무 많고, 모두 참여하기에는 어려운 환경이라는 것은 어느 회사나 비슷한 상황일 것 같아요. 그럼에도 조금씩 나아지기 위한 노력을 꾸준히 이어간다면 더 좋은 결과를 가져올 것이라고 생각해요.

물론 어떻게 하느냐도 중요하지만, 가장 중요한 부분은 모두가 참여하는 것이라는 걸 느꼈어요. 더 발전된 코드리뷰 문화를 만들기 위해 함께 노력하여, 개선된 코드리뷰 문화로 다시 글을 쓸 수 있도록 하겠습니다.
읽어주셔서 감사합니다!
<br>

사람인개발팀 선아리