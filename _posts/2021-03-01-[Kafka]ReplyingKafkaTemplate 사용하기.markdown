---
layout: post
title:  "[Kafka]ReplyingKafkaTemplate 사용하기"
date:   2021-03-01 22:50:00
author: EastGlow
categories: Back-end
---

작년 말부터 사내에 구축되어 있는 Kafka를 이용하여 이것저것 해보고 있다. RabbitMQ같은 메시지 큐 시스템은 익히 들어 알고 있었고 Kafka도 직접 써본 적은 없지만 여러 세미나나 컨퍼런스에서 조금씩 주워들은(?) 경험이 있어서 어느정도 알고는 있었다.

마침 사내 인프라 쪽이 작년부터 MSA 환경으로 넘어가고 있는 과정이었는데 Kafka도 그 과정에서 세팅되었다. 우리 파트에서도 Kafka를 이용하여 여러가지 기능이나 인프라를 구축해보기 위해 이용할 예정이었고 내가 우선적으로 Kafka 관련 업무를 배정받아서 기초적인 프로젝트 세팅 및 기본적인 정보들을 공유해주고 있는 상황이다. (아직 나도 많이 모르는 부분들이 많아서 한창 공부 중이다.)

이전에 Kafka에 대한 간단한 정보들을 정리해둔 글이 있는데 혹시나 나처럼 Kafka를 접한 지 얼마 되지 않은 사람이라면 아래 링크를 한번 보고 오는 것도 좋을 것 같다.

> https://eastglow.github.io/back-end/2020/05/17/Kafka-%EC%B9%B4%ED%94%84%EC%B9%B4-%EA%B8%B0%EC%B4%88%EA%B0%9C%EB%85%90-%EC%A0%95%EB%A6%AC%ED%95%98%EA%B8%B0.html

아무튼, 오늘 소개하고자 하는 것은 KafkaTemplate 중 하나인 `ReplyingKafkaTemplate`이다. Kafka를 이용하고자 하는 목적중 하나를 꼽자면 일반적으로는 `비동기 메시지 큐` 이용을 위해 쓰는 경우가 많을 것이다.

메시지 생산의 역할을 맡는 프로듀서(Producer)는 특정 동작을 취한 후 메시지를 Kafka의 약속된 Topic으로 쏴주기만 하면 그 토픽을 구독하고 있던 컨슈머(Consumer)는 그 Topic으로 들어온 메시지를 소비하여 후속 프로세스를 수행하는 식으로 많이 이용한다.

프로듀서는 메시지를 쏴주고 나서 그 후의 일은 전혀 관여할 필요도 없고, 관여할 수도 없다. 왜냐면 해당 메시지를 소비하는 쪽은 프로듀서와 전혀 다른 컨슈머에서 그 후의 일이 이루어지기 때문이다. 그래서 일반적인 동기 프로세스와 다르게 프로듀서는 메시지만 쏴주면 다시 그 동작을 수행하거나 다른 동작을 수행해도 문제가 없다.
컨슈머 쪽에서도 소비(consume) 능력에 따라서 메시지를 가져와서 후속 프로세스를 진행하면 되기 때문에 프로듀서에서 1초 동안 10개의 메시지를 보냈다고 컨슈머에서 무조건 1초 동안 10개의 메시지를 소비하여 후속 프로세스를 수행할 필요는 없는 것이다. 그냥 컨슈머의 역량에 따라 메시지를 소비하여 진행하면 되는 것이다.
> 아, 물론 컨슈머의 역량을 넘어서는 메시지를 생산하여 컨슈머 쪽에 Lag이 발생하는 것이 무조건 정상은 아니다. 이경우엔 컨슈머의 Config를 조정해주거나 스펙을 올리는 등의 방법으로 소비 능력을 향상시켜주든지, Partition을 추가하여 좀 더 원활한 소비를 할 수 있게 해줘야한다.

그런데 어떤 특정 동작에서는 Kafka로 메시지를 쏴주고 컨슈머 쪽에서 해당 메시지를 이용한 처리를 하고나서 그 결과값을 다시 돌려받고 싶을 수도 있다. 즉, 일반적인 HTTP API나 REST API처럼 Request가 있으면 Response를 받고 싶은 것이다.

