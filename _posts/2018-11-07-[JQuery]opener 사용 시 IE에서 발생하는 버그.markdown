---
layout: post
title:  "[JQuery]opener 사용 시 IE에서 발생하는 버그"
date:   2018-11-07 20:00:00
author: EastGlow
categories: Front-end
---

## $(opener.document)

회사 프로젝트 유지보수 중에 부모창>아이프레임 + 팝업창 구조로 되어 있는 화면이 있었다. 아이프레임에서 팝업창을 하나 열고 그 팝업창에서 수행하는 로직이 하나 있었는데,

```
<script>
$(document).ready(function () {
    var targetValue = $(opener.document).find('#target').css('display');
    alert("targetValue=" + targetValue);
});	
</script>
```

대충 이런 느낌의 함수가 팝업창이 열리면서 document.ready를 통해 실행된다. (실제 함수와는 전혀 다른 소스이지만 부모창에서 특정 값을 가져오는 것은 동일함.)

크롬에서는 아-무 문제 없이 `css('display')` 값을 가져오는데 IE에서는 아무리 해도 `undefined'가 발생하는 것이다.

이거 하나 때문에 한 3-4시간은 구글링하고 이것도 해보고 저것도 해보다가 도저히 안 되길래 스택오버플로우에 질문글을 올렸다. 그런데 답변을 해주던 사람의 소스도 계속 안 먹히는 것이다.

## 그래서 무엇을 했는가?

```
//1.
var targetValue = $(opener.document).find('#target').css('display');
//2.
var targetValue = $('#target', opener.document).css('display');
//3.
var targetValue = opener.document.getElementById('target').style.display;
```

1번은 내가 했던 방식이고 2번은 스택오버플로우에서 답변받은 방법이다. 결론은 두가지 다 안 되더라. 그래서 결국 순수 자바스크립트로 해결보려고 3번을 써보았는데 다행히 저건 먹혔다.

그래서 "겨우 해결했다"하고 유지보수 작업을 마무리 지었는데 몇 시간 뒤에 스택오버플로우 답변자가 새로운 답변을 올려주었더라.

> You should pass your target document as the context of jQuery's function. You can do so using the second parameter of $(selector, context).
> 
> So for you it would be
> 
> `var target = $('.target', opener.document).css('display');`
> 
> But IE has a weird bug where `opener.getComputedStyle()`'s values will all be set to null.
> 
> So the only way I found to workaround that is to load jQuery in the opener itself, and from your popup to call its own jQuery function
> `var targetValue = opener.$('#target').css('display');`

원래대로라면 내가 쓴 방법이나 이 형님(?)이 쓴 방법이나 먹히는 게 정상인데 IE에 이상한 버그가 있다고 한다.-_- 그래서 마지막 줄에 써둔 방법으로 해보라고 전해주길래 써보니깐 먹히더라.

IE의 버그인지 JQuery 버그인지 소스상의 버그인지(라고 하기엔 별 거 없는 코드였다.) 정확하게 밝혀진 건 없지만 어쨌든 JQuery에서 opener Selector를 이용할 때 저러한 버그가 있으니 참고하면 좋을 것 같다.