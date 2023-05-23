---
title: "Lazy Loading의 비동기 작업에 대하여"
categories:
  - Dev
tags:
  - Spring
  - Transactional
  - JPA
layout: single
comments: true
---

무지성으로 서비스 개발을 하던 중, 비동기 작업을 위해 `@Async` 어노테이션을 사용해야 하는 상황이 와서 

해당 어노테이션을 통해 엔티티 객체를 넘기는 코드를 작성하고 테스트를 하는 순간 다음과 같은 익셉션이 발생했다.

```java
@Service
@Transactional
public class MyService {
    
  @Async
  public void doSomething(MyEntity entity) {
    List<Item> items = entity.getOneToManyItems(); // Throws LazyInitializationException
    // ...
  }
}
```

`LazyInitializationException`

왜 그런가 싶어서 열심히 구글링을 통해 찾아본 결과 다음과 같은 이유를 알 수 있었다.

>`LazyInitializationException` 익셉션은 `Hibernate`의 세션이 닫힌 상황에서 `Lazy Loading`을 사용하는 엔티티의 연관 객체를 로드 할 때 발생한다.

-> 비동기 메서드에서 지연 로딩 연관 객체를 호출하면 이 익셉션이 발생되는데, 이미 세션이 닫혀있기 때문이다.

그렇다면 세션을 다시 열어버리면 되는구나! 하고 생각하고 `@Transactional` 어노테이션의 옵션을 통해 다시 한번 시도해 보았다.

```java
@Service
public class MyService {
    
    @Async
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void doSomething(MyEntity entity) {
      List<Item> items = entity.getOneToManyItems(); // Throws LazyInitializationException
      // ...
    }
}
```
세션을 다시 열면 되겠구나 싶어서 새로운 세션을 생성하는 `Propagation.REQUIRES_NEW` 옵션을 사용해보았지만 무용지물이었다. 

이유는 다음과 같았다.


>`Propagation.REQUIRES_NEW`는 새로운 트랜잭션을 시작하고, 현재의 트랜잭션을 일시적으로 중단시킨다. 그러나 비동기 메서드는 별도의 스레드에서 실행되기 때문에, 비동기 메서드 내에서의 트랜잭션 상태와 메인 스레드의 트랜잭션 상태는 독립적이다.

결국 `@Transactional(propagation = Propagation.REQUIRES_NEW)`를 사용하여 새로운 트랜잭션을 시작해도, 비동기 메서드 내에서 로드되는 엔티티는 여전히 원래의 트랜잭션과 연결되어 있지 않은 상태이기 때문에 똑같은 익셉션이 발생한다.

이때부터 오기가 생겨서 이런 저런 시도를 해 보았다.

1. `Hibernate.initialize()`  
  Hibernate의 initalize() 메서드를 사용하면 지연 로딩 된 엔티티나 연관 객체를 초기화 할 수 있다.  
  `LazyInitializationException` 익셉션이 발생 하기 전 강제로 초기화를 시켜 방지할 때 종종 사용한다고 한다.
    ```java
    @Service
    @Transactional
    public class MyService {
        
      @Async
      public void doSomething(MyEntity entity) {
        Hibernate.initalize(entity);
        List<Item> items = entity.getOneToManyItems();
        // ...
      }
    }
    ```
  하지만 미관상 이쁘지도 않고, 권장되는 방법이 아니기에 Fetch Join혹은 DTO 전달을 권장한다고 한다.
2. `Fetch Join`  
  `@Async` 메서드를 호출하기 전 엔티티 객체를 Fetch Join을 통해 연관 엔티티를 전부 로딩하여 넘기는 방식.

    ```java
    public interface MyRepository extends JpaRepository<MyEntity, Long> {

      // Fetch join을 통해 items를 전부 load
      @Query("SELECT DISTINCT m FROM MyEntity m LEFT JOIN FETCH m.items WHERE m.id = :id")
      Optional<MyEntity> findByIdWithAllItems(@Param("id") Long id);
    }
    ```
    ```java
    @Service
    @RequireAllArgsConstructor
    @Transactional
    public class MyEntityService {
        
      private final MyRepository myRepository;

      private final MyService myService;

      @Async
      public void someMethod(Long id) {

        myRepository.findByIdWithAllItems(id
          .ifPresent(myService::doSomething)

        // ..
      }
    }
    ```
    ```java
    @Service
    @Transactional
    public class MyService {
        
      @Async
      public void doSomething(MyEntity entity) {
        List<Item> items = entity.getItems();
        // ...
      }
    }
    ```
  위와 같이 모든 item을 Fetch join을 통해 로드 한 후 넘기게 되면, 세션이 닫혀도 문제없이 호출할 수 있다.  
  하지만 다음과 같은 문제점들이 있다.

    1. item 객체의 `@OneToMany` 연관 객체들을 호출한다면? -> `LazyInitializationException` 발생
    2. 다른 개발자가 doSomething 메서드를 사용 할 때 LazyLoading 해야 하는 사실을 모른다면? -> 같은 고뇌를 겪게 됨
    
