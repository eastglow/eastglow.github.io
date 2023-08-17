---
layout: post
title:  "[Java]Datadog Span Tags (feat. Datadog 도입썰)"
date:   2023-08-17
author: EastGlow
categories: Back-end
---

# 짧게 적어보는 Datadog 도입.ssul

올해 상반기부터 사내 APM 솔루션으로 데이터독을 도입하게 되었다. 이전에는 핀포인트를 사용하고 있었는데 무료인 점에서는 충분히 매력적이나 데이터독의 기능이 너무 막강하여 안 넘어갈 수가 없었다.

> 라고 썼지만 이러한 사내 솔루션 도입은 인프라를 전담하는 팀에서 사전에 시장 조사를 하고 여러 솔루션을 나열하고 그 중에 하나를 골라서 PoC를 진행하여 도입하는거라 사실 보통 개발자들의 의지(?)와는 크게 관련이 없다.

여튼, 좋은게 좋은거라고 새로운 솔루션이 도입 됐으니 안 써볼 수가 없었다. 도입 초기엔 6월에 오픈할 하나의 큰 서비스 때문에 바빴기 때문에 바로 우리팀에서 운영하는 애플리케이션에 적용하진 않았고 어느정도 일이 마무리 된 6월 초 정도에 최초로 데이터독 설정을 적용했던 것으로 기억한다.

확실히 이것저것 수집하는 지표나 수치 값들이 많아서 그런지 엄~청 다양한 기능들이 있었다. 너무나 많기도 하고 듣자하니 데이터독은 기능 하나하나마다 별도의 유료 요금제로 운영되고 있어서 모든 기능을 사용하진 않고 우선은 필요한 기능들 위주로 요금을 지불하여 이용한다고 들었다. (정말 잘 만들긴 했으나 사용자 입장에서는 뭐 하나 하려고해도 다 돈이라 부담스럽긴 하다;)

그 중에서도 자주 쓰는 기능은 크게 아래와 같이 있는 것 같다.

 - Dashboards: 입맛에 따라서 각종 지표 및 수치 값들을 꾸며서(?) 하나의 대시보드에 맞춰서 넣을 수 있는 기능이다. 서비스하는 애플리케이션의 RPS, Latency, Resource Usage 등등 전반적인 지표들을 입맛에 맞게 하나의 대시보드로 만들어서 한눈에 딱 파악할 수 있게 만들어서 사용하고 있다. 기존 핀포인트로는 바로 눈에 보이는 정보로는 RPS정도만 볼 수 있었는데 엄청 다양한 것들을 볼 수 있게 되어서 서비스 운영에 큰 도움이 되는 듯 하다.
 - APM
	 - Traces: 각 요청에 대한 상세 정보(?)를 볼 수 있는 곳이다. 어느 URL로 요청이 들어왔고 어떤 서블릿, 컨트롤러, 메소드에서 처리가 되었고 시간이 얼마나 걸렸고 어떤 쿼리가 실행되었고 등등 하나의 요청에 대한 전반적인 정보들을 담고 있어서 특정 요청에 대한 분석을 하는 것에 도움을 주는 기능이다. 다만, 아직 사내에 적용된 정책은 요금 이슈 때문에 전체 요청을 모두 수집하고 있진 않고 특정 비율, 조건에 따라서 샘플링 된 일부 요청들만 수집이 되고 있어서 간혹 이 기능에서 나오지 않는 요청들도 있다. ㅠㅠ
	 - Services: 하나의 서비스, 애플리케이션, 데이터베이스 등 우리가 흔히 서비스라고 부르는 단위에 대해서 전반적인 정보를 제공하는 기능이다. 앞서 대시보드에서 언급했던 리소스 사용량이나 요청수, 에러수 등등 정말 다양한 정보를 품고 있다.
	 - Data Streams Monitoring: 카프카와 같은 데이터 처리 플랫폼과 연관된 애플리케이션은 이 메뉴에서 모니터링이 가능하다고 한다. 아직 나도 그냥 눈팅(?)하는 정도로 밖에 사용을 안 하고 있는데 카프카와 연동된 컨슈머 애플리케이션을 뭔가 집중적으로 모니터링 할 수 있는 그러한 환경이나 기능은 조금 미비한 듯 하다. 실제로도 이 기능은 기능명 뒤에 자그맣게 "beta"라는 표시가 붙어있기도 하다.
 - Monitors: 데이터독에서 최고로 유용하게 사용하고 있는 기능이 아닐까 싶다. 이 기능에서는 각종 수치나 지표들을 기준으로 특정 조건을 넘어서면 알림이 오게 할 수 있다. 예를 들어서 "지난 10분 동안 요청에 대한 처리 지연시간이 5초 이상 넘어가는 요청이 존재하는 경우 알림을 보내줘"라는 조건이라든가, "요청 대비 오류 발생률이 0.01% 이상인 경우 알림을 보내줘"와 같이 여러가지 상세 조건을 걸어서 알림을 받아볼 수 있다. 그리고 이번 글에서 주요하게 다룰 Span Tags를 통해서도 알림을 걸 수 있는데 이건 아래에서 다시 설명하겠다.


