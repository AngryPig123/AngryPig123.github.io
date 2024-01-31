---
title: 스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술 (5)
author: Angrypig
date: 2023-08-02 11:10:00 +0800
categories: [MVC1]
tags: [BackEnd]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/spring/mvc1.png
---


- 서블릿 - 1997
  - HTML 생성이 어려운

- JSP - 1999
  - HTML 생성은 편리하지만, 비지니스 로직까지 너무 많은 역할 담당

> HTML 생성, 비지니스 로직 두 관심사로 관심사를 나누기 시작

- 서블릿, JSP 조합 MVC 패턴 사용  
  - 모델, 뷰, 컨트롤러 역할을 나누어 개발
  - MVC패턴 자동화, 웹 기술을 편리하게 사용할 수 있는 기능 지원
  - 다양한 프래임워크 등장

<br>

> 자바 웹 기술 역사<br>
> 현재 사용 기술

- 어노테이션 기반의 스프링 MVC 등장
  - @Controller
- 스프링 부트의 등장
  - 스프링 부트는 서버를 내장
  - WAS 를직접 설치하던 과거와 다르게 JAR(빌드결과)에 WAS서버를 포함,<br>
    빌드 배포 단순화

<br>

> 자바 웹 기술 역사<br>
> 최신 기술 - 스프링 웹 기술의 분화

- Web Servlet - Srping MVC
- Web Reactive - Spring WebFlux
  - 최신 기술
  - 비동기 넌 블러킹 처리
  - 최소 쓰레드로 최대 성능 - 쓰레드 컨텍스트 스위칭 비용 효율화
  - 함수형 스타일로 개발 - 동시처리 코드 효율화
  - 서블릿 기술 사용 안함.
- 진입 장벽
  - 난이도가 매우 높음
  - RDB 지원 부족

<br>

> 자바 뷰 템플릿 역사<br>
> HTML을 편리하게 생성하는 뷰 기능

- JSP 
  - 속도 느림, 기능 부족
- 프리마커, 벨로시티
  - 속도 문제 해결, 다양한 기능
- 타임리프
  - 내추럴 템플릿 : HTML의 모양을 유지하면서 뷰 템플릿 적용 가능
  - **<u>스프링 MVC와 강력한 기능 통합</u>**
  
  