내 경우엔 대충 아래와 같은 상황에서 이러한 구조가 필요했는데,
1. 사용자는 Coupon API를 통해서 쿠폰 선물을 보낸다.
2. 쿠폰 선물 보내기가 완료되면 메시지를 하나 생산하는데 Kafka로 쿠폰 선물을 보낼 수신자의 이메일 주소와 메일 제목, 메일 내용을 보낸다.
3. 컨슈머에서는 2번에서 생산한 메시지를 소비하여 다른 서비스에 있는 Mail API를 호출한다. (REST API)
4. 3번의 Mail API는 메일의 발신 성공 여부를 Response로 내려주고 있다. (0: 성공, 1: 실패)
5. 3번의 과정에서 컨슈머는 Mail API를 호출한 뒤, Response를 받을 수 있지만 기존의 KafkaTemplate로는 이 Response를 Coupon API(프로듀서)쪽으로 다시 돌려줄 수가 없다.

그래서 [Spring for Apache Kafka 공식 Docs](https://docs.spring.io/spring-kafka/reference/html/)를 찾아보니 딱 내가 필요로 하던 Template가 구현되어 있었다. 그것이 바로 `ReplyingKafkaTemplate`이다.

간단하게 설명하자면 기존의 컨슈머에서는 메시지를 "소비"하는 것으로 역할이 끝나는데 `ReplyingKafkaTemplate`를 사용하게 되면 "소비"하고 난 뒤에 다시 메시지를 "생산"하는 역할까지 겸하게 된다. 즉, 컨슈머이면서 프로듀서의 역할까지 맡게 되는 것이다. (구분하기 쉽게 프로듀서, 컨슈머를 나눠서 계속해서 얘기하고 있지만 하나의 프로젝트 or API가 프로듀서와 컨슈머로서의 역할을 동시에 가지고 있어도 상관은 없다.)

> 아래 예제 설명이나 코드들은 공식 Docs를 기반으로 설명하고 있기 때문에 실제 소스코드는 공식 Docs의 ReplyingKafkaTemplate 부분을 참고하는게 좋고 아래 내용들은 공식 Docs를 보면서 같이 참고하는 정도로 보면 좋을 듯 하다.

# Consumer 설정

기본적인 컨슈머로서의 설정을 이미 다 되어있다고 가정하고 `ReplyingKafkaTemplate`를 사용하기 위해 필요한 추가 설정들만 다뤄보도록 하겠다.

## ConcurrentKafkaListenerContainerFactory<?, ?> 추가

이미 기존에 컨슈머를 사용 중이라면 `ConcurrentKafkaListenerContainerFactory<?, ?>`로 만들어진 Bean이 하나씩은 설정되어 있을 것이다. `ConcurrentKafkaListenerContainerFactory`가 상속받고 있는 `AbstractKafkaListenerContainerFactory`를 보면 `setReplyTemplate`라는게 있다. 이것을 통해 replyTemplate를 설정해줄 수 있다. 그래서 나는 기존에 쓰던 `ConcurrentKafkaListenerContainerFactory`외에 replyTemplate를 set한 `ConcurrentKafkaListenerContainerFactory`를 하나 더 만들어줬다.

    @Bean(name = "replyKafkaListenerContainerFactory")  
    public ConcurrentKafkaListenerContainerFactory<String, String> replyKafkaListenerContainerFactory(KafkaTemplate<String, String> defaultKafkaTemplate) {  
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();  
        factory.setConsumerFactory(defaultConsumerFactory());  
        factory.setConcurrency(1);  
        factory.setErrorHandler(defaultConsumerErrorHandler);  
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.BATCH);  
      
        factory.setReplyTemplate(defaultKafkaTemplate);  
        //factory.setReplyHeadersConfigurer((k, v) -> k.equals("cat"));  
      
        return factory;  
    }

이미 기존에 `ConcurrentKafkaListenerContainerFactory`가 하나 있기 때문에 `@Bean(name = "replyKafkaListenerContainerFactory") `을 통해 별도로 Bean Name을 지정해줬다.

