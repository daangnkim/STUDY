


Header 컴포넌트를 구현한다고 가정해보자.

##### Monolithic Component
하나의 컴포넌트 안에 모두 구현하는 방식이다. 모든 로직이 하나의 컴포넌트 안에서 구현돼기 때문에 부모 컴포넌트에서의 불필요한 리렌더링을 방지할 수 있다.

##### render prop
왼쪽과 오른쪽, 중앙 배치를 구현하기 쉽다. children prop으로 넘기는 것 역시도 render prop에 해당한다. 상태를 부모 컴포넌트로 올릴 수 있다는 장점이 존재한다. semantic함이 깨지는 이슈가 존재한다.

##### compound
왼쪽과 오른쪽, 중앙 배치를 조금 더 semantic하게 구현할 수 있다. 다만 구조를 지켜야한다는 단점이 존재한다.

##### HOC


## QUESTIONS

- 컴포넌트를 나누는 기준은 뭐가 있을까?
	- 애니메이션을 고려할 수도 있고, 리렌더링을 고려할 수도 있고, 확장성을 고려할 수도 있고...

[Patterns.dev.kr](https://patterns-dev-kr.github.io/)