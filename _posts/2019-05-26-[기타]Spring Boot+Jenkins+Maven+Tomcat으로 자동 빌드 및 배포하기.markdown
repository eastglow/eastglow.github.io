---
layout: post
title:  "[기타]Spring Boot+Jenkins+SVN+Maven+Tomcat 자동 빌드 및 배포하기"
date:   2019-05-26 21:00:00
author: EastGlow
categories: 기타
---

## 들어가기 전에...

Jenkins는 예전부터 써봐야지 생각은 하고 있었는데 사내에서는 써볼 수 있는 곳이 없었고 고객사에는 이미 시스템이 다 구축되어 있어서 Jenkins를 끼워넣기에는(?) 너무 할 일도 많아지고 혹시나 모를 위험에 대한 부담도 컸다. 그러다가 이번에 사내에 작은 프로젝트를 하나 시작하게 되었는데 이것을 사내 서버에 자동으로 빌드부터 배포까지 할 수 있도록 Jenkins를 세팅해보기로 하였다. 처음 세팅해보는 것이기도 하고 아직 모르는 부분이 많기 때문에 미흡한 글일 수 있으니 미리 양해를 구한다.

## 환경

프레임워크 : Spring (Spring Boot 2.1.4)
OS : Windows Server 2019
Subversion : SVN
WAS : Tomcat 8.0
빌드 : Maven
패키징 방식 : WAR

## Jenkins 설치 및 세팅

