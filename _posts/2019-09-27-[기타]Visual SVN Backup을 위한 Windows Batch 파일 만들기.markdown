---
layout: post
title:  "[기타]Visual SVN Backup을 위한 Windows Batch 파일 만들기"
date:   2019-09-27 13:00:00
author: EastGlow
categories: 기타
---
요즘은 Git을 많이 사용하지만 예전에는 SVN, 더 내려가서는 CVS를 많이 사용했었다. 물론 아직까지도 사내에서 SVN을 쓰는 회사들이 많을 것이다. 그 회사들 중 하나에 다니고 있기 때문에 SVN을 사용하고 있으며 백업에 대한 필요성을 느끼게 되어 Batch 파일을 하나 만들었다. 평소에 Batch 파일을 만들어본 적이 있어서 문법적인 부분에 대한 설명이 틀리거나 미흡한 부분이 있을 수 있으니 양해 바란다.

구글링을 통해서 다른 사람들이 만들어놓은 소스들을 조금씩 섞어서 만들었으며 최종적으로 윈도우 스케쥴러에 등록해주면 시간마다 알아서 실행되도록 해두었다. (주 1회 정도면 적당할 거 같다.)

```
ECHO OFF
ECHO Backup SVN to D:/

SET DT=%DATE%
FOR /F "tokens=1-4 delims=-" %%A in ('echo %DT%') Do SET DT=%%A%%B%%C

IF NOT EXIST "D:\SVN_Backup\%DT:~0,4%\%DT:~4,2%\%DT:~6,2%" MD "D:\SVN_Backup\%DT:~0,4%\%DT:~4,2%\%DT:~6,2%"   

PUSHD C:\SVN\Repositories
FOR /d %%i in (*) do echo dump %%i & svnadmin dump %%i > D:\SVN_Backup\%DT:~0,4%\%DT:~4,2%\%DT:~6,2%\%%i.dump
POPD
```

앞에 ECHO 부분은 그냥 CMD창에 찍어주려고 써둔거고... 그 다음줄부터 설명해보겠다.

`SET DT=%DATE%` : 오늘 날짜를 DT라는 변수에 담는다. `yyyy-mm-dd`와 같은 형태로 담긴다.

`FOR /F "tokens=1-4 delims=-" %%A in ('echo %DT%') Do SET DT=%%A%%B%%C` : 오늘 날짜를 앞의 과정에서 `2019-09-27`과 같이 받아서 DT에 담아뒀다. 이것을 `-` 기준으로 잘라서 A, B, C 순서대로 넣어준다. 배열로 치면 {'2019', '09', '27'}이 들어가있는 것이다.

`IF NOT EXIST "D:\SVN_Backup\%DT:~0,4%\%DT:~4,2%\%DT:~6,2%" MD "D:\SVN_Backup\%DT:~0,4%\%DT:~4,2%\%DT:~6,2%"` : 위에서 만든 날짜 배열을 경로로 만들 것인데 `D:\SVN_Backup\`이라는 폴더 안에 `2019\09\27`경로가 존재하지 않는다면(IF NOT EXIST), 해당 디렉토리(`D:\SVN_Backup\2019\09\27`)를 만들어준다.(MD)

`PUSHD C:\SVN\Repositories` : Visual SVN을 처음 설치할 때 Repositories 위치를 어디로 할 것인지 물을텐데 그때 설정한 경로를 PUSHD에 넣어주면 된다. 그러면 Repositories 밑에 있는 각 저장소 디렉토리 목록을 불러오는 명령어이다.

`FOR /d %%i in (*) do echo dump %%i & svnadmin dump %%i > D:\SVN_Backup\%DT:~0,4%\%DT:~4,2%\%DT:~6,2%\%%i.dump` : 위에서 불러온 저장소 경로들에 대해서 FOR문을 돌릴 것이다. 돌리면서 svnadmin에 있는 dump 명령어로 dump 파일을 내보낼 것이며, 내보내는 경로는 아까 위에서 만든 `D:\SVN_Backup\2019\09\27`로 지정된다. 예를 들어 `eastglow`라는 SVN 저장소는 `D:\SVN_Backup\2019\09\27\eastglow.dump`로 파일이 저장되는 것이다.

`POPD` : POPD 명령을 사용하여 직전에 저장한 디렉터리 경로를 꺼내와 그 위치로 다시 이동하여 위의 작업을 반복한다.

위와 같이 Batch 파일을 만들었다면 윈도우 스케쥴러에 등록하면 끝이다. Batch 파일을 스케쥴러에 등록하는 방법은 너무나 많기 때문에 생략한다.

참고로 같은 물리적인 드라이브 말고도 네트워크 드라이브를 잡아뒀다면 해당 네트워크 드라이브로의 디렉토리 설정도 가능하니 원한다면 사용하면 될 듯 하다.
