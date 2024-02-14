---
title: Discrete Mathematics(4)
description: 이산수학
date: 2024-02-13T12:00:000
categories: [ computer science, basic structures ]
tags: [ cs, set, kocw ]    # TAG는 반드시 소문자로 이루어져야함!
---

<h2> Set : 집합 </h2>

- 특징
  - 순서가 없다.
    - {1,2,3} ≡ {2,1,3}
  - 중복을 허용하지 않는다.
    - {1,1,2,3} ≡ {1,2,3}

- 중요 집합들
  - N = {1,2,3,4, ... }, 자연수 집합
  - Z = {...,-2,-1,0,1,2,...}, 모든 정수 집합
  - Z<sup>+</sup> = {1,2,3,...}, 양의 정수 집합
  - Q = {p / q \| p ∈ Z, q ∈ Z, and q ≠ 0}, 유리수 집합
  - R, 실수 집합.

<br>

<h2> Subset : 부분집합 </h2>

- 예시
  - {1,2} ⊆ {1,2,3}
  - {1,2,3} ⊆ {1,2,3}


- 특징
  - 모든 집합 S에 대해서 (∅ ⊆ S) 와 (S ⊆ S)를 만족한다.


- ∀x(x ∈ A ⇒ x ∈ B)
  - A : subset
  - B : superset


- 진부분집합 : A ⊂ B
  - A ⊆ B 그러나 A ≠ B
  - ∀x(x ⊆ A ⇒ x ∈ B) ∧ ∃x(x ∈ B ∧ x ∉ A)

<br>

<h2> Power Set : 멱집합(부분 집합의 집합) </h2>

- 특징
  - 모든 부분 집합의 집합
  - P(S)

- 예시
  - P({0,1}) = {∅,{0},{1},{0,1}}
  - P(∅) = {∅}
  - P({∅}) = {∅,{∅}}

<br>

<h2> Cartesian Products : 데카르트 곱 </h2>

- 특징
  - 두 집합 A, B가 있을때
  - A x B = {(x,y) \| (x ∈ A) ∧ (y ∈ B)}
  - A<sub>1</sub> X A<sub>2</sub> X .... X A<sub>n</sub> = {(x<sub>1</sub>,x<sub>2</sub>,...,x<sub>n</sub>) \| ∀i :
    x<sub>i</sub> ∈ A<sub>i</sub>}

- 예시
  - A={1,2}, B={a,b,c}
    - A X B = {(1,a), (1,b), (1,c), (2,a), (2,b), (2,c)}

<br>

<h2> Union : 합집합 </h2>

- 특징
  - A ∪ B = {x \| x ∈ A ∨ x ∈ B}

- 예시
  - {1,3,5} ∪ {1,2,3} = {1,2,3,5}

<br>

<h2> Intersection : 교집합 </h2>

- 특징
  - A ∩ B = {x \| x ∈ A ∧ x ∈ B}

- 예시
  - {1,3,5} ∩ {1,2,3} = {1,3}

<br>

<h2> Disjoint : 서로소 집합 </h2>

- 특징
  - 서로 다른 두 집합의 교집합이 ∅ 이다. A와 B 는 서로소 집합 이다.

- 예시
  - A = {1,3,5,7}, B = {2,4,6,8}
  - A ∩ B = ∅

<br>

<h2> Difference : 차집합 </h2>

- 특징
  - A - B = {x \| x ∈ A ∧ x ∉ B}
  - 빼는 대상이 되는 집합에 따라 결과가 달라진다

- 예시
  - {1,3,5} - {1,2,3} = {5}
  - {1,2,3} - {1,3,5} = {2}

<br>

<h2> Complement : 여집합 </h2>

- 특징
  - 전체 집합 개념이 들어간다.
  - A<sup>C</sup> = {x \| x ∉ A} or U - A, 여기서 U 는 전체 집합이다.









