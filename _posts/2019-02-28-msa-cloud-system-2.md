---
title: 분산처리 시스템 구조개선 (2)
layout: post
comments: true
social-share: true
show-avatar: true
author: 박동형
bigimg: "/img/distributed/big_image_v2.png"
tags:
- MSA
- Docker
- Docker Swarm
- Apache Kafka
- KStream
- Spring Cloud Stream
---

**도메인 별 프로세스**와 **구조개선을 통해 나아진점**을 설명

## 구성

1. [분산처리 시스템 구조개선 (1)]({{ site.url }}/2019-02-25-msa-cloud-system-1/){: target="_blank" }
2. [분산처리 시스템 구조개선 (2)]({{ site.url }}/2019-02-28-msa-cloud-system-2/){: target="_blank" }


---

<br>

## 도메인 역할 및 프로세스

앞선 포스트에서 설명드렸듯이 기존 시스템의 Engine이 가지고 있던 기능들을 도메인으로 역할을 분리했고, 컨테이너화를 통해
독립적인 애플리케이션으로 구축을 진행했습니다.  

저희가 운영중인 시스템에는 **캠페인 정보, Push 메시지 생성 결과에 대한 Summary 데이터를 저장하는 DB**와 **스케줄러에서 스케줄 관리를 위한 DB**로 2개가 구축되어 있습니다.  
도메인의 분리는 DB의 분리도 같이 진행되야 하지만 각 도메인에서는 캠페인 정보 및 Summary 데이터의 조회, 저장을 위해 같은 DB를 연결하고 있습니다.  
많은 DB 커넥션을 요구하지 않는 시스템으로 데이터 저장은 분리된 도메인에서만 일어나기 때문에 API 구축은 진행하지 않았습니다.

### 관리자
현 시스템의 모든 시작점은 캠페인 정보 등록에서 시작됩니다. Push 제목, 파라미터 정보, 스케줄 정보 등 Push 메시지 생성을 위한 정보가 담겨집니다.  

```java
@Data
@NoArgsConstructor
public class Campaign implements Serializable {

	// 캠페인 아이디
	private Integer id;
	
	// 캠페인 명
	private String name;
	
	// 캠페인 설명
	private String description;
	
	// 캠페인 작성자, 등록자
	private String register;
	
	// 에이전트 타입
	private Agent type;
	
	// 발송 스케쥴
	private SendSchedule schedule;
	
	// 상태
	private State state = State.UNREGISTER;
	
	// 캠페인 최초 등록시간
	private LocalDateTime createdDate;
	
	// 캠페인 최종 수정시간
	private LocalDateTime modifiedDate;
	
	// 푸시 정보
	private Push push;
		
	//푸시 타이틀
	private String title;
		
	public Campaign(int id) {
		this.id = id;
	}
		
}
```

캠페인 객체 정보는 DB에 저장되며 다른 도메인에서도 해당 DB를 통해 정보를 조회하게 됩니다.
사람인 DB는 mysql 5.7이상을 사용하고 있기 때문에 캠페인 내에 **스케줄 정보, 각 Push 별 정보는 json타입**으로 저장하여 하나의 테이블로 저장하고 있습니다.

[<img src="{{site.url}}/img/distributed/admin_1.png" />]({{site.url}}/admin_1.png)  
관리자 흐름도
{: .text-center }

예약상태(RUNNING)를 갖는 캠페인이 등록되면 스케줄러에 실제 스케줄 정보를 등록하고 관리하기 위해 Kafka로 메시지를 발행(published)합니다.  
기존 관리자는 active/standby 형식으로 구축이 되어 있었습니다. 하지만 docker의 도입으로 장애상황에 대해 컨테이너 스케줄링이 가능하므로 
2대의 active/active 형식으로 구축하였고, 세션 공유를 위해 redis를 연동한 session clustering을 구축했습니다.

