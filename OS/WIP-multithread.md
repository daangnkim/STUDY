스레드는 프로세스 내의 가장 작은 실행 단위이다.
멀티스레딩 맥락에서의 스레드는 CPU가 독립적으로 실행하는 명령어의 나열이다.
프로세스는 여러개의 스레드를 가질 수 있고, 각 스레드들은 동일한 메모리를 공유한다.

### 멀티스레드의 장점

1. 효율적인 자원 사용

   단일 스레드는 한 번에 하나의 CPU 코어만 사용이 가능하지만, 멀티 스레딩을 사용함녀 여러 CPU 코어를 동시에 활용 가능하다.

2. 성능 향상

   하나의 작업을 여러개로 쪼개서 여러 스레드에서 실행하기 때문에 더 빠르다.

### Java에서 멀티 스레드를 사용하는 방법

1. `Thread` 클래스 상속받기

   ```java
   public class MyThread extends Thread {
   public void run() {
       // Code to be executed in the new thread
       for (int i = 0; i < 5; i++) {
           System.out.println("Thread " + Thread.currentThread().getId() + ": " + i);
       }
   }
   }

   public class Main {
   public static void main(String[] args) {
       MyThread thread1 = new MyThread();
       MyThread thread2 = new MyThread();
       thread1.start(); // Starts a new thread
       thread2.start(); // Starts another thread
   }
   }
   ```

2. `Runnable`인터페이스 구현하기

   ```java
   public class MyRunnable implements Runnable {
       public void run() {
           // Code to be executed in the new thread
           for (int i = 0; i < 5; i++) {
               System.out.println("Thread " + Thread.currentThread().getId() + ": " + i);
           }
       }
   }

   public class Main {
       public static void main(String[] args) {
           Thread thread1 = new Thread(new MyRunnable());
           Thread thread2 = new Thread(new MyRunnable());
           thread1.start(); // Starts a new thread
           thread2.start(); // Starts another thread
       }
   }
   ```

둘의 차이점은 다음과 같다.

- `Thread` 클래스를 상속하면 다른 클래스를 상속할 수 없다. 그러므로 상속의 이점을 놓치게된다. `Runnable` 인터페이스 구현 방식은 다른 클래스를 상속할 수 있다.

- `Thread` 클래스를 상속하면 각각의 thread는 유니크한 객체를 생성해낸다. 하지만 `Runnable`을 구현하면 동일한 객체를 여러 스레드가 공유하게된다. `Runnable`이 생성하는 객체의 개수가 더 적기 때문에 가능하면 `Runnable`을 쓰자

### Spring Boot에서의 멀티 스레딩

여러가지 방법 중 하나는 `@Async`를 이용하는 것이다.

`@Aysnc`가 있으면 호출될 때 별도의 스레드에서 실행된다. 메인 스레드는 해당 메소드의 실행 완료를 기다리지 않고 다른 작업을 실행할 수 있다. 즉, 메인 스레드의 블록킹을 방지한다.

`@Async`를 사용하기 위해서는 `@EnableAsync`를 Configuration 클래스에 적용해야한다.

```java
package com.example.asyncdemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableAsync;

@SpringBootApplication
@EnableAsync // Enable asynchronous processing
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

package com.example.asyncdemo.service;

import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class AsyncService {

    // Annotating the method with @Async to run it in a separate thread
    @Async
    public void processTask() throws InterruptedException {
        System.out.println("Started processing task in thread: " + Thread.currentThread().getName());

        // Simulate a time-consuming task (e.g., a long computation, IO operation, etc.)
        TimeUnit.SECONDS.sleep(5); // Simulate a delay of 5 seconds

        System.out.println("Finished processing task in thread: " + Thread.currentThread().getName());
    }

    // You can return a Future, CompletableFuture, or ListenableFuture if you want to capture the result
    @Async
    public void anotherAsyncTask() {
        System.out.println("Started another task in thread: " + Thread.currentThread().getName());
    }
}
```

### References

[Introduction to Multithreading : Java & Spring Boot | by Nidhi Upreti | Medium](https://medium.com/@nidhiupreti99/introduction-to-multithreading-java-spring-boot-b4930b73f302#:~:text=Multithreading%20in%20Spring%20Boot%20allows,enable%20multithreading%20in%20your%20applications.)
