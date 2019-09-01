---
layout: post
title:  "[Spring]RestTemplate를 이용하여 카카오 API 사용하기 (1)"
date:   2019-08-20 00:00:00
author: EastGlow
categories: Back-end
---

이번 글에서는 Spring에서 기본적으로 제공하는 RestTemplate를 이용하여 카카오 REST API 및 카카오맵 API를 써본 경험을 남기려 한다.

대략적인 글의 흐름은,

1. Spring Boot 기본 프로젝트 세팅(controller, service, dao, mapper 등 생성)
2. 프론트 화면 구성
3. AJAX를 통해 API 호출
4. Controller에서 API 호출을 요청받으면 API를 처리하는 class를 호출하여 RestTemplate를 이용하여 REST API 통신을 한 뒤, 결과값을 Return

와 같이 진행 될 것이다. 이번편에는 1~2번까지만 다룰 것이며 3~4번은 다음 포스팅에서 다룰 예정이다.

# 1. Spring Boot 기본 프로젝트 세팅

이전 포스팅 중에서도 Spring Boot 기본 프로젝트를 만드는 부분이 있지만 간단하게 한번 정리하고 넘어가고자 한다.

## 1. IntelliJ 에서 Spring Initializr를 이용하여 Spring Boot 프로젝트를 생성

Spring Boot를 한번이라도 이용해봤다면 다들 알 법한 Spring Initializr를 이용하여 Spring Boot 기본 프로젝트를 만들어준다.

의존성은 아래와 같이 선택하였다.

- Spring Boot DevTools : 수정과 동시에 확인을 하기 위해 이용
- Lombok : 말이 필요한가. 획기적으로 코드량을 줄여주는 갓복...
- Spring Web Starter : Web 프로젝트를 만들 때 기본적으로 선택.
- H2 Database : 간단하게 API를 테스트 하기 위한 용도로 인메모리 DB인 H2 DB를 선택.
- Mybatis Framework : JPA는 아직 써보지 않아서 익숙한 Mybatis를 선택.

만들어본 지 좀 지난(...) 프로젝트라 기억이 정확하진 않지만 대략 위와 같이 선택하고 프로젝트를 생성해주었다.

## 2. 패키지 구조 생성

패키지 구조는 어디서나 흔히 볼 수 있는 구조로 만들어주었다. 대략 아래와 같다.

![](/assets/post/20190819_1.png)

java 패키지 밑에는 common, configuration, controller, dao, service, vo, Application.java가 위치하고 있다.

### common

로그인 여부를 체크할 인터셉터와 카카오 API와의 통신을 도울 API 관련 클래스가 존재한다.

#### AuthInterceptor.java

그냥 단순하게 session에 로그인 VO 값이 담겨있는지 체크하여 로그인을 해야 접근 가능한 페이지들에 대하여 체크하는 인터셉터이다.

```java
@Slf4j
@Component
public class AuthInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HttpSession session = request.getSession();
        LoginVO loginVO = (LoginVO) session.getAttribute("loginVO");

        if(loginVO == null || !loginVO.isValidUser()){
            response.setContentType("text/html; charset=UTF-8");

            PrintWriter out = response.getWriter();

            out.println("<script>alert('세션이 만료되어 로그아웃 되었습니다. 다시 로그인 해주세요.'); window.location.href='/login';</script>");
            out.flush();

            return false;
        }

        return true;
    }
}
```

#### KakaoRestApiHelper.java

