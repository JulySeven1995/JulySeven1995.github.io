---
title: "Spring Bean & 주입(Injection)"
categories:
  - Dev
tags:
  - Spring
  - Spring Boot
layout: single
---

Bean & Injection

- `Bean`

  `Spring Framework` 에서 `Bean` 이란 `Spring IoC` 컨테이너가 관리하는 자바 객체를 뜻한다. 간단하게 설명하자면, 자바의 `new` 연산자를 통해 생성하는 객체는 `Bean` 이라고 말할 수 없다. 간혹 `Spring`  기반의 프로젝트에서 비즈니스 로직을 담고있는 메서드를 가진 클래스에 아래와 같은 코드를 볼 수 있을 것이다.

  ```java
  @Autowired
  private TestBean testBean;
  
  public void 메서드() {
  
  	this.testBean.doSomething();
  }
  ```

  `testBean` 이라는 `TestBean`의 객체 변수는 `Spring IoC` 컨테이너를 통해 객체를 `주입` 받아 사용되는 것이다.

  그렇다면 '어떻게' `Spring IoC` 컨테이너에 `Bean`을 등록하는지 대표적인 방법은 다음과 같다.

  1. Java Annotation

     `Annotation`으로 `Bean`을 등록 할 때에는, 등록하려는 클래스에 `@Component`, `@Controller`, `@Service`, `@Repository` 어노테이션이 필연적으로 명시되어야 한다. 해당 `Annotation`을 기재하면서, `Spring IoC` 컨테이너에 이 클래스를 빈으로 등록하겠다 라고 명시하는것이다.

     ```java
     @Component
     public class TestBean {
     		
     		private String name;
     		...
     }
     
     @Controller
     public class TestController {
     		...
     }
     
     @Service
     public class TestService {
     		...
     }
     ```

     > `Spring` 개발자 생활 꿀팁

      `@Component`, `@Controller`, `@Service`, `@Repository` 는 다 똑같다?

     반은 맞고 반은 틀리다. 기능적으로는 모든 `Bean` 객체에 어떤 어노테이션을 사용하던, 사실상 전부 `@Component`를 사용해도 로직을 구성하는데에 별 문제는 없다. 하지만 `Spring Framework` 자체는 협업과 생산성 향상을 위해서 만들어진 프레임워크이며(이 외에도 안정성, 보안성 등등 많은 것이 있다.), 개발 프로젝트에서는 동업자들끼리 '언어'가 아닌 '코드'로 소통해야 하는 경우가 빈번하다.  기능이 되면 그냥 그거 쓰지 하고 넘어가지 말고, 반드시 `Bean` 이 무슨 역할을 하고 어떤 레이어에 속해있어야 하는지 어노테이션으로 상기시켜주자.
     ![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/5fb26a4e-0a25-4bc1-867e-dcbb18df7fc4/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20210422%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20210422T154831Z&X-Amz-Expires=86400&X-Amz-Signature=f5f1532e97a4d3c52cd3a121da7cf27038e5991b4c04a41e5d122db67be9c14b&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22)

  2. XML Config

     단도직입적으로 말하면, 필자는 해당 방법을 선호하지 않는다. 하지만 아직도 많은 회사들의 웹서비스에는 `XML Config` 형식의 Bean이 등록되어 있을 수 있고,  개선 해야 되는 프로젝트에도 있을 확률이 있기 때문에 쓸줄은 몰라도 '읽을줄은 알아야' 하는 등록 방법이다.

     스프링 기반의 프로젝트에서, 주로 아래와 같은 `xml` 파일이 존재 할 것이다. (파일명은 다 다를 수가 있다.) `{projectDir}/src/main/resources/application.xml`

     주로 다음과 같은 형식으로 무언가 `<bean>` 태그들이 존재한다.

     ```xml
     <?xml version="1.0" encoding="UTF-8" ?>
     <beans xmlns="http://www.springframework.org/schema/beans"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
     
         <bean id="testBean" class="com.codewise.component.TestBean">
             <property name="name" value="최재호"/>
         </bean>
     </beans>
     ```

     다음과 같이 `xml`에  `TestBean` 등록 후, 

     ```java
     public class DemoApplication {
     
         public static void main(String[] args) {
             ApplicationContext ctx = new ClassPathXmlApplicationContext("application.xml");
     
             TestBean testBean= ctx.getBean(TestBean.class);
             System.out.println(testBean.getName());
         }
     }
     ```

     불러와서 `Name`을 출력하면, `<property>` 태그에서 입력한 이름이 출력된다.

