---
title: "@NoRepositoryBean"
categories:
  - Dev
tags:
  - Spring Boot
  - JPA
layout: single
comments: true
---

Bean & Injection

`@NoRepositoryBean`

  `Spring Data JPA`  라이브러리는 `JpaRepository` 인터페이스와 `CrudRepository` 인터페이스를 제공하며, 필자도 개발 할 때, 아주 유용하게 사용한다. 

```java
@Repository
public interface UserRepository extends JpaRepository<User, String> {

    Optional<User> findByUserId(String userId);

    void deleteUserByUserId(String userId);
}
```

위와 같이 `JpaRepository`를 자주 사용하는데, `Spring Boot` 개발 경험이 있는 분한테서 `JpaRepository`에 `@Repository` 어노테이션 안붙인다는 말을 들었다. `Context` 등록을 하지도 않고 쓴다고? 아니면 `@Repository` 말고 다른 어노테이션을 명시하나? 별에 별 생각이 들어서 `@Repository` 어노테이션을 제거하고, 테스트코드를 돌려보니까 잘 돌아가더라

![](/assets/images/Curiosity.jpg)

왜 그런가 싶어서, `JpaRepository` 구현부 코드를 까봤더니 아니나 다를까

```java
@NoRepositoryBean
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {

	...

}
```

`@NoRepositoryBean` 이런 어노테이션이 있길래, 뭐하는 녀석인지 조사를 좀 해봤다.

구현 코드의 주석은 다음과 같이 적혀있었다.

> 저장소 인터페이스를 선택할 수 없도록 하여 인스턴스를 생성하는 것을 제외하는 어노테이션

이는 일반적으로 모든 리포지토리에 대해 사용자 지정 리포지토리 기본 클래스와 함께 확장 기본 인터페이스를 제공하여 해당 중간 인터페이스에 선언된 메서드를 구현할 때 사용됩니다. 이 경우 일반적으로 중간 저장소 인터페이스에서 구체적인 저장소 인터페이스를 추출하지만 중간 인터페이스에 대한 스프링 빈(Spring bean)을 생성하려고 하지 않습니다.

번역기의 문제인지, 읽는 사람의 문제인지 도통 이해가 되질 않아, 구글링 해 보았는데, 다음과 같은 설명을 볼 수 있었다.

![](/assets/images/spring-common.png)

`@NoRepositoryBean` 어노테이션은 이 인터페이스가 Repository 용도로서 사용되는 것이 아닌 단지 Repository의 메서드를 정의하는 인터페이스라는 정보를 부여하는 의미로 사용된다.

당장 이해가 어렵고, 설명을 읽어도 이해가 되지 않아, `JpaRepository`는 어노테이션이 등록이 이미 돼 있으므로(`@Retention(RetentionPolicy.RUNTIME)`),  `@Repository` 어노테이션을 붙이지 않아도 된다 라고 이해하고, 관련된 서적이나 강의를 찾아봐야 될 필요성을 느꼈다.

2021-04-24 회고록 끝