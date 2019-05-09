---
layout: post
title:  "[Spring]Eclipse로 Spring Boot 시작하기"
date:   2019-05-08 23:30:00
author: EastGlow
categories: Back-end
---
## Spring Boot

최근 몇 년 사이 순수 Spring Framework보다는 Spring Boot로 만들어진 애플리케이션들이 상당히 많아졌다.

가장 큰 이유로는 Spring Framework의 복잡한 초기 설정들을 기본적으로 다 해결해준다는 점과 가벼움과 간편함이 아닐까 생각한다. 실제로 Spring Boot를 이용한 웹 프로젝트를 생성해보면 클릭과 타이핑 몇 번으로 심플한 웹 MVC 프로젝트가 만들어진다.

회사에서는 Spring Framework를 기반으로 한 전자정부 프레임워크를 주로 이용하기 때문에 사용해볼 기회가 없었는데 회사 내부 프로젝트를 하나 맡게 되어 Spring Boot를 이용해보기로 했다. Spring Boot를 이용하기로 한 이유로는 위에서 언급한 간편함이 제일 컸다. 내부 프로젝트의 규모는 나혼자 진행해도 충분할 만큼 크게 복잡하지 않았다.

그냥 기본적인 CRUD 기능과 파일 업로드 기능, 그리고 잘 꾸며진 UI만 필요했기 때문이다. 로그인이라든지 인증, 시큐리티 등 복잡한 기능들은 일체 들어가지 않는다.

## 시작

들어가기 전에 간단하게 이번 글에 쓰일 프로젝트의 스킬셋을 나열해보자면,

- JDK 1.8
- Embedded Tomcat
- Thymeleaf
- Spring Boot 2.x
- Mybatis
- Lombok
- DevTools
- MySQL
- Eclipse IDE 2019‑03

대략 이렇게 이용할 것이다. 사실 프로젝트를 진행하다보면 하나 둘 더 추가되는 것들이 있을 수 있다.

>참고로 STS 설치 과정은 따로 다루지 않는다. Eclipse Marketplace에서 Spring을 검색하면 제일 상단에 나오는 플러그인을 설치해주면 된다. 혹은 구글링만 해도 많이 나오니 설치해주도록 하자.

### 0. Eclipse에서 Lombok을 사용하려면 플러그인을 따로 설치해줘야 한다. 아래 링크를 참고하여 설정해주도록 하자.

>http://jmlim.github.io/ide/2018/06/15/lombok-eclipse/

### 1. Eclipse를 실행한 뒤 아래 사진처럼 New > Other를 클릭한다.

![](/assets/post/20190508_1.png)

### 2. Spring Boot 안에 있는 Spring Starter Project를 선택한다.

![](/assets/post/20190508_2.png)

### 3. 아래 그림과 같이 요소들을 채워준다. 다르게 해주고 싶은 부분이 있다면 바꿔도 된다.

![](/assets/post/20190508_3.png)

### 4. DevTools, Lombok, MySQL, Mybatis, Thymeleaf, Web을 기본으로 선택해준다.

![](/assets/post/20190508_4.png)

DevTools라는 것을 Spring Boot를 사용하며 처음 써보았는데 상당히 편한 툴 같다. 프로젝트 내의 소스코드가 바뀌면 자동으로 리로드해줘서 개발자가 재시작해줄 필요없이 알아서 적용된다. 또한, LiveReload라는 크롬 확장프로그램을 깔면 HTML이나 JS 등의 소스코드가 바뀌면 알아서 웹페이지도 리로딩 된다.

### 5. Finish를 눌러주면 프로젝트가 생성된다.

![](/assets/post/20190508_5.png)

### 6. 아래 그림이 최초로 프로젝트가 생성되면 볼 수 있는 구조이다. 참고로 나는 Application 이름이 긴 게 싫어서 심플하게 Application으로 바꿔줬다. 위 과정대로 한다면 SpringBootMybatisDemoApplication이라는 이름으로 생성될 것이다.

