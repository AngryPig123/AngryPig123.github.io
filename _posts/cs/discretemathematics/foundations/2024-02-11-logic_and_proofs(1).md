---
title: Discrete Mathematics(1)
description: 이산수학
date: 2024-02-11T12:00:000
categories: [ computer science, logic and proofs ]
tags: [ cs, truth table, kocw ]    # TAG는 반드시 소문자로 이루어져야함!
---


<h2> Propositional Variables </h2>

- 명제를 나타내는 변수
  - p, q, r, s ....

<h2> Negation </h2>

> 부정, ￢p

| p | ￢p |
|---|----|
| T | F  |
| F | T  |

<br>

<h2> Conjunction(and, ∧) </h2>

> and 연산

| p | q | p ∧ q |
|---|---|-------|
| T | T | T     |
| T | F | F     |
| F | T | F     |
| F | F | F     |

<br>

<h2> Disjunction(or, ∨) </h2>

> or 연산

| p | q | p ∨ q |
|---|---|-------|
| T | T | T     |
| T | F | T     |
| F | T | T     |
| F | F | F     |

<br>

<h2> Condition Statements </h2>

- 가정어구 : p ⇒ q
  - if p then q : p(가정, hypothesis), q(결과, conclusion)
  - Converse(반대) : q ⇒ p
  - Inverse(역) : ￢p ⇒ ￢q
  - Contrapositive(대우) : ￢q ⇒ ￢p

| p | q | p ⇒ q |
|---|---|-------|
| T | T | T     |
| T | F | F     |
| F | T | T     |
| F | F | T     |

<br>

<h2> BiCondition Statements </h2>

- 이중 조건문 : p ⇔ q
  - if p and only if q
  - (p ⇒ q) ∧ (q ⇒ p)

| p | q | p ⇒ q | q ⇒ p | p ⇔ q |
|---|---|-------|-------|-------|
| T | T | T     | T     | T     |
| T | F | F     | T     | F     |
| F | T | T     | F     | F     |
| F | F | T     | T     | T     |










