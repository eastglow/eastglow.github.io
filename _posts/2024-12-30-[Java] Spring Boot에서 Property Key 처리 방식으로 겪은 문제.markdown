---
layout: post
title:  "[Java] Spring Boot에서 Property Key 처리 방식으로 겪은 문제"
date:   2024-12-30
author: EastGlow
categories: Back-end
---

# 1. 들어가며

Spring Boot와 캐시는 뗼레야 뗼 수 없는 사이인데 그 중에서도 회사에서는 Redis를 메인으로 쓰고 있다. 그러다보니 Redis와 관련된 커스터마이징이 많은 편이고 그것을 별도의 Auto Configuration으로 만들어서 설정해둔 application yml의 값에 따라 필요한 부분만 자동으로 초기화되도록 해놓고 쓰고 있다.

대표적으로 각 캐시키 별로 설정하는 TTL 세팅 로직이 그것인데 이 TTL 세팅 로직을 처음 만들 때 있었던 문제를 하나 써보려고 한다.

# 2. Property Key

application yml(or properties)에는 애플리케이션과 관련된 수많은 property가 있는데 보통은 key명을 my-name이라거나 name과 같이 하이픈 혹은 단일 단어로 쓰곤 했다.

그런데 Redis TTL 관련된 커스터마이징 로직을 만들다보니 우리팀의 캐시키 규칙은 "계층별 구분은 콜론(:)으로 하자"였고 그러다보니 TTL의 Property 조합은 `캐시키: Duration`과 같이 되었다.

`my:key: 60s`

위 예시에서는 캐시키가 `my:key`, TTL은 `60s`이 되는 것이다.

그럼 저것을 받아주는 Java Properties Class에서는 어떻게 해줬을까? 간단하게 `Map<String, Duration>` 형태로 받아주면 됐다.

## application.yml
```
test:  
  data:  
    my:key: 60s
```

## TestProperties.java
```
@Getter  
@Setter  
@ConfigurationProperties(prefix = "test")  
public class TestProperties {  
  
    private Map<String, Duration> data = new HashMap<>();  
}
```

이때까지만 해도 그냥 잘 작동할 줄 알았다. 그런데 막상 다 해놓고 앱을 띄워보니 이상하게 `my:key`이 아니라 `mykey`로 property가 초기화되는 것이었다. 실제로 디버깅을 해봐도 TestProperties엔 `mykey`가 들어와있었다.

돌이켜보면 application yml에서는 계층 구분을 콜론(:)으로 하고 있었기 때문에 이런 특수문자는 Property의 이름으로 쓸 수 없을거란 생각을 해볼 수 있었을텐데 이때 당시엔 TTL 세팅이 계속 안 되길래 다른데서 자꾸 원인을 찾고 있었다.

![](/assets/post/20241230_3.png)

이렇듯 한참이나 다른데서 삽질을 하고나서야 TestProperties에 값 자체가 애초에 잘못 세팅되고 있다는 것을 발견하게 되었고 이곳저곳 찾다보니 도움이 되는 글을 여럿 볼 수 있었다.

## 참고

> https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config.typesafe-configuration-properties.relaxed-binding
> https://artofcode.wordpress.com/2020/10/29/special-characters-in-a-key-in-spring-boot-yaml-file/
> https://stackoverflow.com/questions/51289856/spring-application-properties-ignoring-slashes-in-strings  
> https://github.com/spring-projects/spring-boot/issues/11386  

# 3. 그럼 어떻게 써야할까?

나처럼 굳이 써야겠다면 방법은 있었다. 바로 `'[]'`로 묶어주면 되는 것이었다.

```
test:
  data:
    '[my:key]': 60s
```
![](/assets/post/20241230_4.png)

위처럼 수정해주고 나면 제대로 설정된 키값으로 들어온 것을 볼 수 있었다.
