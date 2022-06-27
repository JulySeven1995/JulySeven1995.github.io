---
title: "디자인패턴 - Builder Pattern"
categories:
  - Dev
tags:
  - Design Pattern
layout: single
comments: true
---

### Index

1. Summary
2. Usage
3. Conclusion

---

### 1. Summary

[Ref : 위키백과](https://ko.wikipedia.org/wiki/%EB%B9%8C%EB%8D%94_%ED%8C%A8%ED%84%B4)

- What is `Builder Pattern`
    
    구글에서 찾아보면 ‘복합 객체의 생성 과정과 표현 방법을 분리하여 동일한 생성 절차에서 서로 다른 표현 결과를 만들 수 있게 하는 패턴’ 이라고 한다. ‘생성’과 ‘표현'의 분리를 핵심으로 둔 생성패턴.
    
    결과적으로 생성자를 다른 클래스에 위임하는 방식의 패턴이라고 생각할 수 있다.
    
- 특징
    
    예시에 사용할 클래스, Lombok을 사용한다고 가정함.

    ```java
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstrctor
    @Builder
    public class User {
    
        @Builder.Default
        private String name = "재빠른호돌이";
            
        @Builder.Default
        private Integer age = 28;
        
        @Builder.Default
        private Integer height = 170;
        
    }
    ```
    
    1. 필요한 인자만을 이용하여 객체 생성
        
        예시에 상기된 클래스의 경우 기본 값을 사용하는 생성자, 모든 인자를 집어넣는 생성자만을 사용해야만 한다. 만약 기본값 중 name만 바꾸고 싶다면 쓸모 없는 코드들이 굉장히 많이 생성된다.
        
        ```java
        User 유저A = new User();
        유저A.setName("최재호");
        
        User 유저B = new User("최재호", 28, 170);
        ```
        
        이 객체를 생성하여 변경이 필요한 값은 name 뿐이었지만, 유저A는 set메서드 호출, 유저B는 굳이 기본값을 입력해야 하는 상황이 발생했다. 심지어 유저B의 기본값이 바뀐다면, 유저B를 생성하는 코드 변경또한 불가피하다. 이는 Builder Pattern 사용으로 간단하게 해결이 가능하다.
        
        ```java
        User 유저C = User.builder()
                        .name("최재호")
                    .build();
        ```
        
        객체를 생성 후 set메서드 호출, 기본값을 입력하는 불상사를 해결하여 결과적으로 ‘필요한 인자'만을 사용하여 객체를 생성 할 수 있다.
        
    2. 필드 추가의 유연성
        
        User라는 클래스에 핸드폰 번호가 추가 되었다고 가정을 해보자.
        
        ```java
        // nullable
        private String phoneNumber;
        ```
        
        클래스에 명시된 생성자는 인자 없이 생성하는 생성자, 모든 인자를 받는 생성자로 이루어져있고, 1번 항목에서 사용된 유저B 코드는 당연하게도 컴파일 에러를 야기할 것이다.
        
        ```java
        User 유저B = new User("최재호", 28, 170);
        ```
        
        결과적으로 클래스에 필드를 하나 추가하며, 해당 클래스를 생성하는 코드를 일일히 수정해주어야 하는 불상사가 발생한다.
        
        ```java
        User 유저B = User.builder()
                    .name("최재호")
                    .phoneNumber("01041083145")
                .build();
        ```
        
        상기와 같이 builder 클래스를 사용해 객체를 생성할 경우, 필드가 추가적으로 생성된다 하더라도 컴파일 에러가 발생하지 않는다.
        
    3. 생성자에 비해 뛰어난 가독성
        
        로직상에서 User 객체를 생성해야 하는 상황이 왔을때, 생성자에 모든 인자값을 넣어 생성하는 코드를 짠다고 가정을 해보자.
        
        ```java
        User 유저A = new User("최강의사나이", 20, 180, "01012345678");
        ```
        
        생성자에 입력된 인자들이 각각 무슨 값을 의미하는지 정확하게 알 수 없다.
        
        결과적으로 생성자를 직접 확인 해 보아야 알 수 있다는것은 가독성이 떨어진다는 것을 의미한다.
        
        Builder 클래스를 이용하여 객체를 생성할 경우 각 인자의 의미를 알며 생성 할 수 있다.
        
        ```java
        User 유저A = User.builder()
                        .name("최강의사나이")
                        .age(20)
                        .height(180)
                        .phoneNumber("01012345678")
                .build();
        ```
        

---

### 2. Usage

Usage 코드는 실무에서 적용시켰던 것을 각색하여 작성되었다.

```java
@Getter
@Setter
public class UserDTO {

    private String firstName;
    
    private String lastName;

    private String nickname;

    private String phoneNumber;


    @Builder(builderClassName = "DefaultUserBuilder", builderMethodName = "DefaultUserBuilder")
    public User(String nickname, String phoneNumber) {
        
        this.firstName = "undefined";
        this.lastName = "undefined";
        this.nickName = nickname;
        this.phoneNumber = phoneNumber;
    }

    @Builder(builderClassName = "FullInfoUserBuilder", builderMethodName = "FullInfoUserBuilder")
    public User(String firstName, String lastName, String nickname, String phoneNumber) {

        this.firstName = firstName;
        this.lastName = lastName;
        this.nickname = nickname;
        this.phoneNumber = phoneNumber;
    }

    public static of(User user) {
        return UserDTO.FullInfoUserBuilder()
            .firstName(user.getFirstName())
            .lastName(user.getLastName())
            .nickname(user.getNicknamee())
            .phoneNumber(user.getNumber())
        .build();
    }

}
```

Lombok의 힘을 빌려 위와 같이 builder의 메서드명을 지정할 수 있으며,  각 빌더마다 요구하는 인자도 다르게, 개발자가 빌더의 메서드명으로 어떤 목적을 갖고 있는 빌더인지 알 수 있게끔 작성되어 있다.

다음은 Spring Batch에서 제공하는 Reader 클래스의 객체 생성을 일반 생성자로 set 하는것과 Builder 클래스를 통해 생성하는 코드를 비교해보도록 하자.

```java
@Bean
@StepScope
public JpaPagingItemReader<SomeItem> someItemReader(@Value("#{jobParameters[requestDate]}") final Date requestDate) {

    JpaPagingItemReader<SomeItem> walletItemReader = new JpaPagingItemReader<>();
    walletItemReader.setName("someItemReader");
    walletItemReader.setEntityManagerFactory(entityManagerFactory);
    walletItemReader.setQueryString("jpql query...");
    walletItemReader.setParameterValues(Collections.singletonMap("expireDate", requestDate.toInstant()));
    walletItemReader.setSaveState(Boolean.FALSE);
    walletItemReader.setPageSize(CHUNK_SIZE);
    
    return walletItemReader;
}
```

```java
@Bean
@StepScope
public JpaPagingItemReader<SomeItem> someItemReader(@Value("#{jobParameters[requestDate]}") final Date requestDate) {

    return new JpaPagingItemReaderBuilder<SomeItem>()
        .name("someItemReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("jpql query...")
        .parameterValues(Collections.singletonMap("expireDate", requestDate.toInstant()))
        .saveState(Boolean.FALSE)
        .pageSize(CHUNK_SIZE)
      .build();
}
```

Builder 클래스를 사용하지 않으면 첫 번째 코드와 같이 무수하게 수많은 set 메서드를 보며 현기증이 날 수밖에 없는 코드가 작성된다.

반대로 Builder 객체를 사용하여 객체를 생성 할 경우, return inline으로 객체를 생성할 수 있으며, 가독성이 떨어지는 수많은 Set 메서드를 간략화 할 수 있다.

Spring 개발자의 든든한 친구 Baeldung 에서도 Builder 클래스를 활용하여 객체를 생성할 것을 적극 권장하고 있다.

---

### 3. Conclusion

결과적으로 Builder 패턴이 코드를 작성 할 때 필수적인 패턴인가 에 대해서는 확답을 내릴 수는 없다. 

빌더 클래스를 새로 생성해야하고, 대상 클래스의 변경이 있을 때마다 빌더 클래스를 새로 변경해야 하는 상황도 나올 수 있다.(해당 문제는 Lombok을 적절하게 사용하면 최소화 할 수 있다고 생각한다.)

깔끔하고 가독성있고 유지보수에 용이한 코드를 위해 Builder Pattern을 적극적으로 활용하는 것이 좋다고 사료된다.