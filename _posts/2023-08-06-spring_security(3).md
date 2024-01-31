---
title: spring security(3)
author: Angrypig
date: 2023-08-06 17:30:00 +0800
categories: [Spring]
tags: [Security]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/spring/security.png
---

-   **<u>AbstractAuthenticationProcessingFilter 구현</u>**
    -   Http 헤더의 Api-Key, Api-Secret 로 인증하는 방식의 필터 구현.
    -   RestHeaderAuthFilter 클래스 구현.
<br>

```java
@Slf4j
public class RestHeaderAuthFilter extends AbstractAuthenticationProcessingFilter {  //  AbstractAuthenticationProcessingFilter 해당 필터를 구현하여 인증 프로세스를 완성한다.

    public RestHeaderAuthFilter(RequestMatcher requiresAuthenticationRequestMatcher) {
        super(requiresAuthenticationRequestMatcher);
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse) throws AuthenticationException, IOException, ServletException {
        String userName = getUsername(httpServletRequest);
        String password = getPassword(httpServletRequest);

        if (userName == null) userName = "";
        if (password == null) password = "";

        log.info("Authenticating user = {}", userName);
        log.info("Authenticating password = {}", password);

        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(userName, password);    //  인증에 전달.

        return !StringUtils.isEmpty(userName) ? this.getAuthenticationManager().authenticate(token) : null;

    }   //  ToDO : Step1 기본 인증 필터 구현.    여기 까지만 구현하면 인증 성공시 302를 리턴한다.

    private String getPassword(HttpServletRequest httpServletRequest) {
        return httpServletRequest.getHeader("Api-Secret");
    }

    private String getUsername(HttpServletRequest httpServletRequest) {
        return httpServletRequest.getHeader("Api-Key");
    }

}

```

-   **<u>SecurityConfig 설정 추가</u>**
<br>

```java
    public RestHeaderAuthFilter restHeaderAuthFilter(AuthenticationManager authenticationManager) {
        RestHeaderAuthFilter filter = new RestHeaderAuthFilter(new AntPathRequestMatcher("/api/**"));    //  AntPathRequestMatcher ?? ToDO 알아보자
        filter.setAuthenticationManager(authenticationManager);
        return filter;


    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http
                .addFilterBefore(
                        restHeaderAuthFilter(
                                authenticationManager()),
                        UsernamePasswordAuthenticationFilter.class
                );  //  요청헤더에서 해당 자격 증명을 가져온다.

                ......

    }
```

<br>

-   테스트 코드

```java
    @Test
    void deleteBeer() throws Exception {
        mockMvc.perform(
                        delete("/api/v1/beer/{beerId}", "97df0c39-90c4-4ae0-b663-453e8e19c311")
                                .header("Api-Key", "spring").header("Api-Secret", "guru")
                )
                .andExpect(status().isOk());
    }

    @Test
    void deleteBeerHttpBasic() throws Exception {
        mockMvc.perform(
                        delete("/api/v1/beer/{beerId}", "97df0c39-90c4-4ae0-b663-453e8e19c311")
                                .with(httpBasic("spring", "guru"))
                )
                .andExpect(status().is2xxSuccessful());
    }

    @Test
    void deleteBeerNoAuth() throws Exception {
        mockMvc.perform(
                        delete("/api/v1/beer/{beerId}", "97df0c39-90c4-4ae0-b663-453e8e19c311")
                )
                .andExpect(status().isUnauthorized());
    }


    @Test
    void findBeers() throws Exception {
        mockMvc.perform(
                        get("/api/v1/beer/")
                                .header("Api-Key", "spring").header("Api-Secret", "guru")
                )
                .andExpect(status().isOk());
    }

    @Test
    void findBeerById() throws Exception {
        mockMvc.perform(
                        get("/api/v1/beer/{beerId}", "97df0c39-90c4-4ae0-b663-453e8e19c311")
                                .header("Api-Key", "spring").header("Api-Secret", "guru")
                )
                .andExpect(status().isOk());
    }

    @Test
    void findBeerByUpc() throws Exception {
        mockMvc.perform(
                        get("/api/v1/beerUpc/{upc}", "badParameter")
                                .header("Api-Key", "spring").header("Api-Secret", "guru")
                )
                .andExpect(status().isOk());
    }
``` 

-   결과
    -   왠지 모르게 302를 리턴한다
    -   AbstractAuthenticationProcessingFilter 클래스의 successfulAuthentication 여기서 302가 리턴됨
![Redirect](/assets/images/spring/security/302.png)

<br>

-   doFilter, successfulAuthentication 재정의.
    -   RestheaderAuthFilter에 해당 코드 추가.

```java
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Request is to process authentication");
        }

        Authentication authResult = attemptAuthentication(request, response);

        if (authResult != null) {
            this.successfulAuthentication(request, response, chain, authResult);
        } else {
            chain.doFilter(request, response);
        }

    }   //  ToDO : Step2

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {

        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Authentication success. Updating SecurityContextHolder to contain: " + authResult);
        }

        SecurityContextHolder.getContext().setAuthentication(authResult);   //  Spring Security의 컨텍스트 내에서 권한 부여를 설정한다.

    }   //  ToDO Step3

```

-   결과
    -   ![OK](/assets/images/spring/security/200.png)


-   인증이 안된 사용자 정보를 가지고 요청을 보냈을 경우.
```java
    @Test
    void deleteBeerBadCredentials() throws Exception {
        mockMvc.perform(
                        delete("/api/v1/beer/{beerId}", "97df0c39-90c4-4ae0-b663-453e8e19c311")
                                .header("Api-Key", "spring").header("Api-Secret", "guru33")
                )
                .andExpect(status().isUnauthorized());
    }
```

-   서버에러가 나버린다.
    ![Alt text](/assets/images/spring/security/BadCredentials.png)

<br>

-   해결 방법.
    -   AbstractAuthenticationProcessingFilter 클래스의 <br>
        unsuccessfulAuthentication(request, response, e) 를 재정의 하여 doFilter에 적용.
    -   

```java
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Request is to process authentication");
        }

        try {
            Authentication authResult = attemptAuthentication(request, response);
            if (authResult != null) {
                this.successfulAuthentication(request, response, chain, authResult);
            } else {
                chain.doFilter(request, response);
            }
        } catch (AuthenticationException authenticationException) {

            unsuccessfulAuthentication(request, response, authenticationException);

        }

    } 
```

-   결과
    ![BadCredentials](/assets/images/spring/security/BadCredentials_ok.png)