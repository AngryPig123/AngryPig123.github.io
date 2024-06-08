---
title: 관리자 권한, 메뉴 관리 (2)
description: 프로젝트 구성
date: 2024-06-08T13:00:000
categories: [Spring]
tags: [flyway, h2 database, mybatis]
---


[user_menu_authority](https://angrypig123.github.io/posts/user_menu_authority/){:target="\_blank"}

<br>

- 이전에 작성한 ```ERD```를 가지고 실제 스프링 프로젝트에서 어떤 식으로 데이터를 마이그레이션 하고 ```mapper```를 작성하였는지 작성해보려 합니다.
  - ```mapper``` 는 ```mybatis```를 사용

<h2> 데이터 마이그레이션 </h2>

- 실제 프로젝트를 진행하다 보면은 개발 환경에 따라 데이터베이스 싱크가 안맞는 경우가 종종 생겨 개발에 차질이 생기는 경우가 빈번하였습니다. 이를 어떻게 해결하면 좋을까
  고민을 하다가 찾아보게 된게 ```flyway```라이브러리 였습니다. 이를 통해 결과적으로 모든 개발자 및 환경에서 동일한 데이터 베이스 상태를 유지할것을 기대하고 있습니다.
  라이브러리에 대한 설명은 생략하고 적용 방식만 작성해보려 합니다.

<br>

- ```build.gradle``` : 개발 환경을 맞추기 위한 ```dependencies``` 목록입니다. ```h2```데이터 베이스를 이용하여 데이터 베이스 서버가 정해지지 않은 상황에서
  개발자들이 개인의 로컬에 환경을 구성하지 않았으면 하는 생각에 적용하게 되었습니다.
  - ```text
      implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3'
      implementation 'org.flywaydb:flyway-core'
      implementation 'org.flywaydb:flyway-database-oracle'
      runtimeOnly 'com.h2database:h2'
    ```

<br>

- ```application-local.yaml``` : 로컬 단계에서 사용하는 설정 파일로 위의 구성에 필요한 설정값들이 들어가있습니다.
  - 아래와 같이 구성을 하면 로컬에서 스프링 프로젝트가 ```runtime```하는 생명 주기와 같이 실행되는 데이터베이스 서버를 얻을 수 있습니다.
  - ```yaml
      spring:
        datasource:
          driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
          url: jdbc:log4jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE;MODE=ORACLE;DATABASE_TO_LOWER=TRUE;CASE_INSENSITIVE_IDENTIFIERS=TRUE
          username: root
          password: root
      flyway:
        user: root
        password: root
        clean-disabled: true
        locations: classpath:/db/migration
        url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE;MODE=ORACLE;DATABASE_TO_LOWER=TRUE;CASE_INSENSITIVE_IDENTIFIERS=TRUE
      h2:
        console:
          enabled: true
    ```
      - 위의 설정에서 ```locations: classpath:/db/migration```경로에 데이터 베이스 상태 정보를 작성하면 ```flyway```라이브러리를 통해 마치 데이터 베이스를 ```git```을 이용하는것 처럼 버전관리를 할 수 있게됩니다.
        해당 경로에 저번 포스트에서 작성한 ```DDL```문을 해당 경로에 작성하여 넣고 테스트에 필요한 기초 데이터들을 추가적으로 구성하였습니다.

<br>

- ```h2-database```에 ```flyway```를 이용하여 마이그레이션된 메뉴 정보 데이터 입니다.

![image](https://github.com/AngryPig123/AngryPig123.github.io/assets/86225268/6fc2c037-aa66-45ac-ba04-9a14b9ceab0b)


<h2> BackOfficeMenuMapper </h2>

- 메뉴 정보를 가져오기 위한 매퍼 입니다. 사이트의 메뉴 UI를 그려줄 수 있게 전체 메뉴를 가져오는 매퍼와 해당 유저가 접근 권한을 가지고 있는 메뉴만 가져오기 위한 메퍼 2가지 매퍼를 포함하고 있습니다.
  - ```text
      <sql id="selectMenuInfoBasicSql">
          SELECT a.menu_id     AS "menuId",
                 a.name        AS "name",
                 a.DEPTH       AS "depth",
                 a.reference   AS "reference",
                 a.menu_order  AS "menuOrder",
                 a.create_date AS "createDate",
                 a.create_id   AS "createId",
                 a.update_date AS "updateDate",
                 a.update_id   AS "updateId"
          FROM menu a
      </sql>

      <select id="selectAllMenuList" resultType="BackOfficeMenu">
          <include refid="selectMenuInfoBasicSql"/>
      </select>

      <select id="selectMenuListByUserId" resultType="BackOfficeMenu">
          <include refid="selectMenuInfoBasicSql"/>
                   INNER JOIN menu_authority b ON b.menu_id = a.menu_id
                   INNER JOIN user_type_menu_authority c ON c.menu_authority_id = b.menu_authority_id
                   INNER JOIN common_code d ON d.common_code_id = c.common_code_id
                   INNER JOIN "user" e ON e.user_type = d.common_code_id
          WHERE 1 = 1
            AND e.USER_ID = #{userId}
          UNION ALL
          <include refid="selectMenuInfoBasicSql"/>
          WHERE a.menu_id IN (SELECT a.reference AS "reference"
                              FROM menu a
                                       INNER JOIN menu_authority b ON b.menu_id = a.menu_id
                                       INNER JOIN user_type_menu_authority c ON c.menu_authority_id = b.menu_authority_id
                                       INNER JOIN common_code d ON d.common_code_id = c.common_code_id
                                       INNER JOIN "user" e ON e.user_type = d.common_code_id
                              WHERE 1 = 1
                                AND e.USER_ID = #{userId}
                              GROUP BY a.reference)
      </select>
    ```
