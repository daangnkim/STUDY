java.util.Date 클래스는 기존에 존재했었고, Java 8 버전부터 java.time 패키지에서 `LocalDate`, `LocalTime`, `LocalDateTime`을 제공하기 시작했다.

java.time이 해결하는 문제는 다음과 같다.

1. time과 date를 명확히 구분한다.

   기존 `Date`클래스는 date와 time을 명확히 구분하지 못한다. 하지만 java.time 패키지는 이를 명확히 구분할 수 있는 클래스들을 제공한다.

   ```java
    LocalDate today = LocalDate.now();
    LocalTime timeNow = LocalTime.now();
    LocalDateTime currentDateTime = LocalDateTime.now();
   ```

2. Thread-safe하다.

   기존 java.util.Date는 mutable했다. 이는 멀티 스레드 환경에서 예상치 못한 버그로 이어졌다.
   하지만 java.time 패키지의 클래스들은 불변성을 지킨다. 한번 생성되면 변하지 않고, 추가적인 synchronization도 필요하지 않다.

3. 높은 정확도와 유연성

   java.util.Date는 날짜를 더하거나 빼는 메서드가 존재하지 않는다. 그러므로 날짜 연산 과정을 겪어야하는데 매우 복잡하다.

   반면에 java.time 패키지 내 클래스들은 날짜 조작과 관련한 메서드들도 존재하기 때문에 쉽고 간단하게 날짜 조작이 가능하다.

   ```java
   LocalDate nextWeek = LocalDate.now().plusDays(7);
   ```

그럼에도 불구하고 java.util.Date를 써야하는 경우는 다음과 같다.

1. Legacy System 이거나 old codebase인 경우
2. old한 database의 경우 java.util.Date를 요구할 수 있다.

이러한 상황을 대비하여 old date 클래스와 new date 클래스간의 변환 방법을 알아두는 것이 좋다.

```java
// Convert java.util.Date to LocalDate:
import java.time.LocalDate;
import java.util.Date;

public LocalDate convertToLocalDateViaInstant(Date dateToConvert) {
return dateToConvert.toInstant()
        .atZone(ZoneId.systemDefault())
        .toLocalDate();
}

// Convert LocalDate to java.util.Date:
import java.time.LocalDate;
import java.time.ZoneId;
import java.util.Date;

public Date convertToDateViaInstant(LocalDate dateToConvert) {
   return Date.from(dateToConvert.atStartOfDay()
          .atZone(ZoneId.systemDefault())
          .toInstant());
}
```

### References

[Why java.time.LocalDate and LocalDateTime Are Better Than java.util.Date | by RAHUL PODDAR | Medium](https://medium.com/@rpoddar05/why-java-time-localdate-and-localdatetime-are-better-than-java-util-date-f7a444c95d59)
