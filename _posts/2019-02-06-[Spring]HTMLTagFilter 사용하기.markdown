---
layout: post
title:  "[Spring]HTMLTagFilter "
date:   2019-01-31 19:00:00
author: EastGlow
categories: Back-end
---

## HTMLTagFilter?
크로스 사이트 스크립팅(Cross-Site Scripting, XSS)은 쉽게 설명하자면 악의적인 스크립트 코드를 서버로 전송시켜 실행하여 사용자의 쿠키나 세션을 탈취하거나 이상한 페이지로 이동시키는 등의 행위를 하는 것이다.

이러한 XSS 공격은 어떤 프로젝트를 하든지 취약점 점검에 꼭 하나씩은 필터링 되곤 한다.  이러한 XSS 공격을 막기 위해서 보통은 사용자에서 입력받은 값을 서버로 넘길 때, `<script>`의 `<`를 `&lt;`로 치환해버리곤 한다. 또는 아예 on으로 시작하는 모든 행위들을(onclick, onsubmit, onfocus 등) 정규표현식으로 치환하거나... 등의 방법으로 막고 있다.

그러면 이렇게 치환해주는 것을 사용자 입력이 일어나는 모든 곳에 일일이 써줘야할까? 필터를 쓰지 않는다면 공통 함수로 만들어서 입력이 처리되는 Service Implement 단에서 치환해주는 부분에 써주거나 DB Insert 문에서 replace()과 같은 함수로 치환해주는 등의... 아주 귀찮고 무식한 방법을 써야한다. 그렇지 않고서야 모든 URI를 일일이 치환해줄 방법이 없다.

하지만 필터를 이용하면 이것이 가능하다. 예전에 권한 처리와 관련된 인터셉터를 다룬 글을 남긴 적이 있는데 필터와 인터셉터는 비슷해보여도 자세히 들여다보면 서로 다른 친구들이다. 여기에서는 HTMLTagFilter만을 다룰거기 때문에 둘의 대한 자세한 설명은 검색을 해보면 좋을 것 같다.

여튼, 노가다(?)를 하지 않기 위해 우리는 요청이 들어온 모든 URI를 감시(?)하여 모든 스크립트와 관련된 코드를 색출해낼 것이다. 그것을 위해 필요한 것이 필터이며, 필터 중에서도 HTMLTagFilter이다.

## 어떻게 사용할까?

Spring을 많이 접해본 사람들이라면 web.xml이 익숙할 것이고, web.xml을 열어보면 아래와 같은 코드가 항상 맨 위에 먼저 박혀있는 것을 볼 수 있을 것이다.

```
<filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>utf-8</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>encodingFilter</filter-name>
    <url-pattern>*.do</url-pattern>
</filter-mapping>
```

UTF-8 기반의 프로젝트에서 한글이 깨지지 않도록 Encoding Character Set을 미리 처리해주는 필터이다. `param-name`에 들어있는 encoding이라는 변수에 바로 아래의 `param-value`로 utf-8을 담아주고 있다. CharacterEncodingFilter Class를 들여다보면 바로 어떻게 동작하는지 알 수 있는데 기본적으로 encoding이라는 String 변수와 forceEncoding이라는 boolean 변수를 받아주고 있다.
`encoding` 변수는 이름에서 알 수 있듯이 설정할 Character Set의 이름을 넣어주면 되고 필수값이다. 없으면 예외가 발생한다. `forceEncoding`은 필수값은 아니며 true값을 주면 Controller에서 무슨 짓을 해도 무조건 `encoding`변수에서 넣은 Character Set을 따라간다.

encodingFilter는 이런식으로 사용하면 되고 HTMLTagFilter도 비슷하게 사용을 하면 된다.

### web.xml
```
<filter>
    <filter-name>HTMLTagFilter</filter-name>
    <filter-class>egovframework.test.egov.util.HTMLTagFilter</filter-class>
    <init-param>
        <param-name>excludePatterns</param-name>
        <param-value>/admin/*.do,/user/boardlist.do</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>HTMLTagFilter</filter-name>
    <url-pattern>*.do</url-pattern>
</filter-mapping>
```

앞서 설명했던 encodingFilter와 비슷한 구조이다. 조금 다른 점은 원래 `excludePatterns`라는 매개변수는 없는데 제외해주고 싶은 URI가 있어서 넣어주게 되었다. `/admin/*.do`는 /admin/으로 시작하는 모든 URI를 제외하겠다는 뜻이고 `/user/boardlist.do`는 이 URI만 제외하겠다는 뜻이다.

