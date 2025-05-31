## final

1. final로 선언된 변수는 바꿀 수 없다.

```java
class Main {
  public static void main(String[] args) {

    // create a final variable
    final int AGE = 32;

    // try to change the final variable
    AGE = 45;
    System.out.println("Age: " + AGE);
  }
}

// cannot assign a value to final variable AGE
//     AGE = 45;
//     ^
```

2. final로 선언된 변수는 할당을 바로 해야하지만 예외적으로 constructor에서 할당할 수 있다.

```java
public class UserService {
    private final UserRepository userRepository;

    @Autowire
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

3. final로 선언된 메서드는 자식 클래스에서 오버라이드 할 수 없다.

```java
class FinalDemo {
    // create a final method
    public final void display() {
      System.out.println("This is a final method.");
    }
}

class Main extends FinalDemo {
  // try to override final method
  public final void display() {
    System.out.println("The final method is overridden.");
  }

  public static void main(String[] args) {
    Main obj = new Main();
    obj.display();
  }
}
```

4. final로 선언된 클래스는 다른 클래스가 상속받을 수 없다.

```java
// create a final class
final class FinalClass {
  public void display() {
    System.out.println("This is a final method.");
  }
}

// try to extend the final class
class Main extends FinalClass {
  public  void display() {
    System.out.println("The final method is overridden.");
  }

  public static void main(String[] args) {
    Main obj = new Main();
    obj.display();
  }
}
```

## Referneces

[Java final keyword (With examples)](https://www.programiz.com/java-programming/final-keyword)
