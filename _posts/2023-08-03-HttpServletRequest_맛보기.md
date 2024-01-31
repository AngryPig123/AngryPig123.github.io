---
title: 스프링 MVC 1편 - HttpServletRequest 맛보기
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

[요청URL](http://localhost:8089/hello?username=홍길동){:target='_brank'}


```java
@WebServlet(name = "helloServlet", urlPatterns = "/hello")
//  name : 서블릿 이름, urlPatterns : url 매핑 주소
public class HelloServlet extends HttpServlet {
    
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        System.out.println("HelloServlet.service");
        System.out.println("req = " + req);
        System.out.println("resp = " + resp);

        //  Url 파라미터로 값 넣어서 뿌려보기
        Enumeration<String> parameterNames = req.getParameterNames();
        while(parameterNames.hasMoreElements()){
            String key = parameterNames.nextElement();
            System.out.println(key);
            System.out.println(req.getParameter(key));
        }

        //  응답 메세지 보내 보기,   RequestHeader, ResponseHeader 확인해보기.
        resp.setContentType("text/plain");
        resp.setCharacterEncoding("UTF-8");
        resp.getWriter().println("안녕? 이건 성공 메세지야");

        //  appliaction.properties : logging.level.org.apache.coyote.http11=debug
        //  해당 설정을 하면 request, response 정보를 콘솔로 확인할 수 있다.
    }
}
```
![Servlet Container 작동 방식](/assets/images/spring/mvc1/servlet-container.png)

>콘솔 실행 결과

```text
Host: localhost:8089
Connection: keep-alive
sec-ch-ua: "Not/A)Brand";v="99", "Google Chrome";v="115", "Chromium";v="115"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/115.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: http://127.0.0.1:4000/
Accept-Encoding: gzip, deflate, br
Accept-Language: ko,en;q=0.9,en-US;q=0.8,ko-KR;q=0.7
Cookie: JSESSIONID=403689399AAC68330977E1E75BE40636; egovLatestServerTime=1691048779546; egovExpireSessionTime=1691052379546

]
HelloServlet.service
req = org.apache.catalina.connector.RequestFacade@16164b7a
resp = org.apache.catalina.connector.ResponseFacade@1b9bbe5
```









