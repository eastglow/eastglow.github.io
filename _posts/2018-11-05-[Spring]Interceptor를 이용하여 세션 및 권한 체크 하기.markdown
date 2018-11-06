---
layout: post
title:  "[Spring]Interceptor를 이용하여 세션 및 권한 체크 하기"
date:   2018-11-05 20:00:00
author: EastGlow
categories: Back-end
---
## 1. Spring에서 Interceptor란?

Interceptor는 Controller에 들어오는 요청(HttpRequest)과 응답(HttpResponse)를 가로채는 역할을 한다. 쉽게 말해 "eastglow.github.io/boardlist/10" 라는 URI를 사용자가 요청했을 때 그 요청과 받아주는 Controller가 반환하는 응답을 가로채는 녀석이다. 이와 비슷한 기능을 가진 Filter라는 것도 있는데 Interceptor와 다른 점은 실행되는 시점이다.

Interceptor는 DispatcherServlet이 실행된 후(= Controller로 가기 전), Filter는 DispatcherServlet이 실행되기 전에 호출된다.

## 2. 환경설정

### web.xml

```
<!-- dispatcher-servlet.xml -->
<servlet>
    <servlet-name>action</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/config/egovframework/springmvc/*.xml</param-value> 
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
```
먼저 web.xml에 관련 config.xml 파일을 정의해준다.


### egov-com-servlet.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:p="http://www.springframework.org/schema/p"        
        xmlns:context="http://www.springframework.org/schema/context"
        xmlns:oxm="http://www.springframework.org/schema/oxm"
        xmlns:mvc="http://www.springframework.org/schema/mvc"
        xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
                http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
                http://www.springframework.org/schema/oxm http://www.springframework.org/schema/oxm/spring-oxm-4.0.xsd
                http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd">
                
