---
layout: post
title:  "[Kafka]한번에 여러 건의 메시지를 컨슘하는 BatchListener (feat. 멀티 스레드)"
date:   2022-10-24
author: EastGlow
categories: Kafka
---

보통 일반적인 카프카 리스너는 `public void consumeTest(ConsumerRecord<String, TestDto> record, ConsumerRecordMetadata meta)`에서 볼 수 있듯이 ConsumerRecords가 아닌 ConsumerRecord 단수 파라미터를 받게 된다.

한번에 하나의 레코드만 받아다가 하나의 메시지를 처리할 수 있다. 카프카 리스너를 일종의 배치처럼 특정 타이밍마다 혹은 어느정도의 메시지가 쌓였을 때만 컨슘하여 메시지를 한번에 처리할 수 있는 방법이 없을까 찾아보다가 배치 리스너를 찾을 수 있었다.

# 1. Consumer Config

	@Bean(name = "kafkaBatchListenerContainerFactory")  
	public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaBatchListenerContainerFactory() {  
		ConcurrentKafkaListenerContainerFactory<String, Object> factory = new ConcurrentKafkaListenerContainerFactory<>();  
		factory.setConsumerFactory(defaultConsumerFactory());  
		factory.setConcurrency(1);  
		factory.setBatchListener(true);  
		factory.setBatchErrorHandler(new SeekToCurrentBatchErrorHandler());  
		factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.BATCH);  

		return factory;  
	}

일반적인 리스너와 다른 점은 setBatchListener 옵션값을 true로 준다는 것과 setBatchErrorHandler를 통해 BatchErrorHandler를 별도로 등록한다는 점이다. BatchListener가 true인데 만약 BatchErrorHandler가 등록되어 있지 않다면 초기 구동 시 exception이 발생하게 된다.

	public void afterPropertiesSet() {  
		if (this.errorHandler != null) {  
			if (Boolean.TRUE.equals(this.batchListener)) {  
				Assert.state(this.errorHandler instanceof BatchErrorHandler, () -> {  
					return "The error handler must be a BatchErrorHandler, not " + this.errorHandler.getClass().getName();  
				});  
			} else {  
				Assert.state(this.errorHandler instanceof ErrorHandler, () -> {  
					return "The error handler must be an ErrorHandler, not " + this.errorHandler.getClass().getName();  
				});  
			}  
		}
	}

# 2. Consumer Class

실제로 메시지를 컨슘하여 처리하는 컨슈머 처리부에서는 아래와 같이 Consumer Config에서 배치 리스너 타입으로 만든 Container Factory를 설정하여 사용할 수 있다.

	@KafkaListener(topics = "TEST-TOPIC"  
		, groupId = "TEST-CONSUME-GROUP"
		, containerFactory = "kafkaBatchListenerContainerFactory"  
		, properties = {
			ConsumerConfig.FETCH_MIN_BYTES_CONFIG + ":5242880", // 5MB 찰 때까지 대기  
			ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG + ":5000" // or 5초까지 대기  
		}  
	)  
	public void consumeTest(ConsumerRecords<String, List<TestDto>> records) throws Exception {  

		for (ConsumerRecord<String, List<TestDto>> record : records) {  
			// something to do  
		}
	}

ConsumerRecords는 살짝 뜯어보면 내부적으로는 Iterable를 구현하고 있고 생성자를 보면 아래와 같이 List\<ConsumerRecord\>를 받으면서 쉽게 보면 ConsumerRecord를 List로 받는 구조라고 볼 수 있을 것 같다.

	public ConsumerRecords(Map<TopicPartition, List<ConsumerRecord<K, V>>> records) {  
		this.records = records;  
	}

그러다보니 실제로 배치 리스너를 사용하는 컨슈머 메소드에서 파라미터로 받는 ConsumerRecords를 받아서 사용하려고 보면 forEach나 iterator를 볼 수 있다. 쌓인 메시지들을 리스트 형태로 전달받아 사용할 수 있는 것이다.

