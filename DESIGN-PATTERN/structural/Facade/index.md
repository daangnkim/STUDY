여러 서브 시스템에 대한 인터페이스를 제공한다. 커피 머신기의 분쇄, 가열, 추출 과정을 몰라도 된다. 불필요한 복잡성을 숨긴다.

결과적으로 의존성 단순화, 결합도 감소를 이뤄낼 수 있다.

## When to use

1. 라이브러리나 API를 제공할 때, 클라이언트에게 필요한 인터페이스를 제공해야할 때
2. layered architecture에서 각 레이어에 대한 entry point를 제공할 때
3. 레거시 코드를 리펙토링 해야하는데 한 번에 하기 힘들 때

[Introduction To Facade Pattern | Cloudaffle | Everything About Web Development, JavaScript Tutorials, Tips and Tricks](https://cloudaffle.com/series/structural-design-patterns/facade-pattern-classic-implementation/)
