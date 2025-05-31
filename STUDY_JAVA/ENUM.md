컴파일 타임에 status code, type, category와 같이 가능한한 모든 값을 알수 있을 때 사용된다.

내부적으로 class type으로 구현되어 new와 호출, constructor 생성, instance variable이나 method 등의 추가가 가능하지만 다른 클래스를 상속받거나 부모 클래스가 될수는 없다.

모든 enum은 암묵적으로 `public static final`이다.

enum 상수는 실제로 해당 enum 타입의 객체이다.

enum을 정의할 때 나열하는 각 상수들은 그 enum 클래스의 인스턴스가 된다.

```java
enum Day {
    MONDAY, TUESDAY, WEDNESDAY // 각각은 Day 타입의 객체
}

Day today = Day.MONDAY; // MONDAY는 Day 클래스의 객체
```

- [enum in Java | GeeksforGeeks](https://www.geeksforgeeks.org/enum-in-java/)
