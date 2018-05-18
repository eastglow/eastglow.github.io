---
layout: post
title:  "[Spring]Jasypt를 이용하여 .properties 파일 암호화하기"
date:   2018-05-17 20:00:00
author: EastGlow
categories: Back-end
---

## 1. pom.xml 수정

```
<!-- JASYPT: Spring 3.1x Simplified Encryption -->
<dependency>
    <groupId>org.jasypt</groupId>
    <artifactId>jasypt-spring31</artifactId>
    <version>1.9.2</version>
    <scope>compile</scope>
</dependency>
```

## 2. Spring XML 파일 수정

```
<!-- Jasypt -->
<bean id="environmentVariablesConfiguration" class="org.jasypt.encryption.pbe.config.EnvironmentStringPBEConfig">
	<property name="algorithm" value="PBEWithMD5AndDES" />
</bean>

<bean id="configurationEncryptor" class="org.jasypt.encryption.pbe.StandardPBEStringEncryptor">
	<property name="config" ref="environmentVariablesConfiguration" />
	<property name="password" value="EASTGLOW_PASS" />
</bean>

<bean id="propertyConfigurer" class="org.jasypt.spring31.properties.EncryptablePropertyPlaceholderConfigurer">
	<constructor-arg ref="configurationEncryptor" />
	<property name="locations">
		<list>
			<value>/WEB-INF/props/jdbc.properties</value>
		</list>
	</property>
</bean>
```

나는 현재 전자정부 프레임워크를 사용 중이기 때문에 context-datasource.xml 혹은 context-common.xml을 수정하였다. Spring을 사용 중이라면 설정 파일 이름이 다를테니 유의하기 바란다.

### ※ 주의

Spring XML 파일에 이미 context:property-placeholder가 추가되어 있다면 해당 소스는 주석처리하거나 삭제해야 위에서 추가한 bean이 정상적으로 작동된다.

```
<!-- dataSource property를 위한 PropertyPlaceholderConfigurer 설정 -->
<context:property-placeholder location="/WEB-INF/props/jdbc.properties" />
```

처음 암호화 관련 설정을 했을 때 분명히 다 맞게 설정한 거 같은데 아무리 해도 복호화가 안 되어서 설정 파일을 하나하나 뜯어보던 중 위 코드 때문이란걸 알게 되었다.

```
<bean id="propertyConfigurer" class="org.jasypt.spring31.properties.EncryptablePropertyPlaceholderConfigurer">
```
를 통해서 placeholder를 지정해주고 있기 때문에 중복되어 그런거 같다.

## 3. 암호화를 위한 Java Class 생성

```
import org.jasypt.encryption.pbe.StandardPBEStringEncryptor;

public class JasyptUtil {

	public void init(){
		    StandardPBEStringEncryptor pbeEnc = new StandardPBEStringEncryptor();
        
        pbeEnc.setAlgorithm("PBEWithMD5AndDES");
        pbeEnc.setPassword("EASTGLOW_PASS");

        String url = pbeEnc.encrypt("jdbc:oracle:thin:@1.1.1.1:1521:XE");
        String username = pbeEnc.encrypt("USERNAME");
        String password = pbeEnc.encrypt("PASSWORD");

        System.out.println(url);
        System.out.println(username);
        System.out.println(password);
	}
}
```

위와 같이 원하는 문자열을 암호화 시킬 자바 클래스를 생성해준다. 이 클래스를 실행하면 자신이 입력한 문자열이 암호화 된 문자열로 나오는 것을 볼 수 있다.

콘솔창에서 암호화 된 문자열을 복사하여 메모장 같은 곳에 기록해둔다.

추가로 setAlgorithm은 암호화에 사용될 알고리즘 종류를 뜻한다. MD5 말고도 SHA256도 사용 가능하다. 이경우는 따로 라이브러리를 추가해야한다. 소스 또한 조금 달라진다.

setPassword는 암호화 & 복호화에 이용될 키값이다. 2번 XML 파일 수정에서 configurationEncryptor Bean의 property 중에 "password"가 있는데 이 값과 같은 값이 되어야 한다.

## 4. .properties 파일 수정

```
db.driver=oracle.jdbc.driver.OracleDriver
db.dburl=ENC(7SiNuXbWDn7BybSzQFEtim37T2dioPnik4TjxwuOhFR4qKzdrz/m8zl27TDWqwYP)
db.username=ENC(aPzuR1AFxucnvie1h1Jnqw==)
db.password=ENC(l6ZtYkfUKvEwEXlA4n1uAZQEzqB7F/kJ)
```

3번에서 암호화한 문자열을 ENC()로 감싸서 원하는 properties 파일에 붙여넣는다.