`factory.setReplyHeadersConfigurer((k, v) -> k.equals("cat"));`는 주석 처리가 되어 있는데... 공식 Docs를 봐도 해당 부분 소스를 봐도 아직 무엇을 위한 설정인지 이해를 못해서-_-... 우선 주석 처리를 해두고 좀 더 찾아보는 중이다. 메시지를 쏴줄 때 헤더에 특정 값을 설정해줄 수 있는데 그때 쓰는거 같긴한데... 더 찾아보고 알게 되면 추가해둘 예정이다.

다른 부분들은 기존의 컨슈머 설정과 동일하고 `setReplyTemplate`부분이 새로 추가되었다. 그럼 여기서 set되는 `defaultKafkaTemplate`는 어디에 있을까?

## Reply 메시지를 생산하기 위한 Producer 설정 추가

    @Bean  
    public ProducerFactory<String, String> defaultProducerFactory() {  
        Map<String, Object> props = new HashMap<>();  
      
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);  
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33_554_432);  
        props.put(ProducerConfig.ACKS_CONFIG, "1");  
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16_384);  
        props.put(ProducerConfig.LINGER_MS_CONFIG, 50);  
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");  
        props.put(ProducerConfig.RETRIES_CONFIG, 0);  
      
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);  
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);  
        
        return new DefaultKafkaProducerFactory<>(props, new StringSerializer()  
            , new JsonSerializer<String>().noTypeInfo());  
    }  
      
    @Bean(name = "defaultKafkaTemplate")  
    public KafkaTemplate<String, String> defaultKafkaTemplate(ProducerFactory<String, String> defaultProducerFactory) {  
      
        return new KafkaTemplate<>(defaultProducerFactory);  
    }

원래 컨슈머 역할을 하는 프로젝트에는 위와 같은 ProducerFactory 관련 설정이 아예 없었다. 이번에 `ReplyingKafkaTemplate`를 추가하면서 Producer 관련 설정도 필요한 것을 알게 되었고 이에 맞춰서 추가한 것이다.

다른 프로젝트에 있는 프로듀서 쪽 설정과 동일하게 맞춰줬다. 아까 위쪽의 컨슈머 설정에서 `setReplyTemplate`에 set되는 `defaultKafkaTemplate`가 여기서 설정되는 것이다. 이러한 Config 기본 설정들은 Spring for Apache Kafka 공식 Docs에 너무 잘 나와있기 때문에 자세한 설명은 생략하겠다.

## 메시지를 소비하는 메소드

    @KafkaListener(topics = "SEND_MAIL"  
        , groupId = "SEND_MAIL_GROUP"  
        , containerFactory = "replyKafkaListenerContainerFactory")  
    @SendTo("SEND_MAIL_REPLY")  
    public String sendMail(@Header(KafkaHeaders.RECEIVED_TOPIC) String topic  
        , @Header(value = KafkaHeaders.RECEIVED_MESSAGE_KEY, required = false) String messageKey  
        , @Header(KafkaHeaders.RECEIVED_PARTITION_ID) String partitionId  
        , @Header(KafkaHeaders.OFFSET) String offset  
        , @Header(KafkaHeaders.RECEIVED_TIMESTAMP) String timestamp  
        , @Header(KafkaHeaders.GROUP_ID) String groupId  
        , @Header(KafkaHeaders.REPLY_TOPIC) String replyTopic  
        , @Payload String msg) throws Exception {  
      
        RestTemplate restTemplate = new RestTemplate();  
      
        List<MediaType> acceptableMediaTypes = new ArrayList<>();  
        acceptableMediaTypes.add(MediaType.APPLICATION_JSON);  
      
        HttpHeaders httpHeaders = new HttpHeaders();  
        httpHeaders.setAccept(acceptableMediaTypes);  
        httpHeaders.setContentType(MediaType.APPLICATION_JSON);  
      
        JsonObject msgJsonObject = new Gson().fromJson(msg, JsonObject.class);  
        JsonObject reqJsonObject = new JsonObject();  
        reqJsonObject.add("params", msgJsonObject);  
      
        HttpEntity<String> httpEntity = new HttpEntity<>(reqJsonObject.toString(), httpHeaders);  
        ResponseEntity<String> responseEntity = restTemplate.exchange("www.mail-api.com/mail/send", HttpMethod.POST, httpEntity, String.class);  
      
        return responseEntity.getBody().toString();  
    }

