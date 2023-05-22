---
title: "다운로드 대기열 기능 적용기"
categories:
  - Dev
tags:
  - WebFlux
  - Kotlin
  - SSE
layout: single
comments: true
---

작년에 회사에서 Kotlin + WebFlux를 사용해 다운로드 서비스를 하나 만들어 놓은게 있는데,  
일정상의 문제로 미완성 된 상태로 운영 서버에 올라간 녀석이 하나 있었다.  
대용량 파일 다운로드 서비스라고는 거창한 이름이긴 하지만, 다운로드 대기열 기능조차 없어서 여러 클라이언트가 붙어있을 경우 끔찍한 결과를 초래한다. (Network 트래픽 점유율 문제 등)  
최근 회사에서 이걸 수정할 수 있는 여력이 생겨서 문제점이 무엇이었고 어떤 방향으로 개선했고, 개선하면서 어떤 문제와 마주쳤는지 기록해보고자 한다.


### As-Is

문제점

1. 시스템적 문제

    Client 레벨에 굉장히 의존적임
      * Client에서 토큰을 발급 받고, 다운로드 요청을 하지 않는다면 Redis에 저장된 토큰이 만료되기 전까지 남아있음.
2. 파일 다운로드 전송 속도의 한계
    
    현재 파일 다운로드 서비스는 운영되고있는 어플리케이션과 같은 서버에 위치함. 
    서버가 같으므로 네트워크 트래픽 또한 공유. 
    * 여러 사용자가 파일 다운로드 시도를 할 경우 병목현상을 초래할 수 있음.
    
    
3. 파일 다운로드 우선순위에 대한 문제

    병목현상이 일어나지만, 먼저 다운로드 요청을 한 사람이 먼저 다운 받을 수 있는 구조가 아님. 클라이언트 A, B, C가 있다고 가정함
      1. A가 파일 다운로드를 요청하며 다운로드를 받으면서 100%의 전송속도를 할당 받음.
      2. B 또한 파일 다운로드 기능을 통해 파일을 다운로드 받으면 A는 50%의 전송속도, B는 50%의 전송속도를 할당받음.
      3. 그 후 C가 다운로드를 받게되면 각각 33.3%의 전송 속도를 유지하다, A의 다운로드가 끝나면 B와 C가 50%의 속도로 다운로드를 받게 됨.
      
        내려 줄 수 있는 속도와 용량은 한정적인데, 클라이언트가 요청할 수록 먼저 다운로드를 받고 있는 클라이언트에게 불이익이 생김

시나리오

* 토큰 발급
  <div class="mermaid"> 
    sequenceDiagram
      Client ->> Server: 다운로드 토큰 발급 요청
      Server ->> Database: 파일 정보 조회
      Database -->> Server: 파일 정보 반환
      Server ->> Server: 토큰 생성
      Server ->> Redis: 토큰 저장
      Server -->> Client: 토큰 반환
  </div>
* 파일 다운로드
  <div class="mermaid"> 
    sequenceDiagram
      Client ->> Server: 파일 다운로드 요청(토큰)
      Server ->> Redis: 토큰 조회
      Redis -->> Server: 토큰 정보 반환
      Server -->> Client: 파일 전송 Stream
      Server ->> Redis: 토큰 삭제
  </div>



엔드포인트도 단 2개밖에 존재하지 않는다. 토큰 발급 그리고 토큰을 통한 파일 다운로드.

```kotlin
  @GetMapping(
    path = ["/request/download/{requestToken}"], 
    produces = [MediaType.APPLICATION_OCTET_STREAM_VALUE]
  )
  fun download(@PathVariable requestToken: String) // return Flux<ByteBuffer!>
    = fileDownloadService.fileDownloadByRequestToken(requestToken)

  @PostMapping(path = ["/request/generate/{fileId}"])
  fun generate(@PathVariable fileId: Long) // return Mono<String!>
    = fileDownloadService.generateFileDownloadRequestToken(fileId)
```

### To-Be

개선안
  1. 시스템 문제 개선
    
      Client에 의존적인 엔드포인트 개선(발급 당시에 Redis 저장을 제거)
  2. 다운로드 대기열 추가

      다운로드 받을 수 있는 요청의 개수를 정해놓고, 다운로드 요청이 최대치라면, 그 후 요청한 클라이언트를 대기열에 넣어, 들어온 순서대로 처리해 주는 방식.

