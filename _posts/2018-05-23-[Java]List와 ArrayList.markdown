---
layout: post
title:  "[Java]List? ArrayList?"
date:   2018-05-23 23:10:00
author: EastGlow
categories: Back-end
---

## 1. List<?> list = new ArrayList<?>(); vs ArrayList<?> list = new ArrayList<?>();

학부생 시절, 멋모르고 그저 많이 쓰니깐, 편리하니깐 ArrayList를 많이 애용하곤 했다. 물론 현업에서 일하는 지금도 많이 쓰는 녀석이다.

ArrayList를 잠깐 설명하고 지나가자면,

* Collection 혹은 List를 인터페이스로 하여 구현한 클래스.
* 처음 생성했을 때 배열과 달리 기본 크기가 10으로 정해져있다. 10 이상이 되면 자동으로 증가된다.
* 동적으로 크기가 변경되기 때문에 배열과 달리 원소가 추가되어도 따로 배열 내의 원소를 옮겨주는 행위를 하지 않아도 된다.
* add(), remove() 등 여러 편리한 메서드들이 있다.
* 하나의 ListIterator를 사용하여 forward/backward 양방향 순회가 가능하다.  
```
while (iterator.hasNext()) {
	System.out.println(iterator.next()); //forward
}         
while (iterator.hasPrevious()) {
	System.out.println(iterator.previous()); //backward
}
```

뭐... 쉽게 말해 배열보다 좀 더 쓰기 편해진 배열 업그레이드판이라고 보면 될 거 같다. 실제로도 다른 타입들보다 ArrayList가 데이터들을 조회하여 화면 쪽에 뿌릴 때 쓰이는 경우가 많다.


## 2. 두 방식의 차이점

그렇다면 ArrayList는 어떻게 선언하여 써야할까?

보통은 앞 쳅터의 제목처럼 2가지 타입으로 많이들 쓸 것이다. 나는 학부생 땐 후자의 방법으로 많이 써왔었는데 실제 일할 땐 전자로 쓰고 있다.

사실 내가 일하기 시작했을 땐, 기업의 소스 대부분이 전자와 같은 방식으로 쓰고 있었기 때문에 나도 그냥 그렇게 쓰기 시작했다. 그래도 둘의 차이점은 알고 쓰는게 좋지 않을까 싶어서 알아보니 다음과 같은 차이가 있었다.

```
전자(List<?>)의 경우, List라는 인터페이스를 ArrayList라는 클래스로 구현하였고,  
후자(ArrayList<?>)의 경우는 클래스로 바로 구현하였다.
```

처음 딱 이해했을 땐 그냥 인터페이스와 클래스의 차이 아니야? 라고 이해했는데 여러 글들을 찾아보다보니 꽤나 큰 차이점이 있었다.

쉽게 생각하자면 "나는 자동차를 만들거야. 자동차 중에서도 페라리를 만들겠어"와 "나는 페라리를 만들거야"라고 할 수 있을 것 같다.

이걸 좀 더 자바스럽게(?) 설명하자면 전자는 List라는 인터페이스를 통해 선언했기 때문에 ArrayList 말고도 같은 List 인터페이스를 받는 LinkedList로도 선언하여 쓸 수 있다.

하지만 후자는 이미 ArrayList라고 딱 정해버렸기 때문에 바꿀 수가 없다. 자세한 건 밑의 예제코드를 보도록 하자.

```
List<String> list = new ArrayList<String>();

list.add("Hello Word");
list.add("Hello Word 2");

for(String str : list){
  System.out.println(str);
}

list = new LinkedList<String>();

list.add("Hell Word");
list.add("Hell Word 2");

for(String str : list){
  System.out.println(str);
}
```

위 소스가 전자의 경우이다. List라는 인터페이스를 ArrayList라는 클래스로 구현하였다. 그리고 중간에 LinkedList로 다시 선언하였다.

```
Hello Word
Hello Word 2
Hell Word
Hell Word 2
```

컴파일을 해보면 위와 같이 각각의 List에 담은 값들이 정상적으로 출력된다. 하지만 처음부터 ArrayList로 선언하면 어떨까?

```
ArrayList<String> list = new ArrayList<String>();

list.add("Hello Word");
list.add("Hello Word 2");

for(String str : list){
  System.out.println(str);
}

list = new LinkedList<String>();

list.add("Hell Word");
list.add("Hell Word 2");

for(String str : list){
  System.out.println(str);
}
```

```
HelloWorld.java:27: error: incompatible types: LinkedList<String> cannot be converted to ArrayList<String>
    list = new LinkedList<String>();
           ^
1 error
```
보시다시피 LinkedList를 선언하는 부분에서 Convert 에러가 난다. ArrayList를 LinkedList로 변환할 수 없다는 것이다.

이러한 List 인터페이스와 ArrayList 클래스 사이의 관계는 자바의 다형성으로 설명할 수가 있다.

다형성(Polymorphism)이란 하나의 메소드나 클래스가 있을 때 이것들이 다양한 방법으로 동작하는 것을 의미한다.

아마 다형성이라는 단어를 한번이라도 본 사람은 여기까지 글을 읽었을 때 List와 ArrayList 두 차이점을 금방 이해하였을 것이다.

다형성에 관한 자세한 설명은 [여기에서](https://opentutorials.org/module/516/6127) 확인하길 바란다.

## 3. 결론

결론은 둘 중 어떤 방식을 써도 사실 별 관계는 없다고 생각한다. 전자의 방식으로 선언하고 나서 후에 LinkedList로 바꿔서 쓰는 경우는 한번도 본 적이 없고 내가 앞으로 개발하면서도 쓸 일이 없을 거 같다.

다만, 둘의 차이점엔 무엇이 있고 어떤 상황에서 쓰이는 지 정도는 짚고 넘어가면 좋을 듯 하다.