### 스케줄러
[<img src="{{site.url}}/img/distributed/scheduler.png" />]({{site.url}}/scheduler.png)  
스케줄러 흐름도
{: .text-center }
스케줄러는 관리자에서 발행한 메시지를 수신하여 스케줄을 등록하고 에이전트 실행을 위한 메시지 발행하는 역할을 가지고 있습니다. 
다른 시스템과 공용으로 사용되는 시스템으로 구축됐으며, 구축된 여러 에이전트로 메시지를 발행하게 됩니다.  
Quartz 라이브러리는 DB 연결을 통해 clustering 환경을 구축할 수 있습니다. 그에 따라 failover 상황에 대해 대응 할 수 있으며 따로 상태 공유를 위한 개발 공수가 줄어 들게 됩니다.

### 에이전트
[<img src="{{site.url}}/img/distributed/agent_1.png" />]({{site.url}}/agent_1.png)  
에이전트 흐름도
{: .text-center }

핵심 도메인 에이전트는 대량의 실시간성 데이터에 대해 빠른 처리를 할 수 있습니다.  
Spring Cloud Stream 프레임워크를 도입하여 에이전트의 프로세스에 대해 Kafka로 데이터 처리를 진행하고 있습니다.
kafka 전송을 위한 설정은 yaml 파일을 통해 쉽게 설정이 가능하며, 어노테이션 설정으로 프로세스들의 연결을 처리할 수 있습니다.
```yml
spring:
  main:
    web-application-type: NONE
  cloud:
    stream: 
      kafka:
        streams.binder: 
          brokers: {kafka server list}
          min-partition-count: 3
          replication-factor: 3
          configuration: 
            default.key.serde: org.apache.kafka.common.serialization.Serdes$IntegerSerde
            default.value.serde: org.apache.kafka.common.serialization.Serdes$StringSerde
            commit.interval.ms: 1000
        binder: 
          brokers: {kafka server list}
          min-partition-count: 3
          replication-factor: 3
      bindings: 
        target-out.destination: target
        target-in.destination: target
        target-in.group: target
        process-out.destination: process
        process-in.destination: process
        process-in.group: process
        result-out.destination: result
        result-in.destination: result
        result-in.group: result
        summary-out.destination: summary
        summary-in.destination: summary
        summary-in.group: summary
```
<br>
yaml 파일 설정에 Input, Output에 대한 interface를 작성하여 어노테이션 설정을 잡아주면 간단히 프로세스간의 kafka 연결 환경이 설정됩니다.
```java
// 대상자 조회 프로세스 바인더
public interface TargetUserBinder {

	String TARGET_OUT = "target-out";
	String TARGET_IN = "target-in";
	
	@Output(TARGET_OUT)
	MessageChannel targetSearch();
	
	@Input(TARGET_IN)
	SubscribableChannel targetFilter();
	
}

// Push 메시지 생성 프로세스 바인더
public interface PushMessageBinder {

	String PROCESS_OUT = "process-out";
	String PROCESS_IN = "process-in";
	
	@Output(PROCESS_OUT)
	MessageChannel createMessage();
	
	@Input(PROCESS_IN)
	SubscribableChannel messagePush();
	
}
```
<br>
연결에 대한 설정이 완료되면 실제 프로세스에서 Listener 어노테이션 설정으로 데이터 수신, 전송이 이뤄집니다.
```java
// 대상자 조회 후 대상자별로 데이터 설정 후 kafka 전송
@Slf4j
@EnableBinding(value= {TargetUserBinder.class, PushMessageBinder.class})
public class TargetUserProcess {
	
	private PushMessageBinder pushMessageBinder;
	
	@Autowired
	public TargetUserProcess(PushMessageBinder pushMessageBinder) {		
		this.pushMessageBinder = pushMessageBinder;
	}

	//바인더를 통한 데이터 전송
	@StreamListener(TargetUserBinder.TARGET_IN)
	public void targetUserSearch(@Payload TargetMeta targetMeta) {
		pushMessageBinder.createMessage().send(MessageBuilder.withPayload(targetMeta).build());
	}
}

// 대상자별 데이터를 기반으로 메시지 생성 후 전송, 결과 데이터 리턴
@Slf4j
@EnableBinding(value= {PushMessageBinder.class, ResultBinder.class})
public class PushMessageProcess {

	@StreamListener(PushMessageBinder.PROCESS_IN)
	@SendTo(ResultBinder.RESULT_OUT)  // 어노테이션 설정으로 데이터 전송
	public Summary messagePush(@Payload TargetMeta targetMeta) {
		return summary;
	}
}
```
프로세스에서 다음 프로세스 데이터 전송은 **메소드 단위의 어노테이션 설정**과 **바인더를 통한 전송**이 가능합니다.  

