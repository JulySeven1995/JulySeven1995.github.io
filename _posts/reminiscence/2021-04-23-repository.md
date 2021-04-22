---
title: "@Repository? @Service?"
categories:
  - Dev
tags:
  - Spring
  - Spring Boot
layout: single
---

Repository? Service?


  `Spring Framework` 에서는 `IoC`  컨테이너에 `Context` 를 등록하는 여러가지 방법이 있다. `Xml Config` 등록, `@Configuration` 등록, 구현 클래스에 `@Component` , `@Service`, `@Repository` 명시 등등[(Component, Service, Repository 차이는 링크 글 참조)](/dev/injection/)  참조된 글에서 설명하다 시피, 어느 어노테이션을 사용하던간에, `org.springframework.stereotype` 패키지의 어노테이션은 `IoC` 컨테이너에 클래스를 `Context` 로 등록해주는 기능을 한다.

  실제로, `org.springframework.stereotype` 패키지의 `@Component` , `@Service`, `@Repository`, `@Controller`  어노테이션 구현 코드를 보면 다음과 같다.

  ```java
  @Target(ElementType.TYPE)
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Indexed
  public @interface Component {
  
  	String value() default "";
  }
  ```

  ```java
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Component
  public @interface Controller {
  
  	@AliasFor(annotation = Component.class)
  	String value() default "";
  }
  ```

  ```java
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Component
  public @interface Service {
  
  	@AliasFor(annotation = Component.class)
  	String value() default "";
  }
  ```

  ```java
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Component
  public @interface Repository {
  
   @AliasFor(annotation = Component.class)
  	String value() default "";
  }
  ```

  Controller, Service, Repository 어노테이션 모두, `@Component` 어노테이션이 명시 되어있음을 확인 할 수 있으며, `@AliasFor` 로 묶여있음을 확인 할 수 있다. 이는 곧 세 어노테이션 모두  `@Component` 어노테이션과 같은 기능을 한다는것을 확인 할 수 있었다. 즉 세 컴포넌트의 차이는 그저 `@Component`에 역할을 명시하는것이 어노테이션들의 역할이라고 이해 할 수 있다.

  >"그럼 Repository는 어느 클래스에 명시하고, Service는 어느 클래스에 명시해야 하나요? 🤔"

  매우 훌륭한 질문이라고 할 수 있다. 사실 필자가 스프링 프레임워크에 대한 이해도가 적었을 당시에는, 기능적으로 차이가 없으니 전부 `@Component` 어노테이션으로 명시하여 `IoC` 컨테이너에 `Context`를 등록하였다. 하지만 개발은 혼자서 하는것이 아니며, 입으로 주고받는 대화보다 코드로 주고받는 대화가 훨씬 중요하므로, `명시` 라는 개념이 굉장히 중요하다.

  단순명료하게 말 하자면, `Repository` 는 말 그대로, `Resource` 를 주고받는 역할을 하는 인터페이스에 명시하여야 한다. 대표적으로는 `DB CRUD`의 엔드포인트를 하는 인터페이스에 명시한다. 대표적으로 `MyBatis` 와 같은 `SQLMapper` 프레임워크를 연동하여 사용 할 때 다음과 같은 `Repsository` `Context`들이 많이 존재한다.

  ```java
  @Repository
  public interface UserRepository {
      @Select("SELECT * FROM user WHERE USER_ID = #{userId}")                       
      Optional<User> findByUserId(String userId);
  
  		@Update("UPDATE user SET USER_NAME = #{userName}, PASSWORD = #{password} "
  						 + "WHERE USER_ID = #{userId}")                       
      User updateUser(User user);
   
      @Delete("DELETE FROM user WHERE USER_ID = #{userId}")
      void deleteUserByUserId(String userId);
  
  		@Insert("INSERT INTO user(USER_ID, USER_NAME, PASSWORD) "
  						+ "VALUES (#{userId}, #{userName}, #{password})")
  		User createUser(User user);
  }
  ```

  해당 인터페이스는 User 라는 테이블의 값을 가져오거나, 생성하는 저장소의 역할을 한다. 단순하게 저장하고, 가져오는 역할만 수행하는 것이다. 즉 로직이 존재해서는 안된다.

  `Repository` 와 구현 비즈니스 로직간의 중개자 역할을 해 주는 것이, `Service` 의 역할이다. (물론 `Service` 는 다른 여러 기능을 제공하는 갖가지 `Context`로 구현될 수 있다. 지금은 DB CRUD 기준으로 설명한다.)

  ```java
  public interface UserService {
  	
  		Optional<User> getUserById(String userId);
  
      User createUser(User user);
  
      User updateUser(User user);
  
      void deleteUser(String userId);
  
  }
  ```

  ```java
  @Service
  public class UserServiceImpl implements UserService {
  
      private final UserRepository repository;
  
      public UserServiceImpl(UserRepository repository) {
  
          this.repository = repository;
      }
  
      @Override
      public Optional<User> getUserById(String userId) {
  
          return repository.findByUserId(userId);
      }
  
      @Override
      public User createUser(User user) {
  
          repository.createUser(user);
      }
  
      @Override
      public User updateUser(User user) {
  
          return repository.updateUser(user);
      }
  
      @Override
      public void deleteUser(String userId) {
  
          repository.deleteUserByUserId(userId);
      }
  }
  ```

  대강 이렇게 `Service` 가 구현되어, 비즈니스 로직에서 사용된다. 이렇게만 놓고 보면, 문득 이런 생각이 들 수 있다.

  >"그냥 Repository에 있는 메서드 갖다 쓴거 아니야? 이 짓거리를 왜 해?"

  필자가 `Spring Framework` 를 활용해 처음 개발을 했을 때, 가장 먼저 들었던 생각이다.  아래의 예제를 통해 해당 패턴이 개발을 하면서 어떤 이득을 얻을 수 있는지 확인 해 보자.

  ```java
  @Controller
  public class Controller {
  
  	private final UserService userService;
    private final ObjectMapperService objectMapperService;
  
  	public Controller(UserService userService, ObjectMapperService objectMapperService) {
  
          this.userService = userService;
  				this.objectMapperService = objectMapperService;
    }
  
  	@CrossOrigin("*")
    @RequestMapping(method = RequestMethod.POST, path = "/createUser", params = { "userId", "userName", "password" })
    @ResponseBody
    public Map<String, Object> createUser(@RequestParam String userId, @RequestParam String userName, @RequestParam String password) throws ResponseStatusException {
  
  		 // 금지된 USER_ID로 생성 요청이 들어왔을 경우 필터링
  		 if (IllegalUserList.contains(userId)) {
  				throw new ResponseStatusException(HttpStatus.BAD_REQUEST,
  					String.format("Create User Failed, Illegal userId : [%s]", userId));
  		 }
  
       User user = new User(userId, userName, password);
       user = userService.createUser(user);
       return objectMapperService.convertToMap(user);
    }
  
  }
  ```

  만약 위와 같이, DB에 CRUD를 하기 전에 사전작업이 필요한 경우가 로직을 구성하면서 생길 수 있다. 별 문제가 없어 보이는 로직이지만, `createUser()` 메서드는 해당 컨트롤러에서만 사용하는것이 아닌, 범용적으로 사용되는 메서드이다.

  따라서 아래와 같이 Service 구현 메서드 부분에서 사전작업을 해주면, 중복코드 방지에 큰 이바지 할 수 있다. 해당 사전 작업은 필터링이 될 수도 있고, 캐싱이 될 수도 있다.

  ```java
  @Override
  public void createUser(User user) throw IllegalArgumentException {
  
  	 if (IllegalUserList.contains(userId)) {
  				throw new IllegalArgumentException(
  					String.format("Create User Failed, Illegal userId : [%s]", userId));
  	 }
     repository.createUser(user);
  }
  
  ```