# Span Tags

데이터독의 Traces 기능에서는 수집한 요청의 각종 정보들에 대해서 일종의 태그 형태로 수집하여 가지고 있게 되는데 이것을 사용자가 원하는 특정 정보, 값 등을 대입하여 끼워넣을 수가 있다.

쉽게 말해 A라는 API에 로그로 남기던 정보를(예를 들어 API 처리 결과로 만들어진 Response에 들어있는 특정 필드의 값) 태그로 세팅해주면 데이터독은 이를 수집하여 Traces 기능에서 보여준다는 것이다. 백문이 불여일견이라고 아래 스크린샷을 보도록 하자.

![](/assets/post/20230817_1.png)

위 스크린샷에서 볼 수 있듯이 다양한 정보들이 태그 형태로 요청 정보에 세팅이 되었고 데이터독 내에서는 위와 같이 표시가 된다. 내가 세팅한 태그는 `result_code`, `result_msg`라는 필드이며 이것은 API의 처리가 완료되고 난 뒤 Response DTO로 리턴되는 객체 안에 세팅된 필드값들이다. 이 태그를 통해서 나는 요청 trace를 눌러보기만 해도 이 요청은 어떻게 처리가 되어서 완료가 되었는지를 이 result 필드값을 통해 대략적으로 알 수가 있게 되는 것이다.

![](/assets/post/20230817_2.png)

추가로 위 스크린샷처럼 `result_code`에 대한 통계값도 파이 차트를 통해서 손쉽게 눈으로 볼 수 있게 대시보드를 구성할 수도 있다. 이외에도 아까 말한 Monitors 기능에서 이 `result_code` 중 특정 값의 비율이 전체 요청수 대비 많아진 때가 있다면 알림을 받아볼 수도 있다.


## 실제 적용은 어떻게?

Datadog "[Adding tags](https://docs.datadoghq.com/tracing/trace_collection/custom_instrumentation/java/)"와 관련된 문서를 보면 그냥 아래와 같이 코드를 써주면 현재 활성화된 Span에 태그가 세팅된다고 한다.

```java
@WebServlet
class ShoppingCartServlet extends AbstractHttpServlet {
    @Override
    void doGet(HttpServletRequest req, HttpServletResponse resp) {
        // Get the active span
        final Span span = GlobalTracer.get().activeSpan();
        if (span != null) {
          // customer_id -> 254889
          // customer_tier -> platinum
          // cart_value -> 867
          span.setTag("customer.id", customer_id);
          span.setTag("customer.tier", customer_tier);
          span.setTag("cart.value", cart_value);
        }
        // [...]
    }
}
```

실제로 위 방법대로 아주 간단하게 태그를 세팅할 수 있다. 하지만 이것은 매번 필요한 곳에서 `GlobarTrace.get().activeSpan()`을 호출하여 Span 객체를 얻어야하는 번거로움도 있고 태그 세팅이 필요한 곳들에 대해서 일괄적으로 적용하기엔 무리가 있는 코드이다.

그래서 나는 필터를 이용하여 특정 URL 패턴에 해당되는 API들에 대해서만 태그 세팅이 대상이 되도록 적용하고 미리 yml에 정의해둔 특정 필드명을 가졌을 경우에만 최종적으로 `setTag`가 작동하도록 만들기로 하였다.


