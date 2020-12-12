---
layout: post
title:  "[Spring]Swagger2에서 특정 Method 문서화 제외하기"
date:   2020-12-13 01:40:00
author: EastGlow
categories: Back-end
---

현재 사내 Spring Boot 기반 프로젝트는 API 문서화를 Swagger를 이용하여 관리하고 있다.

Swagger 사용의 장점으로는  Swagger html 페이지에서 각 기능을 바로바로 테스트해볼 수 있고 관련 의존성을 추가해주는 것만으로도 손쉽게 문서화를 시킬 수 있다는 점이 있다.

하지만 Annotation 기반으로 작동하다 보니 Swagger 문서화 설정을 위해 기존 코드에 각종 Swagger Annotation이 붙게 되어 거슬릴 수도(?) 있고 생각보다 덕지덕지 붙은 코드가 많아지는 느낌이라 영 깔끔하지만은 않다.

그래도 사내 문서화 표준이 Swagger를 이용하고 있고 계속해서 잘 사용하고 있다. 그런데 사용 중에 특정 Zone에서 특정 Method를 Swagger 문서에서 제외시키고 싶은 경우가 생겼는데 공식 Docs나 구글링을 해봐도 Method 자체를 숨기는 방법은 있지만 특정 값에 의해 숨기는 방법은 별도로 기능 구현을 해야 하는 것 같았다.

> 참고: @ApiOperation(value = "테스트 API", hidden = true) 와 같이 특정 Method에 값을 주면 Swagger 문서에서는 보이지 않게 된다.

위에서 적용하고 하는 상황을 설명하자면 Production(이하 Prod) Zone에서는 Swagger 문서에서 테스트할 수 있게 열어두기엔 위험 부담이 있는 API가 있어서 이 API는 Prod Zone에서는 보여주고 싶지 않은 상황이었다.

아래의 예시 코드를 한번 보도록 하자.

## TestController.java

    @ApiOperation("회원정보 조회 API")  
    @GetMapping("/member/{memberId}")  
    public ResponseDto<TestResDto> getMemberInfo(@ApiParam(value = "회원ID", required = true) @PathVariable String memberId) {  
      
      return ResponseDto.<TestResDto>builder()  
              .data(memberService.getMemberInfo(memberId))  
              .build();  
    }

일반적인 조회 API이다. 실제 운영 중인 서비스에서 숨기고 싶었던 API는 회원에게 쿠폰을 발급해주는 API였는데 편의상 알아보기 쉬운 코드로 바꿔보았다.

`@PathVariable`로 받은 `memberId`를  가지고 Service를 호출하여 회원정보를 조회하는 API이다. 나는 이 API를 Dev, Qa Zone에서만 보여주고 싶고 Staging, Prod Zone에서는 숨기고 싶었다.

## SwaggerConfig.java

    @EnableSwagger2  
    @Configuration  
    public class SwaggerConfig {  
      
        private static final String API_BASE_PACKAGE = "me.eastglow";
      
        @Bean  
        public Docket memberApi() {  
            String apiDomain = "Member";  
            String apiName = apiDomain + " API";  
      
            ApiInfo apiInfo = new ApiInfoBuilder()  
                .title(apiName)  
                .description(apiName + " Document Prototype")  
                .version("0.0.1")  
                .build();  
      
            return new Docket(DocumentationType.SWAGGER_2)  
                .groupName(apiDomain)  
                .apiInfo(apiInfo)  
                .select()  
                .apis(RequestHandlerSelectors.basePackage(API_BASE_PACKAGE))  
                .paths(PathSelectors.any())  
                .build();  
        }  
    }

일반적인 Swagger 설정을 담고 있는 코드이다. `RequestHandlerSelectors`를 통해 지정된 `API_BASE_PACKAGE` 경로에 있는 Controller Method를 찾고 바로 아래의 `PathSeelctors`를 통해 `any()`, 즉 모든 URL이 있는 API를 선택하여 `build()`하고 있는 것이다.

그러면 특정 Method만 `build()`항목에 포함되지 않게 하려면 어떻게 해야 할까? `apis()`에서 그것들만 제외하고 select 하도록 바꿔주면 된다.

그러기 위해선 `apis()`가 받고 있는 파라미터를 한번 볼 필요가 있었다. 

## `apis()` > ApiSelectorBuilder.java

    public ApiSelectorBuilder apis(Predicate<RequestHandler> selector) {
        requestHandlerSelector = and(requestHandlerSelector, selector);
        return this;
    }

보시다시피 파라미터로 `Predicate<RequestHandler>`를 받아주고 있다. Java8부터 `java.util.function`에 추가된 함수형 인터페이스이다. 

