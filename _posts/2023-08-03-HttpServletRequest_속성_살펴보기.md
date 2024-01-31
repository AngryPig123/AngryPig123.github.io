---
title: 스프링 MVC 1편 - HttpServletRequest 속성 살펴보기
author: Angrypig
date: 2023-08-03 18:40:00 +0800
categories: [MVC1]
tags: [HttpServletRequest]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/spring/mvc1.png
---


-  HttpServletRuqest의 역할
    -   개발자가 HTTP 요청 메세지를 편리하게사용할 수 있도록 파싱해주는 역할을 제공 <br>
        그리고 그 결과를 HttpServletRequest에 담아서 제공해준다.

> HTTP 요청 메세지

```java
POST /save HTTP/1.1 
Host: localhost:8089
Content-Type: application/x-www-form-urlencoded

username=kim&age=20
```
-   START LINE
    -   HTTP METHOD
    -   URL
    -   Query String
    -   schema, protocol
-   HEADER
    -   헤더 조회
-   BODY 
    -   form 파라미터 형식 조회
    -   message body 데이터 직접 조회

> 추가로 여러가지 기능 제공.
-   임시 저장소 기능
    -   해당 HTTP  요청이 시작부터 끝날 때 까지 ㅇ지되는 임시 저장소 기능
        -   저장: `request.setAttribute(name,value)`
        -   조회: `request.getAttribute(name)`
    -   세션 관리 기능
        -   `request.getSession(create:true)`


>   요청 정보 살펴보기 <br>
[요청URL](http://localhost:8089/request-header){:target='_brank'}

```text
========================================
req.getMethod() = GET
req.getProtocol() = HTTP/1.1
req.getScheme() = http
req.getRequestURL() = http://localhost:8089/request-header
req.getRequestURI() = /request-header
req.getQueryString() = null
req.isSecure() = false
========================================

host
connection
sec-ch-ua
sec-ch-ua-mobile
sec-ch-ua-platform
upgrade-insecure-requests
user-agent
accept
sec-fetch-site
sec-fetch-mode
sec-fetch-user
sec-fetch-dest
referer
accept-encoding
accept-language
cookie
========================================

[Host 편의 조회]
request.getServerName() = localhost
request.getServerPort() = 8089

[Accept-Language 편의 조회]
locale = ko
locale = en
locale = en_US
locale = ko_KR
request.getLocale() = ko

[cookie 편의 조회]
JSESSIONID: 403689399AAC68330977E1E75BE40636
egovLatestServerTime: 1691048779546
egovExpireSessionTime: 1691052379546

[Content 편의 조회]
request.getContentType() = null
request.getContentLength() = -1
request.getCharacterEncoding() = UTF-8
========================================

[Remote 정보]
request.getRemoteHost() = 0:0:0:0:0:0:0:1
request.getRemoteAddr() = 0:0:0:0:0:0:0:1
request.getRemotePort() = 65155

[Local 정보]
request.getLocalName() = 0:0:0:0:0:0:0:1
request.getLocalAddr() = 0:0:0:0:0:0:0:1
request.getLocalPort() = 8089
========================================
```