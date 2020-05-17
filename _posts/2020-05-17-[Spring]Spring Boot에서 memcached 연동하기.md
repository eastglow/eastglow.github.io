---
layout: post
title:  "[Spring]Spring Boot에서 memcached 연동하기"
date:   2020-05-17 20:15:00
author: EastGlow
categories: Back-end
---

사내에서 기존에 레거시 프로젝트(Spring 기반)와 연동하여 사용 중이던 memcached를 Spring Boot에서 사용해야할 일이 생겼다. 연동하는 과정을 간단하게 정리하여 올려본다. 필요한 부분만 추려냈기 때문에 Spring Boot 세팅이라든지 버전 등은 각자 상황에 맞게 사용하면 될 듯 하다.

# pom.xml : memcached 라이브러리 추가

    <!-- memcached -->
    <dependency>
        <groupId>com.google.code.simple-spring-memcached</groupId>
        <artifactId>spring-cache</artifactId>
        <version>4.1.3</version>
    </dependency>
    <dependency>
        <groupId>com.google.code.simple-spring-memcached</groupId>
        <artifactId>simple-spring-memcached</artifactId>
        <version>4.1.3</version>
    </dependency>
    <dependency>
        <groupId>com.google.code.simple-spring-memcached</groupId>
        <artifactId>xmemcached-provider</artifactId>
    </dependency>

# application.yml : 신규 항목 추가

    cache:
      operationTimeout: 1500
      cacheName: testCache
      expiration: 600 # 초단위(=600초)
      address: localhost:9090

# WebApplication.java : 어노테이션 추가

    @EnableCaching
    @EnableAspectJAutoProxy

# CacheConfig.java : 신규 추가

    package me.eastglow.config;
    
    import java.util.ArrayList;
    
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.cache.CacheManager;
    import org.springframework.cache.annotation.CachingConfigurerSupport;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    
    import com.google.code.ssm.Cache;
    import com.google.code.ssm.CacheFactory;
    import com.google.code.ssm.config.AddressProvider;
    import com.google.code.ssm.config.DefaultAddressProvider;
    import com.google.code.ssm.providers.CacheConfiguration;
    import com.google.code.ssm.providers.xmemcached.MemcacheClientFactoryImpl;
    import com.google.code.ssm.spring.ExtendedSSMCacheManager;
    import com.google.code.ssm.spring.SSMCache;
    
    @Configuration
    public class CacheConfig extends CachingConfigurerSupport {
    
       @Value("${cache.address}")
       private String address;
       
       @Value("${cache.operationTimeout}")
       private String operationTimeout;
       
       @Value("${cache.cacheName}")
       private String cacheName;
       
       @Value("${cache.expiration}")
       private String expiration;
          
       @Bean
        public CacheManager cacheManager() {
    
            CacheConfiguration cacheConfiguration = new CacheConfiguration();
            cacheConfiguration.setConsistentHashing(true);
            cacheConfiguration.setUseBinaryProtocol(true);
            cacheConfiguration.setOperationTimeout(Integer.getInteger(operationTimeout));
            cacheConfiguration.setUseNameAsKeyPrefix(true);
            cacheConfiguration.setKeyPrefixSeparator(":");
    
            MemcacheClientFactoryImpl cacheClientFactory = new MemcacheClientFactoryImpl();
    
            AddressProvider addressProvider = new DefaultAddressProvider(address);
    
            CacheFactory cacheFactory = new CacheFactory();
            cacheFactory.setCacheName(cacheName);
            cacheFactory.setCacheClientFactory(cacheClientFactory);
            cacheFactory.setAddressProvider(addressProvider);
            cacheFactory.setConfiguration(cacheConfiguration);
    
            Cache object = null;
    
            try {
                object = cacheFactory.getObject();
            } catch (Exception e) {
                e.printStackTrace();
            }
            
            SSMCache ssmCache = new SSMCache(object, Integer.parseInt(expiration), true);
            
            ArrayList<SSMCache> ssmCaches = new ArrayList<>();
            ssmCaches.add(0, ssmCache);
            
            ExtendedSSMCacheManager ssmCacheManager = new ExtendedSSMCacheManager();
            ssmCacheManager.setCaches(ssmCaches);
            
            return ssmCacheManager;
        }
    }

# Service Method에서 호출시

    @Override
    @Cacheable(value = "testCache", key = "#itemId")
    public List<TestDto> getTestData(String itemId) {
       return testMapper.selectTestData(itemId);
    }
    
    @Override
    @CacheEvict(value = "testCache", key = "#itemId")
    public void evictTestData(String itemId) {
       log.info(itemId + "의 캐시 삭제");
    }

# 최초 호출시

    @2020-05-17 09:04:50.783  INFO 19324 --- [nio-8080-exec-8] com.google.code.ssm.spring.SSMCache      : Cache miss. Get by key 000001 from cache testCache
    2020-05-17 09:04:50.844  INFO 19324 --- [nio-8080-exec-8] com.google.code.ssm.spring.SSMCache      : Put '[TestDto(itemNm=테스트1, itemCnt=5)]' under key 000001 to cache testCache@

# 이후 호출시

    @2020-05-17 09:05:38.793  INFO 19324 --- [nio-8080-exec-4] com.google.code.ssm.spring.SSMCache      : Cache hit. Get by key 000001 from cache testCache value '[TestDto(itemNm=테스트1, itemCnt=5)]'@

# 기타 참고사항

*  SSM은 Spring AOP를 사용하여 pointcut 및 weaving을 하고 있다. Spring AOP의 proxy는 bean을 감싸고 있기 때문에 같은 class의 내부 호출은 annotation이 있더라도 적용되지 않는다. 즉 내부 메서드 호출로는 Cache가 적용되지 않는다. (http://blog.naver.com/PostView.nhn?blogId=tmondev&logNo=220725135383)
* Key 값은 250자까지만 허용되므로 보통 Parameter로 받은 Dto 자체를 Key로 쓸 순 없었고 별도의 Key 형식을 지정해야된다.
