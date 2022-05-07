---
layout: post
title:  "[Kafka]CommitFailedException"
date:   2022-05-07
author: EastGlow
categories: Kafka
---

# CommitFailedException

어느날 컨슈머 운영 중에 핀포인트에 에러 카운트가 남아있는 것을 보고 해당 파드 로그를 확인하던 중 CommitFailedException과 관련된 로그를 확인하여 대응했던 일이 있었다.

```
java.lang.IllegalStateException: This error handler cannot process 'org.apache.kafka.clients.consumer.CommitFailedException's; no record information is available

...

Caused by: org.apache.kafka.clients.consumer.CommitFailedException: Offset commit cannot be completed since the consumer is not part of an active group for auto partition assignment; it is likely that the consumer was kicked out of the group.
```

위와 같이 커밋에 실패한 모습을 볼 수 있었고 오류 로그를 보았을 때 대략적인 원인으로는 해당 컨슈머가 구독 중이던 토픽 파티션에서 쫓겨난 것(?)으로 보였다. 비슷한 사례가 있는지 찾아보았는데 꽤 흔한 사례였는지 여러 글들을 찾아볼 수 있었다.

> - [https://stackoverflow.com/questions/35658171/kafka-commitfailedexception-consumer-exception](https://stackoverflow.com/questions/35658171/kafka-commitfailedexception-consumer-exception)
> - [https://stackoverflow.com/questions/60924842/kafka-consumer-commitfailedexception](https://stackoverflow.com/questions/60924842/kafka-consumer-commitfailedexception)
> - [https://hackernoon.com/kafka-says-it-is-likely-that-the-consumer-was-kicked-out-of-the-group-az1r34dm](https://hackernoon.com/kafka-says-it-is-likely-that-the-consumer-was-kicked-out-of-the-group-az1r34dm)

## 원인은?

### `heartbeat.interval.ms`
- ConsumerCoordinator는 컨슈머 리밸런스, 오프셋 초기화, 오프셋 커밋을 담당하는데 이 코디네이터 내부에는 Heartbeat 스레드가 존재한다. 이 스레드는 그룹 코디네이터에게 얼마나 자주 컨슈머의 `poll()`로 heartbeat를 보낼 것인지 조정한다.
- 이 값은 항상 `session.timeout.ms`보다 작아야 하며 일반적으로  `session.timeout.ms`의 1/3 이하로 설정한다.
- `heartbeat.interval.ms`의 기본값은 3000ms이다.

### `session.timeout.ms`
- 컨슈머와 브로커 사이의 세션 타임아웃 시간이다. 브로커는  `session.timeout.ms`를 통해서 컨슈머가 생존여부를 파악한다. 만약 컨슈머가 그룹 코디네이터에게 heartbeat 를 보내지 않은 상태에서  `session.timeout.ms`가 지나버리면, 코디네이터는 컨슈머는 종료되거나 장애가 발생한 것으로 판단하고 컨슈머를 컨슈머 그룹에서 제외시켜버린다. 그리고 컨슈머 그룹은 리밸런싱을 수행한다.
- `session.timeout.ms`는 브로커 설정인  `group.min.session.timeout.ms`와  `group.max.session.timeout.ms`  사이 값이어야 한다.  `group.min.session.timeout.ms`의 기본값은 6000ms이며,  `group.max.session.timeout.ms`의 기본값은 버전마다 다르지만 2.7 버전 기준으로 1800000ms이다.
- `session.timeout.ms`의 기본값은 10000ms이다.
    - `session.timeout.ms`를 기본값보다 낮게 설정 시, 실패를 빠르게 감지할 수 있다. 그러나 가비지 컬렉션이나 `poll()`을 완료하는 시간이 길어지게 되면 원하지 않게 리밸런싱이 일어날 수 있다.
    - `session.timeout.ms`를 기본값보다 높게 설정 시, 리밸런싱이 일어날 확률이 적어지지만, 오류를 감지하는데 시간이 오래 걸릴 수 있다.

### `max.poll.interval.ms`
- 컨슈머가 지속적으로 heartbeat만 보내고 실제로 메시지를 가져가지 않는 경우가 있을 수 있다. 이럴 때 해당 컨슈머가 특정한 파티션을 무한정으로 해당 파티션을 점유할 수 없도록 주기적으로 `poll()`을 호출하지 않으면 장애라고 판단한다. 이후 컨슈머 그룹에서 제외한 후, 동일 컨슈머 그룹 내의 다른 컨슈머가 해당 파티션에서 메시지를 들고 갈 수 있도록 한다.
- 쉽게 말하자면 특정 컨슈머가 특정 오프셋의 데이터를  `poll()`해서 메시지를 가져온 뒤, 해당 메시지로 특정 처리를 하는 시간이 `max.poll.interval.ms` 이상으로 걸리게 되면 해당 컨슈머는 정상이 아닌 것으로 간주되어 컨슈머 그룹에서 제외된다.
- `max.poll.interval.ms`의 기본값은 300000ms이다.

> 참고:
> - [https://d2.naver.com/helloworld/0974525](https://d2.naver.com/helloworld/0974525)
> - [https://pasudo123.tistory.com/435](https://pasudo123.tistory.com/435)
> - [https://docs.confluent.io/platform/current/installation/configuration/](https://docs.confluent.io/platform/current/installation/configuration/)
> - [https://kafka.apache.org/documentation/#configuration](https://kafka.apache.org/documentation/#configuration)

위 3가지 Config가 일반적으로 `CommitFailedException`을 일으키는 원인들로 지목됐는데 우리 팀에서 겪은 케이스는 `max.poll.interval.ms`가 문제였다. 특정 토픽의 처리 프로세스에서 특정 쿼리 실행 시간이 10분 이상 걸리게 되면서 `max.poll.interval.ms`의 기본값인 300000ms(5분)을 넘어서게 되면서 해당 컨슈머는 장애로 판단되어 프로세스 처리 도중 리밸런싱이 일어나게 된 것이었다. 상황 재현을 위해 로컬에서 컨슈머를 띄워놓고 `Thread.sleep()`을 통해 딱 5분을 처리 지연시켜보았는데 바로 같은 Exception이 발생하는 것을 확인할 수 있었다.

이렇듯 문제가 된 프로세스의 처리 시간이 오래 걸리게 되면서 해당 파드(컨슈머)에 장애가 났다고 판단하여 컨슈머 그룹에서 쫓겨나게 되었고 그로 인해 카프카 쪽에 커밋이 완료되지 않은 채 리밸런싱이 일어나게 되었다. 그래서 다시 다른 파드에서 이 파티션을 컨슘하게 되고 또 문제가 발생하고... 돌고돌아 작업이 완전히 끝날 때까지 여러번 같은 컨슘 행위가 반복되었다.

## 해결은 어떻게?

가장 원시적인 방법으로는 `max.poll.interval.ms` 값을 더 큰 값으로 바꾸는 것이었다. 하지만 당시엔 문제가 되던 프로세스가 데이터양에 따라 프로세스 처리 시간이 천차만별이었기 때문에 만약 기본값인 5분보다 더 큰 10분으로 설정해도 10분 이상 넘어가는 프로세스 처리가 나오지 않으리란 법이 없어서 효과가 없을거라 판단되었다. 뭐, 결국 카프카 관련 Config를 건드는 방법보단 시간이 오래 걸리던 해당 프로세스의 로직을 개선하는 것으로 해결을 보긴해서 처음 이 오류가 발견되었을 땐 이 값의 변화는 주지 않았다.

위에 써두었듯이 이 오류를 최초로 발견하고 대응하였을 땐 이 토픽 프로세스 자체를 손 보아서 처리시간이 최대 5분을 넘어가지 않도록 해결을 보았다. 그러나 작년 사내에 오픈마켓 시스템이 오픈되면서 사내 데이터베이스에 쌓이는 상품 데이터가 급격하게 늘어나게 됨에 따라 또 다시 같은 현상이 발생하게 되었다.

그러면 두번째 발생 상황 땐 어떻게 해결하였을까... 우선 먼저 하나 짚고 넘어가야 할 부분이 있다. 컨슈머에서 한번의 poll()로 적게는 몇개, 많게는 몇십 몇백개의 메시지를 처리하게 된다. **(몇 개의 메시지까지 가져올 지는 위 두 Config 외에도 fetch.max.bytes, fetch.min.bytes, max.partition.fetch.bytes 등 여러 Config가 복합적으로 관여한다고 한다.)**

> 참고: [https://medium.com/11st-pe-techblog/%EC%B9%B4%ED%94%84%EC%B9%B4-%EC%BB%A8%EC%8A%88%EB%A8%B8-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EB%B0%B0%ED%8F%AC-%EC%A0%84%EB%9E%B5-4cb2c7550a72](https://medium.com/11st-pe-techblog/%EC%B9%B4%ED%94%84%EC%B9%B4-%EC%BB%A8%EC%8A%88%EB%A8%B8-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EB%B0%B0%ED%8F%AC-%EC%A0%84%EB%9E%B5-4cb2c7550a72)
>
> (3년이 지난 글이지만 Kafka의 기본 동작 원리나 이 글에서 설명할 리밸런싱 관련된 부분들을 아주 잘 설명해둔 글이라 한번쯤 읽어두면 도움이 많이 된다.)

쉽게 예를 들자면, 현재 A 토픽의 0번 파티션에는 10개의 메시지가 들어와있으며 이를 구독하는 a-0 파드가 0번 파티션의 0번 오프셋부터 10개의 메시지를 컨슘하게 되었다. *(= 0번 ~ 9번 오프셋 총 10개의 메시지를 가져옴)*

그런데 메시지 1개 당 처리 시간이 대략 40초가 걸린다고 하자. 산술적으로 메시지 10개를 모두 처리하는데는 대략 400초가 걸리게 되는데 위에서 언급한 `max.poll.interval.ms`의 기본값인 300000ms(5분)를 넘어가게 된다. 정해진 시간인 300초 안에 다음 poll()이 없었으므로 Kafka 쪽에서는 이 a-0 파드가 장애가 났다고 판단하고 속해있던 컨슈머 그룹에서 내쫓아버린다. 그러면 나머지 파드들끼리 다시 리밸런싱하여 이 0번 파티션을 구독하게 된다.

그렇게되면 리밸런싱 후 새롭게 처리를 시작하는 파드들에서는 이전에 장애가 발생한 파드에서 처리가 완료된 0번 ~ 6번 오프셋 메시지(40초*7개=280초)는 컨슘하지 않고 300초가 넘어가게 되면서 처리한 7번 ~ 9번 오프셋 메시지만을 다시 컨슘해서 처리하는가... 그건 또 아니다.

위에서 얘기하였듯이 `max.poll.records`에 설정한 값대로 poll()하였을 때 메시지를 일종의 벌크 단위로 가져와서 모두 정상 처리가 되어야 카프카 쪽으로 커밋을 완료하게 된다. 그래서 리밸런싱 후 다른 파드가 0번 파티션을 다시 컨슘하면 0번 ~ 9번 오프셋 메시지를 다시 처음부터 처리하게 되는 것이다. 실제로 이번 사례에서도 서로 다른 파드들이 이미 처리된 오프셋을 재시도하여 또 처리하는 경우가 발견되었다.

### 당시 남아있던 오류 로그의 일부
- 원본 로그의 날짜나 토픽명은 이 글에선 임의의 값으로 변경함

```
Apr 14, 2022 @ 03:43:58.000 Caused by: org.apache.kafka.clients.consumer.CommitFailedException: Offset commit cannot be completed since the consumer is not part of an active group for auto partition assignment; it is likely that the consumer was kicked out of the group.

......

Apr 14, 2022 @ 03:14:15.000 2022-04-14 03:14:15.332 INFO 1 --- [ntainer#1-0-C-1] o.s.k.l.KafkaMessageListenerContainer : eastglow-consume-group: partitions assigned: [eastglow-consumer-eastglow-test-1, eastglow-consumer-eastglow-test-0, eastglow-command-eastglow-test2-1, eastglow-command-eastglow-test2-0]

Apr 14, 2022 @ 03:14:15.000 2022-04-14 03:14:15.332 INFO 1 --- [ntainer#2-0-C-1] o.s.k.l.KafkaMessageListenerContainer : eastglow-consume-group: partitions assigned: [eastglow-consumer-comm-notifyFailMsg-0]

Apr 14, 2022 @ 03:14:15.000 2022-04-14 03:14:15.332 INFO 1 --- [ntainer#0-0-C-1] o.s.k.l.KafkaMessageListenerContainer : eastglow-consume-group: partitions assigned: [eastglow-consumer-eastglow-test3-0]

Apr 14, 2022 @ 03:14:15.000 2022-04-14 03:14:15.331 INFO 1 --- [ntainer#0-0-C-1] o.s.k.l.KafkaMessageListenerContainer : eastglow-consume-group: partitions assigned: [eastglow-consumer-eastglow-test3-2]

Apr 14, 2022 @ 03:14:15.000 2022-04-14 03:14:15.203 INFO 1 --- [ntainer#0-0-C-1] o.s.k.l.KafkaMessageListenerContainer : eastglow-consume-group: partitions revoked: [eastglow-consumer-eastglow-test3-0]

Apr 14, 2022 @ 03:14:15.000 2022-04-14 03:14:15.233 INFO 1 --- [ntainer#2-0-C-1] o.s.k.l.KafkaMessageListenerContainer : eastglow-consume-group: partitions revoked: [eastglow-consumer-comm-notifyFailMsg-0]

Apr 14, 2022 @ 03:14:14.000 2022-04-14 03:14:14.523 INFO 1 --- [ntainer#1-0-C-1] o.s.k.l.KafkaMessageListenerContainer : eastglow-consume-group: partitions revoked: [eastglow-consumer-eastglow-test-2, eastglow-command-eastglow-test2-2]

Apr 14, 2022 @ 03:14:14.000 2022-04-14 03:14:14.581 INFO 1 --- [ntainer#0-0-C-1] o.s.k.l.KafkaMessageListenerContainer : eastglow-consume-group: partitions revoked: [eastglow-consumer-eastglow-test3-2]

Apr 14, 2022 @ 03:14:14.000 2022-04-14 03:14:14.629 INFO 1 --- [ntainer#2-0-C-1] o.s.k.l.KafkaMessageListenerContainer : eastglow-consume-group: partitions revoked: [eastglow-consumer-comm-notifyFailMsg-2]

Apr 14, 2022 @ 03:14:13.000 2022-04-14 03:14:13.924 INFO 1 --- [ntainer#1-0-C-1] o.s.k.l.KafkaMessageListenerContainer : eastglow-consume-group: partitions revoked: [eastglow-consumer-eastglow-test-1, eastglow-command-eastglow-test2-1]
```

위와 같은 상황에서 우선은 시간이 오래 걸리던 데이터를 분석하여 먼저 관련 CRUD 쿼리를 다시 손 보았고 두번째로는 `max.poll.records`와 `max.poll.interval.ms` 설정을 바꿔주게 되었다. 문제가 되던 프로세스는 데이터가 적으면 문제가 되지 않지만 상황에 따라 적게는 몇 건부터 많게는 몇 십 만건까지 처리 대상 데이터 모수의 차이가 있어서 아무리 쿼리 튜닝을 해도 느린 부분이 있기 때문에 짧으면 30초, 길면 2~3분 이상 까지 처리가 걸리는 경우도 있다고 판단되었다.

그리고 이 문제가 되던 프로세스는 1번의 처리에 1개의 마스터 데이터와 그 하위에 존재하는 N개의 종속 데이터를 처리해주고 있다. 즉 마스터 데이터를 기준으로 보면 메시지 1개 당 데이터 1개 단건 단위로 이루어지기 때문에 가능하면 1개의 메시지만 컨슘하여 처리한 뒤 카프카로 커밋해주는 방식이 오히려 좋을 수도 있다는 의견이 나왔다.

그리고 관련 사례들을 구글링해보니 `max.poll.records`를 1로 설정하면 컨슈머의 컨슘 능력이 하락할 수 있다는 의견도 있으나, 어차피 카프카 쪽에 메시지를 요청할 때 이미 파티션별로 어느정도 요청받고 전달할 메시지를 가지고 있기 때문에 늘 새로운 요청마다 새로운 메시지를 꺼내서 주는게 아니라서 큰 성능 하락은 없다고 하여 설정해보기로 했다. (웹으로 치면 일종의 캐싱 관련 영역이라고 보면 될 것 같다. 매번 DB를 뒤져서 데이터를 주는게 아닌 캐시된 데이터를 넘겨준다고 보면 된다. 이미 적재된 메시지의 내용은 바뀔 일이 없을테니깐 말이다.)

`max.poll.records`를 **1로 설정해줌**과 동시에 `max.poll.interval.ms`도 **기존의 300초에서 600초**까지 늘려주었다. 물론 이 토픽에 한해서만 이 두 설정을 해주었기 때문에 다른 토픽들은 기존처럼 기본값대로 움직이고 있다. 참고로 설정해둔 KafkaListener마다 개별 properties를 설정할 수도 있다. (하단의 참고 부분에 기재해두었다.)

현재까지 이렇게 운영해온 지도 몇 달 지났는데 특별히 해당 토픽에 대해서 문제도 발생하지 않고 있고 데이터 정합성에도 큰 문제를 보이지 않고 있다.

## 참고

관련 사례들을 찾다보니 아래와 같은 것들도 알 수 있었다.

-  `max.poll.interval.ms`  등의 Config는  `@KafkaListener`  단위로 설정도 가능하였다. 글로벌로 설정하는 Config가 기본값이고 실제 Listener가 생성될 때는 별도의 값을 설정했다면 그 값을 따라가게 되어있다.
	```
	@KafkaListener(topics = "myTopic", groupId = "group", properties = {
		"max.poll.interval.ms:60000",
		ConsumerConfig.MAX_POLL_RECORDS_CONFIG + "=100"
	})
	```
