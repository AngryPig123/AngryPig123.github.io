---
title: spring security(4-3) 수정 버전.
author: Angrypig
date: 2023-08-13 15:00:00 +0800
categories: [Spring]
tags: [Jpa]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/spring/security.png
---

- `SecurityConfig`

  - 전체 설정

  ```java
  @Configuration
  @RequiredArgsConstructor
  public class SecurityConfig extends WebSecurityConfigurerAdapter {
  
      private final AuthenticationProviderService authenticationProviderService;
      private final CustomAuthenticationEntryPoint customAuthenticationEntryPoint;
      private final CustomAuthenticationSuccessHandler customAuthenticationSuccessHandler;
      private final CustomAuthenticationFailureHandler customAuthenticationFailureHandler;
  
      @Override
      protected void configure(HttpSecurity http) throws Exception {
  
          http
                  .exceptionHandling()
                  .authenticationEntryPoint(customAuthenticationEntryPoint)
  
                  .and()
                  .formLogin()
                  .successHandler(customAuthenticationSuccessHandler)
                  .failureHandler(customAuthenticationFailureHandler)
                  .permitAll()
  
                  .and()
                  .authorizeRequests(authority -> {
                      authority
                              .antMatchers("/", "/login", "/members/registration/**").permitAll();
                  })  //  authorization
  
                  .authorizeRequests()
                  .anyRequest()
                  .authenticated()
  
                  .and()
                  .csrf().disable();  //  csrf disable
  
      }
  
      @Override
      protected void configure(AuthenticationManagerBuilder auth) throws Exception {
          auth.authenticationProvider(authenticationProviderService);
      }
  
  }
  ```

- `AuthenticationProviderService`

  - 인증 공급자

  ```java
  @Slf4j
  @Service
  @RequiredArgsConstructor
  public class AuthenticationProviderService implements AuthenticationProvider {
  
      private final JpaUserDetailsService jpaUserDetailsService;
      private final BCryptPasswordEncoder bcryptPasswordEncoder;
  
      @Override
      @Transactional  //  필수
      public Authentication authenticate(Authentication authentication) throws AuthenticationException {
  
          String memberId = authentication.getName();
          String password = authentication.getCredentials().toString();
          CustomUserDetails userDetails = jpaUserDetailsService.loadUserByUsername(memberId);
  
          bcryptPasswordEncoder.matches(password, userDetails.getPassword());
          if (bcryptPasswordEncoder.matches(password, userDetails.getPassword())) {
  
              if (!userDetails.getMember().getEnabled()) {
                  throw new DisabledException("활성화가 되지 않은 회원 입니다. 인증을 다시 진행해 주세요.");
              }
  
              return new UsernamePasswordAuthenticationToken(
                      userDetails.getUsername(),
                      userDetails.getPassword(),
                      userDetails.getAuthorities());
          } else {
  
              throw new BadCredentialsException("권한이 없습니다.");
  
          }
  
      }
  
      @Override
      public boolean supports(Class<?> authentication) {
          return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
      }
  
  }
  ```

  - `JpaUserDetailsService`

    - 유저 상세 서비스

    ```java
    @Slf4j
    @Service
    @RequiredArgsConstructor
    public class JpaUserDetailsService implements UserDetailsService {
    
        private final MemberRepository memberRepository;
    
        @Override
        public CustomUserDetails loadUserByUsername(String id) {
            log.info("JpaUserDetailsService loadUserByUsername 진입");
            Supplier<UsernameNotFoundException> s = () -> new UsernameNotFoundException("가입이 안된 사용자 입니다.");
            return new CustomUserDetails(memberRepository.selectByLoginMemberId(id).orElseThrow(s));
        }
    
    }
    ```

    - 유저 상세

    ```java
    public class CustomUserDetails implements UserDetails {
    
        private final Member member;
    
        public CustomUserDetails(Member member) {
            this.member = member;
        }
    
        public Member getMember() {
            return member;
        }
    
        @Override
        public Collection<? extends GrantedAuthority> getAuthorities() {
            return member.getMemberAuthority().stream()
                    .map(
                            m -> new SimpleGrantedAuthority(m.getAuthority().getAuthorityName())
                    )
                    .collect(Collectors.toList());
        }
    
        @Override
        public String getPassword() {
            return member.getPassword();
        }
    
        @Override
        public String getUsername() {
            return member.getMemberId();
        }
    
        @Override
        public boolean isAccountNonExpired() {
            return true;
        }
    
        @Override
        public boolean isAccountNonLocked() {
            return member.getLocked();
        }
    
        @Override
        public boolean isCredentialsNonExpired() {
            return true;
        }
    
        @Override
        public boolean isEnabled() {
            return member.getEnabled();
        }
    
    }
    ```

    - 유저 Entity

    ```java
    @Entity
    @Getter
    @Table(name = "member")
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    public class Member extends BaseDateEntity {
    
        @Id
        @Column(name = "member_no")
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long memberNo;
    
        @Column(name = "member_id", unique = true, nullable = false)
        private String memberId;
    
        @Column(name = "password", nullable = false)
        private String password;
    
        @Column(name = "email", unique = true, nullable = false)
        private String email;
    
        @Column(name = "first_nm", nullable = false)
        private String firstNm;
    
        @Column(name = "last_nm", nullable = false)
        private String lastNm;
    
        @Enumerated(EnumType.STRING)
        private UserAuth userAuth;
    
        @OneToMany(mappedBy = "member", cascade = CascadeType.REMOVE, fetch = FetchType.LAZY)
        private List<MemberRegistrationToken> memberRegistrationTokens = new ArrayList<>();
    
        private Boolean locked = false;
        private Boolean enabled = false;
    
        @PrePersist
        public void prePersist() {
            if (userAuth == null) userAuth = UserAuth.USER;
            if (locked == null) locked = false;
            if (enabled == null) enabled = false;
        }
    
        @OneToMany(mappedBy = "member", cascade = CascadeType.REMOVE, fetch = FetchType.LAZY)
        private List<MemberAuthority> memberAuthority = new ArrayList<>();
    
        public void setEncodedPassword(String password) {
            this.password = password;
        }
    
        @Builder
        public Member(String memberId, String password, String email, String firstNm, String lastNm, UserAuth userAuth) {
            this.memberId = memberId;
            this.password = password;
            this.email = email;
            this.firstNm = firstNm;
            this.lastNm = lastNm;
            this.userAuth = userAuth;
        }
    
        public MemberDto toDto() {
            return MemberDto.builder()
                    .memberId(this.memberId)
                    .password(this.password)
                    .email(this.email)
                    .firstNm(this.firstNm)
                    .lastNm(this.lastNm)
                    .enabled(this.enabled)
                    .createdDt(super.getCreateDt())
                    .updatedDt(super.getUpdateDt())
                    .build();
        }
    
    }
    ```