[Jenkins 다운로드 링크](https://jenkins.io/download/)에 들어가서 LTS 영역에 있는 Windows를 클릭하면 Windows용 Jenkins를 다운받을 수 있다. 설치방법과 초기세팅 방법은 구글링만 해도 너무 많이 나오기 때문에 넘어가도록 하겠다.

### Jenkins Global Tool Configuration

Jenkins에서 사용할 전역 설정들을 설정하는 곳이다. Jenkins 메인 화면에서 왼쪽 메뉴 중 Jenkins 관리 > Global Tool Configuration을 클릭한다. 여기서 설정해줘야 할 부분은 JDK, Maven이다.

#### JDK

JDK 부분을 보면 Name, JAVA_HOME 부분이 있다. Name은 그냥 JDK 1.8 등으로 지어주면 되며 JAVA_HOME은 JDK 설치 경로를 적어주면 된다.

![](/assets/post/20190526_1.PNG)

#### Maven

JDK와 마찬가지로 Name을 적어주고 MAVEN_HOME에는 경로를 적어준다. 혹은 Install automatically를 체크하면 빌드 시 자동으로 Maven을 Install하여 사용하게 된다. 까는게 귀찮다면(...) 이 옵션을 사용하면 될 것 같다.

![](/assets/post/20190508_2.PNG)

## Jenkins와 SVN 연결하기

Jenkins를 설치하고 계정 생성 후 기본적인 세팅을 다 하고 나면 "Jenkins에 오신 것을 환영합니다." 라는 메시지를 볼 수 있는 메인화면을 보게 될 것이다. 여담으로 나는 Jenkins의 기본 포트번호를 다른 것으로 바꾸었는데 바꾸고나서 재시작하고 다시 로그인하니 빈 페이지가 뜨면서 아무것도 나오지 않았다. 이럴땐 `1) Jenkins 재시작`을 해보고 안되면 `2) 재설치`를 해보도록 하자.

아무튼 모든 것을 제대로 하였다면 메인화면이 잘 나올 것이고 왼쪽 메뉴 중 `새로운 Item`을 클릭한다. 아이템 이름을 입력하고 `Freestyle project`를 클릭한다. 여기서 입력한 아이템 이름으로 Jenkins 설치경로/workspace에 SVN에서 받은 소스들이 저장된다. 예를 들어 아이템 이름을 `test`를 줬다면 `Jenkins 설치경로/workspace/test`라는 경로에 SVN에서 받은 소스들이 저장된다.

아래에서 별도로 기재하지 않은 파트는 내가 설정한 부분이 없는 파트이므로 참고하기 바란다.

### General

설명에는 기본적인 아이템에 대한 설명을 적어준다. 그리고 밑에 옵션 중에 나는 `오래된 빌드 삭제`를 체크해주고 빌드 이력 유지 기간을 30일로 주었다. 이 옵션들은 나도 아직 정확히 다 아는 것은 아니라서 오래된 빌드 삭제만 체크해주었다. 본인이 필요하다고 느끼는 것이 있다면 체크해주도록 하자.

### 소스 코드 관리

실질적인 SVN 관련 정보들을 설정하는 부분이다. Git을 쓴다면 Git을 클릭하면 되고 여기선 SVN을 사용할 것이기 때문에 Subversion을 클릭한다. 그러면 아래에 상세정보를 입력하는 란이 나온다.

![](/assets/post/20190508_3.PNG)

- Repository URL : 연결할 SVN 프로젝트 URL을 적어주도록 한다.
- Credentials : SVN에 접속할 자격증명정보를 적어주는 부분이다. 처음에 `-none-`으로 선택되어 있을텐데 옆에 Add를 클릭한다. 그러면 Jenkins가 나올텐데 또 클릭한다. 그러면 Add Credentials창이 나오게 된다. Username과 Password에 SVN 계정 정보를 입력하고 Add를 클릭하면 계정 정보가 저장된다. 다시 원래 창으로 돌아오면 셀렉트 박스를 클릭하여 방금 추가한 계정정보를 클릭한다. 제대로 추가한 정보라면 아무 메시지도 뜨지 않을 것이고 계정 정보가 유효하지 않다면 오류 메시지가 출력될 것이다.
- Local module directory (Option) : SVN 저장소 아래의 하위 경로를 표시해주는 부분이다. Repository URL에서 하위 경로까지 다 명시해주었다면 굳이 입력해줄 필요는 없다.
- 나머지 옵션들은 그냥 기본값으로 두면 된다.

### 빌드 유발

여기서 내가 사용한 옵션은 `Build periodically`이다. 일정한 주기를 두고 빌드를 실행하려면 해당 옵션을 주면 된다. 나는 현재 `30 12,18 * * *`으로 주었는데 분/시/일/월/요일/명령 순으로 뜻하는 명령어이다. 즉, `매일 12시 30분, 18시 30분에 빌드를 실행한다.`라는 뜻이 된다.

### Build

- Maven Version : 아까 위에서 만들어준 Maven 정보를 선택해주도록 한다.
- Goals : 보통 clean install을 써준다. 다른 옵션이 필요하다면 알아서 써주면 된다.

추가로 나는 빌드 완료 후 WAR 파일을 Tomcat webapps 경로 밑에 옮겨주고 Tomcat을 재시작 해주기 위해 Windows batch command를 추가해주었다. `Add build step`을 클릭하면 `Execute Windows batch command`를 볼 수 있다. 클릭하면 command를 입력할 수 있는 칸이 하나 나온다.

사실 빌드 후 조치에 Deploy war/ear to a container를 설정해주었다면 따로 이럴 필요가 없는 것 같은데 이상하게 빌드 후 Deploy war/ear to a container를 통해 Tomcat 경로에 자동으로 WAR 파일까지 배포하고 나서 웹 사이트에 접속하면 404 에러가 났다. 수동으로 Tomcat을 재시작해주면 정상 동작이 되지만 이것에 대한 해결법을 찾지 못하여(*이것에 대한 해결방법을 아시는 분이 있다면 도움의 손길을...*) 빌드 후 배포 방식을 바꾸기로 했다. 아마 WAR가 배포되고 프로젝트 소스가 압축 해제된 후 Tomcat이 자동으로 Reload되도록 되어 있는데 Reload에 문제가 있는건지 제대로 경로를 못 찾는 듯 하였다. 그래서 그냥 빌드 후 배포방식을 바꿔주었다.

빌드 후 Jenkins에서 배포까지 해주던 방식을 빌드까지만 하여 WAR 파일이 생성되면 그것을 Windows batch command를 통해 Tomcat의 webapps 경로에 복사해준 뒤 Tomcat/bin 폴더 안에 있는 startup.bat, shutdown.bat를 통해 재시작을 해주기로 했다. 아래 명령어가 이 과정에 해당한다.

```
@echo off
copy /Y "C:\Program Files (x86)\Jenkins\workspace\test_project\target\test_project.war" D:\apache-tomcat-8.0.49\webapps\test_project.war
D:
cd apache-tomcat-8.0.49\bin
cmd /c shutdown.bat
cmd /c waitfor SomethingThatIsNeverHappening /t 5 2>NUL
cmd /c startup.bat
```

2번째 줄부터 설명하자면 copy 명령어를 통해 Jenkins 설치 경로 아래의 workspace에 생성된 SVN 소스 안에서 WAR 파일을 찾는다. 그것을 Tomcat 경로 아래의 webapps에 복사해준다. 그런 다음 Tomcat 경로로 이동한 다음 bin 폴더 안에 기본적으로 있는 shutdown.bat 파일을 통해 Tomcat을 꺼준다. 혹시나 꺼지는 데 걸릴 시간이 있을 수 있어서 그것을 고려하여 5초 간 일시정지를 해준다. 그런 다음 startup.bat을 통해 Tomcat을 다시 시작해준다.

참고로 Jenkins의 workspace 안에 생성되는 폴더명은 앞에서 말했지만 이 프로젝트의 이름(아이템 이름)으로 생성된다. 그리고 target 폴더 안에 생성된 WAR 파일의 이름은 pom.xml에서 미리 설정 가능하다. 기본적으로는 프로젝트 이름_버전명.war로 저장될 것이다. 하지만 다른 이름으로 사용하고 싶다면 아래와 같이 설정해주면 된다.

```
<build>
	<finalName>test_project</finalName>
	<plugins>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
		</plugin>
	</plugins>
</build>
```

그리고 한가지 더 쓰자면 pom.xml 윗부분에 보면 groupId나 artifactId 등을 적는 부분이 있을텐데 거기에 `<packaging>war</packaging>`을 추가해줘야 WAR 파일로 패키징되니 이부분을 꼭 기재해주도록 한다.

```
<modelVersion>4.0.0</modelVersion>
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.1.4.RELEASE</version>
	<relativePath/> <!-- lookup parent from repository -->
</parent>
<groupId>me.eastglow</groupId>
<artifactId>test</artifactId>
<packaging>war</packaging>
<name>test_project</name>
<description>Test Project</description>
```

여기까지 설정을 완료했다면 저장을 해주고 `Build Now`를 클릭하여 빌드를 해본다. 실행되는 빌드를 클릭하고 `Console Output`을 클릭하면 현재 진행되고 있는 빌드 과정을 볼 수 있다. 제대로 된다면 아무 오류없이 완료되고 빌드 이름 옆의 불도 파란불이 들어올 것이다. 오류가 나더라도 오류 메시지가 남으니 그것을 보고 해결하면 된다.