그리고 @KafkaListener에 달아둔 설정값 중 ConsumerConfig와 관련된 값을 2개 추가해둔 것을 볼 수 있는데 이러한 값들을 통해 배치 리스너가 얼마 간의 간격으로 혹은 얼만큼의 메시지가 쌓이면 컨슘해올 것인지를 설정해줄 수 있다. 나도 아직 슈도코드 느낌으로 배치 리스너가 다건의 메시지를 한번에 컨슘해올 수 있는지 정도만 확인해본 상태라 각 Config에 따른 배치 리스너 컨슘 동작의 변화는 다른 글들을 참고해보는 것이 더 도움이 될 것 같다.

# 3. feat. 멀티 스레드?

멀티 스레드는 카프카와 관련된 행위는 아니고 이렇게 배치 리스너로 가져온 다건의 메시지를 조금이라도 더 빠른 처리 퍼포먼스를 내고 싶어서 생각해보다가 간단하게 구현해보고 테스트 해봤던 부분이다.

## Async Config

	@Configuration  
	@EnableAsync  
	public class AsyncConfig extends AsyncConfigurerSupport {  

		@Override  
		public Executor getAsyncExecutor() {  
			ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();  
			executor.setCorePoolSize(10);  
			executor.setMaxPoolSize(50);  
			executor.setQueueCapacity(500);  
			executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy()); // 더이상 대기할 큐 공간이 없으면 exception을 요청한 스레드에서 직접 처리하도록 한다  
			executor.setWaitForTasksToCompleteOnShutdown(true); // 스레드에서 현재 작업 중인 task를 완료한 후 종료  
			executor.initialize();  

			return executor;  
		}  
	}

멀티 스레드 사용을 위한 Config는 워낙 설명이 잘 되어 있는 글들이 많기 때문에 그냥 가볍게 언급하고 넘어가도록 하겠다. 위처럼 별도의 Configuration을 만들고 @EnableAsync를 선언하여 @Async가 붙은 메소드는 비동기로 처리하겠다고 설정할 수 있다.

한가지 멀티 스레드로 Async 메소드를 테스트하다보니 이슈가 있었던게 단시간에 엄청나게 많은 호출이 있는 메소드의 경우 메소드를 호출할 때마다 스레드가 생기다보니 설정해둔 스레드 풀 사이즈와 큐 Capa를 넘어서는 경우가 허다했다. 그러다보니 처리되지 못하고 exception이 나버리는 경우가 생기게 되었고 이것을 어떻게 처리해야하나 싶어서 찾아보니 그냥 테스트를 계속 해보면서 스레드 풀 사이즈의 최적을 찾아내든가, 아니면 setRejectedExecutionHandler를 설정하여 CallerRunsPolicy() 정책으로 적용될 수 있게 하는 방법이 있었다.

이걸 설정하게 되면 기존엔 모든 스레드 풀 및 대기 큐가 꽉 차게 되면 발생하던 RejectedExecutionException이 발생하지 않고 그 exception이 발생한 스레드에서 처리하지 못한 작업을 재시도하게 된다. 즉, Caller에게 재처리를 위임하게 되는 것이다. 이렇게 설정해두면 더 이상 RejectedExecutionException이 발생하지 않으며(*명시적으로 이 exception을 발생시키지 않는 한 발생할 일이 없다고 한다.) 스레드 풀 및 대기 큐가 메소드의 호출양보다 적더라도 처리가 되는 것을 볼 수 있다. 오늘 포스팅에서 멀티 스레드를 중점적으로 다룰려고 했던 것은 아니기 때문에 이정도만 하고 간단하게 넘어가도록 한다.

# 4. 마치며

아직 배치 리스너롤 실제 업무 영역에 적용하여 성능이나 로직 상의 이슈 등을 완벽하게 체크하고 써본건 아니기 때문에 오늘은 간단하게 이런게 있다~정도로만 짚고 넘어가려고 한다. 추후에 배치 리스너를 적용하여 제대로 써볼 일이 생기면 다시 다뤄보도록 하겠다.

> 참고:  
> https://yaboong.github.io/spring/2020/06/07/kafka-batch-consumer-unintended-listener-invoking/  
> https://docs.spring.io/spring-kafka/reference/html/  