간단한 REST API 호출용 메소드이다. `SEND_MAIL`이라는 Topic으로 들어온 메시지를 소비하는 메소드인데 못보던 어노테이션이 하나 붙어있다. 바로 `@SendTo`이다. 이것을 통해 메시지를 생산해줄 수 있는 것이다.

대략적으로 `sendMail` 메소드를 설명하자면, `SEND_MAIL` Topic의 메시지를 받아서 그것을 JsonObject로 변환해준 뒤, `www.mail-api.com/mail/send`라는 주소로 RestTemplate를 이용하여 POST 호출을 해주고 있다.
그 뒤, API 호출 결과가 ReponseEntity의 Body에 들어오게 되는데 그것을 바로 return하고 있다. 일반적인 `@KafkaListener`는 아무것도 return하지 않는 void를 가지게 되는데 `ReplyingKafkaTemplate`를 이용하여 동작하는 것은 Return Type을 가지게 된다. `containerFactory = "replyKafkaListenerContainerFactory"`에 설정된 `replyKafkaListenerContainerFactory`는 아까 위쪽에서 컨슈머 설정할 때 추가해준 `setReplyTemplate`를 해줬던 Factory이다.

그래서 위 메소드가 끝나면 ResponseEntity의 Body String이 return이 되는데 `@SendTo`에 설정된 `SEND_MAIL_REPLY`라는 Topic으로 메시지가 생산되게 된다.

그러면 최초로 메시지를 생산한 프로듀서 쪽에서는 어떤 설정을 해줘야할까?


# Producer 설정

## ReplyingKafkaTemplate 관련 설정 추가

    @Bean(name = "replyKafkaTemplate")  
    public ReplyingKafkaTemplate<String, String, String> replyKafkaTemplate(ProducerFactory<String, String> defaultProducerFactory, ConcurrentMessageListenerContainer<String, String> replyContainer) {  
      
        return new ReplyingKafkaTemplate<>(defaultProducerFactory, replyContainer);  
    }  
      
    @Bean  
    public ConcurrentMessageListenerContainer<String, String> replyContainer(ConcurrentKafkaListenerContainerFactory<String, String> replyKafkaListenerContainerFactory) {  
      
        ConcurrentMessageListenerContainer<String, String> replyContainer = replyKafkaListenerContainerFactory.createContainer("SEND_MAIL_REPLY");  
        replyContainer.getContainerProperties().setGroupId("REPLY_CONSUME_GROUP");  
        replyContainer.setAutoStartup(false);  
        replyContainer.setBatchErrorHandler(new SeekToCurrentBatchErrorHandler());  
      
        return replyContainer;  
    }

완전 진짜 간소하게 축소한 코드이다. 보통의 KafkaTemplate는 `KafkaTemplate<?, ?>`라는 타입으로 Bean을 생성해주는데 ReplyingKafkaTemplate은 `ReplyingKafkaTemplate<?, ?, ?>`로 별도로 생성해주었다.

더불어 `ConcurrentMessageListenerContainer<?, ?>`를 인자값으로 받고 있는데 여기에 Reply Topic에 대한 설정, Reply Group에 대한 설정을 할 수 있다.

`replyKafkaListenerContainerFactory.createContainer`에 있는 `SEND_MAIL_REPLY`를 혹시 기억하는가? 아까 위쪽에서 `@SendTo`에 설정한 Topic명과 같다. 여기엔 하나가 아닌, 여러 Reply Topic명을 더 추가할 수 있다. (`String... topic`이 파라미터 타입이라 가변인자값으로 복수개의 Topic을 받을 수 있다.)

그리고 바로 아래에서 `setGroupId`를 통해 Reply Topic명으로 생성된 메시지를 소비할 Group명을 세팅해줄 수 있다.

그런데 `replyContainer`가 파라미터로 받는 `ConcurrentKafkaListenerContainerFactory`는 어디에 있을까? 뭔가 낯이 익지 않는가?

