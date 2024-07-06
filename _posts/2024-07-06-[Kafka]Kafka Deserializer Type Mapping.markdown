---
layout: post
title:  "[Kafka]Kafka Deserializer Type Mapping"
date:   2024-07-06
author: EastGlow
categories: Kafka
---

# 1. 들어가며

특정 카프카 토픽을 팀 내부에서만 사용하는 경우엔 보통 이슈가 없지만 타 팀과 공유해서 같이 사용하는 토픽의 경우 Produce하는 주체에서 생성하는 메시지 타입과 Consume하는 타 팀에서 설정한 메시지 타입이 서로 다른 경우가 있다.

이경우 Producer에서 생산한 메시지에 들어있는 header엔 아래와 같이 메시지의 타입값이 들어있는데(헤더값이 비어있는 경우도 있음) 이 타입과 Consumer의 타입이 서로 불일치하면 ClassCastException 에러가 발생하게 된다.

![](/assets/post/20240706_1.png)

물론 위와 같이 메시지의 헤더에 타입 정보가 포함되는 경우는 JsonSerializer로 메시지를 생산했을 때이고, 그 안에서도 ADD_TYPE_INFO_HEADERS 값을 false로 주지 않고 기본값으로 쓸 때의 얘기이다.

```
public JsonSerializer(ObjectMapper objectMapper) {
    this.addTypeInfo = true;
    this.typeMapper = new DefaultJackson2JavaTypeMapper();
    this.typeMapperExplicitlySet = false;
    Assert.notNull(objectMapper, "'objectMapper' must not be null.");
    this.objectMapper = objectMapper;
}
```

JsonSerializer의 생성자 내 addTypeInfo 기본값은 true로 되어 있어서 생산된 메시지의 헤더엔 타입 정보가 기본적으로 포함되게 된다.

## 1) 클래스명이 다를 때

Producer와 Consumer의 Dto 클래스명이 서로 다를 때 ClassCastException이 발생한다.

me.eastglow.dto.TestReqDto  
me.eastglow.dto.OtherReqDto

```
org.springframework.kafka.listener.ListenerExecutionFailedException: Listener method 'public void me.eastglow.consumer.TestConsumer.consumeTest(org.apache.kafka.clients.consumer.ConsumerRecord<java.lang.String, me.eastglow.dto.OtherReqDto>,org.springframework.kafka.listener.adapter.ConsumerRecordMetadata) throws java.lang.Exception' threw exception; nested exception is java.lang.ClassCastException: class me.eastglow.dto.TestReqDto cannot be cast to class me.eastglow.dto.OtherReqDto (me.eastglow.dto.TestReqDto and me.eastglow.dto.OtherReqDto are in unnamed module of loader 'app'); nested exception is java.lang.ClassCastException: class me.eastglow.dto.TestReqDto cannot be cast to class me.eastglow.dto.OtherReqDto (me.eastglow.dto.TestReqDto and me.eastglow.dto.OtherReqDto are in unnamed module of loader 'app')
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.decorateException(KafkaMessageListenerContainer.java:2093) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeOnMessage(KafkaMessageListenerContainer.java:2064) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeOnMessage(KafkaMessageListenerContainer.java:2025) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeRecordListener(KafkaMessageListenerContainer.java:1962) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeWithRecords(KafkaMessageListenerContainer.java:1902) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeRecordListener(KafkaMessageListenerContainer.java:1790) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeListener(KafkaMessageListenerContainer.java:1505) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.pollAndInvoke(KafkaMessageListenerContainer.java:1164) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.run(KafkaMessageListenerContainer.java:1059) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:264) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run(FutureTask.java) ~[na:na]
    at java.base/java.lang.Thread.run(Thread.java:829) ~[na:na]
Caused by: java.lang.ClassCastException: class me.eastglow.dto.TestReqDto cannot be cast to class me.eastglow.dto.OtherReqDto (me.eastglow.dto.TestReqDto and me.eastglow.dto.OtherReqDto are in unnamed module of loader 'app')
    at me.eastglow.consumer.TestConsumer.consumeTest(TestConsumer.java:69) ~[classes/:na]
    at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
    at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:na]
    at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
    at java.base/java.lang.reflect.Method.invoke(Method.java:566) ~[na:na]
    at org.springframework.messaging.handler.invocation.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:171) ~[spring-messaging-5.2.15.RELEASE.jar:5.2.15.RELEASE]
    at org.springframework.messaging.handler.invocation.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:120) ~[spring-messaging-5.2.15.RELEASE.jar:5.2.15.RELEASE]
    at org.springframework.kafka.listener.adapter.HandlerAdapter.invoke(HandlerAdapter.java:48) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.adapter.MessagingMessageListenerAdapter.invokeHandler(MessagingMessageListenerAdapter.java:325) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.adapter.RecordMessagingMessageListenerAdapter.onMessage(RecordMessagingMessageListenerAdapter.java:87) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.adapter.RecordMessagingMessageListenerAdapter.onMessage(RecordMessagingMessageListenerAdapter.java:52) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeOnMessage(KafkaMessageListenerContainer.java:2044) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    ... 11 common frames omitted
```