적용하기 위한 기술
  * SSE(Server Sent Event)
    
    Server -> Client 단방향 통신 기술. Server의 업데이트 정보를 스트리밍으로 내려줄 수 있음

    * SSE vs Websocket

      * 통신방식
        * SSE: 단방향 통신(Server -> Client)
        * Websocket: 양방향 통신

      * 프로토콜
        * SSE: 일반 HTTP 프로토콜
        * Websocket: HTTP를 기반으로 한 프로토콜(연결을 HTTP로 연결하고, 연결이 완료되면 Websocket 프로토콜로 전환됨)

      -> SSE는 단방향 통신을 위한 간단한 연결 시스템, Websocket은 양방향 통신을 위한 프로토콜.

    -> Client에게 지속적으로 대기열 번호를 내려주기위해 사용됨


  시나리오

  * 토큰 발급 
  <div class="mermaid"> 
    sequenceDiagram
      Client ->> Server: 다운로드 토큰 발급 요청
      Server ->> Database: 파일 정보 조회
      Database -->> Server: 파일 정보 반환
      Server ->> Server: 토큰 생성
      Server -->> Client: 토큰 반환
  </div>

  * 다운로드 대기열 연결
  <div class="mermaid"> 
  sequenceDiagram
    Client ->> Server: 다운로드 대기열 등록 요청(토큰)
    Server ->> Redis: 다운로드 대기열에 등록(토큰 enqueue)
    loop 대기열 순번 대기
      Server ->> Redis: 토큰의 대기 번호 검색
      Redis -->> Server: 토큰의 대기 번호 반환
      alt is 토큰 대기 필요
        Server -->> Client: 대기번호 반환
      else is 다운로드 가능
        Server -->> Client: 대기 종료
        Server ->> Redis: 토큰 dequeue
        Server ->> Redis: 다운로드 가능 토큰 등록
      end
    end 
    Server ->> Client: stream 종료
  </div>

  * 파일 다운로드
  <div class="mermaid"> 
    sequenceDiagram
      Client ->> Server: 다운로드 요청(토큰)
      Client ->> Server: 다운로드 요청
      loop 파일 다운로드
        Server -->> Client: 파일 전송
        alt is EOF
          Server ->> Redis: 다운로드 토큰 제거
        end
      end
      Server -->> Client: 파일 다운로드 완료
  </div>
Server 코드

1. 엔드포인트
    ```kotlin

      @GetMapping(
        path = ["/request/download/{requestToken}"], 
        produces = [MediaType.APPLICATION_OCTET_STREAM_VALUE]
      )
      fun download(@PathVariable requestToken: String) // return Flux<ByteBuffer!>
        = fileDownloadService.download(requestToken)

      // 다운로드 대기열 등록
      @GetMapping(
        path = ["/request/order/{requestToken}"], 
        produces = [MediaType.TEXT_EVENT_STREAM_VALUE]
      )
      fun enqueue(@PathVariable requestToken: String) // return Flux<Long!>
        = fileDownloadService.subscribeDownloadQueue(requestToken)

      @GetMapping(path = ["/request/generate/{fileId}"])
      fun generate(@PathVariable fileId: Long) // return Mono<String!>
        = fileDownloadService.generateFileDownloadRequestToken(fileId)
    ```

2. Service
    ```kotlin
      fun subscribeDownloadQueue(requestToken: String): Flux<Long?> {

          // 다운로드 대기열에 있는 동안 신청 받지 않음
          if (!downloadQueueService.isExists(requestToken)) {
              downloadQueueService.enqueue(requestToken)
          }

          // flux interval 기능을 통해 1초마다 다운로드 토큰 큐 조회
          return Flux.interval(Duration.ofSeconds(1))
              .mapNotNull {
                  // 다운로드 가능 여부 체크
                  val orderPosition = downloadQueueService.getPosition(requestToken)
                  val downloadingTokens = fileDownloadRequestRepository.getSize()
                  return@mapNotNull (orderPosition + downloadingTokens)
              }
              .takeUntil { it == 0L }
              .doOnCancel { downloadQueueService.drop(requestToken) } // 연결 취소시 대기열에서 제거
              .doOnError { downloadQueueService.drop(requestToken) } // 연결 에러시 대기열에서 제거
              .doOnComplete { fileDownloadRequestRepository.put(downloadQueueService.dequeue()) }
      }
    ```
  
Client 코드

클라이언트는 `TypeScript`로 구성되어있음.
  ```typescript
    const eventSource = new EventSource(대기열_API_URL + 토큰);
    eventSource.onmessage = event => {
      if (event.data === '0') {
        eventSource.close()
        // TODO : 파일 다운로드 API 호출
      }
    };

    eventSource.onerror = error => {
      console.log(error);
    };
  ```