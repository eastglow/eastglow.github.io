---
layout: post
title:  "[Spring]Bean 생성 시 이름에 패키지명까지 포함하기"
date:   2018-12-30 23:00:00
author: EastGlow
categories: Back-end
---
## 문제

보통 Controller를 만들어줄 때, 같은 이름의 Controller가 프로젝트 내에 2개 이상이 있다면 처음 Spring이 시작할 때 오류가 나곤 한다. 이유는 같은 이름의 Bean이 생성되기 때문에 그러는데 나는 같은 이름의 Controller를 생성할 필요가 있었기에 이것을 해결해야했다.

- /user/HomeController
- /admin/HomeController

와 같이 같은 이름의 Controller가 패키지명만 다르게 필요하였다. 찾아보니 이것을 도와주는 Spring 설정이 있었다. Spring에서 기본 제공하는 `BeanNameGenerator`를 implements하여 재정의한 BeanNameGenerator를 만들어 사용하면 위와 같은 설정을 할 수 있다.

## 방법

```
import org.springframework.beans.factory.annotation.AnnotatedBeanDefinition;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.BeanNameGenerator;
import org.springframework.context.annotation.AnnotationBeanNameGenerator;
import org.springframework.core.type.AnnotationMetadata;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RestController;

import java.util.Set;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class FullBeanNameGenerator implements BeanNameGenerator {

    private static final Logger LOGGER = LoggerFactory.getLogger(FullBeanNameGenerator.class);

    /*
     *  이 클래스는 Spring이 기본적으로 사용하는 BeanNameGenerator. 컨트롤러가 아닐경우 이 클래스를 사용하려고 선언
     */
    private final AnnotationBeanNameGenerator defaultGenerator = new AnnotationBeanNameGenerator();
    
    @Override
    public String generateBeanName(final BeanDefinition definition, final BeanDefinitionRegistry registry) {    
        final String result;
        
        // definition이 컨트롤러일 경우 패키지 이름을 포함한 Bean 이름을, 아닐경우 Spring 기본형식을 따름
        if (isController(definition)) {
            result = generateFullBeanName((AnnotatedBeanDefinition) definition);
        } else {
            result = this.defaultGenerator.generateBeanName(definition, registry);
        }

        LOGGER.debug("Registered Bean Name : " + result);

        return result;
    }

    private String generateFullBeanName(final AnnotatedBeanDefinition definition) {
        // 패키지를 포함한 전체 이름을 반환
        return definition.getMetadata().getClassName();
    }

    private Set<String> getAnnotationTypes(final BeanDefinition definition) {
        final AnnotatedBeanDefinition annotatedDef = (AnnotatedBeanDefinition) definition;
        final AnnotationMetadata metadata = annotatedDef.getMetadata();
        return metadata.getAnnotationTypes();
    }

    /*
     * Controller인지 판별하는 메서드
     */
    private boolean isController(final BeanDefinition definition) {
        if (definition instanceof AnnotatedBeanDefinition) {            
            // definition에 속한 모든 Annotation을 가져옴.
            final Set<String> annotationTypes = getAnnotationTypes(definition);
            
            // annotation 중 @Controller이거나 @RestController 일 경우 Controller로 인식
            for (final String annotationType : annotationTypes) {
                if (annotationType.equals(Controller.class.getName())) {
                    return true;
                }
                if (annotationType.equals(RestController.class.getName())) {
                    return true;
                }
            }

        }
        return false;
    }
}
```

나도 구글링하면서 여러 개의 글을 찾아보다가 찾은 코드이다. `generateBeanName`메서드를 통해서 먼저 Controller인지 판별한 후, Controller라면 다시 `generateFullBeanName`메서드를 통해 패키지를 포함한 전체 이름을 반환하도록 하였다. 이렇게 되면 위에서 예로 들었던 2개의 Controller의 Bean 이름은 아래와 같이 만들어지게 된다.

- user.HomeController
- admin.HomeController

이렇게 패키지명까지 포함하게 되면서 Spring 시작 시, Bean 이름을 생성할 때 오류가 나지 않고 정상적으로 생성이 되게 된다. 위와 같이 benaNameGenerator를 만들어줬다면 이것을 사용하기 위해 아래와 같이 설정해줘야 한다.

전자정부 프레임워크를 기준으로 egov-com-servlet.xml에서 아래와 같이 써주도록 한다.

```
<!-- 패키지 내 Controller, Service, Repository 클래스의 auto detect를 위한 mvc 설정. name-generator를 통해 컨트롤러는 패키지명까지 포함하여 bena 이름을 등록하도록 설정 -->
<context:component-scan base-package="egovframework" name-generator="egovframework.egov.util.FullBeanNameGenerator">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Service"/>
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Repository"/>
</context:component-scan>
```

name-generator 부분에 방금 만든 FullBeanNameGenerator 경로를 써주면 Bean 이름을 만들 때 저것을 사용하여 만들게 된다.