![](/assets/post/20190508_6.png)

#### Application.java

```
package me.eastglow;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@MapperScan("me.eastglow.mapper")
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

}
```

@MapperScan은 mapper interface가 있는 경로를 명시하여 해당 경로를 스캔하고 거기에 있는 클래스들은 SqlSessionFactoryBean을 주입 받게 된다.

### 7. Application.java에 오른쪽 클릭 > Run As > Spring Boot App 을 클릭하면 앱이 실행된다. 하지만 아래와 같은 오류 로그가 올라오며 실행이 되지 않을 것이다.

![](/assets/post/20190508_7.png)

```
Description:

Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.

Reason: Failed to determine a suitable driver class


Action:

Consider the following:
	If you want an embedded database (H2, HSQL or Derby), please put it on the classpath.
	If you have database settings to be loaded from a particular profile you may need to activate it (no profiles are currently active).
```

이유는 MySQL, Mybatis 관련 의존성을 추가해주었는데 연결할 DB의 정보가 application.properties에 없기 때문이다. application.properties는 이름에서 알 수 있듯이 이 프로젝트에 필요한 설정정보들이 기록되는 파일이다.

### 8. pom.xml과 application.properties를 수정해주자.

#### application.properties

```
spring.datasource.driverClassName=net.sf.log4jdbc.sql.jdbcapi.DriverSpy
spring.datasource.url=jdbc:log4jdbc:mysql://localhost/testdb?autoReconnect=true&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
spring.datasource.username=test
spring.datasource.password=test

mybatis.type-aliases-package=me.eastglow.vo
mybatis.configuration.map-underscore-to-camel-case=true
mybatis.configuration.default-fetch-size=100
mybatis.configuration.default-statement-timeout=30
mybatis.mapper-locations=classpath:mapper/*.xml
```

1번째줄부터 사용할 DB의 Driver, DB 주소, DB Username, DB Password이다.

보통 MySQL은 아래와 같이 설정을 해줄텐데 나는 log4jdbc를 사용할 거라서 위와 같이 적어주었다. log4jdbc를 사용하지 않을 거면 아래와 같이 적어준다.

```
spring.datasource.driverClassName=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost/testdb?autoReconnect=true&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
spring.datasource.username=test
spring.datasource.password=test
```

그리고 여기서 하나 짚고 넘어갈 부분이 있는데 spring.datasource.url 쪽의 맨 뒷 부분을 보면 serverTimezone을 써준 것이 보일 것이다. 저 부분을 생략하고 그냥 하니깐 DB와 커넥션을 맺을 때 오류가 났었다. 구글링을 해보니 mysql connector의 특정 버전 이상에서 Timezone을 인식하지 못하여 발생하는 문제라던데 자세한건 아래 링크를 참고하면 좋을 듯 하다.

>https://yenaworldblog.wordpress.com/2018/01/24/java-mysql-%EC%97%B0%EB%8F%99%EC%8B%9C-%EB%B0%9C%EC%83%9D%ED%95%98%EB%8A%94-%EC%97%90%EB%9F%AC-%EB%AA%A8%EC%9D%8C/

DB 관련 설정을 다 적어주고 바로 아래에 mybatis 관련 설정들을 적어준다. type-aliases-package에 명시한 패키지 아래의 클래스들은 @Alias를 통해 별칭을 지어 사용할 수 있다. map-underscore-to-camel-case는 DB 컬럼은 보통 단어가 2개 이상이면 언더바(_)로 구분 지어준다. 그것을 VO 객체에 매핑시켜줄 때 자동으로 camel case 형태로 들어가도록 해주는 설정이다.

그 외에 mapper-locations를 통해 실제 SQL이 적혀있는 xml 경로를 명시해준다.

#### pom.xml