바로 아래 filter-mapping은 HTMLTagFilter를 `.do`로 시작하는 모든 URI에 적용하겠다는 뜻이다. 정리하자면
> .do로 끝나는 모든 URI에 HTMLTagFilter를 적용할건데, /admin/으로 시작하거나 /user/boardlist.do URI는 적용하지 않을 것이다.

라고 할 수 있겠다.

### HTMLTagFilter.java
```
public class HTMLTagFilter implements Filter {
    private FilterConfig config;
    private ArrayList<String> urlList;

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        String url = req.getServletPath();
        boolean allowedRequest = false;
        
        for(String excUrl : urlList){
            if(excUrl.contains("*.do")){
                String tmpUrl = excUrl.split("\\*.do")[0];
                if(url.contains(tmpUrl)){
                    allowedRequest = true;
                }
            }else{
                if(url.equals(excUrl)){
                    allowedRequest = true;
                }              
            }
        }

        if (!allowedRequest) {
            chain.doFilter(new HTMLTagFilterRequestWrapper((HttpServletRequest) request), response);
        } else {
            chain.doFilter(req, (HttpServletResponse) response);
        }
    }

    public void init(FilterConfig config) throws ServletException {
        String urls = config.getInitParameter("excludePatterns");
        StringTokenizer token = new StringTokenizer(urls, ",");
        
        urlList = new ArrayList<String>();
        
        while (token.hasMoreTokens()) {
            String url = token.nextToken();
            urlList.add(url);
        }
        
        this.config = config;
    }

    public void destroy() {

    }
}
```

위 코드가 실제 HTLMTagFilter가 작동되는 소스코드이다. 기본적으로 `init`을 통해 초기화해주고 있는데 아까 위에서 추가했던 `excludePatterns`를 불러와준다. 불러온 제외할 URI를 콤마(,)단위로 나눠준 다음, urlList라는 전역변수에 담아준다.

그런 다음 `doFilter`를 통해 실질적인 필터링을 거치게 된다. `init`에서 담아줬던 제외 URI 목록을 토대로 현재 요청받은 URI가 여기에 속해있는지 체크를 하게된다. 만약 속해있다면 `allowedRequest`에 true가 담길 것이고 그렇지 않다면 기본값인 false로 남게 된다. false로 남아있다면 다시 HTMLTagFilterRequestWrapper class에서 요청받은 URI에 대하여 치환이 이루어지게 된다. 아래 소스코드가 HTMLTagFilterRequestWrapper의 소스코드이다.

```
public class HTMLTagFilterRequestWrapper extends HttpServletRequestWrapper {

	public HTMLTagFilterRequestWrapper(HttpServletRequest request) {
		super(request);
	}

	public String[] getParameterValues(String parameter) {

		String[] values = super.getParameterValues(parameter);

		if (values == null) {
			return null;
		}

		for (int i = 0; i < values.length; i++) {
			if (values[i] != null) {
				StringBuffer strBuff = new StringBuffer();
				for (int j = 0; j < values[i].length(); j++) {
					char c = values[i].charAt(j);
					switch (c) {
						case '<':
							strBuff.append("&lt;");
							break;
						case '>':
							strBuff.append("&gt;");
							break;
						case '&':
							strBuff.append("&amp;");
							break;
						case '"':
							strBuff.append("&quot;");
							break;
						case '\'':
							strBuff.append("&apos;");
							break;
						default:
							strBuff.append(c);
							break;
					}
				}
				values[i] = strBuff.toString();
			} else {
				values[i] = null;
			}
		}

		return values;
	}

	public String getParameter(String parameter) {

		String value = super.getParameter(parameter);

		if (value == null) {
			return null;
		}

		StringBuffer strBuff = new StringBuffer();

		for (int i = 0; i < value.length(); i++) {
			char c = value.charAt(i);
			switch (c) {
				case '<':
					strBuff.append("&lt;");
					break;
				case '>':
					strBuff.append("&gt;");
					break;
				case '&':
					strBuff.append("&amp;");
					break;
				case '"':
					strBuff.append("&quot;");
					break;
				case '\'':
					strBuff.append("&apos;");
					break;
				default:
					strBuff.append(c);
					break;
			}
		}

		value = strBuff.toString();

		return value;
	}
}
```

보시다시피 <. >,   &, ", \를 다 치환해주고 있다. 이렇듯 필터를 통해 간단하게 모든 URI에 대하여 XSS 공격을 방어할 수 있게 된다. 기본적으로 전자정부 프레임워크에서 HTMLTagFilter를 제공하고 있으며 나도 이걸 조금 변형시켜(제외할 URI를 처리하는 부분) 만들어서 쓰는 중이다. 인터셉터나 필터는 매우매우 유용하니 잘 알아두고 쓰면 좋을 것이다.
