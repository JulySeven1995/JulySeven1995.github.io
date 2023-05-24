---
title: "Spring Boot + Keycloak 적용기"
categories:
  - Dev
tags:
  - Keycloak
  - Java
  - Spring Boot
layout: single
comments: true
---

지금 다니고 있는 직장에서 입사 하자 마자 Keycloak이라는 OAuth2 인증 프로토콜을 적용시키라는 막중한 임무를 받았기에,  
우여곡절 끝에 적용시킨 일지를 적어보고자 한다.  
사내에서 직접 구축한 OAuth 클라이언트와 인가 서버를 걷어내면서 작업했기에 더 힘들었던것 같다.

## Keycloak
>오픈 소스 기반의 싱글 사인온(SSO) 및 ID 및 액세스 관리 솔루션

Java로 작성된 플랫폼으로, OAuth 2.0 및 OpenID Connect과 같은 표준 기반 인증 및 권한 부여 프로토콜을 구현하고 있다.  
Keycloak은 사용자 인증, 권한 부여, 세션 관리 등 다양한 보안 기능을 제공하여 개발자들이 보안 관련 작업을 간소화하고 안정적인 인증 시스템을 구축할 수 있도록 지원한다.

Keycloak의 중요 기능들은 다음과 같다.

1. _SSO_  
  사용자가 한 번 인증하면 여러 애플리케이션 또는 서비스에 대해 재인증 없이 접근할 수 있는 SSO 기능을 제공  
  -> 우리 회사에서 적용하려고 했던 궁극적인 이유
2. _사용자 인증 및 관리_  
  사용자 계정 생성, 비밀번호 재설정, 이메일 확인 등의 기능을 포함하여 사용자 관리 기능 지원
3. _클라이언트 애플리케이션 지원_ 
  웹 애플리케이션, 모바일 앱, 백엔드 어플리케이션 등 다양한 플랫폼에서 Keycloak을 통해 인증 및 권한 부여를 처리 가능
4. _다양한 인증 방식_  
  사용자 아이디/비밀번호 인증, 소셜 미디어 계정을 통한 인증(Google, Facebook 등), SAML, LDAP, OpenID Connect 등의 인증 프로토콜을 지원
5. _액세스 제어 및 권한 부여_  
  사용자의 역할과 권한을 관리하고, 리소스에 대한 접근 권한을 제어
6. _클라우드 및 컨테이너 환경 지원_  
  클라우드 환경과 컨테이너 기반 아키텍처 지원. Kubernetes, Docker 등과 통합하여 확장성과 유연성을 제공


### Step 1. 설치
설치가 되지 않으면 죽도밥도 안되니 일단 Keycloak을 설치해보도록 하자.

준비물 : docker, terminal

1. Image Pull  
  `docker pull jboss/keycloak` 명령어를 통해 이미지를 받는다.
    ```
    Last login: Tue May 23 15:57:27 on ttys002
    jaeho@mac ~ % docker pull jboss/keycloak
    Using default tag: latest
    latest: Pulling from jboss/keycloak
    ac10f00499d5: Pull complete
    96d53117c12e: Pull complete
    1d929376eb7f: Pull complete
    93e1e1b6d192: Pull complete
    f353ba0db29e: Pull complet
    Digest: sha256:abdb1aea6c671f61a594af599f63fbe78c9631767886d9030bc774d908422d0a
    Status: Downloaded newer image for jboss/keycloak:latest
    docker.io/jboss/keycloak:latest
    ```

