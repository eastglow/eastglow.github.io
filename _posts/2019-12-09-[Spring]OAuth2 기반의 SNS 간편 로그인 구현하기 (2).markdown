---
layout: post
title:  "[Spring]OAuth2 기반의 SNS 간편 로그인 구현하기 (2)"
date:   2019-12-09 18:00:00
author: EastGlow
categories: Back-end
---

`OAuth2 기반의 SNS 간편 로그인 구현하기 (1)`에서 각 플랫폼 별로 설정하는 법을 알아보았다. API를 쓰기 위한 준비는 모두 완료되었으니 실제 프로젝트에서는 어떻게 이용하면 되는지 살펴보도록 하자.

# 2. Front-end 및 Back-end에서 이용하기

## 1) Front-end

전에 다뤘던 REST API 글에서와 마찬가지로 View 쪽은 JSP와 Javascript로 구성된다. 이부분은 각자 알아서 구성을 하면 될 것이고... Front 쪽에서 필요한 것은 `(1) 버튼 이미지`와 `(2) 버튼에 들어갈 링크`이다. 버튼 이미지는 각 플랫폼 API 문서에 들어가면 제공하고 있거나 직접 만들어쓰거나 하면 될 것이고... 버튼에 들어갈 링크 역시 API 문서에 어떻게 만들어 쓰면 되는지 다 나와있다.

Front-end 쪽에서 실행되는 SNS 간편 로그인 or 회원가입 프로세스를 간단히 정리하자면 아래와 같다.

1. 사용자가 로그인 페이지에 접근한다.
2. 로그인 페이지의 각 플랫폼 별 간편 로그인 버튼이 3개가 있을 것이다.
3. 사용자는 간편 로그인을 시도할 것이다.
4. 각 로그인 버튼에는 각 플랫폼 별 로그인 페이지로 이동되는 링크가 걸려있다.
5. 해당 링크를 통해 플랫폼 별 로그인을 시도하여 성공하면 redirect URI로 설정해둔 페이지로 돌아오게 된다.

대략적인 프로세스는 위와 같고 아래의 소스는 위 프로세스 중 4번에 해당하는 링크를 만들어주는 메서드이다. Front-end 쪽인데 Java 소스가 나오는 이유는 버튼 링크를 만들어주는 부분도 Javascript 변수에 넣어두고 처리하기 보단 서버 쪽에서 properties 파일에 키값이나 기타 필요한 값들을 담아놓고 쓰는게 편할 거 같아서 그랬다.

### SnsController.java 중 getNaverLoginUri 소스

```
@RequestMapping(value={"/naver/Login.json"})  
public ModelAndView getNaverLogin(HttpServletRequest request, HttpServletResponse response, HttpSession session) throws Exception {  
    ModelAndView mav = new ModelAndView();  
    mav.addObject("apiUri", snsUtil.getCallbackUri("naver"));  
  
    return mav;  
}
```


### SnsUtil.java 중 getCallbackUri 소스

```
public String getCallbackUri(String type) throws Exception {  
    SecureRandom random = new SecureRandom();  
    String state = new BigInteger(130,random).toString();  
    String apiUrl = "";
    
    if("naver".equals(type)){  
        apiUrl = "https://nid.naver.com/oauth2.0/authorize";  
        apiUrl += "?response_type=code";  
        apiUrl += "&client_id="+naverApiKey;  
        apiUrl += "&redirect_uri=https://eastglow.com/naver/Join.do";  
        apiUrl += "&state="+state;  
    }else if("kakao".equals(type)){  
        apiUrl = "https://kauth.kakao.com/oauth/authorize";  
        apiUrl += "?client_id="+kakaoApiKey;  
        apiUrl += "&redirect_uri=https://eastglow.com/kakao/Join.do";  
        apiUrl += "&response_type=code";  
        apiUrl += "&state="+state;  
        apiUrl += "&encode_state=true";  
    }  
  
    return apiUrl;  
}
```

