---
layout: post
title:  "[Spring]다중 DataSource 설정하기"
date:   2019-01-10 19:00:00
author: EastGlow
categories: Back-end
---
> 전자정부 프레임워크 3.7 버전을 사용 중이기 때문에 이 프레임워크를 기준으로 설명함. Spring  4 기반으로 만들어졌기 때문에 크게 다른 부분은 없으리라 생각됨.

## 기본 설정
### 1. MSSQL JDBC 드라이버 추가 & pom.xml 설정
```
<!-- 보통 commons-dbcp는 기본적으로 추가되어 있기 때문에 이미 있다면 추가 X -->
<dependency>
    <groupId>commons-dbcp</groupId>
    <artifactId>commons-dbcp</artifactId>
    <version>1.4</version>
</dependency>  

<!-- jdbc to mssql -->
<dependency>
    <groupId>com.microsoft.sqlserver.jdbc</groupId>
    <artifactId>sqljdbc</artifactId>
    <version>3.0</version>
    <scope>system</scope>
    <systemPath>${basedir}/src/main/webapp/WEB-INF/lib/sqljdbc42.jar</systemPath>
</dependency>
<!-- jdbc to mssql -->
```
MSSQL JDBC jar 파일은 라이센스 문제로 Maven에서는 제공하지 않는다고 한다. MS 홈페이지에서 직접 jar 파일을 받아서 WEB-IFN/lib 경로에 넣어주고 Maven으로 잡아주도록 한다.

MSSQL JDBC 다운로드: https://www.microsoft.com/ko-kr/download/details.aspx?id=11774

### 2. context-datasource.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:jdbc="http://www.springframework.org/schema/jdbc"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
						http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
						http://www.springframework.org/schema/jdbc  http://www.springframework.org/schema/jdbc/spring-jdbc-4.0.xsd">

    <!-- MYSQL DB  -->
    <bean id="dataSourceSpied" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    	<property name="driverClassName" value="${db.driver}"/>
        <property name="url" value="${db.dburl}" />
        <property name="username" value="${db.username}"/>
        <property name="password" value="${db.password}"/>
        <property name="initialSize" value="5"/>
        <property name="maxActive" value="30"/>
        <property name="maxIdle" value="30"/>
        <property name="maxWait" value="30000"/>
        <property name="testOnBorrow" value="true"/>
        <property name="validationQuery" value="SELECT 1 FROM DUAL"/>
        <property name="testWhileIdle" value="true"/>
    	<property name="timeBetweenEvictionRunsMillis" value="60000"/>    	
    </bean>    
 	   
    <bean id="dataSource" class="net.sf.log4jdbc.Log4jdbcProxyDataSource">
        <constructor-arg ref="dataSourceSpied" />
        <property name="logFormatter">
            <bean class="net.sf.log4jdbc.tools.Log4JdbcCustomFormatter">
                <property name="loggingType" value="MULTI_LINE" />
                <property name="sqlPrefix" value="## SQL ## : "/>
            </bean>
        </property>
    </bean>
    
    <!-- MSSQL DB  -->
    <bean id="dataSourceSpied2" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="${db2.driver}"/>
        <property name="url" value="${db2.dburl}" />
        <property name="username" value="${db2.username}"/>
        <property name="password" value="${db2.password}"/>
        <property name="initialSize" value="5"/>
        <property name="maxActive" value="30"/>
        <property name="maxIdle" value="30"/>
        <property name="maxWait" value="30000"/>
        <property name="testOnBorrow" value="true"/>
        <property name="validationQuery" value="SELECT 1"/>
        <property name="testWhileIdle" value="true"/>
    	<property name="timeBetweenEvictionRunsMillis" value="60000"/>
    </bean>
  
    <bean id="dataSource2" class="net.sf.log4jdbc.Log4jdbcProxyDataSource">
        <constructor-arg ref="dataSourceSpied2" />
        <property name="logFormatter">
            <bean class="net.sf.log4jdbc.tools.Log4JdbcCustomFormatter">
                <property name="loggingType" value="MULTI_LINE" />
                <property name="sqlPrefix" value="## SQL ## : "/>
            </bean>
        </property>
    </bean>    
