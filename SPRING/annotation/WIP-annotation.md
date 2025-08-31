## GenerationType IDENTITY, SEQUENCE, TABLE

##### GenerationType.IDENTITY
auto-incremented database column에 의존한다.
데이터베이스가 auto-incremented column 기능을 제공해야한다.

[Spring Boot Annotation @GeneratedValue | by Hrishi D | Medium](https://medium.com/@swapnildalimbkar01/spring-boot-annotation-generatedvalue-55e257fbd3d9)


****
## 양방향 관계

팀과 선수의 관계는 양방향 관계이다. 소유 측(Owning Side)는 데이터베이스에 외래 키를 가지는 쪽이다. 즉 관계를 저장하는 쪽이 소유 측이다. 역방향 측(Inverse Side)는 자바 코드에서만 관계를 표현하고, 실제 데이터베이스에는 영향을 주지 않는다.
팀과 선수에서 선수는 team_id가 존재하므로 관계를 저장하는 소유 측이며, 팀은 선수 목록을 참조하지만 이 관계를 나타내는 컬럼이 존재하지 않는다. mappedBy 속성으로 소유 측을 지정 가능하다.

## @OneToOne, @OneToMany, @ManyToOne, @ManyToMany

테이블간의 논리적인 연결을 형성한다.

#### @OneToMany
- Inverse Side를 나타낸다. Person Entity와 Address Entity를 따진다면, Person은 @OneToMany 어노테이션을 기입하고, Address Entity에는 @ManyToOne 어노테이션을 기입한다.
- parent-child 관계로, 하나의 entity가 여러 entity와 관계를 가진다.
- mappedBy, cascade, fetch 속성을 가진다.
	- mappedBy는 child entity 내에서 관계를 저장하는 `필드`를 기입한다.
#### @ManyToMany
- 여러 Entity가 여러 Entity와 관계를 가진다.
- 학생과 강의간의 관계가 @ManyTo@Many 관계이다.

#### @OneToOne
- 하나의 Entity가 하나의 Entity와 관계를 가진다.
- 예를들면 사람과 여권과의 관계이다.

[Understanding JPA Relationships: @ManyToMany, @OneToMany, and @OneToOne | by Davoud Badamchi | Feb, 2025 | Medium](https://medium.com/@davoud.badamchi/understanding-jpa-relationships-manytomany-onetomany-and-onetoone-ab84aa1953c1)

****

## cascade

cascade는 parent entity에서 child entity로 전파되는 `persist`, `remove` persistence 동작을 기입한다.
@OneToMany에서 fetch type은 기본적으로 lazy loading이다.


****
## FetchType

1. Lazy Fetch Type
   Hibernate가 entity마다 proxy를 생성한다. 관련한 데이터를 호출할 때 가져온다. 대용량 데이터가 존재하는 one-to-many, many-to-many에서 유용하다.

2. Eager Fetch Type
   proxy 없이 곧 바로 데이터를 가져온다. one-to-one 관계에선 이롭다.

[Hibernate Relations, Cascade Types, Fetch Types and Orphan Removal | by Berat Yesbek | Medium](https://beratyesbek.medium.com/hibernate-relations-cascade-types-fetch-types-and-orphan-removal-ad9681758843)

[Working with JPA FetchTypes. When working in SEF, we get many… | by Piumal Rathnayake | Medium](https://piumal1999.medium.com/working-with-jpa-fetchtypes-dc09386cf2ea)

****
## @email annotation

@Email은 EmailValidator와 연결된다. EmailValidator는 값을 가져와서 pattern matching을 실행한다. null인경우 validation을 건너띄기 때문에 필요에 따라 @NotNull이 존재해야한다.

[How Spring Boot Handles Validation Annotations | Medium](https://medium.com/@AlexanderObregon/how-spring-boot-handles-validation-annotations-33b987c1a5cb)

****
## @CreationTimestamp vs @CreatedDate

@CreatedDate는 Spring Data Annotation으로, Spring이 지원하는 모든 저장소(JPA, JDBC, R2DBC, MongoDB, Cassandra)에서 사용이 가능하다.

@CreationTimestamp는 Hibernate Annotation으로, Hibernate에서만 사용이 가능하다.

[What's the difference between @CreationTimestamp and @CreatedDate in Spring boot jpa? - Stack Overflow](https://stackoverflow.com/questions/66149224/whats-the-difference-between-creationtimestamp-and-createddate-in-spring-boot)

****

## @Entity vs @Table

`@Entity`는 자바 클래스를 Entity로 만든다. Entity는 관계형 데이터 베이스에서 테이블을 나타낸다. JPA에게 Entity 클래스의 인스턴스는 데이터베이스 테이블의 row에 매핑됨을 말하는 것이다.

`@Table`은 Entity가 데이터베이스의 어떤 테이블에 매핑돼야하는지 명시하기 위한 용도이다. `@Entity`와 함께 쓰인다.

클래스가 ORM 동작을 수행하게 만들려면 `@Entity`이 필수이다.

[@table vs @entity in Spring boot. In Spring Boot (and generally in Spring… | by Meet2sudhakar | Medium](https://medium.com/@meet2sudhakar/table-vs-entity-in-spring-boot-a84e092976fd)

****
## @AllArgsConstructor vs @RequiredArgsConstructor vs @NoArgsConstructor + @Data


참고로 constructor란, 인스턴스가 생성될때 호출되는 메서드로, 클래스와 동일한 이름을 가지고있다. 만약 constructor가 private인 경우, class 외부에서 new 연산자 호출이 불가하다.
default constructor는 어떠한 constructor도 기입하지 않았을 때 자바 컴파일러가 자동으로 생성되는 no-args constructor이다.
[Java Constructors (With Examples)](https://www.programiz.com/java-programming/constructors)



![](SPRING/annotation/image-1.png)

JPA 명세상 모든 persistent 클래스들은 no-args constructor가 필요하다. 그렇기 때문에 `@AllArgsConstructor`를 사용하는 경우 `@NoArgsConstructor`도 반드시 사용해야한다.


[Difference Between Lombok @AllArgsConstructor, @RequiredArgsConstructor and @NoArgConstructor | Baeldung](https://www.baeldung.com/java-lombok-constructor-annotations-comparison)
[spring boot - Why to use @AllArgsConstructor and @NoArgsConstructor together over an Entity? - Stack Overflow](https://stackoverflow.com/questions/68314072/why-to-use-allargsconstructor-and-noargsconstructor-together-over-an-entity)
[@Data Annotation In SpringBoot - Avinashsoni - Medium](https://medium.com/@avinashsoni9829/data-annotation-in-springboot-dc18ae965e9c)

****

## LocalTime vs LocalDate vs LocalDateTime

****

## Column(columnDefinition = "TEXT")


****

## Builder

[Why should we use Lombok's @Builder annotation ? - DEV Community](https://dev.to/umr55766/why-should-we-use-lombok-s-builder-annotation-249n)

****

## Column

#### @Column(columnDefinition = "TEXT")
JPA가 자동으로 SQL 컬럼 타입을 정하기 전에, 직접 명시할 수 있다. "TEXT" type은 대량의 텍스트를 저장하기 위한 타입으다. 

****

### @Tag


---

## @Auditing
[Auditing :: Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/auditing.html)


----

## @EntityListener
[JPA Entity Lifecycle Events | Baeldung](https://www.baeldung.com/jpa-entity-lifecycle-events)


----

### @Value
[Using @Value :: Spring Framework](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/value-annotations.html)