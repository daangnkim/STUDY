IoC는 객체의 생성과 관리를 개발자가 아닌 framework에게 맡기는 것을 의미한다. 전통적으로 객체의 생성과 관리를 개발자가 담당했으나, IoC는 이 책임을 프레임워크에게 전가한다. 그리고 IoC Container는 스프링 부트의 코어 컴포넌트로, (1) bean의 라이프 사이클을 다루며, (2) bean의 생성과 인스턴스화하고, (3) 필요한 곳에 주입하는 역할을한다.

IoC를 구현하는 여러가지 방법으로 (1) Service Locator (2) Events (3) Delegate (4) DI가 존재하며, spring boot는 DI를 이용하여 IoC를 구현한다.

## IoC Container의 종류

1. BeanFactor

- 의존성 주입을 제공하는 가장 기본적인 IoC Container다.

2. ApplicationContext

- 더 진보된 Container로 이벤트 전파, bean 생성을 위한 선언적 메커니즘등을 제공한다.

## DI의 종류

1. Constructor Injection

- constructor를 통해서 의존성을 주입한다.

2. Setter Injection

- setter 메서드를 통해서 의존성을 주입한다.

3. Field Injection

- @Autowired를 이용하여 field에 바로 의존성을 주입한다.

## 스프링 부트는 @Autowired를 권하지 않는다.

@Autowired는 네 가지 문제점을 가지고 있다.

1. 숨겨진 의존성
   의존성이 알아서 주입되기 때문에 겉으로 잘 드러나지 않는다. 이는 클래스가 어떤 기능을 하는지 파악하기 어렵게 만든다.

2. 불변성 이슈
   의존성이 `mutable`하게 된다. 이는 컴포넌트의 불변성을 감소시키며 버그를 추적하기 어렵게 만든다.

3. 테스트의 어려움
   테스트를 셋업할 때 mocking하거나 직접 필드를 주입해야 하는 등의 보일러 코드를 작성해야한다.

그래서 스프링 개발팀은 `Constructor Injection`을 추천한다.

## 왜 `Constructor Injection`을 추천할까?

1. 명시적 의존성
   클래스의 인스턴스 생성시 필요한 의존성을 모두 주입하도록 강제된다.

2. 불변성 유지
   `Constructor Injection`은 `final` 필드의 사용을 권장하고, `final`은 불변성을 보장한다.

3. 향상된 테스트
   constructor에 목킹한 의존성을 단순히 패싱하기만 하면 된다.

## `@RequiredArgsConstructor`의 등장

lombok의 `@RequiredArgsConstructor`를 사용하면 boilerplate 코드를 제거하면서 `Constructor Injection`구현이 가능해진다.

```java
@Service
@RequiredArgsConstructor
public class MyService {

    private final MyRepository myRepository;
}
```

## 그럼에도 불구하고 `@Autowired`가 유용한 경우

`@Autowired`는 deprecated된 것도 아니고, 완전히 사용하지 말라고 고지한 것도 아니다.

경우에 따라서 의존성이 선택적으로 존재하거나, 객체 생성 후 값을 주입해야 할 때 setter 주입을 사용할 수 있는데, 이때 다음과 같이 `@Autowired`를 이용할 수 있다.

```java
@Service
public class MyService {

    private MyRepository myRepository;

    @Autowired
    public void setMyRepository(MyRepository myRepository) {
        this.myRepository = myRepository;
    }
}
```

## Autowired를 이용한 예제

알람 서비스는 email과 SMS를 이용하고, 이러한 서비스는 환경 설정과 의존성이 필요하다.

1. Notification Service를 위한 인터페이스 만들기

```java
public interface NotificationService {
    void sendNotification(String message);
}
```

2. Email과 SMS 각각을 위한 Notification Service 구현

```java
import org.springframework.stereotype.Service;

@Service
public class EmailNotificationService implements NotificationService {

    @Override
    public void sendNotification(String message) {
        System.out.println("Sending email notification: " + message);
    }
}
```

```java
import org.springframework.stereotype.Service;

@Service
public class SmsNotificationService implements NotificationService {

    @Override
    public void sendNotification(String message) {
        System.out.println("Sending SMS notification: " + message);
    }
}
```

3. Notification Controller 정의하고 constructor based로 의존성 주입하기

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class NotificationController {

    private final EmailNotificationService emailNotificationService;
    private final SmsNotificationService smsNotificationService;

    @Autowired
    public NotificationController(EmailNotificationService emailNotificationService,
                                  SmsNotificationService smsNotificationService) {
        this.emailNotificationService = emailNotificationService;
        this.smsNotificationService = smsNotificationService;
    }

    @GetMapping("/sendEmail")
    public String sendEmail(@RequestParam String message) {
        emailNotificationService.sendNotification(message);
        return "Email notification sent!";
    }

    @GetMapping("/sendSms")
    public String sendSms(@RequestParam String message) {
        smsNotificationService.sendNotification(message);
        return "SMS notification sent!";
    }
}
```

4. Application Setup

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class NotificationServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(NotificationServiceApplication.class, args);
    }
}
```

애플리케이션이 시작되면 Spring Boot는 자동으로 Stereotype Annotation을 스캔한뒤 bean으로 등록한다.

## References

[Understanding IoC Container in Spring Boot with a Real-Time Project Example | by A cup of JAVA coffee with NeeSri | Medium](https://neesri.medium.com/understanding-ioc-container-in-spring-boot-with-a-real-time-project-example-6e28519ab444)

[Spring Doesn’t Recommend @Autowired Anymore | by Dulanjaya Sandaruwan | Medium](https://medium.com/@dulanjayasandaruwan1998/spring-doesnt-recommend-autowired-anymore-05fc05309dad#:~:text=The%20Problem%20with%20%40Autowired&text=This%20reduces%20the%20immutability%20of,field%20injection%20can%20be%20tricky.)