## 2) 클래스 경로가 다를 때

단순히 클래스명이 다른것뿐만 아니라 클래스 경로까지 모두 일치해야한다.

당연한 소리이지만 클래스 경로가 달라도 ClassCastException이 발생한다.

me.eastglow.dto.TestReqDto  
my.eastglow.dto.TestReqDto

```
org.springframework.kafka.listener.ListenerExecutionFailedException: Listener method 'public void me.eastglow.consumer.TestConsumer.consumeTest(org.apache.kafka.clients.consumer.ConsumerRecord<java.lang.String, my.eastglow.dto.TestReqDto>,org.springframework.kafka.listener.adapter.ConsumerRecordMetadata) throws java.lang.Exception' threw exception; nested exception is java.lang.ClassCastException: class me.eastglow.dto.TestReqDto cannot be cast to class my.eastglow.dto.TestReqDto (me.eastglow.dto.TestReqDto and my.eastglow.dto.TestReqDto are in unnamed module of loader 'app'); nested exception is java.lang.ClassCastException: class me.eastglow.dto.TestReqDto cannot be cast to class my.eastglow.dto.TestReqDto (me.eastglow.dto.TestReqDto and my.eastglow.dto.TestReqDto are in unnamed module of loader 'app')
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.decorateException(KafkaMessageListenerContainer.java:2093) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeOnMessage(KafkaMessageListenerContainer.java:2064) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeOnMessage(KafkaMessageListenerContainer.java:2025) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeRecordListener(KafkaMessageListenerContainer.java:1962) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeWithRecords(KafkaMessageListenerContainer.java:1902) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeRecordListener(KafkaMessageListenerContainer.java:1790) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeListener(KafkaMessageListenerContainer.java:1505) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.pollAndInvoke(KafkaMessageListenerContainer.java:1164) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.run(KafkaMessageListenerContainer.java:1059) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:264) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run(FutureTask.java) ~[na:na]
    at java.base/java.lang.Thread.run(Thread.java:829) ~[na:na]
Caused by: java.lang.ClassCastException: class me.eastglow.dto.TestReqDto cannot be cast to class my.eastglow.dto.TestReqDto (me.eastglow.dto.TestReqDto and my.eastglow.dto.TestReqDto are in unnamed module of loader 'app')
    at me.eastglow.consumer.TestConsumer.consumeTest(TestConsumer.java:63) ~[classes/:na]
    at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
    at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:na]
    at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
    at java.base/java.lang.reflect.Method.invoke(Method.java:566) ~[na:na]
    at org.springframework.messaging.handler.invocation.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:171) ~[spring-messaging-5.2.15.RELEASE.jar:5.2.15.RELEASE]
    at org.springframework.messaging.handler.invocation.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:120) ~[spring-messaging-5.2.15.RELEASE.jar:5.2.15.RELEASE]
    at org.springframework.kafka.listener.adapter.HandlerAdapter.invoke(HandlerAdapter.java:48) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.adapter.MessagingMessageListenerAdapter.invokeHandler(MessagingMessageListenerAdapter.java:325) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.adapter.RecordMessagingMessageListenerAdapter.onMessage(RecordMessagingMessageListenerAdapter.java:87) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.adapter.RecordMessagingMessageListenerAdapter.onMessage(RecordMessagingMessageListenerAdapter.java:52) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeOnMessage(KafkaMessageListenerContainer.java:2044) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    ... 11 common frames omitted
```

