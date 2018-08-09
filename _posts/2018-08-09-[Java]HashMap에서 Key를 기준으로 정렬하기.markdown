---
layout: post
title:  "[Java]HashMap에서 Key를 기준으로 정렬하기"
date:   2018-08-09 22:00:00
author: EastGlow
categories: Back-end
---
## 1. HashMap

HashMap은 Map Interface 중 하나를 구현한 것으로 Key, Value 한 쌍을 데이터로 가지고 있는 Java Collection의 한 종류이다. Key 값에 대응하는 Value 값을 하나씩 가지고 있으며, 중복된 Key의 저장을 허용하지 않는다. 만약에 중복된 Key를 저장한다면 나중에 들어온 Key의 Value 값으로 앞에 넣었던 Value 값이 덮어씌워진다.

```
Map<String, Integer> dataMap = new HashMap<>();
dataMap.put("data1", 1);
dataMap.put("data2", 2);
dataMap.put("data3", 3);
```

위와 같이 넣어주면 String을 Key로 하고 Integer를 Value로 가지는 HashMap이 만들어진다. HashMap의 또다른 특징은 넣는 순서와 관계없이 저장된다는 것이다. 순서를 보장받고 싶다면 LinkedHashMap을 이용하면 된다. 그리고 hashing을 사용하기 때문에 많은 양의 데이터를 검색하는데 뛰어난 성능을 보인다고 한다.

## 2. Key를 기준으로 정렬하기

DB에서 데이터를 리스트로 받아오는 경우가 있었다. 하나의 데이터 리스트에는 컬럼이 5개 정도 있었는데 각 컬럼마다 정수값이 들어있다. 예를 들면 아래와 같이 되어 있다고 보면 될 것 같다.

| pkid | column1 | column2 | column3 | column4 |
|:------:|:------:|:------:|:------:|:------:|
|   1   |   10   |   20   |   30   |   40   |

이렇게 뽑아온 리스트를 각 colum1 ~ 4의 값들을 큰 순으로 정렬하여 column 이름을 구해서 뿌려줘야했다. 쿼리로 짜보려했지만 도저히 생각이 안 났고 쿼리보단 서버단에서 처리하는게 더 나을 것 같아서 자바로 짜보았다. 물론 구글신의 힘을 빌어...ㅎㅎ

```
public class HashMapTest {
	public static void main(String[] args) {
		List<Entity> dataList = new ArrayList<>();
		
		Entity entity = new Entity();
		entity.setColumn1(10);
		entity.setColumn2(20);
		entity.setColumn3(30);
		entity.setColumn4(40);
		dataList.add(entity);

		for(Entity e : dataList){
			Map<String, Integer> resultMap = new LinkedHashMap<>();
			Map<String, Integer> dataMap = new HashMap<>();

			dataMap.put("column1", e.getColumn1());
		    dataMap.put("column2", e.getColumn2());
		    dataMap.put("column3", e.getColumn3());
		    dataMap.put("column4", e.getColumn4());
		    
		    List<String> resultList = sortByValue(dataMap);
		    Iterator<String> it = resultList.iterator();
		    		    
		    while(it.hasNext()) {
		    	String key = it.next();
		    	resultMap.put(key, dataMap.get(key));
		    }
		    
		    e.setResultMap(resultMap);
		}
	}
	
	public static List<String> sortByValue(Map<String, Integer> map) {
	    List<String> list = new ArrayList<String>();
	    list.addAll(map.keySet());

	    Collections.sort(list, new Comparator<String>() {
	        public int compare(String o1, String o2) {
	            int v1 = map.get(o1);
	            int v2 = map.get(o2);
	            return ((Comparable<Integer>) v2).compareTo(v1);
	        }
	    });

	    return list;
	}	
}

class Entity{
	private int column1;
	private int column2;
	private int column3;
	private int column4;
	private Map<String, Integer> resultMap;
	
	public int getColumn1() {
		return column1;
	}
	public void setColumn1(int column1) {
		this.column1 = column1;
	}
	public int getColumn2() {
		return column2;
	}
	public void setColumn2(int column2) {
		this.column2 = column2;
	}
	public int getColumn3() {
		return column3;
	}
	public void setColumn3(int column3) {
		this.column3 = column3;
	}
	public int getColumn4() {
		return column4;
	}
	public void setColumn4(int column4) {
		this.column4 = column4;
	}
	public Map<String, Integer> getResultMap() {
		return resultMap;
	}
	public void setResultMap(Map<String, Integer> resultMap) {
		this.resultMap = resultMap;
	}	
}
```

