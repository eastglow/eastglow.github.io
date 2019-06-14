---
layout: post
title:  "[Spring]Spring Boot에서 Internationalization 설정하기"
date:   2019-06-13 20:00:00
author: EastGlow
categories: Back-end
---

## 서론

Spring 환경에서 다국어 메시지를 처리하려면 보통 Message Property 파일을 언어별로 따로 두고 사용하곤 했다. 기존에 Spring이나 전자정부 프레임워크에서 사용할 때도 역시나 XML을 통해서 MessageSource설정을 통해 각 Locale에 맞는 messages.properties 파일을 읽어오고 JSP 상에서 `<spring:message>`태그를 통해 코드에 맞는 메시지를 보여주었다.

Spring Boot 역시 Spring이기 때문에 똑같이 사용하면 되는데 Boot는 이런 것들이 기본 설정이 다 되어 있기 때문에 엄청 편하게 쓸 수 있었다. 다만, 내가 좀 삽질을 해서 헤매긴 했지만... 어쨌든 지금은 잘 설정해서 사용 중이다.

## 본론

### MessageSourceAutoConfiguration.class

Spring Boot 라이브러리 중 `spring-boot-autoconfigure.jar`라는 파일이 있다. 이 안에 보면 `org.springframework.boot.autoconfigure.context`경로 밑에 `MessageSourceAutoConfiguration.class`를 볼 수 있는데 여기에 Internationalization와 관련된 Message 설정들이 담겨있다. Spring Boot가 참 좋은게 이런 기본 설정들이 너무 잘 돼 있다는 것이다. 때문에 Spring Boot가 기본 설정이 너무 편하긴 하지만 단순히 설정만 하고 넘어가는 것이 아니라 이런 jar파일 하나까지 다 열어보고 어떻게 설정이 되어 있나 살펴보는 것도 큰 도움이 된다고 생각한다.

아무튼 여기에 오면 prefix가 spring.messages로 시작하는 것들을 설정 관련된 property로 받아들인다고 써있는데 이 property들은 application.properties(혹은 yaml)에서 설정 가능하다.

나는 기본적으로 아래와 같은 설정들을 사용 중이다.
```
spring.messages.basename=message/messages
spring.messages.encoding=UTF-8
#spring.messages.cache-duration=0
#spring.messages.fallback-to-system-locale=false
```
첫번째 줄은 messages.properties 파일들이 위치할 경로이다. 저렇게 설정해두면 `resource/message/messages*.properties` 파일들이 검색된다. 여기서 내가 한가지 삽질한 게 basename을 따로 설정 안하면 messages 경로로 잡힌다고 써져있길래 `resource/messages/messages*.properties`에 넣고 테스트 해봤는데 계속 제대로 경로가 안 잡혀서 오류가 났었다.

Spring 공식 Doc도 보고 이곳저곳 구글링을 통해 알아보니 basename을 따로 설정 안하면 `resource/messages/messages*.properties` 경로가 아니라 `resource/messages*.properties` 이렇게 해야 정상적으로 잡힌다.

spring.messages.basename=eastglow/messages 로 잡는다면 `resource/eastglow/messages*.properties` 파일들이 잡히는 것이다. 디폴트 경로값을 잘못 이해해서 계속 삽질하다가 구글링과 MessageSourceAutoConfiguration.class 파일 및 기타 설정 코드들을 뜯어보고 나서야 제대로 이해하고 썼다. 참고로 여러 경로의 message 파일들을 잡으려면 basename=eastglow,eastglow2 등과 같이 콤마로 구분해주면 된다.

두번째 줄은 누구나 알 수 있듯이 기본 Encoding 값이다. 어차피 UTF-8로 설정되어 있기 때문에 굳이 안 해줘도 되는데 걍 명시적으로 해두는 게 마음이 놓여서 해두었다.

세번째와 네번째 줄은 굳이 해주지 않아도 되는 듯하여 그냥 참고만 할 수 있도록 주석처리 해두었다. 세번째 줄은 properties 파일들의 캐시 여부(일정 시간을 두고 캐시 리로딩을 할거면 설정해주면 된다.), 네번째 줄은 해당 Locale에 맞는 properties 파일이 없을 경우, 기본 디폴트 파일(messages.properties)을 쓸 것인가 말 것인가 하는 옵션이라고 한다.

### 설정은 끝났고 사용은 어떻게?

사실 Locale 값에 따라 다른 언어를 보여주게 하려면 Interceptor를 통해 Locale 값을 바꿔주는 부분을 추가해줘야 하는데 나는 굳이 필요없어서 하지 않았다. (나는 나라, 언어에 따른 메시지 처리가 아닌 코드를 정의한 properties파일을 통해 단순 메시지 처리를 하려고 했기 때문에)

