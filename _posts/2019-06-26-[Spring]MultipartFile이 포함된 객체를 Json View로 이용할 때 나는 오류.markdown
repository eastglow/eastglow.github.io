---
layout: post
title:  "[Spring]MultipartFile이 포함된 객체를 Json View로 이용할 때 나는 오류"
date:   2019-06-26 13:00:00
author: EastGlow
categories: Back-end
---

## 오류의 시작

전자정부 프레임워크 3.8 버전으로 프로젝트를 진행 중인데 Json 데이터를 return 해줄 일이 생겼다. 그냥 Spring Boot에서 하던 것처럼 `@ResponseBody` 혹은 `@RestController`를 선언한 뒤 Map에 담아서 return 해줬는데 계속 오류가 나면서 안 되는 것이다.

찾다찾다 전자정부 프레임워크 포털에 가서 Q&A를 찾아보니 3.8 버전부터는 아래와 같이 jsonView Bean을 설정하고 ModelAndView를 이용해서 사용해야한다고 적혀있었다. 아래 코드를 servlet 설정 관련 xml에 적어준다.

### egov-com-servlet.xml
```
<bean id="jsonView" class="org.springframework.web.servlet.view.json.MappingJackson2JsonView">
	<property name="contentType" value="text/html;charset=UTF-8"/>
</bean>
```
### pom.xml
```
<dependency>
	<groupId>com.fasterxml.jackson.core</groupId>
	<artifactId>jackson-databind</artifactId>
	<version>2.9.9</version>
</dependency>
<dependency>
	<groupId>com.fasterxml.jackson.core</groupId>
	<artifactId>jackson-core</artifactId>
	<version>2.9.9</version>
</dependency>
<dependency>
	<groupId>com.fasterxml.jackson.core</groupId>
	<artifactId>jackson-annotations</artifactId>
	<version>2.9.9</version>
</dependency>
```

### TestController.java
```
// Controller에서 사용 시
@RequestMapping(value={"/test"})
@ResponseBody
public ModelAndView test(HttpServletRequest request, HttpServletResponse response, HttpSession session) throws Exception{
	ModelAndView mav = new ModelAndView("jsonView");
	Map<String, Object> resultMap = new HashMap<>();
	
	resultMap.put("code", "200");
	resultMap.put("msg", "test");
	
	mav.addObject("resultMap", resultMap);
	
    return mav;
}
```

위처럼 작성해서 돌려보면 Json 형태로 데이터를 잘 떨어뜨려주는 것을 확인할 수 있었다. Spring Boot에서는 기본적으로 `spring-boot-starter-web`에서 `@RestController`나 `@ResponseBody`를 통해 리턴할 타입을 보고 알아서 MessageConverter에 매핑해주는 세팅이 되어 있다.

Json 형태는 `MappingJackson2HttpMessageConverter`를 통해 변환된다고 하는데 Spring Boot가 아닌, 일반 Spring이나 전자정부 프레임워크는 이런 설정이 안 되어 있어서 제일 처음에 그냥 `@ResponseBody`를 달고 return했을 때 오류가 났었던 것이다. Spring Boot를 쓰다가 그냥 Spring을 써보니 손이 되게 많이 간다고 생각되었다.-_-

한가지 추가로 MessageConverter가 동작할 수 있도록 도와주는 라이브러리가 위에 적어둔 `com.fasterxml.jackson.*`라이브러리들이다. 그러니 꼭 pom.xml에 추가해주고 Json return을 사용하도록 한다.

## 근데 오류가 또난다?

첨부파일이 없는 화면에서는 글을 저장하고 Ajax를 통해 Java단에서 DB 입력 관련 프로세스를 처리한 후, 결과값을 JsonView를 이용하여 return 해주어도 문제가 없었다.

그런데 `input type="file"`이 들어간 화면에서는 첨부파일이 있든 없든 Java단에서 프로세스를 다 거치고 결과값을 JsonView를 이용하여 return 하면 아래와 같은 오류 메시지가 나며 return이 안 되었다.

```
No serializer found for class java.io.ByteArrayInputStream and no properties discovered to create BeanSerializer multipartfile
```

이리저리 찾아보니 Json 형태로 return해줄 때, Json 형태로 바꿀 객체의 필드들은 getter를 가지고 있어야 한다. getter가 없으면 오류가 난다고 하였고 `input type="file"`에 해당하는 MultipartFile 타입은 getter, setter를 만들어줬는데도 오류가 나고 있었다. 아마 file 형태이다보니 Json 형태로 변환할 수가 없어서 나는 듯하다. 추가로 이 필드는 굳이 Json 형태로 return 해줄 필요도 없다.

아니, 제일 중요한건 내가 Json 형태로 return 해줄 객체는 MultipartFile 필드가 들어가는 VO 객체가 아니라 그냥 일반 HashMap<String, Object> 객체였다.

### 1) 왜 HashMap만을 ModelAndView에 addObject하였는데 VO 객체 값들이 같이 세팅되려할까?

정확한 답은 얻을 수 없었지만 추측으로는 아래 소스에서 Ajax를 통해 넘어온 Form 객체가 BoardVO에 매핑되면서 자동으로 적재(?)되는게 아닌가 싶다.

```
public ModelAndView boardProcess(BoardVO boardVO, ...){
    ModelAndView mav = new ModelAndView("jsonView");
    Map<String,Object> resultMap = new HashMap<>();
    ...
    ...
    mav.addObject("resultMap", resultMap);

    return mav;
}
```

보시다시피 return 해주려는 ModelAndView에는 resultMap만 추가했다. 그런데 오류 메시지에는 BoardVO 안에 있는 MultipartFile 필드가 계속 Serializer가 없다면서 메시지가 찍혔다.

ModelAndView를 return 하기 전에 `clear()`하고 resultMap만 `addObject`해봐도 똑같고 return 하기 전에 ModelAndView에 적재된 Model을 `getModel()`을 통해 로그로 찍어봐도 resultMap만 나올 뿐이었다.

### 2) 그래서 어떻게 해결하였는가?

찾아보니 `com.fasterxml.jackson.*`라이브러리에서 Annotation을 하나 제공하는게 있는데 바로 `@JsonIgnore`이다.  이 Annotation의 역할을 찾아보니 이것을 선언한 필드는 Json 형태로 변환할 때 무시하고 넘어간다고 한다.

이것을 보자마자 딱 해결할 수 있는 실마리를 찾았다고 생각하였고 BoardVO 안에 있는 MultipartFile 필드 위에 이 Annotation을 선언해주니 그제서야 오류가 나지 않고 제대로 Json 객체가 return 되었다.

## 끝

Spring Boot에서는 별 설정없이 그냥 `@ResponseBody`나 `@RestController`로 Json 형태로 return 받는 것을 간단하게 할 수 있었는데 Spring에서는 예상치 못한 오류를 맞게 되어 하루동안 헤맨 것 같다. 한번 겪어봤으니 다음에는 같은 문제로 헤매지 않을 것 같다...^^
