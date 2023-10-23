---
layout: post
title:  "[Java]Mybatis Generator"
date:   2023-10-23
author: EastGlow
categories: Back-end
---

# DTO 클래스를 생성하기 너무 귀찮다...

사내 프로젝트들은 기본적으로 A라는 테이블에 대해서 1대1로 매핑되는 DTO를 만들어서 쓰는 편이다. 아, 물론 이번 글 제목에도 나와있듯이 이 Table-DTO 관계에 쓰이는 DTO 클래스들은 개발자가 직접 Getter, Setter, ToString... 등을 달아서 생성하는건 아니다.

사내 공용 라이브러리에서는 이미 Mybatis Generator 플러그인을 이용해서 DB에 있는 모든 테이블들을 DTO Java Model로 만들어주고 있다. 그러면 이 글은 왜 쓰느냐? 앞에서 한 얘기는 사내 레거시 프로젝트에 해당되는 케이스이고 MSA 프로젝트는 이러한 레거시 라이브러리를 참조할 수 없기에 각 팀에서 직접 이러한 DTO 클래스를 생성, 관리하고 있었다.

우리팀도 이것에 예외는 없었고 실제로 우리팀에서 자주 쓰는 테이블들은 DTO 클래스를 별도로 만들어서 사용 중이다. 여기서 DTO 클래스라고 하였지만 테이블을 Java Model로 매핑한 객체는 사용 목적에 따라 Domain 클래스도 될 수도 있다. 그냥 범용적으로 많이들 알고 있고 사용하는 DTO라는 명칭으로 딱 정해서 설명에 용이하고자 하였으니 적당히(?) 봐주시면 될 거 같다.

여하튼, 테이블은 신규 생성이 잦지않고 이미 만들어진 특정 테이블들만 사용하기 때문에 사실 매핑 DTO 클래스를 한번 만들어두면 필드 추가가 있거나 어떤 특별한 상황이 아닌 이상 수정할 일은 없긴 하지만 이러한 DTO 클래스를 굳이 코드로 관리해야할까? 싶어서 Mybatis Generator를 적용해보기로 했다.

# Mybatis Generator를 적용해보자

## build.gradle

현재 우리팀 MSA 프로젝트는 maven, gradle를 혼합해서 사용 중이나 공용 DTO가 들어갈 프로젝트는 gradle을 사용 중이라 gradle 기준으로 설명하고자 한다. (gradle 버전은 7.5.1이다.)

우선은 `com.thinkimi.gradle.MybatisGenerator`라는 Mybatis Generator 오픈소스를 이용하였다. 찾아보니 비슷한 역할을 하는 몇가지가 더 있는거 같긴한데 얘가 제일 먼저 검색되었기도 하고 다 비슷비슷해보여서 이걸로 정했다.

```
buildscript {  
	ext {  
		springBootVersion = "2.3.12.RELEASE"
		lombokVersion = "1.18.24"
		ojdbcVersion = "19.3.0.0"
		mybatisGeneratorCoreVersion = "1.4.2"
		mybatisGeneratorPluginVersion = "2.4"
	}
  
	dependencies {  
		classpath "org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}"  
	}  
}  
  
plugins {  
	id "idea"  
	id "maven-publish"  
	id "java-library"  
	id "io.spring.dependency-management" version "1.1.0"  
	id "com.thinkimi.gradle.MybatisGenerator" version "${mybatisGeneratorPluginVersion}"  
}
```

그냥 흔하게 볼 수 있는 `build.gradle`의 초반부 구성이다. 이번 글에서 필수적으로 써야할 것은 두말할 것도 없이 `com.thinkimi.gradle.MybatisGenerator`이다. 버전은 2.4.1 버전이 최신인거 같은데 자세히는 기억 안 나지만 뭔가 이슈가 있어서 2.4 버전을 사용하고 있다.

```
dependencies {  
	...
	(생략)
	...
	
	implementation group: "org.mybatis.generator", name: "mybatis-generator-core", version: "${mybatisGeneratorCoreVersion}"  
	implementation group: "com.oracle.ojdbc", name: "ojdbc8", version: "${ojdbcVersion}"  
	
	...
	(생략)
	...
}
```

