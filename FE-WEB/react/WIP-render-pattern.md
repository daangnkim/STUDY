
Header 컴포넌트를 구현한다고 가정해보자.



##### children prop
왼쪽과 오른쪽, 중앙 배치를 일일히 구현해야한다. 

##### render prop
왼쪽과 오른쪽, 중앙 배치를 구현하기 쉽다. `react-router-dom`의 Layout 내에 배치하는 경우, 부모 컴포넌트 상태 관리에 참여하게 되면서 전역 렌더링을 유발할 수 있다. (물론 `React.memo`를 사용하여 커버는 가능하다.) 이는 앞선 children prop도 마찬가지이다.

##### compound
왼쪽과 오른쪽, 중앙 배치를 조금 더 semantic하게 구현할 수 있다.

##### HOC


[Patterns.dev.kr](https://patterns-dev-kr.github.io/)