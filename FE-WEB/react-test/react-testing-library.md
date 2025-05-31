### 관련한 패키지들

##### `jest`
test runner, assertion library, mocking 기능을 제공한다. 

##### `jsdom`
nodejs에서 사용하기 위해 웹 표준을 자바스크립트로 구현한 것이다.

##### `jest-environment-jsdom`
jest 테스트 프레임워크에서 jsdom을 사용할 수 있게 해주는 패키지. 가상의 DOM 환경을 만들어서 실제 브라우저 없이도 DOM API를 사용할 수 있게 만듬.


##### `@testing-library/react`
react-testing-library에 대한 코어 라이브러리다.

##### `@testing-library/user-event`
user interaction과 관련한 event를 시뮬레이션하는 라이브러리다

##### `@testing-library/jest-dom`
DOM 상태와 관련한 다양한 assert를 작성하는 경우가 존재하는데, 이와 관련한 다양한 test matcher(어떤 값을 여러 방식으로 테스트 할수있도록 만듣는 함수)들을 제공한다.



### 철학

> 테스트가 실제 소프트웨어 사용 방식과 유사할수록, 더 큰 신뢰도를 제공한다.

RTL은 구현 세부사항에 대한 테스트가 아닌, 사용자 관점에서 테스트하는 것을 강조한다.

그리고 다음 세 가지 원칙이 존재한다

1. 컴포넌트를 렌더링할 때 컴포넌트 인스턴스가 아닌 실제 DOM Node를 렌더링한다. 그러니까 실제 사용자가 브라우저에서 보는 것과 동일한 방식으로 테스트한다. 컴포넌트 내부의 상태나 함수에 접근하지 않는다.

2. 실제 사용자가 애플리케이션을 사용하는 방식대로 테스트한다. 버튼을 테스트할 때 버튼의 내부 상태를 확인하는 것이 아닌, 실제 DOM Node에 클릭 Query를 보내고 예상되는 결과가 나타나는지 확인한다.

3. API 구현이 단순하고 유연하다.

### 특징

이 라이브러리는 테스트 러너가 아니다.

Jest를 추천하지만 특정 테스팅 프레임워크에 종속되지 않는다.

react-testing-library(이하 rtl)은 react component 테스트를 위한 툴이다. DOM Node를 테스트하는 `DOM Testing Library`위에 만들어졌다.

react-hooks-testing-library의 renderHook이 react-testing-library로 병합되는중에 있다.

[jest를 타입스크립트와 함께 사용하는 방법](https://jestjs.io/docs/getting-started#using-typescript)

[jsdom/jsdom: A JavaScript implementation of various web standards, for use with Node.js](https://github.com/jsdom/jsdom)

[Getting Started · Jest](https://jestjs.io/docs/getting-started)

[Guiding Principles | Testing Library](https://testing-library.com/docs/guiding-principles)

[Installation | React Hooks Testing Library](https://react-hooks-testing-library.com/installation#renderer)

[React Testing Library | Testing Library](https://testing-library.com/docs/react-testing-library/intro/)

[Using Matchers · Jest](https://jestjs.io/docs/using-matchers)

[Setup Jest and React Testing Library in a React project | a step-by-step guide - DEV Community](https://dev.to/ivadyhabimana/setup-jest-and-react-testing-library-in-a-react-project-a-step-by-step-guide-1mf0)
[A Guide to Testing React Components with Jest and React Testing Library | Keploy Blog](https://keploy.io/blog/community/a-guide-to-testing-react-components-with-jest-and-react-testing-library)
