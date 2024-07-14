---
layout: post
title:  "[Java]Redis, Lettuce, Failover"
date:   2024-07-14
author: EastGlow
categories: Back-end
---

# 1. 들어가며

우리팀은 캐시 시스템으로 전사 공용 Couchbase를 사용하다가 2023년 하반기에 우리 팀 전용으로 Redis를 구축하였다.

Redis를 구축하고 제대로 사용한 지도 반년정도(fyi. 최초로 글을 작성해뒀던 시점) 된 거 같은데 그동안 운이 좋아서 문제가 없었을 뿐이지 언제든 Redis 서버에 문제가 생겨서 애플리케이션에 영향을 줄 수 있을 듯 했다.

> 여기서 잠깐, Couchbase에서 Redis로 넘어온 이유
> - Couchbase가 전사 공용이다보니 사용 및 확장에 제한이 많고 이것의 사용을 위해 사내 라이브러리 디펜던시가 생김
>   - 이 디펜던시 때문에 다른 캐시(로컬 캐시나 Redis 같은 다른 글로벌 캐시)를 쓰고 싶어도 Configuration 구성 과정에서 충돌이 생겨서 사용을 못했음
> - Redis는 여러가지 데이터 구조를 지원하기도 했고 Couchbase보다 관련 자료들이 많아서 운영하기가 좋을 듯 했음

그러다가 어느날, 아래와 같은 로그가 우리팀 컨슈머 서비스에 올라왔고 Redis와 관련된 처리부에서 실패 오류가 난 것을 확인할 수 있었다.

```
2024-00-00T00:00:05.473369797Z [2m2024-00-00 00:00:05.473[0;39m [32m INFO[0;39m [35m1[0;39m [2m---[0;39m [2m[xecutorLoop-1-2][0;39m [36mi.l.core.protocol.ConnectionWatchdog    [0;39m [2m:[0;39m Reconnecting, last destination was /x.x.x.x:1001
2024-00-00T00:00:05.476219209Z [2m2024-00-00 00:00:05.475[0;39m [32m INFO[0;39m [35m1[0;39m [2m---[0;39m [2m[ioEventLoop-6-2][0;39m [36mi.l.core.protocol.ReconnectionHandler   [0;39m [2m:[0;39m Reconnected to x.x.x.x:1001
2024-00-00T00:00:07.673862592Z [2m2024-00-00 00:00:07.673[0;39m [32m INFO[0;39m [35m1[0;39m [2m---[0;39m [2m[xecutorLoop-1-1][0;39m [36mi.l.core.protocol.ConnectionWatchdog    [0;39m [2m:[0;39m Reconnecting, last destination was /x.x.x.x:1000
2024-00-00T00:00:07.677330394Z [2m2024-00-00 00:00:07.677[0;39m [32m INFO[0;39m [35m1[0;39m [2m---[0;39m [2m[ioEventLoop-6-1][0;39m [36mi.l.core.protocol.ReconnectionHandler   [0;39m [2m:[0;39m Reconnected to x.x.x.x:1000
2024-00-00T00:00:07.691202597Z org.springframework.data.redis.RedisSystemException: Redis exception; nested exception is io.lettuce.core.RedisException: io.lettuce.core.RedisException: Currently not connected. Commands are rejected.
```

Redis Exception으로 인해 해당 서비스 메서드의 실행 도중에 오류가 났고 결과적으로 로직 실행이 완료되지 못했었다. (다행히 실패날 경우 재처리 될 수 있게 별도 로직이 있어서 큰 이슈는 없었다.)

위 오류가 발생했던 당시에 애플리케이션에서 Redis 서버로 지속적인 Reconnect를 왜 하는지 찾아봤었는데 당시 기억으로는 ConnectionWatchdog > ReconnectionHandler에서 Redis 서버가 살아있는지 체크하는, 일종의 Health Check가 있었던 걸로 기억한다.

어쨌든, 이 Health Check가 되는 시점과 Redis 캐시 처리 시점이 절묘하게 맞물려서 Redis Command가 실행 거부로 떨어졌고 그로 인해 Exception이 발생했었다. 이때 일을 계기로 Redis Lettuce 환경에 대한 failover 테스트를 해보기로 결심했다.

# 2. Redis Cluster Mode