- `CustomAuthenticationEntryPoint`

  - EntryPoint

  ```java
  @Slf4j
  @Component
  @RequiredArgsConstructor
  public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {
  
      private final ObjectMapper objectMapper;
  
      @Override
      public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
  
          response.setContentType("application/json");
          response.setCharacterEncoding("UTF-8");
          response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
  
          String requestURL = request.getRequestURL().toString();
  
          ErrorDetails.ErrorDetailsDateToString responseData =
                  ErrorDetails.ErrorDetailsDateToString.builder()
                          .timeStamp(THIS_LOCAL_DATE_TIME.toString())
                          .message("로그인 후 이용해 주시기바랍니다. 감사합니다.")
                          .details(requestURL)
                          .build();
  
          response.getWriter().write(objectMapper.writeValueAsString(responseData));
  
      }
  
  }
  ```

- `CustomAuthenticationSuccessHandler`

  - 로그인 성공 핸들러

  ```java
  @Slf4j
  @Component
  @RequiredArgsConstructor
  public class CustomAuthenticationSuccessHandler implements AuthenticationSuccessHandler {
  
      private final ObjectMapper objectMapper;
      private final MemberRepository memberRepository;
  
      @Override
      @Transactional
      public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
  
  //        Supplier<CustomUserNotFoundException> s = () -> new CustomUserNotFoundException("가입이 안된 사용자 입니다."); //  ToDO
  
          response.setContentType("application/json");
          response.setCharacterEncoding("UTF-8");
          response.setStatus(HttpServletResponse.SC_OK);
  
          String username = request.getParameter("username");
  
          Member member = memberRepository.selectByLoginMemberId(username).get(); //  ToDO
          log.info("member = {}", member);
  
          response.getWriter().write(objectMapper.writeValueAsString(member.toDto()));
  
      }
  
  }
  ```

- `CustomAuthenticationFailureHandler`

  - 로그인 실패 핸들러

  ```java
  @Slf4j
  @Component
  @RequiredArgsConstructor
  public class CustomAuthenticationFailureHandler implements AuthenticationFailureHandler {
  
      private final ObjectMapper objectMapper;
  
      @Override
      public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
  
          response.setContentType("application/json");
          response.setCharacterEncoding("UTF-8");
          response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
  
          String username = request.getParameter("username");
          String requestUri = request.getRequestURI();
  
          String errorMessage = getErrorMessage(exception).toString();
  
          ErrorDetails.ErrorDetailsDateToString detailMessage;
          if (!errorMessage.isEmpty()) {
              detailMessage =
                      ErrorDetails.ErrorDetailsDateToString.builder()
                              .timeStamp(LocalDateTime.now().toString())
                              .message(errorMessage)
                              .details(requestUri)
                              .build();
          } else {
              detailMessage =
                      ErrorDetails.ErrorDetailsDateToString.builder()
                              .timeStamp(LocalDateTime.now().toString())
                              .message(String.format("아이디(%s)와 비밀번호를 다시 확인해 주세요.", username))
                              .details(requestUri)
                              .build();
          }
  
          response.getWriter().write(objectMapper.writeValueAsString(detailMessage));
      }
  
      private StringBuffer getErrorMessage(AuthenticationException exception) {
  
          StringBuffer errorMessage = new StringBuffer();
  
          if (exception instanceof DisabledException) {
  
              DisabledException disabledException = (DisabledException) exception;
              errorMessage.append("DisabledException : ");
              errorMessage.append(disabledException.getMessage());
  
          } else if (exception instanceof BadCredentialsException) {
  
              BadCredentialsException disabledException = (BadCredentialsException) exception;
              errorMessage.append("BadCredentialsException : ");
              errorMessage.append(disabledException.getMessage());
  
          }
  
          return errorMessage;
  
      }
  
  }
  ```
