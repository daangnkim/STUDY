## useId (v18)

accessibility 관련 속성에 전달 가능하다.

id를 하드코딩하는 것은 좋지 않기때문에 적합하다.

컴포넌트 트리에서 위치를 기반으로 생성하기 때문에 서버와 클라이언트 모두 동일한 값을 갖는 것을 보장한다.

[useId – React](https://react.dev/reference/react/useId)

## useDeferredValue & useTransition (v18)

concurrent rendering에서 설명한다.

## useSyncExternalStore

## useInsertionEffect

## useImperativeHandle (v18)

리액트는 기본적으로 declarative하지만 어쩔 수 없이 imperative 해야할 때 사용한다.

부모 컴포넌트가 자식 컴포넌트의 DOM 요소의 exposed handle을 결정한다.

두 번째 인자인 `createHandle`은 ref에 할당된다. 세 번째 인자가 변경될 때만 createHandle 함수가 재생성된다.

`createHandle` 반환 객체의 메서드명은 DOM 요소의 이름을 따르지 않아도 된다.

> 컴포넌트 내에 존재하는 value들은 모두 reactive하다. reactive value는 컴포넌트 내부에 존재하는 prop, state, 변수, 함수 등을 말하며 reactive value가 변경되면 rerendering을 발생시킬 수 있다. 그러므로 훅은 의존하는 reactive value를 dependency array에 넣어야한다.

[useImperativeHandle – React](https://react.dev/reference/react/useImperativeHandle)

[What Are "Reactive Values" in React? - Designcise](https://www.designcise.com/web/tutorial/what-are-reactive-values-in-react)

[Lifecycle of Reactive Effects – React](https://react.dev/learn/lifecycle-of-reactive-effects)

## use (v19)
