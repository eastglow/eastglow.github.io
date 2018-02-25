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

대략적으로 병합하는 과정을 설명하자면,

1. first 클래스를 가진 tr만 골라내어 해당 tr의 text값을 가져온다.
2. length가 1보다 큰 tr에만 rowspan을 줘야하기 때문에 죠건문을 통해 한번 필터링 해준다.
3. 1번에서 가져온 tr 리스트에서 제일 첫번째 tr에 rowspan을 추가해준다.(attr)
4. 마지막으로 rowspan을 추가한 첫번째 tr 외의 tr은 다 지워준다.(remove)

위와 같은 과정을 통해 최종적으로 같은 text값을 가진 tr끼리 병합할 수 있게 된다. 다만, contatins는 1을 찾는다면 "1", "11", "121" 등 1이 들어간 모든 값을 찾기 때문에 정말 유니크한 값이 들어갔을 때만 사용할 수 있다. 그 외의 경우엔 contatins가 아닌, 앞에서 잠깐 언급한 item의 개수 등을 통하여 확실하게 필터링할 수 있는 방법으로 진행하여야 한다.