대상자에 대해 메시지 전송이 끝나면 결과를 통합하여 summary 데이터를 저장하고 있으며, kstream을 이용하여 간편히 데이터 처리를 진행했습니다.
```java
@Slf4j
@EnableBinding(value= {ResultBinder.class, SummaryBinder.class})
public class SummaryProcess {
	
	@StreamListener(ResultBinder.RESULT_IN)
	@SendTo(SummaryBinder.SUMMARY_OUT)
	public KStream<Integer, Summary> process(KStream<Integer, Summary> input) {
		
		// 4. 요약 처리 
		return input
			.selectKey((k,v) -> v.getId())
			.groupByKey(Serialized.with(Serdes.Integer(), serde))
			.reduce((v1, v2) -> {
				Summary summary = new Summary(v1.getId(), v1.getEventMessageId(), v1.getCampaignId());
				summary.setSuccess(v1.getSuccess() + v2.getSuccess());
				summary.setError(v1.getError() + v2.getError());
				summary.setMissing(v1.getMissing() + v2.getMissing());
				summary.setFiltered(v1.getFiltered() + v2.getFiltered());
				return summary;
			}, Materialized.<Integer, Summary, KeyValueStore<Bytes, byte[]>>as(STORE_NAME).withKeySerde(Serdes.Integer()).withValueSerde(serde))
			.toStream().map((k, v) -> new KeyValue<>(k, v));
	}
	
	/*
	 * 5. 요약 수신
	 */
	@StreamListener(SummaryBinder.SUMMARY_IN)
	public void listener(@Payload Summary data) {
		// reduce 처리된 데이터에 대해 DB 업데이트 처리
	}
}
```
kstream의 데이터 집합 처리는 상태, 무상태 처리로 구분이 되어집니다. 저희 같은 경우 reduce를 통한 데이터 처리를 진행했고 각 상태를 저장하여 이전의 결과를
사용해야 하므로 따로 windowedBy 처리를 하지 않고 Key-Value 상태 저장소를 사용했습니다.  
reduce 처리가 된 데이터는 마이크로 배치 단위로 전송되어 Summary 데이터로 저장이 되며, 총 데이터 수가 맞으면 해당 스케줄에 대해 Complete 상태 업데이트가 됩니다.

현재까지 설명을 도식화하면 다음과 같습니다.

[<img src="{{site.url}}/img/distributed/agent_process_1.png" />]({{site.url}}/agent_process_1.png)  
에이전트 프로세스 - 1
{: .text-center }
[<img src="{{site.url}}/img/distributed/agent_process_2.png" />]({{site.url}}/agent_process_2.png)  
에이전트 프로세스 - 2
{: .text-center }

---

<br>
## 구조개선... 그 뒤에는...?
현재 시스템은 안정적으로 운영되고 있으며, 기존 문제점에 대해서도 개선되어 운영에 있어서 많은 변화가 생겼습니다.  

> 메시지 전송 손실 비율 **0%**  
> Push 생성 시간 감소 **2min**  
> 빌드, 배포, 구동 시간 단축 **약 70% 감소 (10min -> 3min)**  
> Docker Swarm 구성하에 **1click**으로 쉽게 Scale out 가능  
> 도메인별 자유로운 운영 가능

컨테이너 장애 상황에 대해 메시지 손실없이 대응이 되는 상황으로 처음 기대하고 설계대로 진행되고 있으며, 기존 대비 프로세스 시간도 감축되어 기대외의 결과도 얻었습니다.  
빌드에서 애플리케이션 구동까지의 시간은 체감이 될 정도로 줄어들었으며, 새로운 노드 추가에 대해서도 replica 설정 한번에 쉽게 scale out이 가능합니다.

아직은 훌륭한 구조개선이란 평가는 어렵지만, 충분히 의미있는 프로젝트였다고 생각되며 사람인 IT 연구소 조직에 처음으로 도입된
클라우드 환경을 고려한다면 앞으로의 운영노하우가 기대되는 프로젝트였습니다.

감사합니다.