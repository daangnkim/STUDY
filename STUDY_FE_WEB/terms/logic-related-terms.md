
다음 용어들에 대해서 정리해보자

1. domain
2. domain logic vs business logic
3. application logic vs business logic
4. service logic
5. presentation logic

`domain` 은 관심 대상(area of concern)이다. 관심 대상이 `business`면 domain은 `business domain`이된다. 

`business logic`은 시스템이 하는 것(**What system does**)을 정의한다. (규칙 관점. 즉, 유효성 검사, 작업절차 등)

`application logic`은 시스템이 이것을 어떻게 하는가(**How system does**)를 정의한다. (실행 관점. 즉, API, 데이터베이스, 캐싱 등). `business logic`과 `user interaction`의 `bridge`다. 

`domain logic`과 `business logic`은 비슷하지만, 굳이 구분한다면 `domain logic`은 개체, 모델간의 관계 정의에 관한 것이며, `business logic`은 context에 적용되는 규칙을 의미한다. 예를들면, `X 그룹의 주문은 Z 금액을 초과하면 Y 만큼의 Discount를 적용한다.`는 `business logic`이다.

axios.get, axios.post 처럼 REST 요청을 실제로 수행하는 함수를 `service logic`이라고 부르고, UI 상태 관리, 이벤트 핸들링 등 인터페이스와 관련된 로직을 `presentation logic`이라고 부른다.

[Business Logic vs. Application Logic in Systems Design - DEV Community](https://dev.to/msnmongare/business-logic-vs-application-logic-in-systems-design-17ip)

[architecture - What is domain logic? - Stack Overflow](https://stackoverflow.com/questions/360860/what-is-domain-logic)

[oop - Difference between domain model, conceptual model and business model etc - Stack Overflow](https://stackoverflow.com/questions/25947537/difference-between-domain-model-conceptual-model-and-business-model-etc)