### 1) 필요한 정보들은 yml에 세팅

```yaml
custom:
	datadog:
		enabled: true
		include-url-patterns:
			- /eastglow/*
		target-fields:
			- resultCode,resultMsg
```

먼저 yml에 필터를 적용할 URL 패턴과 어떤 필드를 태그로 세팅할 지에 대한 필드명을 명시해주도록 한다. enabled 같은 경우는 명시적으로 true or false를 써두도록 하여 실제로 데이터독 커스텀 필터와 관련된 Bean들이 초기화될 때 이 값을 바라보고 초기화될 지 말 지 결정하도록 하는 용도로 쓰고 있다. 자세한건 이따 나올 필터 관련 클래스에서 보도록 하자.


### 2) yml 정보를 토대로 initialize

#### CustomDatadogProperties.java
```java
@Getter
@Setter
@ConfigurationProperties(prefix = "custom.datadog")
public class CustomDatadogProperties {

	private List<String> includeUrlPatterns = new ArrayList<>();
	private List<String> targetFields = new ArrayList<>();
	private boolean enabled = false;
}
```

yml값을 불러와서 Properties로 만들어주는 클래스이다. 아까 설정한 값들을 필드로 품고 있다.


#### CustomDatadogPropertiesInitializer.java
```java
@Configuration
@ConditionalOnProperty(value = "custom.datadog.enabled", havingValue = "true")
@EnableConfigurationProperties(CustomDatadogProperties.class)
public class CustomDatadogPropertiesInitializer {

	private Map<String, String> targetFieldMap = new HashMap<>();

	@Getter
	private String podName = "";

	@Getter
	private List<String> includeUrlPatterns = new ArrayList<>();

	public CustomPromDatadogPropertiesInitializer(CustomDatadogProperties customDatadogProperties) {
		initialize(customDatadogProperties);
	}

	private void initialize(CustomDatadogProperties customDatadogProperties) {

		List<String> targetFields = customDatadogProperties.getTargetFields();

		targetFields.stream()
				.flatMap(fields -> Arrays.stream(fields.split(",\\s*")))
				.map(String::trim)
				.forEach(field -> targetFieldMap.put(field, convertToUnderscore(field)));

		includeUrlPatterns = customDatadogProperties.getIncludeUrlPatterns();

		try {
			String hostname = InetAddress.getLocalHost().getHostName();
			if (StringUtils.isNotBlank(hostname)) {
				podName = hostname;
			}
		} catch (UnknownHostException e) {
			podName = "알수없음";
		}
	}

	public String getTagName(String key) {
		return targetFieldMap.get(key);
	}

	private String convertToUnderscore(String str) {

		try {
			return str.replaceAll("([a-z])([A-Z])", "$1_$2").toLowerCase();
		} catch (Exception e) {
			return str;
		}
	}
}
```

property들을 실제로 초기화해주는 클래스이다. 먼저 필드명 값들을 콤마 기준으로 나누고 공백을 제거한 뒤, 필드명을 카멜 케이스에서 언더스코어 케이스로 치환해준다. (케이스 치환해주는 부분은 일종의 팀 네임 컨벤션이라 생략해도 상관없다.)

그리고나서 미리 만들어둔 Map에다가 원래의 필드명은 key, 언더스코어 케이스로 치환한 필드명은 value로 넣어준다.

이후 필터를 적용할 URL 패턴을 담아주고 마지막으로 현재 애플리케이션이 올라가있는 파드명을 세팅해주면 초기화 과정이 끝이 난다.

참고로 `@ConditionalOnProperty`때문에 `custom.datadog.enabled` 값이 `true`여야만 위 클래스가 정상적으로 Bean으로 초기화된다.


