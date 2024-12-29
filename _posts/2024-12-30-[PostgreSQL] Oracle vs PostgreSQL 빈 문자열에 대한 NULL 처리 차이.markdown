---
layout: post
title:  "[PostgreSQL] Oracle vs PostgreSQL 빈 문자열에 대한 NULL 처리 차이"
date:   2024-12-30
author: EastGlow
categories: Database
---

# 1. 들어가며

Oracle을 메인으로 쓰던 애플리케이션에 있던 SELECT 쿼리를 다른 PostgreSQL 메인의 애플리케이션으로 옮겨오던 작업을 하고 있었다.

당연하게도 Oracle 쪽과 데이터가 동일한 조건에서 동일하게 나오는지 검증해보는 작업이 있었고 그 과정에서 특정 조건의 경우 두 DB 기반의 애플리케이션이 서로 다른 결과값을 리턴하게 되어서 살펴보게 되었다.

원인을 알고보니 정말 간단하고 기본적인 내용이었는데 평소에는 Oracle을 메인으로 다루고 있다보니 PostgreSQL 기반의 애플리케이션에서 쿼리를 짤 때는 크게 생각 안 하고 있던 부분이라 리마인드 겸 글을 남겨본다.

# 2. 원인은?

Oracle과 PostgreSQL 두 DB 간의 빈 문자열 처리 방식 차이에서 오는 문제였다.

```
-- Oracle
AND     (
                :value IS NULL
            OR  (:value = 'Y' AND A.COL1 LIKE 'TEST%')
        )
 
-- PostgreSQL
and     (
                :value is null
            or  (:value = 'Y' and a.col1 like 'TEST%')
        )
```

정말 간단한 조건식인데 value에 따라서 `is null` 조건을 타고 끝나거나, 아니면 그 다음 `or` 조건을 타게 되는 식이다.

여기서 문제는 무엇이었을까? 만약 value에 빈 문자열이 들어오게 되면 Oracle과 PostgreSQL은 서로 다른 조건을 타게 된다는 것이다.

## Oracle의 빈 문자열 처리

![](/assets/post/20241230_1.png)
- Oracle에서는 빈 문자열('')을 NULL과 동일하게 취급
- 따라서 '' IS NULL은 Oracle에서 TRUE로 평가됨

## PostgreSQL의 빈 문자열 처리

![](/assets/post/20241230_2.png)
- PostgreSQL에서는 빈 문자열('')을 단순한 비어있는 문자열로 간주하며, NULL과 동일하게 취급하지 않음
- 따라서 '' IS NULL은 PostgreSQL에서 FALSE로 평가됨

# 3. 그럼 어떻게 써야할까?

```
and     (
                nullif(:value, '') is null
            or  (:value = 'Y' and a.col1 like 'TEST%')
        )
```

위와 같이 nullif를 통해 처리하면 된다. nullif는 :value와 빈 문자열('')이 같다면 NULL을 반환하고 아니라면 :value를 그대로 반환한다.

- :value가 NULL이라면? NULL이 그대로 반환되어 `is null` 조건을 충족한다.
- :value가 빈 문자열('')이라면? :value와 빈 문자열('')이 서로 같기 때문에 NULL이 반환되어 `is null` 조건을 충족한다.
- :value가 다른 문자열이라면? 빈 문자열이 아니기 때문에 해당 문자열이 그대로 반환되고 or 뒤의 조건을 체크하게 된다. 여기서 Y라는 문자가 들어왔다면 `a.col1 like` 조건도 충족하게 된다.
