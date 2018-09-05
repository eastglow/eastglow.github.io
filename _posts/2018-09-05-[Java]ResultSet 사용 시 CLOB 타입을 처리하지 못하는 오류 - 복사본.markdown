---
layout: post
title:  "[Java]ResultSet 사용 시 CLOB 타입을 처리하지 못하는 오류"
date:   2018-09-05 23:10:00
author: EastGlow
categories: Back-end
---
## ResultSet 사용 시 CLOB 타입을 처리하지 못하는 오류

Spring에선 사실상 볼 일이 거의 없는 ResultSet. 하지만 Spring 이전 Framework에선 아주 많이 쓰인다. 나도 현업에서 매일 같이 쓰고 있기 때문에 빼놓을 수 없는 녀석인데 오늘 ResultSet을 이용하여 DB에서 빼온 데이터를 VO 객체에 담으려고 하였더니 오류가 났었다.

```
java.sql.SQLException: Conversion to String failed
```

뭐 직역하자면... "문자열로 변환하지 못했습니다." 다. 내가 가져오려던 컬럼은 CLOB 타입이었고 Java 단에서는 String 형태의 변수에 담아주려고 했다. 개발할 때 빼놓을 수 없는 구글링을 해보니 해결법은 못 찾겠고 계속 삽질하다가 문득 생각나는 게 하나 있었다. 예전에도 CLOB 타입 데이터를 처리하기 위해 이것저것 하다가 본 글이었는데 ojdbc 버전이 낮으면 CLOB 타입 데이터를 String 변수에 바로 담을 수 없다고 했던 글이 기억났다.

그래서 혹시나하고 ojdbc 버전을 보니 ojdbc4 버전이었다. 상당히 낮은 버전이었고 우선은 프로젝트 JDK 버전과 맞추어 ojdbc6 버전을 라이브러리 폴더에 넣어주었다.

그랬더니 오류는 사라지고 String 변수에 CLOB 타입 데이터가 정상적으로 담겼다. 결과적으로 보면 별 거 아니었지만 라이브러리 버전도 개발할 때 신중히 고려하여 쓰는 것이 중요하다는 생각이 든다.