_**결과적으로는 클래스명이든 클래스 경로든 뭔가 하나라도 다르면 ClassCastException이 발생함**_

# 2. 관련된 옵션들을 알아보자

## 1) JsonDeserializer.TRUSTED_PACKAGES

이 옵션은 Deserialize를 할 때, 받은 메시지의 타입을 신뢰할 수 있는 패키지에 속해있는지 체크할 때 영향을 주는 옵션이다.

```
org.springframework.kafka.listener.ListenerExecutionFailedException: Listener failed; nested exception is org.springframework.kafka.support.serializer.DeserializationException: failed to deserialize; nested exception is java.lang.IllegalArgumentException: The class 'me.eastglow.dto.TestReqDto' is not in the trusted packages: [java.util, java.lang]. If you believe this class is safe to deserialize, please provide its name. If the serialization is only done by a trusted source, you can also enable trust all (*).
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.decorateException(KafkaMessageListenerContainer.java:2096) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.checkDeser(KafkaMessageListenerContainer.java:2107) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeOnMessage(KafkaMessageListenerContainer.java:2012) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeRecordListener(KafkaMessageListenerContainer.java:1962) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeWithRecords(KafkaMessageListenerContainer.java:1902) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeRecordListener(KafkaMessageListenerContainer.java:1790) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeListener(KafkaMessageListenerContainer.java:1505) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.pollAndInvoke(KafkaMessageListenerContainer.java:1164) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.run(KafkaMessageListenerContainer.java:1059) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:264) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run(FutureTask.java) ~[na:na]
    at java.base/java.lang.Thread.run(Thread.java:829) ~[na:na]
Caused by: org.springframework.kafka.support.serializer.DeserializationException: failed to deserialize; nested exception is java.lang.IllegalArgumentException: The class 'me.eastglow.dto.TestReqDto' is not in the trusted packages: [java.util, java.lang]. If you believe this class is safe to deserialize, please provide its name. If the serialization is only done by a trusted source, you can also enable trust all (*).
    at org.springframework.kafka.support.serializer.ErrorHandlingDeserializer.deserializationException(ErrorHandlingDeserializer.java:216) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.support.serializer.ErrorHandlingDeserializer.deserialize(ErrorHandlingDeserializer.java:191) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.apache.kafka.clients.consumer.internals.Fetcher.parseRecord(Fetcher.java:1324) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher.access$3400(Fetcher.java:129) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher$CompletedFetch.fetchRecords(Fetcher.java:1555) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher$CompletedFetch.access$1700(Fetcher.java:1391) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher.fetchRecords(Fetcher.java:683) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher.fetchedRecords(Fetcher.java:634) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.KafkaConsumer.pollForFetches(KafkaConsumer.java:1289) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:1243) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:1216) ~[kafka-clients-2.5.1.jar:na]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doPoll(KafkaMessageListenerContainer.java:1246) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.pollAndInvoke(KafkaMessageListenerContainer.java:1146) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    ... 5 common frames omitted
Caused by: java.lang.IllegalArgumentException: The class 'me.eastglow.dto.TestReqDto' is not in the trusted packages: [java.util, java.lang]. If you believe this class is safe to deserialize, please provide its name. If the serialization is only done by a trusted source, you can also enable trust all (*).
    at org.springframework.kafka.support.converter.DefaultJackson2JavaTypeMapper.getClassIdType(DefaultJackson2JavaTypeMapper.java:126) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.support.converter.DefaultJackson2JavaTypeMapper.toJavaType(DefaultJackson2JavaTypeMapper.java:100) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.support.serializer.JsonDeserializer.deserialize(JsonDeserializer.java:472) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.support.serializer.ErrorHandlingDeserializer.deserialize(ErrorHandlingDeserializer.java:188) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    ... 16 common frames omitted
```

만약 저 옵션을 `JsonDeserializer.TRUSTED_PACKAGES, "*"` 와 같이 설정하지 않고 사용하면 위와 같은 에러가 발생하게 된다. 기본적으로 신뢰할 수 있는 패키지에 속해있는 class만 deserialize할 수 있기 때문이다.

