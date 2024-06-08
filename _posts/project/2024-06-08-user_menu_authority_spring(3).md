---
title: 관리자 권한, 메뉴 관리 (4)
description: CustomAuthenticationProvider
date: 2024-06-08T16:00:000
categories: [ Spring ]
tags: [ spring security, spring security authentication provider ]
---

[user_menu_authority](https://angrypig123.github.io/posts/user_menu_authority_spring(2)/){:target="\_blank"}

<br>

- 이전에 포스팅에서 작성한 ```UserDetails```를 구현한 ```User```엔티티를 ```AuthenticationProvider```에서 어떤 식으로 사용하는지 작성해보려 합니다.
  - ```AuthenticationProvider```에 대한 설명은 생략하겠습니다.

- 해당 설정으로 기대하는 기능은 로그인한 사용자의 메뉴 권한정보 셋팅과 접근 가능한 메뉴 리스트를 ```SecurityContextHolder```에 포함하
  여 이를 이용한 권한체크와 화면단에 메뉴 UI를 그려주는 기능입니다.

<br>

<h2> CustomAuthenticationProvider </h2>

- 간단한 인증 공급자 코드 입니다. 포함하고 있는 유저 정보가 있으면 이전에 작성한 메소드들을 이용하여 필요한 데이터를 설정해줍니다.
  - 이전 글에 포스팅한 ```User```엔티티를 매핑해주는 ```mybatis```코드를 보면 알겠지만 유저가 접근 할 수 있는 메뉴 리스트를 따로 매핑해주지 않았습니다.
    해당 정보는 ```CustomAuthenticationProvider```를 이용하여 서버단에서 구성하여 매퍼 코드를 단순화 하기 위함이었습니다.
    - ```BackOfficeMenuService backOfficeMenuService```에 대한 설명은 해당 다음 포스팅에 하겠습니다.
  - ```java
    @Slf4j
    @Component
    @RequiredArgsConstructor
    public class CustomAuthenticationProvider implements AuthenticationProvider {

        private final PasswordEncoder passwordEncoder;
        private final DefaultUserService defaultUserService;
        private final BackOfficeMenuService backOfficeMenuService;

        @Override
        public Authentication authenticate(Authentication authentication) throws AuthenticationException {

            String email = authentication.getName();
            String password = authentication.getCredentials().toString();

            User userDetails = (User) defaultUserService.loadUserByUsername(email);

            if (passwordEncoder.matches(password, userDetails.getPassword())) {
                List<BackOfficeMenuRes> backOfficeMenuRes = backOfficeMenuService.backOfficeMenuToBackOfficeMenuRes(userDetails.getUserId());
                userDetails.loginUserMenuList(backOfficeMenuRes);
                return new UsernamePasswordAuthenticationToken(userDetails, password, userDetails.getAuthorities());
            } else {
                throw new BadCredentialsException("Bad credentials");
            }

        }

        @Override
        public boolean supports(Class<?> authentication) {
            return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
        }

    }
    ```

<br>

- 위의 설정으로 기대하는 결과는 ```User```엔티티의 모든 필드가 채워지는것 입니다. 이전 포스팅에서는 유저가 접근 할 수 있는 메뉴 리스트가 비어있었습니다.
  해당 설정을 인증 공급자를 통해 설정되는것을 기대했기에 해당 필드가 의도한대로 셋팅이 되었는지 확인하기 위한 테스트 코드를 작성하고 그 결과를 ```JSON```형태로 직관해보겠습니다.
  - ```setSecurityContextHolder```를 구성하는 부분은 작성되어있는 ```CustomAuthenticationProvider```코드와 같게 구성하여 화면단에서 유저가 로그인한 이후의
    흐름을 모방하기 위해 작성되었습니다. 실제 로그인 앤드포인트를 호출하기가 귀찮아서 작성하게 되었습니다.(어차피 걔도 인증 공급자를 호출할거니까....)
  - 테스트와 관련된 다른 코드 설명은 생략하겠습니다.
  - ```java
    @Slf4j
    @SpringBootTest
    @ActiveProfiles("local")
    @TestMethodOrder(MethodOrderer.OrderAnnotation.class)
    class PostingTest {

        private final String ADMIN_USER_ID = "ad_sub_admin";
        private final String ADMIN_PASSWORD = "1234";

        @Autowired
        private ObjectMapper objectMapper;

        @Autowired
        private PasswordEncoder passwordEncoder;

        @Autowired
        private DefaultUserService defaultUserService;

        @Autowired
        private BackOfficeMenuService backOfficeMenuService;

        private void setSecurityContextHolder(String userId, String password) {
            Authentication authentication = authenticationHelper(userId, password);
            SecurityContext securityContext = SecurityContextHolder.createEmptyContext();
            securityContext.setAuthentication(authentication);
            SecurityContextHolder.setContext(securityContext);
        }

        private Authentication authenticationHelper(String userId, String password) {
            User userDetails = (User) defaultUserService.loadUserByUsername(userId);
            if (passwordEncoder.matches(password, userDetails.getPassword())) {
                List<BackOfficeMenuRes> backOfficeMenuRes = backOfficeMenuService.backOfficeMenuToBackOfficeMenuRes(userDetails.getUserId());
                userDetails.loginUserMenuList(backOfficeMenuRes);
                return new UsernamePasswordAuthenticationToken(userDetails, password, userDetails.getAuthorities());
            } else {
                throw new BadCredentialsException("Bad credentials");
            }
        }

        @AfterEach
        void afterEach() {
            clearSecurityContextHolder();
        }

        protected void clearSecurityContextHolder() {
            SecurityContextHolder.clearContext();
        }

        @Test
        @Order(1)
        void securityContextHolderSetUserTest() throws Exception {

            setSecurityContextHolder(ADMIN_USER_ID, ADMIN_PASSWORD);
            SecurityContext context = SecurityContextHolder.getContext();

            Authentication authentication = context.getAuthentication();
            Assertions.assertNotNull(authentication);

            Object principal = authentication.getPrincipal();
            Assertions.assertInstanceOf(User.class, principal);

            User user = (User) principal;
            Assertions.assertFalse(user.getUserMenuAuthorityResList().isEmpty());

            log.info("user = {}", objectMapper.writeValueAsString(user));

        }

    }
    ```

<br>

- 츨력된 결과 입니다. 인증 공급자를 통해 사이트를 구성하는데 필요한 해당 유저에 대한 모든 정보를 설정할 수 있게 된거같습니다.
  해당 값을 이용하여 화면을 구성하는건 이제 ```FRONT```의 몫입니다.
  - ![result](https://github.com/AngryPig123/AngryPig123.github.io/assets/86225268/ee9d3cc1-1ddc-40ae-81cf-3dbe56023eee)
