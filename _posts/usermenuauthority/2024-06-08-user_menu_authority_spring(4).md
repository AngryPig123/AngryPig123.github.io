---
title: 관리자 권한, 메뉴 관리 (5)
description:
date: 2024-06-08T21:30:000
categories: [ Spring ]
tags: [ spring security, method security ]
---

[user_menu_authority](https://angrypig123.github.io/posts/user_menu_authority_spring(3)/){:target="\_blank"}

<br>

- 이전 포스팅글들에서 구성한 정보를 가지고 어떤 식으로 인가 처리를 하는지에 대해 작성해보려 합니다.
  - 실제 코드에서는 ```@PreAuthorize```, ```@PostAuthorize```를 이용하여 ```authority```에 대한 사용자 제한을 구현했습니다.

<br>


<h2>테스트 컨트롤러</h2>

- 테스트를 위한 컨트롤러입니다. 코드를 보면 알겠지만 ```[메뉴명]_[CRUD]```와 같은 식으로 접근 권한을 주었습니다.
  - ```java
    @Slf4j
    @RestController
    @RequiredArgsConstructor
    @RequestMapping(path = "/api/menu-access-test")
    public class MenuAccessTestController {

        @GetMapping(path = "/adspm-create")
        @PreAuthorize("hasAnyAuthority('ST_ADSPM_CREATE')")
        public ResponseEntity<String> stAdspmCreate() {
            return new ResponseEntity<>("success", HttpStatus.OK);
        }

        @GetMapping(path = "/adspm-read")
        @PreAuthorize("hasAnyAuthority('ST_ADSPM_READ')")
        public ResponseEntity<String> stAdspmRead() {
            return new ResponseEntity<>("success", HttpStatus.OK);
        }

        @GetMapping(path = "/adspm-update")
        @PreAuthorize("hasAnyAuthority('ST_ADSPM_UPDATE')")
        public ResponseEntity<String> stAdspmUpdate() {
            return new ResponseEntity<>("success", HttpStatus.OK);
        }

        @GetMapping(path = "/adspm-delete")
        @PreAuthorize("hasAnyAuthority('ST_ADSPM_DELETE')")
        public ResponseEntity<String> stAdspmDelete() {
            return new ResponseEntity<>("success", HttpStatus.OK);
        }

        @GetMapping(path = "/aspm-create")
        @PreAuthorize("hasAnyAuthority('ST_ASPM_CREATE')")
        public ResponseEntity<String> stAspmCreate() {
            return new ResponseEntity<>("success", HttpStatus.OK);
        }

        @GetMapping(path = "/aspm-read")
        @PreAuthorize("hasAnyAuthority('ST_ASPM_READ')")
        public ResponseEntity<String> stAspmRead() {
            return new ResponseEntity<>("success", HttpStatus.OK);
        }

        @GetMapping(path = "/aspm-update")
        @PreAuthorize("hasAnyAuthority('ST_ASPM_UPDATE')")
        public ResponseEntity<String> stAspmUpdate() {
            return new ResponseEntity<>("success", HttpStatus.OK);
        }

        @GetMapping(path = "/aspm-delete")
        @PreAuthorize("hasAnyAuthority('ST_ASPM_DELETE')")
        public ResponseEntity<String> stAspmDelete() {
            return new ResponseEntity<>("success", HttpStatus.OK);
        }

    }
    ```

<br>

- ```flyway```를 이용해 초기에 설정한 메뉴 권한 정보 입니다.
  - ```sql
    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_ADA', 'ST_ADSPM_CREATE', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_ADA', 'ST_ADSPM_READ', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_ADA', 'ST_ADSPM_UPDATE', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_ADA', 'ST_ADSPM_DELETE', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_ADA', 'ST_ASPM_CREATE', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_ADA', 'ST_ASPM_READ', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_ADA', 'ST_ASPM_UPDATE', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_ADA', 'ST_ASPM_DELETE', 'site_admin');


    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_ADSA', 'ST_ADSPM_READ', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_ADSA', 'ST_ASPM_READ', 'site_admin');


    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_AA', 'ST_ASPM_CREATE', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_AA', 'ST_ASPM_READ', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_AA', 'ST_ASPM_UPDATE', 'site_admin');

    INSERT INTO user_type_menu_authority(common_code_id, menu_authority_id, create_id)
    VALUES ('UT_AA', 'ST_ASPM_DELETE', 'site_admin');

    INSERT INTO "user"(user_id, password, user_type)
    VALUES ('admin', '1234', 'UT_ADA');

    INSERT INTO "user"(user_id, password, user_type)
    VALUES ('sub_admin', '1234', 'UT_ADSA');

    INSERT INTO "user"(user_id, password, user_type)
    VALUES ('affiliate_admin', '1234', 'UT_AA');
    ```

<br>

- 테스트 코드 설정 클래스 입니다. 설명은 생략하겠습니다.
  - ```java
    @SpringBootTest
    @AutoConfigureMockMvc
    @ActiveProfiles("local")
    @TestMethodOrder(MethodOrderer.OrderAnnotation.class)
    public abstract class TestControllerTestBasic {

        @Autowired
        protected MockMvc mockMvc;

        @Autowired
        protected UserMapper userMapper;

        @Autowired
        protected ObjectMapper objectMapper;

        @MockBean
        protected BackOfficeMenuMapper backOfficeMenuMapper;

        protected final String MAIN_LOG_IN_PAGE = "/auth/loginPage";

        protected final String BACK_OFFICE_MENU_MODEL_KEY = "_backOfficeMenuRes";

        protected final String ADMIN_USER_ID = "admin";
        protected final String ADMIN_PASSWORD = "1234";

        protected final String SUB_ADMIN_USER_ID = "sub_admin";
        protected final String SUB_ADMIN_PASSWORD = "1234";

        protected final String AFFILIATE_ADMIN_USER_ID = "affiliate_admin";
        protected final String AFFILIATE_ADMIN_PASSWORD = "1234";

        @Autowired
        protected AuthenticationManagerBuilder authenticationManagerBuilder;

        @Autowired
        private PasswordEncoder passwordEncoder;

        @Autowired
        private DefaultUserService defaultUserService;

        @Autowired
        private BackOfficeMenuService backOfficeMenuService;

        @Autowired
        private CacheManager cacheManager;

        protected void setSecurityContextHolder(String userId, String password) throws Exception {
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

        protected void clearSecurityContextHolder() {
            SecurityContextHolder.clearContext();
        }

        @BeforeEach
        void beforeEach(WebApplicationContext wac) {
            mockMvc = MockMvcBuilders
                    .webAppContextSetup(wac)
                    .apply(springSecurity()).build();
        }

        @AfterEach
        void afterEach() {

            //  security context holder 초기화
            clearSecurityContextHolder();

            //  ehcache 초기화
            Collection<String> cacheNames = cacheManager.getCacheNames();
            for (String cacheName : cacheNames) {
                Cache cache = cacheManager.getCache(cacheName);
                Objects.requireNonNull(cache).clear();
            }
        }

    }
    ```

<br>

- 실제 테스트 코드를 실행하는 클래스 입니다. 간단하게 메소드에 선언된 권한, ```setSecurityContextHolder()```로 설정된 유저의 권한에 맞게 요청에 대한 응답 상태값을 검증하였습니다.
  관리자별로 설정한 권한대로 전체 테스트가 잘 통과하는것을 볼 수 있습니다.
  - ```java
    class TestControllerTest extends TestControllerTestBasic {


        @Test
        @Order(1)
        void adminAuthorityTest() throws Exception {
            setSecurityContextHolder(ADMIN_USER_ID, SUB_ADMIN_PASSWORD);

            mockMvc.perform(get("/api/menu-access-test/adspm-create"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/adspm-read"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/adspm-update"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/adspm-delete"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/aspm-create"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/aspm-read"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/aspm-update"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/aspm-delete"))
                    .andExpect(status().isOk());

        }

        @Test
        @Order(2)
        void subAdminAuthorityTest() throws Exception {
            setSecurityContextHolder(SUB_ADMIN_USER_ID, SUB_ADMIN_PASSWORD);

            mockMvc.perform(get("/api/menu-access-test/adspm-create"))
                    .andExpect(status().isForbidden());

            mockMvc.perform(get("/api/menu-access-test/adspm-read"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/adspm-update"))
                    .andExpect(status().isForbidden());

            mockMvc.perform(get("/api/menu-access-test/adspm-delete"))
                    .andExpect(status().isForbidden());

            mockMvc.perform(get("/api/menu-access-test/aspm-create"))
                    .andExpect(status().isForbidden());

            mockMvc.perform(get("/api/menu-access-test/aspm-read"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/aspm-update"))
                    .andExpect(status().isForbidden());

            mockMvc.perform(get("/api/menu-access-test/aspm-delete"))
                    .andExpect(status().isForbidden());

        }

        @Test
        @Order(3)
        void affiliateAdminAuthorityTest() throws Exception {
            setSecurityContextHolder(AFFILIATE_ADMIN_USER_ID, AFFILIATE_ADMIN_PASSWORD);

            mockMvc.perform(get("/api/menu-access-test/adspm-create"))
                    .andExpect(status().isForbidden());

            mockMvc.perform(get("/api/menu-access-test/adspm-read"))
                    .andExpect(status().isForbidden());

            mockMvc.perform(get("/api/menu-access-test/adspm-update"))
                    .andExpect(status().isForbidden());

            mockMvc.perform(get("/api/menu-access-test/adspm-delete"))
                    .andExpect(status().isForbidden());

            mockMvc.perform(get("/api/menu-access-test/aspm-create"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/aspm-read"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/aspm-update"))
                    .andExpect(status().isOk());

            mockMvc.perform(get("/api/menu-access-test/aspm-delete"))
                    .andExpect(status().isOk());

        }

    }
    ```
    - ![result](https://github.com/AngryPig123/AngryPig123.github.io/assets/86225268/035c8ccf-5d2b-491e-a231-db3c89c71eb4)
