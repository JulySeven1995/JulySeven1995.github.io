---
title: "코틀린 공부 일지"
categories:
  - Dev
tags:
  - Kotlin
  - Java
layout: single
comments: true
---

## 1. Why Kotlin

- 코틀린이 뭡니까
    
    > [IntelliJ IDEA](https://namu.wiki/w/IntelliJ%20IDEA)의 개발사 [JetBrains](https://namu.wiki/w/JetBrains) 에서 2011년에 공개한 [오픈 소스](https://namu.wiki/w/%EC%98%A4%ED%94%88%20%EC%86%8C%EC%8A%A4)  [프로그래밍 언어](https://namu.wiki/w/%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D%20%EC%96%B8%EC%96%B4). 
    [JVM](https://namu.wiki/w/Java%20Virtual%20Machine) 기반의 언어이며, Java와 유사하지만 더 간결한 문법과 다양한 기능을 추가하였다. 
    [Java](https://namu.wiki/w/Java) 언어와 상호 운용이 100% 지원된다.
    > 
    > 
    > ref. 나무위키
    > 
- 왜 코틀린입니까?
    - 간결한 문법
        
        아래에 기재되는 여러 이유들을 통해 Java에 비해 굉장히 간결한 코드를 작성 할 수 있음
        
    - 정적 타입 지정 언어
        
        개발자가 타입을 선언하지 않아도 됨
        
    - Null 안정성
    코틀린을 사용해야하는 가장 큰 이유 중 하나 `TypeScript`와 유사한 Null 안정성을 지원함.
    Java의 경우 이를 `Optional` 이란 객체로 Wrapping 하여 사용하지만 코틀린은 기본제공
    → 런타임 중 NPE 발생 여지를 사전에 차단 해 줌
    - 함수형 언어
        
        Java에서는 Null Safety를 보장하기 위해 `Optional`이란 Wrapping 객체를 사용하고, 함수형을 표방하기 위해 `Predicate` 라는 인터페이스를 사용하고 있음. 하지만 결과적으로 Java에서 함수(method)는 2등급 시민으로, 1등급 시민인 ‘값’ 이 될수는 없기에 상기된 인터페이스로 위장하여 넘어감.
        → Kotlin은 함수를 1등급 시민으로 취급하여 `Argument`로 전달이 가능
        
    - Java와의 호환성
        
        Kotlin은 Java로 구현된 모든 라이브러리, 프레임워크를 사용 할 수 있음. 자바 Application에 Kotlin Class를 만들어서 사용 할 수도 있음. 
        → 선도적인 IT기업에서는 Kotlin + Spring Boot 어플리케이션이 많아지는 추세
        
    - (Optional) Android Application 개발 언어
        
        → 안드로이드 어플리케이션 언어로 채택된 코틀린을 숙달함으로써 개발 플랫폼 확장 선택지 증가
        
    - 탄탄한 공식 레퍼런스
        - [Kotlin References](https://www.notion.so/5dd5d46688904221a8c1866196a30eed)
        - [Baeldung - Kotlin](https://www.baeldung.com/kotlin/)
        
        → 구글에서 작정하고 밀어주는 언어 답게 레퍼런스 문서가 굉장히 잘 정리되어 있음

## 2. Java vs Kotlin

### Primitive Type

Java와 Kotlin 모두 기본적인 데이터(int, long, float, double, boolean 등)타입이 존재하며, Java에서는 이에 대한 Reference Type이 정의되어 있음. 하지만 코틀린은 두개가 통일 된 상태로 존재.

Java의 Reference Type은 연산에 사용하면 Wrapping, Unwrapping 하는 과정에서 리소스를 소모하기때문에 연산하는 로직에서는 사용을 지양함.

→ 코틀린은 컴파일 할 때 컴파일러가 Wrapping이 필요한 경우 Wrapper 클래스로 묶어서 컴파일 하게 됨. 즉 개발자가 신경을 쓸 필요가 없음

### Define Variable Type

Java는 기본적으로 `[타입] [변수명]` 순으로 변수를 선언하며, 앞에 부가적으로 상수(final), 정적(static), 접근지정자를 붙여 선언해주는 형태임.

```java
String first = "first"
final String secend = "second"
```

코틀린에서는 `[상수/변수] [변수명]` 순으로 변수를 선언하며 앞에 부가적으로 접근지정자, 뒤에 데이터의 타입과 nullable 여부를 붙여 선언해주는 형태임.

```kotlin
var first: String? = "first"
val second: String = "second"
```

타입스크립트와 비슷한 형태를 가지고있으며, var는 가변형 val는 불변형을 뜻함.

### Optional vs Nullable(Wildcard Type)

사실상 코틀린을 사용하는데 가장 큰 이유 중 하나. Java에서는 Null Safety를 지원하기 위해 Java8 버전부터 `Optional`이라는 Wrapper 클래스를 지원함. 하지만 코틀린은 기본적으로 TypeScript와 같이 `?` 와일드 카드를 이용하여 Null Safety를 지원함

```java
// JAVA
Optional<String> text = Optional.ofNullable("text");
```

```kotlin
// Kotlin
var text: String? = "text"
```

코틀린에서는 Optional에서 지원하는 map, filter, ifPresent 등의 메서드들을 함수형으로 구현 할 수 있음

```java
// JAVA
text.filter(item -> item.contains("t")
    .map(item -> item + " append String")
    .ifPresent(item -> System.out.println(item));
```

```kotlin
//Kotlin
text?.takeIf { it.contains("t") }
    .let { it + " append String" }
    .also { println("${it}") } 
```

Optional에서 지원하는 기능을 완벽하게 수행 할 수 있으며, `it`이라는 키워드를 통해 좀 더 간결하게 코드를 구성 할 수 있음

### Elvis Operator

위 내용의 연장선이라고 볼 수 있는데, Optional 객체에서 데이터를 가져오는 방식과 코틀린의 nullable 타입에서 데이터를 가져오는 방식을 비교하며 설명함.

```java
// Java

Optional<String> first = Optional.ofNullable("first");
// will print first
System.out.println(text.orElse("Variable 'first' is empty")); 

Optional<String> second = Optional.empty();
// will throw NoSuchElementException
System.out.println(second.orElseThrow()); 
```

orElse 메서드로 대체값을 리턴하거나 orElseThrows 메서드를 사용하여 인출 불가능 할 시 예외를 발생 시킬 수 있음.

코틀린에서는 다음과 같이 표현이 가능함

```kotlin
// Kotlin

var first: String? = "first"
// will print first
print(first ?: "Variable 'first' is emtpy")

var second: String? = null
// will throw NoSuchElementException
print(second ?: throw NoSuchElementException())

```

대체값을 ?: 연산자로 지정 해 줄수 있는데, 이를 엘비스 연산자라고 호칭함.

이 엘비스 연산자는 Early return 기능 또한 지원하는데 다음과 같은 Java 메서드가 있다고 가정함

```java
// Java

public void doSomething(Long num) {
    if (num == null) {
        return;
    }

    ...
}
```

들어온 인자값이 null 일 경우에 하위 로직을 실행시키지 않고 종료하는 코드를 코틀린의 엘비스연산자를 통해 다음과 같이 표현 할 수 있음.

```kotlin
// Kotlin

fun doSomething(num: Long?) {
    
    num ?: return

    ...
}
```

### Lambda(익명 함수)

내가 Java를 사용하면서 가장 답답했던 점은 Function을 인자로 넘길 수 없었다는 것. 만약 우리가 과일을 파는 어플리케이션을 만들어서 과일을 필터링 하는 로직을 만든다고 가정을 할 때, 다음과 같은 문제에 봉착하게 될 수 있음

<div class="mermaid">
sequenceDiagram
    핸들러->>저장소: 사과만 보여주세요
    저장소-->>핸들러: 사과 여기 있어요!
</div>

우리는 이런 요구 사항이 있으면 다음과 같은 메서드를 구성하게 됨

```java
private List<String> fruits = List.of(
        "사과", "청사과", "서양배", "배", "포도", "청포도"
);

public List<String> extractApples() {
    
    return this.fruits.stream()
            .filter(과일 -> 과일.contains("사과"))
        .toList();
}
```

만약 다음과 같은 요구사항이 온다면?

<div class="mermaid">
sequenceDiagram
    핸들러->>저장소: 배만 보여주세요
    저장소-->>핸들러: 배 여기 있어요!
</div>

<div class="mermaid">
sequenceDiagram
    핸들러->>저장소: 포도만 보여주세요
    저장소-->>핸들러: 포도 여기 있어요!
</div>

```java
private List<String> fruits = List.of(
        "사과", "청사과", "서양배", "배", "포도", "청포도"
);

public List<String> extractApples() {
    
    return this.fruits.stream()
            .filter(fruit -> fruit.contains("사과"))
        .toList();
}

public List<String> extractPear() {
    
    return this.fruits.stream()
            .filter(fruit -> fruit.contains("배"))
        .toList();
}

public List<String> extractGrapes() {
    
    return this.fruits.stream()
            .filter(fruit -> fruit.contains("포도"))
        .toList();
}
```

이렇게 멍청하게 메서드를 일일히 작성하다보면 “아! 이렇게 하지 말고 조건을 인자로 받아야지!” 라고 생각하고 필터링 값을 인자로 올리게 됨

```java
private List<String> fruits = List.of(
        "사과", "청사과", "서양배", "배", "포도", "청포도"
);

public List<String> extractFruitsByTarget(String target) {
    
    return this.fruits.stream()
            .filter(fruit -> fruit.contains(target))
        .toList();
}
```

이정도면 끄떡 없겠지! 라고 생각하지만 개발자들의 생각보다 요구사항은 깐깐하고 우리들을 힘들게 함.

<div class="mermaid">
sequenceDiagram
    핸들러->>저장소: 사과랑 배 보여주시고 서양배는 뽑아오지 마세요
    저장소-->>핸들러: 나보고 어떻게 하라고!
</div>

사과랑 배를 요구하는데 또 서양배는 걸러달라고 하니 인자 하나만으로는 택도 없다는 것을 알게 됨

요구사항이 늘어날때마다 멍청하게 메서드를 늘려야지 라고 생각하지만, 우리는 이미 Stream의 필터 기능을 사용하고 있다는것을 다시 깨닫고 다음과 같은 방법을 사용함

```java
// 저장소
private List<String> fruits = List.of(
        "사과", "청사과", "서양배", "배", "포도", "청포도"
);

public List<String> extractFruitsByPredicate(Predicate<String> predicate) {
    
    return this.fruits.stream()
            .filter(predicate)
        .toList();
}
```

```java
// 핸들러
public void doJob() {
    ...

    Predicate<String> predicate = (fruit) -> 
            (fruit.contains("사과") || fruit.contains("배")) 
            && fruit.equals("서양배");

    List<String> fruits = fruitRepository.extractFruitsByPredicate(predicate);
    ...
}
```

위와 같이 익명함수를 Predicate라는 인터페이스에 담아 넘기는 것. 과일저장소는 이제 종류별로 메서드를 만들어서 제공 할 필요가 없어졌음. 하지만 우리는 이 문제를 해결하기 위해 익명함수를 Predicate에 집어넣고 그걸 인자로 넘기고 너무 많은 코스트를 사용함.

코틀린에서는 다음과 같이 표현 될 수 있음

```kotlin
// 저장소
private val fruits = listOf(
    "사과", "청사과", "배", "서양배", "포도", "청포도"
)

fun extractFruitsByPredicate(filter: (String) -> Boolean): List<String> {

        // case 1
    return fruits.filter(filter)
}

fun extractFruitsByPredicate(filter: (String) -> Boolean): List<String> {

        // case 2
    return fruits.filter { fruit -> filter(fruit) }
}

fun extractFruitsByPredicate(filter: (String) -> Boolean): List<String> {

        // case 3
    return fruits.filter { filter(it) }
}
```

눈에 띄게 바뀐점은, 인자로 받는 부분이 Predicate 인터페이스가 아닌, String을 받아 Boolean으로 변환되는 함수를 받는다는 것. 또한 stream을 호출할 필요도 없어졌다. 코틀린의 Collection에서 체이닝으로 제공하는 filter 함수를 사용하여, 인자로 Predicate 인터페이스가 아닌, 함수를 넣어주면 동작함.

호출하는 핸들러 코드를 살펴보면 다음과 같음

```kotlin
fun doJob() {

    ...
    val predicate = { fruit: String ->
        (fruit.contains("사과") || fruit.contains("배")) 
                && fruit != "서양배"
    }

    fruitRepository.extractFruitsByPredicate(predicate)
    ...

}
```

함수를 변수에 담아 넘기는 행위가 Java에서 `Predicate` 인터페이스에 익명함수를 담아 넘기는것과 비슷하여 얼핏 딱히 큰 장점이 없어보이지만, 코틀린에서는 마지막 인자값이 함수일 경우 함수를 호출하면서 함수의 body를 받는 기능을 제공함. 예를 들면 다음과 같음

```kotlin
fun doJob() {

    ...
    fruitRepository.extractFruitsByPredicate {
        (it.contains("사과") || it.contains("배")) && it != "서양배"
    }
    ...

}
```

fruitRepository의 extractFruitsByPredicate를 호출하면서 body를 넘길 수 있는 것을 확인 할 수 있음. 

이는 상기된 내용의 Optional vs Nullable 단에서 코틀린에서 제공하는 takeIf, let, also 등의 함수의 형태와 유사하다는 것을 확인 할 수 있음 

```kotlin
// StandardKt.class

@kotlin.internal.InlineOnly 
@kotlin.SinceKotlin 
public inline fun <T> T.also(block: (T) -> kotlin.Unit): T { 
    contract { /* compiled contract */ }; 
    /* compiled code */ 
}

@kotlin.internal.InlineOnly 
public inline fun <T> T.apply(block: T.() -> kotlin.Unit): T { 
    contract { /* compiled contract */ }; 
    /* compiled code */ 
}

@kotlin.internal.InlineOnly 
public inline fun <T, R> T.let(block: (T) -> R): R { 
    contract { /* compiled contract */ }; 
    /* compiled code */ 
}

@kotlin.internal.InlineOnly 
public inline fun <T, R> T.run(block: T.() -> R): R { 
    contract { /* compiled contract */ }; 
    /* compiled code */ 
}

@kotlin.internal.InlineOnly 
@kotlin.SinceKotlin 
public inline fun <T> T.takeIf(predicate: (T) -> kotlin.Boolean): T? { 
    contract { /* compiled contract */ }; 
    /* compiled code */ 
}

@kotlin.internal.InlineOnly 
@kotlin.SinceKotlin 
public inline fun <T> T.takeUnless(predicate: (T) -> kotlin.Boolean): T? { 
    contract { /* compiled contract */ }; 
    /* compiled code */ 
}
```