#### `describe`

관련한 테스트를 그룹핑한다. name과 test를 포함하는 함수를 인자로 받는다.

#### `it` and `test`

둘다 교환 가능하며, 취향에 따라 사용 가능하다. name과 test assertion을 포함하는 함수를 인자로 받는다.

#### `render`

react component를 렌더링하기 위해 사용

#### `fireEvent`

유저의 상호작용을 시뮬레이션하기 위해 사용

#### `screen`

렌더링된 컴포넌트에 대해서 assert하거나 쿼리할 수 있는 유틸리티 함수들 제공

#### `cleanup`

각 테스트마다 렌더링된 컴포넌트를 clean 하기 위해 사용. 보통 `afterEach` 훅 내에서 사용.


[When to describe, it and test. The describe, it, and test keywords are… | by Leo Voon | Medium](https://leovoon.medium.com/when-to-describe-it-and-test-ec2aba477b59)