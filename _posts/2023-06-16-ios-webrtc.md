---
title: iOS에서 음성대화 서비스 구현하기 - WebRTC 기술의 활용
author: 김은미
tags:
- iOS
- WebRTC
- Janus
---

안녕하세요. 사람인 IT연구소 서비스인프라개발팀 김은미입니다.    

사람인은 커리어 전반을 케어 할 수 있도록 여러 서비스를 준비하고 있습니다. 

2023년 런칭한 ‘멘토링매치 서비스’는 회사, 직무 등 커리어에 관해 궁금한 점이 있는 멘티와 멘토가 사람인 내 일대일 음성 대화를 통해 경험과 정보를 나누는 커리어 상담 서비스입니다. 

이 글에서는 iOS에서 음성 대화 서비스를 제공하기 위해 WebRTC 기술을 활용하는 과정을 소개하고자 합니다. 
<br><br>
# WebRTC
<br>
<img src="/img/webrtc-janus/ios_webrtc1.jpg" width="600px" />

> 다음은 WebRTC 동작을 이해하는 간략한 다이어그램입니다.


WebRTC(Web Real-Time Communication)는 브라우저상에서 영상이나 오디오와 같은 미디어를 스트림하고 데이터 교환까지 할 수 있게 하는 기술입니다. 

서버 없이 peer 간 통신이 가능하다면 더 private하고 서버 유지 비용도 없을 것입니다. WebRTC는 peer를 직접적으로 연결해 미디어를 전송할 수 있도록 합니다. 
<br>

#### iOS에서 WebRTC 통신은 크게 아래와 같은 작업으로 구성됩니다. 

1.  상대에게 보낼 나의 미디어 데이터 수집 
2.  signaling 과정, connection 을 맺기 위한 정보 수집 
3.  RTCPeerConnection 맺기 
4.  맺은 connection 을 통해 미디어/데이터 교환 
<br>

먼저 상대에게 보낼 나의 미디어 데이터를 수집합니다. iOS에서는 영상의 경우 AVCaptureSession을, 음성의 경우 AVAudioSession을 이용하여 각 미디어 데이터를 읽을 수 있습니다. 

peer 간 connection을 맺기 위해서는 IP, Port, Codec 등 정보 교환이 필요합니다. 

WebRTC 자체에서 지원하는 기능은 아니라서 별도의 시그널링 서버를 통해 peer 간 세션 정보를 교환합니다. 시그널링 서버 기술의 표준은 존재하지 않아 WebSocket, STOMP, gRPC 등 다양한 방식으로 구현되고 있습니다.

멘토링매치 서비스에서는 WebSocket을 사용하였습니다. 

시그널링 과정에서 서로의 네트워크 정보를 교환할 때 사용자들은 서로 다른 환경에서 연결되기 때문에 다양한 환경에서 동작할 수 있어야 합니다. 

그래서 NAT 및 방화벽 환경에 대한 고려가 필요합니다. **NAT**(NetWork Address Translation)란 IP 주소 절약 및 편의성 그리고 보안을 위한 네트워크 주소 변환을 의미하는 기술입니다. 

<br>
<img src="/img/webrtc-janus/ios_webrtc2.jpg" width="800px" />

> 다음은 NAT 환경을 고려하여 표현한 WebRTC 동작을 나타내는 다이어그램입니다.


방화벽을 통과해야 하는 경우, 단말에 공개 IP가 없어서 주소값을 할당해야 하는 경우, 라우터가 직접 연결을 허용하지 않는 경우와 같은 환경에서 발생하는 연결 문제를 해결하고 최적의 경로를 찾기 위해  **ICE**, STUN, TURN 을 사용합니다. 

※ 각 기술 및 프로토콜에 대한 자세한 설명은 여러 글에서 기술하고 있어 생략합니다.

peer 간 통신이 가능한 connection 경로를 찾았다면 멀티미디어 컨텐츠 연결을 설명하기 위해 해상도, 형식, 코덱, 암호화 등의 미디어 구성 정보 교환이 필요합니다. 

이때 시그널링은 **SDP**(Session Description Protocol)를 사용하여 이뤄집니다. 