pom.xml은 log4jdbc를 사용할 사람만 수정해주면 된다. 아래와 같이 log4jdbc 의존성을 추가해준다.

```
<dependency>
    <groupId>org.bgee.log4jdbc-log4j2</groupId>
    <artifactId>log4jdbc-log4j2-jdbc4.1</artifactId>
    <version>1.16</version>
</dependency>
```

그리고나서 log4jdbc의 설정을 위해 아래 9번 항목의 프로젝트 구조를 참고하여 log4jdbc.log4j2.properties, logback.xml 2개 파일을 만들어준다.

##### log4jdbc.log4j2.properties

```
log4jdbc.spylogdelegator.name=net.sf.log4jdbc.log.slf4j.Slf4jSpyLogDelegator
log4jdbc.dump.sql.maxlinelength=0
```

##### logback.xml

```
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{yyyyMMdd HH:mm:ss.SSS} [%thread] %-3level %logger{5} - %msg %n</pattern>
    </encoder>
  </appender>
  
  <logger name="jdbc" level="OFF"/>
  
  <logger name="jdbc.sqlonly" level="OFF"/>
  <logger name="jdbc.sqltiming" level="DEBUG"/>
  <logger name="jdbc.audit" level="OFF"/>
  <logger name="jdbc.resultset" level="OFF"/>
  <logger name="jdbc.resultsettable" level="DEBUG"/>
  <logger name="jdbc.connection" level="OFF"/>
  
  <root level="INFO">
    <appender-ref ref="STDOUT" />
  </root>
  
</configuration>
```

여기까지 설정을 마치고 다시 Run As > Spring Boot App을 클릭해보면 실행이 정상적으로 되는 것을 확인할 수 있다. localhost:8080을 브라우져 주소창에 입력하면 아래와 같은 화면을 볼 수 있다. 오류 화면이 나오지만 아무것도 하지 않고 기본 설정만 하면 당연히 나오는 화면이므로 그냥 나오는 것만 확인하도록 하자.

![](/assets/post/20190508_8.png)

### 9. 기본적인 구조를 생성해준다. 구조는 본인 취향에 따라 달라질 수 있으니 꼭 나와 같이 하지 않아도 된다.

![](/assets/post/20190508_9.png)

구조를 대략적으로 설명하자면 mapper에는 실제 Mybatis와 관련된 SQL이 적혀있는 xml과 연결될 interface가 들어가있다.

#### TestMapper.java

```
package me.eastglow.mapper;

import java.util.List;

import org.apache.ibatis.annotations.Mapper;

import me.eastglow.vo.TestVO;

@Mapper
public interface TestMapper {

	List<TestVO> selectTestList() throws Exception;
}
```

service에는 service interface 및 이 interface를 구현한 Service들이 들어가있다.

#### TestService.java

```
package me.eastglow.service;

import java.util.List;

import me.eastglow.vo.TestVO;

public interface TestService {

	List<TestVO> getTestList() throws Exception;
}
```

#### TestServiceImpl.java

```
package me.eastglow.service.impl;

import java.util.List;

import javax.annotation.Resource;

import org.springframework.stereotype.Service;

import me.eastglow.mapper.TestMapper;
import me.eastglow.service.TestService;
import me.eastglow.vo.TestVO;

@Service
public class TestServiceImpl implements TestService {
	
	@Resource
	private TestMapper testDao;
	
	@Override
	public List<TestVO> getTestList() throws Exception {
		return testDao.selectTestList();
	}	
}
```

vo에는 이름에서 알 수 있듯이 DB에서 가져온 데이터들과 매핑될 객체들이 들어가있다.

#### TestVO.java

```
package me.eastglow.vo;

import org.apache.ibatis.type.Alias;

import lombok.Data;

@Data
@Alias("testVO")
public class TestVO {
	
	private String title;
}
```

web에는 Controller가 들어가있다.

#### TestController.java

