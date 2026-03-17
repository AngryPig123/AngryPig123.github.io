---
title: 관리자 권한, 메뉴 관리 (3)
description: spring security 설정, UserDetail 설정
date: 2024-06-08T16:00:000
categories: [ Spring ]
tags: [ spring security, spring security user details, mybatis ]
---


[user_menu_authority](https://angrypig123.github.io/posts/user_menu_authority_spring(1)/){:target="\_blank"}

<br>

- 이전에 작성한 구성 정보를 가지고 ```spring security```적용하여 메뉴별 ```CRUD```권한 체크와 로그인 사용자의 접근 가능한 메뉴정보를 가져오는 기능을 작성해 보려 합니다.
  이를 통해 가지고 있는 권한에 맞는 메뉴와 ```CRUD```권한별 인가 설정을 할 수 있게 되었습니다.


<br>


<h2> spring security config 설정</h2>

- ```spring security config``` 설정 정보 입니다.
  - ```securityFilterChain()```메소드 안에 들어가있는 메소드 들은 의존주입 받은 ```httpSecurity```를 이용하여 각 메소드별로 별개의 설정을 하게 하여 메소드명만으로도 구성정보를 쉽게 알아볼 수 있게 하였습니다.
    - 이번 포스팅에서는 ```securityFilterChain()```안에있는 구성 정보중 ```authenticationProviderConfig()```구성만 살펴보겠습니다.
  - ```@EnableMethodSecurity```를 적용하여 컨트롤러(메소드)별로 접근 권한을 설정하게 하였습니다.
  - ```java
    @Slf4j
    @Configuration
    @EnableMethodSecurity
    @RequiredArgsConstructor
    public class WebSecurityConfig {

        private final HttpSecurity httpSecurity;

        @Bean
        public SecurityFilterChain securityFilterChain() throws Exception {
            corsConfig();
            csrfConfig();
            headerConfig();
            authorizeHttpRequestsConfig();
            formLoginConfig();
            oAuth2LoginConfig();
            logOutConfig();
            exceptionHandlingConfig();
            authenticationProviderConfig();
            return httpSecurity.build();
        }

    }
    ```

<br>

<h2> User </h2>

- ```User```엔티티 클래스 입니다. 로그인한 사용자의 정보를 해당 엔티티로 구성하여 인증 공급자를 거쳐 ```SecurityContextHodler```안에서 관리되게 됩니다.
  로그인한 유저의 메뉴별 CURD 권한 리스트와, 메뉴 리스트를 포함하고 있습니다.
  - ```java
    @Getter
    @ToString
    @NoArgsConstructor
    public class User implements UserDetails {

        private String userId;
        private String password;
        private String userType;
        private List<BackOfficeMenu> userMenuAuthorityList;
        private List<BackOfficeMenuRes> userMenuAuthorityResList;

        @Override
        public Collection<? extends GrantedAuthority> getAuthorities() {
            return userMenuAuthorityList.stream()
                    .map(BackOfficeMenu::getMenuAuthorityId)
                    .map(SimpleGrantedAuthority::new)
                    .toList();
        }

        public void loginUserMenuList(List<BackOfficeMenuRes> backOfficeMenuRes){
            this.userMenuAuthorityResList = backOfficeMenuRes;
        }

        ...

    }
    ```
    - ```getAuthorities()``` 부분에서 가져온 ```BackOfficeMenu```에서 메뉴 권한 부분을 권한 리스트로 설정해줍니다. 작성하다 느낀건데 ```BackOfficeMenu```클래스의 이름이 마음에 안드는군요.
      todo 목록으로 해당 클래스명을 ```BackOfficeMenuAuthority```로 변경한다. 라고 기록후 넘어가겠습니다.

<br>

- ```mybatis``` 코드 입니다. 쿼리된 결과를 ```User```엔티티에 매핑시켜줍니다.
  - ```text
    <resultMap id="user" type="User">
            <result property="userId" column="userId"/>
            <result property="password" column="password"/>
            <result property="userType" column="userType"/>

            <collection property="userMenuAuthorityList" ofType="BackOfficeMenu">
                <result property="menuAuthorityId" column="menuAuthorityId"/>
                <result property="menuId" column="menuId"/>
                <result property="name" column="name"/>
                <result property="depth" column="depth"/>
                <result property="reference" column="reference"/>
                <result property="menuOrder" column="menuOrder"/>
                <result property="createDate" column="createDate"/>
                <result property="createId" column="createId"/>
                <result property="updateDate" column="updateDate"/>
                <result property="updateId" column="updateId"/>
            </collection>

        </resultMap>

        <sql id="selectUserInfoBasicSql">
            SELECT b.menu_authority_id AS "menuAuthorityId",
                   e.user_id           AS "userId",
                   e.password          AS "password",
                   e.user_type         AS "userType",
                   d.menu_id           AS "menuId",
                   d.name              AS "name",
                   d.depth             AS "depth",
                   d.reference         AS "reference",
                   d.menu_order        AS "menuOrder",
                   d.create_date       AS "createDate",
                   d.create_id         AS "createId",
                   d.update_date       AS "updateDate",
                   d.update_id         AS "updateId"
            FROM common_code a
                     INNER JOIN user_type_menu_authority b ON b.common_code_id = a.common_code_id
                     INNER JOIN menu_authority c ON b.menu_authority_id = c.menu_authority_id
                     INNER JOIN menu d ON d.menu_id = c.menu_id
                     INNER JOIN "user" e ON e.user_type = a.common_code_id
            WHERE 1 = 1
        </sql>

        <select id="selectUserByUserIdAndPassword" resultMap="user">
            <include refid="selectUserInfoBasicSql"/>
            AND e.user_id = #{userId}
            AND e.password = #{password}
        </select>

        <select id="selectUserByUserId" resultMap="user">
            <include refid="selectUserInfoBasicSql"/>
            AND e.user_id = #{userId}
        </select>
    ```

<br>

- ```mybatis``` 의 ```resultMap```이 의도한대로 ```User```엔티티에 바인딩이 되는지 간단한 테스트를 통해 직관해 보도록 하겠습니다.
  - ```@MybatisTest```를 이용하여 매퍼 테스트를 진행하였고 나머지 설명은 생략하겠습니다.
    - ```ObjectMapper```의 경우 전체 스프링 ```Bean```을 끌어오는게 아니라 주입이 되지 않아 직접 생성하여 사용하였습니다. 객체를 생성하여 테스트 하던 중에
      ```ObjectMapper```에서 ```LocalDateTime```의 형변환이 안되어 추가적으로 ```@BeforeEach```에서 형변환이 가능하게 해주는 모듈을 추가 설정 해주었습니다.
  - ```java
    @Slf4j
    @MybatisTest
    @ActiveProfiles("local")
    @AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
    class UserMapperTest {

        @Autowired
        private UserMapper userMapper;

        private final ObjectMapper objectMapper = new ObjectMapper();

        @BeforeEach
        void setup() {
            objectMapper.registerModule(new JavaTimeModule());
        }

        @Test
        void selectUserByUser() throws Exception {
            Optional<User> admin = userMapper.selectUserByUserIdAndPassword("ad_sub_admin", "1234");
            Assertions.assertTrue(admin.isPresent());
            User user = admin.get();
            Assertions.assertNotNull(user);
            log.info("user info = {}", objectMapper.writeValueAsString(user));
        }

    }
    ```
    - 유저 정보를 ```JSON```형태로 출력하였고 아래는 그 결과입니다.
      - ![result](https://github.com/AngryPig123/AngryPig123.github.io/assets/86225268/302c82d8-053f-4e46-8e12-6850663363e3)

<br>

- 다음 포스팅에서는 ```AuthenticationProvider```구성과 그 이후 설정들을 작성해보겠습니다.
