---
title: 스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술 (6)
author: Angrypig
date: 2023-08-03 18:30:00 +0800
categories: [MVC1]
tags: [Servlet]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/spring/mvc1.png
---

  **<u>프로젝트 생성</u>**<br>
  [start.spring.io](https://start.spring.io/)
  ![CREATE PROJECT](/assets/images/spring/mvc1/create_project.png)

> build.gradle.kts

```java
plugins {
	java
	war
	id("org.springframework.boot") version "2.7.14"
	id("io.spring.dependency-management") version "1.0.15.RELEASE"
}

group = "hello"
version = "0.0.1-SNAPSHOT"

java {
	sourceCompatibility = JavaVersion.VERSION_11
}

configurations {
	compileOnly {
		extendsFrom(configurations.annotationProcessor.get())
	}
}

repositories {
	mavenCentral()
}

dependencies {
	implementation("org.springframework.boot:spring-boot-starter-web")
	compileOnly("org.projectlombok:lombok")
	annotationProcessor("org.projectlombok:lombok")
	providedRuntime("org.springframework.boot:spring-boot-starter-tomcat")
	testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.withType<Test> {
	useJUnitPlatform()
}
```
