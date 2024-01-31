---
title: spring security(1)
author: Angrypig
date: 2023-08-06 12:00:00 +0800
categories: [Spring]
tags: [Security]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/spring/security.png
---

-   **<u>Security Test</u>**
    -   유저 설정 변경. 기존 admin, UUID 로그인 방식을 properties에서 변경가능.
    -   
```java
spring.security.user.name=user
spring.security.user.password=password
```

        -   mockMvc.perform(get("/beers/find").with(httpBasic("user", "password"))) <br>
            테스트 코드에서 설정한 정보를 가지고 인증 테스트를 진행할 수 있다.

<br>

-   테스트 코드 작성
>   다음과 같은 설정 필요.
<br>

```java
@WebMvcTest
public class BeerControllerIt {
    @Autowired
    WebApplicationContext wac;

    MockMvc mockMvc;

    @BeforeEach
    void setUp() {
        mockMvc =
                MockMvcBuilders
                        .webAppContextSetup(wac)
                        .apply(springSecurity())    <=  해당 부분이 security 테스트를 가능하게 해준다.
                        .build();
    }
```
<br>

```java
    @Test
    @WithMockUser("spring") <=====  Mock User 설정.
    void findBeers_mock_user() throws Exception {
        mockMvc.perform(get("/beers/find"))
                .andExpect(status().isOk())
                .andExpect(view().name("beers/findBeers"))
                .andExpect(model().attributeExists("beer"));
    }

    @Test
    void findBeers_ok() throws Exception {
        mockMvc.perform(get("/beers/find").with(httpBasic("user", "password")))
                .andExpect(status().isOk())
                .andExpect(view().name("beers/findBeers"))
                .andExpect(model().attributeExists("beer"));
    }

    @Test
    void findBeers_fail() throws Exception {
        mockMvc.perform(get("/beers/find").with(httpBasic("foo", "bar")))
                .andExpect(status().is4xxClientError());
    }
```
<br>

-   **<u>SecurityConfig</u>**
-   
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
          인증 권한 설정이 들어가는 부분
        }
}
```


-   **<u>인증 권한 URL 패턴 설정.</u>**
    -   HttpMethod Method 별로 권한 설정을 다르게 할 수 있다
        -   ex) .antMatchers( **<u>HttpMethod.GET</u>** , "/api/v1/beer/**").permitAll();

```java
http
    .authorizeRequests(authority -> {
        authority.antMatchers("/", "/webjars/**", "/login", "/resources/**").permitAll()
                    .antMatchers("/beers/find/**", "/beers*").permitAll()
                    .antMatchers(HttpMethod.GET, "/api/v1/beer/**").permitAll()
                    .mvcMatchers(HttpMethod.GET, "/api/v1/beerUpc/{upc}").permitAll();
    })
    .authorizeRequests()
    .anyRequest().authenticated()
    .and().formLogin()
    .and().httpBasic();
```

<br>

-   **<u>테스트 코드</u>**

```java
    @Test
    void testGetIndexSlash() throws Exception {
        mockMvc.perform(get("/"))
                .andExpect(status().isOk());
    }

    @Test
    void findBeers() throws Exception {
        mockMvc.perform(get("/api/v1/beer/"))
                .andExpect(status().isOk());
    }

    @Test
    void findBeerById() throws Exception {
        mockMvc.perform(get("/api/v1/beer/{beerId}", "badParameter"))
                .andExpect(status().isBadRequest());
    }

    @Test
    void findBeerByUpc() throws Exception {
        mockMvc.perform(get("/api/v1/beerUpc/{upc}", "badParameter"))
                .andExpect(status().isOk());
    }
```

<br>

>   **<u>antMatchers VS mvcMatchers</u>** <br>
>   mvcMatchers : antMatchers 의 확장된 버전으로 좀 더 명확한 표현을 위해 사용. <br>
>   대부분 antMatchers 사용.

![Alt text](image.png)