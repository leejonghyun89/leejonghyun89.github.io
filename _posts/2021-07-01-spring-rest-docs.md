---
layout: post
title: Spring Rest Docs 적용기
tags: Spring, Java, SpringRestDocs
math: true
---

## 목차

- [Spring Rest Docs 도입 이유](#1)
- [Spring Rest Docs와 Swagger 장단점](#2)
- [Spring Rest Docs 작성 방법](#3)
- [기능 추가](#4)
- [adoc 파일 만들기](#5)
- [참고자료](#6)

<a name="1" />

## 1. Spring Rest Docs 도입 이유

지금까지 개발을 진행하면서 API 명세에 대한 정보 문서화 또는 Swagger 사용을 해왔습니다.  
매번 API 개발을 할 때마다 명세에 대한 정보를 문서화를 하였고,  
Swagger 사용을 위해 매번 Controller, Dto 단에 Swagger 어노테이션을 추가해야 하니  
코드 보기가 좋지 않았습니다.

그러다가 API 명세서를 자동화해주는 것을 찾다가 Spring Rest Docs 찾게 되어,  
Spring Rest Docs를 적용하게 되었습니다.

<a name="2" />

## 2. Spring Rest Docs와 Swagger 장단점

|      |              Spring Rest Docs              |                 Swagger                 |
| :--: | :----------------------------------------: | :-------------------------------------: |
| 장점 |          제품코드에 영향이 없다.           | API 테스트 해 볼수 있는 화면을 제공한다 |
|      | 테스트 케이스가 성공해야 문서 작성이 된다. |             적용하기 쉽다.              |
| 단점 |              적용하기 어렵다.              |  제품코드에 어노테이션을 추가해야한다.  |
|      |                                            |    제품코드와 동기화가 안될수 있다.     |

<a name="3" />

## 3. Spring Rest Docs 작성 방법

- 개발 환경

  - Spring Boot 2.4.6
  - Gradle 6.9
  - JUnit5
  - MockMvc
  - AsciiDoc

- ./build.gradle

        plugins {
            id "org.springframework.boot" version "2.4.6"
            id "io.spring.dependency-management" version "1.0.11.RELEASE"
            id "org.asciidoctor.convert" version "1.5.3" // (1)
            id "java"
        }

        group = "kr.co.rest.doc"
        version = "0.0.1-SNAPSHOT"
        sourceCompatibility = "11"

        configurations {
            compileOnly {
                extendsFrom annotationProcessor
            }
        }

        configurations.all {
            exclude group: "org.springframework.boot", module: "spring-boot-starter-tomcat"
            exclude group: "io.undertow", module: "undertow-websockets-jsr"
        }

        repositories {
            mavenCentral()
        }

        ext {
            snippetsDir = "build/generated-snippets"
        }

        asciidoctor {
            dependsOn test // (2)
        }

        test {
            outputs.dir snippetsDir
            useJUnitPlatform()
        }

        asciidoctor {
            "org.springframework.restdocs:spring-restdocs-asciidoctor:2.0.3.RELEASE"
        }

        bootJar {
            dependsOn asciidoctor // (3)
            from("${asciidoctor.outputDir}/html5") { // (4)
                into "BOOT-INF/classes/static/docs"
            }
        }

        dependencies {
            implementation "org.springframework.boot:spring-boot-starter-web"
            implementation "org.springframework.boot:spring-boot-starter-undertow"
            implementation "org.springframework.boot:spring-boot-starter-security"
            implementation "org.springframework.boot:spring-boot-starter-validation"
            implementation "org.springframework.boot:spring-boot-starter-data-jpa"
            implementation "org.springframework.security.oauth:spring-security-oauth2:2.5.0.RELEASE"
            implementation "org.springframework.data:spring-data-envers"
            implementation "com.navercorp.lucy:lucy-xss-servlet:2.0.0"
            implementation "com.auth0:java-jwt:3.10.0"
            implementation "org.springframework.restdocs:spring-restdocs-mockmvc"
            implementation "org.springframework.restdocs:spring-restdocs-asciidoctor"

            compileOnly "org.projectlombok:lombok"
            compileOnly "com.querydsl:querydsl-apt"

            runtimeOnly "org.mariadb.jdbc:mariadb-java-client:2.7.3"
            runtimeOnly "com.h2database:h2"

            annotationProcessor "org.projectlombok:lombok"
            annotationProcessor "org.springframework.boot:spring-boot-configuration-processor"

            testImplementation "org.springframework.boot:spring-boot-starter-test" // (5)
            testImplementation "org.springframework.security:spring-security-test"
            testImplementation "org.springframework.restdocs:spring-restdocs-mockmvc"
        }

  - (1) AsciiDoc 파일을 컨버팅하고 build 디렉토리에 복사하기 위한 플러그인입니다.
  - (2) gradle build 시 test -> asciidoctor 순으로 수행됩니다.
  - (3) gradle build 시 asciidoctor -> bootJar 순으로 수행됩니다.
  - (4) gradle build 시 **./build/asciidoc/html5/** 에 html 파일이 생성 됩니다.  
    실제 배포시 BOOT-INF/classes 가 classpatch가 되기 때문에 아래와 같이 파일을 복사해야합니다.
  - (5) MockMvc 를 restdocs 에 사용할 수 있게 해주는 라이브러리입니다.

- test/\*\*/ApiDocumentationUtils

  ```java
  public interface ApiDocumentationUtils {
      static OperationRequestPreprocessor getDocumentRequest() {
          return preprocessRequest(
              modifyUris() // (1)
                  .scheme("https")
                  .host("docs.api.com")
                  .removePort(),
              prettyPrint() // (2)
          );
      }

      static OperationResponsePreprocessor getDocumentResponse() {
          return preprocessResponse(prettyPrint()); // (3)
      }
  }
  ```

  - (1) 문서상 uri 를 기본값인 http://localhost:8080 에서 https://docs.api.com 으로 변경하기 위해 사용합니다.
  - (2) 문서의 request 예쁘게 출력하기 위해 사용합니다.
  - (3) 문서의 response 를 예쁘게 출력하기 위해 사용합니다.

- test/\*\*/AccountControllerTest

  ```java
      @AutoConfigureRestDocs // (1)
      @SpringBootTest
      @AutoConfigureMockMvc
      @ExtendWith(RestDocumentationExtension.class)
      @ActiveProfiles("junit")
      public class AccountControllerTest {

          @Autowired
          private MockMvc mockMvc;

          @DisplayName("Account 로그인")
          @Test
          public void loginAccount() throws Exception {

              HashMap<String, String> loginForm = new HashMap<>();
              loginForm.put("loginId", "test");
              loginForm.put("password", "test2021@@");

              ResultActions result = mockMvc.perform(post("/api/accounts/login")
                  .accept(MediaType.APPLICATION_JSON)
                  .contentType(MediaType.APPLICATION_JSON)
                  .characterEncoding(StandardCharsets.UTF_8.name())
                  .content(objectMapper.writeValueAsString(loginForm)));

              result.andExpect(status().isOk())
                  .andDo(document("login-account", // (2)
                      ApiDocumentationUtils.getDocumentRequest(),
                      ApiDocumentationUtils.getDocumentResponse(),
                      requestFields(
                          fieldWithPath("loginId").type(JsonFieldType.STRING).description("로그인 ID"),
                          fieldWithPath("password").type(JsonFieldType.STRING).description("로그인 비밀번호")
                      ),
                      responseFields(
                          fieldWithPath("access_token").type(JsonFieldType.STRING).description("Access Token"),
                          fieldWithPath("refresh_token").type(JsonFieldType.STRING).description("Refresh Token")
                      )
                  )
              );
          }
      }
  ```

  - (1) getDocumentRequest에 선언된 uri 와 동일 기능을 제공합니다. 아래의 우선순위가 적용됩니다.
    > 1. @AutoConfigureRestDocs 에 uri 정보가 선언되어있으면 적용하며, 없으면 2단계로
    > 2. getDocumentRequest 에 uri 정보가 설정되어있으면 적용하며, 없으면 3단계로
    > 3. 기본 설정값 적용 (http://localhost:8080)
  - (2) test 수행 시 **./build/grenerated-snippets/** 하위에 지정한 문자열의 폴더 하위에 문서가 작성됩니다.

- Test 수행 시 결과

  > 그림 1. 폴더구조

  > ![image_1](/images/spring/spring-rest-docs-image-1.png)

  > 그림 2. http-request

  > ![image_2](/images/spring/spring-rest-docs-image-2.png)

  > 그림 3. http-response

  > ![image_3](/images/spring/spring-rest-docs-image-3.png)

  > 그림 4. request-fields

  > ![image_4](/images/spring/spring-rest-docs-image-4.png)

  > 그림 5. response-fields

  > ![image_5](/images/spring/spring-rest-docs-image-5.png)

<a name="4" />

## 4. 기능 추가

이 블로그를 참고하시고 진행하시는 개발자의 결과 페이지와 실제로 제가 올린 결과 페이지가 다를겁니다.  
기본 결과 페이지 request-fileds, response-fileds 필드 값이 영어로 노출이 될겁니다.  
필드 값을 한글로 표기 핳기 위해 request-fileds, response-fileds 에 대해 커스텀을 하였습니다.

커스텀하여 사용하기 위해서는 **<span style="color: red;">/test/resources/org/springframework/restdocs/templates/asciidoctor/</span>**  
디텍토리를 생성 후 커스텀 하기 위한 파일.snippet 을 생성해줍니다.

snippet 을 커스텀 하기 위한 기본 예제는 [링크](https://github.com/spring-projects/spring-restdocs/tree/main/spring-restdocs-core/src/main/resources/org/springframework/restdocs/templates/asciidoctor) 참고해주세요.

1. 필드값 한글화 변경 및 필수값 여부 추가

   > 그림 6. request-fileds.snippet

   > ![image_6](/images/spring/spring-rest-docs-image-6.png)

   - 실제 필수값 여부 적용 방법

   ```java
        @DisplayName("Account 로그인")
        @Test
        public void loginAccount() throws Exception {

            ...
            requestFields(
                fieldWithPath("loginId").type(JsonFieldType.STRING).description("로그인 ID"), // optional 미사용시 필수값 optional 사용시 필수값 X
                fieldWithPath("password").type(JsonFieldType.STRING).description("로그인 비밀번호")
            ),
            ...
        }
   ```

   > 그림 7. response-fileds.snippet

   > ![image_7](/images/spring/spring-rest-docs-image-7.png)

<a name="5" />

## 5. adoc 파일 만들기

test 가 성공했으면 ./buld/gnerated-snippets 디렉토리 안에 각각 test 단위로 디렉토리가 생성된 것을 확인할 수 있습니다.  
API 명세서를 실제 화면으로 보여주기 위해선 adoc 파일을 생성해야 합니다.

adoc 파일 경로는 **<span style="color: red;">/src/docs/asciidoc/</span>** 하위에 xxxxx.adoc 디렉토리를 생성합니다.

실제로 grdale 환경과 maven 환경의 디렉토리 구조는 다릅니다!!

- 빌드 환경에 따른 aodc 경로

| Build Tool |       Source fiels        |        Generated files        |
| :--------: | :-----------------------: | :---------------------------: |
|   Maven    | src/main/asciidoc/\*.adoc | target/generated-docs/\*.html |
|   Gradle   | src/docs/asciidoc/\*.adoc | build/asciidoc/html5/\*.html  |

- adoc 파일

        ifndef::snippets[]
        :snippets: ../../../build/generated-snippets
        endif::[]
        = Rest Docs Account API Document
        :doctype: book
        :toc: left
        :sectnums:
        :toclevels: 3
        :source-highlighter: highlightjs

        == Account
        === Account 로그인
        ==== 요청
        include::{snippets}/login-account/http-request.adoc[]
        include::{snippets}/login-account/request-fields.adoc[]
        ==== 응답
        include::{snippets}/login-account/http-response.adoc[]
        include::{snippets}/login-account/response-fields.adoc[]

- URL 요청에 따른 API 명세 노출

**WebMvcConfigurer** 상속 받은 MvcConfig에서 static/docs 안에 저장 된 html 접근할 수 있도록 추가합니다.

```java
    public class MvcConfig implements WebMvcConfigurer {

        ...
        @Override
        public void addResourceHandlers(final ResourceHandlerRegistry registry) {
            registry.addResourceHandler("/docs/**")
            .addResourceLocations("classpath:/static/docs/");
        }
        ...
    }
```

위와 같이 추가 후 **http://localhost:8080/docs/adoc파일명.html** 접속 시 API 명세서가 나오는 것을 확인할 수 있습니다.

> 그림 8. Spring Rest Docs 화면

![image_8](/images/spring/spring-rest-docs-image-8.png)

<a name="6" />

## 참고자료

[Spring Rest Docs 적용](https://techblog.woowahan.com/2597/)