위 메서드가 Back-end 쪽에서 버튼 링크를 만들어주는 부분이다. 위쪽에서 먼저 언급했던 로그인 프로세스 과정 중 2번 과정인 로그인 페이지에 접근했을 때, AJAX를 통해 위 메서드를 호출하여 각 타입별로 링크를 전달 받아 로그인 버튼에 꽂아준다. 간단히 코드화해보자면 아래와 같다.

1. 로그인 페이지 접근 시, `document.ready`함수를 통해 AJAX로 `/naver/Login.json`을 호출한다.
2. 1번을 호출하면 `SnsController`의 `getNaverLoginUri`를 호출하게 되고 그 안에서 `SnsUtil.getCallbackUri`를 호출하여 return 값에서 볼 수 있듯이 각 플랫폼 별로 링크 String 변수를 전달받는다. (페이스북은 social 관련 디펜던시가 있어서 따로 메서드가 있다. 뒤쪽에서 설명하겠다.)
3. `$('#naverBtn').prop('href', data.apiUrl);` 대충 이런 느낌(?)의 AJAX Success 시 callback Function을 호출하여 각 버튼에 링크값을 넣어준다.

참고로 플랫폼 별로 받는 파라미터 중 redirect_uri는 앞에서 설명했던 플랫폼 별 설정 중 redirect URI를 설정하는 부분에서 본인이 설정해둔 값이 들어가야한다. 다른 값이 들어가게 되면 호출 시 권한 오류가 나게 된다.

어쨌든 위와 같이 AJAX를 통해 Java Controller를 호출하여 버튼 링크를 만들어줘도 좋고 Javascript에서 만들어줘도 좋으니 버튼 링크만 정상적으로 잘 만들어주도록 하자. 그렇게 해서 버튼에 링크가 제대로 담기게 되면 해당 링크를 클릭했을 때, 각 플랫폼 별 로그인 페이지가 새 창으로 열리게 된다. 거기서 로그인을 하면 redirect URI로 설정해둔 페이지로 리다이렉트 되게 된다.


## 2) Back-end

앞의 과정을 잘 거쳐왔다면 플랫폼 별 로그인 후에 리다이렉트되는 URI가 잘 호출되었을 것이다. 나같은 경우는 `https://eastglow.com/naver/Join.do`를 설정해주었기 때문에 네이버 로그인이 성공했다면 위 URI로 돌아왔을 것이다.

### SnsController.java 중 getNaverJoin 소스

```
@RequestMapping(value={"/naver/Join.do"})  
public String getNaverJoin(HttpServletRequest request, HttpServletResponse response, HttpSession session, Model model,  
  @RequestParam(value="code",required=false,defaultValue="") String code,  
  @RequestParam(value="state",required=false,defaultValue="") String state) throws Exception {  
  
    String accessToken = "";  
  
    // 접근 토큰 발급 요청  
    ResponseEntity<Map> accessTokenEntity = snsUtil.getNaverAccessToken(code, state);  
    Map<String, Object> accessTokenMap = accessTokenEntity.getBody();  
    accessToken = (accessTokenMap != null && !accessTokenMap.isEmpty()) ? String.valueOf(accessTokenMap.get("access_token")) : "";  
  
    // 회원 프로필 조회  
    ResponseEntity<Map> memberProfileEntity = snsUtil.getNaverMemberProfile(accessToken);  
    Map<String, Object> memberProfileMap = memberProfileEntity.getBody();  
    Map<String, Object> responseMap = (Map<String, Object>) memberProfileMap.get("response");  
  
    model.addAttribute("responseMap", responseMap);  
  
    return "join";  
}
```

### SnsUtil.java 소스

