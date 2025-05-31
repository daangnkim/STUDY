## IoC와 DI

```java
public class Maths {
    Addition addition = new Addition();

    public void addSomething(int number) {
        addition.add(number)
    }
}
```

Maths 클래스는 Addition 클래스에 직접적으로 의존중이다. 우리는 Maths 인스턴스가 생성되면서 Addition 클래스를 제어할 수 없다.

IoC는 객체의 생성과 관리를 개발자가 아닌 framework에게 맡기는 것을 의미한다. 그리고 IoC의 한 가지 방법으로 DI가 존재한다.

```java
public class Maths {
    Addition addition;

    public void setAdditionField(Addition addition) {
        this.addition = addition;
    }

    public void addSomething(int number) {
        addition.add(number);
    }
}
```

Maths 클래스와 Addition 클래스간의 직접적인 연결을 제거하고, Maths가 어떤 addition에게 의존하는지 우리가 제어할 수 있게된다.

## Bean이란?

Bean은 어떤 클래스에도 주입할 수 있는 객체이며, 이 객체의 인스턴스화나 의존성에 대해 걱정할 필요가 없습니다. 이러한 부분은 프레임워크 자체에서 처리해 주기 때문입니다

Bean은 IoC Container(혹은 Application Context)에서 관리된다.

## Bean 생성 방법

여러가지 방법 중 하나는 @Component를 이용하는 것이다. Addition 클래스를 다음과 같이 Bean으로 만들 수 있다.

```java
@Component
public class Addition {
    public void add(int number) {
        System.out.println("Added " + number + "!");
    }
}
```

Addition 클래스는 application context에 존재하기 때문에, 이 bean을 @Autowired을 이용하여 다음과 같이 주입이 가능하다.

```java
@Component
public class Maths {

    @Autowired
    Addition addition;

    public void addSomething(int number) {
        addition.add(number)
    }
}
```

@Autowired는 application context에서 bean을 찾아 다른 bean에 주입하는 역할을 한다. auto-wiring 로직이 동작하려면 bean이어야 하므로 Maths 클래스 역시도 @Component 어노테이션을 적용해야한다.

다시 정리하면 DI는 클래스 내부에서 특정 클래스를 초기화하는 것이 아니라, 외부에서 주입하는 것을 의미한다.

## 동일한 타입의 bean이 존재하는 상황

만약 동일한 타입의 bean이 여러개 존재한다면 `@Qualifier`를 `@Autowired`와 함께 이용할 수 있다.

```java
@Autowired
@Qualifier("specificBean")
private MyService myService;
```

혹은 `@Primary`를 사용하면 `@Qualifier`를 사용하지 않았을 때 기본적으로 사용되는 bean을 지정할 수 있다.

```java
@Primary
@Service
public class DefaultServiceImpl implements MyService {
    // implementation
}

@Service
public class OtherServiceImpl implements MyService {
    // implementation
}
```

## Stereotype Annotation

Bean을 생성할 수 있는 Annotation을 Stereotype Annotation이라고 부르고, 이 Annotation에는 @Controller, @Service, @Repository, @Component가 존재한다.

1. @Component

- 특정 도메인에 속하지 않는 utility class, helper class 등 일반적으로, 애플리케이션 전반적으로 재사용이 가능한 클래스에 사용한다.

```java
package com.bank.utils;

import org.springframework.stereotype.Component;

@Component
public class TransactionUtility {

    // Method to validate transaction amount against a limit
    public boolean validateTransactionLimit(double amount, double limit) {
        return amount <= limit;
    }

    // Method to calculate transaction fee based on amount
    public double calculateTransactionFee(double amount) {
        return amount * 0.02; // 2% transaction fee
    }

    // Method to format transaction details
    public String formatTransactionDetails(String transactionId, double amount) {
        return String.format("Transaction ID: %s, Amount: %.2f", transactionId, amount);
    }
}
```

2. @Service

- 비지니스 로직을 수행하는 클래스에 사용한다. three-tier 아키텍쳐에서 controller와 repository 사이에 존재하는 Service 레이어를 나타내게된다.

```java
@Service
public class UserService {
    public User getUserById(Long id) {
        // Business logic to retrieve a user by ID
    }
}
```

3. @Repository

- DAO를 수행하는 역할을 하게된다. 즉, 데이터 접근과 관련한 기능을 하며, 이를 통해 비지니스 로직과 데이터 접근간 관심사 분리가 가능해진다.

```java
@Repository
public class UserRepository {
    public User findUserById(Long id) {
        // Data access logic to retrieve a user by ID
        // Automatic exception translation provided by @Repository
    }
}
```

4. @Controller

- presentation layer의 역할을 수행한다. 사용자의 요청을 받아 응답을 내려주는 역할 및 URL 매핑, 화면과의 상호작용 등을 담당한다.

```java
@Controller
@RequestMapping("/users")
public class UserController {
    private final UserService userService;

    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable Long id) {
        User user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    }
}
```

## Spring Bean의 라이프사이클

1. Instantiation

- bean의 인스턴스 생성

2. Dependency Injection

- bean에 의존성 주입

3. Initialization

- @PostConstruct와 같은 커스텀 초기화 메서드 호출

4. Usage

- 애플리케이션에 의해 사용

5. Destruction

- 컨테이너가 셧다운하기전에 @PreDestroy와 같은 커스텀한 destory 메서드 호출

[Spring Boot and Beans ! 🍃. Spring boot is one of the most widely… | by Yogesh Ananda Nikam | Geek Culture | Medium](https://medium.com/geekculture/spring-boot-and-beans-b2797b488fde)

[Stereotype Annotations in Spring framework | by Jawad Zaarour | Medium](https://medium.com/@zjawad333/stereotype-annotations-in-spring-framework-b65f0e7fa377)

[Understanding the Differences and Usages of @Service, @Repository, @Controller, and @Component Annotations in Spring Boot | by Davoud Badamchi | Medium](https://medium.com/@davoud.badamchi/spring-boot-annotationsunderstanding-the-differences-and-usages-of-service-repository-f94e076bbcdc)