디펜던시에도 마찬가지로 위와 같이 추가해주도록 하였다. 사내 DB가 Oracle이라 DB 연결을 위한 ojdbc 디펜던시도 추가해주었다.

```
mybatisGenerator {  
	verbose = true  
	configFile = "src/main/resources/generatorConfig.xml"  

	dependencies {  
		mybatisGenerator "com.oracle.ojdbc:ojdbc8:${ojdbcVersion}"  
		mybatisGenerator "org.mybatis.generator:mybatis-generator-core:${mybatisGeneratorCoreVersion}"
		mybatisGenerator "org.apache.commons:commons-lang3:3.12.0"  
		mybatisGenerator files("${buildDir}/classes/java/main/")  
	}
}
```

mybatisGenerator 설정이다. verbose는 ture로 설정해두면 코드 생성 프로세스의 상세한 정보와 진행 상황이 콘솔에 출력된다. configFile은 상세 설정이 기재된 파일 경로이다. 기본적으로 저 경로에 두고 쓰는 듯 하였다.

mybatisGenerator 내에도 따로 디펜던시를 명시해줘야 했는데 위와 같이 써주면 된다. ojdbc와 mybatis-generator-core 말고도 더 추가된게 있는데 commons-lang3는 커스텀 플러그인 클래스에서 쓰일 예정이고 맨 마지막 files로 클래스 빌드 디렉토리가 하나 추가되어 있다.

이걸 왜 했냐면 이 Mybatis Generator를 적용하려는 프로젝트에 커스텀 플러그인 기능을 적용하고 싶어서 별도의 클래스를 하나 만들었다. 그런데 mbGenerator 명령어로 실행하면 자꾸 이 커스텀 클래스를 못 찾는 것이었다. 그래서 이것저것 해보다가 결국 해결한 방법이 compileJava를 통해 자바 클래스들을 컴파일하여 그 결과물을 mybatisGenerator에서 참조하도록 하였다.

다른 사람들은 커스텀 플러그인 클래스를 어떻게 적용했나 찾아보니 일단 내가 찾은 예시들은 모두 아예 별도의 프로젝트를 따로 만들어서 그것을 jar로 만든 후, gradle에서 외부 라이브러리로 참조하도록 하는 방법을 사용하고 있었다. 결국 mybatisGenerator의 디펜던시에서 참조하려면 컴파일된 결과물이 있어야 한다는 것이었고 그럼 그냥 gradle task 순서만 잘 조정해서 먼저 컴파일 한 후 mbGenerator를 실행하면 되지 않을까 해서 해보니 다행히 잘 되었다.


## generatorConfig.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<!DOCTYPE generatorConfiguration  
  PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"  
 "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">  
  
<generatorConfiguration>  
	<context id="TESTDB" targetRuntime="MyBatis3" defaultModelType="flat">  
		<property name="javaFileEncoding" value="UTF-8"/>
		
		<plugin type="my.eastglow.common.plugin.MybatisGeneratorLombokPlugin">  
			<property name="defaultAnnotations" value="getter,setter,toString,noArgsConstructor,allArgsConstructor"/>  
		</plugin>  
	
		<commentGenerator type="my.eastglow.common.util.MybatisCommentGenerator"/>  
		
		<jdbcConnection driverClass="oracle.jdbc.driver.OracleDriver"  
			connectionURL="jdbc:oracle:thin:@127.0.0.1:1538:testdb"  
			userId="EASTGLOW"
			password="qwer1234">
		</jdbcConnection>
			
		<javaModelGenerator targetPackage="my.eastglow.common.dto" targetProject="src/main/java">
			<property name="enableSubPackages" value="false"/>
			<property name="trimStrings" value="false"/>
		</javaModelGenerator>
		
		<table schema="TESTDB" tableName="%" catalog="TABLE">
			<domainObjectRenamingRule searchString="^.*$" replaceString="$0BaseDto"/>
		</table>
	</context>
