---
title: spring security(4-2) 전체 설정
author: Angrypig
date: 2023-08-12 15:00:00 +0800
categories: [Spring]
tags: [Security]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/spring/security.png
---

- 전체 구성
    - 구성 : 로그인 성공, 실패, 인증안된 사용자에 대한 return을 JSON 형식으로 반환시킨다.
    - 회원가입 : 이메일을 통한 토큰인증으로 진행.
        - 관리자용 Gmail 계정을 사용하여 인증 메일 발송.
        - member, member_registration_token 2개의 테이블을 이용한 인증
    - 역할, 권한 설정 :
        - 역할 : ADMIN, USER, GUEST
        - 권한 : member, authority, member_authority  3개의 테이블을 사용한 다대다 권한 설정.
    
    ```jsx
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
                    .authenticationEntryPoint(customAuthenticationEntryPoint)   //  인증되지 않은 사용자가 접근. ToDO
    
                    .and()
                    .formLogin()
                    .successHandler(customAuthenticationSuccessHandler) //  로그인에 성공함. ToDO
                    .failureHandler(customAuthenticationFailureHandler) //  로그인에 실패함. ToDO
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
    

- 인증되지 않는 사용자가 접근
    - `CustomAuthenticationEntryPoint`: `AuthenticationEntryPoint` 구현. ErrorDetails 형식으로 리턴한다.
        - 구현 코드
        
        ```jsx
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
        
    
- 사용자 로그인 성공
    - `CustomAuthenticationSuccessHandler` : `AuthenticationSuccessHandler` 구현, MemberDto를 반환한다.
        - 구현 코드
        
        ```jsx
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
        
- 사용자 로그인에 실패
    - `CustomAuthenticationFailureHandler` : `AuthenticationFailureHandler` 구현. ErrorDetails 형식으로 리턴한다.
        - 구현 코드
        
        ```jsx
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
        
                ErrorDetails.ErrorDetailsDateToString detailMessage =
                        ErrorDetails.ErrorDetailsDateToString.builder()
                                .timeStamp(THIS_LOCAL_DATE_TIME.toString())
                                .message(String.format("아이디(%s)와 비밀번호를 다시 확인해 주세요.", username))
                                .details(requestUri)
                                .build();
        
                response.getWriter().write(objectMapper.writeValueAsString(detailMessage));
            }
        
        }
        ```
        
- 인증 공급자 서비스
    - `AuthenticationProviderService` : `AuthenticationProvider` 구현
        - `JpaUserDetailsService` , `BCryptPasswordEncoder` 사용
            - `JpaUserDetailsService` 구현 코드
            
            ```jsx
            @Slf4j
            @Service
            @RequiredArgsConstructor
            public class JpaUserDetailsService implements UserDetailsService {
            
                private final TemplateEngine templateEngine;
                private final EmailService emailService;
                private final MemberRepository memberRepository;
                private final BCryptPasswordEncoder bcryptPasswordEncoder;
                private final MemberRegistrationTokenService memberRegistrationTokenService;
            
                @Override
                public CustomUserDetails loadUserByUsername(String id) {
            
                    log.info("JpaUserDetailsService loadUserByUsername 진입");
                    Supplier<UsernameNotFoundException> s = () -> new UsernameNotFoundException("가입이 안된 사용자 입니다.");
            
                    return new CustomUserDetails(memberRepository.selectByLoginMemberId(id).orElseThrow(s));
            
                }
            
                public String signUpMember(MemberDto memberDto) {
            
                    log.info("signUpMember!!");
            
                    Member memberEntity = memberDto.toEntity();
                    String email = memberEntity.getEmail();
            
                    log.info("member entity = {}", memberEntity);
            
                    boolean memberExists = memberRepository.findByEmail(email).isPresent();
            
                    if (memberExists) throw new ResourceDuplicateException("Member Register", "email", email);
            
                    String encodedPassword = bcryptPasswordEncoder.encode(memberEntity.getPassword());
                    log.info("encodedPassword = {}", encodedPassword);
            
                    memberEntity.encodedPassword(encodedPassword);
            
                    memberRepository.save(memberEntity);
            
                    //  ToDO : send confirmation token
                    //  활성화를 false 로 default 하고 이메일 인증이 완료되면 true 로 바꾼다.
            
                    String token = UUID.randomUUID().toString();
                    LocalDateTime now = LocalDateTime.now();
            
                    MemberRegistrationToken memberRegistrationToken =
                            MemberRegistrationToken.builder()
                                    .token(token)
                                    .createdDt(now)
                                    .expiredDt(now.plusMinutes(15L))
                                    .member(memberEntity)
                                    .build();
            
                    memberRegistrationTokenService.saveConfirmationToken(memberRegistrationToken);
            
                    //  ToDo : send email
                    String link = "http://localhost:8089/members/registration/confirm?token=" + token;
                    emailService.send(
                            memberEntity.getEmail(),
                            buildEmail(memberEntity.getFirstNm().concat(memberEntity.getLastNm()), link));
            
                    return token;
            
                }
            
                private String buildEmail(String name, String link) {
                    Context context = new Context();
                    context.setVariable("name", name);
                    context.setVariable("link", link);
                    return templateEngine.process("email-send", context);
                }
            
            }
            ```
            
- `JpaUserDetailsService` 의존성
    - `TemplateEngine` : 이메일 발송 form 을 그려주기 위한 의존성 (Thymeleaf 사용.)
        - 사용 부분
        
        ```jsx
        private String buildEmail(String name, String link) {
                Context context = new Context();
                context.setVariable("name", name);
                context.setVariable("link", link);
                return templateEngine.process("email-send", context);
            }
        ```
        
        - email-send.html
        
        ```jsx
        <!DOCTYPE html>
        <html xmlns:th="http://www.thymeleaf.org">
        <head>
            <meta charset="UTF-8">
            <title>Email Confirmation</title>
        </head>
        <body>
        <div style="font-family: Helvetica, Arial, sans-serif; font-size: 16px; margin: 0; color: #0b0c0c">
            안녕하세요 <span th:text="${name}"></span>님,<br>
            등록해 주셔서 감사합니다. 계정을 활성화하려면 아래 링크를 클릭하세요:<br>
            <a th:href="${link}">지금 활성화하기</a><br>
            링크는 15분 후에 만료됩니다. 곧 뵙겠습니다.<br>
        </div>
        </body>
        </html>
        ```
        
    - `EmailService` : email을 발송해 주기 위한 의존성
        - 사용 부분
        
        ```jsx
        String link = "http://localhost:8089/members/registration/confirm?token=" + token;
                emailService.send(
                        memberEntity.getEmail(),
                        buildEmail(memberEntity.getFirstNm().concat(memberEntity.getLastNm()),
                                link)
                );
        ```
        
        - `String link = "http://localhost:8089/members/registration/confirm?token=" + token;` 
        토큰 정보가 포함된 URL 을 html 코드에 심어 클릭시 자동으로 승인이 되게 만든다.
        - 가입을 시도하는 회원의 email, name, confirm url 을 파라미터로 넘긴다.
        
        ```jsx
                emailService.send(
                        memberEntity.getEmail(),
                        buildEmail(memberEntity.getFirstNm().concat(memberEntity.getLastNm()),
                                link)
                );
        ```
        
    - `MemberRepository`, `BCryptPasswordEncoder` : 설명 생략
    - `MemberRegistrationTokenService` : 이메일 인증에 사용되는 토큰을 관리하기 위한 서비스
        - `MemberRegistrationTokenRepository`
        
        ```jsx
        public interface MemberRegistrationTokenRepository extends JpaRepository<MemberRegistrationToken, Long> {
        
            Optional<MemberRegistrationToken> findByToken(String token);
        
            @Transactional
            @Modifying
            @Query("UPDATE MemberRegistrationToken m " +
                    "SET m.confirmedDt = :confirmedDt " +
                    "WHERE m.token = :token")
            int updateConfirmedAt(@Param("token") String token,@Param("confirmedDt") LocalDateTime confirmedDt);
        
        }
        ```
        
        - `MemberRegistrationTokenService`
        
        ```jsx
        public interface MemberRegistrationTokenService {
            void saveConfirmationToken(MemberRegistrationToken memberRegistrationToken);
            Optional<MemberRegistrationToken> getToken(String token);
        
            int setConfirmedAt(String token);
        }
        ```
        
- 유저 디테일
    - `CustomUserDetails` : `UserDetails` 구현
        - 구현 코드
        
        ```jsx
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