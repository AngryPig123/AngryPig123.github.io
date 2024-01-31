---
title: spring security(2)
author: Angrypig
date: 2023-08-06 12:10:00 +0800
categories: [Spring]
tags: [Security]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/spring/security.png
---

![Security Flow](/assets/images/spring/security/security_flow.png)

-   Security 구현 목록.
1.  Authenication Filter
2.  Authentication Manager
3.  Authentication Provider
4.  User Detail Service
5.  Password Encoder
6.  Security Context

<br>

>   **<u>4.  User Detail Service</u>** <br>
>   User Detail Service 중 In-Memory 방식 구현 <br>
>   기존 properties 파일에 설정했던 username, password 삭제

-   SecurityConfig 클래스에 userDetailsService() 추가.

```java
    @Bean   //  <= 중요.
    @Override
    protected UserDetailsService userDetailsService() {
        UserDetails admin =
                User.withDefaultPasswordEncoder()
                        .username("spring")
                        .password("guru")
                        .roles("ADMIN")
                        .build();
        UserDetails user =
                User.withDefaultPasswordEncoder()
                        .username("user")
                        .password("password")
                        .roles("USER")
                        .build();
        return new InMemoryUserDetailsManager(admin, user);
    }
```

<br>

>   테스트 코드 작성

```java
    @Test
    void initCreationForm() throws Exception{
        mockMvc.perform(get("/beers/new").with(httpBasic("user","password")))
                .andExpect(status().isOk())
                .andExpect(view().name("beers/createBeer"))
                .andExpect(model().attributeExists("beer"));
    }
```

<br>

>   또 다른 구현 방법 <br>
>   UserDetailsService Bean 주입 없이 하는 방법.

```java
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("spring")
                .password("guru")
                .roles("ADMIN")
                .and()
                .withUser("user")
                .password("password")
                .roles("USER");
    }
```

>   실행 결과

```java
java.lang.IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"
```

-   비밀번호를 설정할때에 PasswordEncoder 설정을 안해줘서 생긴 에러
    -   password("password") 설정 부분에 사용할 PasswordEncoder 설정을 해줘야 한다.
    -   ex ) `password("{noop}password")`
    -   변경된 코드
        -   
```java
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("spring")
                .password("{noop}guru")     <====
                .roles("ADMIN")
                .and()
                .withUser("user")
                .password("{noop}password")     <====
                .roles("USER");
    }
```

<br>
<br>

>   **<u>5. PasswordEndoder</u>**
>   비밀번호 암호화 방식 설정
-   **<u>PasswordEncoderFactories.createDelegatingPasswordEncoder()</u>**
    -   여러 암호화 방식을 지원해주는 기능.
    -   지원하는 암호화 방식
        -
```java
    public static PasswordEncoder createDelegatingPasswordEncoder() {
        String encodingId = "bcrypt";
        Map<String, PasswordEncoder> encoders = new HashMap();
        encoders.put(encodingId, new BCryptPasswordEncoder());
        encoders.put("ldap", new LdapShaPasswordEncoder());
        encoders.put("MD4", new Md4PasswordEncoder());
        encoders.put("MD5", new MessageDigestPasswordEncoder("MD5"));
        encoders.put("noop", NoOpPasswordEncoder.getInstance());
        encoders.put("pbkdf2", new Pbkdf2PasswordEncoder());
        encoders.put("scrypt", new SCryptPasswordEncoder());
        encoders.put("SHA-1", new MessageDigestPasswordEncoder("SHA-1"));
        encoders.put("SHA-256", new MessageDigestPasswordEncoder("SHA-256"));
        encoders.put("sha256", new StandardPasswordEncoder());
        encoders.put("argon2", new Argon2PasswordEncoder());
        return new DelegatingPasswordEncoder(encodingId, encoders);
    }
```
    -   테스트
        -

```java
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth
                .inMemoryAuthentication()
                    .withUser("spring")
                    .password("{bcrypt}$2a$10$4VDe.dlvi8pBZ5pDs2WP8OjPIbjvgpcAulD.TlOR2iR3PWdnnioSq")
                    .roles("ADMIN")

                .and()
                    .withUser("user")
                    .password("{sha256}965fabb53e8b1060c46852dd3121c88eff5b3cbed80ec003b221ae3caf721dc524df53bb5c94fd36")
                    .roles("USER")

                .and()
                    .withUser("scott")
                    .password("{ldap}{SSHA}Q8ugr4L4yfmTyLVuC0IxyDdp+wvsbPKTA0KH9Q==")
                    .roles("CUSTOMER");
    }

    @Test
    void initCreationFormSpring() throws Exception{
        mockMvc.perform(get("/beers/new").with(httpBasic("spring","guru")))
                .andExpect(status().isOk())
                .andExpect(view().name("beers/createBeer"))
                .andExpect(model().attributeExists("beer"));
    }

    @Test
    void initCreationForm() throws Exception{
        mockMvc.perform(get("/beers/new").with(httpBasic("user","password")))
                .andExpect(status().isOk())
                .andExpect(view().name("beers/createBeer"))
                .andExpect(model().attributeExists("beer"));
    }


    @Test
    void initCreationFormWithScott() throws Exception{
        mockMvc.perform(get("/beers/new").with(httpBasic("scott","tiger")))
                .andExpect(status().isOk())
                .andExpect(view().name("beers/createBeer"))
                .andExpect(model().attributeExists("beer"));
    }
```
![암호화 결과](/assets/images/spring/security/password_endoder.png)