> 참고:  [https://www.baeldung.com/spring-kafka-trusted-packages-feature](https://www.baeldung.com/spring-kafka-trusted-packages-feature)

메시지의 일관성 유지와 보안성을 지키기 위한 수단으로 저 옵션이 켜져있는게 좋다고 하는데 사내 카프카는 내부망에서만 접근할 수 있고 신뢰할 수 있는 패키지를 지속적으로 관리하는게 쉽지 않기 때문에 다들 기본적으로는 `"*"`와 같이 설정하여 전체 패키지를 신뢰할 수 있는 패키지로 두고 쓰는 듯 하다.

```
props.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
```

물론 아래와 같이 설정하면 특정 패키지만 신뢰할 수 있는 패키지로 설정하여 메시지를 정상적으로 받을 수도 있다.

```
props.put(JsonDeserializer.TRUSTED_PACKAGES, "me.eastglow.dto");
```

참고로 아래 경로의 패키지는 기본적으로 신뢰할 수 있는 패키지에 포함되어 있다. (Map, String 등)

```
private static final List<String> TRUSTED_PACKAGES = Arrays.asList("java.util", "java.lang");
```

## 2) JsonDeserializer.USE_TYPE_INFO_HEADERS

역질렬화를 할 때 헤더에 있는 타입 정보를 쓸 건지 말 건지를 정하는 옵션이다. 기본적으로는 true로 되어있다.

```
this.typeMapper.setTypePrecedence(this.useTypeHeaders ? TypePrecedence.TYPE_ID : TypePrecedence.INFERRED);
```

위와 같이 useTypeHaeders의 값에 따라 헤더의 TYPE_ID를 볼건지, 아니면 타입값을 추론해서 볼건지 정해진다.

- 메시지 헤더 타입정보를 보지 않는다는건 기본 매핑 클래스(VALUE_DEFAULT_TYPE)를 본다고 생각하면 된다. 그래서 이 기본 매핑 클래스가 없으면 아예 역직렬화를 할 수 없게 된다. (자세한 내용은 아래에서 후술)

```
public T deserialize(String topic, Headers headers, byte[] data) {
    if (data == null) {
        return null;
    } else {
        ObjectReader deserReader = null;
        JavaType javaType = null;
        if (this.typeResolver != null) {
            javaType = this.typeResolver.resolveType(topic, data, headers);
        }
 
        if (javaType == null && this.typeMapper.getTypePrecedence().equals(TypePrecedence.TYPE_ID)) {
            javaType = this.typeMapper.toJavaType(headers);
        }
 
        if (javaType != null) {
            deserReader = this.objectMapper.readerFor(javaType);
        }
 
        if (this.removeTypeHeaders) {
            this.typeMapper.removeHeaders(headers);
        }
 
        if (deserReader == null) {
            deserReader = this.reader;
        }
 
        Assert.state(deserReader != null, "No type information in headers and no default type provided");
 
        try {
            return deserReader.readValue(data);
        } catch (IOException var7) {
            IOException e = var7;
            throw new SerializationException("Can't deserialize data [" + Arrays.toString(data) + "] from topic [" + topic + "]", e);
        }
    }
}
```

위 Deserialize 메서드에서 보면 JavaType이라는 객체를 통해서 실제로 메시지를 어떤 클래스에 매핑할 것인지가 정해지는데 JavaType은 몇개의 if문을 통해 결정이 된다.

### 2-1) typeResolver

VALUE_TYPE_METHOD에 설정한 메서드를 통해 후에 나올 JavaType을 resolve할 resolver를 설정하는 구간이다. 뭔가 커스텀하게 특정 필드는 새롭게 조합해서 매핑하고 다른 필드는 그대로 쓰고 이런식으로 커스텀하게 매핑이 필요할 때 쓰는 듯 하다.

```
private void setUpTypeMethod(Map<String, ?> configs, boolean isKey) {
    if (isKey && configs.containsKey("spring.json.key.type.method")) {
        this.setUpTypeResolver((String)configs.get("spring.json.key.type.method"));
    } else if (!isKey && configs.containsKey("spring.json.value.type.method")) {
        this.setUpTypeResolver((String)configs.get("spring.json.value.type.method"));
    }
 
}
 
private void setUpTypeResolver(String method) {
    try {
        this.typeResolver = this.buildTypeResolver(method);
    } catch (IllegalStateException var3) {
        IllegalStateException e = var3;
        if (e.getCause() instanceof NoSuchMethodException) {
            this.typeResolver = (topic, data, headers) -> {
                return (JavaType)SerializationUtils.propertyToMethodInvokingFunction(method, byte[].class, this.getClass().getClassLoader()).apply(data, headers);
            };
        } else {
            throw e;
        }
    }
}
```

### 2-2) TYPE_ID

USE_TYPE_INFO_HEADERS를 별도로 false 설정하지 않았다면 헤더엔 TYPE_ID가 들어있을텐데(물론 Prodcuer 쪽에도 타입 정보 포함 설정을 했을 경우) 이 TYPE_ID를 가지고 매핑될 클래스를 찾아준다.

2-1)에서 기본 매핑될 클래스를 찾았더라도 2-2)에서 TYPE_ID로 매핑되는 다른 클래스가 있다면 이것을 우선으로 보게 된다. 예를 들어 VALUE_DEFAULT_TYPE를 MyDto로 해뒀어도 메시지의 헤더 타입 정보에 OtherDto로 들어있으면 최종적으로 메시지 값이 매핑될 클래스는 OtherDto가 된다.