이부분에 대해서는 아주 정말 잘 정리한 글이 있어서 참고 링크를 남겨두니 보면 좋을 것 같다.
>참고 : [https://okky.kr/article/364038](https://okky.kr/article/364038)

아무튼 Locale 변경에 관련된 Interceptor 설정을 따로 하지 않는다면 한국에서는 기본으로 `ko_KR`로 잡히게 된다. messages_ko_KR.properties가 최우선으로 잡히게 되며 이 파일이 없으면 messages.properties가 잡히게 된다.

application.properties에서 기본적인 설정을 다 마쳤다면 이제 resource 경로 아래에 자신이 설정한 basename 경로에 따라 properties 파일들만 잘 생성해주면 모든 준비는 끝난다. MessageSource는 이미 Boot에서 기본적으로 @Bean을 통해 생성해주기 때문에 따로 @Bean을 통해 생성해줄 필요가 없다. (Override를 통한 추가적인 설정을 위해서가 아니라면 말이다.)

#### Java에서 사용하기

##### messages.properties
```
error.404=페이지를 찾을 수 없습니다.
error.500=서버 내부의 오류로 인해 서비스를 제공할 수 없습니다.
```

##### CustomErrorController.java
```
import javax.servlet.RequestDispatcher;
import javax.servlet.http.HttpServletRequest;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.web.servlet.error.ErrorController;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;

import com.sogood.common.HandlerInterceptor;

@Controller
public class CustomErrorController implements ErrorController {
 
    private static final Logger LOGGER = LoggerFactory.getLogger(HandlerInterceptor.class);
	
    private static final String ERROR_PATH = "/error";
     
    @Autowired
    private MessageSource messageSource;
    
    @Override
    public String getErrorPath() {
        return ERROR_PATH;
    }
    
    @RequestMapping("/error")
    public String handleError(HttpServletRequest request, Model model) {
        Object status = request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
        
        LOGGER.info("===================");
        LOGGER.info(messageSource.getMessage("error.404", null, LocaleContextHolder.getLocale()));
        LOGGER.info("===================");
        
        model.addAttribute("code", status.toString());
        
        return "error";
    }
 
}
```

위 Controller는 Spring Boot App 내에서 Error 처리를 위해 만든 Controller이다. Boot는 기본적으로 /error URI에 대해서 error.jsp (JSP를 사용했을 경우)를 찾아가도록 설정되어 있었다. 이것을 따로 Controller로 만들어서 에러 코드라든지 기타 정보들을 model에 담아 내려줄 수 있도록 Controller를 따로 만들어서 적용하였다. ErrorController를 implements하여 구현하면 된다.

위와 같이 설정해주고 /error를 받는 메서드에서 Log를 찍어보면 아래와 같이 나오는 것을 확인할 수 있다.

```
20190614 15:34:05.601 [http-nio-80-exec-4] INFO c.s.c.HandlerInterceptor - =================== 
20190614 15:34:05.601 [http-nio-80-exec-4] INFO c.s.c.HandlerInterceptor - 페이지를 찾을 수 없습니다. 
20190614 15:34:05.601 [http-nio-80-exec-4] INFO c.s.c.HandlerInterceptor - =================== 
```

#### JSP에서 사용하기

현재 진행 중인 프로젝트가 JSP를 View로 사용하기 있기 때문에 JSP를 기반으로 설명하겠다. JSP는 Spring Tag Library를 이용하면 된다. `<%@ taglib prefix="spring" uri="http://www.springframework.org/tags"%>`를 JSP 상단에 추가하면 spring과 관련된 태그들을 사용할 수 있게 된다.

메시지 출력을 원하는 부분에 `<spring:message code="error.${code}"/>`와 같이 써주면 위에서 썼던 CustomErrorController의 /error에 매핑되는 메서드에서 내려준 code에 맞는 메시지를 보여준다. 만약에 code 값이 404라면 error.404 코드에 매핑되는 메시지를 불러와서 보여주게 된다.

## 끝

경로 설정 때문에 많은 시간을 삽질로 보내고(...) 겨우 해결다. 확실히 실제 설정이 이루어지는 Spring Boot 설정 파일을 직접 뜯어보고 어떻게 돌아가는지 확인하고 나니 더욱 이해가 잘 되는 듯 하다. 기존 Spring에서는 메시지 설정을 하려면 XML 설정도 해야하고 엄청 귀찮았는데 간편하게 할 수 있어서 정말 좋은 것 같다.
