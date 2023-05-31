---
title: "Keycloak 토큰을 효율적으로 관리해보기 #1"
categories:
  - Dev
tags:
  - Keycloak
  - OIDC
  - Spring Boot
layout: single
comments: true
---

[Spring Boot + Keycloak](https://julyseven1995.github.io/dev/spring-boot-keycloak) 포스팅과 이어짐

이전 포스팅에서 Spring Boot + Keycloak을 통해 OIDC 연결을 성공적으로 마쳐보았다.  
해당 구조는 Client에서 엔드포인트에 접근 할 때, Bearer에 담겨진 토큰으로 Keycloak에 UserEndpoint를 호출하여  
성공적으로 Fetching이 되었다면 살아있는 세션의 토큰, 실패했다면 죽은 토큰으로 간주하는 방식이다.  
하지만 여기서 크나큰 문제가 있는데, Client가 엔드포인트에 접근 할 때 마다 전부 Fetching한다는 것이다.

<div class="mermaid">
sequenceDiagram
  actor A as client
  A->>Server: 액세스 토큰과 함께 Endpoint Call
  Server->>Keycloak: 토큰 검증
  Keycloak-->>Server: 검증 완료
  Server->>Server: 작업 수행
  Server-->>A: 응답
</div>

> 모든 API 콜에 대해 OAuth2 서버에 인증 요청을 보낸다.

클라이언트가 늘어날수록 부담이 클 것 같은 구조라서 시스템적으로 개선이 필요할 것 같다는 생각이 들었다.  


### `Redis` 활용

토큰을 어디다 저장 해 놓고 있으면 되지 않을까? 란 생각이 들어서 생각해낸 방법이다.  

우선 `Redis`를 사용하기 위해 다음과 같은 세팅을 해주자.

1. `Redis` 설치(Docker)  
    1. Image Pull  
      `docker pull redis` 명령어로 redis image를 받는다.
        ```shell
        jaeho@mac ~ % docker pull redis
        Using default tag: latest
        latest: Pulling from library/redis
        d981f2c20c93: Pull complete
        5b8f51f5c4bb: Pull complete
        2d3d8bc9388e: Pull complete
        849e9b1b24f1: Pull complete
        6e2590bc72d8: Pull complete
        bdd261b6469d: Pull complete
        Digest: sha256:f9724694a0b97288d2255ff2b69642dfba7f34c8e41aaf0a59d33d10d8a42687
        Status: Downloaded newer image for redis:latest
        docker.io/library/redis:latest
        ```
    2. Redis 실행  
      `docker run -p 6379:6379 --name jaeho-redis -d redis` 명령어로 redis를 실행시킨다.

        ```shell
        jaeho@mac ~ % docker run -p 6379:6379 --name jaeho-redis -d redis
        690eceb9e7a640ddbaf7904ca603c205b834459f287c3037b15c55a3511dd8bc
        ```
2. Project Dpendency 설정  
    maven
    ```xml    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    ```
    gradle
    ```kotlin
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    ```
3. Redis Config 설정 및 Repository 생성  
  application.yml
    ```yaml
    spring: 
      redis:
        host: localhost
        port: 6379
        username: jaeho-redis
    ```
    RedisRepository.java  
    Redis에 value를 저장하고 읽기 위해 생성
    ```java
    @Repository
    public class RedisRepository {

        private final ValueOperations<String, String> operations;

        public RedisRepository(RedisTemplate<String, String> redisTemplate) {

            this.operations = redisTemplate.opsForValue();
        }


        public String set(String key, String value, Long expires) {

            this.operations.set(key, value, Duration.ofSeconds(expires));
            return value;
        }

        public Optional<String> get(String key) {

            return Optional.ofNullable(this.operations.get(key));
        }

    }
    ```
    TokenRepository.java  
    Token 저장소가 Redis가 아닌 다른 저장소가 될 수도 있기에 `interface`로 추상화
    ```java
    public interface TokenRepository {
    
        Optional<String> get(String key);

        void set(String key, String value, Long expires);
    }
    ```
    RedisTokenRepository.java  
    Token 저장소 구현체
    ```java
    @Repository
    @RequiredArgsConstructor
    public class RedisTokenRepository implements TokenRepository {
        
        private final String namespace = "token:";

        private final RedisRepository redisRepository;

        public Optional<String> get(String key) {

            return this.redisRepository.get(namespace + key);
        }

        public void set(String key, String value, Long expires) {

            this.redisRepository.set(namespace + key, value, expires);
        }
    }
    ```
4. OAuth2TokenValidator 수정

    ```java
    // 생성자에 tokenRepository 추가
    private final TokenRepository tokenRepository;
    ```
    Redis에 Token이 존재 할 경우 인증이 되었다고 판단하여 Success 처리
    ```java
    @Override
    public OAuth2TokenValidatorResult validate(Jwt token) {

        JwtAuthenticationToken jwtAuthenticationToken = new JwtAuthenticationToken(token);

        // 발급된 토큰은 sid(sessionId)를 유지한다.
        String sid = String.valueOf(token.getClaims().get("sid"));

        Optional<String> existsToken = this.tokenRepository.get(sid);
        if (existsToken.isPresent() && existsToken.get().equals(token.getTokenValue())) {

            log.debug("토큰 발견!");
            SecurityContextHolder.getContext().setAuthentication(jwtAuthenticationToken);
            return OAuth2TokenValidatorResult.success();
        }

        ..
    }
    ```
    Redis에 Token이 존재하지 않을 경우, OAuth2서버에 검증요청을 보낸 후 토큰을 저장 후 Success 처리
    ```java
    @Override
    public OAuth2TokenValidatorResult validate(Jwt token) {

      ...
    
      HttpHeaders headers = new HttpHeaders();

      headers.set("Authorization", "Bearer " + token.getTokenValue());

      // 유저 정보 Fetching 을 통해 token 유효성 체크
      restTemplate.exchange(
          userInfoEndpoint,
          HttpMethod.GET,
          new HttpEntity<String>(headers),
          Object.class
      );

      Long expiresAt = Objects.requireNonNull(token.getExpiresAt()).toEpochMilli();

      Long duration = (expiresAt - Instant.now().toEpochMilli()) / 1000;

      this.tokenRepository.set(sid, token.getTokenValue(), duration);

      log.debug("토큰이 없어서 저장!");

      SecurityContextHolder.getContext().setAuthentication(jwtAuthenticationToken);

      return OAuth2TokenValidatorResult.success();
    }
    ```
  5. 테스트 결과  
    1. Token이 Redis에 저장되지 않았을 때.
        ```shell
        2023-05-26 15:46:37.184 DEBUG 18766 --- [nio-8080-exec-2] c.j.jaeho.oauth2.JwtTokenValidator       : 토큰이 없어서 저장!
        ```
        저장된 토큰 확인
        ![](/assets/images/saved-token.png)  
    2. Token이 Redis에 저장 되었을 때.
      ```shell
        2023-05-26 15:49:24.910 DEBUG 25939 --- [nio-8080-exec-2] c.j.jaeho.oauth2.JwtTokenValidator       : 토큰 발견!
      ```
시나리오는 다음과 같다.

<div class="mermaid">
sequenceDiagram
  actor A as client
  A->>Server: 액세스 토큰과 함께 Endpoint Call
  Server->>Redis: 토큰 유무 확인
  Redis-->>Server: 토큰 유무 응답
  alt is 토큰 없음
    Server->>Keycloak: 토큰 검증
    Keycloak-->>Server: 검증 완료
    Server->>Redis: 토큰 저장(Expire Time 지정)
  end
  Server->>Server: 작업 수행
  Server-->>A: 응답
</div>

## __문제점 발견!!__  
위와 같이 구성해놓으면 1차적으로 Redis를 호출하게 될 것이고,  
`Redis`에 토큰의 유무에 따라 Keycloak에 검증 요청 횟수가 줄어들테니 문제 없겠지 라고 생각을 했지만  
`Redis`에 담겨진 토큰의 세션이 죽었다면(관리자가 임의로 세션을 끊었다면)  
Client는 죽어버린 토큰으로 서버에 액세스 할 수 있게 된다.  
실제로 강제로 세션을 죽여버리고 endpoint에 접근한다면 다음과 같이 동작한다.

1. Session 강제 종료
  ![](/assets/images/keycloak-kill-session.png)  
2. endpoint 호출
    ```shell
    2023-05-26 15:54:38.208 DEBUG 25939 --- [nio-8080-exec-4] c.j.jaeho.oauth2.JwtTokenValidator       : 토큰 발견!
    ```
    이렇게 세션이 죽어도 Redis에 토큰이 저장되있으니 문제없이 엔드포인트에 접근할 수 있다는 치명적인 문제가 발생한다.


다음 포스팅에는 `Keycloak`의 `BackChannel Logout`을 통해 효율적으로 토큰을 관리하는 방법을 포스팅 해보도록 하겠다.