```
if (javaType == null && this.typeMapper.getTypePrecedence().equals(TypePrecedence.TYPE_ID)) {
    javaType = this.typeMapper.toJavaType(headers);
}
```

### 2-3) deserReader == null

그렇게 2-1), 2-2) 과정을 거쳐서 JavaType이 null이 아닌 객체가 되었다면 objectMapper를 통해 ObjectReader를 찾아냈겠지만 typeResolver도 없고 메시지 헤더의 TYPE_ID도 없는 경우엔 기본 설정된 매핑 클래스를 찾아가게 된다.

```
private void setupTarget(Map<String, ?> configs, boolean isKey) {
    try {
        JavaType javaType = null;
        if (isKey && configs.containsKey("spring.json.key.default.type")) {
            javaType = this.setupTargetType(configs, "spring.json.key.default.type");
        } else if (!isKey && configs.containsKey("spring.json.value.default.type")) {
            javaType = this.setupTargetType(configs, "spring.json.value.default.type");
        }
 
        if (javaType != null) {
            this.initialize(javaType, TypePrecedence.TYPE_ID.equals(this.typeMapper.getTypePrecedence()));
        }
 
    } catch (LinkageError | ClassNotFoundException var4) {
        Throwable e = var4;
        throw new IllegalStateException(e);
    }
}
```

VALUE_DEFAULT_TYPE로 설정해둔 클래스가 메시지를 읽어올 기본 ObjectReader 매핑 클래스가 된다.

```
private void initialize(@Nullable JavaType type, boolean useHeadersIfPresent) {
    this.targetType = type;
    Assert.isTrue(this.targetType != null || useHeadersIfPresent, "'targetType' cannot be null if 'useHeadersIfPresent' is false");
    if (this.targetType != null) {
        this.reader = this.objectMapper.readerFor(this.targetType);
    }
 
    this.addTargetPackageToTrusted();
    this.typeMapper.setTypePrecedence(useHeadersIfPresent ? TypePrecedence.TYPE_ID : TypePrecedence.INFERRED);
}
```

그래서 만약에라도 typeResolver도 없고 메시지 헤더의 TYPE_ID도 없어서 JavaType이 null이라도 기본 매핑 클래스가 있다면 위 코드에 의해 기본 ObjectReader가 할당되어 문제없이 메시지를 컨버팅 할 수 있게 된다.

그런데 만약 typeResolver, TYPE_ID, VALUE_DEFAULT_TYPE 셋 다 없으면 `"No type information in headers and no default type provided"` 에러가 발생하면서 정상적으로 메시지를 역직렬화 할 수 없다면서 에러가 발생하게 되는것이다.

USE_TYPE_INFO_HEADERS를 false로 설정하고 기본 매핑 클래스를 지정하지 않고 메시지를 받아보면 아래와 같은 에러를 볼 수 있다.

- false로 설정했기 때문에 메시지 헤더의 타입 정보를 보지 않게 되었고 기본 매핑 클래스도 없기 때문에 컨슈머는 받은 메시지를 어떤 클래스로 컨버팅 해야할 지 알 수 없는 상태