#### CustomDatadogAutoConfiguration.java
```java
@Configuration
@ConditionalOnBean(CustomDatadogPropertiesInitializer.class)
public class CustomDatadogAutoConfiguration {

	@Bean
	@DependsOn("customDatadogPropertiesInitializer")
	public FilterRegistrationBean<CustomDatadogFilter> customDatadogFilter(ObjectMapper objectMapper, CustomDatadogPropertiesInitializer customDatadogPropertiesInitializer) {

		List<String> includeUrlPatterns = customDatadogPropertiesInitializer.getIncludeUrlPatterns();
		FilterRegistrationBean<CustomDatadogFilter> registrationBean = new FilterRegistrationBean<>();

		registrationBean.setFilter(new CustomDatadogFilter(objectMapper, customDatadogPropertiesInitializer));

		if (!includeUrlPatterns.isEmpty()) {
			registrationBean.addUrlPatterns(includeUrlPatterns.toArray(new String[includeUrlPatterns.size()]));
		}

		registrationBean.setName("customDatadogFilter");
		return registrationBean;
	}
}
```

이부분이 실질적으로 필터를 초기화해주는 부분이다. 다만, 여러가지 전제조건들이 달려있다.

첫번째로 CustomDatadogPropertiesInitializer 클래스가 Bean으로 존재하고 있어야 한다. (@ConditionalOnBean)
둘째로  CustomDatadogPropertiesInitializer가 먼저 초기화되고 난 뒤에 필터가 초기화되어야 한다. (@DependsOn)

그렇게 모든 전제조건을 만족하고 나면 실제 필터의 초기화가 이루어질텐데 아까 앞에서 초기화 된 값들을 이용하고 있다. 먼저 CustomDatadogFilter 클래스를 FilterRegistrationBean에다가 `setFilter`로 설정해주고 그 다음 URL 패턴을 받아와서 필터가 적용될 패턴으로 등록해준다. 그러면 이제 필터가 적용될 환경이 모두 갖추어졌다.


### 3) 실제로 적용될 필터

#### CustomDatadogFilter.java
```java
@Slf4j
@RequiredArgsConstructor
public class CustomDatadogFilter extends OncePerRequestFilter {

	private final ObjectMapper objectMapper;
	private final CustomDatadogPropertiesInitializer customDatadogPropertiesInitializer;
	private final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

	public static final String TAG_PREFIX = "params.";

	@Override
	protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain filterChain) throws ServletException, IOException {

		ContentCachingRequestWrapper cacheReq = new ContentCachingRequestWrapper(req);
		ContentCachingResponseWrapper cacheRes = new ContentCachingResponseWrapper(res);

		filterChain.doFilter(cacheReq, cacheRes);

		Map<String, Object> paramMap = new HashMap<>();

		try {
			Map<String, Object> reqMap = parseRequestBody(cacheReq);
			Map<String, Object> resMap = parseResponseBody(cacheRes);

			paramMap.putAll(cacheReq.getParameterMap());
			paramMap.putAll(reqMap);
			paramMap.putAll(resMap);
		} catch (Exception e) {
			// parse 도중 에러가 나면 paramMap을 비워둔 채로 마무리 한다.
		}

		final Span span = GlobalTracer.get().activeSpan();

		if (span != null) {

			paramMap.forEach((key, value) -> {

				String tagName = customDatadogPropertiesInitializer.getTagName(key);
				if (StringUtils.isNotBlank(tagName)) {
					span.setTag(TAG_PREFIX + tagName, String.valueOf(value));
				}
			});

			span.setTag(TAG_PREFIX + "pod_name", customDatadogPropertiesInitializer.getPodName());

		}

		cacheRes.copyBodyToResponse();
	}

	private Map<String, Object> parseRequestBody(ContentCachingRequestWrapper cacheReq) {

		try {
			byte[] buf = cacheReq.getContentAsByteArray();

			if (buf.length > 0) {

				Map<String, Object> resultMap = new LinkedHashMap<>();
				Map<String, Object> resMap = objectMapper.readValue(buf, new TypeReference<Map<String, Object>>() {
				});

				resMap.entrySet().forEach(res -> {
					String key = res.getKey();
					Object value = res.getValue();

					resultMap.put(key, convertToString(value));
				});

				return resultMap;
			}
		} catch (Exception e) {
			log.error("CustomDatadogFilter parseRequestBody exception", e);
		}

		return Collections.emptyMap();
	}

	private Map<String, Object> parseResponseBody(ContentCachingResponseWrapper cacheRes) {

		try {
			byte[] buf = cacheRes.getContentAsByteArray();

			if (buf.length > 0) {

				Map<String, Object> resultMap = new LinkedHashMap<>();
				Map<String, Object> resMap = objectMapper.readValue(buf, new TypeReference<Map<String, Object>>() {
				});

				resMap.entrySet().forEach(res -> {
					String key = res.getKey();
					Object value = res.getValue();

					resultMap.put(key, convertToString(value));
				});

				return resultMap;
			}
		} catch (Exception e) {
			log.error("CustomDatadogFilter parseRequestBody exception", e);
		}

		return Collections.emptyMap();
	}

	private String convertToString(Object value) {

		try {
			if (value instanceof Date) {
				Date date = (Date) value;
				return formatter.format(date.toInstant().atZone(ZoneId.systemDefault()).toLocalDateTime());
			} else if (value instanceof Number || value instanceof String || value instanceof Boolean) {
				return String.valueOf(value);
			}
		} catch (Exception e) {
			log.error("CustomDatadogFilter convertToString exception", e);
		}

		return "";
	}
}
```