위 코드가 전체적인 코드이다. 물론 예제로 만들기 위해 어느정도 간소화하였다.

```
List<Entity> dataList = new ArrayList<>();
		
Entity entity = new Entity();
entity.setColumn1(10);
entity.setColumn2(20);
entity.setColumn3(30);
entity.setColumn4(40);
dataList.add(entity);
```

먼저 List에 Entity 객체를 세팅해준다. 위의 표에서 봤던 값들을 각 컬럼번호에 담아준다. (실제 프로그램에선 DAO를 통해 DB 값을 받아올 것이다.)

```
for(Entity e : dataList){
    Map<String, Integer> resultMap = new LinkedHashMap<>();
    Map<String, Integer> dataMap = new HashMap<>();

    dataMap.put("column1", e.getColumn1());
    dataMap.put("column2", e.getColumn2());
    dataMap.put("column3", e.getColumn3());
    dataMap.put("column4", e.getColumn4());

    List<String> resultList = sortByValue(dataMap);
    Iterator<String> it = resultList.iterator();

    while(it.hasNext()) {
        String key = it.next();
        resultMap.put(key, dataMap.get(key));
        System.out.println(key);
    }

    e.setResultMap(resultMap);
}
```

(데이터가 1개 밖에 없지만) 리스트에 들어있는 Entity 객체를 뽑아서 루프를 돌려준다. 나는 각 컬럼의 이름이 필요했기 때문에 HashMap에 담을 때 컬럼 숫자값과 함께 컬럼명을 Key 값으로 담아주었다. 컬럼 숫자값은 단순 수치이기 때문에 겹칠 수도 있어 Key로 이용할 수 없지만 컬럼명은 유니크하기 때문에 Key 값으로 사용이 가능하였다.

```
public static List<String> sortByValue(Map<String, Integer> map) {
List<String> list = new ArrayList<String>();
list.addAll(map.keySet());

Collections.sort(list, new Comparator<String>() {
    public int compare(String o1, String o2) {
        int v1 = map.get(o1);
        int v2 = map.get(o2);
        return ((Comparable<Integer>) v2).compareTo(v1);
    }
});

return list;
}	
```

이 포스팅의 핵심인 sort 관련 코드이다. 앞에서 데이터를 담아주었던 HashMap을 파라미터로 받아와서 Key값만 뽑아와서(map.keySet()) list로 변환해준다. 그리고 Collections 안에 있는 sort를 이용하여 compare 메서드를 구현해준다. compare를 통해 list에 있는 값들을 2개씩 뽑아와서(Key값) 그 Key값을 통해 int 형태의 컬럼 숫자값을 서로 비교해준다. 그러면 정렬이 완료된다.

```
while(it.hasNext()) {
    String key = it.next();
    resultMap.put(key, dataMap.get(key));
    System.out.println(key);
}
```
```
================
column4
column3
column2
column1
================
```

최종적으로 내림차순으로 정렬되어 가장 큰 값이 들어가있는 "column4"가 맨 처음 나오게 된다. 이것을 나는 컬럼명과 컬럼숫자값을 같이 화면단으로 넘겨줘야했기 때문에 다시 LinkedHashMap에 담아서 마무리해주었다. (순서를 유지하기 위해서)