- `Dpendency Injection`

  `Spring Framework`에서 의존성 주입(`Dependency Injection` 이하 `DI` )이란, `Bean` 으로 등록된 클래스를 `Ioc` 컨테이너로부터 넘겨받는것을 의미한다. 절대 로직상에서 '생성' 되지 않는다.

  `DI` 의 방법은 세가지가 있는데 다음과 같다.

  1. `Field Injection`

     실무에서 가장 자주 보이는 `DI` 방식이며, 사실 따지고보면 제일 편리한 방법이다. 위의 `Bean` 설명에 있던 코드를 보면, 다음과 같이 사용한다.

     ```java
     @Service
     public class TestService {
     
     	@Autowired
     	private TestBean testBean;
     
     }
     ```

     하지만 이 방법은 추천하지 않으며, 이유는 아래의 `Constructor Injection` 에서 후술하도록 한다.

  2. `Setter Injection`

     필자가 개발자로서의 연차는 오래되지 않았고, 경험도 적지만, 실무에서 한번도 보지 못한 방식이다. 

     ```java
     @Service
     public class TestService {
     
     	private TestBean testBean;
     	
     	@Autowired
     	public void setTestBean(TestBean testBean) {
     		this.testBean = testBean;
     	}
     }
     ```

     해당 방법도 `Field Injection` 과 함께 추천하지 않는다. 이유는 마찬가지로 후술한다.

  3. `Constructor Injection`

     `Bean` 의 생성자에서 주입을 받는 방법이다. 코드의 예시는 다음과 같다.

     ```java
     @Service
     public class TestService {
     
     	private final TestBean testBean;
     	
     	public void TestService(TestBean testBean) {
     		this.testBean = testBean;
     	}
     }
     ```

     `Constructor Injection` 을 사용해야 하는 이유는 다음과 같다.

     1. `final` 선언 가능

        처음부터 지금까지 글을 잘 읽었다면 위의 두 `DI` 극명하게 다른 차이점이 보일것이다. 그것은 `final` 선언이 가능해 졌다는 것이다. 객체를 `final` 로 선언함으로, 런타임에서 해당 객체의 불변성을 '보장' 해 줄수 있다. 

     2. 순환참조 방지

        `Constructor Injection` 방식과, `Field Injection` 는 `Bean` 을 주입하는 순서 자체가 다르다. `Field Injection` 에서는 `Bean` 을 생성 한 후, 주입하려는 `Bean` 을 찾는 방식이고, `Constructor Injection` 방식은, `Arguments`의 `Bean`을 먼저 찾는다. 닭이 먼저냐, 달걀이 먼저냐의 차이지만, `Constructor Injection` 에서는 `Runtime`으로 넘어가기 전 순환참조 에러를 캐치해낼 수 있다.

     3. 테스트 코드 작성 용이

        `Test` 코드를 작성 할 때,  `Field Injection` 방식을 사용한 `Bean` 을 테스트 하려면, `Spring Ioc` 컨테이너를 참조해야하고, 불러와야하는 의존성이 한두개가 아닌데, `Constructor Injection` 방식의 `Bean` 은 단순히 `new` 연산자를 통해 객체를 생성하여(인자로 사용되는 `Bean`은 `Mock`객체로 넣어주자) 테스트를 수월하게 할 수 있다.