바로 아까 컨슈머 쪽에서도 같은 설정을 해준 적이 있다. 

## ConcurrentKafkaListenerContainerFactory<?, ?> 추가

기존의 프로듀서 역할을 하던 프로젝트에는 컨슈머 관련 설정이 없었다. 아까 위에서 설명했던 컨슈머와는 정반대의 상황이다. (컨슈머 쪽에서는 프로듀서 관련 설정이 없었다.)

    @Bean(name = "replyKafkaListenerContainerFactory")  
    public ConcurrentKafkaListenerContainerFactory<String, String> replyKafkaListenerContainerFactory(KafkaTemplate<String, String> defaultKafkaTemplate) {  
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();  
        factory.setConsumerFactory(defaultConsumerFactory());  
        factory.setConcurrency(1);  
        factory.setErrorHandler(defaultConsumerErrorHandler);  
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.BATCH);  
      
        factory.setReplyTemplate(defaultKafkaTemplate);  
        //factory.setReplyHeadersConfigurer((k, v) -> k.equals("cat"));  
      
        return factory;  
    }

컨슈머 쪽과 똑같이 추가해주면 된다. 그러면 바로 위쪽에서 `replyContainer`가 파라미터로 받는 `ConcurrentKafkaListenerContainerFactory`를 이 Bean 객체로 받을 수 있는 것이다. 그러면 프로듀서 쪽에서도 ReplyingKafkaTemplate를 쓰기 위한 준비가 끝난 것이다.

# 실제 ReplyingKafkaTemplate를 이용한 메시지 생산

## ReplyingProducer.java

    @Service  
    public class ReplyingProducer {  
      
      public ReplyingProducer(ReplyingKafkaTemplate<String, String, String> replyingKafkaTemplate) {  
        this.replyingKafkaTemplate = replyingKafkaTemplate;  
      }  
      
      private final ReplyingKafkaTemplate<String, String, String> replyingKafkaTemplate;  
      
     private String sendMessage(final String topic, final String replyTopic, final String key, final String msg) throws Exception {  
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, msg);  
      
        record.headers().add(new RecordHeader(KafkaHeaders.REPLY_TOPIC, replyTopic.getBytes()));  
      
        RequestReplyFuture<String, String, String> replyFuture = replyingKafkaTemplate.sendAndReceive(record);  
      
        SendResult<String, String> sendResult = replyFuture.getSendFuture().get(10, TimeUnit.SECONDS);  
      
        ConsumerRecord<String, String> consumerRecord = replyFuture.get(10, TimeUnit.SECONDS);  
      
        return consumerRecord.value();  
      }  
    }

일반적인 KafkaTemplate는 send라는 메소드로 메시지를 보내게 되는데 ReplyingKafkaTemplate는 `sendAndReceive`라는 메소드로 보내게 된다.

`RequestReplyFuture`를 return하게 되는데 그 안에 있는 `ConsumerRecord` 객체 안의 `value()`가 아까 `@SendTo`가 달려있던 `@KafkaListener`에서 return한 ResponseEntity의 Body String값이다.


아직 Kafka에 대한 이해도가 높지 않은 까닭도 있고 시행착오를 겪으며 설정한 코드들을 통해 남에게 설명한다는 점이 어렵기도 해서 제대로 된 설명이 되었는지 잘 모르겠다. 사실 이 글만 봐서는 완벽하게 이해가 되지 않을 것이고 애초에 Spring-Kafka에 대한 어느정도 이해도가 있는 사람, 그리고 Spring for Apache Kafka 공식 Docs를 보면서 참고하라고 쓰는 글이기 때문에 완전 아무것도 모르는 초보자가 보기엔 이해가 힘든 글일 수 있으니 이점은 양해바란다.

나도 아직은 메시지를 생산하고 소비한 뒤 다시 되돌려받는 과정까지만 해본 상태이고 더 깊은 단계까진 가보지 않아서 이후에 좀 더 흥미로운 부분을 발견하거나 관련 세팅, Config들을 알게 되면 내용을 더 추가해보도록 하겠다.