```
package me.eastglow.web;

import java.util.ArrayList;
import java.util.List;

import javax.annotation.Resource;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import me.eastglow.service.TestService;
import me.eastglow.vo.TestVO;

@Controller
public class TestController {
	
	@Resource
	private TestService testSvc;
	
	@RequestMapping(value="/index", method=RequestMethod.GET)
	public String index(Model model) throws Exception {
		List<TestVO> testList = new ArrayList<>();
		testList = testSvc.getTestList();
		
		model.addAttribute("testList", testList);
		
		return "index";
	}
}
```

#### Test.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="me.eastglow.mapper.TestMapper">

	<select id="selectTestList" resultType="testVO">
		SELECT TITLE
		FROM TEST
		ORDER BY TITLE
	</select>
</mapper>
```

마지막으로 Thymeleaf를 이용한 View 화면을 만들어준다. Controller에서 return하여 보여주는 View 화면들은 기본적으로 resources/templates 경로로 잡혀있기 때문에 꼭 이 경로 안에 html을 만들어줘야 한다.

#### index.html

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head lang="ko">
	<meta charset="UTF-8">
	<meta http-equiv="X-UA-Compatible" content="IE=edge"/>
	<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=2.0, minimum-scale=1.0, user-scalable=no"/>
	<title>Index</title>
	<link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css" integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T" crossorigin="anonymous">
	<script src="https://code.jquery.com/jquery-3.3.1.slim.min.js" integrity="sha384-q8i/X+965DzO0rT7abK41JStQIAqVgRVzpbzo5smXKp4YfRvH+8abtTE1Pi6jizo" crossorigin="anonymous"></script>
	<script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js" integrity="sha384-UO2eT0CpHqdSJQ6hJty5KVphtPhzWj9WO1clHTMGa3JDZwrnQq4sF86dIHNDz0W1" crossorigin="anonymous"></script>
	<script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js" integrity="sha384-JjSmVgyd0p3pXB1rRibZUAYoIIy6OrQ6VrjIEaFf/nJGzIxFDsf4x0xIM+B07jRM" crossorigin="anonymous"></script>
</head>
<body>
	<div class="container">
		<h1>Test Itmes</h1>
		<table class="table">
			<thead class="thead-dark">
				<tr>
					<th scope="col">번호</th>
					<th scope="col">제목</th>
				</tr>
			</thead>
			<tbody>
				<tr th:each="item, stat : ${testList}">
					<td scope="row" th:text="${stat.count}"></td>
					<td th:text="${item.title}"></td>
				</tr>
			</tbody>
		</table>
	</div>
</body>
</html>
```

Thymeleaf 문법에 관해서는 따로 다루지 않는다. 구글링이나 공식홈페이지 문서에 잘 나와있으니 참고하도록 하자. (사실 나도 이번에 처음 사용해보는 거라 아직 익숙하지 않음...^^;)

이렇게 모두 생성하여 설정을 마쳤다면 다시 한번 Run As > Spring Boot App을 눌러주도록 하자.

### 10. 아래와 같이 잘 실행되면 성공이다.

![](/assets/post/20190508_10.png)

그냥 간단히 Mybatis를 통해 select한 List를 View에 뿌려주는 페이지를 만들어보았다. 아마 기본적인 설정이나 관련된 자료들을 찾는 시간을 제외하고 대략 30분도 안 걸려서 프로젝트 생성부터 화면 띄우는 것까지 완성한 것 같다.

이번 글에서는 간단하게 Spring Boot와 Mybatis를 이용하여 DB에서 가져온 데이터를 View에 뿌려주는 것까지 해보았다. 다음번에는 Mybaits가 아니라 JPA를 공부하여 Mybatis와 마찬가지로 DB와 연동하여 View에 데이터를 뿌려주는 것까지 해볼 예정이다. JAP는 아직 공부 중이라 조금 시간이 흐른 뒤(?) 정리하여 올리도록 해야겠다.
