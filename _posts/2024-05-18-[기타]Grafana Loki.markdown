---
layout: post
title:  "[기타]Grafana Loki"
date:   2024-05-18
author: EastGlow
categories: Back-end
---

# 들어가며

현재 우리팀에서 서비스 하고 있는 MSA 환경의 서비스들은 로그를 사내 인프라팀에서 제공하는 ES에서 수집하고 Kibana와 결합하여 데이터를 시각화 하거나 로그를 조회하는 용도로 사용하고 있다.
그런데 우리팀 뿐만 아니라 다른팀들도 같이 사용하고 있다보니 로그 수집 용량에 한계가 있어서 최대 2주까지만 로그 보관이 가능한 상황이었다. 운영 문의를 처리하다보면 2주는 기본이고 한달, 많게는 몇달 전 이슈까지 확인해야하는 상황이 허다하다.

그러다보니 팀 내에서도 2주는 너무 짧은 것 같다, 별도의 로그 수집 체계를 구축하는게 어떻겠냐 라는 의견이 나왔었고 결과적으로 이번글에서 다룰 Grafana Loki를 검토해보게 되었다. 스포(?)를 좀 하자면 결과적으로는 사용 안 하게 되었지만(이유는 마지막에 후술하겠다.) 검토 과정에서 정리한 내용들과 간단한 테스트 과정을 정리하여 글로 남겨두었다.