2. Docker run  
  `docker run -p 9000:8080 -e KEYCLOAK_USER=jaeho -e KEYCLOAK_PASSWORD=1234 jboss/keycloak`  
  콘솔 관리자 계정을 지정해주고 실행시켜주자.
    ```
    =========================================================================

    Using Embedded H2 database

    =========================================================================

    =========================================================================

    JBoss Bootstrap Environment

    JBOSS_HOME: /opt/jboss/keycloak

    ...

    07:15:47,439 INFO  [org.jboss.as.server] (Controller Boot Thread) WFLYSRV0212: Resuming server
    07:15:47,442 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0025: Keycloak 16.1.0 (WildFly Core 18.0.0.Final) started in 10777ms - Started 674 of 975 services (696 services are lazy, passive or on-demand)
    07:15:47,443 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0060: Http management interface listening on http://127.0.0.1:9990/management
    07:15:47,443 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0051: Admin console listening on http://127.0.0.1:9990
    ```
    Admin console listening 어쩌구가 나오면 성공적으로 실행된것이다.  
    http://127.0.0.1:9000 으로 접속해서 웰컴 화면이 잘 나타나는지 확인해보자.  
    1. __Welcome 페이지__  
      ![](/assets/images/keycloak-welcome.png)  
      Administration Console 탭을 누르고, 명령어에 입력한 계정 정보를 입력하면 관리자로써 관리 콘솔 페이지에 접근 할 수 있다.  
    2. __Login 페이지__  
      ![](/assets/images/keycloak-administrator-login.png)
    3. __Console 페이지__  
      ![](/assets/images/keycloak-console-intro.png)  

### Step 2. 설정

  1. __계정 생성 & 세팅__  
    실제 서비스에 접근 하기 위한 계정을 생성해야한다. (콘솔에 로그인 한 계정은 Admin 계정)
    ![](/assets/images/keycloak-create-user01.png)  
    *유저 관리 탭*  
    ![계정생성](/assets/images/keycloak-create-user02.png)  
    *계정 정보 입력*  
    ![비밀번호 설정](/assets/images/keycloak-create-user03.png)  
    *비밀번호 설정*  
  2. __클라이언트 세팅__  
    어플리케이션에서 접근할 클라이언트를 세팅해 주어야 한다.  
    ![](/assets/images/keycloak-create-client01.png)  
    ![](/assets/images/keycloak-create-client02.png)  
    ![](/assets/images/keycloak-create-client03.png)  
    *Access type 변경*  
    ![](/assets/images/keycloak-create-client04.png)  
    *Credentials 탭의 Secret 생성 확인*  
  3. __Realm 세팅__  
    Access Token의 Life Cycle 세팅을 해 준다(테스트 편의성)  
    ![](/assets/images/keycloak-create-client05.png)  
    *하단의 save 버튼 누르는것을 잊지 말자*

