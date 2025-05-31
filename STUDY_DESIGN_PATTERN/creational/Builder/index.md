복잡한 객체를 단계적으로 생성하는 패턴

ConcreteBuilder는 프로덕트 생성 단계 각각을 정의한다.

클라이언트 코드가 빌더를 직접 호출하여 프로덕트를 생성할 수도 있지만, Director 클래스에서 프로덕트 생성 단계를 조합하는 것이 효율적일 수 있다. 특히 프로덕트 생성 과정이 복잡할수록 더욱 추천된다.

## When to use

1. 프로덕트와 관련한 속성이 여러개 존재하고 특정 속성은 선택적이고 특정 속성은 필수적인 경우
2. 프로덕트가 여러 단계에 걸쳐 생성되고, 특정한 순서대로 생성돼야하는 경우
3. 다양한 방식으로 프로덕트가 생성될 수 있는 경우, 그리고 모든 조합 구성에 대한 생성자를 제공하는 것이 비현실적인 경우
4. 매개변수가 많고, 매개변수 유형이 명확하지 않은 경우

```typescript
const configProblematic = new ServerConfigProblematic(
  "localhost",
  8080,
  true,
  30000,
  10
);

// VS

const config = new ServerConfig.Builder()
  .setHost("localhost")
  .setPort(8080)
  .setUseSSL(true)
  .setTimeout(30000)
  .setMaxConnections(10)
  .build();
```

[The Builder Pattern In TypeScript | Cloudaffle | Everything About Web Development, JavaScript Tutorials, Tips and Tricks](https://cloudaffle.com/series/creational-design-patterns/builder-pattern/)