```
@Value("${sns.kakao.api.key}") private String kakaoApiKey;  
@Value("${sns.kakao.secret.key}") private String kakaoSecretKey;  
  
@Value("${sns.naver.api.key}") private String naverApiKey;  
@Value("${sns.naver.secret.key}") private String naverSecretKey;  
  
@Value("${sns.facebook.api.key}") private String facebookApiKey;  
@Value("${sns.facebook.secret.key}") private String facebookSecretKey;  
  
private static final String KAKAO_TOKEN_URI = "https://kauth.kakao.com/oauth/token";  
private static final String KAKAO_INFO_URI = "https://kapi.kakao.com/v2/user/me";  
private static final String KAKAO_VERIFY_URI = "https://kapi.kakao.com/v1/user/access_token_info";  
  
private static final String NAVER_TOKEN_URI = "https://nid.naver.com/oauth2.0/token";  
private static final String NAVER_INFO_URI = "https://openapi.naver.com/v1/nid/me";  
private static final String NAVER_VERIFY_URI = "https://openapi.naver.com/v1/nid/verify";

public ResponseEntity<Map> getNaverAccessToken(String code, String state) throws Exception {  
    RestTemplate restTemplate = new RestTemplate();  
  
    MultiValueMap<String, String> paramMap = new LinkedMultiValueMap<>();  
    paramMap.add("grant_type", "authorization_code");  
    paramMap.add("client_id", naverApiKey);  
    paramMap.add("client_secret", naverSecretKey);  
    paramMap.add("code", code);  
    paramMap.add("state", state);  
  
    URI uri = URI.create(NAVER_TOKEN_URI);  
  
    return restTemplate.postForEntity(uri, paramMap, Map.class);  
}  
  
public ResponseEntity<Map> getNaverMemberProfile(String accessToken) throws Exception {  
    RestTemplate restTemplate = new RestTemplate();  
    HttpHeaders headers = new HttpHeaders();  
  
    headers.add("Authorization", "Bearer " + URLEncoder.encode(accessToken, "UTF-8");  
    headers.add("Content-Type", MediaType.APPLICATION_FORM_URLENCODED_VALUE + ";charset=UTF-8");  
  
    URI uri = URI.create(NAVER_INFO_URI);  
    RequestEntity<String> rq = new RequestEntity<>(headers, HttpMethod.POST, uri);  
    ResponseEntity<Map> re = restTemplate.exchange(rq, Map.class);  
  
    return re;  
}
```

기존 소스를 알아보기 쉽게 좀 정리하다보니 생략된 부분도 있을 것이고 간략화 한 부분도 있어서 매끄럽지 않은 부분이 있을 수도 있다. 이 글 자체가 완전 Java를 처음 다뤄보는 분들을 대상으로 한 글은 아니기 때문에 어느정도 위 코드만으로 이해가 가능한 분들이 읽을 것을 생각하고 코드 정리를 하였으니 양해 바란다.

여튼 `https://eastglow.com/naver/Join.do`를 호출하게 되면 SnsController에 있는 getNaverJoin를 호출하게 되는데 이 안에서 먼저 accessToken을 얻기 위한 행위를 하게 된다. 그것이 바로 SnsUtil의 getNaverAccessToken이다.

RestTemplate를 생성하여 API Key 값과 Secret Code 값을 담아주고 네이버 API의 token을 요청하는 URI를 호출해준다. 그러면 네이버 REST API에서는 유효한 accessToken 값을 내려줄 것이고 이것을 가지고 현재 로그인을 시도한 계정의 프로필 값을 불러올 것이다.

프로필 값을 받아오는 부분은 getNaverMemberProfile 메서드이다. 받아온 accessToken을 파라미터로 받고 있는데 이것을 header 쪽에 Authorization로 담아주면 권한 인증이 이루어지게 된다. 그런 다음 계정 프로필 조회를 하는 REST API URI를 호출하면 Map 형태로 정보가 담기게 된다. Map에 담기는 정보는 아래 링크에서 확인해볼 수 있다.