3. `Fetch from Repository `
  `Repository`(혹은 조회 서비스) 서비스를 통해 @Async 메서드에서 다시 엔티티를 조회하는 방식
    ```java
    @Service
    @Transactional
    @RequireAllArgsConstructor
    public class MyService {

      private final MyRepository myRepository;
      
      @Async
      public void doSomething(Long entityId) {

        MyEntity entity = myRepository.findById(entityId).orElseThrow();

        List<Item> items = entity.getItems();
        // ...
      }
    }
    ```
    호출하는 Service에서 엔티티 객체를 넘기지 않고, 엔티티를 식별 할 수 있는 정보를 넘겨 서비스에서 엔티티를 조회 한 후 사용하는 방식이다.  
    하지만 다른 서비스에서 `Repository`를 의존하고 있으므로 그렇게 좋은 방법으로 보이지는 않는다.
4. `CompletableFuture`  
  `CompletableFuture`는 Java에서 비동기 프로그래밍을 지원하기 위한 기능이다. `Future` 인터페이스의 확장으로서, 비동기 작업을 병렬로 실행하거나, 작업간의 의존성을 설정하는 등 체이닝 기능을 통해 여러개의 작업을 연속적으로 실행 할 수 있다.  
  아직 필자도 많이 사용해본 기능은 아니기 때문에 좀 더 정확한 내용은 다음 포스팅에 다루기로 하겠다.
  ```java
  @Service
  public class MyService {
        
      public CompletableFuture<?> doSomething(MyEntity entity) {

        return CompletableFuture.supplyAsync(() -> {
          
          List<Item> items = entity.getItems();
          
          // ..
        });
      }
  }
  ```
  다음과 같이 구성하여 doSomthing 메서드를 호출 할 경우 `LazyInitializationException` 익셉션이 발생하지 않는다.  
  `Hibernate`의 세션이 닫힌게 아니라 호출자의 세션을 물고있는 스레드가 갈라져서 작업을 하는 개념이기 때문이다.  
  하지만 `CompletableFuture` 기능을 사용하여 Transaction 처리를 할 때에는 큰 주의를 요해야 한다.  
  마치 `open-in-view`와 문제가 비슷한데,  
  비동기 메서드(CompletableFuture)가 끝나기 전까지 기존의 세션이 닫히지 않는다. 즉 커넥션이 닫히지 않는다는 의미고,  
  비동기 메서드의 수행 시간이 늘어난다거나 하는 여러가지 이유로, 커넥션을 제대로 반납하지 못한다거나, 엔티티가 롤백되어 데이터의 정합성이 깨진다거나 하는 문제가 발생 할 수 있다.  
  해당 기능을 통해 트랜잭션을 관리하는건 리스크가 크다고 판단하여 보류하였다.

5. `DTO` 전달  
  사실 서비스에서 엔티티 객체를 넘기고, 해당 엔티티 객체를 받아서 처리하는 방식이 잘못된것으로 사료된다.  
  각 서비스들은 '정보'를 넘겨야 하지 엔티티 자체를 넘기면 서비스들 사이간의 경계가 무너지기 마련이다.  
  doSomething이 MyEntity와 그 클래스의 하위 연관 객체의 정보가 필요하다면 다음과 같은 DTO를 통해 정보를 넘기는것이 일반적이다.
    ```java
    public record MyEntityDTO(
      Long id,
      String information,
      List<ItemDTO> items
    ) {

      public MyEntityDTO(MyEntity entity) {
        this (
          entity.getId(),
          entity.getInformation(),
          entity.getItems().stream()
            .map(ItemDTO::new)
          .toList()
        );
      }
    }

    public record ItemDTO(
      Long id,
      String name
    ) {

      public ItemDTO(Item item) {
        this(
          item.getId(),
          item.getName()
        );
      }
    }

    @Service
    public class MyService {
        
        @Async
        public void doSomething(MyEntityDTO entityDTO) {
          List<ItemDTO> items = entityDTO.items();
          // ...
        }
    }

    @Service
    @RequireAllArgsConstructor
    @Transactional
    public class MyEntityService {
        
        private final MyRepository myRepository;

        private final MyService myService;

        @Async
        public void someMethod(Long id) {

          myRepository.findById(id)
            .map(MyEntityDTO::new)
            .ifPresent(myService::doSomething)

          // ..
        }
    }
    ```
  위와 같은 방법으로 이 문제를 해결하게 된다면, MyService은 Transactional을 가질 필요가 없어지고 단순히  
  그 안의 정보만을 통해 로직을 수행 할 수 있게된다.  
  모든 개발 방식에 정답은 없지만, 좋은 방법과 나쁜 방법은 존재하듯이,  
  귀찮더라도 레이어를 지키게 되면 예상치 못한 에러를 방지 할 수 있다 라고 생각한다.  
  물론 필자가 시도했던 방법보다 훨씬 좋은 방법이 있겠지만. 당장 짱구를 굴려 나온 방식은 이게 최선이라고 생각한다.