### Step 3. 로그인 테스트  
  지금까지 설정한 클라이언트 정보, 유저 정보를 통해 Access Token을 발급하는 테스트를 진행해보자.
  ```shell
  curl --location 'http://localhost:9000/auth/realms/master/protocol/openid-connect/token' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'client_id=jaeho-client' \
  --data-urlencode 'client_secret=sCMQZdkQO2SJMmjRk5zW22y3EMFmO5j9' \
  --data-urlencode 'username=jaeho.choi' \
  --data-urlencode 'password=1234' \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'scope=openid'
  ```

  Response
  ```json
  {
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJ4UDFyUERZQUgtaHJOVmVVd2NjSDBGN3RlTmdaNGhCWFRLem9vcE42MVVvIn0.eyJleHAiOjE2ODQ4ODk0ODcsImlhdCI6MTY4NDg4NzY4NywianRpIjoiOTRhMTYwMWYtMTBmOS00ZjkwLTg3NjMtMmUyYzhiOTkwY2RkIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo5MDAwL2F1dGgvcmVhbG1zL21hc3RlciIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiJlNjg0MzU5Ni04NzNkLTQyMGEtYjc1My04MGVhMjgzOTlmMTgiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJqYWVoby1jbGllbnQiLCJzZXNzaW9uX3N0YXRlIjoiOGI2YjdhYjItMmIzNy00YzEwLWFlZjEtMWJjODAzYjUxMThmIiwiYWNyIjoiMSIsImFsbG93ZWQtb3JpZ2lucyI6WyJodHRwOi8vbG9jYWxob3N0OjgwODAiXSwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbImRlZmF1bHQtcm9sZXMtbWFzdGVyIiwib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgZW1haWwiLCJzaWQiOiI4YjZiN2FiMi0yYjM3LTRjMTAtYWVmMS0xYmM4MDNiNTExOGYiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsIm5hbWUiOiJKYWVobyBDaG9pIiwicHJlZmVycmVkX3VzZXJuYW1lIjoiamFlaG8uY2hvaSIsImdpdmVuX25hbWUiOiJKYWVobyIsImZhbWlseV9uYW1lIjoiQ2hvaSIsImVtYWlsIjoianVseXNldmVuMTk5NUBnbWFpbC5jb20ifQ.TwkMwHLMJSXhg1yyRU9_dPu5EiOQA5wqOf_Kjgxja541mXow1KImjaNS7K7F89CnWWtkaZqUbDIE9lc0Auyunavhg_4vw615Xb1yjmn8c3DARj6zrIjgGOrHvU_LDg2irKfeH8uTS7lzqhgV01--6ehBT_OAJBjFAXnQ0TEGxcg1fAyIO032u6gKJpUmZlqyR2z0m2OvKCgjRCMe9cUCgKOyUL7jgFBrWQc5SXzgroE5ju7u0MFlcpVCw4xf6uBWb9sFLm6k04qgIRZ8ycW7eT9K8YXHbeBDMt5MotssQzACXRgD8loLwwihgf22goj5aOOLHiRkDn6l4J5nMYE3mg",
    "expires_in": 1800,
    "refresh_expires_in": 3600,
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICI4ZjMzMjJkOC1kMTMyLTRmOWYtYTJkOS00NDU3YjI0MjliNjAifQ.eyJleHAiOjE2ODQ4OTEyODcsImlhdCI6MTY4NDg4NzY4NywianRpIjoiNjY1ZTFhZWUtZWQ5MS00ZThiLTk2ODMtNzE4YmM3ZmEzNjBhIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo5MDAwL2F1dGgvcmVhbG1zL21hc3RlciIsImF1ZCI6Imh0dHA6Ly9sb2NhbGhvc3Q6OTAwMC9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJzdWIiOiJlNjg0MzU5Ni04NzNkLTQyMGEtYjc1My04MGVhMjgzOTlmMTgiLCJ0eXAiOiJSZWZyZXNoIiwiYXpwIjoiamFlaG8tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6IjhiNmI3YWIyLTJiMzctNGMxMC1hZWYxLTFiYzgwM2I1MTE4ZiIsInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgZW1haWwiLCJzaWQiOiI4YjZiN2FiMi0yYjM3LTRjMTAtYWVmMS0xYmM4MDNiNTExOGYifQ.Zt3X13xaP-40tHpNhKrTOQIkP8IIzw4HoGnYNwgxm24",
    "token_type": "Bearer",
    "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJ4UDFyUERZQUgtaHJOVmVVd2NjSDBGN3RlTmdaNGhCWFRLem9vcE42MVVvIn0.eyJleHAiOjE2ODQ4ODk0ODcsImlhdCI6MTY4NDg4NzY4NywiYXV0aF90aW1lIjowLCJqdGkiOiJiNTYzZTYzMC1lOWM5LTRkMTgtYWQxYi1iMjZlNjA0ZGI5YzkiLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjkwMDAvYXV0aC9yZWFsbXMvbWFzdGVyIiwiYXVkIjoiamFlaG8tY2xpZW50Iiwic3ViIjoiZTY4NDM1OTYtODczZC00MjBhLWI3NTMtODBlYTI4Mzk5ZjE4IiwidHlwIjoiSUQiLCJhenAiOiJqYWVoby1jbGllbnQiLCJzZXNzaW9uX3N0YXRlIjoiOGI2YjdhYjItMmIzNy00YzEwLWFlZjEtMWJjODAzYjUxMThmIiwiYXRfaGFzaCI6InlhMS1CdU5HSHE1T0VfdlQyeC1kNFEiLCJhY3IiOiIxIiwic2lkIjoiOGI2YjdhYjItMmIzNy00YzEwLWFlZjEtMWJjODAzYjUxMThmIiwiZW1haWxfdmVyaWZpZWQiOmZhbHNlLCJuYW1lIjoiSmFlaG8gQ2hvaSIsInByZWZlcnJlZF91c2VybmFtZSI6ImphZWhvLmNob2kiLCJnaXZlbl9uYW1lIjoiSmFlaG8iLCJmYW1pbHlfbmFtZSI6IkNob2kiLCJlbWFpbCI6Imp1bHlzZXZlbjE5OTVAZ21haWwuY29tIn0.WtfpE3TUT8Y3sSYfI6GKKWxpfVNI7OdmjwAraPmaXmBPHEkt5nCbXwtQsg6YKotNLLHJdzOg_GtKPzg67tuR8eJDBbf2ei_UvnSTcWVMY3okI_X2pwGlPH_JS6rXH-ADPUKIxVvhgKrXbNdksLaO99mnyQJgKrB8foISgHDrtl9Y2Mdp6yBLPewQKt3ekmpcQYKTRuDNICp7vSUi_h72LA08ySh5_82rHVKTSp6NDrLRl1D569vU2G8ekm1XuiF_jh9bsV7cQBT9NtVL0XE28z2h30nEmv3a-U-nwMd0G70xzZC9SWaPRPZo5BJWAS11dQu10bZG1zfzvoGYBzfXBA",
    "not-before-policy": 0,
    "session_state": "8b6b7ab2-2b37-4c10-aef1-1bc803b5118f",
    "scope": "openid profile email"
}
```
JWT 토큰을 발급받을 수 있는것을 확인 할 수 있다.

