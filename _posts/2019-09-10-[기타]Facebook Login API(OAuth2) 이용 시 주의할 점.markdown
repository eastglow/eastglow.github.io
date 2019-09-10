---
layout: post
title:  "[기타]Facebook Login API(OAuth2) 이용 시 주의할 점"
date:   2019-09-10 13:00:00
author: EastGlow
categories: 기타
---

기존 사이트를 리뉴얼하는 작업을 진행하던 중 SNS 간편 로그인 기능(Facebook, Kakao, Naver)을 개발할 일이 있었다. 기존의 로그인은 ID, PW를 이용하여 DB에서 계정 정보를 조회해온 후 세션에 담아주고 있었다. (즉, 세션에 계정 정보 관련된 값이 없으면 로그인이 되어 있지 않다.)

SNS 간편 로그인 기능이 들어가게 되면 대략 아래와 같은 프로세스로 로그인이 바뀌게 된다.

1. SNS 로그인 하기를 클릭한다.
2. 그러면 SNS 로그인 창이 열린다.
3. SNS 로그인을 하게 되면 미리 설정해둔 Redirect URL로 이동하게 된다.
4. Redirect URL(이하 URL)은 현재 개발 중인 사이트의 로그인 프로세스가 실행되는 URL로 가게 되어있다.
5. URL에서 client_id, secret_key 등으로 Access Token, Refresh Token 등을 먼저 취득한 후, 이것을 이용하여 SNS 사용자 정보를 조회해온다.
6. 조회해온 SNS 사용자 정보를 토대로 현재 사이트에 회원가입 되어있는지 DB에서 체크한 뒤 있으면 로그인 프로세스를 계속 진행하고 없으면 간편 회원가입 페이지로 넘긴다.
7. 회원가입 프로세스는 제쳐두고 우선 로그인 프로세스를 진행하게 되면 DB에서 조회해온 사용자 정보를 세션에 담고난 뒤 메인 페이지로 redirect 시킨다. (Controller 단에서 return "redirect:index.do"와 같이 이동시킨다.)

대략 SNS 간편 로그인 프로세스는 위와 같다. 여기서 중요한 점이 Kakao, Naver와 달리 Facebook은 무조건 https 환경에서만 API를 호출할 수 있다. 그래서 로컬 환경에서 Open SSL을 이용하여 톰캣에 SSL 설정을 해주었고 443 포트로 정상적인 https 환경이 실행되는 것을 확인하였다.

Facebook API 호출도 잘 되는 것을 확인하였고 로그인 프로세스도 정상적으로 실행되었다. 그런데 문제가 하나 있었다. Facebook 로그인을 한 뒤에 로그아웃(기존의 세션을 제거하는 로직말곤 딱히 없다.)을 하고 나서 다른 방법들로(기존의 ID, PW 방식 로그인 혹은 Kakao, Naver 간편 로그인) 로그인을 하면 로그인이 제대로 안 되고 로그인 정보가 담긴 세션도 생성되지 않는 것이었다.

이유는 Facebook API를 통해 Redirect된 URL은 https이며 이 https로 시작하는 URL을 통해 로그인이 이루어지고 있었는데 세션 또한 https 환경에서 생성이 되게 된 것이다. 그런데 이 상태에서 로그아웃을 하면 https로 주소가 계속 남게 되고 로그인 페이지로 이동하면 https://test.co.kr/login.do 와 같이 이동하게 된다. (기존 사이트는 https가 적용되지 않은 상태이다. 개발 중이기 때문에 http로만 진행 중이었다.)

그래서 만약 다른 방식으로 로그인을 하게 되면 https가 계속 앞에 붙게 되고 로그인 프로세스 또한 https가 붙은 URL을 호출하여 타게 된다. 그런데 정작 로그인 후 redirect 되는 URL은 http://test.co.kr/index.do 와 같이 http가 붙은 URL로 가게 되어 있다. 때문에 세션이 없는 것이 당연했고 https://test.co.kr/index.do로 들어가면 세션이 다시 있는 것을 확인할 수 있었다.

이 문제를 해결하기 위해서는 2가지 정도의 방법이 있다.

1. 사이트 전체에 https를 적용한다. (사실 이게 보안상의 이유를 생각해봐도 가장 맞다.)
2. Spring Filter에 https에서 생긴 세션을 쿠기로 구운 후, 다시 http 세션을 생성해주는 클래스를 적용한다.

1번이 가장 쉽고 맞는 방법이지만 상황이 여의치 않다면 2번의 방법을 이용해야할 것이다. 내 경우는 우선 로컬 개발 환경에서 톰캣의 web.xml에 아래와 같은 내용을 넣어서 강제로 https 리다이렉트 되도록 해주었더니 해결되었다.

```
<security-constraint>
	<web-resource-collection>	
		<web-resource-name>SSL Forward</web-resource-name>	
		<url-pattern>/*</url-pattern>	
	</web-resource-collection>	
	<user-data-constraint>	
		<transport-guarantee>CONFIDENTIAL</transport-guarantee>	
	</user-data-constraint>	
</security-constraint>

<security-constraint>	
	<web-resource-collection>	
		<web-resource-name>HTTPS or HTTP</web-resource-name>
		<url-pattern>/resources/*</url-pattern>
	</web-resource-collection>	
	<user-data-constraint>	
		<transport-guarantee>NONE</transport-guarantee>	
	</user-data-constraint>	
</security-constraint>
```

상단의 소스는 SSL로 강제 리다이렉트 시켜주는 부분이고 하단의 소스는 JS, HTML 등 정적 소스들에 대하여 http, https 둘 다 호출할 수 있도록 해주는 부분이다.

돌이켜보면 되게 간단한 원인이었고 문제 해결도 쉬운 것이었는데 하루동안 뭐가 문제인지 헤매고 다녔다. 요약하자면 http와 https 간에는 서로 세션 공유가 안 된다는 점만 빨리 생각해냈다면 금방 해결했을 문제였다. 끝.
