---
layout: post
title:  "[Spring]RestTemplate를 이용하여 카카오 API 사용하기 (2)"
date:   2019-09-01 23:50:00
author: EastGlow
categories: Back-end
---

> 프로젝트 전체 소스 : https://github.com/eastglow/spring-boot-kakao-api-sample

# 3. AJAX를 통해 API 호출

## 1) 프론트 화면에서 AJAX를 통해 호출하는 과정

그냥 진짜 별 거 없다. 카카오 API 사용하기 (1)의 2번 항목에 있는 프론트 화면 스크린샷을 보면 로그인 화면이 있고 그 다음 메인 화면이 있다. 거기서 검색어를 입력하고 검색을 클릭하면 API를 호출하여 검색하고 결과값을 리턴 받아 3번째 사진처럼 검색 리스트를 쭉 뿌려준다.

이에 필요한 Javascript 소스는 [여기](https://github.com/eastglow/spring-boot-kakao-api-sample/blob/master/src/main/resources/static/js/index.js)에서 확인할 수 있다.

소스를 보면 알겠지만 딱히 특별한 건 없다. `place.search`를 통해 기본적인 검색 함수가 실행되며, 그 외에 리스트 아래에 표시되는 페이징 부분을 클릭하면 `place.list`함수가 실행된다. 자세히 들여다봐도 단순히 검색 관련 Form을 `serialize()`하여 AJAX를 통해 API 쪽으로 던져주는 것 밖에 없다. API를 호출하고 결과를 받고 나면 `place.makeList`를 통해 리스트 HTML 부분을 만들어서 새로이 넣어준다.

## 2) API를 요청받는 Java Controller 부분

`/place`라는 URI를 호출하면 검색 관련 API Controller가 호출된다. 전체 Controller 소스는 [여기](https://github.com/eastglow/spring-boot-kakao-api-sample/blob/master/src/main/java/me/eastglow/controller/SearchController.java)에서 확인할 수 있다.

```java
@GetMapping(value="/place")
public ResponseEntity searchPlaceByKeyword(HttpSession session, SearchVO paramVO) throws Exception {
    ResponseEntity re = kakaoRestApiHelper.getSearchPlaceByKeyword(paramVO);
    LoginVO loginVO = (LoginVO) session.getAttribute("loginVO");

    paramVO.setUserPkid(loginVO.getPkid());

    if("search".equals(paramVO.getSearchType())){
        // 내 검색 히스토리 등록
        searchService.createHistory(paramVO);

        // 인기 키워드 등록
        searchService.mergeKeyword(paramVO);
    }

    return re;
}
```

`/place`가 매핑되는 부분의 소스이다. 따로 API의 호출과 결과를 받는데 도움을 줄 클래스를 만들어 쓰고 있다. kakaoRestApiHelper의 `getSearchPlaceByKeyword`에 검색 관련 파라미터가 담긴 VO를 넘기면 그 안에서 API를 호출하고 결과값을 받아서 ResponseEntity 형태로 반환해준다.

추가로 `palce.search`로 `/place`를 호출했다면 프론트 화면 우측에 있는 `내 검색 히스토리` 및 `인기 키워드`와 관련된 테이블에 데이터가 Insert 되도록 되어 있다.


# 4. 실질적으로 API 호출을 처리하는 과정

```java
@Resource
private KakaoRestApiHelper kakaoRestApiHelper;
```

위와 같이 API를 실질적으로 처리하는 클래스는 `@Component`를 통해 Bean으로 등록하여 어느 곳에서든지 `@Resource`를 통해 의존성 주입만 해주면 사용할 수 있도록 해주었다. 참고로 `@Bean`과 `@Component`의 용도는 비슷하지만 조금 다른 부분이 있다. `@Bean`은 보통 외부 라이브러리와 같이 개발자가 직접적으로 컨트롤하기 힘든 것들을 Bean으로 등록하기 위해 사용하고, `@Component`는 마찬가지로 Bean으로 등록해주지만 개발자가 직접 개발한 클래스 등 컨트롤이 가능한 것들에 주로 사용한다고 한다.

```java
@Slf4j
@Component
public class KakaoRestApiHelper {
    @Value("${kakao.restapi.key}")
    private String restApiKey;

    private static final String API_SERVER_HOST  = "https://dapi.kakao.com";
    private static final String SEARCH_PLACE_KEYWORD_PATH = "/v2/local/search/keyword.json";

    public ResponseEntity<String> getSearchPlaceByKeyword(SearchVO searchVO) throws Exception {
        String queryString = "?query="+URLEncoder.encode(searchVO.getKeywordNm(), "UTF-8")+"&page="+searchVO.getCurrentPage()+"&size="+searchVO.getPageSize();
        RestTemplate restTemplate = new RestTemplate();
        HttpHeaders headers = new HttpHeaders();

        headers.add("Authorization", "KakaoAK " + restApiKey);
        headers.add("Accept", MediaType.APPLICATION_JSON_VALUE);
        headers.add("Content-Type", MediaType.APPLICATION_FORM_URLENCODED_VALUE + ";charset=UTF-8");

        URI url = URI.create(API_SERVER_HOST+SEARCH_PLACE_KEYWORD_PATH+queryString);
        RequestEntity<String> rq = new RequestEntity<>(headers, HttpMethod.GET, url);
        ResponseEntity<String> re = restTemplate.exchange(rq, String.class);

        return re;
    }
}
```

실제 API를 호출하는 부분을 위 소스에서 확인할 수 있다. 파라미터로 전달받은 검색 관련 VO에서 값들을 빼내어 URI 뒤에 붙을 쿼리 스트링을 만들어주고 RestTemplate를 생성해준다.

HttpHeaders는 실제 API Key를 헤더에 실어서 보내야하기 때문에 필요하여 만들어주었다. RequestEntity를 통해 URI와 헤더를 탑재(?)하여 GET 방식으로 요청하도록 해주었다. 결과값은 ResponseEntity의 exchange를 통해 String 형태로 받도록 하였다.

![](/assets/post/20190901_1.png)

[키워드로 장소 검색](https://developers.kakao.com/docs/restapi/local#%ED%82%A4%EC%9B%8C%EB%93%9C%EB%A1%9C-%EC%9E%A5%EC%86%8C-%EA%B2%80%EC%83%89)을 이용하여 결과를 받아보면 위 사진과 같은 형태로 받을 수 있다. 아래 소스는 다시 JavaScript 부분의 소스이다. 파라미터로 받은 `data`에 위 사진에서 보이는 Response 결과값이 담겨있다.

```
// 리스트를 만들어주는 함수
makeList : function(data) {
    $('#place_list').empty();

    var frm = document.searchForm;
    var list = data.documents;
    var totalCount = data.meta.pageable_count;

    if(list.length > 0){
        var contents = '';

        list.forEach(function(item, index, array){
            contents += '<div class="card">\n';
            contents += '   <div class="card-body" data-value="'+item.id+'">\n';
            contents += '       <h6 class="card-subtitle mb-2 text-muted">'+item.category_name+'</h6>\n';
            contents += '       <h5 class="card-title">'+item.place_name+'</h5>\n';
            contents += '       <h6 class="card-subtitle mb-2 text-muted phone">'+item.phone+'</h6>\n';
            if(trim(item.road_address_name) != ''){
                contents += '       <p class="card-text"><span class="badge badge-dark">도로명</span><span class="road-adres">'+item.road_address_name+'</span></p>\n';
            }
            if(trim(item.address_name) != ''){
                contents += '       <p class="card-text"><span class="badge badge-dark">지번</span><span class="adres">'+item.address_name+'</span></p>\n'
            }
            contents += '       <button type="button" id="modalBtn" class="btn btn-primary" data-toggle="modal" data-target="#mapModal" data-lng="'+item.x+'" data-lat="'+item.y+'">상세조회</button>\n';
            contents += '   </div>\n';
            contents += '</div>\n';
        })

        $('#place_list').html(contents);
        $("#place_list > .card").first().prop("tabindex", -1).focus();

        var totalPage = parseInt(totalCount / 10);

        if (totalCount > 10 * totalPage) {
            totalPage++;
        }

        $('#pagination').twbsPagination({
            initiateStartPageClick: false,
            totalPages: totalPage,
            visiblePages: 10,
            first: '<<',
            prev: '<',
            next: '>',
            last: '>>',

            onPageClick: function (event, page) {
                frm.currentPage.value = page;
                place.list();
            }
        });
    }else{
        var contents = '';

        contents += '<div class="card">\n';
        contents += '   <div class="card-body text-center">\n';
        contents += '       <h3 class="my-xl-5">검색된 결과가 없습니다.</h3>\n';
        contents += '   </div>\n';
        contents += '</div>\n';

        $('#place_list').html(contents);
    }
}
```

실제 검색 결과 리스트의 값이 담겨있는 부분은 `var list = data.documents`이며, 페이징을 위한 총 결과 개수는 `var totalCount = data.meta.pageble_count`에 담겨있다. 이것들을 이용하여 list를 HTML로 결과값을 만들어준다.


# 글을 마치며...

기존에 API를 활용한다고 하면 보통 JavaScript, JQuery를 이용하여 AJAX를 통해 바로 API를 호출하고 값을 리턴받는 형태로 많이 썼었는데 이렇게 Java를 통해 RestTemplate라는 것을 이용해보는 것은 처음이었다.

RestTemplate 자체를 안 써봐서 처음에 좀 헤맸었는데 여러 예제를 찾아보고 수정에 수정을 거듭하다보니 그래도 좀 깔끔한(?) 코드가 나온 듯 하다. 뭔가 또 하나 배운 거 같지만 동시에 아직도 갈 길이 멀다는 것을 느꼈다.