## Spring Boot + Keycloak
이제 Keycloak의 기본적인 세팅도 마쳤고, Access Token 발급도 해냈으니, Spring Boot Application과 연동해 보도록 하겠다.  
1. __프로젝트 세팅__  
  [Spring initalizr](https://start.spring.io/)에서 Spring Boot 프로젝트 생성을 해주면 된다.
  ![](/assets/images/keycloak-spring-init-project01.png)  
  *추가 하고 싶은 의존성이 있다면 추가해도 무방하다.*  

    프로젝트를 열었다면, application.yml을 다음과 같이 세팅해준다.
    ```yml
    spring:
      application:
        name: jaeho-project
      security:
        oauth2:
          client:
            provider:
              oidc:
                issuer-uri: 'http://localhost:9000/auth/realms/master'
            registration:
              oidc:
                client-id: jaeho-client
                client-secret: sCMQZdkQO2SJMmjRk5zW22y3EMFmO5j9
                scope:
                  - openid
                  - profile
                  - email
    ```

2. __Jwt Validator 클래스 생성__  
  Spring Security 설정에 요청받은 토큰을 인증하기 위한 Custom Validator를 만들어준다.
    ```java
    @Slf4j
    public class JwtTokenValidator implements OAuth2TokenValidator<Jwt> {

        private final String userInfoEndpoint;

        private final OAuth2Error error;
        
        private final RestTemplate restTemplate = new RestTemplate();
        
        public JwtTokenValidator(String userInfoEndpoint) {
            
            this.userInfoEndpoint = userInfoEndpoint;
            this.error = new OAuth2Error("invalid_token", "User information fetching failed", userInfoEndpoint);
        }

        @Override
        public OAuth2TokenValidatorResult validate(Jwt token) {

            HttpHeaders headers = new HttpHeaders();

            headers.set("Authorization", "Bearer " + token.getTokenValue());

            try {

                // 유저 정보 Fetching 을 통해 token 유효성 체크
                restTemplate.exchange(
                    userInfoEndpoint,
                    HttpMethod.GET,
                    new HttpEntity<String>(headers),
                    Object.class
                );

                // Context holder에 요청받은 토큰으로 Authentication을 만들어 넣어준다.
                SecurityContextHolder.getContext().setAuthentication(new JwtAuthenticationToken(token));

                return OAuth2TokenValidatorResult.success();

            } catch (Exception e) {
                log.error("Token Validation Failed. ->", e);
                return OAuth2TokenValidatorResult.failure(error);
            }
        }
    }

    ```
3. __Security Configuration 설정__  
  http 접근 제어를 위한 Security Configuration을 설정해준다.

    ```java
    @EnableWebSecurity
    public class SecurityConfiguration {

        private final ClientRegistration clientRegistration;
        public SecurityConfiguration(ClientRegistrationRepository clientRegistrationRepository) {

            this.clientRegistration = clientRegistrationRepository.findByRegistrationId("oidc");
        }

        @Bean
        public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

            http.csrf()
                .disable()
                .authorizeHttpRequests()
                .mvcMatchers("/api/**").authenticated()
            .and()
                .oauth2ResourceServer()
                .jwt()
                .jwtAuthenticationConverter(this.jwtAuthenticationConverter());

            return http.build();
        }


        @Bean
        public JwtDecoder jwtDecoder() {

            NimbusJwtDecoder jwtDecoder = JwtDecoders.fromIssuerLocation(clientRegistration.getProviderDetails().getIssuerUri());

            jwtDecoder.setJwtValidator(new JwtTokenValidator(clientRegistration.getProviderDetails().getUserInfoEndpoint().getUri()));

            return jwtDecoder;
        }

        private Converter<Jwt, AbstractAuthenticationToken> jwtAuthenticationConverter() {

            JwtAuthenticationConverter converter = new JwtAuthenticationConverter();

            converter.setJwtGrantedAuthoritiesConverter(new JwtGrantedAuthoritiesConverter());

            return converter;
        }

    }
    ```

  4. __Endpoint 생성__  
    토큰의 유무에 따라 잘 요청을 잘 거르는지 확인하기 위해 간단한 엔드포인트를 구현해보자.
      ```java
      @RestController
      @RequestMapping("/api")
      public class JaehoController {

          @GetMapping(path = "/hello")
          public ResponseEntity<String> hello() {

              return ResponseEntity.ok("Hello!");
          }
      }
      ```
  5. __테스트__  
    모든 세팅이 끝났으니 프로젝트를 실행시키고, 토큰 없이 hello api를 호출해보자.
      ```shell
      curl -L -k -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/api/hello
      ```
      권한이 없어 401이 응답값으로 나타나는것을 알 수 있다.
      그렇다면 Access token을 발급받아서 호출하면 잘 될까

      ```shell
      curl --location 'http://localhost:8080/api/hello' \
      --header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJ4UDFyUERZQUgtaHJOVmVVd2NjSDBGN3RlTmdaNGhCWFRLem9vcE42MVVvIn0.eyJleHAiOjE2ODQ4OTUxODcsImlhdCI6MTY4NDg5MzM4NywianRpIjoiYWNmYjU0ODAtYjczMS00YmFkLTkzYTEtZGY0MTYxMGQyNzVjIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo5MDAwL2F1dGgvcmVhbG1zL21hc3RlciIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiJlNjg0MzU5Ni04NzNkLTQyMGEtYjc1My04MGVhMjgzOTlmMTgiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJqYWVoby1jbGllbnQiLCJzZXNzaW9uX3N0YXRlIjoiNTA3ZTE3YWMtYmM5Mi00M2Q5LTgxNjUtZGVkZjE4MGJkNmZhIiwiYWNyIjoiMSIsImFsbG93ZWQtb3JpZ2lucyI6WyJodHRwOi8vbG9jYWxob3N0OjgwODAiXSwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbImRlZmF1bHQtcm9sZXMtbWFzdGVyIiwib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgZW1haWwiLCJzaWQiOiI1MDdlMTdhYy1iYzkyLTQzZDktODE2NS1kZWRmMTgwYmQ2ZmEiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsIm5hbWUiOiJKYWVobyBDaG9pIiwicHJlZmVycmVkX3VzZXJuYW1lIjoiamFlaG8uY2hvaSIsImdpdmVuX25hbWUiOiJKYWVobyIsImZhbWlseV9uYW1lIjoiQ2hvaSIsImVtYWlsIjoianVseXNldmVuMTk5NUBnbWFpbC5jb20ifQ.PjEHSyGLt95aJhOJM35eJ06vNNngjXe-urdn9c7lYwEhd4882E8zKCwBQe4Cl8xDwnkV04Xlv2NW2uA9uNOQI-HQLusKzqRtfpndc5JefPqh60gL5g1Qvgh3mhWHo2wz2y3r6YPQaJvlP2bADj-zzN7VU8vUARlVgfm4JvLpO27O_UzuF72NMt98ICT6XbHGdQqbCBunwwXwa0X-NQNBZTOxvh4mVXwAcmVJ0aTrT-v5XnCNZcc_JYKyVYLYiQfEkyVgStdpetCGyDMkO03I_yPETEs7-9bQJ3AJEQgWqiUVmtVE6EWIMWNlxm_EBhSryS12Bjq9v4znHYg2gkrqZg'
      ```
      정상적으로 잘 살아있는 토큰이라면, Hello라는 Response Body를 확인 할 수 있다!