일반적으로 Redis를 사용한다고 하는 곳들은 대부분 Standalone이나 Sentinel로 구성하기 보단 Cluster 환경을 구축하는 것으로 보인다. Cluster Mode는 16383개의 슬롯을 각 노드가 나눠가지는 구조인데 데이터들은 이 슬롯에 각각 저장되게 된다.

그렇기 떄문에 예를 들어 "test"라는 키의 데이터와 "test1"이라는 키의 데이터는 같은 노드에 저장이 되지 않을 수 있고, "test"가 있는 노드에 접속하여 "test2"를 조회했을 때 나오지 않을 수 있다. (redis-cli에서는 -c 옵션으로 붙으면 타 노드의 데이터도 조회가 가능하긴 하다.)

> 참고로 클러스터 환경에서도 특정 노드에만 키가 저장되게 할 수 있다. Hash Tag라는 것을 이용하면 되는데 {test}:1234와 {test}:5678과 같이 {} 안에 같은 키를 지정하면 같은 노드로 향하게 된다.
>
> [https://redis.io/docs/reference/cluster-spec/#hash-tags](https://redis.io/docs/reference/cluster-spec/#hash-tags)

> 이 글에서는 클러스터 환경, 구성 등에 대한 자세한 설명이나 정보는 다루지 않겠다.

우리팀 Redis도 이러한 Cluster 환경을 구축해놨고 3대의 물리 서버에 각 서버마다 마스터-슬레이브 노드를 1개씩 띄워놨다. (총 3대의 서버, 3개의 마스터, 3개의 슬레이브가 존재)

# 3. failover 테스트와 관련된 옵션들

## 1) ClientOptions.disconnectedBehavior

보통 RedisException이 발생하게 되는 옵션 중에는 Lettuce에 있는 `disconnectedBehavior` 옵션에 따라 갈리게 되는데 아래와 같다.

1. DEFAULT: 아무것도 설정하지 않았을 때 동작하는 기본 옵션이며 autoReconnect값에 따라 다르게 동작한다. true인 경우엔 ACCEPT_COMMANDS, false인 경우엔 REJECT_COMMANDS로 동작한다. (autoReconnect는 기본 true이다. 즉, ACCEPT_COMMANDS로 동작하게 된다.)
2. ACCEPT_COMMANDS: 이 옵션은 만약 Redis Command를 날렸는데 서버 연결이 되지 않은 상태일 때, 서버가 연결될 때까지 기다리게 된다. 이 기다리는 시간에 해당되는 `commandTimeout`이 있으며 기본값은 1분이다. 
   - 1분동안 서버가 재연결되지 않으면 명령어 실행이 실패하며 아래 에러가 발생한다.  
   `org.springframework.dao.QueryTimeoutException: Redis command timed out; nested exception is io.lettuce.core.RedisCommandTimeoutException: Command timed out after 1 minute(s)`
3. REJECT_COMMANDS: 이 옵션은 만약 Redis Command를 날렸는데 서버 연결이 되지 않은 상태일 때, 바로 Exception이 발생하게 된다. 즉, 서버가 재연결될 때까지 기다리지 않는다.    
   `org.springframework.data.redis.RedisSystemException: Redis exception; nested exception is io.lettuce.core.RedisException: io.lettuce.core.RedisException: Currently not connected. Commands are rejected.`

그래서 이 옵션은 서비스 성격에 따라 사용하면 될 것 같은데 만약 명령어가 실행될 노드가 장애가 났을 경우, 해당 명령어가 즉각 실패로 떨어지고 exception 상황을 인지할 수 있게 하려면 `REJECT_COMMANDS` 옵션을 쓰면 좋을 것 같고 조금의 시간(=timeout)이 걸리더라도 일단 재연결을 기다려보자! 하면 `ACCEPT_COMMANDS`를 쓰면 될 듯 하다.

어쨌거나 `ACCEPT_COMMANDS` 옵션을 사용하게 되면 timeout 설정 시간만큼의 지연 시간이 생기기 때문에 서로 장단점이 있는 옵션이라 생각된다.

## 2) ClusterTopologyRefreshOptions

Cluster 환경에서의 failover에 가장 중요한(?) 역할을 한다고 생각되는 옵션이다. 이 옵션이 없으면 운영 중이 노드 중 하나가 장애가 발생했을 때 애플리케이션 쪽에서는 해당 노드가 장애 상태인지 제대로 인지하지 못한다. 즉, 애플리케이션은 해당 노드로 Redis 명령어를 날려서 exception이 발생하기 전까지는 장애가 생긴 노드인 것을 모르는 것이다.

그래서 아래 참고 자료들을 보면 Redis를 Cluster 모드로 운영한다면 Lettuce Config에 ClusterTopology 설정은 웬만하면 필수적으로 해야한다고 한다.

> [https://lettuce.io/core/release/reference/#clientoptions.cluster-specific-options](https://lettuce.io/core/release/reference/#clientoptions.cluster-specific-options)
>
> [https://blog.leocat.kr/notes/2022/04/15/lettuce-config-for-redis-cluster-topology-refresh](https://blog.leocat.kr/notes/2022/04/15/lettuce-config-for-redis-cluster-topology-refresh)

처음 이 옵션들을 가지고 failover 테스트를 할 땐 정확히 어떤 차이가 있는지 잘 몰랐었는데 상황을 아래와 같이 나눠서 확인해보니 그 의미를 알 수 있었다.

### 2-1) ClusterTopologyRefreshOptions가 없을 때

먼저 같은 노드로 계속 명령어가 날아갈 수 있도록 {test}:1111과 같이 Hash Tag를 지정하여 테스트 중에는 특정 노드로 계속 명령어가 갈 수 있도록 하였다.

1. 최초로 set 명령어를 날렸을 때 x.x.x.x:1000 노드로 명령어가 실행된 것을 볼 수 있었다.
2. x.x.x.x:1000 노드를 shutdown하면 ConnectionWatchdog와 ReconnectionHandler에 의해 지속적으로 재연결 시도를 하는 것을 볼 수 있었다.
3. 다시 set 명령어를 날렸을 때, x.x.x.x:1000는 아직 내려간 상태이기 때문에 `REJECT_COMMANDS` 옵션에 의해 바로 명령어 실행이 거부되는 것을 볼 수 있었다. 여기서 알 수 있는 점은 장애가 발생한 노드인지 아닌지 애플리케이션 쪽에서는 명령어를 날리기 전까지 모르는 것이다. 그래서 명령어를 날려보고 나서야 안 되는 것을 알고 Exception을 발생시켰다.

### 2-1) ClusterTopologyRefreshOptions가 있을 때

이 옵션이 설정되고 난 뒤는 상황이 달라진다. 이번에도 똑같이 x.x.x.x:1000 이 노드를 다운시켜 보았다.

1. 최초로 set 명령어를 날렸을 때는 기존과 별 다를 바 없어보였다.
2. 하지만 마스터 노드를 shutdown했을 때 좀 다른 점을 볼 수 있었다. 지속적으로 재연결을 시도하는 것은 똑같지만 중간중간에 `ClusterTopologyRefreshScheduler`, `ClusterTopologyRefresh`에서 로그가 발생하는 것을 새롭게 볼 수 있었다. 그리고 로그를 더 살펴보니 `0ada2e53b1cee67511cff699ff90dd7e24ec043f x.x.x.x:1000@1000 master - 0 1711528303096 23 disconnected 5461-10922` 와 같이 연결이 끊긴 것을 인지하고 있는 것을 볼 수 있었다.
3. 그리고 역시나 똑같이 shutdown 된 이후에 다시 set 명령어를 날려보았는데 ClusterTopology 옵션을 설정하고 난 뒤에는 에러가 나지 않고 다른 노드로 정상적으로 실행되는 것을 볼 수 있었다.

# 4. 최종적으로 설정한 옵션

```
@Bean
@ConditionalOnMissingBean
public RedisConnectionFactory redisConnectionFactory(RedisProperties redisProperties) {

	private final CustomRedisProperties customRedisProperties;

	// topology 자동 업데이트 옵션 추가
	// enablePeriodicRefresh(tolpology 정보 감시 텀) default vaule : 60s
	ClusterTopologyRefreshOptions clusterTopologyRefreshOptions = ClusterTopologyRefreshOptions.builder()
			.enablePeriodicRefresh(Duration.ofSeconds(60L))
			.enableAllAdaptiveRefreshTriggers()  // MOVED, ASK, PERSISTENT_RECONNECTS, UNCOVERED_SLOT, UNKOWN_NODE trigger시 refresh 진행
			.adaptiveRefreshTriggersTimeout(Duration.ofSeconds(60L))
			.build();

	CustomRedisProperties.ClientOptions customClientOptions = customRedisProperties.getClientOptions();

	ClientOptions clientOptions = ClusterClientOptions.builder()
			.disconnectedBehavior(StringUtils.isBlank(customClientOptions.getDisconnectedBehavior()) ? ClientOptions.DisconnectedBehavior.DEFAULT : this.getDisconnectBehavior(customClientOptions.getDisconnectedBehavior()))
			.topologyRefreshOptions(clusterTopologyRefreshOptions)
			.build();

	Duration commandTimeout = customClientOptions.getCommandTimeout();

	LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
			.clientOptions(clientOptions)
			.commandTimeout(commandTimeout == null ? Duration.ofSeconds(30L) : commandTimeout)
			.readFrom(ReadFrom.REPLICA_PREFERRED)
			.build();
			
	RedisClusterConfiguration redisClusterConfiguration = new RedisClusterConfiguration(redisProperties.getCluster().getNodes());
	redisClusterConfiguration.setPassword(redisProperties.getPassword());

	return new LettuceConnectionFactory(redisClusterConfiguration, clientConfig);
}
```

[https://github.com/redis/lettuce/wiki/Client-Options](https://github.com/redis/lettuce/wiki/Client-Options)

- 아래 값들은 application.yml 설정값을 통해 별도로 받고 있으나, 기본값을 기준으로 설명해두었다.
- ClusterTopologyRefreshOptions
  - enablePeriodicRefresh(Duration.ofSeconds(60L)): 60초마다 주기적으로 topology 정보를 업데이트 할 수 있도록 한다. 
  - enableAllAdaptiveRefreshTriggers(): MOVED, ASK 등 일부 명령어가 실행될 시 업데이트 할 수 있도록 한다. 
  - adaptiveRefreshTriggersTimeout(Duration.ofSeconds(60L)): 2번째 옵션과 연관된 옵션인데 특정 명령어에 의해 업데이트가 될 때 너무 잦은 업데이트가 일어나면 문제가 될 수 있기 때문에 1번 업데이트가 됐으면 60초 동안은 업데이트가 일어나지 않게 하는 옵션이다.
- ClientOptions.disconnectedBehavior: 팀 내부 논의를 통해 Redis를 정상적으로 사용할 수 없는 상태일 땐 타임아웃 시간만큼 지연되는 것보단 바로 REJECT 처리되어 에러나는 편이 나을 것 같다는 의견이 대다수라 REJECT_COMMANDS를 적용하였다.
- LettuceClientConfiguration.commandTimeout 
  - 명령어를 날렸을 때 실행 지연을 보장받는 시간 설정이다. ACCEPT_COMMANDS 설정을 했을 때 유효한 값일 것이고 60초는 너무 길 것 같아서 30초로 설정해두었다.

# 5. 해결된 줄 알았으나...

## 1) 재연결 과정에서 마스터&슬레이브 복제 이슈

```
2024-00-00 00:00:00.244  WARN 1 --- [ioEventLoop-6-1] i.l.core.StatefulRedisConnectionImpl     : READONLY failed: LOADING Redis is loading the dataset in memory
2024-00-00 00:00:00.748  WARN 1 --- [xecutorLoop-1-2] i.l.c.c.topology.ClusterTopologyRefresh  : Cannot retrieve partition view from RedisURI [host='x.x.x.x', port=1000], error: java.util.concurrent.ExecutionException: io.lettuce.core.RedisLoadingException: LOADING Redis is loading the dataset in memory
```

[https://github.com/redis/lettuce/issues/2560](https://github.com/redis/lettuce/issues/2560)

위 이슈에 의하면 Lettuce 6 버전부터 해결되었다고 한다. 현재 팀에서 쓰는 spring-data-redis에 엮인 버전은 5.3.7 버전이라 일단은 피할 수 없는 이슈 같다. (추후 버전업해서 테스트 예정)

> In Lettuce 6.0 we improved the handshake process to disconnect the connection and attempt a reconnect later on when LOADING is gone.

## 2) Redis Exception 발생 시 핸들링

사례들을 찾아보니 크게 아래 2가지 정도로 처리하고 있는 듯 하였다. 우리팀은 내부 논의를 통해 CacheErrorHandler를 구성하기로 했다.

- CacheErrorHandler 
- Resilience4J의 서킷 브레이커와 같은 것을 활용한 fallback method 구성
  
> [https://velog.io/@chs98412/%EB%A0%88%EB%94%94%EC%8A%A4-%EC%9E%A5%EC%95%A0-%EC%B2%98%EB%A6%AC%ED%95%98%EA%B8%B0](https://velog.io/@chs98412/%EB%A0%88%EB%94%94%EC%8A%A4-%EC%9E%A5%EC%95%A0-%EC%B2%98%EB%A6%AC%ED%95%98%EA%B8%B0)

### 2-1) @Cacheable 관련 failover 테스트

별도의 Redis 에러 핸들링이 없는 상태에서 @Cacheable이 붙은 메서드를 호출했을 때, Redis 서버 장애가 나면 어떻게 될까?

일반적으로 READ의 경우엔 현재 클라이언트 쪽 설정인 `LettuceClientconfiguration.readFrom` 옵션이 `ReadFrom.REPLICA_PREFERED`로 되어있는데 이 옵션은 조회의 경우 slave 노드에서 먼저 읽고 slave 노드가 불능 상태이면 master 노드에서 읽겠다는 것이다.
> [https://github.com/redis/lettuce/wiki/ReadFrom-Settings](https://github.com/redis/lettuce/wiki/ReadFrom-Settings)

그리고 아래 테스트에서 하나 더 중요한 옵션이 있는데 Redis 서버 쪽에 있는 `cluster-require-full-coverage`가 그것이다. 현재 우리팀 Redis는 `yes`로 되어 있는데 이경우엔 master 노드 3개 중 1개만 죽어도 모든 노드 운영이 불가해지는 옵션이다.
> [http://www.redisgate.com/redis/cluster/cluster-require-full-coverage.php](http://www.redisgate.com/redis/cluster/cluster-require-full-coverage.php)

#### 2-1-1) slave 노드 shutdown

`ReadFrom.REPLICA_PREFERED` 옵션에 의해 @Cacheable이 실행되는 조회(REDIS get)의 경우엔 slave 노드로 연결이 된다. 그래서 먼저 slave 노드를 shutdown 해보았다. 이경우엔 별 문제없이 바로 master 노드를 바라보게 되어 따로 에러없이 잘 조회되었다.

#### 2-1-2) master 노드 shutdown

slave 노드에 이어 master 노드까지 shutdown하면 어떻게 될까? 결과부터 말하자면 클라이언트는 완전 불능 상태가 되어버린다. 이미 다른 2대의 master는 각각 1개씩 slave를 가지고 있는 상태이고 이 상태에서 slave를 다운시키고 뒤이어 그 slave의 master를 다운시켰으니 결과적으로 master 3대 중 1대는 아예 불능 상태가 된 것이다.

그렇기 때문에 `cluster-require-full-coverage=yes` 설정에 의해서 master 3대 중 1대가 죽어버려서 모든 master 노드가 서비스 불가 상태로 바뀐다.

살아있는 master 노드에 들어가서 cluster info를 실행해보면 이 노드는 멀쩡하지만 cluster_state가 fail 상태인 것을 볼 수 있었다. 그래서 @Cacheable을 설정한 메서드를 실행해보면 아래와 같이 타임아웃이 발생하며 정상적인 메서드 실행이 안 되는 것을 확인할 수 있었다.

`org.springframework.dao.QueryTimeoutException: Redis command timed out; nested exception is io.lettuce.core.RedisCommandTimeoutException: Command timed out after 30 second(s)`

> ACCEPT_COMMANDS 옵션으로 테스트해서 타임아웃이 난거고 REJECT_COMMANDS였다면 타임아웃 없이 바로 Reject Exception이 발생한다.

위 상황이 꼭 @Cacheable에만 국한되는 것이 아니라 Redis 서비스 자체가 불가해진 상황이라 애플리케이션 전역에 영향을 미치게 된다. 여기서 확인해보고 싶었던 것은 *별도의 에러 핸들링이 없을 때 @Cacheable이 있는 메서드에서 Redis 에러가 발생하면 어떻게 될까* 였다.

**요약하자면 아무런 에러 핸들링없이 @Cacheable을 사용하면 Redis 장애 발생 시 해당 메서드 실행 자체가 에러나게 된다.**

## 3) 그래서 CacheErrorHandler를 만들었다.

Redis Exception 발생 시 핸들링 하는 방법 중 하나인 CacheErrorHandler를 등록해보려고 한다. 이 핸들러를 통해 Cache와 관련된 Exception을 전역적으로 관리할 수 있다.

### 3-1) Cache 관련 Configuration이 초기화되는 순서

1. AbstractCachingconfiguration
   1. AbstractCachingConfigruation은 @Configuration이 붙어있기 때문에 Spring 초기화 과정에서 자동으로 초기화 대상으로 선정된다.
   2. 현재 프로젝트엔 CachingConfigurerSupport를 상속받은 별도의 CustomRedisAutoConfiguration이 존재하고, 이것은 AbstractCachingConfigruation의 configureres 객체에 담겨있는 것을 볼 수 있었다. (1개만 설정 가능하다.)
2. ProxyCachingConfiguration
   1. 이후 AbstractCachingConfiugration을 상속받은 ProxyCachingConfiugration에 의해 CacheInterceptor bean이 생성된다.
3. CacheInterceptor, CacheAspectSupport
   1. CacheInterceptor는 CacheAspectSupport를 상속받고 있고 CacheAspectSupport에 있는 configure에 의해 errorHandler가 선택되어 세팅된다.

### 3-2) SimpleCacheErrorHandler

1. 별도의 Custom CacheErrorHandler를 등록하지 않으면 등록되는 기본 핸들러이다. CacheAspectSupport 클래스에서 보면 this.errorHandler가 new SingletonSupplier에 담기는 것을 볼 수 있다. 이 클래스에서 볼 수 있듯이 별도의 errorHandler 객체를 받았으면 그걸 쓰고 아니면 SimpleCacheErrorHandler::new를 하여 사용한다. 
2. 이 핸들러에서는 기본적으로 Cache Exception이 발생하면 바로 throw를 하고 있다. 그래서 만약 @Cacheable이라든지 Redis Operation을 통해 Redis Command를 실행했는데 에러가 발생하면 해당 에러가 바로 throw 되어 정상적으로 메서드 실행을 마칠 수 없게 된다.
3. extends CachingConfigurerSupport
   1. 별도의 Custom CacheErrorHandler를 등록하려면 위 CachingConfigurerSupport를 상속받아 @Override 메서드를 구현해야한다.

```
@Configuration
@ConditionalOnProperty("spring.redis.host")
@AutoConfigureBefore(RedisAutoConfiguration.class)
@EnableConfigurationProperties(CustomRedisProperties.class)
public class CustomRedisAutoConfiguration extends CachingConfigurerSupport {

    (생략)...

    @Override
    public CacheErrorHandler errorHandler() {
        log.info("### customCacheErrorHandler init");
        return new CustomRedisCacheErrorHandler();
    }

    (생략)...

}
```

위와 같이 CustomRedisAutoConfiguration 클래스에서 CachingConfigurerSupport를 상속받아서 `errorHandler()` 메서드를 오버라이드해서 직접 Custom CacheErrorHandler를 return하도록 설정하면 끝이다.

그러면 이제는 exception을 throw하지 않고 로그만 찍고 끝낸 뒤 기존 메서드 로직을 정상적으로 이어가게 된다.

우리 서비스 같은 경우에는 위처럼 캐시 영역에 장애가 났을 때 직접 DB를 조회하더라도 DB 리소스에 어느정도 여유가 있기에 가능한 설정이라고 생각한다.

일반적으로는 아래 3가지 정도의 방법으로 대응할 수 있을 것 같다.

1. 일정 트래픽 이상이 들어오면 차단되도록 Rate Limiter를 설정하거나
2. 서킷 브레이커로 일정 장애 비율 이상 넘어가면 요청 자체가 차단되도록 하거나
3. 글로벌 캐시에 장애가 발생하면 로컬 캐시가 동작하도록 구성하거나
