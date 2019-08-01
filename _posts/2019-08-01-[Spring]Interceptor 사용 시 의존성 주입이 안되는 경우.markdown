---
layout: post
title:  "[Spring]Interceptor 사용 시 의존성 주입이 안되는 경우"
date:   2019-08-01 23:00:00
author: EastGlow
categories: Back-end
---

## Interceptor에서 왜 Service Layer를 호출하지 못하지?

오늘 신규 프로젝트 개발 중에 해결이 안되는 상황을 겪었다. Custom Interceptor 하나를 등록해뒀는데 이 Interceptor 안에서 `@Resource`로 의존성 주입을 한 Service 객체가 작동을 안하는 것이다.

코드는 대략 아래와 같았다.

```java
@Slf4j
public class CustomInterceptor extends HandlerInterceptorAdapter {

    @Resource
    private CustomService customSvc;
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
    	log.debug("Custom Interceptor > postHandle ===> " + customSvc.selectData());
    }
}
```

Controller에 진입하고나서 ModelAndView를 return 하기 직전에 특정 행위를 모든 URL마다 포함해야했다. 그래서 생각해보다가 Interceptor를 이용하기로 했고 그 중에 postHandle을 Override하여 기능을 구현하면 될 거 같았다.

실제 코드는 위와 같이 단순 log용은 아니고 좀 더 복잡하지만 저걸로도 충분히 설명이 가능하여 간략하게 줄였다. 그냥 CustomService를 `@Resource`로 주입받아서 DB에서 특정 데이터를 조회해오는 부분이다.

그런데 막상 실행하니깐 저 log가 안 찍히는 것이다. CustomService을 구현한 CustomServiceImpl에서 `selectData()`라는 곳에 log를 찍어봐도 안 찍혔다. 그냥 애초에 CustomService 자체가 실행이 안되는 것이었다.


## 원인은 WebMvcConfigurer를 구현한 WebMvcConfig에 있었다.

요새는 xml 설정보다는 Spring에서 Java Config로 기본적인 설정들을 많이 하는 추세이다. 기존의 xml 설정들도 다 걷어내고 Java Config로 바꾸곤 했는데 이 프로젝트 역시 Java Config 기반이었다. WebMvcConfig는 대략 아래와 같다.

```java
@Slf4j
@Configuration
@EnableWebMvc
public class WebMvcConfig implements WebMvcConfigurer {
    
    (...생략...)
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new CustomInterceptor()).addPathPatterns("/**");
    }
    
    (...생략...)
}
```

위와 같이 처음에 만들었던 Interceptor를 `addInterceptors`를 Override하여 등록해주는 과정이다. 그런데 이 과정에 문제가 있었다. 위와 같이 `new()`를 통해 Interceptor 객체를 만들어서 등록하면 Spring Container에서 이 Interceptor를 관리하지 못한다고 한다.

>참고 : https://stackoverflow.com/questions/18218386/cannot-autowire-service-in-handlerinterceptoradapter

내가 구현한 코드처럼 Interceptor를 등록하면 Spring이 이것을 관리하지 못하게 되고, Interceptor에서 `@Resource`나 `@Autowired`를 통해 의존성을 주입하려고 해도 Spring에서 관리하지 않는 것이다보니 안되는 것이었다. 그럼 어떻게 바꾸면 잘 작동할까?

```java
@Slf4j
@Configuration
@EnableWebMvc
public class WebMvcConfig implements WebMvcConfigurer {
    
    (...생략...)
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(customInterceptor()).addPathPatterns("/**");
    }
    
    @Bean
    public CmsmInterceptor customInterceptor() {
        return new CustomInterceptor();
    }
    
    (...생략...)
}
```

위와 같이 `@Bean`을 생성하면 Spring에서 이 Bean을 실행하면서 만들고 관리하게 된다. 이것을 통해 Interceptor를 등록하게 되면 정상적으로 Service Layer를 `@Resource`를 통해 주입받고 작동하게 된다.
`@Bean`이라는 어노테이션을 정말 많이 사용하는 것 같은데 몰랐던 부분을 이렇게 알아가게 되니 좋았던 것 같다. 뭐든 잘 알아보고 써야한다는 점을 오늘 또 깨달은 하루였다.