```
org.springframework.kafka.listener.ListenerExecutionFailedException: Listener failed; nested exception is org.springframework.kafka.support.serializer.DeserializationException: failed to deserialize; nested exception is java.lang.IllegalStateException: No type information in headers and no default type provided
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.decorateException(KafkaMessageListenerContainer.java:2096) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.checkDeser(KafkaMessageListenerContainer.java:2107) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeOnMessage(KafkaMessageListenerContainer.java:2012) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeRecordListener(KafkaMessageListenerContainer.java:1962) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doInvokeWithRecords(KafkaMessageListenerContainer.java:1902) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeRecordListener(KafkaMessageListenerContainer.java:1790) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.invokeListener(KafkaMessageListenerContainer.java:1505) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.pollAndInvoke(KafkaMessageListenerContainer.java:1164) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.run(KafkaMessageListenerContainer.java:1059) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515) ~[na:na]
    at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264) ~[na:na]
    at java.base/java.lang.Thread.run(Thread.java:829) ~[na:na]
Caused by: org.springframework.kafka.support.serializer.DeserializationException: failed to deserialize; nested exception is java.lang.IllegalStateException: No type information in headers and no default type provided
    at org.springframework.kafka.support.serializer.ErrorHandlingDeserializer.deserializationException(ErrorHandlingDeserializer.java:216) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.support.serializer.ErrorHandlingDeserializer.deserialize(ErrorHandlingDeserializer.java:191) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.apache.kafka.clients.consumer.internals.Fetcher.parseRecord(Fetcher.java:1324) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher.access$3400(Fetcher.java:129) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher$CompletedFetch.fetchRecords(Fetcher.java:1555) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher$CompletedFetch.access$1700(Fetcher.java:1391) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher.fetchRecords(Fetcher.java:683) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.internals.Fetcher.fetchedRecords(Fetcher.java:634) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.KafkaConsumer.pollForFetches(KafkaConsumer.java:1314) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:1243) ~[kafka-clients-2.5.1.jar:na]
    at org.apache.kafka.clients.consumer.KafkaConsumer.poll(KafkaConsumer.java:1216) ~[kafka-clients-2.5.1.jar:na]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.doPoll(KafkaMessageListenerContainer.java:1246) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.pollAndInvoke(KafkaMessageListenerContainer.java:1146) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    ... 4 common frames omitted
Caused by: java.lang.IllegalStateException: No type information in headers and no default type provided
    at org.springframework.util.Assert.state(Assert.java:76) ~[spring-core-5.2.15.RELEASE.jar:5.2.15.RELEASE]
    at org.springframework.kafka.support.serializer.JsonDeserializer.deserialize(JsonDeserializer.java:483) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    at org.springframework.kafka.support.serializer.ErrorHandlingDeserializer.deserialize(ErrorHandlingDeserializer.java:188) ~[spring-kafka-2.5.14.RELEASE.jar:2.5.14.RELEASE]
    ... 15 common frames omitted
```

# 3. 해결방법

>아래 방법대로 하는게 무조건 정답은 아니고 각자 상황에 따라 옵션을 다르게 줄 수 있으니 참고만 하면 좋을 듯 하다.

## 1) JsonDeserializer.USE_TYPE_INFO_HEADERS는 기본값 그대로 true 상태로 두는게 좋다.

보통 VALUE_DEFAULT_TYPE를 별도로 설정하지 않고 운영하는 경우가 많을텐데 그렇게 되면 메시지 헤더의 타입 정보를 빼버리면 매핑할 클래스 정보를 찾을 구석이 없게 된다.

그래서 어떠한 클래스인지에 대한 메시지 헤더 타입 정보는 바라볼 수 있도록 위 옵션은 default true 값 그대로 두는게 좋다고 생각된다.

## 2) JsonDeserializer.TRUSTED_PACKAGES는 "*"로 설정한다.

신뢰할 수 있는 패키지를 지속적으로 관리할 수 있다면 일관성, 보안성 측면에서 `"*"`로 전체 설정하는 것보다 좋지만 그렇지 않다면 우선은 전체 설정하는게 좋은 듯 하다. 그래야 Producer와 Consumer 간에 메시지 타입이 서로 달라도 역직렬화 과정에서 문제가 없을 것이고 운영하는데에 있어 조금 더 유연하게 대처할 수 있을 거라 생각된다.

