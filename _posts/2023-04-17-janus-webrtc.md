---
title: Janus 를 활용한 WebRTC 기반의 음성대화 서비스 개발
author: 정다운
tags:
- ''
- WebRTC
- Janus
---

---

안녕하세요. 사람인 서비스인프라개발팀 정다운 입니다.   
 
앞서 3/28일, 사람인 멘토링 매치 서비스가 런칭하였습니다. 🎊🥳  

멘토링 매치 서비스는 회사, 직무 등 커리어 관련해서 궁금한 점이 있는 사람들이  
사람인 내 1:1 음성 대화를 통해 경험과 정보를 얻을 수 있는 서비스입니다.  
많은 관심 부탁드립니다. 🤗

저는 멘토링 매치 서비스 중 1:1음성대화 기능을 제공하는 백엔드 개발을 맡았으며  
WebRTC기술과 미디어서버 역할을 담당한 Janus를 음성대화에 어떻게 활용하였는지에 대해 다뤄보고자 합니다. 

---
# 목차
1. [WebRTC](#webrtc)  
2. [미디어 서버](#미디어-서버)  
3. [JANUS](#janus)  
	- [AudioBridge Plugin](#audiobridge-plugin)  
	- [Event handler](#event-handler)  
4. [음성대화 시스템 구성](#음성대화-시스템-구성)  
	- [MRB(Media Resource Broker)](#media-resource-broker)  
	- [음성대화 절차](#음성대화-절차)  
5. [개발중 발생한 이슈](#개발중-발생한-이슈)  
	- [유효기간이 없는 미디어서버 대화방](#유효기간이-없는-미디어서버-대화방)  
	- [미디어서버 재할당](#미디어서버-재할당)  
6. [마치며](#마치며)  
7. [참고자료](#참고자료)

<br>

---
# WebRTC

WebRTC(Web Real-Time Communicatino)는 단말간 P2P통신을 가능하게 해주는 웹 표준 기술이며 빠른 응답이 요구되는 화상/음성 통신에 사용됩니다.

WebRTC통신을 위해서는 세 가지 역할을 담당할 Application이 필요하며, 아래 그림으로 간단하게 설명 하겠습니다.

<img src="/img/webrtc-janus/webrtc-1.png" width="600px" title="WebRTC-Communication" />

1. **Signaling**

    연결할 Peer의 정보를 교환해주는 역할을 담당하며 SDP프로토콜을 사용합니다.  
		각 Peer는 미디어 통신의 세부 정보(미디어 타입 및 코덱 정보 등), 통신할 수 있는 경로(ip/port) 등 여러 데이터를 교환합니다.  
    그리고 모르는 사람끼리 연결해주면 안 되니, 서로를 식별할 수 있는 정보를 추가해서 어떤 Peer끼리 연결시해줄지를 결정해야 합니다.
    

1. **STUN (Session Traversal Utilities for NAT)**

    외부에서 접근할 수 있는 단말의 경로(ip/port)를 찾아주는 역할을 담당합니다.  
    상대방이 자신의 단말과 통신하기 위해서는 외부에서 어디로 접속해야 하는지 알아야겠죠?
    
2. **TRUN (Traversal Using Relays around NAT)**

    단말간 P2P 통신이 불가능할 때 중간에서 미디어 데이터를 Relay 해주는 역할을 담당합니다.  
    통신이 불가능한 환경은 드물지만, 방화벽이나 NAT 설정 등 예외적인 요인으로 통신할 수 없을 때 마지막 수단으로 사용됩니다.
    

요약하자면 Peer는 Signaling 서버를 통해 상대 Peer와 통신에 필요한 정보를 교환하며 STUN 서버를 이용하여 자신과 통신할 수 있는 경로(ip/port)를 찾아 함께 전달 합니다.  
이 절차가 정상적으로 수행된 후에 WebRTC통신이 이루어지며, 미디어 데이터를 주고받게 됩니다.

- 글에서는 이해를 돕기 위해 WebRTC통신을 위한 상세한 절차가 생략되었으며, 더 자세한 내용은 **[WebRTC For The Curious](https://webrtcforthecurious.com/)**에 잘 정리되어 있으니 참고하시면 좋을 것 같습니다.  
<br><br>

---
# 미디어 서버

위에서 설명해 드린 WebRTC통신은 Signaling을 거쳐 Peer 간 직접 통신(P2P)을 하게 됩니다.  
만약 1:1통신이 아니라 더 많은 Peer와 통신하게 되면 어떻게 될까요?

![MediaServer-Type](/img/webrtc-janus/webrtc-2.png)

P2P통신의 경우 각 단말은 (N-1) * 2 만큼 미디어 스트림을 열어 데이터를 송수신해야 합니다.  
10명이라면 18개의 미디어 스트림을 열고 데이터를 인코딩(encoding) / 디코딩(decoding)하게 되는데 저사양의 단말에서 성능을 보장하기 어렵게 됩니다.

그렇기 때문에 다자간 통신서비스를 구현한다면 클라이언트의 단말에서 가져야 할 부담을 분담하기 위해 대부분 중앙에 미디어서버를 두게 됩니다.

미디어서버는 각 Peer의 미디어 스트림을 중계해 주거나 믹싱(mixing)하는 역할을 하고 모든 Peer가 미디어서버를 거치기 때문에 Signaling 역할까지 담당합니다.

### SFU (Selective Forwarding Unit)

1개의 송신 스트림(Up Stream)만 보내며 미디어서버에서 모든 Peer에게 전송(중계)합니다.  
수신 스트림(Down Stream)은 N-1개를 사용합니다.

### MCU (Multipoint Control Unit)

1개의 송신 스트림(Up Stream)과 1개의 수신 스트림(Down Stream)을 사용합니다.  
미디어서버에서 스트림을 믹싱(mixing)하여 각 단말은 총 2개의 스트림(Up/Down)만 사용합니다.  

<br><br> 
사람인에서도 1:1 음성 대화 서비스 외에도 차후 영상 및 다자간 통신 서비스도 고려해야 하므로 미디어 서버가 필요했습니다.

#### 기능 요구사항
- 음성/화상통신 지원
- 안정적인 1:1 및 다자간 통신
- 인증관리
- 레코딩 및 화면 공유

기능 요구사항과 비용 및 라이선스, [구글 트렌드](https://trends.google.co.kr/trends/explore?q=mediasoup,janus,openvidu,kurento,jitsi&hl=ko) 및 [미디어 서버 테스트 자료](https://webrtchacks.com/sfu-load-testing/) 등을 취합하여 5개 정도의 오픈소스를 검토하였으며

오픈소스 성숙도 등을 고려하여 최종 [Janus](https://janus.conf.meetecho.com/)로 선정하였습니다.

- 화상채팅 앱 Azar로 유명한 하이퍼커넥트 역시 Janus를 이용하여 미디어서버를 구축했다는 [발표내용](https://youtu.be/75HTXiqOcdQ?t=1098)도 있었기에 조금 더 신뢰가 갔습니다.

<br>

---
# JANUS
<img src="/img/webrtc-janus/webrtc-3.png" width="200px" title="janus-logo" />
![Janus-core](/img/webrtc-janus/webrtc-4.png)

Janus는 Meetecho에서 개발한 범용 WebRTC 미디어서버 입니다. (’야누스’ 라고 부릅니다!)

Client와 Janus서버 간 미디어 데이터 중계 이외의 다른 기능은 제공하지 않지만 [플러그인](https://janus.conf.meetecho.com/docs/pluginslist.html)을 사용하면 추가적인 기능을 사용할 수 있습니다.

또한 Event handler를 사용하여 Janus에서 발생하는 이벤트를 수집할 수 있습니다.
<br><br>
## **AudioBridge Plugin**

음성대화방 생성/수정/삭제, 참여 인원 확인, 강제 퇴장, 음소거, 녹음 등 음성 대화에 관련된 여러 기능을 [API로 제공](https://janus.conf.meetecho.com/docs/audiobridge.html)하고 있습니다.

대화방 생성시 설정한 room/pin 값으로 음성대화방에 참여할수 있으며 음성대화는 앞서 설명한 [MCU 방식](#mcu-multipoint-control-unit)의 미디어서버로 동작합니다.

#### Janus 음성대화방 생성 API - body
```java

{
    "request" : "create",
    "room" : "Audio-Room1", // 대화방 아이디
    "description" : "1번 음성대화방", // 대화방 설명
    "secret" : "dps2E1", // 대화방 관리자 비밀번호
    "pin" : "123456", // 대화방 입장 비밀번호
    "record" : false // 대화방 레코딩(녹음) 여부
}
```
<br>
## **Event handler**

Janus에서 발생하는 이벤트를 지정한 backend에 Json데이터로 전송할 수 있으며 정의된 이벤트 타입 코드를 포함하여 전달됩니다.

사용자 액션(미디어서버 접속, 대화방 입장, 대화방 퇴장) 및 미디어서버 기동/중지 등의 이벤트를 사용하여 필요한 기능을 구현할 수 있습니다.

<img src="/img/webrtc-janus/webrtc-5.png" width="500px" title="Janus-EventTypes" />

```text
# janus.eventhandler.wsevh.jcfg

general: {
    enabled = true
    # backend로 전달할 이벤트 리스트
    events = "sessions,handles,jsep,webrtc,plugins,transports,core,external"
    grouping = true
    json = "indented"
    backend = "ws://event-listener.co.kr/handle"
}
```
<br>
## **주요 이벤트**

Core 이벤트는 미디어서버의 기동/중지 데이터를 수신할 수 있어 미디어서버의 장애 발생 시 활용할 수 있고 Session, Plugins 이벤트는 
사용자 액션 (미디어서버 접속, 대화방 입장/퇴장)에 대한 데이터를 수신할 수 있습니다.

이 세 가지 이벤트는 활용도가 높기 때문에 사용을 추천해 드립니다.

### **Core**
- 미디어서버의 기동/중지 이벤트

```java
// 미디어서버 중지 시 발생하는 이벤트 데이터
{
    "emitter": "MyJanusInstance", 
    "type": 256, 
    "subtype": 2, 
    "timestamp": 1672634437191463, 
    "event": {
       "status": "shutdown", 
       "signum": 15
    }
}
```
<br>
### **Session**
- 미디어서버를 이용하는 사용자 세션의 생성과 삭제 이벤트

```java
// 사용자 세션 생성 시 발생하는 이벤트 데이터
{
    "emitter": "MyJanusInstance",
    "type": 1,
    "timestamp": 1666924542180148,
    "session_id": 3277887048814827,
    "event": {
       "name": "created",
       "transport": {
          "transport": "janus.transport.http",
          "id": "16940507185212483599"
       }
    }
 }
```
<br>
### **plugins**
- 플러그인 관련 이벤트 (대화방 생성 및 사용자 입장/퇴장)

```java
// 음성대화방 입장시 발생하는 이벤트 데이터
{
    "emitter": "MyJanusInstance",
    "type": 64,
    "timestamp": 1666924980198028,
    "session_id": 1210004910893710,
    "handle_id": 6598715083910126,
    "opaque_id": "사용자 고유ID", // client측에서 설정
    "event": {
       "plugin": "janus.plugin.audiobridge",
       "data": {
          "event": "joined",
          "room": "audioRoom1",
          "id": 448482516212195,
          "private_id": 2786398561,
          "display": "홍길동"
       }
    }
 }
```
<br><br>

---
# 음성대화 시스템 구성

미디어서버의 경우 장애나 자원 부족 문제가 발생할 수 있기 때문에 다수의 미디어서버를 사용하며 필요시 증설하여 운영하게 되는데

여러 미디어서버를 사용하게 된다면 사용자가 어떤 미디어서버에 방을 만들어야 할지, 내가 접속할 대화방이 어디 있는지 알지 못합니다.

그렇기 때문에 사용자 대신 미디어서버를 선택하고 제공해줄 중계자 역할이 필요합니다.

![중계자](/img/webrtc-janus/webrtc-6.png)

<br><br><br>
## MRB(Media Resource Broker)

미디어서버의 중계자 역할로, 각 미디어서버의 대화방 아이디를 유니크하게 관리하고 사용자가 대화방에 접속할 때 어떤 미디어서버로 가야 하는지 알려줍니다.

또한 미디어서버 이벤트 수신(Janus Event Listener)역할도 담당하여 미디어서버에서 발생되는 이벤트를 수신하여 처리합니다.

### 대화방 생성
![대화방생성](/img/webrtc-janus/webrtc-7.png)

대화방은 Host 권한을 가진 사용자만 생성할 수 있으며, 별도의 Auth서버를 통해 Host 권한을 부여받은 유저만 대화방을 생성할 수 있습니다.

대화방 생성 시 MRB는 수집된 미디어 서버의 자원을 체크하여 가장 적은 자원을 사용하고 있는 미디어서버를 선별하여 대화방을 생성합니다.

대화방 생성 시 별도의 스토리지에 대화방 아이디와 어떤 미디어서버에 생성되었는지 저장합니다.


#### 미디어서버 선정을 위한 스코어 계산

```java
// 리소스별 가중치 설정
private static final int CPU_WEIGHT = 35;
private static final int MEM_WEIGHT = 20;
private static final int SESSION_WEIGHT = 5;
private static final int HANDLE_WEIGHT = 10;
private static final int SOCKET_WEIGHT = 30;

public double calculate(ServerMetrics metrics) {
    // 매트릭 정보가 없을 시 임의값 지정
    double totalScore = (float) ((Math.random() * 20) + 455500);

    if(metrics != null) {
        double cpuScore = CPU_WEIGHT * metrics.getCpu();
        double memScore = MEM_WEIGHT * metrics.getMem();
        double socketScore = SESSION_WEIGHT * metrics.getSocket();
        double sessionScore = HANDLE_WEIGHT * metrics.getSession();
        double handleScore = SOCKET_WEIGHT * metrics.getHandle();

        totalScore = cpuScore + memScore + socketScore + sessionScore + handleScore;
    }

    return totalScore / 100;
}
```
<br>
### 대화방 입장
    
![대화방입장](/img/webrtc-janus/webrtc-8.png)

대화방 입장 요청 시 MRB에서 관리하는 대화방 정보에서 미디어서버의 위치(접근경로)와 미디어서버에 접근할수 있는 인증토큰을 발급하여 함께 전달합니다. 

인증토큰은 미디어서버를 사용하기 위한 토큰이며 외부에서 인증되지 않은 사용자의 접근을 차단합니다. 

MRB는 인증토큰을 발급하기 위한 인증키를 미디어서버와 공유하고 있습니다.  
* 인증토큰은 되도록 짧은 시간의 유효기간을 갖도록 설정해야 합니다.  
<br>

### 이벤트 리스너
    
![이벤트리스너](/img/webrtc-janus/webrtc-9.png)
    
앞서 설명해 드린 Janus Event Handler 기능을 사용하여 MRB가 이벤트 리스너 역할을 하도록 하였습니다.
    
이벤트 중 사용자 대화방 입장/퇴장 정보는 별도로 관리하며 대화방 입장 시 인원 제한 체크, 실시간 사용자 현황 등에 활용하고 있습니다.
    
<br><br>
## 음성대화 절차

![음성대화절차](/img/webrtc-janus/webrtc-10.png)

Host 권한을 가진 유저가 대화방을 생성하게 되면, MRB에서 유니크하게 관리되는 대화방 아이디를 전달받게 됩니다.

사용자들은 공유된 대화방 아이디로 MRB에 입장요청을 하고 전달받은 미디어서버에 모여 대화를 진행하게 됩니다.

#### 대화절차
1. 대화방을 생성할 사용자가 Host권한을 부여받고 MRB로 대화방 생성 요청
2. 대화방을 생성한 사용자는 대화방 아이디를 상대방에게 공유
3. 대화방에 참석할 사용자는 MRB에게 대화방 아이디를 전달하여 입장 요청
4. 입장 정보를 가지고 대화방에 입장

  <br> 
#### 입장요청 시 제공하는 정보
- 미디어서버 접속 정보 (URL 및 인증토큰)
- 대화방 아이디, 입장 비밀번호
- TURN서버 정보

```json
{
    "code": "000",
    "data": {
        "roomNm": "audioTest Room",
        "roomId": "대화방아이디",
        "server": "webrtc-media-server.co.kr",
	"token": "미디어서버 인증토큰",
        "ice": [
            {
		"urls": "turn:webrtc-turn-server.co.kr",
                "username": "webrtc-turn",
                "credential": "webrtc-turn"
            }
        ]
    }
}
```

대화방 아이디는 WebRTC통신에서 어떤 Peer와 통신할지 Signaling을 위한 유니크한 키 라고 할수 있습니다.

Janus에서는 대화방 아이디로 Peer의 미디어 스트림을 엮어주고 데이터를 주고받게 해주는 역할을 합니다.

<img src="/img/webrtc-janus/webrtc-11.png" width="600px" title="MCU-AudioBridge" />
<br><br>

---
# 개발중 발생한 이슈
<br>
## 유효기간이 없는 미디어서버 대화방

Janus에 생성되는 대화방은 별도의 유효기간을 제공하지 않고 있습니다.

대화방 정보는 메모리에 관리되어 API로 직접 삭제하거나 서버를 재기동 해야 대화방 정보가 삭제됩니다.

그렇기 때문에 서비스 단에서 유효기간을 지정해놓아도 사용자가 입장요청에서 거절당할 뿐 실제 미디어서버에는 대화방이 남아있는 상태가 됩니다.

계속해서 대화방이 남아있게 되면 장기간 운영 시 데이터 누적으로 인한 문제나(메모리 및 중복) 이미 유효기간이 지난 대화방에 접속 등
예기치 못한 문제가 발생할 수 있기 때문에 별도의 배치를 이용하여 주기적으로 Janus에 있는 대화방을 삭제하도록 처리하였습니다.
- 대화방 생성시 설정한 관리자 비밀번호가 있어야 삭제가 가능합니다.

![대화방삭제처리](/img/webrtc-janus/webrtc-12.png)
<br><br>
## 미디어서버 재할당

위 설계대로라면, 대화방 아이디는 특정 미디어서버에 종속되고 해당 미디어서버가 문제가 생겼을 때 사용자는 미디어서버의 상태를 
알지 못하고 MRB로부터 전달받은 미디어서버로 접근하지만 결국 접속이 불가능한 상태가 됩니다.

이때, 문제가 생긴 미디어서버를 사용하는 대화방에 새로운(정상적인) 미디어서버를 재할당해주어 대화가 가능하도록 해주는 처리가 필요합니다.

![미디어서버재할당](/img/webrtc-janus/webrtc-13.png)

MRB서버에서 미디어서버의 종료 이벤트를 수신하게 되면 새로운 미디어서버를 재할당해주는 처리와 재할당된 미디어서버에 기존 대화방을 다시 만들어주는 작업을 추가하였습니다.

- Janus는 대화방을 메모리로 관리하기 때문에 중지 시 모든 대화방이 삭제됩니다.

<br>
#### 미디어서버 중지 이벤트 수신 처리
- 중지 이벤트가 발생한 미디어서버를 사용하는 대화방에 신규 미디어서버를 할당

```java
public void coreEventProcess(Event<Map> event) {
  LocalDateTime eventTime = convTimestampToLocalDateTime(event.getTimestamp()); // 이벤트 발생시간
  String serverId = event.getEmitter(); // 이벤트가 발생한 서버
  CoreSubType subType = CoreSubType.resolve(event.getSubtype());

  rtcServerRepository.findById(serverId)
    .ifPresentOrElse(rtcServer -> {
      MediaServerState state = MediaServerState.findMediaServerStatus(rtcServer.getServerSts());
	    switch (state) {
		...
		// 생략
		..
		if (CoreSubType.SERVER_SHUTDOWN == subType) {
		  // 중지된 서버에서 진행중인 대화방 조회
		  List<RtcRoom> reAllocateRoom = rtcRoomRepository.findAllByExpiryDateAfterAndServerId(LocalDateTime.now(), serverId);
		  if(reAllocateRoom.size() > 0) {
			// 대화방에 신규 미디어서버 할당
			RtcServer newServer = mediaService.allocateMediaServer();
			reAllocateRoom.stream().forEach(room -> {
			  room.setServerId(newServer.getServerId());
			  rtcRoomRepository.saveAndFlush(room);
			});
		  }
		}
	    }
	});
}
```
<br>
#### 미디어서버 기존 대화방 신규생성 처리
- 사용자 입장요청시 미디어서버에 대화방 아이디가 존재하는지 체크하고, 없다면 생성

```java
@PostMapping("/입장요청")
public ResMessage<ResJoinRoom> joinRoom(@RequestBody ReqJoinRoom joinRoom) {
    ...
    // 생략
    ..

    // 대화방 정보 조회
    RtcRoom findRoom = roomService.findRoomById(joinRoom.getRoomId())

    // 대화방 유효기간, 패스워드 체크
    RoomValidator.validRoom(findRoom)
            .checkExpiry()
            .checkRoomKey(joinRoom.getRoomKey())
            .valid();

    // 대화방 유무 체크 및 생성
    roomService.createRoomWhenNotExists(findRoom);

    // Janus MediaServer Access Token
    String toekn = tokenProvider.generateJanusToken();

    // Get ICE(Turn) Server
    List iceServer = mediaService.getAvailableTurnServer();

    ...
    // 생략
    ..

    return response;
}
```
<br>

---

# 마치며

WebRTC는 많은 기술들이 얽혀있습니다. 깊게 들어가면 NAT, 음성/영상처리, 코덱 등 네트워크 기술과 미디어에 관련된 내용도 알아야 하고 쉽게 풀어나갈 기술이 아니라고 느꼈습니다.

어려운 만큼 알아갈수록 흥미도 느꼈고, 앞으로 더 많은 서비스에 응용하여 사용할 수 있는 매력적인 기술이라 생각이 됩니다.

차후 화상채팅 및 대규모 미디어 처리를 위해 미디어서버의 수평적 확장(scale-out)을 위한 프로세스나 인프라 구성에 더 힘써야 할 것 같습니다.

감사합니다.
<br><br>

---
# 참고자료

[https://webrtc.org/?hl=ko](https://webrtc.org/?hl=ko)  
[https://webrtcforthecurious.com/](https://webrtcforthecurious.com/)  
[https://developer.mozilla.org/ko/docs/Web/API/WebRTC_API](https://developer.mozilla.org/ko/docs/Web/API/WebRTC_API)  
[https://janus.conf.meetecho.com/](https://janus.conf.meetecho.com/)
