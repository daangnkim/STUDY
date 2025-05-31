인스턴스가 오직 하나만 생성되도록 하면서 전역에서 인스턴스에 접근할 수 있도록하는 패턴

## When to use

1. 전역 변수인 경우
2. initialization 비용이 많이 드는 경우
3. 프린터 관리자처럼 단일 제어 지점을 가져야하는 리소스를 관리하는 경우 = 각 인스턴스가 동일하며 서로 다른 상태를 유지하지 않는 경우

[Understanding Singleton Design Pattern | Cloudaffle | Everything About Web Development, JavaScript Tutorials, Tips and Tricks](https://cloudaffle.com/series/creational-design-patterns/singleton-design-patterns/)
