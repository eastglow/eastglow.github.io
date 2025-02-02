---
layout: post
title:  "[PostgreSQL] ElectricSQL 맛보기"
date:   2025-02-02
author: EastGlow
categories: Database
---

# 1. ElectricSQL?

어느날 회사에서 업무를 하다가 팀장님이 잠깐 부르셔서 간 적이 있었다. 화면에는 ElectricSQL이라는 것을 띄워두시고 이것에 대해 한번 간단하게 사전 조사 겸 실제 업무에 활용할 수 있을지 살펴봐달라고 부탁하셨다.

마침 연말이라 2024년 한해동안 해왔던 업무나 문서 정리를 하던 참이라 급한 업무는 없었고 간단하게 설명해주시는 ElectricSQL이라는 것의 활용 목적, 컨셉 등을 들어보니 재밌겠다 싶어서 찾아보았다.

간단하게 한줄 요약하자면 PostgreSQL과 `Application 간의 Data Sync를 실시간으로 맞춰주는 서드파티앱?`이라고 하면 되려나? 우리가 흔히 아는 CDC의 역할을 하는 것이라고 보면 될 것 같다.

이것을 어떻게 업무에 활용해봤으면 하는지에 대해 팀장님이 간단하게 말씀해주시길, 현재 글로벌 캐시로 Redis를 사용 중인데 여기저기서 서비스되는 애플리케이션들이 동시에 바라봐야하는 글로벌 캐시 성격의 데이터 말고

- 여러 영역에서 공통으로 사용하는 공통코드
- 정말 바뀔 일이 없지만 바뀔 가능성이 있는 상수에 가까운(?) 데이터
- Redis에 넣어두고 쓰기엔 휘발되어 날아가면 위험한 데이터

위와 같은 성격의 고정 데이터들을 각 애플리케이션 로컬 캐시 혹은 로컬 저장소(ex. SQLite와 같은 가벼운 로컬 데이터베이스)에 넣어두고 최대한 네트워크 I/O나 외부 통신없이 데이터 정합성을 지키면서도 로컬에서 관리하여 빠르게 조회할 수 있게 할 수 없을까? 라는 고민에서 찾아보게 되었다고 하셨다.

즉, PostgreSQL 데이터베이스를 사용하면서 동시에 ElectricSQL을 사용하게 되면 아래와 같은 흐름으로 각 애플리케이션의 로컬 데이터 정합성을 유지하게 될 것이라 생각했다.

1. PostgreSQL에는 공통코드 테이블 `COMMON_CODE`가 있다.
2. k8s 클라우드 환경에서는 하나의 서비스 내에 여러 Pod가 띄워져 있을 수도 있고, 기존의 레거시 환경의 서비스에서는 한 서버에 여러 인스턴스가 띄워져 있을 수도 있다.
3. 이러한 환경에서 `COMMON_CODE`라는 테이블의 데이터를 모든 인스턴스에서 동일한 데이터를 가지게 하려면 Redis에 데이터를 올려두고 바라보는게 가장 관리가 쉽긴 하나, Redis에 문제가 생겨서 데이터가 날아가면 다시 적재해야하고 그 시간동안은 제대로 된 데이터 조회가 안 될 수도 있다.
4. 그렇기에 각 인스턴스에서 독자적으로 로컬 캐시 혹은 로컬 데이터베이스(ex. SQLite)에 `COMMON_CODE` 데이터를 가지게 하고 이 인스턴스들은 ElectricSQL을 구독하고 있다가 데이터 변경이 감지되면 로컬 데이터를 Update하는 식이다.

![](/assets/post/20250202_1.png)

위 텍스트를 사진으로 표현하자면 위 사진과 같은 컨셉이 될 것이다. 설명은 거창했지만 결국은 간단한 CDC 역할을 할 녀석이 ElectricSQL이라는 것이다.

# 2. 간단하게 Docker로 환경을 구성해서 테스트해보자

백문이 불여일견이라고 간단하게 Docker를 통해 PostgreSQL, ElectricSQL을 설치해서 테스트해보기로 했다.

## 1) PostgreSQL 세팅

ElectricSQL이 지원하는 최소 버전이 14 버전부터라서 PostgreSQL은 14 버전 이상으로 설치해주도록 한다.

![](/assets/post/20250202_2.png)

- 현재 지원하는 DB는 PostgreSQL 뿐인듯 하다.
- PostgreSQL 버전은 14 버전 이상이어야 한다.
- PostgreSQL의 `wal_level` 옵션을 `logical`로 설정해야한다. (로그 기반의 CDC를 위해)

```
--14버전 이미지 Pull
docker pull postgres:14

--14버전으로 컨테이너 띄우기
docker run -d \
  --name postgresql --network my-network -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_USER=user \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=test \
  -e TZ=Asia/Seoul \
  -p5432:5432\
  postgres:14\
  -c wal_level=logical \
  -c max_replication_slots=4\
  -c max_wal_senders=4
```

