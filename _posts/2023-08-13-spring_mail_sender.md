---
title: spring mail sender
author: Angrypig
date: 2023-08-13 16:00:00 +0800
categories: [Spring]
tags: [Mail]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/spring/thumbnail.png
---

- `RegistrationServiceImpl` :회원가입 이메일 인증 서비스

  - 전체 로직

  ```java
  @Slf4j
  @Service
  @RequiredArgsConstructor
  public class RegistrationServiceImpl implements RegistrationService {
  
      private final EmailService emailService;
      private final MessageSource messageSource;
      private final TemplateEngine templateEngine;
      private final MemberRepository memberRepository;
      private final BCryptPasswordEncoder bcryptPasswordEncoder;
      private final MemberRegistrationTokenService memberRegistrationTokenService;
  
      @Value("${local.dev.host}")
      private String DEV_HOST;
  
      @Override
      @Transactional
      public RegistrationResponse register(MemberDto memberDto) {
  
          Member memberEntity = memberDto.toEntity();
          Optional<Member> memberRepositoryByEmail = memberRepository.findByEmail(memberEntity.getEmail());
  
          if (memberRepositoryByEmail.isEmpty()) {
              sendEmail(memberEntity);
              return RegistrationResponse.builder()
                      .message(messageSource.getMessage("MRG001", null, LocaleContextHolder.getLocale()))
                      .code("MRG001")
                      .build();
          }
  
          Member findByMember = memberRepositoryByEmail.get();
  
          if (findByMember.getEnabled()) {
              return RegistrationResponse.builder()
                      .message(messageSource.getMessage("MRG002", null, LocaleContextHolder.getLocale()))
                      .code("MRG002")
                      .build();
          }
  
          Optional<MemberRegistrationToken> registrationToken = memberRegistrationTokenService.findByMember(memberRepositoryByEmail.get());
  
          if (registrationToken.isEmpty()) {
              return RegistrationResponse.builder()
                      .message(messageSource.getMessage("MRG005", null, LocaleContextHolder.getLocale()))
                      .code("MRG005")
                      .build();
          }
  
          MemberRegistrationToken memberRegistrationToken = registrationToken.get();
  
          if (memberRegistrationToken.getExpiredDt().isAfter(LocalDateTime.now())) {
              return RegistrationResponse.builder()
                      .message(messageSource.getMessage("MRG003", null, LocaleContextHolder.getLocale()))
                      .code("MRG003")
                      .build();
          }
  
          this.deleteMemberAndRegisterToken(memberRegistrationToken);
          return RegistrationResponse.builder()
                  .message(messageSource.getMessage("MRG004", null, LocaleContextHolder.getLocale()))
                  .code("MRG004")
                  .build();
  
      }
  
      @Override
      @Transactional
      public RegistrationResponse confirmToken(String token) {
  
          Optional<MemberRegistrationToken> memberRegistrationToken = memberRegistrationTokenService.getToken(token);
  
          if (memberRegistrationToken.isEmpty()) {
              return RegistrationResponse.builder()
                      .message(messageSource.getMessage("MRG006", null, LocaleContextHolder.getLocale()))
                      .code("MRG006")
                      .build();
          }
  
          MemberRegistrationToken registrationToken = memberRegistrationToken.get();
  
          if (registrationToken.getMember().getEnabled()) {
              return RegistrationResponse.builder()
                      .message(messageSource.getMessage("MRG002", null, LocaleContextHolder.getLocale()))
                      .code("MRG002")
                      .build();
          }
  
          if (registrationToken.getExpiredDt().isBefore(LocalDateTime.now())) {
              this.deleteMemberAndRegisterToken(registrationToken);
              return RegistrationResponse.builder()
                      .message(messageSource.getMessage("MRG004", null, LocaleContextHolder.getLocale()))
                      .code("MRG004")
                      .build();
          }
  
          MemberDto memberDto = memberRegistrationTokenService.returnMemberDto(token);
          memberRegistrationTokenService.setConfirmedAt(token);
          memberRepository.enableMember(memberDto.toEntity().getEmail());
          return RegistrationResponse.builder()
                  .message(messageSource.getMessage("MRG007", null, LocaleContextHolder.getLocale()))
                  .code("MRG007")
                  .build();
      }
  
      private void deleteMemberAndRegisterToken(MemberRegistrationToken memberRegistrationToken) {
          memberRepository.delete(memberRegistrationToken.getMember());
      }
  
      private void sendEmail(Member memberEntity) {
  
          //  회원이 아닌 경우.
          String encodedPassword = bcryptPasswordEncoder.encode(memberEntity.getPassword());
          memberEntity.setEncodedPassword(encodedPassword);
          memberRepository.save(memberEntity);
  
          String token = UUID.randomUUID().toString();
          LocalDateTime now = LocalDateTime.now();
  
          MemberRegistrationToken memberRegistrationToken =
                  MemberRegistrationToken.builder()
                          .token(token)
                          .createdDt(now)
                          .expiredDt(now.plusMinutes(15L))
  //                        .expiredDt(now.plusSeconds(10L))
                          .member(memberEntity)
                          .build();
  
          memberRegistrationTokenService.saveConfirmationToken(memberRegistrationToken);
  
          String link = UriComponentsBuilder
                  .fromUriString(DEV_HOST)
                  .path("/members/registration/confirm")
                  .queryParam("token", token)
                  .toUriString();
  
          emailService.send(memberEntity.getEmail(), buildEmail(memberEntity.getFirstNm().concat(memberEntity.getLastNm()), link));
  
      }
  
      private String buildEmail(String name, String link) {
          Context context = new Context();
          context.setVariable("name", name);
          context.setVariable("link", link);
          return templateEngine.process("email-send", context);
      }
  
  }
  ```

- `EmailServiceImpl` : 이메일 발송

  - 전체 로직

  ```java
  @Slf4j
  @Service
  @RequiredArgsConstructor
  public class EmailServiceImpl implements EmailService {
  
      private final JavaMailSender javaMailSender;
  
      @Async
      @Override
      public void send(String to, String email) {
  
          try {
  
              MimeMessage message = javaMailSender.createMimeMessage();
              MimeMessageHelper helper = new MimeMessageHelper(message, "UTF-8");
              helper.setText(email, true);
              helper.setTo(to);
              helper.setSubject("Confirm your email");
              javaMailSender.send(message);
  
          } catch (MessagingException e) {
              log.error("failed to send email", e);
              throw new IllegalStateException("failed to send email");
          }
  
      }
  
  }
  ```

- 발송 템플릿

  - Thymeleaf

  ```java
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

- 컨트롤러

  - `MemberRegistrationController`

  ```java
  @Slf4j
  @Controller
  @RequiredArgsConstructor
  @RequestMapping(path = "/members/registration")
  public class MemberRegistrationController {
  
      private final RegistrationService registrationService;
      private final MemberRegistrationTokenService memberRegistrationTokenService;
  
      @PostMapping
      @ResponseBody
      public ResponseEntity<RegistrationResponse> register(@Valid @RequestBody MemberDto memberDto) {
          return new ResponseEntity<>(registrationService.register(memberDto), HttpStatus.OK);
      }
  
      @GetMapping(path = "/confirm")
      public String confirm(@RequestParam("token") String token, Model model) {
  
          RegistrationResponse registrationResponse = registrationService.confirmToken(token);
          MemberDto memberDto = memberRegistrationTokenService.returnMemberDto(token);
          if (memberDto != null) {
              model.addAttribute("member", memberDto);
          }
          model.addAttribute("registrationResponse", registrationResponse);
          return "register";
  
      }
  
  }
  ```