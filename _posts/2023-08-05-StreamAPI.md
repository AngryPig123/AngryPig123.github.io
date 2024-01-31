---
title: 스트림 시작하기
author: Angrypig
date: 2023-08-05 15:40:00 +0800
categories: [Java]
tags: [Stream]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/java/thumbnail.png
---

-   **<u>스트림 : 데이터 처리 연산을 지원하도록 소스에서 추출된 연속된 요소</u>**
    -   filter,   map,   reduce,   find,   match,   sort 등등...
    -   여러 연산으로 순차적 병렬 실행 가능.
>   스트림 예제 코드
    -
```java
        List<String> collect = menu.stream()
                .filter(d -> d.getCalories() > 300)
                .map(Dish::getName)
                .limit(3)
                .collect(Collectors.toList());
        System.out.println(collect);
menu 리스트의 요소중 칼로리가 300 초과인 메뉴의 이름을 리스트로 3개 가져온다
```

>   **<u>filter</u>** <br>
>   람다를 인수로 받아 스트림에세 특정 요소를 제외시킨다.

>   **<u>map</u>** <br>
>   람다를 이용해 한 요소를 다른 요소로 변환, 정보를 추출한다.

>   **<u>limit</u>** <br>
>   스트림의 크기를 축소한다.

>   **<u>collect</u>** <br>
>   스트림을 다른 형식으로 변환한다.

-   **<u>스트림의 특징</u>**
    -   순차적으로 값에 접근.
    -   탐색된 스트림의 요소는 소비.
    -   반복을 알아서 처리하고 결과 스트림값을 어딘가에 저장해주는 내부 반복 사용.
        ```java
                List<String> collect = menu.stream()
                .map(Dish::getName)
                .collect(Collectors.toList());
        ```
        -   컬렉션의 경우 아래와 같이 사용자가 직접 요소를 반복해야한다.
            ```java
                List<String> names = new ArrayList<>();
                for(Dish d: menu){
                    names.add(d.getName());
                }    
            ```
<br>

-   **<u>스트림 연산</u>**
```java
        List<String> collect = menu.stream()
                .filter(d -> d.getCalories() > 300) <= 중간 연산
                .map(Dish::getName) <= 중간 연산
                .limit(3) <= 중간 연산
                .collect(Collectors.toList());  <=  최종 연산
        System.out.println(collect);
위와 같이 연산을 크게 2가지로 구분 가능.
```
<br>
-   **<u>중간 연산</u>**
    -   filter, sorted같은 중간 연산은 다른 스트림을 반환. 즉 중간 연산을 연결해서 사용 가능.
    -   중간 연산을 합친 다음에 합쳐진 연산을 최종 연산으로 한번에 처리.
    -   LAZY 하다.
        ```java
                List<String> collect = menu.stream()
                .filter(d -> {
                    System.out.println("filtering"+d.getName());
                    return d.getCalories() > 300;
                })
                .map(d -> {
                    System.out.println("mapping"+d.getName());
                    return d.getName();
                })
                .limit(3)
                .collect(Collectors.toList());
        System.out.println(collect);
        ```
        -   **<u>출력결과</u>** : 300 칼로리가 넘는 요리중 처음 3개만 선택.
            -   ```java
                    filteringpork
                    mappingpork 
                    filteringbeef
                    mappingbeef
                    filteringchicken
                    mappingchicken
                    [pork, beef, chicken]
                ```
            -   처음 3개만 선택된 이유 : limit연산, 쇼트서킷 기법
            -   filter와 map은 서로 다른 연산이지만 하나로 병합 : 루프 퓨전 기법
<br>

-   **<u>최종 연산</u>**
    -   중간 연산이 마친 뒤 반환하는 연산.

연산|형식|반환형식|연산의 인수|함수 디스크립터|
-|-|-|-|-|
filter|중간 연산|Stream &lt;T&gt;|Predicate &lt;T&gt;|T -> boolean|
map|중간 연산|Stream &lt;T&gt;|Function&lt;T,R&gt;|T -> R|
limit|중간 연산|Stream &lt;T&gt;|||
sorted|중간 연산|Stream &lt;T&gt;|Comparator&lt;T&gt;|(T, T) -> int|
distinct|중간 연산|Stream &lt;T&gt;|||

연산|형식|목적
-|-|-|
forEach|최종 연산|스트림의 각 요소를 소비하면서람다를 적용. void 반환|
count|최종 연산|스트림의 요소 개수를 반환한다. long 반환|
collect|최종 연산|스트림을 리듀스해서 리스트, 맵, 정수 형식의 컬렉션 생성|