`키워드로 장소 검색`이라는 REST API를 이용해야했다. (https://developers.kakao.com/docs/restapi/local#%ED%82%A4%EC%9B%8C%EB%93%9C%EB%A1%9C-%EC%9E%A5%EC%86%8C-%EA%B2%80%EC%83%89)

아래의 `getSearchPlaceByKeyword`를 통해 검색 관련 파라미터들을 받은 후, queryString을 만들어서 REST API 통신을 하도록 한다. header에는 인증을 위한 API Key를 담아주고 RestTemplate의 exchange를 이용하여 return 받은 결과를 String 형태로 담아주고 있다. 더 자세한 설명은 4번 항목에서 하겠다.

```java
@Slf4j
@Component
public class KakaoRestApiHelper {
    @Value("${kakao.restapi.key}")
    private String restApiKey;

    private static final String API_SERVER_HOST  = "https://dapi.kakao.com";
    private static final String SEARCH_PLACE_KEYWORD_PATH = "/v2/local/search/keyword.json";

    public ResponseEntity<String> getSearchPlaceByKeyword(SearchVO searchVO) throws Exception {
        String queryString = "?query="+URLEncoder.encode(searchVO.getKeywordNm(), "UTF-8")+"&page="+searchVO.getCurrentPage()+"&size="+searchVO.getPageSize();
        RestTemplate restTemplate = new RestTemplate();
        HttpHeaders headers = new HttpHeaders();

        headers.add("Authorization", "KakaoAK " + restApiKey);
        headers.add("Accept", MediaType.APPLICATION_JSON_VALUE);
        headers.add("Content-Type", MediaType.APPLICATION_FORM_URLENCODED_VALUE + ";charset=UTF-8");

        URI url = URI.create(API_SERVER_HOST+SEARCH_PLACE_KEYWORD_PATH+queryString);
        RequestEntity<String> rq = new RequestEntity<>(headers, HttpMethod.GET, url);
        ResponseEntity<String> re = restTemplate.exchange(rq, String.class);

        return re;
    }
}
```

### configuration

기본적인 Web MVC 설정이 들어있는 WebMvcConfig 클래스와 H2 DB에 더미 계정 정보(비밀번호 암호화를 위해 xml이 아닌, java 단에서 초기 데이터를 적재)를 넣어주기 위한 InitDataLoder가 존재한다.

#### InitDataLoader.java

```java
@Component
public class InitDataLoader implements CommandLineRunner {

    @Resource
    private LoginService loginService;

    @Override
    public void run(String... args) throws Exception {
        loginService.setInitUserData();
    }
}
```

CommandLineRunner에 있는 run을 Override하여 구현하면 최초 애플리케이션 실행 시 원하는 행위를 실행할 수 있다고 한다. 나는 더미 계정 정보를 넣기 위해 run 안에 `etInitUserData`가 실행되도록 하였다.

```java
@Override
public void setInitUserData() throws Exception {
    for(int i=1; i<=3; i++){
        LoginVO tempVO = new LoginVO("test"+i, bCryptPasswordEncoder.encode("test"+i), "테스터"+i);
        loginMapper.insertInitUserData(tempVO);
    }
}
```

```
<insert id="insertInitUserData" parameterType="loginVO">
    INSERT INTO TBL_USER(USER_ID, USER_PW, USER_NM)
    VALUES (#{userId}, #{userPw}, #{userNm})
</insert>
```

해당 메소드는 아래와 같이 생겼으며 단순히 더미 계정 정보 중 비밀번호의 암호화가 필요하여 이런식으로 초기화하도록 하였다. 굳이 암호화가 필요없다면 `data.sql`에 Insert 쿼리를 적어두면 된다. H2 DB를 사용한다면 최초 애플리케이션 실행 시 `data.sql`이라는 파일에 원하는 쿼리를 적어두면 자동으로 실행되니 참고하기 바란다.

#### WebMvcConfig.java

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Resource
    private AuthInterceptor authInterceptor;

    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder() {
        BCryptPasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder();
        return bCryptPasswordEncoder;
    }

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("forward:/login");
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/index");
    }
}
```

특별한 설정은 없다. 암호화 시 이용할 Encoder를 Bean으로 선언해두었으며, "/"에 대해서 login 페이지로 이동되도록 설정하고 "/index"에 대해서 앞에서 설명한 세션 체크 인터셉터가 적용되도록 해두었다.


### controller

흔히들 아는 Controller들이 존재하는 곳이다. 크게 특별한 건 없어서 여기서부터 자세한 설명은 생략한다.

### dao

Mapper Interface가 존재한다.

### service

Service 및 Service Implement가 존재한다.

### vo

VO가 존재한다.

resources에는 로그인 및 API 데이터를 활용할 관련 테이블들을 만들어주는 쿼리가 든 sql 파일, mapper 관련 쿼리 xml 파일, 정적 리소스 파일들이 위치한다. 그 외엔 application.properties와 같은 설정파일들이 있다.

참고로 톰캣은 외장이 아닌, 내장 톰캣을 이용하였다. (jar 파일로 만들었을 때 단독으로 실행될 수 있도록.)

# 2. 프론트 화면 구성

사실 프론트 쪽 개발자가 아니다보니 react나 vue는 잘 몰라서 jsp로 할까하다가 최대한 SPA(Single-Page Application) 느낌(?)이 나도록 구현해보고 싶어서 html에 js로만 만들어보았다. 어설픈 SPA 흉내내기에 불과했지만 이것 조차 나한텐 너무 빡셌다...

사용한 것들은 html, css, bootstrap, javascript, jquery 정도이다. REST API 통신은 jquery의 Ajax를 이용하였다. 내가 만든 화면의 모습들은 아래와 같다. 그냥 진짜 기능만 잘 돌아가고 깔끔해보이도록 만들어두어 초라(...)하다.

![](/assets/post/20190819_2.png)

![](/assets/post/20190819_3.png)

![](/assets/post/20190819_4.png)

![](/assets/post/20190819_5.png)

이 화면을 구성하는 실질적인 Back단 요소들은 다음편에서 다루도록 하겠다.