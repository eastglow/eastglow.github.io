---
layout: post
title:  "[Kafka]Kafka Partitioner"
date:   2022-05-07
author: EastGlow
categories: Kafka
---

# Default Partitioner

사내에 본격적으로 Kafka 도입이 된 지도 1년이 좀 넘은 듯 하다. 구축부터 세팅, 관리까지는 별도의 DevOps팀이 있어서 이쪽 팀에서 책임져주고 있다. 이렇게 Kafka를 도입하여 실제 Producer/Consumer를 개발하여 운영해오면서 겪어본 사례 중 하나를 소개해보고자 한다.
Kafka 도입 후에 컨슈머에서는 별도의 Custom Partitioner를 사용하고 있지 않았고, 메시지를 발행할 때도 별도의 Key값 지정 또한 해주지 않고 있었다. 이유는 Spring for Apache Kafka에서 구현된 Default Partitioner의 작동 방식은 기본적으로 Key값이 null일 경우엔 별도의 계산을 거치지 않고 현재 생성된 파티션 개수를 이용하여 라운드-로빈 방식으로 파티션 번호를 구해서 메시지를 할당하고 있다.

그렇기 때문에 팀 내에서도 Custom Partitioner나  Key값 설정에 대한 별도의 니즈(?)가 없었고 3개면 3개, 5개면 5개의 파티션에 메시지가 알아서 분할되어 잘 들어가고 있었기 때문이다.

> 참고: [https://jaceklaskowski.gitbooks.io/apache-kafka/content/kafka-producer-internals-DefaultPartitioner.html](https://jaceklaskowski.gitbooks.io/apache-kafka/content/kafka-producer-internals-DefaultPartitioner.html)


## 이슈 발생

그런데 컨슈머를 운영하다보니 하나 문제점을 발견할 수 있었다. 모든 토픽 프로세스에 대한 문제는 아니고, 우리 팀에서 운영 중인 A토픽 처리 프로세스에서 문제가 있었는데 이 A토픽을 컨슘하여 처리하던 프로세스엔 데이터베이스에서 A토픽의 메시지에 포함된 PK에 해당되는 필드값으로 insert 혹은 update해주는 로직이 있었다.
당연히 PK에 해당되는 값으로 처리하려면 동시성 제어가 필수이고 같은 PK값으로 다른 한 쪽에서 처리 중인데 또다른 한 쪽에서 무언가를 하려고하면 "ORA-00001" 오류 메시지를 마주할 수 있게 된다.

A토픽의 메시지는 같은 PK 필드값을 가진 채 여러 개가 발행될 수 있는 상황이었고 그로 인해 해당 토픽을 구독 중이던 3개의 컨슈머 파드들은 서로 각자 다른 파드(컨슈머)였지만 같은 PK 필드값을 가진 메시지를 동시에 컨슘하여 가져다 쓰는 상황이 발생할 수 있었다.

> 현재 사내에서는 토픽 내 파티션 개수는 기본 3개를 생성하는게 규칙이고 A토픽 역시 메시지 양이 엄청 많은 편은 아니었기에 기본 3개만을 이용하여 사용하고 있었다. 대부분의 Kafka 관련 자료에서 "파드 수 = 파티션 수"로 권장하고 있고 어차피 동일한 컨슈머 그룹 내 컨슈머들은 동일한 파티션에 붙을 수 없기에 컨슈머 수가 파티션 수보다 많아져봤자 놀고 있는 컨슈머가 생기기 때문에 리소스가 남아버린다.

그래서 같은 PK를 가진 메시지를 어떻게 순차 처리를 해줄까 고민하다가 발견한 것이 Spring for Apache Kafka에서 제공하고 있는 Default Patitioner에 대한 것이었다. 물론, Custom Partitioner를 별도로 구현해도 되었지만 Default Partitioner에서 제공하는 파티션 분할 로직으로도 충분히 위 문제의 대응이 가능했었고 당시엔 하루하루 이슈 발생에 대한 위험성(?) 때문에 불안한 상태였다.
여하튼, Default Partitioner를 정상적으로 이용하기 위해 위 A토픽 처리 프로세스에 대한 메시지 발행 시 기본 Key값을 할당해주었는데 그 Key값은 PK 필드값을 사용하였다. 만약 Key값이 존재한다면 해당 Key의 Bytes값으로 특정 해쉬값 처리를 한 뒤 mod 처리하여 Partition 번호를 구해오도록 되어 있었기 때문에 동일한 PK값을 가진 메시지들은 같은 파티션으로만 쌓이게 되었고 1개의 파티션은 1개의 파드(컨슈머)만 구독이 가능하기 때문에 순차 처리가 가능하게 되었다.

*즉, PK값이 같은 메시지는 무조건 같은 파티션으로만 가고 그 파티션은 하나의 파드(컨슈머)만이 바라보고 처리하기 때문에 서로 다른 파드에서 같은 PK값을 처리할 일은 없어진 것이다.*

아래는 DefaultPartitioner 내 코드 중 `key != null` 일 때 동작하는 코드 중 일부이다.

```
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic); // 해당 토픽의 유효한 파티션 리스트 구함
int numPartitions = partitions.size(); // 유효한 파티션 개수 (일반적으로 3개)
return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions; // murmur2 해쉬값 치환 후 위에서 구한 파티션 개수와 mod 연산 실행
```

## 이슈 해결

위 사례를 예로 들어 설명하자면 0000000001, 0000000002, 0000000003, 0000000004 라는 4개의 PK값을 가진 메시지 4개가 있다고 하자. 실제 연산되어 나오는 파티션은 조금 다를 수 있겠지만 쉽게 설명하자면 홀수의 메시지는 0번 파티션으로, 짝수의 메시지는 1번 파티션으로만 들어가게 되는 것이다.
하지만 위와 같이 같은 파티션으로 같은 값의 PK를 Key로 하여 메시지를 발행하게 된다면 결과적으로는 순차 처리를 하게 되어 PK 위반 오류는 발생하지 않게 되었지만 그만큼 메시지 컨슘 능력이 떨어지게 되는 단점도 생기게 되었다. 하지만 팀원들과 논의해본 결과 A토픽 처리 프로세스 특성상 들어온 순서대로 순차 처리되는게 맞다고 판단되어 이부분은 어느정도 감안하고 넘어가기로 하였다.

> 아, 물론 같은 PK를 가진 동일한 내용의 메시지가 중복되어 단시간에 여러 번 발행하는 로직 쪽도 근본적으로는 문제가 있기 때문에 이부분도 같이 손보게 되었다. 이전보다 같은 PK값을 가진 메시지가 들어오는 케이스가 현저히 적어졌기에 위 컨슈머 쪽 개선 내용과 같이 좋은 시너지를 낼 수 있었다.

여기서 좀 더 특별한 파티션 분할 로직이 필요하다면 `org.apache.kafka.clients.producer.Partitioner`  인터페이스를 직접 구현한 뒤,  `partitioner.class`  Config를 추가하여 설정하면 된다고 한다. Custom Partitioner까지는 필요한 상황이 아니었기에 Default Partitioner로 현재까지 큰 이슈없이 운영하고 있다.
