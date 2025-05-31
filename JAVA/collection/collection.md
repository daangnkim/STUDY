## Java Collections Framework

![alt text](JAVA/collection/image.png)

[이미지 출처](https://www.freecodecamp.org/news/java-collections-framework-reference-guide/)

자바 컬렉션 프레임워크(JCF)는 자료구조(List, Set, Map)와 알고리즘(Sorting, Searching, Inserting, Deleting)을 구현한 인터페이스 및 클래스 집합을 제공한다. 다른 표현으로 객체들의 집합을 관리하고 조작할 수 있게 도와준다.

그리고 다음 네 가지 장점을 가진다.

1. 이미 만들어진 자료 구조의 사용
2. 최적화된 성능을 가진 collection들 구현
3. collection들의 메서드 명들의 일관성 (`add`, `remove`, `size` 등)
4. 멀티 스레드 환경에서 사용 가능하도록 디자인된 collection들

여기서 컬렉션들이라고 하면 `ArrayList`, `PriorityQueue`, `HashSet` 등을 의미한다.

단점은 다음과 같다.

1. collection들은 기본적으로 `Thread Safe`하지 않다. 메뉴얼하고 싱크를 맞춰야한다.
2. `concurrent collection`과 `weak references`등의 이해하기 어려운 기능들도 존재한다
3. 다이나믹 리사이징이나 해싱으로 인해 몇몇 collection들은 성능상 오버헤드가 존재한다.

프레임워크 내 모든 인터페이스와 클래스들은 `java.util` 패키지에 존재한다. 또한 `Collection`과 `Collections`를 혼동하는데 `Collection`은 JCF 내의 인터페이스 `Collections`는 collection 원소를 대상으로 실행할 수 있는 정적 메서드들을 제공하는 클래스이다.

## Iterable vs Collection

JSF의 근본적인 인터페이스로 `Iterable`, `Collection이` 존재한다. `Iterable`은 `Collection` 내의 요소들을 순회할 수 있게 만들며, `Collection`은 `Iterable`을 상속받는다. `Iterable`은 오직 `Iterator()`라는 메서드 하나만 정의한다.

`Collection`은 `Iterable`의 모든 기능을 포함하면서 `add`, `remove` 등의 기능도 포함한다.

```java
// Importing necessary classes
import java.util.Collection;
import java.util.ArrayList;
import java.util.Iterator;

public class IterableVsCollection {
    public static void main(String[] args) {
        // Step 1: Create a Collection (ArrayList in this case)
        Collection<String> collection = new ArrayList<>();

        // Step 2: Add elements to the collection
        collection.add("Java");
        collection.add("Python");
        collection.add("C++");

        // Step 3: Iterate through the collection using Iterable interface
        Iterator<String> iterator = collection.iterator();
        while(iterator.hasNext()) {
            String language = iterator.next();
            System.out.println(language);
        }

        // Step 4: Use Collection methods directly
        System.out.println("Total languages: " + collection.size());
        System.out.println("Is Python in collection? " + collection.contains("Python"));
    }
}

// 출력
// Java
// Python
// C++
// Total languages: 3
// Is Python in collection? true
```

### Java Collection Interface

서브 인터페이스로 `List`, `Set`, `Queue`가 존재한다.

### List Interface

중복을 허용하는 정렬된 컬렉션으로, `ArrayList`, `LinkedList`, `Vector`, `Stack`이 존재한다.

### Set Interface

중복을 허용하지 않는, 정렬되지 않은 컬렉션으로, `HashSet`, `LinkedHashSet`, `TreeSet`등이 존재한다.

### Map Interface

키와 값으로 저장하는 컬렉션으로, 중복 키는 허용을 하지 않으며, `HashMap`, `LinkedHashMap`, `TreeMap` 등이 존재한다.

### Queue Interface

`PriorityQueue`, `LinkedList`가 존재한다.

## References

[Java Collections Framework](https://www.programiz.com/java-programming/collections)

[Java Collections Framework Explained | by Adarsh GS | Medium](https://medium.com/@adarshgs.909/java-collections-framework-explained-78c80f1cb6fc)

[Difference Between Iterable and Collection in Java](https://www.javaguides.net/2023/11/iterable-vs-collection-in-java.html)

[How to Use the Java Collections Framework – A Guide for Developers](https://www.freecodecamp.org/news/java-collections-framework-reference-guide/)
