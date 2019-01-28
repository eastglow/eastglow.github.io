---
layout: post
title:  "[MySQL]A테이블 검색 시 B테이블의 특정 컬럼 값 이용하기"
date:   2019-01-28 23:00:00
author: EastGlow
categories: Data-base
---

## 내용

전자정부+MySQL 기반의 웹 프로젝트를 개발하던 중에 하나의 기능이 필요하게 되었다. A라는 테이블의 리스트를 뽑아오는 화면인데 또다른 B라는 테이블의 특정 컬럼에 있는 특정 값을 검색 조건으로 달아줘야 했다.

대략적으로 테이블 구조를 그려보자면,
- Table A

| USER_ID | USER_NM |
|--|--|
| 1 | 이름_1 |
| 2 | 이름_2 |
| 3 | 이름_3 |

- Table B

| PKID | ADMIN_ID |
|--|--|
| 1 | 1,3 |
| 2 | 1 |
| 3 | 1,2,3 |

이런식이다. B테이블에서 PKID로 ADMIN_ID를 뽑아온 후, A테이블에서는 뽑아온 ADMIN_ID가 없는 USER_NM을 뽑아오고자 했다.

보통 A테이블의 USER_ID와 B테이블의 ADMIN_ID가 숫자값이라면,
```
SELECT USER_NM
FROM A
WHERE USER_ID NOT IN (SELECT ADMIN_ID
						FROM B
						WHERE PKID = 1)
```
위의 쿼리로 쉽게 해결될 수 있다. 위 쿼리를 실행하면 실질적으로 `WHERE USER_ID NOT IN = (1,3)`와 같이 실행이 된다. 

하지만 아래와 같이 USER_ID, ADMIN_ID 값이 VARCHAR 형태라면 말이 달라진다.

- Table A

| USER_ID | USER_NM |
|--|--|
| ID_1 | 이름_1 |
| ID_2 | 이름_2 |
| ID_3 | 이름_3 |

- Table B

| PKID | ADMIN_ID |
|--|--|
| 1 | ID_1,ID_3 |
| 2 | ID_3 |
| 3 | ID_1,ID_2,ID_3 |

실제 개발 중인 프로젝트의 테이블의 위와 같이 콤마(,)로 ID를 구분하여 데이터를 집어넣고 있었다. 위쪽에서 봤던 `WHERE USER_ID NOT IN =` 쿼리를 그대로 가져다 쓴다면 제대로 검색이 되지 않을 것이다.
왜냐하면 `WHERE USER_ID NOT IN = (ID_1,ID_3)` 이렇게 들어가기 때문이다. 제대로 검색이 되게 하려면 `WHERE USER_ID NOT IN =('ID_1','ID_3')`처럼 아포스트로피(')로 감싸줘야 한다.

물론, Mybatis를 이용한다면 Mybatis의 foreach문을 통해 처리할 수 있겠지만 번거롭기도 하고 쿼리로 처리하고 싶어서 찾아보던 중 MySQL의 `FIND_IN_SET`이라는 함수를 찾았다.

> FIND_IN_SET(text, string) : 문자열(string) 목록 내의 문자(text) 위치를 반환합니다.

첫번째 인자로 위치를 찾을 문자를 입력하고 두번째 인자로 위치 검색을 하려는 문자열을 입력해준다.

```
SELECT T.*
FROM (  	
    SELECT USER_ID
        , USER_NM
        , (SELECT ADMIN_ID FROM B WHERE PKID = 1) ADMIN_ID    
    FROM A
    ORDER BY USER_NAME
) T
WHERE FIND_IN_SET(T.USER_ID, T.ADMIN_ID) = 0
```

`FIND_IN_SET`을 이용한 쿼리이다. ADMIN_ID 컬럼을 추가하였고 이 ADMIN_ID 컬럼 안에 USER_ID가 있는지 `FIND_IN_SET`으로 찾아준다. 만약 없다면 0이 나올 것이고 있다면 1 이상의 값이 나오게 된다.

`WHERE FIND_IN_SET(T.USER_ID, T.ADMIN_ID) = 0`의 의미는 결국에는 ADMIN_ID에 USER_ID가 없는 값들만 뽑아오게 하려는 것이다. 정리해보자면 B테이블에서 PKID=1에 해당하는 ADMIN_ID를 뽑아와서 A테이블에 비교를 하는데, A테이블의 값들 중 뽑아온 ADMIN_ID에 포함되지 않는 데이터만 검색하는 쿼리이다.