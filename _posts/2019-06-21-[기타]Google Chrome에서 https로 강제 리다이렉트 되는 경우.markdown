---
layout: post
title:  "[기타]Google Chrome에서 https로 강제 리다이렉트 되는 경우"
date:   2019-06-21 13:00:00
author: EastGlow
categories: 기타
---

## Google Chrome에서 https로 강제 리다이렉트 되는 경우

내 로컬 PC에서 이클립스로 프로젝트 하나를 구동한 다음 localhost로 접속하니 자꾸 https로 가게 되는 경우가 생겼다. 처음에는 프로젝트나 이클립스 혹은 톰캣 설정 문제인가 살펴봤는데 딱히 문제될 게 없었다. 또한, 조금 전까지만 해도 http로 잘 붙었기 때문에 문제가 될 거 같진 않았다. 결정적으로 익스플로러에서는 정상적으로 http로 접속되고 있었다.

그래서 뭘까 하다가 여러 키워드로 검색하다가 발견한 게 Chrome에서는 https로 접속했던 사이트에 대하여 다음번 접속 때도 https로 강제 리다이렉트 시키는 경우가 있다고 한다.

이런 경우엔 아래와 같이 조치를 취하면 원래대로 돌릴 수 있다.

1. Chrome 주소창에 `chrome://net-internals/#hsts` 를 입력한다.
2. 왼쪽 메뉴 중 맨 아래의 Domain Security Policy를 클릭한다.
3. 맨 아래에 보면 Delete domain security policies가 있고 Input Box가 하나 있는데 거기에 `localhost`를 입력하고 Delete를 눌러준다. (나는 localhost에서 이 문제가 발생해서 localhost를 지운거고 만약 다른 사이트에서 이 문제가 발생한다면 해당 사이트 주소를 입력해준다.)
4. 여기까지 했는데도 계속 https로 붙는다면 `chrome://settings/clearBrowserData` 를 주소창에 입력하여 문제가 발생한 시점부터의 인터넷 기록, 캐시, 쿠키 등을 삭제해준다.

