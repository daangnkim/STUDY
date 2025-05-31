
- Spring Boot는 `JPA(Jakarta Persistence API)`를 이용하여 `객체 관계 맵핑(ORM)`을 구현
- ORM은 자바 클래스와 데이터베이스 테이블을 매핑
- Java Persistence API는 2019년을 기점으로 Jakarta Persistence API로 이름이 변경됐다.
- JPA는 명세이고, JPA를 구현한 프로바이더들은 여러가지가 있는데, `Hibernate`는 이중 하나이다. Spring Boot의 디폴트 JPA는 Hibernate이다.
- `Spring Data`는 relational 혹은 non-relational 데이터베이스와의 인터랙션을 쉽게 만드는 프레임워크이다.
- `Spring Data JPA`는 JPA 위에 존재하는 레이어로(그러니까 `Hibernate`에 추가 추상화 계층을 제공하는 레이어로), 데이터베이스 작업을 간단하게 만들어주는 도구이다. JPA를 사용하는 경우 CRUD가 아래처럼 단순해진다.

```java
// Spring Data JPA 사용 예제
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // 기본 CRUD 메소드는 자동으로 제공됨

    // 다음과 같이 메소드 이름만으로 쿼리 생성 가능
    List<User> findByUsername(String username);
    User findByEmail(String email);

    // @Query 어노테이션으로 직접 JPQL 작성 가능
    @Query("SELECT u FROM User u WHERE u.email LIKE %:domain%")
    List<User> findByEmailDomain(@Param("domain") String domain);
}
```

[Hibernate Relations, Cascade Types, Fetch Types and Orphan Removal | by Berat Yesbek | Medium](https://beratyesbek.medium.com/hibernate-relations-cascade-types-fetch-types-and-orphan-removal-ad9681758843)

[java - What's the difference between Hibernate and Spring Data JPA - Stack Overflow](https://stackoverflow.com/questions/23862994/whats-the-difference-between-hibernate-and-spring-data-jpa)

[Introduction to the Spring Data Framework | GeeksforGeeks](https://www.geeksforgeeks.org/introduction-to-the-spring-data-framework/)

[Difference between JPA and Spring Data JPA | GeeksforGeeks](https://www.geeksforgeeks.org/difference-between-jpa-and-spring-data-jpa/)
