---
title: spring security(4-1) 구성 엔티티
author: Angrypig
date: 2023-08-12 12:00:00 +0800
categories: [Spring]
tags: [Jpa]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/spring/security.png
---


- BaseDateEntity
    - 생성, 수정 날짜를 공통으로 하기 위한 MappedSuperClass
    
    ```jsx
    @Data
    @MappedSuperclass
    @EntityListeners(AuditingEntityListener.class)
    public abstract class BaseDateEntity {
    
        @CreatedDate
        @Column(name = "created_dt", updatable = false)
        private LocalDateTime createDt;
    
        @LastModifiedDate
        @Column(name = "updated_dt")
        private LocalDateTime updateDt;
    
    }
    ```
    
- Member
    - 회원 엔티티
    
    ```jsx
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
    
        public void encodedPassword(String password) {
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
                    .createdDt(super.getCreateDt())
                    .updatedDt(super.getUpdateDt())
                    .build();
        }
    
    }
    ```
    
- Authority
    - 권한 엔티티
    
    ```jsx
    @Entity
    @Getter
    @Table(name = "authority")
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    public class Authority extends BaseDateEntity {
    
        @Id
        @Column(name = "authority_no")
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long authorityNo;
    
        @Column(nullable = false, unique = true)
        private String authorityName;
    
        @OneToMany(mappedBy = "authority", cascade = CascadeType.REMOVE, fetch = FetchType.LAZY)
        private List<MemberAuthority> memberAuthority = new ArrayList<>();
    
        @Builder
        public Authority(String authorityName) {
            this.authorityName = authorityName;
        }
    
    }
    ```
    

- MemberAutiority
    - 회원 - 권한 엔티티
    
    ```jsx
    @Entity
    @Getter
    @Table(name = "member_authority",
            uniqueConstraints = {@UniqueConstraint(columnNames = {"member_no", "authority_no"})})
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    public class MemberAuthority extends BaseDateEntity {
    
        @Id
        @Column(name = "member_authority_no")
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long memberAuthorityNo;
    
        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "member_no")
        private Member member;
    
        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "authority_no")
        private Authority authority;
    
        @Builder
        public MemberAuthority(Member member, Authority authority) {
            this.member = member;
            this.authority = authority;
        }
    
    }
    ```
    
- MemberRegistrationToken
    - 회원 - 토큰 엔티티
    
    ```jsx
    @Entity
    @Getter
    @Table(name = "member_registration_token")
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    public class MemberRegistrationToken {
    
        @Id
        @Column(name = "member_registration_token_no")
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long memberRegistrationTokenNo;
    
        @Column(nullable = false)
        private String token;
        @Column(name = "created_dt", nullable = false)
        private LocalDateTime createdDt;
    
        @Column(name = "expired_dt", nullable = false)
        private LocalDateTime expiredDt;
    
        @Column(name = "confirmed_dt")
        private LocalDateTime confirmedDt;
    
        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "member_no", nullable = false)
        private Member member;
    
        @Builder
        public MemberRegistrationToken(String token, LocalDateTime createdDt, LocalDateTime expiredDt, Member member) {
            this.token = token;
            this.createdDt = createdDt;
            this.expiredDt = expiredDt;
            this.member = member;
        }
    
        public void setToken(String token) {
            this.token = token;
        }
    
        public void setCreatedDt(LocalDateTime createdDt) {
            this.createdDt = createdDt;
        }
    
        public void setExpiredDt(LocalDateTime expiredDt) {
            this.expiredDt = expiredDt;
        }
    
        public void setConfirmedDt(LocalDateTime confirmedDt) {
            this.confirmedDt = confirmedDt;
        }
    
    }
    ```