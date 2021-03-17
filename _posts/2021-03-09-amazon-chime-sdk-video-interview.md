---
title: Amazon Chime SDK를 활용한 영상면접 서비스 개발
layout: post
comments: true
social-share: true
show-avatar: true
author: 김갑훈
subtitle: 코로나19로 인한 면접방식의 변화
tags:
- AWS
- Chime
- SDK
- Interview
- VideoInterview
- COVID19
- WebRTC
---

## 들어가며
안녕하세요. 사람인 신규 서비스 개발 및 운영을 담당하고 있는 개발1팀 김갑훈입니다.<br><br>
코로나19로 인하여 많은 기업들이 재택근무를 시행하고 있으며 이와 더불어 비대면 서비스의 도입 및 활용이 증가하고 있습니다.<br>
이에 따라 채용 과정의 필수 요소인 면접도 비대면으로 전환되고 있는 추세입니다.<br><br>
사람인에서는 이러한 변화에 대응하기 위해 비대면으로 면접을 진행할 수 있는 영상면접 서비스를 2020년 9월 런칭하여 서비스하고 있습니다.<br>
서비스 런칭 후 시간이 많이 지났지만 늦게나마 영상면접 서비스를 개발하고 적용한 내용을 공유하려고 합니다.<br>

```SATA
서비스 구현에 대한 전체적인 내용을 작성하기에는 양이 너무 많아
이 글에서는 Amazon Chime SDK(이하 Chime SDK)를 활용한 부분을 중점적으로 작성하였습니다.
```

현재 영상면접 서비스는 PC에서만 제공되고 있으며 모바일 지원 및 채팅 기능 등 다양한 기능이 추가될 예정입니다.

--------------------------------------------------------------------------------------------------------------------------------------

## 서비스 흐름
사람인에서는 기업회원이 등록한 공고의 지원자 또는 인재Pool에 이력서를 공개한 개인회원에게 면접일정을 요청하고 요청 받은 개인회원은 면접일정 확인 후 수락/거절하여 확정된 면접일에 알림을 보내주는 등의 면접일정 관리 기능을 제공하고 있었습니다.<br><br>
이번에 구현하게 된 영상면접 서비스는 기존 면접일정 관리 기능에 영상면접을 추가하여 해당 면접일에 영상면접 서비스와 연동되는 형태로 구현되었습니다.<br>
또한 기존 공고 지원자, 인재Pool 공개 회원뿐만 아니라 이름/이메일/휴대폰 번호를 입력하여 면접 대상자를 등록하는 단독진행 형태도 추가되었습니다.<br><br>
면접 대상자별 서비스 흐름은 아래와 같습니다.<br>
[<img src="{{site.url}}/img/video-interview/flow-applicant.png" />]({{site.url}}/img/video-interview/flow-applicant.png)
[<img src="{{site.url}}/img/video-interview/flow-direct.png" />]({{site.url}}/img/video-interview/flow-direct.png)

면접 입장부터 면접 종료 프로세스까지 영상면접을 위한 별도 도메인으로 구성되며 Chime을 활용하여 구성됩니다.

--------------------------------------------------------------------------------------


## 기술 검토
영상면접 서비스를 위해서는 사용자간 음성, 영상 미디어를 송수신하는 기술이 필요합니다.<br>
웹상에서 서비스하기 위해서는 WebRTC 기술을 활용하여 구현해야 합니다.<br>
> WebRTC(Web Real-Time Communications)란, 웹 어플리케이션 및 사이트들이 별도의 소프트웨어 없이 음성, 영상 미디어 혹은 텍스트, 파일 같은 데이터를 브라우저끼리 주고 받을 수 있게 만든 기술입니다.<br>
> WebRTC로 구성된 프로그램들은 별도의 플러그인이나 소프트웨어 없이 P2P 화상회의 및 데이터를 공유합니다.<br>

WebRTC 기술을 활용하여 모든 기능을 자체적으로 구현하려면 시그널 서버 등 많은 부분을 구현해야 하며 해당 기술을 구현해 본 경험이 없어 많은 시간이 필요했습니다.<br>
그래서 저희는 해당 기술을 제공하는 서비스를 검토하였으며 아래와 같은 사항을 고려하였습니다.
* 지원 개발언어
* 구현 형태 (API, SDK 등)
* 과금 형태
* UI 커스터마이징 가능 여부
* 녹화 가능여부
* 서비스 안정성

알서포트, 말톡노트, 구루미, 유프리즘, Amazon Chime SDK 등의 서비스를 검토하였으며 아래와 같은 이유로 Amazon Chime SDK을 이용하기로 결정하였습니다.<br>
* SDK 제공으로 인한 UI 등 다양한 형태 커스터마이징 가능
* 실제 사용시간에 따른 과금
    * 서비스 초기 이용 수요 확인 시에는 월 단위 일정 금액 과금보다 장점
* AWS Kinesis 서비스를 활용하여 실시간 영상 저장 및 분석 서비스 구축 가능
* AWS 내 SaaS형 서비스와 연계하여 확장 용이
* 서비스 안정성 높음

--------------------------------------------------------------------------------------

## Amazon Chime SDK
Amazon Chime은 Amazon 웹사이트 상에서 화상회의 및 전화통화 서비스를 제공하고 있으며<br>
AWS Chime SDK를 이용하여 오디오/비디오를 송수신하고 컨텐츠를 공유할 수 있는 실시간 미디어 애플리케이션을 구현할 수 있습니다.<br>
AWS Chime 관리자 계정과 독립적으로 작동하며 Amazon Chime에서 진행하는 미팅과는 연동되지 않습니다.<br>