위 `apis()`의 파라미터로 넘겨주던 값은 `RequestHandlerSelectors.basePackage()`인데 세세히 설명하기엔 하나하나 다 설명이 필요하게 되기도 하고, 나도 대략적인 코드만 이해하고 구현한 거라 완벽한 설명이 불가능하기에 적당히 "basePackage()에 받은 경로 아래에 있는 것들을 select 하여 apis()로 넘겨준다"정도로 이해하면 될 것 같다.

어쨌든 중요한 건 `apis()`에 넘겨주는 Selector를 바꿔주는 것이다. `basePackage` 기준으로만 select 해오던 것을 특정 Annotation이 붙은 것은 제외하고 select 하도록 바꿔줄 것이다. 그러기 위해서는 우선 Annotation을 하나 만들어주도록 한다.

## IgnoreSwaggerInit.java

    @Retention(RetentionPolicy.RUNTIME)  
    @Target({ElementType.METHOD})  
    public @interface IgnoreSwaggerInit {  
      
        String[] value() default "prod";  
    }

Method에만 붙일 수 있는 Annotation이고 숨기고 싶은 Zone을 String 배열 형태로 받게 되어있다. (여러 Zone에서 숨기고 싶을 수도 있으니깐)

## TestController.java

    @IgnoreSwaggerInit({"stg","prod"})
    @ApiOperation("회원정보 조회 API")  
    @GetMapping("/member/{memberId}")  
    public ResponseDto<TestResDto> getMemberInfo(@ApiParam(value = "회원ID", required = true) @PathVariable String memberId) {  
      
      return ResponseDto.<TestResDto>builder()  
              .data(memberService.getMemberInfo(memberId))  
              .build();  
    }

아까 위에서 봤던 `TestController`이다. 제일 윗줄을 보면 `@IgnoreSwaggerInit` Annotation이 추가된 것을 볼 수 있다. `getMemberInfo` Method를 Staging, Prod Zone에서는 숨길 것이다.

## SwaggerConfig.java

    @EnableSwagger2  
    @Configuration  
    public class SwaggerConfig {  
    
        private static final String API_BASE_PACKAGE = "me.eastglow";
    
        @Bean  
        public Docket memberApi() {  
            String apiDomain = "Member";  
            String apiName = apiDomain + " API";  
    
            ApiInfo apiInfo = new ApiInfoBuilder()  
                .title(apiName)  
                .description(apiName + " Document Prototype")  
                .version("0.0.1")  
                .build();  
    
            return new Docket(DocumentationType.SWAGGER_2)  
                .groupName(apiDomain)  
                .apiInfo(apiInfo)  
                .select()  
                .apis(getSelector(API_BASE_PACKAGE)
                .paths(PathSelectors.any())  
                .build();  
        }  
    
        private Predicate<RequestHandler> getSelector(String basePackage){
    
            String profiles = Profiles.getZone().value();
    
            return Predicates.and(RequestHandlerSelectors.basePackage(basePackage), handler
                    -> {
                            Optional<IgnoreSwaggerInit> optional = handler.findAnnotation((IgnoreSwaggerInit.class));
                            if (optional.isPresent() && Arrays.asList(optional.get().value()).contains(profiles)) {
                                return false;
                            } else {
                                return true;
                            }
                    });
        }
    }

`SwaggerConfig` 코드는 조금 많이 바뀌었다. 우선 `getSelector`라는 별도의 Selector 처리 Method가 추가되었다.

코드를 살펴보면 먼저 현재 실행되는 Zone의 이름을 가져오고 있으며, `Predicates.and()`를 통해 return 할 `RequestHandler`를 체크하고 있다.

`RequestHandlerSelectors.basePackage(basePackage)`를 통해 조회된 handler를 가지고 `findAnnotation`을 통해 `IgnoreSwaggerInit` Annotation을 가지고 있는 것들을 Optional 객체로 구해온다.

해당 Optional 객체의 `isPresent()`가 true값이라면 null이 아닌 객체인 것이고 그 객체가 가지고 있는 `value()`값, 즉 위에서 `@IgnoreSwaggerInit({"stg","prod"})`를 통해 넘겨준 `"stg","prod"`가 든 String 배열과 처음에 구한 `String profiles`가 포함되어있는지(contains) 체크하여 포함이 되어 있으면 `fasle`를 return 하고 해당 메서드를 제외하고, 아니면 `true`를 return 하여 해당 메서드를 포함시킨다.

SwaggerConfig의 수정을 정상적으로 마쳤다면 `@IgnoreSwaggerInit` Annotation을 추가한 메서드가 Swagger 문서화에 포함되지 않는 것을 확인할 수 있다. 