크게 복잡한 로직이 있는 필터는 아니기에 코드에 대한 긴 설명은 없어도 될 것이라 생각된다. 그래서 짤막하게 추가 설명이 필요할만한 부분들에 대해서만 짚고 넘어가겠다.

##### OncePerRequestFilter
우선 OncePerRequestFilter를 상속받은 이유는 원래 GenericFilterBean 이것을 상속받아서 구현을 했었는데 이상하게 자꾸 필터가 2번씩 실행이 되는 것이었다. 찾아보니 기본적인 필터를 상속받아 구현하게 되면 서블릿 요청에 따라 1번보다 많게 필터가 실행될 수 있다...라는 원인1과 스프링부트 AutoConfiguration 어딘가에서 필터 Bean이 자동으로 먼저 등록이 되고 있어서 내가 커스텀하여 등록하는 필터 Bean을 코드로 생성해두면 2번 중복으로 피터가 등록되는 원인2가 있었는데 이게 좀 된 일이라 정확한 원인은 기억이 안 난다.(;;)

아무튼 무언가의 이유로 오직 한번의 필터 실행만을 보장하는 OncePerRequestFilter를 상속받아 구현해두게 되었다.

##### ContentCachingRequestWrapper & ContentCachingResponseWrapper
req&res의 경우, 한번이라도 그 내용을 취하게 되면 그 다음부터는 사용할 수 없게 된다고 한다. 그래서 Caching Wrapper로 감싸서 반복하여 재사용 가능하도록 만들어서 사용하도록 하였다.

##### convertToString
convertToString에서는 Date, String, Number, Boolean 형태일 때만 변환하도록 해두었다. 이유는 이것들 이외에 Byte, Collection(List, Map...), 기타 등등의 타입들은 String으로 표현할 수 없거나 표현하기가 애매한 타입들이 대부분이기 때문에 대표적으로 사람이 읽을 수 있는 정보로 표현할 수 있는 타입들만 변환하도록 했다.

##### 계층형 구조로 Tag를 세팅하는 법?
위 코드를 보다보면 `TAG_PREFIX`를 달아준 것을 볼 수 있을텐데 이렇게 해두면 최종적으로 태그명은 `params.result_code`와 같이 들어가게 된다. 이렇게 해야 아까 위에서 봤던 스크린샷에서 처럼
```json
params {
	result_code: 99
}
```
이런식으로 Map 형태? 계층형 구조?로 태그가 나오게 된다. 처음엔 이걸 몰라서 엄청 삽질을 했었다; (공식 Docs에도 안 적혀있는 것 같...)

## 마치며...

여기까지 데이터독의 Tag를 세팅하는 배경설명, 과정, 실제 소스코드 설명 등을 정리해보았다. 아직 데이터독 자체를 도입한 지 얼마 안 됐기 때문에 완전 깊게 사용해본건 아니라 모자란 부분이 많지만 점차 사용을 많이 해보면서 좀 더 이 솔루션 활용도를 높일 수 있을 것으로 생각된다.

아주 다양한 기능과 설정들을 제공하고 있기 때문에 개발자가 활용만 잘 한다면 정말 엄청나게 서비스 운영에 도움이 될 수 있을 것 같고 이걸 만지는 개발자도 재미와 매력을 느낄 수 있는 좋은 솔루션(가격만 빼고...^^)이라고 생각한다.
