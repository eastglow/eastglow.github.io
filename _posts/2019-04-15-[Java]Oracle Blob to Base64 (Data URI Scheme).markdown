---
layout: post
title:  "[Java]Oracle Blob to Base64 (Data URI Scheme)"
date:   2019-04-15 19:00:00
author: EastGlow
categories: Back-end
---
## 서론

보통 이미지 파일 처리는 웹루트 밖의 첨부파일 폴더에 업로드를 하고 DB Table에서는 해당 파일의 경로와 이름만 가지고 있도록 쓰고 있다. DB에 직접 Blob 타입으로 저장할 수도 있지만 보편적으론 전자의 방법을 많이 이용하는 것 같다. (용량 문제도 있고 Blob 타입은 변환도 해야하고 여러모로 귀찮은 것 같다.)

그런데 이번에 고객사의 DB View를 조회해보니 이미지 파일을 Blob 타입으로 넘겨주고 있었다. 매번 이미지 파일을 직접 업로드하고 경로를 따서 사용하다가 Blob 타입으로 받아보니 어떻게 해야하나 고민이 되었다. 관련 자료를 찾아보니 대략적인 프로세스는 아래와 같았다.

`Blob Data Select => Java에서 Bytes[]로 변환 후 다시 Base64로 Encode => HTML에서 Data URI Scheme을 통해 보여주기`

>참고 1 : [https://en.wikipedia.org/wiki/Data_URI_scheme](https://en.wikipedia.org/wiki/Data_URI_scheme)
>참고 2 : [https://hyeonseok.com/soojung/webstandards/2011/02/17/641.html](https://hyeonseok.com/soojung/webstandards/2011/02/17/641.html)


## 본론

### Java encodeBlobToBase64(Blob data)
```
/**
 * Oracle BLOB 타입을 Base64로 변환
 * @param BLOB
 * @return
 */
public static String encodeBlobToBase64(Blob data){      
    String base64Str = "";
    byte[] blobToBytes = null;
    
    try {   
        if(data != null && data.length() > 0){
            blobToBytes = data.getBytes(1l, (int) data.length());
            base64Str = Base64.encodeBase64String(blobToBytes);
        }
    } catch (Exception e) {
        LOGGER.debug("encodeBlobToBase64 error", e);
    }
    
    return base64Str;
} 
```

### JSP
```
<c:forEach var="v" items="${dataList}" varStatus="status">
    <div style="background-image:url('data:image/png;base64,${dataUtil:encodeBlobToBase64(v.blobData)}')"></div>
</c:forEach>
```

현재 프로젝트는 JSP가 View로 쓰이고 있기 때문에 위와 같이 c:forEach로 리스트가 처리되고 있다. JSTL을 이용하고 있기 때문에 JSP 스크립틀릿을 이용한 Blob 처리는 하지 않고 있다. (최대한 JSP에 Java 코드가 들어가는걸 지양하고 있다.)

그래서 Custom tld 파일을 만들어서 쓰고있고(dataUtil이라 칭함.) 그 안에 아래와 같이 실제 `encodeBlobToBase64`메서드가 있는 경로를 선언하여 사용 중이다.

### Custom tld 파일
```
<function>
    <description>Oracle BLOB 타입을 Base64로 변환</description>
    <name>encodeBlobToBase64</name>
    <function-class>egovframework.test.util.DataUtil</function-class>
    <function-signature>
        java.lang.Byte encodeBlobToBase64(java.sql.Blob)
    </function-signature>
</function>
```

위와 같이 처리하면 Blob 데이터가 Base64로 제대로 Encode 되는 것을 확인할 수 있었고 data image 또한 잘 나온다.

하나 문제가 있다면 표현해야할 리스트의 양이 많아지면 이미지를 변환하여 로딩하는 시간이 꽤 걸린다는 것이다. 페이징 처리를 하거나 무언가 다른 방법을 써야할 거 같은데 이부분은 아직 잘 모르겠다. 다행히도 현재 개발된 페이지는 검색 조건이 기본적으로 달려있어서 한 페이지 내에 보여지는 리스트의 개수가 그리 많지 않아서 로딩 속도에는 문제가 없다.
