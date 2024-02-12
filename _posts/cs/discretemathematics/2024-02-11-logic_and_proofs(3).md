---
title: Discrete Mathematics(3)
description: 이산수학
date: 2024-02-12T17:00:000
categories: [ computer science, logic and proofs ]
tags: [ cs, truth table, kocw ]    # TAG는 반드시 소문자로 이루어져야함!
---

<h2> Predicates : 술어 </h2>

- x 는 3보다 크다
  - x : 값
  - 3보다 크다 : P = 술어
  - P(x)
- Q(x,y) = x는 y + 3과 같다.
  - Q(1,2) false ⇒ 1 ≠ 2 + 3
  - Q(3,0) true ⇒ 3 = 0 + 3
- n-place predicate
  - P(x<sub>1</sub>, x<sub>2</sub>, ... x<sub>n</sub>)

<br>

<h2> Quantifiers </h2>

- Quantifiers : 양화(공식을 만족하는 개인의 수를 지정하는 연산자)
  - Universal quantification : 보편 양화사
    - ∀xP(x)
    - "주어진 어떠한 ~에 대해서도", "모든 ~에 대해서"
  - Existential quantification : 존재 양화사
    - ∃xP(x)
    - 주어진 술어를 만족시키는 객체가 논의 영역에 적어도 하나 존재함.
  - Precedence of Quantifiers : 양화사의 우선 순위
    - 모든 논리 연산자에 대해 우선 순위를 가지고 있다.
    - ∀xP(x) ∨ Q(x) ≡ (∀xP(x)) ∨ Q(x)

<br>

<h2> Logical Equivalences Involving Quantifiers </h2>

- 수량자와 관련된 논리적 동등성

| statement       | equivalent statement |
|-----------------|----------------------|
| ∀x(P(x) ∧ Q(x)) | (∀xP(x)) ∧ (∀xQ(x))  |
| ￢∃xP(x)         | ∀x￢P(x)              |
| ￢∀xP(x)         | ∃x￢P(x)              |

<br>

<h2> Nested Quantifiers </h2>

- ∀x∃y(x + y = 0)
  - 모든 x에 대하여 x + y = 0 을 만족시키는 y가 존재한다.
  - ∀xQ(x)에서 Q(x)는 ∃y(x + y = 0)
  - true
- ∃y∀x(x + y = 0)
  - 어떤 y는 모든 x에 대하여 x + y = 0를 만족시키는 y가 존재한다.
  - ∃yQ(y) 에서 Q(y)는 ∀x(x + y = 0)
  - false
- Negation nested quantifiers
  - ￢∀x∃y(xy=1) ≡ ∃x￢∃y(xy=1) ≡ ∃x∀y￢(xy=1) ≡ ∃x∀y(xy≠1)

<br>

<h2> Valid Arguments : 전제가 true 일때 결론이 반드시 true 이다. </h2>

- Argument
  - 제안 순서 (P<sub>1</sub>, P<sub>2</sub>, ... P<sub>n</sub>)
  - P<sub>n</sub> 이 되기 위해서는 이전의 제안들이 모두 ∧ 되어야한다.
  - P<sub>1</sub> ∧ P<sub>2</sub> ∧ ... ∧ P<sub>n-1</sub> ⇒ P<sub>n</sub>

<br>

<h2> Rules of Inference : 추론의 규칙 </h2>

- 기존에 검증된 true 를 가지고 새로운 사실을 밝혀내는것.

| rule of inference           | name                   |
|-----------------------------|------------------------|
| p ^ (p ⇒ q) ⇒ ∴q            | Modus ponens           |
| ￢q ∧ (p ⇒ q) ⇒ ∴￢q          | Modus tollens          |
| (p ⇒ q) ∧ (q ⇒ r) ⇒ ∴p ⇒ r  | Hypothetical syllogism |
| (p ∨ q) ∧ ￢p ⇒ ∴q           | Disjunctive syllogism  |
| p ⇒ ∴p ∨ q                  | Addition               |
| p ∧ q ⇒ ∴ p                 | Simplification         |
| p ∧ q ⇒ ∴p ∧ q              | Conjunction            |
| (p ∨ q) ∧ (￢p ∨ r) ⇒ ∴q ∧ r | Resolution             |

<br>
