---
title: 람다 표현식 - Comparator<T>
author: Angrypig
date: 2023-08-05 13:30:00 +0800
categories: [Java]
tags: [Comparator]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/java/thumbnail.png
---

- **<u> Comparator &lt;T&gt; </u>**
  - 두 객체의 대,소를 결정하는 역할.
  - sort 알고리즘에 사용
  - 단일 메서드인 `compare(T o1, T o2)`를 가지고 있다.
  - 예 ) 리스트 정렬, 이진 검색 트리 구현.

> Employee 클래스

```java
class Employee {
    private String name;
    private int age;

    public Employee() {
    }

    public Employee(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Employee{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

> 익명 내부 클래스를 사용한 커스텀 정렬 방식 : 인터페이스 직접 구현

```java
        Collections.sort(employees, new Comparator<Employee>() {
            @Override
            public int compare(Employee employee1, Employee employee2) {
                return employee1.getName().compareTo(employee2.getName());
            }
        });
```

> 람다 표현식을 사용한 커스텀 정렬 방식 : 람다 표현식 사용

```java
        Collections.sort(employees,
                (employee1, employee2) ->
                        employee1.getName().compareTo(employee2.getName())
        );
```

> 리스트의 요소를 람다 표현식을 사용하는 정렬 방식

```java
        employees.sort(
                (employee1, employee2) -> 
                        employee1.getName().compareTo(employee2.getName())
        );
```

> 메소드 참조를 사용한 커스텀 정렬 방식

```java
        employees.sort(Comparator.comparing(Employee::getName));
```