</beans>
```
context-datasource.xml에 DataSource 설정을 해주도록 한다. 원래 MySQL을 메인으로 쓰고 있었는데 추가로 MSSQL DB가 필요하게 되어 아래에 추가해주었다. BasicDataSource 밑에 있는 Log4jdbcProxyDataSource는 콘솔창에 SQL 처리결과를 출력해주기 위해 설정해두었다.

### 3. context-mapper.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" 
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xmlns:context="http://www.springframework.org/schema/context"
		xsi:schemaLocation="http://www.springframework.org/schema/beans 
							http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
							http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd">

	<!-- SqlSession setup for MyBatis Database Layer -->
	<bean id="sqlSession" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<property name="configLocation" value="classpath:/egovframework/mapper/mapper-config.xml" />
		<property name="mapperLocations" value="classpath:/egovframework/mapper/mysql/*.xml" />  
	</bean>
	
	<bean id="otherSqlSession" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource2" />
		<property name="configLocation" value="classpath:/egovframework/mapper/mapper-config-others.xml" />
		<property name="mapperLocations" value="classpath:/egovframework/mapper/others/*.xml" />
	</bean>

	<!-- MapperConfigurer setup for MyBatis Database Layer with @Mapper("deptMapper") in DeptMapper Interface -->
 	<bean class="egovframework.rte.psl.dataaccess.mapper.MapperConfigurer">
		<property name="basePackage" value="egovframework.test.service.impl" />
		<property name="sqlSessionFactoryBeanName" value="sqlSession" />
	</bean>
	
 	<bean class="egovframework.rte.psl.dataaccess.mapper.MapperConfigurer">
		<property name="basePackage" value="egovframework.test.service.otherImpl" />
		<property name="sqlSessionFactoryBeanName" value="otherSqlSession" />
	</bean>
</beans>
```
생성한 DataSource를 mapper에 쓰기 위해 설정해주는 부분이다. 원래 `sqlSession` 하나만 설정해뒀었고, `otherSqlSession`은 새로 추가하였다. 전자정부 프레임워크 포털 Q&A를 살펴보니 MapperConfigurer를 설정할 때, basePackage는 bean끼리 겹치지 않게 서로 다른 경로로 써주는게 좋다고 한다. 아무래도 서로 다른 DB를 이용하다보니 같은 경로의 패키지를 쓰면 문제가 될 수 있다고 하였다.

## 2. Java 및 Mapper 설정
### 1. BoardMapper.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="egovframework.test.service.otherImpl.BoardMapper">
	
	<select id="selectBoardCount" parameterType="boardVO" resultType="int">
		SELECT COUNT(1) CNT 
		FROM BOARD_TABLE
	</select>
	
	<select id="selectBoardList" parameterType="boardVO" resultType="egovMap">
		SELECT PKID, TITLE, CONTENTS
		FROM BOARD_TABLE
		ORDER BY WDATE DESC			
		OFFSET (#{currentPage}-1)*#{pageSize} ROWS
		FETCH NEXT #{pageSize} ROWS ONLY
	</select>
</mapper>
```

쿼리를 담아줄 mapper.xml 파일을 생성한다. mapper.xml의 이름은 3번째 줄에 있는 `<mapper namespace="egovframework.test.service.otherImpl.BoardMapper">`의 `BoardMapper`와 동일하게 지어주도록 하자.

### 2. BoardMapper.java
```
package egovframework.test.service.otherImpl;

import java.util.List;

import egovframework.rte.psl.dataaccess.mapper.Mapper;
import egovframework.test.vo.BoardVO;

/**
 * 게시판 처리에 관한 데이터처리 매퍼 클래스
 */

@Mapper("BoardMapper")
public interface BoardMapper {
    
    /**
     * 게시판 전체 레코드 수
     */
    int selectBoardCount(BoardVO vo) throws Exception;
    
    /**
     * 게시물 리스트
     */
    List<?> selectBoardList(BoardVO vo) throws Exception;
}
```

1번에서 만들어준 BoardMapper.xml과 연결되는 부분이다. 1번 xml 파일에서 쿼리를 만들어두면 여기서 그 쿼리의 id값을 서로 매칭시켜 불러올 수 있다.

### 3. BoardServiceImpl.java
```
package egovframework.test.service.otherImpl;

import java.util.List;

import egovframework.rte.fdl.cmmn.EgovAbstractServiceImpl;
import egovframework.test.egov.EgovProperties;
import egovframework.test.service.BoardService;
import egovframework.test.vo.BoardVO;

import javax.annotation.Resource;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;


@Service("BoardService")
public class BoardServiceImpl extends EgovAbstractServiceImpl implements BoardService {

	private static final Logger LOGGER = LoggerFactory.getLogger(BoardServiceImpl.class);
		
	@Resource(name="BoardMapper")
    private BoardMapper boardDao;
	
    /**
     * 전체 게시물 수 
     */
    @Override
    public int getBoardCount(BoardVO vo) throws Exception { 
        return boardDao.selectBoardCount(vo);
    } 
    
    /**
     * 게시물 리스트 
     */
    @Override
    public List<?> getBoardList(BoardVO vo) throws Exception {  
        return boardDao.selectBoardList(vo);
    }
}
```

BoardMapper.java를 직접적으로 호출하여 사용할 ServiceImpl 부분이다. 여기서 주의해야할 점은 `@Resource(name="BoardMapper")`를 선언할 때 name은 2번에서 만들어준 `@Mapper("BoardMapper")`이름과 동일하게 설정해주는 것이다. 이름이 다르거나 오타가 있으면 오류가 난다.

### 4. BoardService.java
```
package egovframework.test.service;

import java.util.List;
import egovframework.test.vo.BoardVO;


public interface BoardService {
    
    /**
     * 게시판 전체 레코드  수
     */
    public int getBoardCount(BoardVO vo) throws Exception; 
    
    /**
     * 게시물 리스트 
     */
    public List<?> getBoardList(BoardVO vo) throws Exception ;
}
```

Controller에서 호출할 Service 부분이다. 별 건 없고 ServiceImpl쪽과 이름이 잘 맞는지만 유의하여 만들어준다.