...

    <mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/*.do"/>
            <mvc:exclude-mapping path="/log*.do"/>
            <bean class="egovframework.egov.util.AuthInterceptor" />
        </mvc:interceptor>
    </mvc:interceptors>
</beans>
```

param-value에 써줬던 경로에 servlet.xml 파일을 정의해준다. 이름은 편하신데로... 나는 전자정부 프레임워크를 이용하고 있기 때문에 대부분의 패키지나 파일 이름에 egov가 붙어있다.

`<mvc:mapping path="" />` 는 Interceptor로 가로챌 URI를 명시해준다. *.do를 명시해주면 .do로 끝나는 모든 URI를 Interceptor로 가로채겠다는 것이다.

`<mvc:exclude-mapping path="" />`는 그 중에서 제외할 URI를 명시해주는 것이다. login, logout은 굳이 세션이나 권한 체크가 필요없기 때문에 제외해주었다.


### AuthInterceptor.java

```
public class AuthInterceptor extends WebContentInterceptor {

    /**
     * 세션에 계정정보(SessionVO)가 있는지 여부로 인증 여부를 체크한다. 계정정보(SessionVO)가 없다면, 로그인 페이지로 이동한다.
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws ServletException {
        SessionVO sessionVo = null;

        try {
            sessionVo = (SessionVO) SessionUtil.getSessionAttribute(request, "sessUser");

            if (sessionVo != null && sessionVo.getSessUserID() != null) {
                return true;
            } else {
                ModelAndView modelAndView = new ModelAndView("redirect:/forward.do");
                modelAndView.addObject("msgCode", "세션이 만료되어 로그아웃 되었습니다. 다시 로그인 해주세요.");
                modelAndView.addObject("returnUrl", "/login.do");
                throw new ModelAndViewDefiningException(modelAndView);
            }
        } catch (Exception e) {
            ModelAndView modelAndView = new ModelAndView("redirect:/forward.do");
            modelAndView.addObject("msgCode", "세션이 만료되어 로그아웃 되었습니다. 다시 로그인 해주세요.");
            modelAndView.addObject("returnUrl", "/login.do");
            throw new ModelAndViewDefiningException(modelAndView);
        }
    }

    /**
     * 세션에 메뉴권한(SessionVO)이 있는지 여부로 메뉴권한 여부를 체크한다. 계정정보(SessionVO)가 없다면, 로그인 페이지로 이동한다.
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        SessionVO sessionVo = null;
        String requestURI = request.getRequestURI();

        try {
            if (!requestURI.equals("/index.do")) {
                sessionVo = (SessionVO) SessionUtil.getSessionAttribute(request, "sessUser");

                if (sessionVo != null && sessionVo.getSessUserID() != null) { // 세션이 있을 경우만 체크
                    HashMap<String, Object> menuAuthMap = (HashMap<String, Object>) modelAndView.getModel().get("menuAuth");
                    String sessUserAuth = sessionVo.getSessUserAuth();
                    String menuCode = String.valueOf(menuAuthMap.get("menuCode"));
                    boolean authCheck = false;
                    StringTokenizer st = new StringTokenizer(sessUserAuth, ",");

                    while (st.hasMoreTokens()) {
                        String authCode = st.nextToken();
                        if (menuCode.equals(authCode)) {
                            authCheck = true;
                        }
                    }

                    if (!authCheck) {
                        ModelAndView mav = new ModelAndView("redirect:/forward.do");
                        mav.addObject("msgCode", "권한이 없습니다.");
                        mav.addObject("returnUrl", "/index.do");
                        throw new ModelAndViewDefiningException(mav);
                    }
                } else { // 세션이 없으면 로그인 페이지로 이동
                    ModelAndView mav = new ModelAndView("redirect:/forward.do");
                    mav.addObject("msgCode", "세션이 만료되어 로그아웃 되었습니다. 다시 로그인 해주세요.");
                    mav.addObject("returnUrl", "/login.do");
                    throw new ModelAndViewDefiningException(mav);
                }
            }
        } catch (Exception e) { // 그 외 예외사항은 index로 이동
            ModelAndView mav = new ModelAndView("redirect:/forward.do");
            mav.addObject("msgCode", "권한이 없습니다.");
            mav.addObject("returnUrl", "/index.do");
            throw new ModelAndViewDefiningException(mav);
        }
    }
}
```

WebContentInterceptor 라는 클래스를 상속 받은 AuthInterceptor.java의 소스이다. WebContentInterceptor는 HandlerInterceptor 인터페이스를 구현한 클래스이다.

HandlerInterceptor는 preHandle(), postHandle(), afterCompletion() 3가지 메서드를 가지고 있다. 이 메서드들을 오버라이딩하여 이용하면 된다.

> **preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)** : Controller로 가기 전에 실행되는 메서드이다. 주로 로그인 체크나 세션 존재 여부를 체크할 때 사용한다.
> 
> **postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)** : Controller로 가서 View로 가기 전에 실행되는 메서드이다. 나는 이 메서드로 메뉴 권한 체크를 해주고 있다. 메뉴 권한 체크를 위해서는 사용자가 요청한 URI에 대한 메뉴코드 값과 현재 로그인 한 사용자에게 허용된 메뉴코드 값을 알아야하기 때문에 Controller를 우선 타야한다고 생각하여 여기에 구현하였다.
> 
> **afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)** : View까지 모두 로드된 이후에 실행되는 메서드이다. 딱히 아직은 쓸 일이 없어서 구현해본 적은 없다.


## preHandle() - 세션 및 로그인 체크

세션 체크하는 소스를 postHandle()에 구현해두었다. 먼저 세션에 저장된 사용자 정보를 받아와서 그 세션 VO와 세션 안의 아이디가 유효한 지 체크한다. 유효하다면 문제 없이 Controller로 통과시켜도 되기 때문에 return true를 해주고 유효하지 않다면 login 페이지로 리다이렉트 시킨다. forward.do는 쉽게 설명하자면 하나의 공통 페이지인데 거기서 메시지 코드 값과 returnUrl을 받아서 메시지를 alert창으로 표시해주고 이후에 returnUrl로 이동시켜준다.

## postHandle() - 메뉴권한 체크

메뉴권한 체크는 사용자가 요청한 URI에 해당하는 컨트롤러가 가지고 있는 고유 메뉴코드가 필요하다. 그래서 Controller를 타기 전이 아닌, 탔을 때 체크를 하도록 postHandle()에 구현하였다. 세션 체크와 마찬가지로 먼저 세션값을 받아온다. 다만, index 페이지는 매뉴코드가 없기 때문에 request.getRequestURI()를 통해 요청받은 URI를 구해서 만약 그 URI가 "/index.do"면 따로 체크하지 않도록 하였다.

그 다음은 세션 체크와 마찬가지로 세션의 유효성부터 체크한 후 만약 유효하다면 ModelAndView를 통해 요청한 URI에 해당하는 Controller가 반환하는 값들을 받아온다. 거기서 menuAuth라는 메뉴 관련 정보 객체를 HashMap으로 담아주었다. 그 HashMap 안에는 "menuCode"라는 메뉴코드 값이 들어있으며, 그 값과 사용자의 세션 정보에서 가져온 메뉴 권한값을 서로 비교하여 만약 menuCode 값이 사용자의 세션 정보에 들어있는 메뉴 권한값에 있으면 통과시키고 없으면 index 페이지로 리다이렉트 시킨다.


Spring을 안 썼거나, 학창시절 같았으면 Controller마다 공통 코드를 넣어서 세션이나 권한 체크를 했을 것이다. 하지만 Spring에는 이미 이러한 기능을 돕기 위해 Interceptor라는 좋은 것이 있어서 이번 기회에 활용해보았다. 아직 100% 활용하고 있다고 할 순 없지만 그래도 나름 잘 적용된 듯 하다.
