---
layout: post
title:  "[JQuery]동적인 Rowspan 하기"
date:   2018-02-25 19:00:00
author: EastGlow
categories: Front-end
---

화면 쪽을 만지다보면 동적으로 테이블 병합을 해줘야할 때가 있다. 보통 jsp를 이용할 땐 JSTL이나 Java문을 이용하여 병합하면 되지만 순수 자바스크립트 코드만으로 병합하려면 앞의 방법보다는 조금 번거로워 진다.

병합해야하는 row에만 item을 몇 개 가지고 있는지 data-value 등으로 값을 준 다음 그것을 이용하여 병합하는 방법도 있고, 이번 포스트에서 사용한 text값을 이용한 방법 등이 있다.

나는 마침 JQuery의 contains 기능을 쓸 일이 있어서 이것을 이용하여 만들어보았다.

<script async src="//jsfiddle.net/eastglow/zowknbjb/1/embed/"></script>
