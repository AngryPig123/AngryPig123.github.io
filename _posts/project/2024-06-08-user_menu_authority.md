---
title: 관리자 권한, 메뉴 관리 (1)
description: 유저, 메뉴 CURD 메뉴 권한 테이블 설계
date: 2024-06-08T12:00:000
categories: [Database]
tags: [oracle, relationship]
---

<br>

- 최근에 참여한 프로젝트에서 ```3DEPTH```메뉴에 해당하는 ```CRUD``` 작업의 권한을 권한 유형을 통해 관리하고자 하는것이 요구사항이었습니다.
<br><br>


- 위의 요구사항을 충족시키기 위해 ```ERD``` 설계를 진행하면서 데이터 베이스 내에서 메뉴에 접근하는 권한을 어떻게 하면 효율적으로 관리할 수 있을지
구조를 고민하는 과정에 대해 작성해 보려 합니다.<br><br>


- 결과적으로 이를 통해 전체 메뉴관리, 메뉴별 ```CRUD```권한 관리, 권한 유형 관리 더 나아가 ```spring security```를 이용한
  인증, 인가 관리 까지 할 수 있는 기본적인 틀을 구성하게 되었습니다.<br><br>


<h2> 데이터 베이스 설계 </h2>

- 설계의 시작은 ```user```테이블로 시작을 했습니다. 해당 테이블에서는 ```user_type```으로  권한 유형을 가지고 있습니다. 예제를 단순화 시키기 위해
  해당 기능을 구현하는데 필요 없는 컬럼은 선언하지 않았습니다. ```user_type```은 공통 코드 테이블에서 따로 관리하도록 하였습니다.
  - ```sql
    CREATE TABLE "user"
    (
      user_id   VARCHAR2(50) PRIMARY KEY,
      password  VARCHAR2(255) NOT NULL,
      user_type VARCHAR2(50)  NOT NULL
    );
    ```

<br>

- ```user```테이블의 ```user_type```와 관계를 맺고 있는 ```common_code```테이블 입니다. 권한 그룹을 따로 관리할거 없이 프로젝트 전역에서
  쓰이는 공통 코드테이블로 관리해도 무방하다 판단되었습니다.
  - ```sql
      CREATE TABLE common_code
      (
        common_code_id VARCHAR2(50) PRIMARY KEY,
        reference_code VARCHAR2(50),
        use_yn         CHAR(1) DEFAULT 1 NOT NULL,
        description    VARCHAR2(50)
      );
    ```

<br>

- ```user_type_menu_authority```테이블 입니다 ```common_code```의 ```user_type```코드 그룹과 다대다 관계를 풀어내기 위한 테이블 입니다.
  해당 테이블 구성 후 ```menu_authority``` 테이블을 생성하여 관계를 맺게 됩니다.
  - ```sql
    CREATE TABLE user_type_menu_authority
    (
      common_code_id    VARCHAR2(50) NOT NULl,
      menu_authority_id VARCHAR2(50) NOT NULL,
      create_date       DATE DEFAULT SYSDATE,
      create_id         VARCHAR2(50) NOT NULL
    );
    ```

<br>

- ```menu_authority```테이블 입니다. ```3DEPTH```메뉴의 ```CURD```권한을 가지고 있는 테이블입니다.
  - ```sql
    CREATE TABLE menu_authority
    (
      menu_authority_id VARCHAR2(50) PRIMARY KEY,
      menu_id           VARCHAR2(50) NOT NULL,
      create_date       DATE DEFAULT SYSDATE,
      create_id         VARCHAR2(50) NOT NULL,
      update_date       DATE DEFAULT SYSDATE,
      update_id         VARCHAR2(50)
    );
    ```

<br>

- ```menu``` 테이블 입니다. ```DEPTH```와 ```REFERENCE``` 컬럼을 이용해 메뉴 정보를 구성할 수 있었습니다.
  - ```sql
    CREATE TABLE menu
    (
      menu_id     VARCHAR2(50) PRIMARY KEY,
      name        VARCHAR2(50)      NOT NULL,
      depth       CHAR(1) DEFAULT 1 NOT NULL,
      reference   VARCHAR2(50),
      menu_order  NUMBER(2),
      create_date DATE    DEFAULT SYSDATE,
      create_id   VARCHAR2(50)      NOT NULL,
      update_date DATE    DEFAULT SYSDATE,
      update_id   VARCHAR2(50)
    );
    ```

<br>

- 생성된 테이블에 대한 외래키 제약조건을 설정하는 추가 코드입니다.
  - ```sql
    ALTER TABLE "user"
    ADD CONSTRAINT fk_user_type_code_id FOREIGN KEY (user_type)
    REFERENCES common_code (common_code_id);

    ALTER TABLE user_type_menu_authority
    ADD CONSTRAINT pk_user_type_menu_authority PRIMARY KEY (common_code_id, menu_authority_id);

    ALTER TABLE user_type_menu_authority
    ADD CONSTRAINT fk_common_code_id FOREIGN KEY (common_code_id)
    REFERENCES common_code (common_code_id);

    ALTER TABLE user_type_menu_authority
    ADD CONSTRAINT fk_menu_authority_id FOREIGN KEY (menu_authority_id)
    REFERENCES menu_authority (menu_authority_id);

    ALTER TABLE menu_authority
    ADD CONSTRAINT fk_menu_id FOREIGN KEY (menu_id)
    REFERENCES menu (menu_id);
    ```

<br>

- ERD 다이어그램 입니다.

![erd](https://github.com/AngryPig123/AngryPig123.github.io/assets/86225268/f11ad2ca-857b-4f1a-819e-21e42b670cca)

