---
layout: post
title:  "[MySQL]Oracle to MySQL"
date:   2019-01-08 19:00:00
author: EastGlow
categories: Data-base
---
## Oracle to MySQL

기존에 개발 중이던 프로젝트는 Oracle DB 기반으로 만들어져있는데 새로 시작할 프로젝트의 DB는 MySQL인 경우가 있었다. Oracle과 MySQL은 구문이 서로 다른 부분이 많기 때문에 쿼리를 변경해야만 했다. 쿼리 변경을 하면서 기억에 남는 것들을 몇 가지 정리해본다.

크게 많이 바꾼건 없고 몇 가지 자잘한 부분만 바뀌었다. 혹시나 개발 진행하다가 더 생각나는 부분이 있으면 추가하도록 하겠다.

| Oracle | MySQL | 비고 |
|--|--|--|
| NVL | ifnull |  |
| rownum | limit | MySQL의 '@변수' 사용자변수를 이용하여 각 row마다 rownum을 만들어줄 수 있지만 속도에 영향을 미치고 필요없기 때문에 limit을 이용하도록 한다.  |
|  | limit | ifnull |
| TO_CHAR(SYSDATE, 'YYYYMMDDHH24MISS') | DATE_FORMAT(SYSDATE(), '%Y%m%d%H%i%s')| 현재 시간을 '201901011200' 형식으로 바꿔주는 부분이다. |
| '단어'\|\|'단어' | CONCAT('단어', '단어') | MySQL은 \|\|를 통해서 단어를 합치지 못하고 CONCAT을 이용해야 한다. |
| SUBSTR | SUBSTRING |  |
