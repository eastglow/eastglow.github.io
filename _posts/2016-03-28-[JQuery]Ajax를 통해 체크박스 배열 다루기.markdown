---
layout: post
title:  "[JQuery]Ajax를 통해 체크박스 배열 다루기"
date:   2016-03-28 19:00:00
author: EastGlow
categories: Front-end
---

![](/assets/post/image1.png)
<체크박스 선택 화면>


### 1. Javascript 소스
```
function deleteNmeCard(){ //명함 체크박스 삭제
    <!-- 명함 고유값과 고유값을 담을 Array를 선언 -->    
    var nmeCardSeq ="";
    var checkArray = new Array();  
    
    <!-- forEach문을 통해 체크된 고유값을 Array에 push로 넣음 -->
    <c:forEach var="selectNmeCardList" items="${selectNmeCardList}" varStatus="status">
        nmeCardSeq= "#"+"${selectNmeCardList.nmeCardSeq}"; // #을 붙여주는 이유는 id값을 참고하기 위함임.
        if($(nmeCardSeq).is(":checked")){
            checkArray.push("${selectNmeCardList.nmeCardSeq}");            
        }
    </c:forEach>
 
    <!-- 만약 체크된 값이 없다면 삭제할 명함을 선택하라는 알림창을 띄움 -->
    if(checkArray.length == 0){
        alert("삭제할 명함을 선택하세요.")
    }
 
    <!-- Array 길이가 0이 아니라면 정말 삭제할 것인지 재차 확인함 -->
    else{
        if (confirm("삭제하시겠습니까?") == true){    //확인
            <!-- Ajax 부분. POST 방식으로 고유값을 담은 Array를 컨트롤러로 보냄 -->
            $.ajax({
                type : 'POST',
                url : 'deleteNmeCard.do',
                data : {  
                    0:0,
                    checkArray : checkArray},
                    <!-- 성공적으로 컨트롤러가 돌았다면 메인 화면으로 돌아감 -->
                    success: function pageReload(){
                        location.href="/nmeCardSelectForm.do"
                    },
                    error:function(request,status,error){
                        alert("code:"+request.status+"\n"+"message:"+request.responseText+"\n"+"error:"+error);
                    }
                });
                <!-- Array와 고유값을 초기화 시켜줌 -->
                checkArray= new Array();
                nmeCardSeq="";
            }
            <!-- 만약 취소하면 페이지를 reload 함 -->
            else{   //취소    
                location.reload(true);
            }
        }
    }        
            
$(document).ready(function(){ //체크박스 전체선택,전체해제
    <!--#checkAll은 위 그림의 "전체선택" 버튼에 해당 -->
    $("#checkAll").click(function() {
        $("input[name=chkbox]:checkbox").prop("checked",true); // name이 chkbox인 input 타입들의 checked값을 "true"로 바꿈
    });
    <!-- #unCheckAll은 위 그림의 "전체해제" 버튼에 해당 -->
    $("#unCheckAll").click(function() {
        $("input[name=chkbox]:checkbox").prop("checked",false); // name이 chkbox인 input 타입들의 checked값을 "false"로 바꿈
    });
});
```

### 2. Java 소스

#### Controller
```
//명함 삭제 완료
@RequestMapping(value = "/deleteNmeCard.do", method={RequestMethod.GET, RequestMethod.POST})
public String deleteNmeCard(@RequestParam(value="checkArray[]") List<Integer> deleteList, @ModelAttribute("nmeCardValueObject") NmeCardValueObject nmeCardValueObject, ModelMap model) throws Exception {
    <!-- Ajax를 통해 Array로 받은 "deleteList"를 하나씩 빼내어 ArrayList에 넣음 -->
    ArrayList<Integer> deleteArray = new ArrayList<Integer>();
    for(int i=0;i<deleteList.size();i++){
        deleteArray.add(deleteList.get(i));
    }
    <!-- ArrayList로 담은 것을 Service->ServiceImple->DataAccessObject->Mapper로 보내 처리 -->
    nmeCardService.deleteNmeCard(deleteArray);
    return "redirect:/nmeCardSelectForm.do";
}
```

#### DAO
```
//명함 삭제 완료
public void deleteNmeCard(ArrayList<Integer> deleteArray) throws Exception {
    <!-- Controller에서 전달한 ArrayList를 for문을 돌려 고유값을 하나씩 꺼내어 Delete 문을 돌림 -->
    for(int i=0; i<deleteArray.size();i++){
        int nmeCardSeq = deleteArray.get(i);
        getSqlSession().delete("nmeCardMapper.deleteNmeCard", nmeCardSeq);
    }
}
```

#### Mapper
```
<!-- 명함 삭제 -->
<delete id="deleteNmeCard" parameterType="int">
    DELETE FROM
    nmecard_tb
    WHERE
    nmecard_seq in (#{nmeCardSeq});
</delete>
```


대략적으로 설명을 하자면, 체크박스마다 해당 명함의 고유값을 담고 있다. 체크되면 해당 체크박스의 고유값을 자바 스크립트 부분에서 Array에 담고 그것을 Ajax를 통해 Controller로 보내서 처리한다.

두번째로 전체선택, 전체해제 부분은 메뉴버튼 하위메뉴에서 전체선택 버튼을 누르면 모든 체크박스의 checked 값이 "true"값이 되며 전체해제 버튼을 누르면 "false"값으로 바뀌게 해두었다.

위에서 체크한 박스의 값들을 Array에 담아 넘기면 Controller에서 RequestParam으로 받아 하나씩 꺼내 ArrayList에 담는다.

그 ArrayList를 DAO로 넘겨서 하나씩 Delete 문을 실행한다.

당시에 개발 기간 데드라인도 다가오는 상태였고 최대한 기능적인 부분부터 마무리 지어야 했기 때문에 위와 같이 조금은 비효율적이지만 확실하게 작동되는 방식으로 작성하였다.