![](/assets/post/20250202_3.png)

위 명령어가 제대로 동작했다면 컨테이너가 무사히 올라간 것을 볼 수 있을 것이다.

![](/assets/post/20250202_4.png)

그리고 실제 PostgreSQL로 접속해서 test DB에 `common_code` 테이블도 생성해주고 데이터도 하나 넣어보았다.

## 2) ElectricSQL 세팅

```
-- 최신버전 이미지 Pull
docker pull electricsql/electric

-- 이미지로 docker run
docker run --name electric-sql --network my-network -e "DATABASE_URL=postgresql://user:password@postgresql:5432/test" -p 3000:3000 electricsql/electric
```

![](/assets/post/20250202_5.png)

ElectricSQL 역시 정상적으로 띄워진 것을 확인할 수 있을 것이다.

## 3) HTTP API를 호출해보자

### 3-1) 최초 init 요청

https://electric-sql.com/docs/api/http#syncing-shapes  
https://electric-sql.com/docs/guides/shapes#subscribing-to-shapes

![](/assets/post/20250202_6.png)

```
GET http://localhost:3000/v1/shape?table=common_code&offset=-1
```
- 이 요청을 최초에 해줌으로써 "내가 ElectricSQL을 통해서 PostgreSQL의 특정 Shape(데이터 모델을 뜻하는 듯하다.)을 구독하겠다."를 알리는 의미로 보인다.
- 즉, table명이 common_code이고 offset이 -1(최초 요청)인 shape를 조회하겠다는 뜻을 가지고 있다.

![](/assets/post/20250202_7.png)

호출에 성공했다면 위와 같은 response를 볼 수 있을 것이다.

### 3-2) 이후 변경되는 데이터 가져오기

이후 변경 데이터를 가져오는건 Long Polling 롱 폴링 방식으로 구현하라고 가이드가 되어 있는데 이때 호출하는 API는 아래와 같다.

![](/assets/post/20250202_8.png)

http://localhost:3000/v1/shape?table=common_code&live=true&offset=0_0&handle=44926195-1738486878772

맨 뒤에 붙는 handle은 위에서 호출했던 최초 /shape 요청의 response에 담겨있는 header > electric-handle 값을 사용해서 호출하면 된다. offset은 제일 처음엔 그냥 0_0으로 호출하고 그다음부터는 받은 response의 제일 마지막 offset값으로 호출하면 된다.

![](/assets/post/20250202_9.png)
- 위 사진 기준으로는 handle 값은 `44926195-1738486878772`가 된다.

#### 응답을 기다리는 동안 변경된 데이터가 없는 경우

위 URL로 호출해보면 아래와 같이 204 No Content를 받아볼 수 있을 것이다. 왜냐면 응답을 기다리는 동안엔 데이터가 변경된게 없기 때문이다.

![](/assets/post/20250202_10.png)

#### 응답을 기다리는 동안 변경된 데이터가 있는 경우

호출을 한 뒤 응답을 기다리는 동안 데이터를 새로 insert 해보았다. 그럼 응답을 기다리다가 바로 응답을 받아서 뱉어내는 모습을 볼 수 있을 것이다.

![](/assets/post/20250202_11.png)

만약 변경된 데이터를 받았다면 그 다음 호출부터는 offset 파라미터를 이전 response에 있는 header의 electric-offset을 사용해서 새로운 호출을 하고 다시 Long Polling 응답을 기다리면 된다.

(offset 파라미터의 의미가 "이 offset 이후에 변경된 데이터가 있는지 감지해라"라는 의미인거 같다.)

# 3. 마무리

그래서 현재 업무에 쓰고 있냐? 라고 묻는다면 그렇지 않다. 도입을 보류한 이유가 몇가지 있었는데,

1. 현재 서비스에 사용하고 있는 PostgreSQL 버전은 12 버전대 (이 이유가 가장 크다.)
2. 아직 Java, Spring과 관련된 라이브러리나 지원이 전무한 것으로 보인다. HTTP API를 가지고 Long Polling 방식으로 Data Sync를 직접 구현해야한다. (=배보다 배꼽이 커질 것 같다.)
3. 이럴거면 그냥 Kafka 커넥터를 이용하여 CDC를 구성하는게 더 낫지 않을까...라는 의문이 들었다.

결국엔 도입 검토를 위한 사전 조사 정도만 가볍게 하게 된 셈이고 현재 업무 도입은 무한정 보류해둔 상태이다. (아마도 안 하지 않을까?)

그래도 나름 흥미로운 녀석이라 글로 남겨두면 언젠가 또 기억나서 써먹어보지 않을까 해서 남겨둔다. 가벼운 용도의 CDC 목적으로 구성하기엔 괜찮은 녀석 같다.