[접근 토큰을 이용하여 프로필 API 호출하기](https://developers.naver.com/docs/login/devguide/#3-4-5-%EC%A0%91%EA%B7%BC-%ED%86%A0%ED%81%B0%EC%9D%84-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-%ED%94%84%EB%A1%9C%ED%95%84-api-%ED%98%B8%EC%B6%9C%ED%95%98%EA%B8%B0)

return 된 Map 안에 보면 response라는 객체가 담겨있는데 이 안에 계정과 관련된 정보들이(아이디, 닉네임, 이름, 메일주소 등) 담겨있다. 그래서 최종적으로 아래와 같이 return 받은 Map에서 response를 따로 빼내어 사용하고 있다.

```
// 회원 프로필 조회  
ResponseEntity<Map> memberProfileEntity = snsUtil.getNaverMemberProfile(accessToken);  
Map<String, Object> memberProfileMap = memberProfileEntity.getBody();  
Map<String, Object> responseMap = (Map<String, Object>) memberProfileMap.get("response");
```

카카오의 경우 네이버와 호출하는 주소만 다를 뿐 전반적인 과정은 같기 때문에 이 글 끝에 올려둔 전체 소스에서 확인해보길 바란다.

페이스북은 spring social 관련 디펜던시를 이용하여 진행하였기 때문에 RestTemplate로 진행한 카카오, 네이버와는 조금 소스가 다르다.

페이스북은 SnsUtil.java에서 미리 2개의 Bean을 등록해두고 시작한다. 아래 소스가 그것들이다.

```
@Bean  
public FacebookConnectionFactory facebookConnectionFactory() {  
    return new FacebookConnectionFactory(facebookApiKey, facebookSecretKey);  
}  
  
@Bean  
public OAuth2Parameters oAuth2Parameters() {  
    OAuth2Parameters param = new OAuth2Parameters();  
    param.setRedirectUri("https://eastglow.com/facebook/Join.do");  
  
    return param;  
}
```

그러고나서 SnsController쪽을 보면 getFacebookLogin, getFacebookJoin를 통해서 각각 로그인 버튼 링크를 만들어주고 로그인 후에 리다이렉트 되는 페이지로의 이동을 다루게 된다. 소스를 보는 편이 이해가 더 빠를 거라 생각되어 별도의 설명은 생략하도록 하겠다.

글을 마치면서 전체 프로세스를 한번 정리해보자면,

1. 사용자가 로그인 페이지에 접근한다.
2. 로그인 페이지에서 각 플랫폼 별 로그인 버튼에 넣어줄 링크값을 서버단에서 만들어서 넣어준다.
3. 사용자가 만들어진 로그인 링크를 클릭하면 각 플랫폼 별 로그인 페이지로 이동한다.
4. 사용자가 로그인을 성공하게 되면 redirect URI로 설정해둔 페이지로 이동한다.
5. (나같은 경우) `/{플랫폼이름}/Join.do`으로 이동하게 되는데 이 URI로 접근하게 되면 먼저 사용자의 accessToken을 취득한 뒤, 그 accessToken을 통해서 사용자의 프로필 조회를 하게 된다.
6. 최종적으로 사용자의 프로필 조회로 받은 정보를 JSP 쪽으로 내려주어 완료하게 된다.

나는 단순히 accessToken이 필요한 프로필 조회 API를 이용하기 위해 accessToken을 취하고 프로필 조회만 하는 기능을 구현했지만 보통은 refreshToken도 필요하고 expire 됐을 경우에 다시 token을 발급받고 하는 과정 등이 필요할 수도 있다. 이러한 로직을 구현하려면 DB나 Redis 등의 캐쉬를 이용해서 하는 예제들을 본 듯 하다. 이러한 경우는 나중에 기회가 된다면 다뤄보도록 하겠다.

***

## SnsController.java

```
package me.eastglow.controller;  
  
import me.eastglow.common.util.SnsUtil;  
import lombok.extern.slf4j.Slf4j;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.beans.factory.annotation.Value;  
import org.springframework.http.ResponseEntity;  
import org.springframework.social.connect.Connection;  
import org.springframework.social.facebook.api.Facebook;  
import org.springframework.social.facebook.api.UserOperations;  
import org.springframework.social.facebook.api.impl.FacebookTemplate;  
import org.springframework.social.facebook.connect.FacebookConnectionFactory;  
import org.springframework.social.oauth2.AccessGrant;  
import org.springframework.social.oauth2.GrantType;  
import org.springframework.social.oauth2.OAuth2Operations;  
import org.springframework.social.oauth2.OAuth2Parameters;  
import org.springframework.stereotype.Controller;  
import org.springframework.ui.Model;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RequestParam;  
import org.springframework.web.servlet.ModelAndView;  
  
import javax.annotation.Resource;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
import javax.servlet.http.HttpSession;  
import java.util.HashMap;  
import java.util.Map;  
  
@Slf4j  
@Controller  
public class SnsApiController {  
  
    @Resource  
    private SnsUtil snsUtil;  
  
    @Resource  
    private FacebookConnectionFactory connectionFactory;  
  
    @Resource  
    private OAuth2Parameters oAuth2Parameters;  
  
    @RequestMapping(value={"/facebook/Login.json"})  
    public ModelAndView getFacebookLogin(HttpServletRequest request, HttpServletResponse response, HttpSession session) throws Exception {  
        ModelAndView mav = new ModelAndView();  
        OAuth2Operations oauthOperations = connectionFactory.getOAuthOperations();  
        String apiUri = oauthOperations.buildAuthorizeUrl(GrantType.AUTHORIZATION_CODE, oAuth2Parameters);  
  
        mav.addObject("apiUri", apiUri);  
  
        return mav;  
    }  
  
    @RequestMapping(value={"/facebook/Join.do"})  
    public String getFacebookJoin(HttpServletRequest request, HttpServletResponse response, HttpSession session, Model model,  
  @RequestParam(value="code",required=false,defaultValue="") String code) throws Exception {  
  
        String accessToken = "";  
  
        // 접근 토큰 발급 요청  
        OAuth2Operations oAuth2Operations = connectionFactory.getOAuthOperations();  
        AccessGrant accessGrant = oAuth2Operations.exchangeForAccess(code, oAuth2Parameters.getRedirectUri(), null);  
        accessToken = accessGrant.getAccessToken();  
  
        Long currentTime = System.currentTimeMillis();  
        Long expireTime = accessGrant.getExpireTime();  
  
        if (expireTime != null && expireTime < currentTime) {  
            accessToken = accessGrant.getRefreshToken();  
        };  
  
        // 회원 프로필 조회  
        Connection<Facebook> connection = connectionFactory.createConnection(accessGrant);  
        Facebook facebook = connection == null ? new FacebookTemplate(accessToken) : connection.getApi();  
        UserOperations userOperations = facebook.userOperations();  
  
        String [] fields = { "id", "email", "name"};  
        Map<String, Object> memberProfileMap = facebook.fetchObject("me", Map.class, fields);  
  
        model.addAttribute("memberProfileMap", memberProfileMap);  
  
        return "join";  
    }
  
    @RequestMapping(value={"/naver/Login.json"})  
    public ModelAndView getNaverLogin(HttpServletRequest request, HttpServletResponse response, HttpSession session) throws Exception {  
        ModelAndView mav = new ModelAndView();  
        mav.addObject("apiUri", snsUtil.getCallbackUri("naver"));  
  
         return mav;  
    }  
  
    @RequestMapping(value={"/naver/Join.do"})  
    public String getNaverJoin(HttpServletRequest request, HttpServletResponse response, HttpSession session, Model model,  
  @RequestParam(value="code",required=false,defaultValue="") String code,  
  @RequestParam(value="state",required=false,defaultValue="") String state) throws Exception {  
  
        String accessToken = "";  
  
        // 접근 토큰 발급 요청  
        ResponseEntity<Map> accessTokenEntity = snsUtil.getNaverAccessToken(code, state);  
        Map<String, Object> accessTokenMap = accessTokenEntity.getBody();  
        accessToken = (accessTokenMap != null && !accessTokenMap.isEmpty()) ? String.valueOf(accessTokenMap.get("access_token")) : "";  
  
        // 회원 프로필 조회  
        ResponseEntity<Map> memberProfileEntity = snsUtil.getNaverMemberProfile(accessToken);  
        Map<String, Object> memberProfileMap = memberProfileEntity.getBody();  
        Map<String, Object> responseMap = (Map<String, Object>) memberProfileMap.get("response");  
  
        model.addAttribute("responseMap", responseMap);  
  
        return "join";  
    }  
  
    @RequestMapping(value={"/kakao/Login.json"})  
    public ModelAndView getKakaoLogin(HttpServletRequest request, HttpServletResponse response, HttpSession session) throws Exception {  
        ModelAndView mav = new ModelAndView();  
        mav.addObject("apiUri", snsUtil.getCallbackUri("kakao"));  
  
        return mav;  
    }  
  
    @RequestMapping(value={"/kakao/Join.do"})  
    public String getKakaoJoin(HttpServletRequest request, HttpServletResponse response, HttpSession session, Model model,  
  @RequestParam(value="code",required=false,defaultValue="") String code,  
  @RequestParam(value="state",required=false,defaultValue="") String state) throws Exception {  
  
        String accessToken = "";  
  
        // 접근 토큰 발급 요청  
        ResponseEntity<Map> accessTokenEntity = snsUtil.getKakaoAccessToken(code);  
        Map<String, Object> accessTokenMap = accessTokenEntity.getBody();  
        accessToken = (accessTokenMap != null && !accessTokenMap.isEmpty()) ? String.valueOf(accessTokenMap.get("access_token")) : "";  
  
        // 회원 프로필 조회  
        ResponseEntity<Map> memberProfileEntity = snsUtil.getKakaoMemberProfile(accessToken);  
        Map<String, Object> memberProfileMap = memberProfileEntity.getBody();  
        Map<String, Object> kakaoAccountsMap = (Map<String, Object>) memberProfileMap.get("kakao_account");  
  
        model.addAttribute("memberProfileMap", memberProfileMap);  
  
        return "join";  
    }  
}
```

## SnsUtil.java

```
package me.eastglow.common.util;  
  
import lombok.extern.slf4j.Slf4j;  
import org.springframework.beans.factory.annotation.Value;  
import org.springframework.context.annotation.Bean;  
import org.springframework.http.*;  
import org.springframework.social.facebook.connect.FacebookConnectionFactory;  
import org.springframework.social.oauth2.OAuth2Parameters;  
import org.springframework.stereotype.Component;  
import org.springframework.util.LinkedMultiValueMap;  
import org.springframework.util.MultiValueMap;  
import org.springframework.web.client.RestTemplate;  
  
import java.math.BigInteger;  
import java.net.URI;  
import java.net.URLEncoder;  
import java.security.SecureRandom;  
import java.util.Map;  
  
@Slf4j  
@Component  
public class SnsUtil {  
  
    @Value("${sns.kakao.api.key}") private String kakaoApiKey;  
    @Value("${sns.kakao.secret.key}") private String kakaoSecretKey;  
  
    @Value("${sns.naver.api.key}") private String naverApiKey;  
    @Value("${sns.naver.secret.key}") private String naverSecretKey;  
    @Value("${sns.facebook.api.key}") private String facebookApiKey;  
    @Value("${sns.facebook.secret.key}") private String facebookSecretKey;  
  
    private static final String KAKAO_TOKEN_URI = "https://kauth.kakao.com/oauth/token";  
    private static final String KAKAO_INFO_URI = "https://kapi.kakao.com/v2/user/me";  
    private static final String KAKAO_VERIFY_URI = "https://kapi.kakao.com/v1/user/access_token_info";  
  
    private static final String NAVER_TOKEN_URI = "https://nid.naver.com/oauth2.0/token";  
    private static final String NAVER_INFO_URI = "https://openapi.naver.com/v1/nid/me";  
    private static final String NAVER_VERIFY_URI = "https://openapi.naver.com/v1/nid/verify";  
  
    @Bean  
    public FacebookConnectionFactory facebookConnectionFactory() {  
        return new FacebookConnectionFactory(facebookApiKey, facebookSecretKey);  
    }  
  
    @Bean  
    public OAuth2Parameters oAuth2Parameters() {  
        OAuth2Parameters param = new OAuth2Pareters();  
        param.setRedirectUri("https://eastglow.com/facebook/Join.do");  
  
        return param;  
    }  
  
    public String getCallbackUri(String type) throws Exception {  
        SecureRandom random = new SecureRandom();  
        String state = new BigInteger(130,random).toString();  
        String apiUrl = "";  
  
        if("naver".equals(type)){  
            apiUrl = "https://nid.naver.com/oauth2.0/authorize";  
            apiUrl += "?response_type=code";  
            apiUrl += "&client_id="+naverApiKey;  
            apiUrl += "&redirect_uri=https://eastglow.com/naver/Join.do";  
            apiUrl += "&state="+state;  
        }else if("kakao".equals(type)){  
            apiUrl = "https://kauth.kakao.com/oauth/authorize";  
            apiUrl += "?client_id="+kakaoApiKey;  
            apiUrl += "&redirect_uri=https://eastglow.com/kakao/Join.do";  
            apiUrl += "&response_type=code";  
            apiUrl += "&state="+state;  
            apiUrl += "&encode_state=true";  
        }  

        return apiUrl;  
    }  
  
    public ResponseEntity<Map> getNaverAccessToken(String code, String state) throws Exception {  
        RestTemplate restTemplate = new RestTemplate();  
  
        MultiValueMap<String, String> paramMap = new LinkedMultiValueMap<>();  
        paramMap.add("grant_type", "authorization_code");  
        paramMap.add("client_id", naverApiKey);  
        paramMap.add("client_secret", naverSecretKey);  
        paramMap.add("code", code);  
        paramMap.add("state", state);  
  
        URI uri = URI.create(NAVER_TOKEN_URI);  
  
        return restTemplate.postForEntity(uri, paramMap, Map.class);  
    }  
  
    public ResponseEntity<Map> getNaverMemberProfile(String accessToken) throws Exception {  
        RestTemplate restTemplate = new RestTemplate();  
        HttpHeaders headers = new HttpHeaders();  
  
        headers.add("Authorization", "Bearer " + URLEncoder.encode(accessToken, "UTF-8");  
        headers.add("Content-Type", MediaType.APPLICATION_FORM_URLENCODED_VALUE + ";charset=UTF-8");  
  
        URI uri = URI.create(NAVER_INFO_URI);  
        RequestEntity<String> rq = new RequestEntity<>(headers, HttpMethod.POST, uri);  
        ResponseEntity<Map> re = restTemplate.exchange(rq, Map.class);  
  
        return re;  
    }  
  
    public ResponseEntity<Map> getKakaoAccessToken(String code) throws Exception {  
        RestTemplate restTemplate = new RestTemplate();  
  
        MultiValueMap<String, String> paramMap = new LinkedMultiValueMap<>();  
        paramMap.add("grant_type", "authorization_code");  
        paramMap.add("client_id", kakaoApiKey);  
        paramMap.add("redirect_uri", httpsBaseUrl+kakaoCallbackUri);  
        paramMap.add("code", code);  
        paramMap.add("client_secret", kakaoSecretKey);  
  
        URI uri = URI.create(KAKAO_TOKEN_URI);  
  
        return restTemplate.postForEntity(uri, paramMap, Map.class);  
    }  
  
    public ResponseEntity<Map> getKakaoMemberProfile(String accessToken) throws Exception {  
        RestTemplate restTemplate = new RestTemplate();  
        HttpHeaders headers = new HttpHeaders();  
  
        headers.add("Authorization", "Bearer " + URLEncoder.encode(accessToken, "UTF-8");  
        headers.add("Content-Type", MediaType.APPLICATION_FORM_URLENCODED_VALUE + ";charset=UTF-8");  
  
        URI uri = URI.create(KAKAO_INFO_URI);  
        RequestEntity<String> rq = new RequestEntity<>(headers, HttpMethod.POST, uri);  
        ResponseEntity<Map> re = restTemplate.exchange(rq, Map.class);  
  
        return re;  
    }  
}
```
