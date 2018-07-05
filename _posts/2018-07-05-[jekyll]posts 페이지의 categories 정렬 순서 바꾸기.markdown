---
layout: post
title:  "[jekyll]posts 페이지의 categories 정렬 순서 바꾸기"
date:   2018-07-05 23:30:00
author: EastGlow
categories: 기타
---
## 문제

GitHub에서 정적 페이지 생성을 도와 블로그를 운영할 수 있게 해준 Jekyll. 쓰다보니 Posts 페이지의 카테고리 정렬 순서를 바꾸고 싶어졌다. 루비 언어는 아예 모르고 구글링을 해가며 찾아서 겨우 바꿨다.

## 해결

###posts.html의 원래 코드
~~~
{% for category in site.categories %}
    {% capture cat %}{{ category | first }}{% endcapture %}
    	<h2 id="{{cat}}">{{ cat | capitalize }}</h2>
    {% for desc in site.descriptions %}
    {% if desc.cat == cat %}
    	<p class="desc"><em>{{ desc.desc }}</em></p>
    {% endif %}
{% endfor %}
<ul class="posts-list">
    {% for post in site.categories[cat] %}
        <li>
        <strong>
            <a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
        </strong>
        <span class="post-date">- {{ post.date | date_to_long_string }}</span>
        </li>
    {% endfor %}
</ul>
{% if forloop.last == false %}<hr>{% endif %}
{% endfor %}
~~~

대충 자바식(?)으로 풀어보자면 site.category라는 배열 같은 것을 받아서 for문을 통해 category 값을 하나씩 빼내는 것 같다. capture의 용도는 정확히는 모르지만 아마 변수 선언과 같은 역할을 하는 듯 하다. 두번째 줄의 first는 무슨 역할을 하는지 잘 모르겠다.

여튼 꺼낸 category를 쭉 찍어주고 description도 찍어주고 category 안에 있는 post도 쭉 찍어준다. post는 date_to_long_string을 보아 날짜순으로 정렬하는 듯 하다.

###posts.html의 바꾼 코드
~~~
{% assign sorted_cats = site.categories | sort %}
	{% for category in sorted_cats %}
    	<h2 id="{{cat}}">{{ cat | capitalize }}</h2>
    {% for desc in site.descriptions %}
    {% if desc.cat == cat %}
    	<p class="desc"><em>{{ desc.desc }}</em></p>
    {% endif %}
{% endfor %}
<ul class="posts-list">
    {% for post in site.categories[cat] %}
        <li>
        <strong>
            <a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
        </strong>
        <span class="post-date">- {{ post.date | date_to_long_string }}</span>
        </li>
    {% endfor %}
</ul>
{% if forloop.last == false %}<hr>{% endif %}
{% endfor %}
~~~
맨 위부터 두줄만 바꾸었다. assing도 무언가 변수 선언의 느낌이 드는 코드 같다. 이전과 달리 site.categories를 꺼내서 뒤에 sort를 통해 알파벳 순으로 정렬을 해주고 해준 category 배열(?)을 이전과 같이 for문을 통해 찍어주면 완성이다.
