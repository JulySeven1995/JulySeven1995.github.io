---
title: "Kotlin Data vs Java Record"
categories:
  - Dev
tags:
  - Kotlin
  - Java
layout: single
comments: true
---

### __Kotlin - Data class__

코틀린에서는 데이터를 다루기 위한 클래스를 기본적으로 제공한다.  
바로 `data` 라는 클래스인데, 우리가 Java에서 DTO를 구현하기 위해 필요한 갖가지 기본 함수(toString, getter, equals 등등)들을 제공해준다.  
백문이 불여일견이듯이, Java의 class로 표현한 DTO와 Kotlin의 Data class로 표현한 DTO의 코드를 살펴보면 다음과 같다.

1. _Java_
    ```java
    public class Person {

        private final Long id;

        private final String firstName;

        private final String lastName;

        private final LocalDateTime dateOfBirth;

        public Person(Long id, String firstName, String lastName, LocalDateTime dateOfBirth) {
            this.id = id;
            this.firstName = firstName;
            this.lastName = lastName;
            this.dateOfBirth = dateOfBirth;
        }

        @Override
        public String toString() {
            // toString 구현
        }

        @Override
        public boolean equals() {
            // equals 구현
        }

        @Override
        public String hashCode() {
            // hashCode 구현
        }

        public Long id() {
            return this.id;
        }

        // getter 구현...
    }
    ```
    물론 `Lombok`을 이용해 보일러플레이트 코드의 양을 비약적으로 줄일 수 있긴 하지만,  
    Lombok도 결국에는 보일러플레이트 코드를 어노테이션으로 구현해주는것이고,  
    Lombok의 어노테이션 자체가 보일러플레이트 코드가 되어버린다.
2. _Kotlin_
    ```kotlin
    data class Person(
        val id: Long,
        val firstName: String,
        val lastName: String,
        val dateOfBirth: LocalDateTime,
    )
    ```
    너무나도 간단하게 DTO 클래스가 완성되었다. `val`은 `final`을 뜻하는 변수로 이해하면 된다(가변형은 `var`)  
    kotlin의 data class는 getter, toString, equals, hashCode등의 함수들을 기본적으로 구현해준다.

    테스트코드
    ```kotlin
    val person1 = Person(1, "재호", "최", LocalDateTime.now())
    val person2 = Person(2, "Saki", "Yokoyama", LocalDateTime.now())

    // toString()
    println("person1: $person1")
    println("person2: $person2")

    // equals()
    println("person1 == person2: ${person1 == person2}")

    // hashCode()
    println("person1 hashCode: ${person1.hashCode()}")
    println("person2 hashCode: ${person2.hashCode()}")
    ```
    테스트 코드 결과
    ```
    person1: Person(id=1, firstName=재호, lastName=최, dateOfBirth=2023-05-22T17:30:45.123)
    person2: Person(id=2, firstName=Saki, lastName=Yokoyama, dateOfBirth=2023-05-22T17:30:45.123)
    person1 == person2: false
    person1 hashCode: -527288411
    person2 hashCode: -1411844362 
    ```

### __Java - Record class__

`Java`에서도 손가락만 물고 있을 수는 없었는지, Kotlin의 Data 클래스와 대응되는 Record 클래스를 지원하기 시작했다.  
Java 14 부터 지원되었으며, 공식적으로는 Java 16 부터 지원되었다.
```java
public record Person(
    Long id,
    String firstName,
    String lastName,
    LocalDateTime dateOfBirth
) {

}
```
테스트코드
```java
Person person1 = new Person(1L, "재호", "최", LocalDateTime.now());
Person person2 = new Person(2L, "Saki", "Yokoyama", LocalDateTime.now());

// toString()
System.out.println("person1: " + person1.toString());
System.out.println("person2: " + person2.toString());

// equals()
System.out.println("person1 equals person2: " + person1.equals(person2));

// hashCode()
System.out.println("person1 hashCode: " + person1.hashCode());
System.out.println("person2 hashCode: " + person2.hashCode());
```
테스트 코드 결과
```
person1: Person[id=1, firstName=재호, lastName=최, dateOfBirth=2023-05-22T17:30:45.123]
person2: Person[id=2, firstName=Saki, lastName=Yokoyama, dateOfBirth=2023-05-22T17:30:45.123]
person1 equals person2: false
person1 hashCode: 979
person2 hashCode: 1025
```

### Conclusion

* __공통점__
    1. 데이터 저장 및 표현: 둘 다 데이터를 저장하고 표현하기 위한 클래스이다.
    2. 속성 선언: 간결한 구문으로 속성을 선언 할 수 있다.
    3. 자동 셍성 메서드: 컴파일러가 자동으로 toString, equals, hashCode 등의 메서드를 생성한다.
* __차이점__
    1. record는 모든게 final이다.
        * Data 클래스는 속성을 `val`(불변) or `var`(가변) 로 구성할 수 있다.
        * Record 클래스는 속성이 전부 final이다. -> 상속을 할 수 없다.
    2. record는 패턴 매칭이 가능하다 (`instanceof` 키워드 사용 가능)
        * 하지만 kotlin에는 `when` 이라는 비슷한 기능을 제공한다