Chime SDK는 2019년 11월에 1.0.0 버전이 최초로 릴리즈 되어 국내외 레퍼런스가 많지 않은 상황으로<br>
대부분 Amazon에서 제공하는 [공식 문서](https://docs.aws.amazon.com/chime/latest/dg/meetings-sdk.html){:target="_blank"}를 참고하였습니다.<br>


Chime SDK에서는 기본적으로 Meeting과 Attendee라는 개념이 존재합니다.<br>
화상회의를 예로 들면 진행되는 회의 자체가 Meeting과 동일하며 회의에 참석하는 참석자가 Attendee와 동일합니다.<br>

애플리케이션 구현 시 [API](https://docs.aws.amazon.com/chime/latest/APIReference/API_Operations.html){:target="_blank"}를 통해 Meeting 세션과 Attendee 세션을 생성해야 합니다.<br>
<br>
### 사전 준비사항
Chime SDK를 구현하기 위해서는 아래 사항이 필수로 준비되어야 합니다.
* AWS 계정
* Amazon Chime API actions 권한이 부여된 정책을 가진 IAM role
    * AmazonChimeFullAccess
    * AmazonChimeUserManagement
    * AmazonChimeReadOnly
    * AmazonChimeSDK
* Server Application
    * API를 통해 Meeting 세션 및 Attendee 세션을 관리(등록/수정/삭제)하고 해당 리소스를 Client Application에 제공하는 역할
    * AWS SDK를 이용하여 구현하고 다양한 언어 지원
        * Javascript, Python, PHP, .NET, Ruby, Java, Go, Node.js, C++
        * Android, iOS 등
* Client Application
    * Server Application에서 전달 받은 리소스를 이용하여 화면 구현 및 미디어 리소스(오디오/비디오 등)를 제어
    * AWS SDK client library를 이용하여 구현하고 Javascript, Android, iOS 언어 지원

<br>
### 사용 가능 Region
다른 AWS 서비스와 마찬가지로 Chime SDK를 이용하기 위해서는 사용할 Region을 선택해야 합니다.<br>
현재 기준 [총 18개 Region](https://docs.aws.amazon.com/chime/latest/dg/chime-sdk-meetings-regions.html){:target="_blank"}을 지원하며 2020년 8월 서울 리전이 추가되었습니다.<br>
<br>
### 요금
요금은 선결제 비용 없이 사용한 만큼 지불하는 요금제를 제공합니다.<br>
각 참석별 분당 0.0017 USD가 부과되며 Meeting 세션에 연결되는 시점부터 과금됩니다.<br>

예를 들면 1개의 Meeting에 4명의 참석자가 참여하여 총 1시간을 진행한 경우 비용은 다음과 같습니다.<br>
> 4( 참석자) * 60(진행시간) * 0.0017 = 0.408 USD

<br>
### 제한사항
대부분의 서비스들처럼 Chime SDK도 사용상 제한사항이 존재합니다.<br>

**사용 한도**
* 최대 동시 Meeting 진행 가능 수 : 250 (조절 가능)
* Meeting당 최대 참석자 수 : 250
* Meeting당 최대 오디오 스트림 갯수 : 250
* Meeting당 최대 비디오 스트림 갯수 : 16 (조절 가능)
* Meeting당 최대 콘텐츠 공유 갯수 : 2
* API 호출 : 초당 최대 10개 요청 (조절 가능)

```SATA
사용 한도 증가가 필요한 경우 AWS측에 요청하면 사용량을 모니터링하여 반영한다고 하며
조절 가능으로 표기된 항목만 요청 가능합니다.
```

**Meeting 자동 종료**<br>
Meeting 세션 생성 후 아래와 같은 상황이 되면 자동 종료되고 해당 세션은 더 이상 사용할 수 없습니다.
* 5분 이상 오디오 연결이 없는 경우 자동 종료
* 30분 이상 비디오 연결이 없는 경우 자동 종료
* Meeting 생성 후 24시간 경과 시 자동 종료
* Meeting 참석자가 1명인 상태로 30분 동안 오디오 연결이 없는 경우 자동 종료
* Meeting 참석자가 여러명이지만 6시간 동안 오디오 상태 변경이 없는 경우 자동 종료

비디오는 CPU 및 대역폭 가용성에 따라 최대 1280x720 해상도까지 사용 가능합니다.<br>
또한 비디오 제한은 카메라 성능에 따라 달라질 수 있습니다.<br>
<br>
### 지원 브라우저 및 OS
JavaScript를 이용하여 Chime SDK 구현 시 WebRTC 기술 사용으로 인해 구형 IE 브라우저에서는 사용할 수 없습니다.<br>
Chrome, Firefox, Safari, Edge, Opera 등 IE 브라우저를 제외한 대부분의 브라우저에서 사용 가능하지만<br>
브라우저별 최소 지원 버전이 존재하며 일부 브라우저는 콘텐츠 공유 등 일부 기능 사용이 불가능합니다.<br>

모바일앱으로 구현 시 iOS는 10.0 이상 Android는 5.0 이상 버전이 필요합니다.<br>

자세한 내용은 [여기](https://docs.aws.amazon.com/chime/latest/dg/meetings-sdk.html#mtg-browsers){:target="_blank"}에서 확인하실 수 있습니다.<br>


--------------------------------------------------------------------------------------

## 프로토타입 구현
기술 검토를 진행한 시점에서는 큰 틀에서 상위 기획만 나온 상태였고 상세기획이 작성되기 위해서는 Chime SDK를 이용해 구현 가능한 기능 파악이 필요하였습니다.<br>
그래서 저희는 프로토타입을 먼저 구현해보기로 하였습니다.<br>

구현 목표를 다음과 같이 설정하고 진행하였습니다.<br>

**사람인**
* 기업회원 면접관리 페이지 내 면접일정 등록 시 화상면접 생성 프로세스 구현
* 개인회원 화상면접 가능 리스트 구현 및 화상면접 입장 프로세스 구현


**영상면접 (별도 도메인)**
* 사람인 로그인 연동
* 면접방 입장 시 로그인 체크 및 면접 참여 가능 유효성 체크
* 최초 진입 시 사용할 마이크/카메라/스피커 등 선택 화면 구현
* 면접방 내 접속자 리스트 제공
* 면접방 내 마이크/카메라/스피커 등 기기 변경 및 ON/OFF 기능 구현
* 면접방 내 화면 공유 기능 구현
* 면접방 내 비디오 노출 화면 구현
* 현재 접속한 사용자는 하단에 표기
* 다른 참여자는 상단에 표기
* 면접방 나가기, 종료 기능 구현
* 면접방에 면접관으로 입장 시 응시자 이력서 노출 구현
* 면접방에 응시자로 입장 시 동의 화면 노출 후 입장하도록 구현

영상면접 서비스의 Server Application은 사람인 사이트와 로그인 연동이 필요하여 PHP 기반의 Laravel Framework를 이용하여 구현하였으며<br>
Client Application은 Laravel Framework 내 mix를 이용하여 스크립트를 build하여 사용하였으며 React, Vue.js 등 framework은 사용하지 않고 Native로 작성하였습니다.<br><br>
참고로 WebRTC 기술을 이용하기 위해서는 서버 구성 시 https 프로토콜을 사용하여야 합니다.<br><br>
Chime SDK 구현은 [Amazon Chime SDK for JavaScript](https://github.com/aws/amazon-chime-sdk-js){:target="_blank"} 에서 제공하는 Sample을 참고하였습니다.<br>
UI 구성은 프로토타입 취지에 맞게 간단히 구현하기 위해 Bulma CSS Framework를 사용하였습니다.<br>

### 구현 화면
**기기 선택 화면**
[<img src="{{site.url}}/img/video-interview/prototype-device.png" />]({{site.url}}/img/video-interview/prototype-device.png)

**면접 메인**
[<img src="{{site.url}}/img/video-interview/prototype-main.png" />]({{site.url}}/img/video-interview/prototype-main.png)

실제 구현 전에 프로토타입을 먼저 구현해보니 Chime SDK에서 제공하는 기능을 미리 파악하여 기획 방향을 결정하는데 도움이 되었으며<br>
프로토타입에서 구현한 코드를 실제 구현 시 참고하여 개발 방향 결정 및 개발 품질 향상에도 도움이 되었던 것 같습니다.<br>

--------------------------------------------------------------------------------------

## 서비스 구현
Server Application은 프로토타입과 동일한 이유로 PHP 기반의 Laravel Framework를 이용하였습니다.<br>
Client Application은 Vue.js를 사용하였으며 상태 관리를 위해 Vuex도 추가로 사용하였습니다.<br>
SPA 형태로 구현하였으며 Vue Router를 사용하는 대신에 Inertia.js를 사용하였습니다.<br>
> 일반적인 SPA 형태 구현 시 API를 통해 데이터를 조회하지만<br>
> [Inertia.js](https://inertiajs.com){:target="_blank"}를 사용하면 서버단 로직을 API를 이용하지 않고<br>
> 기존 MPA 형태 그대로 사용하면서 SPA 형태를 구현할 수 있는 패키지입니다.<br>
> 서버단은 Laravel, Rails를 지원하며 클라이언트단은  Vue.js, React, Svelte를 지원합니다.<br>

```SATA
이후 내용은 Chime SDK를 활용한 부분만 작성되었으며
이해를 돕기 위해 실제 구현내용을 수정해서 작성하였습니다.

개발 당시 amazon-chime-sdk-js 최신버전은 1.17.0으로 해당 버전 기준으로 작성되었으며
현재 기준 2.5.0 버전까지 릴리즈 되어 작성한 내용과 다른 부분이 있을 수 있습니다.
```

**Meeting 세션 생성**  
Chime SDK를 이용하여 제공하는 기능을 사용하려면 제일 먼저 Meeting 세션을 생성해야 합니다.<br>
서버단에서 Chime API를 통해 생성한 Meeting 세션 정보와 Attendee 세션 정보를 이용하여 생성합니다.<br>
> Meeting 세션은 특정 일시에 사용하도록 예약 생성은 불가능하며 실제로 사용될 시점에 세션을 생성해야 합니다.<br>
> 세션을 미리 생성하게 되면 일정 시간 사용하지 않을 경우 자동 종료 조건에 의해 사용하실 수 없습니다.<br>

Meeting 세션 Sample
```json
{"Meeting": {"MeetingId": "a8da5bf4-c40e-4c32-92d6-3aadfacb28ec", "MediaRegion": "ap-northeast-1", "MediaPlacement": {"AudioHostUrl": "adc5998a46194f9743b725854174545a.k.m3.an1.app.chime.aws:3478", "SignalingUrl": "wss://signal.m3.an1.app.chime.aws/control/a8da5bf4-c40e-4c32-92d6-3aadfacb28ec", "ScreenDataUrl": "wss://bitpw.m3.an1.app.chime.aws:443/v2/screen/a8da5bf4-c40e-4c32-92d6-3aadfacb28ec", "TurnControlUrl": "https://ccp.cp.ue1.app.chime.aws/v2/turn_sessions", "AudioFallbackUrl": "wss://haxrp.m3.an1.app.chime.aws:443/calls/a8da5bf4-c40e-4c32-92d6-3aadfacb28ec", "ScreenSharingUrl": "wss://bitpw.m3.an1.app.chime.aws:443/v2/screen/a8da5bf4-c40e-4c32-92d6-3aadfacb28ec", "ScreenViewingUrl": "wss://bitpw.m3.an1.app.chime.aws:443/ws/connect?passcode=null&viewer_uuid=null&X-BitHub-Call-Id=a8da5bf4-c40e-4c32-92d6-3aadfacb28ec"}}, "@metadata": {"headers": {"date": "Wed, 10 Mar 2021 04:03:29 GMT", "content-type": "application/json", "content-length": "844", "x-amzn-requestid": "3ea0f8fd-9ae3-4009-9dc6-91143cd3f562"}, "statusCode": 201, "effectiveUri": "https://service.chime.aws.amazon.com/meetings", "transferStats": {"http": [[]]}}}
```

Attendee 세션 Sample
```json
{"Attendee": {"JoinToken": "MzFlNjAyZDItZTI2Ni1jMTU0LWUwYmItZTJiYWExZDk4MDkzOjdlMjliODk3LThkMDYtNGRlMC04OTcyLWY1MjFlYjhkYzc3Zg", "AttendeeId": "31e602d2-e266-c154-e0bb-e2baa1d98093", "ExternalUserId": "06ecd0a2-45f8-4e64-ad30-32c0c67f399e"}, "@metadata": {"headers": {"date": "Wed, 10 Mar 2021 04:04:57 GMT", "content-type": "application/json", "content-length": "247", "x-amzn-requestid": "62a9d5ad-2cbb-4a31-8902-6b3591ac8b24"}, "statusCode": 201, "effectiveUri": "https://service.chime.aws.amazon.com/meetings/a8da5bf4-c40e-4c32-92d6-3aadfacb28ec/attendees", "transferStats": {"http": [[]]}}}
```

Logger는 에러 및 디버깅 정보를 출력하는 용도로 사용되며 ConsoleLogger는 Console에 출력되고 MeetingSessionPOSTLogger는 설정한 주기마다 지정한 URL로 메세지를 전달합니다.<br>
저희는 개발 시에는 콘솔에 출력되도록 설정하였습니다.  
```javascript
import {
    ConsoleLogger,
    DefaultDeviceController,
    DefaultMeetingSession,
    LogLevel,
    MeetingSessionConfiguration,
    MeetingSessionPOSTLogger,
} from 'amazon-chime-sdk-js';

createMeetingSession(meetingInfo, attendeeInfo) {
        const configuration = new MeetingSessionConfiguration(meetingInfo, attendeeInfo);
        configuration.enableWebAudio = false;
        configuration.enableUnifiedPlanForChromiumBasedBrowsers = true;

        const logger = (this.env === 'local')
            ? new ConsoleLogger('InterviewLogger', LogLevel.WARN)
            : new MeetingSessionPOSTLogger('InterviewLogger', configuration, 85, 2000, window.route('logs'), LogLevel.WARN);

        const deviceController = new DefaultDeviceController(logger);
        this.meetingSession = new DefaultMeetingSession(configuration, logger, deviceController);
}
```

**지원 브라우저 체크**  
서비스 구현 시 Chime SDK에서 지원하지 않는 브라우저로 접속한 경우 예외처리가 필요하였습니다.<br>
Chime SDK에서 아래와 같이 간단히 체크할 수 있도록 제공하고 있습니다.<br>
```javascript
import { DefaultBrowserBehavior } from 'amazon-chime-sdk-js';

 const defaultBrowserBehavior = new DefaultBrowserBehavior({ enableUnifiedPlanForChromiumBasedBrowsers: true });

if (!defaultBrowserBehavior.isSupported()) {
    // 지원하지 않는 브라우저 처리
}
```

**기기 허용여부 체크**  
브라우저상에서 마이크, 카메라를 사용하기 위해서는 기기 사용 권한을 허용으로 설정해야 합니다.<br>
Chime SDK 내부에서 기본적으로 미디어 기기를 사용하기 위해 navigator.mediaDevices.getUserMedia를 호출하면서 브라우저 좌측 상단에 권한 설정 팝업이 발생합니다.<br>
기기 권한이 허용으로 설정되지 않은 경우 특정 문구를 보여주는 등 로직이 필요한 경우 Chime SDK에서 제공하는 setDeviceLabelTrigger를 이용하여 구현할 수 있습니다.<br>
```javascript
meetingSession.audioVideo.setDeviceLabelTrigger(
    async () => {
        // 권한 허용되어 있지 않은 경우 처리할 로직 구현
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: true });
        // 권한 허용 후 처리할 로직 구현
        return stream;
    }
);
```

**기기 리스트 조회 및 선택**  
영상면접을 진행하기 위해서는 접속한 사용자가 사용할 기기(마이크, 카메라, 스피커)를 선택해야 합니다.<br>
여러개의 기기가 연결되어 있을 수 있기 때문에 사용 가능한 기기 리스트를 보여주고 사용자가 선택하게 해야 합니다.<br>
아래와 같이 생성된 Meeting 세션을 통해 사용자에게 연결된 기기 리스트를 조회할 수 있습니다.<br>
```javascript
const audioInputDevices = await meetingSession.audioVideo.listAudioInputDevices();    // 마이크
const videoInputDevices = await meetingSession.audioVideo.listVideoInputDevices();    // 카메라
const audioOutputDevices = await meetingSession.audioVideo.listAudioOutputDevices();  // 스피커
```

리스트 조회 시 deviceId, label이 포함된 배열이 전달되며 해당 값을 이용해 Select Box 등으로 사용자가 선택하도록 구현합니다.<br>
deviceId 값은 사용할 기기 선택 시  사용됩니다.<br>

서비스 이용 도중 블루투스 기기 연결/해제 등 사용 가능한 기기 리스트가 변경되는 상황을 위해 Observer를 등록할 수 있습니다.<br>
아래와 같이 연결된 기기에 변경사항이 발생한 경우 기기 리스트를 갱신하는 등의 로직을 추가할 수 있습니다.<br>
```javascript
meetingSession.audioVideo.addDeviceChangeObserver({
    audioInputsChanged: list => {
        // 마이크 리스트 갱신
    },
    videoInputsChanged: list => {
        // 카메라 리스트 갱신
    },
    audioOutputChanged: list => {
        // 스피커 리스트 갱신
    },
});
```

사용자가 사용할 기기를 선택하면 Chime SDK에 사용할 기기를 설정해야 합니다.<br>
```javascript
// 마이크
await meetingSession.audioVideo.chooseAudioInputDevice(deviceId);

// 카메라
await meetingSession.audioVideo.chooseVideoInputDevice(deviceId);

// 스피커
await meetingSession.audioVideo.chooseAudioOutputDevice(deviceId);
```

카메라 화질은 별도로 설정하지 않는 경우 기본 540p 화질로 설정되며 아래와 같이 별도로 설정하실 수도 있습니다.<br>
설정 시 width, heigh, frameRate, maxBandwidthKbps 값을 전달합니다.<br>
```javascript
meetingSession.audioVideo.chooseVideoInputQuality(1280, 720, 30, 2500);
```

**기기 테스트 및 확인**  
사용자가 사용할 기기를 설정하면 해당 기기가 정상적으로 동작하는지 테스트하는 기능을 구현합니다.<br>

마이크 동작 여부를 테스트 하기 위해서는 AnalyserNode를 활용하며 Chime SDK에서 AnalyserNode를 생성하는 메소드를 제공합니다.<br>
해당 메소드와 requestAnimationFrame을 이용하여 아래와 같이 마이크 동작 여부를 실시간으로 보여줄 수 있습니다.<br>
```javascript
startAudioPreview() {
    const analyserNode = meetingSession.audioVideo.createAnalyserNodeForAudioInput();

    if (!analyserNode?.getByteTimeDomainData) {
        return;
    }

    const data = new Uint8Array(analyserNode.fftSize);
    let frameIndex = 0;
    this.analyserNodeCallback = () => {
        if (frameIndex === 0) {
            analyserNode.getByteTimeDomainData(data);
            const lowest = 0.01;
            let max = lowest;
            data.forEach((f) => {
                max = Math.max(max, (f - 128) / 128);
            });
            const normalized = (Math.log(lowest) - Math.log(max)) / Math.log(lowest);
            const percent = Math.min(Math.max(normalized * 100, 0), 100);
       }
       frameIndex = (frameIndex + 1) % 2;
       requestAnimationFrame(this.analyserNodeCallback);
    };
		
    requestAnimationFrame(this.analyserNodeCallback);
}
```

카메라 동작 여부를 테스트하기 위해서는 Chime SDK에서 제공하는 startVideoPreviewForVideoInput 메소드를 활용합니다.<br>
해당 메소드를 사용하기 위해서는 video 엘리먼트가 필요하며 해당 엘리먼트에 사용자의 카메라 영상이 보여지게 됩니다.<br>
```javascript
const videoEl = document.getElementById('video_preview');
meetingSession.audioVideo.startVideoPreviewForVideoInput(videoEl);
```

스피커 동작 여부를 테스트하기 위해서는 AudioContext를 이용하여 일정 시간동안 특정 소리를 발생시켜 audio 엘리먼트를 통해 송출하고 사용자가 듣고 확인할 수 있도록 구현합니다.<br>
AudioContext와 ChimeSDK에서 제공하는 메소드를 이용하여 아래와 같이 구현할 수 있습니다.<br>
```javascript
import { DefaultAudioMixController, TimeoutScheduler } from 'amazon-chime-sdk-js';

export default class TestSound {
    constructor(endCallback, sinkId, frequency = 440, durationSec = 1, rampSec = 0.1, maxGainValue = 0.1) {
        if (!sinkId || sinkId === 'None') {
            return;
        }

        const audioContext = new (window.AudioContext || window.webkitAudioContext)();
        const gainNode = audioContext.createGain();
        gainNode.gain.value = 0;
        const oscillatorNode = audioContext.createOscillator();
        oscillatorNode.frequency.value = frequency;
        oscillatorNode.connect(gainNode);
        const destinationStream = audioContext.createMediaStreamDestination();
        gainNode.connect(destinationStream);
        const { currentTime } = audioContext;
        const startTime = currentTime + 0.1;
        gainNode.gain.linearRampToValueAtTime(0, startTime);
        gainNode.gain.linearRampToValueAtTime(maxGainValue, startTime + rampSec);
        gainNode.gain.linearRampToValueAtTime(maxGainValue, startTime + rampSec + durationSec);
        gainNode.gain.linearRampToValueAtTime(0, startTime + rampSec * 2 + durationSec);
        oscillatorNode.start();
        const audioMixController = new DefaultAudioMixController();
        audioMixController.bindAudioDevice({ deviceId: sinkId });
        audioMixController.bindAudioElement(new Audio());
        audioMixController.bindAudioStream(destinationStream.stream);
        new TimeoutScheduler((rampSec * 2 + durationSec + 1) * 1000).start(() => {
            audioContext.close();
            endCallback();
        });
    }
}
```

저희가 구현할 기기 선택 페이지에서는 선택한 기기가 정상적으로 동작하는지 확인하여 Check 표기를 해주는 기능이 필요하였습니다.<br>
마이크의 경우 마이크 동작 여부를 확인하는 AnalyserNode를 통해 발생되는 데이터가 있는 경우 정상 동작으로 처리하였으며<br>
카메라와 스피커는 `navigator.mediaDevices.getUserMedia`를 이용해 전달되는 Stream이 존재하는 경우 정상 동작으로 처리하였습니다.<br>
```javascript
try {
    // 카메라 체크 시
    stream = await navigator.mediaDevices.getUserMedia({ video: true });

    // 스피커 체크 시
    stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    if (stream) {
        // 정상 동작
    }
} catch (error) {}
```

Chime SDK 1.17.0 버전부터 위와 같은 상황을 체크하는 Meeting readiness checker가 추가되었으며 내부적으로는 저희가 구현한 소스와 유사하게 구현되어 있습니다.<br>
자세한 내용은 [공식 문서](https://github.com/aws/amazon-chime-sdk-js){:target="_blank"}에서 확인하실 수 있습니다.<br>

**세션 시작**  
사용할 기기 설정 등 준비가 완료되면 참여자간 면접을 진행하기 위해 세션을 시작합니다.<br>
세션 시작 시 오디오가 사용될 엘리먼트를 설정해야 합니다.<br>
추가로 세션 시작 후 참여자 본인의 비디오가 바로 공유되도록 구현하기 위해서는 비디오를 시작하도록 설정해야 합니다.<br>
```javascript
// 오디오 엘리먼트 바인딩
await meetingSession.audioVideo.bindAudioElement(audioEl);

// 세션 시작
meetingSession.audioVideo.start();

// 참여자 본인 비디오 시작
meetingSession.audioVideo.startLocalVideoTile();
```

**Observer**  
세션 사용과 관련하여 이벤트 발생 시 관련 로직을 추가할 수 있도록 Observer를 등록할 수 있습니다.<br>
예를 들어 세션이 시작된 경우 특정 로직 실행이 필요하면 아래와 같이 추가하실 수 있습니다.<br>
```javascript
meetingSession.audioVideo.addObserver({
    audioVideoDidStart: () => {
        console.log('Session started');
    },
});
```
아래와 같이 다양한 Observer를 등록할 수 있으며 자세한 내용은 [공식 문서](https://github.com/aws/amazon-chime-sdk-js){:target="_blank"}에서 확인하실 수 있습니다.<br>
* estimatedDownlinkBandwidthLessThanRequired
* videoNotReceivingEnoughData
* metricsDidReceive
* audioVideoDidStartConnecting
* audioVideoDidStart
* audioVideoDidStop
* videoTileDidUpdate
* videoTileWasRemoved
* videoAvailabilityDidChange
* connectionDidBecomePoor
* connectionDidSuggestStopVideo
* videoSendDidBecomeUnavailable

> Component 기반 프레임워크(React, Vue Angular 등)를 사용한다면 Component가 마운트 될 때 Observer를 등록해야 하며
> 마운트 해제될 때 `meetingSession.audioVideo.removeObserver(observer)`를 통해 Observer를 제거해야 합니다.<br>

**마이크, 카메라 ON/OFF**  
다른 참여자에게 참여자 본인의 소리를 음소거 처리하거나 영상을 보여주지 않도록 설정할 수 있습니다.<br>
> 단, 반대로 참여자 본인에게만 다른 특정 참여자의 소리를 음소거 처리하는 기능은 현재 제공하고 있지 않습니다.<br>

```javascript
// 마이크 ON
meetingSession.audioVideo.realtimeUnmuteLocalAudio();

// 마이크 OFF
meetingSession.audioVideo.realtimeMuteLocalAudio();

// 카메라 ON
meetingSession.audioVideo.startLocalVideoTile();

// 카메라 OFF
meetingSession.audioVideo.stopLocalVideoTile();
```

**참여자 입장/퇴장**  
실시간으로 다른 참여자가 입장하거나 퇴장하는 상황을 전달 받아 참여자 리스트 관리 및 비디오 레이아웃 구현 등에 사용합니다.<br>
또한 실시간으로 해당 참여자의 오디오 관련 정보(볼륨, 음소거 여부 등)를 전달 받아 사용할 수 있습니다.<br>
볼륨 정보를 활용하여 현재 참여자가 대화를 하고 있는지 표기하였으며 `subscribeToActiveSpeakerDetector` 메소드를 사용하여 구현하실 수도 있습니다.<br>
참여자 리스트 관리는 Vuex Store에 저장하여 활용하였습니다.<br>
```javascript
meetingSession.audioVideo.realtimeSubscribeToAttendeeIdPresence(
    (presentAttendeeId, present) => {
        if (!present) {
            // 참여자가 퇴장한 경우 처리
            // presentAttendeeId로 기존 참여자 리스트에서 제외 등
            return;
        }

        meetingSession.audioVideo.realtimeSubscribeToVolumeIndicator(
            presentAttendeeId,
            async (attendeeId, volume, muted, signalStrength) => {
                const baseAttendeeId = new DefaultModality(attendeeId).base();
                if (baseAttendeeId !== attendeeId) {
                    // 화면 및 콘텐츠 공유 시에도 하나의 참여자로 구분되어 해당 케이스인 경우 제외
                    return;
                }

                // 참여자 리스트에 attendeeId에 해당하는 참여자가 없는 경우 추가

                // 해당 참여자 정보에 volume, muted, signalStrength 정보 업데이트
            },
        );
    },
);
```

**비디오 레이아웃 구성**  
영상면접 서비스에서는 참여자를 크게 면접관과 응시자로 구분합니다.<br>

참여자 형태별로 최대 4명까지 접속 가능합니다. (면접관 4 : 응시자 4)<br>

참여자 본인과 동일한 형태의 참여자를 하단에 작은 사이즈로 노출하고 다른 형태의 참여자는 상단에 보다 큰 사이즈로 노출합니다.<br>
참여자 수에 따라 사이즈는 자동으로 조절되도록 CSS로 구현하였습니다.<br>

videoTileDidUpdate, videoTileWasRemoved Observer를 이용해 참여자의 비디오가 연결되거나 해제될 경우 비디오 레이아웃이 갱신되도록 구현하였습니다.<br>
비디오 연결 시 동작하는 videoTileDidUpdate Observer에서는 boundAttendeeId로 참여자를 구분하고 tileId로 특정 위치의 video 엘리먼트를 바인딩하여 영상이 노출되도록 처리합니다.<br>
비디오 연결 해제 시 동작하는 videoTileWasRemoved Observer에서는 tileId로 연결 해제 시 동작할 로직을 구현합니다.<br>
```javascript
meetingSession.audioVideo.addObserver({
    async videoTileDidUpdate: tileState => {
        if (!tileState.boundAttendeeId || tileState.isContent || !tileState.tileId) {
            return;
        }

        const attendeeId = tileState.boundAttendeeId;
				
        // 참여자 리스트에 해당 attendeeId가 없는 경우 추가
				
        // 참여자 정보에 tileId 업데이트
        
        const videlEl = '';  // 해당 참여자의 비디오를 연결할 video 엘리먼트
        meetingSession.audioVideo.bindVideoElement(tileState.tileId, videoEl);
    },

    videoTileWasRemoved: tileId => {
        // 참여자 정보에 tileId 제거
    }
});
```

**참여자간 간단한 메세지 전송**  
Chime SDK에서는 `realtimeSendDataMessage`와 `realtimeSubscribeToReceiveDataMessage` 메소드를 이용해 참여자간 최대 2kb의 실시간 메세지를 서로 전송할 수 있습니다.<br>
간단한 채팅을 구현할 수 있으며 저희는 참여자 상태 변경 알림 및 종료시간 연장 알림 등 알림 서비스에서 활용하였습니다.<br>
```javascript
// 메세지 전송
meetingSession.audioVideo.realtimeSendDataMessage(topic, message);

// 메세지 수신
meetingSession.audioVideo.realtimeSubscribeToReceiveDataMessage(topic, callback);
```

Chime SDK 2.1.0 버전부터 Messaging session 기능이 추가되어 사용할 수 있으며 해당 기능 사용 시 별도로 비용이 부과됩니다.<br>


**세션 종료**  
영상면접이 종료되거나 참여자가 면접방을 나갈 경우 세션을 종료해야 합니다.<br>

Meeting 세션 전체를 종료하려면 해당 세션을 삭제하는 Chime API를 호출해야 합니다.<br>
Meeting 세션 종료 시 참여자들은 자동으로 세션이 종료되며 종료 시 페이지를 이동하는 등의 처리 로직이 필요한 경우 `audioVideoDidStop` Observer를 추가하여 구현합니다.<br>
`audioVideoDidStop` Observer에는 `sessionStatus` 상태 코드가 전달되는데 해당 코드를 이용하여 다양한 종료 상황에서의 처리 로직을 추가할 수 있습니다.<br>
저희는 다른 기기에서 동일한 참여자 세션으로 다중 접속한 경우에 대한 처리를 추가로 구현하였습니다.<br>
```javascript
import { MeetingSessionStatusCode } from 'amazon-chime-sdk-js';

meetingSession.audioVideo.addObserver({
    audioVideoDidStop: sessionStatus => {
        const statusCode = sessionStatus.statusCode();

        if (statusCode === MeetingSessionStatusCode.AudioCallEnded) {
            // Meeting 세션이 종료된 경우
        } else if (statusCode === MeetingSessionStatusCode.AudioJoinedFromAnotherDevice) {
            // 다른 기기에서 동일한 참여자 세션으로 다중 접속한 경우
        }
    },
});
```

개별 참여자만 면접방에서 나가는 경우는 Chime SDK에서 제공하는 `stop` 메소드를 사용합니다.<br>
```javascript
meetingSession.audioVideo.stop();
```

<br>
최종적으로 구현된 영상면접 진행 페이지입니다.<br>
면접관으로 접속한 화면이며 면접관 1 : 응시자 2 레이아웃입니다.<br>
[<img src="{{site.url}}/img/video-interview/main.jpg" />]({{site.url}}/img/video-interview/main.jpg)

--------------------------------------------------------------------------------------

## 이슈 해결
대부분의 규모가 큰 프로젝트가 그렇듯이 저희 프로젝트도 개발 완료 후 QA를 진행하면서 많은 오류들이 발생하였습니다.<br>
간단한 오류들도 많았지만 그 중에서도 기억에 남는 이슈를 정리해 보았습니다.<br>

**해상도 이슈**  
저희는 기기 선택화면에서 비디오 화질도 사용자가 선택할 수 있도록 구현하였습니다.<br>
360p, 540p, 720p 3가지 화질을 선택할 수 있도록 하였는데요.<br>

사용자 환경에 따라 지원 가능한 최대 해상도가 달랐으며 최대 해상도보다 높은 화질을 선택한 경우 비디오 화면이 노출되는 사이즈가 의도치 않은 사이즈로 적용되는 현상이 발생했습니다.<br>
내부적으로 논의 후 지원 가능한 최대 해상도보다 높은 화질은 선택하지 못하도록 처리하기로 하였습니다.<br>
그러기 위해서는 현재 사용자 환경에서 지원 가능한 최대 해상도를 가져오는 방법이 필요했습니다.<br>

여러 방법을 검색해 본 후 저희는 `MediaStreamTrack.getCapabilities()`를 활용하기로 하였습니다.<br>
해당 기능을 이용하면 현재 사용자 환경에서 지원하는 최대 width 값과 최대 height 값을 가져올 수 있었고 해당 값 이상의 화질은 선택하지 못하도록 적용하였습니다.<br>

아래와 같이 구현하였으며 아쉽게도 Firefox 브라우저에서는 지원하지 않습니다.<br>
```javascript
async checkVideoResolutionByTrackCapabilities(width, height) {
        let stream;
        try {
            stream = await navigator.mediaDevices.getUserMedia({ video: true });
            if (!stream) {
                return true;
            }

            const [track] = stream.getTracks();

            // Firefox 등 getCapabilities를 지원하지 않는 경우
            if (typeof track.getCapabilities === 'undefined') {
                return true;
            }

            const capabilities = track.getCapabilities();
            return capabilities.width.max >= width && capabilities.height.max >= height;
        } catch (error) {
            return true;
        }
    }
```

**Firefox 브라우저에서 스피커 리스트 조회 불가 이슈**  
기기를 선택하는 페이지에서 스피커 기기 리스트를 조회하여 사용자가 선택할 수 있도록 구현하였는데 Firefox 브라우저에서만 스피커 리스트를 가져오지 못하는 현상이 발생하였습니다.<br>
리스트 조회만 불가능하였고 실제 면접 진행 시 소리는 정상적으로 출력되었습니다.<br>

Chime SDK Github Repository 내 [Issues](https://github.com/aws/amazon-chime-sdk-js/issues/238){:target="_blank"}에서 동일한 내용의 문의 및 답변을 확인할 수 있었습니다.<br>

원인은 Firefox 브라우저에서 오디오 기기 ID를 설정하는 setSinkId 기능을 제공하지 않아 발생한 문제였습니다.<br>
해결 방법은 Firefox 설정에서 media.setsinkid.enabled 값을 true로 설정하는 방법뿐이었고 소리는 정상적으로 출력되어 면접 진행 시 이슈는 없었기 때문에 해당 상황에서는 스피커 리스트를 기본값만 노출되도록 처리하였습니다.<br>

위 해상도 이슈와 스피커 리스트 조회 불가 이슈 등 Firefox에서만 여러 이슈가 발생하여 Firefox 브라우저로 접근한 경우는 다른 브라우저 이용을 권장하는 안내 문구를 추가하였습니다.<br>

**그래픽카드 이슈**  
QA 진행 중 특정 PC에서만 영상 노출 화면에 Hardware Acceleration을 지원하지 않는다는 문구가 노출되는 현상이 있었습니다.<br>

[Chime SDK Faq](https://aws.github.io/amazon-chime-sdk-js/modules/faqs.html)에서는 콘텐츠 공유 사용 시 사용하는 `HTMLMediaElement.captureStream` API가 크롬 88버전 이상에서는 이슈가 있으며 크롬 설정에서 hardware acceleration 설정을 off로 변경해야 한다는 내용이 있었습니다.<br>

저희는 해당 상황은 아니었기 때문에 해결 방법을 더 찾아보았고 해당 PC가 Windows 7 OS를 사용하고 있는데 그래픽카드가 Windows 7용 드라이버가 존재하지 않아 3dp 프로그램에서 제공하는 드라이버를 사용하고 있는 것을 발견하였습니다.<br>
해당 그래픽카드 드라이버를 기본 드라이버로 변경 후 테스트 시 정상 동작하는걸 확인할 수 있었습니다.<br>

**메모리 효율화**  
QA 진행 시 사양이 좋지 않은 PC에서 참여자수가 많을 경우 버벅거리는 현상이 발생하였습니다.<br>

해당 PC에서 실행하는 프로그램이 많기도 하였고 크롬 브라우저의 경우 기본적으로 점유하는 메모리양이 많은 점도 있었지만 애플리케이션단에서 메모리 사용을 줄이는 방안을 고민하였으며 아래와 같은 내용을 적용하였습니다.<br>

1. GC에 의해 메모리가 관리되고 있지만 메모리를 많이 차지하는 사용하지 않는 변수들은 명시적으로 제거되도록 변경하였습니다.<br>
2. SAP 형태 구현으로 인해 아래와 같이 이전 페이지에서 주기적으로 실행되는 동작들에 대해 다음 페이지 이동 시 실행되지 않도록 처리하였습니다.<br>
* 남은 시간 표기 등을 위해 `setInterval` 사용
* 마이크 동작 여부 표기를 위한 `requestAnimationFrame` 사용 등

3. 공식 문서에서 제공하는 내용과 같이 Chime SDK에 등록된 Observer도 Component가 unmounted 되었을 경우 제거되도록 추가하였습니다.<br>
4. 기기 선택화면에서 제공한 비디오 미리보기 기능도 다음 페이지 이동 시 `meetingSession.audioVideo.stopVideoPreviewForVideoInput`를 통해 중지되도록 추가하였습니다.<br>

최종적으로 기존 메모리 사용량 대비 10% 정도 감소 효과가 있었지만 기본 환경 이슈로 버벅거림 현상이 완전히 해소되지는 않았습니다.<br>

추후 여유가 있을 때 좀 더 자세히 확인해 볼 예정입니다.<br>

--------------------------------------------------------------------------------------

## 마치며
Chime SDK를 활용한 레퍼런스가 많이 부족한 상황에서 개발을 진행하며 어려운 부분도 많았지만 영상 서비스와 관련된 경험을 할 수 있어서 개인적으로 매우 의미있는 프로젝트였던 것 같습니다.<br>

실시간 영상 서비스 구현과 관련해서 Chime SDK에서 기본적인 기능들을 쉽게 사용할 수 있도록 제공하고 있고 업데이트를 통해 새로운 기능과 유지보수가 계속 진행되고 있어 관련 서비스 진행 시 사용을 고려해봐도 좋을 것 같습니다.<br>

지속적인 추가 기능 구현 등 개선을 통해 코로나19 상황에서 많은 구인/구직자 분들에게 도움이 되는 서비스가 되었으면 좋겠습니다.<br>

이번 프로젝트 진행을 위해 기획, 디자인, UI개발, 개발, SE, DA, QA, 등 많은 분들이 참여해 주셨는데 정말 고생 많으셨습니다.<br>
AWS 담당 매니저님도 많은 도움 주셔서 감사합니다.<br>

--------------------------------------------------------------------------------------
## 관련문서
* [AWS SDK](https://aws.amazon.com/ko/getting-started/tools-sdks/){:target="_blank"}
* [AWS SDK for PHP](https://github.com/aws/aws-sdk-php/tree/2.8){:target="_blank"}
* [Amazon Chime SDK Documentation](https://docs.aws.amazon.com/chime/latest/dg/meetings-sdk.html)
* [Amazon Chime API Reference](https://docs.aws.amazon.com/chime/latest/APIReference/Welcome.html){:target="_blank"}
* [Amazon Chime SDK PHP API Reference](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-chime-2018-05-01.html#createmeeting){:target="_blank"}
* [Amazon Chime SDK for JavaScript](https://github.com/aws/amazon-chime-sdk-js){:target="_blank"}
* [Amazon Chime SDK Javascript API Reference](https://aws.github.io/amazon-chime-sdk-js/){:target="_blank"}
* [Amazon Chime SDK for Android](https://github.com/aws/amazon-chime-sdk-android){:target="_blank"}
* [Amazone Chime SDK for iOS](https://github.com/aws/amazon-chime-sdk-ios){:target="_blank"}
