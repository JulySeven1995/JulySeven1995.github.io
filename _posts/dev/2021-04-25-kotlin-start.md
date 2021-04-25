---
title: "Kotlin + Spring Boot 연습 프로젝트"
categories:
  - Dev
tags:
  - Spring
  - Spring Boot
  - Kotlin
layout: single
comments: true
---

## Kotlin + Spring Boot



\- `kotlin`

  `Kotlin`의 존재를 알게 된 건, 2019년 7월인가 8월인가, 첫번째 직장에 면접보러 갔을때, 면접관이 `Scala` 와 `Kotlin`이 뭔지 아냐고 물어봐서 호기심을 가지고 구글에 검색해 봤었던 것 같다. (이런게 있구나 신기하다 하고 넘어갔을듯 하다)

  그런데 최근에 많은 IT 기업에서 백엔드 개발에 `Kotlin`을 채택하는 추세가 늘어나고 있다고 한다. 문법도 엄청 간결하다고 하고, `null` 안정성 제공에 함수형 언어에... 등등등 제일 신기했던건 `Javascript` 처럼 `===` 비교문이 있었다는것. 

  사실 뭐 백문이 불여일견이라고, 한번 써보고싶어졌다. 깊게 써보지도 않을꺼고, 그냥 간단하게 프로젝트 세팅하고, Rest로 RDB CRUD 하는 프로젝트를 만들어 보았다. 앞으로 생각나는게 있거나, 구현해보고싶은거, 회사에서 만들어본 로직을 복기하는 느낌으로 점차 추가하려고 한다.



  프로젝트 구조는 아래처럼 만들었다.

```shell
  D:\dev\julyseven-kotlin-practice>gradle -q projects


  \------------------------------------------------------------

  Root project 'julyseven-kotilin-practice'

  \------------------------------------------------------------


  Root project 'julyseven-kotilin-practice'

  +--- Project ':julyseven-api'

  \--- Project ':julyseven-common'
  
```



  개인적으로 모노리스 구조를 굉장히 싫어해서, 그냥 간단한 연습용 프로젝트도 어지간해선 멀티모듈 프로젝트로 구성하는걸 선호한다. kotlin + gradle 으로 멀티모듈을 구성하는 예제가 쉽고 간단하게 정리된 곳이 많이 있어서 도움을 많이 받았따. 이번에는 `Http Method` 로 간단하게  `CRUD`  하는 프로젝트를 구성하려고 한다.



`common` 모듈은 주로 `Jpa` 관련된 클래스들이 정의되어 있고, `H2 Database` 만 쓸거라서, 딱히 `Config` 설정은 별도로 하지 않았다. 오늘 프로젝트에는 간단하게 `User`라는 `Entity` 만 설정하고 말았다. `Kotlin`도 `Exposed`, `Ktorm`이라는 `ORM` 프레임워크가 있다고는 하는데... 사실 이거까지 공부하면 피아노를 연습할 시간이 없을것 같다. 😫

```kotlin
@Entity
@Table(name = "USER")
data class User(
    @Id
    @Column(name = "USER_ID")
    var userId: String,
    @Column(name = "USER_NAME")
    var userName: String,
    @Column(name = "password")
    var password: String
) {

}
```

```kotlin
interface UserRepository : JpaRepository<User, String> {

    fun findByUserId(userId: String): Optional<User>

    fun findAllByUserName(userName : String): List<User>

}
```

너무나 간단해서 눈물이 나올 지경이지만, 나는 장대한 프로젝트를 원하는게 아니라, `Hello World` 급의 프로젝트를 만드려고 했다. 걸음마도 못땠는데 운전을 어떻게 하리


`api` 모듈은 `Rest Api` 느낌으로 간단하게 `User` Entity를 `CRUD` 하는 형식의 `Controller`를 만들어 주었다. `repository`는 물론 `Service`로 접근한다.

```kotlin
interface UserService {

    fun getUser(userId: String): Optional<User>

    fun createUser(user: User): User

    fun updateUser(user: User): User

    fun deleteUser(userId: String)
}
```