</generatorConfiguration>
 ```

구글링하거나 플러그인 공식 Docs에서 쉽게 찾아볼 수 있는 설정 xml이다. 딱히 설명할 부분은 없고 한가지 추가 설명할 부분은 `MybatisGeneratorLombokPlugin`이라는 클래스를 별도의 커스텀 플러그인 클래스로 등록해둔 부분이다. 이부분은 아래에서 조금 더 설명하겠다.

## MybatisGeneratorLombokPlugin

```java

@Slf4j  
public class MybatisGeneratorLombokPlugin extends PluginAdapter {  

	private final List<String> defaultAnnotationList = new ArrayList<>();  

	@Override  
	public boolean validate(List<String> warnings) {  

		// 기본으로 적용할 annotation
		String defaultAnnotations = properties.getProperty("defaultAnnotations");  

		if (StringUtils.isNotBlank(defaultAnnotations)) {  
			String[] array = defaultAnnotations.split(",");  
			for (String annotation : array) {  
				defaultAnnotationList.add(annotation);  
			}  
		} else {  
			// 기본 설정이 없는 경우 @Data만 넣어줌  
			defaultAnnotationList.add("data");  
		}  

		return true;  
	}  

	@Override  
	public boolean modelBaseRecordClassGenerated(TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {  
		addLombokAnnotations(topLevelClass, introspectedTable);  
		return true;
	}  

	@Override  
	public boolean modelPrimaryKeyClassGenerated(TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {  
		addLombokAnnotations(topLevelClass, introspectedTable);  
		return true;
	}  

	@Override  
	public boolean modelRecordWithBLOBsClassGenerated(TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {  
		addLombokAnnotations(topLevelClass, introspectedTable);  
		return true;
	}  

	private void addLombokAnnotations(TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {  

		String tableName = introspectedTable.getFullyQualifiedTableNameAtRuntime();  

		for (String value : defaultAnnotationList) {  
			Annotations annotations = Annotations.getValueOf(value);  

			topLevelClass.addImportedType(annotations.javaType);  
			topLevelClass.addAnnotation(annotations.name);  
		}  
	}  

	private enum Annotations {  
		DATA("data", "@Data", "lombok.Data"),  
		GETTER("getter", "@Getter", "lombok.Getter"),  
		SETTER("setter", "@Setter", "lombok.Setter"),  
		BUILDER("builder", "@Builder", "lombok.Builder"),  
		ALL_ARGS_CONSTRUCTOR("allArgsConstructor", "@AllArgsConstructor", "lombok.AllArgsConstructor"),  
		NO_ARGS_CONSTRUCTOR("noArgsConstructor", "@NoArgsConstructor", "lombok.NoArgsConstructor"),  
		REQUIRED_ARGS_CONSTRUCTOR("requiredArgsConstructor", "@RequiredArgsConstructor", "lombok.RequiredArgsConstructor"),  
		EQUALS_AND_HASH_CODE("equalsAndHashCode", "@EqualsAndHashCode", "lombok.EqualsAndHashCode"),  
		TO_STRING("toString", "@ToString", "lombok.ToString");  

		private final String paramName;  
		private final String name;  
		private final FullyQualifiedJavaType javaType;  

		Annotations(String paramName, String name, String className) {  
			this.paramName = paramName;  
			this.name = name;  
			this.javaType = new FullyQualifiedJavaType(className);  
		}  

		private static Annotations getValueOf(String paramName) {  

			for (Annotations annotation : Annotations.values()) {  
				if (String.CASE_INSENSITIVE_ORDER.compare(paramName, annotation.paramName) == 0) {  
					return annotation;  
				}  
			}  

			return null;  
		}  
	}  
}
```

실제로 테이블을 Java Model로 변환해주는 담당하는 메서드들을 구현한 클래스이다. 나는 여기에서 커스터마이징 하고자 했던 부분은 생성되는 Java DTO 클래스에 기본적으로 특정 Lombok Annotation을 달고자 하였다. 이 기본 Annotation은 아까 위에서 본 generatorConfig.xml에 있던 defaultAnnotations에 정의해둔 것들이다. 위와 같이 정의해둔 커스텀 플러그인 클래스를 xml 파일에서 plugin type에 기재해두면 실제 결과물이 만들어질 때 이 클래스를 통해 만들어지게 된다.

## mbGenerator

mbGenerator task를 실행하면 위에서 설명한 Mybatis Generator 설정들을 토대로 결과물이 만들어지게 된다. 하지만 쉽게 되면 재미가 없지... 실제로 이제 사내 넥서스에 빌드 결과물을 올리고 나서 보니 이상하게 DTO 클래스들이 생성이 안된 채로 올라가는 것이었다. 내가 실행한 빌드 스크립트에 써둔 gradle 명령어는 아래와 같았다.

```
./gradlew clean compileJava mbGenerator publish --stacktrace
```

이전 빌드 결과물이 남아있을지도 모르니 clean을 해주고, 그다음 아까 글 초반부에 설명한 커스텀 플러그인 클래스 참조를 위해 먼저 compileJava를 실행해준다. 그리고 mbGenrator를 실행하여 실제 DTO 클래스들이 생성되고 publish를 통해 사내 넥서스에 업로드까지 잘 되는 것을 기대하였는데 업로드는 물론 잘 되었다. 하지만 빌드 결과물엔 DTO 클래스들이 빠져있었다.

그래서 이리저리 명령어를 조합해서 돌려보니 아래와 같이 돌렸을 때 DTO 클래스들이 제대로 포함이 되어 업로드 되는 것을 확인했다.

```
./gradlew clean compileJava mbGenerator --stacktrace && ./gradlew clean publish --stacktrace
```

mbGenerator까지는 동일하나 publish 실행 전에 clean을 통해 빌드된 클래스 잔여물(?)들을 한번 싹 밀고 나서 publish 해주도록 하였다. 어차피 publish에서 빌드하고 jar로 만들고 업로드까지 해주는 task이기 때문에 clean을 한다해도 문제는 없었다. 오히려 clean을 안해주니깐 mbGenerator 앞에서 실행한 compileJava 기준으로 빌드된 결과물이 적용이 되어서(= mbGenerator로 DTO가 생성되기 전) publish가 끝난 결과물엔 DTO 클래스들이 없는게 아닐까 생각된다.

어찌됐든 이렇게 해서 Mybatis Generator를 무사히 적용하여 다른 MSA 프로젝트에서 MG가 적용된 라이브러리를 참조했을 때, 자동으로 생성된 DTO들이 잘 참조되는 것까지 확인하여 마무리하였다.

끝으로 위 과정을 진행하면서 겪었던 몇가지 이슈사항을 체크해보자면,

- 기존에 라이브러리 프로젝트는 gradle 6.3 버전이었는데 이 버전에서는 mbGenrator에서 참조하는 configFile 경로도 제대로 못 찾고 xml에서도 class not found를 표출하며 경로 관련된 문제가 계속 있엇다. 구글링과 GPT에게 물어봐도 특별한 답은 못 찾아서 혹시나 하는 마음에 gradle 버전업을 진행하였는데(7.5.1) 다행히도 이것으로 해결이 되었다.
- mbGenerator 이후 publish를 진행하여 업로드된 라이브러리 jar 결과물에 자동 생성되었어야 할 DTO 클래스들이 없었던 문제는 위에서 자세히 설명해두었으니 확인해보면 될 것 같다.
- 커스텀 플러그인 클래스를 mbGenerator에서 못 찾는 것도 아까 build.gradle 설명 부분에서 자세히 설명해두었으니 참고하기 바란다.

위 3가지 정도의 이슈사항이 있었던 것 같다. 이것들을 해결한다고 이것저것 삽질도 많이 한 거 같은데 다행히도 잘 해결되었다. 테이블 정보를 기반으로 일관된 형태의 DTO 클래스들을 생성할 수 있게 해주는 플러그인이라서 확실히 편하기도 하고 코드 관리에도 도움이 되는 것 같아서 좋은 듯 하다.