# Grafana Loki
> [https://github.com/grafana/loki?tab=readme-ov-file#loki-like-prometheus-but-for-logs](https://github.com/grafana/loki?tab=readme-ov-file#loki-like-prometheus-but-for-logs)
> 
> [https://grafana.com/docs/loki/latest/get-started/overview/](https://grafana.com/docs/loki/latest/get-started/overview/)

Loki는 수평 확장이 가능하고 가용성이 높은 다중 테넌트 로그 수집 시스템이다. Prometheus 프로메테우스에서 영감을 받아서 만들어졌다고 한다. 비용 효율적이고 작동하기 쉽게 디자인 되었다.

일반적인 로그 시스템과 다른 점은 로그 내용을 인덱싱하지 않고 각 로그 스트림에 대한 레이블 셋(메타데이터)을 인덱싱한다.

![](https://grafana.com/docs/loki/latest/get-started/loki-overview-2.png)

## 다른 로그 시스템과의 차이점
-   로그 전체를 인덱싱을 하지않으며, 압축되고 구조화되지 않은 로그를 저장하고 메타데이터만 인덱싱함으로써 Loki는 작동이 더 간단하고 실행 비용이 더 저렴
-   매트릭을 pull하여 수집하는게 아니라 반대로 push를 통해 수집받는(?) 방식에서 Prometheus와 다름
-   Prometheus에서 이미 사용하고 있는 것과 동일한 레이블을 사용하여 로그 스트림을 인덱스 및 그룹화하므로 이미 Prometheus에서 사용하고 있는 것과 동일한 레이블을 사용하여 지표와 로그 간에 원활하게 전환 가능
-   Pod 라벨과 같은 메타데이터는 자동으로 스크래핑되고 색인이 생성되어 Kubernetes  Pod 로그를  저장하는 데 특히 적합함
-   Grafana에서 기본 지원중 (Grafana v6.0 필요)

## Loki 서비스 구성
1. Loki: 로그 저장 및 쿼리 처리를 하는 역할 (server)
2. Promtail: 로그를 수집하여 Loki로 보내는 역할 (agent)
3. Grafana: 수집한 로그를 시각화하는 역할 (visualizer)

## Loki 아키텍처
1. 다중 테넌시: 다중 테넌트 모드에서 실행될 때 모든 데이터는 X-Scope-OrgID HTTP 헤더에서 가져온 테넌트 ID에 의해 분할될 수 있다. 다중 테넌트 모드가 아닐 땐 헤더는 무시되고 테넌트 ID는 "fake"라는 값으로 세팅된다. 이것은 인덱스와 저장된 청크를 나타내게 된다. 
2. 단일 저장소: Loki는 모든 데이터를 단일 객체 스토리지 백엔드에 저장한다.

## Loki 구성요소
![구성요소_다이어그램](https://grafana.com/docs/loki/latest/get-started/loki_architecture_components.svg)

## Loki 설치 및 구성
[https://github.com/grafana/loki/releases/tag/v2.9.6](https://github.com/grafana/loki/releases/tag/v2.9.6)

[https://grafana.com/docs/loki/latest/setup/install/](https://grafana.com/docs/loki/latest/setup/install/)

[https://grafana.com/docs/loki/latest/setup/install/local/](https://grafana.com/docs/loki/latest/setup/install/local/)

[https://grafana.com/docs/loki/latest/send-data/promtail/installation/](https://grafana.com/docs/loki/latest/send-data/promtail/installation/)

# 로컬에서 Grafana Loki 세팅해서 돌려보기

## 1. Docker 설치
[https://docs.docker.com/desktop/install/mac-install/](https://docs.docker.com/desktop/install/mac-install/)

![](/assets/post/20240518_1.png)

![](/assets/post/20240518_2.png)

## 2. centos7 이미지 설치 및 Docker 기본 세팅
공식 Docs: [https://grafana.com/docs/loki/latest/setup/install/docker/](https://grafana.com/docs/loki/latest/setup/install/docker/)

> centos8은 지원 기간이 종료되어서 yum install 시 제대로 동작하지 않고 보안적으로도 안 좋다해서 지원 기간이 남은 centos7을 쓰는게 좋다고 한다
>
> [https://dpcalfola.tistory.com/entry/error-%EB%8F%84%EC%BB%A4-Error-Failed-to-download-metadata-for-repo-appstream-Cannot-prepare-internal-mirrorlist-No-URLs-in-mirrorlist-%ED%95%B4%EA%B2%B0%EB%B0%A9%EB%B2%95](https://dpcalfola.tistory.com/entry/error-%EB%8F%84%EC%BB%A4-Error-Failed-to-download-metadata-for-repo-appstream-Cannot-prepare-internal-mirrorlist-No-URLs-in-mirrorlist-%ED%95%B4%EA%B2%B0%EB%B0%A9%EB%B2%95)

### 1) centos7 이미지 pull

1. docker 명령어로 받기 
> docker pull centos:centos7

2. Docker Desktop에서 받기
![](/assets/post/20240518_3.png)

### 2) Docker Volume create
> 후에 나올 단계 중에 Loki와 관련된 config yaml 파일을 받아놔야 하는 부분이 있는데 도커 컨테이너를 띄울 때 이 파일을 사용해야해서 공통으로 쓸 수 있는 볼륨을 만들고 centos7에 마운트해서 도커 이미지를 띄워줘야한다.

1. docker 명령어로 create (볼륨명은 자유롭게)
```
docker volume create {볼륨명}
```

2. Docker Desktop에서 create  
![](/assets/post/20240518_4.png)

그냥 좌측 메뉴 중 Volumes에 들어가서 Create하면 됨

### 3) Docker Network create

1. 생성할 컨테이너들을 하나의 네트워크로 묶어주기 위해 Docker Network를 생성해준다.
```
docker network create {네트워크명}
```

### 4) Docker Image run

1. 명령어 옵션을 쓰기 때문에 CLI에서 진행했음

```
docker run -it -d --name centos7 --network {네트워크명} -v {볼륨명}:/{볼륨 내 디렉토리명} centos:centos7
```

2. docker ps 명령어로 띄워진 컨테이너 확인이 가능하고 Docker Desktop에서도 확인 가능함
![](/assets/post/20240518_5.png)

![](/assets/post/20240518_6.png)

## 3. Loki, Promtail, Grafana 세팅

### 1) Loki, Promtail config yaml 받고 컨테이너 띄우기

1. 좀전에 띄운 centos 컨테이너에 접속해서 wget으로 config 파일 2개를 받아준다. 경로는 아까 생성한 data01 볼륨 안에 자유롭게 잡아준다.
```
cd /data01
mkdir loki-config
cd loki-config
wget https://raw.githubusercontent.com/grafana/loki/v2.9.4/cmd/loki/loki-local-config.yaml -O loki-config.yaml
wget https://raw.githubusercontent.com/grafana/loki/v2.9.4/clients/cmd/promtail/promtail-docker-config.yaml -O promtail-config.yaml
```

![](/assets/post/20240518_7.png)

2. 아래 명령어를 통해 Loki & Promtail 컨테이너를 띄워준다.
```
docker run --name loki --network {네트워크명} -d -v {볼륨명}:/{볼륨 내 디렉토리명} -p 3100:3100 grafana/loki:2.9.4 -config.file=/{볼륨 내 디렉토리명}/loki-config.yaml
docker run --name promtail --network {네트워크명} -d -v {볼륨명}:/{볼륨 내 디렉토리명} grafana/promtail:2.9.4 -config.file=/{볼륨 내 디렉토리명}/promtail-config.yaml
```

### 2) 최종적으로 Loki & Promtail 컨테이너가 올라와있고 정상 실행 중인걸 볼 수 있다.
- Loki
  ![](/assets/post/20240518_8.png)
  - 매트릭도 정상 출력되고 있다.
    ![](/assets/post/20240518_10.png)
- Promtail
  ![](/assets/post/20240518_9.png)

### 3) Grafana 컨테이너 띄우기

1. Loki를 사용하려면 Grafana는 당연히 있어야한다. Grafana도 컨테이너로 띄워주자
```
docker run -d -p 3000:3000 --name=grafana --network {네트워크명} -v {볼륨명}:/{볼륨 내 디렉토리명} grafana/grafana-enterprise
```

2. 이후 데이터소스 추가에 가서 Loki를 선택하여 아래와 같이 추가해보면 정상적으로 추가되는 것을 볼 수 있다.

![](/assets/post/20240518_11.png)
![](/assets/post/20240518_12.png)

## 4. Local Spring Boot에서 로그 보내보기

[https://www.baeldung.com/spring-boot-loki-grafana-logging](https://www.baeldung.com/spring-boot-loki-grafana-logging)

### 1) logback-spring.xml 설정

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>        
   <appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
        <http>
            <url>http://localhost:3100/loki/api/v1/push</url>
        </http>
        <format>
            <label>
                <pattern>app=${name},host=${HOSTNAME},level=%level</pattern>
                <readMarkers>true</readMarkers>
            </label>
            <message>
                <pattern>
                    {
                    "level":"%level",
                    "class":"%logger{36}",
                    "thread":"%thread",
                    "message": "%message",
                    "requestId": "%X{X-Request-ID}"
                    }
                </pattern>
            </message>
         </format>
     </appender>
     
     <root level="INFO">
        <appender-ref ref="LOKI" />
     </root>
</configuration>
```

### 2) Grafana에서 조회

![](/assets/post/20240518_13.png)

# 마치며

간략하게 Grafana Loki에 대해서 알아보고 로컬에서 Loki, Promtail(위 테스트 과정에서 직접적으로 수집에 쓰이진 않았지만), Grafana 컨테이너를 각각 띄워서 로그를 받아서 띄워주는 것까지 해보았다.

앞서 글 도입부에서 말했듯이 검토만 하고 끝났었는데 이유는 사내 ES를 이용하는 각 서비스의 로그를 최적화하는 작업을 선행했는데 생각보다 로그 수집량이 많이 줄어들었다.
그러다보니 기존보다 로그 수집 용량이 줄어들어 보관 기간을 오래 가져갈 수 있게 되었고 아마 당분간은 Grafana Loki를 통해 별도의 수집 체계를 구축할 일은 없을 듯 하다.

뭐 언젠가는 또 써볼 일이 생길 수도 있으니 이번 기회에 찍먹해볼 수 있어서 좋았다고 생각한다.
