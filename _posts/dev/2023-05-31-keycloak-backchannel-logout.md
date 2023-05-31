---
title: "Keycloak 토큰을 효율적으로 관리해보기 #2"
categories:
  - Dev
tags:
  - Keycloak
  - OIDC
  - Spring Boot
layout: single
comments: true
---

[Keycloak 토큰을 효율적으로 관리해보기 #1](https://julyseven1995.github.io/dev/oauth2-token-warder) 포스팅과 이어짐

## BackChannel Logout
Backchannel Logout은 인증된 사용자가 로그아웃할 때 실시간으로 관련된 클라이언트들에게  
로그아웃 이벤트를 전달하는 메커니즘이다. Backchannel Logout은 OIDC 프로토콜의 일부로 정의되어 있다.  
다음과 같은 시나리오에 유용하다.

사용자가 로그아웃하면, 해당 사용자의 세션이 유효하지 않게 되므로 다른 클라이언트에서도 사용자 세션을 적절히 처리한다.  
Backchannel Logout을 통해 다른 클라이언트에게 사용자 세션을 무효화하도록 지시한다.

위의 포스팅에서 세션이 종료된 상태에서 토큰이 캐시(Redis)에 남겨져 있을 때 캐시를 지우게 하게끔 명령을 보내는 방식으로 사용 할 수 있다.

### 설정
BackChannel Logout을 설정하는 방법은 다음과 같다.  
  1. __엔드포인트 생성__

      Client와 연결되는 Application에서 BackChannel Logout 요청을 받기 위한 엔드포인트를 뚫어주어야 한다.  
      ```java
      @RestController
      @RequestMapping("/sso")
      @RequiredArgsConstructor
      public class LogoutResource {

          private final TokenRepository tokenRepository;

          @PostMapping(path = "/logout", consumes = {MediaType.APPLICATION_FORM_URLENCODED_VALUE})
          @ResponseStatus(HttpStatus.OK)
          public void logout(@RequestParam("logout_token") String logoutToken) throws ParseException {

              // 토큰 확인용 출력
              System.out.println(logoutToken);

              String sid = String.valueOf(JWTParser.parse(logoutToken).getJWTClaimsSet().getClaim("sid"));
              tokenRepository.delete(sid);
          }
      }
      ```
  2. __Keycloak 설정__

      Keycloak Admin Console에서 설정 할 Client를 선택해 엔드포인트를 바인딩 해 주어야 한다.  
      어플리케이션에서 생성한 엔드포인트를 어플리케이션의 주소와 함께 바인딩 해준다.  
      ![](/assets/images/keycloak-backchannel-logout1.png)

  3. __Backchannel Logout 동작 확인__

      1. 액세스 토큰 발급을 통해 로그인 한 세션을 확인한다
          
          액세스 토큰 발급
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
          발급 결과 확인
          ```json
          {
            "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJpcnJKZmdDUGlLOXZpcUZOQzJoNVJYcFduak9Ba0NBNEJsYjlYY1FiX0djIn0.eyJleHAiOjE2ODU0OTY4NzMsImlhdCI6MTY4NTQ5NTA3MywianRpIjoiZDA3YmViZmQtZTcwNy00MjJlLWIxODUtNWEzMDA0OWJkZjgzIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo5MDAwL2F1dGgvcmVhbG1zL21hc3RlciIsImF1ZCI6WyJqYWVoby1yZWFsbSIsIm1hc3Rlci1yZWFsbSIsImFjY291bnQiXSwic3ViIjoiY2NmZmQ3MjktYTg4NS00NTZhLTkwYWYtZDdjOWUyMTlkY2M4IiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiamFlaG8tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6IjU5MjA0M2M1LWQzN2YtNGQwMi04NmQxLTllMDJhMmIyNTRjYSIsImFjciI6IjEiLCJhbGxvd2VkLW9yaWdpbnMiOlsiKiJdLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsiY3JlYXRlLXJlYWxtIiwiZGVmYXVsdC1yb2xlcy1tYXN0ZXIiLCJvZmZsaW5lX2FjY2VzcyIsImFkbWluIiwidW1hX2F1dGhvcml6YXRpb24iXX0sInJlc291cmNlX2FjY2VzcyI6eyJqYWVoby1yZWFsbSI6eyJyb2xlcyI6WyJ2aWV3LWlkZW50aXR5LXByb3ZpZGVycyIsInZpZXctcmVhbG0iLCJtYW5hZ2UtaWRlbnRpdHktcHJvdmlkZXJzIiwiaW1wZXJzb25hdGlvbiIsImNyZWF0ZS1jbGllbnQiLCJtYW5hZ2UtdXNlcnMiLCJxdWVyeS1yZWFsbXMiLCJ2aWV3LWF1dGhvcml6YXRpb24iLCJxdWVyeS1jbGllbnRzIiwicXVlcnktdXNlcnMiLCJtYW5hZ2UtZXZlbnRzIiwibWFuYWdlLXJlYWxtIiwidmlldy1ldmVudHMiLCJ2aWV3LXVzZXJzIiwidmlldy1jbGllbnRzIiwibWFuYWdlLWF1dGhvcml6YXRpb24iLCJtYW5hZ2UtY2xpZW50cyIsInF1ZXJ5LWdyb3VwcyJdfSwibWFzdGVyLXJlYWxtIjp7InJvbGVzIjpbInZpZXctcmVhbG0iLCJ2aWV3LWlkZW50aXR5LXByb3ZpZGVycyIsIm1hbmFnZS1pZGVudGl0eS1wcm92aWRlcnMiLCJpbXBlcnNvbmF0aW9uIiwiY3JlYXRlLWNsaWVudCIsIm1hbmFnZS11c2VycyIsInF1ZXJ5LXJlYWxtcyIsInZpZXctYXV0aG9yaXphdGlvbiIsInF1ZXJ5LWNsaWVudHMiLCJxdWVyeS11c2VycyIsIm1hbmFnZS1ldmVudHMiLCJtYW5hZ2UtcmVhbG0iLCJ2aWV3LWV2ZW50cyIsInZpZXctdXNlcnMiLCJ2aWV3LWNsaWVudHMiLCJtYW5hZ2UtYXV0aG9yaXphdGlvbiIsIm1hbmFnZS1jbGllbnRzIiwicXVlcnktZ3JvdXBzIl19LCJhY2NvdW50Ijp7InJvbGVzIjpbIm1hbmFnZS1hY2NvdW50IiwibWFuYWdlLWFjY291bnQtbGlua3MiLCJ2aWV3LXByb2ZpbGUiXX19LCJzY29wZSI6Im9wZW5pZCBwcm9maWxlIGVtYWlsIiwic2lkIjoiNTkyMDQzYzUtZDM3Zi00ZDAyLTg2ZDEtOWUwMmEyYjI1NGNhIiwiZW1haWxfdmVyaWZpZWQiOmZhbHNlLCJuYW1lIjoiSmFlaG8gQ2hvaSIsInByZWZlcnJlZF91c2VybmFtZSI6ImphZWhvLmNob2kiLCJnaXZlbl9uYW1lIjoiSmFlaG8iLCJmYW1pbHlfbmFtZSI6IkNob2kiLCJlbWFpbCI6Imp1bHlzZXZlbjE5OTVAZ21haWwuY29tIn0.pe507VUKFWXWVF4DhvJBlURcMnjSA6zOc6a5PJWYXuJww-AM9Df1PqY6nDXPoG81NLVcnPuXa5EJpnesfU0WaI3jgXHq3-jXLWv7__LaBWjAk2iA5cfJyh8ALKnA1UxEnFtxhb8thOtAGkd75vBHnKS_YvCoJCOlu4pvIFrs0e1NVNqLwCN79Xx4JDLrl3jKNukSEbG2kShz-vO_UpHpKXwjWf_UlT78v-hjSKcano76gDtTaJ68wGOIM4kIT0ED8Ib8NjUNiJvME3nVgqQTLAtfe3HWvFu2B1c8a2UKh-N_QDkN4Bbvuefi0EF3DKasvoqN5tBmjVRZ3_Wux086xQ",
            "expires_in": 1800,
            "refresh_expires_in": 3600,
            "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJhMzI0YWUwNy03MjVjLTQwZDYtODZkYS1hYjYyYWEwZWE1MjcifQ.eyJleHAiOjE2ODU0OTg2NzMsImlhdCI6MTY4NTQ5NTA3MywianRpIjoiMzM4ZjZiOGEtMmQ2NC00NGQ2LWI5NWEtMjFmNjAxYWNlODE1IiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo5MDAwL2F1dGgvcmVhbG1zL21hc3RlciIsImF1ZCI6Imh0dHA6Ly9sb2NhbGhvc3Q6OTAwMC9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJzdWIiOiJjY2ZmZDcyOS1hODg1LTQ1NmEtOTBhZi1kN2M5ZTIxOWRjYzgiLCJ0eXAiOiJSZWZyZXNoIiwiYXpwIjoiamFlaG8tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6IjU5MjA0M2M1LWQzN2YtNGQwMi04NmQxLTllMDJhMmIyNTRjYSIsInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgZW1haWwiLCJzaWQiOiI1OTIwNDNjNS1kMzdmLTRkMDItODZkMS05ZTAyYTJiMjU0Y2EifQ.PviN8GoWR7RxemfPg2wRM14maf3uq10mNZvKjgUOCQA",
            "token_type": "Bearer",
            ...
          }
          ```

          [jwt.io](https://jwt.io) 사이트에서 액세스 토큰 컨버팅하여 sid 확인
          ```json
          {
            ...
            "scope": "openid profile email",
            "sid": "592043c5-d37f-4d02-86d1-9e02a2b254ca",
            "email_verified": false,
            "name": "Jaeho Choi",
            "preferred_username": "jaeho.choi",
            "given_name": "Jaeho",
            "family_name": "Choi",
            "email": "julyseven1995@gmail.com"
          }
          ```

      2. Admin Console을 통해 세션 종료
          ![](/assets/images/keycloak-backchannel-logout2.png)
          로그아웃 버튼을 통해 세션을 종료시킨다.
      3. Application Console 확인
          ```shell
          2023-05-31 10:12:11.335 DEBUG 5938 --- [  XNIO-1 task-1] o.s.security.web.FilterChainProxy        : Securing POST /sso/logout
          2023-05-31 10:12:11.336 DEBUG 5938 --- [  XNIO-1 task-1] s.s.w.c.SecurityContextPersistenceFilter : Set SecurityContextHolder to empty SecurityContext
          2023-05-31 10:12:11.337 DEBUG 5938 --- [  XNIO-1 task-1] o.s.s.w.a.AnonymousAuthenticationFilter  : Set SecurityContextHolder to anonymous SecurityContext
          2023-05-31 10:12:11.339 DEBUG 5938 --- [  XNIO-1 task-1] o.s.security.web.FilterChainProxy        : Secured POST /sso/logout
          eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJpcnJKZmdDUGlLOXZpcUZOQzJoNVJYcFduak9Ba0NBNEJsYjlYY1FiX0djIn0.eyJpYXQiOjE2ODU0OTU1MzEsImp0aSI6IjgxZjljZGU3LWExYmItNDA5Mi05MmVkLTI3OWExNzE3ODExMCIsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6OTAwMC9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJhdWQiOiJqYWVoby1jbGllbnQiLCJzdWIiOiJjY2ZmZDcyOS1hODg1LTQ1NmEtOTBhZi1kN2M5ZTIxOWRjYzgiLCJ0eXAiOiJMb2dvdXQiLCJzaWQiOiI1OTIwNDNjNS1kMzdmLTRkMDItODZkMS05ZTAyYTJiMjU0Y2EiLCJldmVudHMiOnsiaHR0cDovL3NjaGVtYXMub3BlbmlkLm5ldC9ldmVudC9iYWNrY2hhbm5lbC1sb2dvdXQiOnt9fX0.k8Gh1O9ieMEnoHe9Vw638v77S4PeRL8vRFtvrqkmp-PHL0EWDKWJ1UCrMJbvKeiktphk5uVldmdSw-PIM9uNmhvGavSbi64pJCimLx57zaVBujzaYWFEKs7Gj3KeCs7fGXxHexmPKxTH9p4k0fr8otd8UO2A5tVv4_mhF6QEjIIwJu0yQIRV_QK5T6FTba6pNGcU4Lhal-1VnvhE20oIS84PXAiDDx9Wc7_N__j_YT2TUk3CZ8FjRCufYCVDpwg1OaMWE92YQYqHJ_wI_fpf4PWjjfauam8e8JORDGnAgEbY_MV3BykEOnfK7ohGXqVPi8PX4nosNAPRPNRdYnUhHg
          2023-05-31 10:12:11.365 DEBUG 5938 --- [  XNIO-1 task-1] w.c.HttpSessionSecurityContextRepository : Did not store anonymous SecurityContext
          2023-05-31 10:12:11.366 DEBUG 5938 --- [  XNIO-1 task-1] s.s.w.c.SecurityContextPersistenceFilter : Cleared SecurityContextHolder to complete request
          ```
          로그아웃 토큰을 컨버팅하면 다음과 같은 결과를 확인 할 수 있다.
          ```json
          {
            "iat": 1685495531,
            "jti": "81f9cde7-a1bb-4092-92ed-279a17178110",
            "iss": "http://localhost:9000/auth/realms/master",
            "aud": "jaeho-client",
            "sub": "ccffd729-a885-456a-90af-d7c9e219dcc8",
            "typ": "Logout",
            "sid": "592043c5-d37f-4d02-86d1-9e02a2b254ca",
            "events": {
              "http://schemas.openid.net/event/backchannel-logout": {}
            }
          }
          ```
          동일한 SID를 갖고 있으므로 토큰을 SID를 통해 식별 할 수 있음을 확인 할 수 있다.  
          ps. SID는 토큰을 Refresh해도 변경되지 않는다.

### _Conclusion_
OAuth2 호출 부하를 막기 위해 여러 시도를 해 보았다. 시나리오는 다음과 같다.  
<div class="mermaid">
sequenceDiagram
  actor user as user
  user->>Keycloak: 로그아웃 요청
  Keycloak->>Service: BackChannel Logout 요청
  Service->>Cache(Redis): 토큰(세션)) 삭제
</div>