SDP는 멀티미디어 세션을 설명하는 정보를 담고 있습니다. 일련의 텍스트 기반 필드로 구성되어 있으며, *필드 = 파라미터1 파라미터2 파라미터3 ... 파라미터n*  형식으로 표현합니다. 각 필드에 대한 정보는 [링크](http://www.juandebravo.com/webrtc-repo/sdp-offer) 참고하시면 좋을 것 같습니다.  

peer 간 SDP Offer-제안과 SDP Answer-응답을 교환하며 양측이 호환 가능한 미디어 조건에 동의하게 됩니다. 

이렇게 SDP 정보를 교환하고 처리하는 과정은 SDP 협상이라고 하며, 협상 과정이 완료되면 합의된 조건을 사용하여 RTCPeerConnection 을 만들 수 있습니다.

RTCPeerConnection을 통해 WebRTC는 일반적으로 영상/음성 미디어 데이터와 텍스트 데이터를 주고 받을 수 있습니다.
<br><br>
# Janus를 이용한 WebRTC 구현

[**Janus**](https://janus.conf.meetecho.com/)는 Meetecho에서 개발한 WebRTC 오픈 소스 프로젝트로, 위에서 설명한 WebRTC를 사용할 수 있게 구현한 서버입니다. 

Janus는 기본적인 WebRTC 프로토콜의 처리 기능을 포함하며, Plugin 형태로 여러 기능을 추가할 수 있도록 설계되어 있습니다. 

그 중 **audio bridge plugin**은 음성 그룹 채팅 기능을 제공하며, 서버에서 Mixing을 하는 MCU 방식으로 음성 그룹 채팅에 참여하는 Peer의 음성을 Mixing해서 전달합니다. 

오디오 스트림의 믹싱, 분리 및 조정, 오디오 캡처 및 재생, 믹서, 효과, 음량 조절 등의 기능을 제공합니다.

멘토링매치 서비스는 Janus를 사용하여 구축되었으며 음성 대화 서비스를 위해 audio bridge plugin 기능을 개발했습니다. 

※ Janus를 활용한 WebRTC 백엔드 개발에 대한 자세한 내용은 [링크](https://saramin.github.io/2023-04-17-janus-webrtc/)를 참고하세요.   

<u>Janus를 통해 시그널링과 미디어 전송이 이루어지는 과정</u>은 다음과 같은 단계로 구성됩니다.

<br>
<img src="/img/webrtc-janus/ios_webrtc3.jpg" width="800px" />

> 다음은 시그널링 단계를 보여주는 다이어그램입니다.

### 1. 세션 생성
각 peer는 시그널링 서버와 연결하는 단일 세션이 필요합니다. 먼저 세션을 생성합니다. 
### 2. 플러그인에 연결할 하나 이상의 핸들 생성
세션을 생성한 후, 해당 세션을 플러그인에 연결하여 플러그인 핸들을 생성합니다. 각 janus 세션에는 여러 개의 플러그인 핸들을 동시에 포함할 수 있습니다. 

플러그인에서 생성된 handleID는 client에서 PeerConnection을 관리하는 값으로 사용됩니다.
### 3. 플러그인과 상호작용

이제 플러그인에 메시지를 보낼 수 있으며, 플러그인의 기능을 활용하여 PeerConnection이 전송하거나 수신한 미디어를 조작할 수 있습니다. 

<u>음성 대화를 위한 audio bridge plugin과의 상호작용</u>은 다음과 같은 프로세스로 진행됩니다. 

- janus audio bridge plugin에 연결한 후, `create request`를 통해 room을 생성합니다. 
- `join request`를 통해 오디오 룸에 참여하고, joined 이벤트를 기다립니다. joined 이벤트에는 다른 참가자의 목록도 포함되어 있습니다.
- WebRTC 협상을 위해 PeerConnection의 SDP를 생성하고 `Configure request`를 통해 JSEP 응답으로 `Offer request`를 전달하여 미디어 관련 설정을 합니다. 플러그인이 설정을 완료할 때까지 기다립니다.
- 이제 대화할 수 있습니다.
- 대화 중에는 음소거/음소거 해제 요청을 플러그인에 전달할 수 있습니다.
- 대화 중에는 여러 플러그인의 이벤트를 실시간으로 전달받아 업데이트할 수 있습니다. 
    - joined: 사용자의 참여
    - leaving: 탈퇴
    - muted: 음소거, 음소거 해제 
    - talking: 사용자가 말하고 있는지 여부
- `leave request`를 전달하여 room에서 나갈 수 있습니다.  
- `destroy request`를 전달하여 room을 파괴할 수 있습니다.

※ audio bridge plugin을 요청할 때 사용할 수 있는 항목 및 구체적인 옵션은 [문서](https://janus.conf.meetecho.com/docs/audiobridge)에서 확인하세요. 
### 4.  WebRTC 협상 프로세스

WebRTC 파트에서 설명한 것처럼, peer 간의 연결을 맺기 위한 정보 교환 과정으로 WebRTC RTCPeerConnection을 사용하기 위해 SDP 협상 프로세스를 진행합니다.

<u>SDP 교환</u>은 다음과 같은 단계로 이루어집니다. 

- peer A → create Offer 
									 → `send Offer SDP` 
- peer B  → `recive Offer SDP`
										→ peerConnection set SDP 
										→ create Answer 
										→ `send Answer SDP peer`
- peer A  → `recieve Answer SDP` 
										→ peerConnection set SDP

Peer A는 자신의 SDP를 사용하여 Offer 제안을 생성하고 시그널링 서버를 통해 전송합니다. Peer B는 Offer을 받으면 자신의 정보를 포함하여 응답인 Answer를 생성하고 이를 다시 Peer A에게 전송합니다. 
### 5. iCE Candidate

미디어를 전송하고 수신하기 위해 각 Peer의 네트워크 정보가 필요합니다. ICE 프레임워크를 사용하여 네트워크 연결 정보를 찾으며, 이 과정은 후보 리스트를 교환하는 방식으로 이루어집니다. 

각 Peer에서 여러 후보가 제안되며, 수신된 후보 중 가장 적합한 후보를 선택하여 연결을 설정합니다.

이렇게 미디어 정보와 네트워크 정보 교환 과정이 완료되면 미디어 전송이 이루어집니다. 
<br><br>
# AVAudioSession을 이용한 오디오 기능 구현
<br>
<img src="/img/webrtc-janus/ios_webrtc4.png" width="600px" />

> 다음은 [apple developer 오디오 세션 가이드](https://developer.apple.com/library/archive/documentation/Audio/Conceptual/AudioSessionProgrammingGuide/Introduction/Introduction.html)에서 제공하고 있는 iOS 오디오 동작에 대한 이미지입니다.

AVAudioSession은 iOS 앱과 운영체제 즉 오디오 하드웨어 간의 중개자 역할을 합니다. AVAudioSession을 사용하여 입력(Inputs), 출력(Outputs), 앱(App), 그리고 여러 앱 간의 오디오 동작을 관리할 수 있습니다.

WebRTC 통신을 통해 미디어 정보를 전달받으면 앱에서는 AVAudioSession을 활용하여 미디어를 제공하고 조작하기 위해 시스템과 통신해야 합니다. 

### 오디오 세션 구성 및 활성화
```swift
do {
	let config = RTCAudioSessionConfiguration()
	config.category = AVAudioSession.Category.playAndRecord.rawValue
	config.categoryOptions = [.mixWithOthers, .allowBluetoothA2DP, .allowBluetooth]
	try rtcAudioSession.setConfiguration(config)
	try rtcAudioSession.setActive(true)
} catch let error {
	...
}
```

먼저 WebRTC 에서 사용할 오디오 세션을 구성하고 활성화합니다. 오디오 세션의 Category와 CategoryOptions을 설정하여 앱이 어떤 오디오 모드를 사용하는지 알려줍니다. 

[앱 유형별 지침](https://developer.apple.com/library/archive/documentation/Audio/Conceptual/AudioSessionProgrammingGuide/AudioGuidelinesByAppType/AudioGuidelinesByAppType.html) 에 따라 Category를 .playAndRecord로 설정했습니다. 

우리 앱에서만 오디오를 사용하는 것이 아니기 때문에, .mixWithOthers 옵션을 지정하여 보이스 채팅 기능을 사용할 때 사용자가 다른 앱과 함께 사용할 수 있도록 다른 오디오와 혼합 가능한 형태로 제공합니다.

사용자가 핸드폰으로만 대화할 수도 있지만, 에어팟이나 다른 블루투스 기기와의 연동을 지원하기 위해 .allowBluetoothA2DP, .allowBluetooth 옵션도 추가합니다.

이렇게 필요한 기능에 맞게 옵션을 구성하고 오디오 세션을 활성화하여 변경 내용을 반영할 수 있습니다. 

### Change Route 발생
예를 들어, 디바이스의 기본 마이크와 스피커로 대화하다가 이어폰을 연결하면 자동으로 이어폰으로 전환되는 것이 일반적입니다. 마찬가지로 이어폰을 사용하다가 빼면 음악 플레이어는 멈추고, 통화의 경우에는 기본 마이크와 스피커로 전환하여 대화할 수 있습니다.  

AVAudioSession에서 제공하는 `routeChangeNotification`을 사용하여 route가 변경되었다는 알림을 받을 수 있고, CurrentRoute 속성에서 현재 오디오의 Input, Output의 PortType을 확인할 수 있습니다. 이를 통해 기능에 맞춰 UI를 업데이트하고 미디어를 제어할 수 있습니다.

또한, 사용자가 수동으로 입력 및 출력 경로를 변경해야 하는 경우 `AVRoutePickerView`를 호출하여 사용자가 원하는 Route를 선택하는 기능을 제공할 수 있습니다. 

이처럼 사용자가 익숙한 오디오 사용의 흐름을 위해 Route 변경 이벤트를 처리해야 합니다. 

### Interrupt 대응
대화 중에는 다음과 같은 예외 상황이 발생할 수 있습니다. 이때 앱의 기능에 따라 적절한 처리가 필요합니다. 

- 백그라운드
- 다른 오디오 재생
- 통화
- 기기 잠김
- 무음 모드

대표적으로 대화 중에 전화가 오면 어떻게 될까요? 전화는 우선순위가 높으므로 앱에서 재생하는 오디오는 멈추게 됩니다. 

앱에서 오디오 재생이 예외 상황으로 인해 멈추게 되면 사용자가 알 수 있도록 UI에서 적절한 처리를 해주어야 합니다. 또한, 전화를 끊었을 때 멈췄던 오디오를 다시 사용할 수 있도록 처리가 필요할 수도 있습니다. 

Interrupt가 발생했을 때, AVAudioSession의 `interruptionNotification` 알림을 받을 수 있습니다. begin과 ended 신호를 읽어 interrupt에 대한 처리를 할 수 있습니다. 
<br><br>
# 마치며
지금까지 iOS에서 ‘멘토링매치’ 음성대화 서비스를 개발하는데 사용한 대표적인 기술에 대해 살펴보았습니다. 

WebRTC 기술은 오랜 역사와 지속적인 발전을 통해 현재의 실시간 소통이 가능하게 하는 데 큰 공헌을 한 기술입니다. 

다양한 기술 요소들이 함께 활용되며, 네트워크와 미디어 등 다양한 측면에서 완성도가 필요한 서비스라고 생각합니다. 지속적으로 서비스의 품질과 다양성을 향상시키도록 노력할 것입니다.

사람인은 구인 구직을 돕기 위한 서비스 제공을 위해 계속해서 연구하고 기술을 도입하고 있습니다. 기대와 관심 부탁드리며, 이 글을 마치겠습니다. 
<br><br>
# 참고자료

[https://WebRTC.org](https://webrtc.org/)  
[https://janus.conf.meetecho.com](https://janus.conf.meetecho.com/) 
[https://developer.apple.com/library/archive/documentation/Audio/Conceptual/AudioSessionProgrammingGuide/Introduction/Introduction.html](https://developer.apple.com/library/archive/documentation/Audio/Conceptual/AudioSessionProgrammingGuide/Introduction/Introduction.html) 
[https://developer.mozilla.org/ko/docs/Web/API/WebRTC_API](https://developer.mozilla.org/ko/docs/Web/API/WebRTC_API) 
[https://webrtcforthecurious.com/](https://webrtcforthecurious.com/)  
[http://www.juandebravo.com/webrtc-repo/sdp-offer](http://www.juandebravo.com/webrtc-repo/sdp-offer)

<br><br>