```kotlin
@RestController
class JpaController(
    private val userService: UserService,
    private val objectConvertService: ObjectConvertService
) {

    private val logger = LoggerFactory.getLogger(this.javaClass)


    @CrossOrigin("*")
    @RequestMapping(method = [RequestMethod.GET], path = ["/getUserInfo"], params = ["userId"])
    @ResponseBody
    fun getUserInfo(@RequestParam userId: String): Map<String, Any> {

        logger.info("[GET] User Information Request, UserId = [{}]", userId)

        return userService.getUser(userId)
            .map(objectConvertService::convertToMap)
            .orElseThrow {
                logger.error("[GET] User Information Request Failed, UserId = [{}]", userId)
                ResponseStatusException(HttpStatus.NOT_FOUND, "User Not Found")
            }
    }

    @CrossOrigin("*")
    @RequestMapping(method = [RequestMethod.POST], path = ["/createUser"], params = ["userId", "userName", "password"])
    @ResponseBody
    fun createUser(@RequestParam userId: String, @RequestParam userName: String, @RequestParam password: String): Map<String, Any> {

        logger.info("[POST] User Create Request, UserId = [{}]", userId);

        val user = User(userId, userName, password)

        try {
            userService.createUser(user)
        } catch(e : Exception) {
            logger.error("[POST] User Create Request Failed, UserId = [{}]", userId);
            throw ResponseStatusException(HttpStatus.BAD_REQUEST, "Create User Failed, -> " + e.message);
        }

        return objectConvertService.convertToMap(user)
    }

    @CrossOrigin("*")
    @RequestMapping(method = [RequestMethod.PUT], path = ["/updateUser"], params = ["userId"])
    @ResponseBody
    fun updateUser(@RequestParam userId: String, @RequestParam userName: String, @RequestParam password: String): Map<String, Any> {

        logger.info("[PUT] User Update Request, UserId = [{}]", userId);

        val user = User(userId, userName, password)

        try {
            userService.updateUser(user)
        } catch(e : Exception) {
            logger.error("[PUT] User Update Request Failed, UserId = [{}]", userId);
            throw ResponseStatusException(HttpStatus.BAD_REQUEST, "Update User Failed, -> " + e.message);
        }

        return objectConvertService.convertToMap(user)
    }

    @CrossOrigin("*")
    @RequestMapping(method = [RequestMethod.DELETE], path = ["/deleteUser"], params = ["userId"])
    fun deleteUser(@RequestParam userId: String): ResponseEntity<Any> {

        logger.info("[DELETE] Delete User Request, UserId = [{}]", userId);

        try {
            userService.deleteUser(userId)
            return ResponseEntity("Delete User Succeed", HttpStatus.OK)
        } catch (e: Exception) {
            logger.error("[DELETE] User Update Request Failed, UserId = [{}]", userId, e);
            throw ResponseStatusException(HttpStatus.BAD_REQUEST, "Delete User Failed, -> " + e.message);
        }
    }
}
```

이렇게 간단한 코드를 짜면서도 느낀거지만, 확실히 `Java`에 너무 익숙해진게 아닌가 싶었다. 엔티티 객체를 생성하려니까 습관적으로  `new` 키워드를 작성하게 되고, 객체에서 `getter/setter` 를 찾다가 아뿔싸! 하게되고 😂 자바에서는 stream을 정말 즐겨썼는데, 막상 이런 함수형 언어를 사용하니까 익숙치 않아서 쩔쩔맸다. 아직 코틀린의 코자도 모르는 상태라서, 뭐가 좋고 나쁜지는 모르겠지만, 확실히 자바 보다는 정신사나운게 덜하다고 해야하나



앞으로 이 프로젝트를 기반으로, 장난감이나 조금씩 만들어보려고 한다.

> [연습 프로젝트 Github 링크](https://github.com/JulySeven1995/kotlin-practice/tree/20210425-StartProject )

> [Kotlin/Gradle/Spring Boot 멀티 모듈 프로젝트 구성(하나셋님)](https://www.youtube.com/watch?v=Of4hT2TpAlY)