## 3) JsonDeserializer.VALUE_DEFAULT_TYPE

이 옵션에 대한 설정은 2가지 방법으로 나눌 수 있을 것 같다.

1. ConsumerFactory에 설정을 일괄로 적용
    ```
    @Bean(name = "jsonConsumerFactory")
    public ConsumerFactory<String, Object> jsonConsumerFactory() {
    
      Map<String, Object> props = this.getDefaultConfig();
    
      props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
      props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
      props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
    
      props.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
      props.put(JsonDeserializer.VALUE_DEFAULT_TYPE, OtherReqDto.class);
    
      return new DefaultKafkaConsumerFactory<>(props);
    }
    ```
    보통 KafkaListener의 containerFacotry를 위와 같이 ConsumerFactory Bean으로 만들어서 지정하여 쓰고 있을텐데 위 코드처럼 VALUE_DEFAULT_TYPE을 특정 클래스로 추가해주면 이 Bean을 참조하는 모든 리스너에 일괄로 적용할 수 있다. 그런데 개인적으로는 일괄 적용되기 때문에 별로 추천하진 않는 방법이다.

    혹시나 메시지 헤더 타입 정보가 비어 있는 경우엔 기본 매핑 클래스로 우선 역직렬화가 시도될 것이기 때문에 만에 하나의 예상치 못한 메시지에 대한 이슈를 방지하기 위한 목적으로는 기본 설정해둬도 나쁘지 않을 것 같다.


2. @KafkaListener 내 properties로 개별 선언
    ```
    @KafkaListener(topics = KafkaConst.Topic.TEST_TOPIC
            , groupId = KafkaConst.Group.TEST_GROUP
            , containerFactory = "jsonKafkaListenerContainerFactory"
            , properties = {
                    JsonDeserializer.VALUE_DEFAULT_TYPE + ":me.eastglow.dto.TestReqDto"
            }
    )
    ```
    - 개별 리스너에 properties를 선언하여 개별 적용할 수 있다. 위와 같이 해두면 TestReqDto가 위 리스너가 컨슘하는 메시지의 기본 매핑 클래스로 설정된다.

그런데 위 2가지 설정 다 한가지 문제가 있다. 바로 공교롭게도 메시지 헤더 타입 정보가 존재하는 경우엔 아무런 쓸모가 없는 설정이라는 것이다. 아까 위에서 `"2-2) TYPE_ID"` 부분을 설명할 때 얘기했지만 매핑될 클래스를 찾을 때 1순위로 메시지 헤더 타입 정보를 보고 2순위로 기본 매핑 클래스를 보기 때문이다.

그래서 만약 무조건 이 카프카 리스너로 들어오는 메시지는 특정 클래스에 매핑되어야 한다! 한다면 아래와 같이 별도 설정을 해주면 된다.

```
@KafkaListener(topics = KafkaConst.Topic.TEST_TOPIC
        , groupId = KafkaConst.Group.TEST_GROUP
        , containerFactory = "jsonKafkaListenerContainerFactory"
        , properties = {
                JsonDeserializer.USE_TYPE_INFO_HEADERS + ":false",
                JsonDeserializer.VALUE_DEFAULT_TYPE + ":me.eastglow.dto.TestReqDto"
        }
)
```

USE_TYPE_INFO_HEADER를 false로 설정하고 VALUE_DEFAULT_TYPE을 별도의 클래스로 지정하면 이 리스너가 컨슘하는 메시지는 헤더의 타입 정보는 무시하고 무조건 TestReqDto로 역직렬화를 시도하게 된다.



이게 싫다면 아래와 같이 설정하면 별도의 추가 설정없이 메시지를 역직렬화 할 수 있게 된다. 경우에 따라서 적절하게 설정하여 운영하면 될 듯 하다.

# 3줄 요약

-   USE_TYPE_INFO_HEADER는 true 그대로 두기
-   TRUSTED_PACKAGES는 `"*"` 전체로 설정
-   Producer의 DTO와 Consumer의 DTO의 경로부터 클래스명까지 모두 일치시키기
