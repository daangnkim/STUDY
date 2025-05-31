`useState`는 initialState를 넣어서 state와 setter 함수를 반환받는다.
`reducer`는 state와 action을 넣어서 새로운 state를 반환받는다.
`useReducer`는 reducer와 initialState를 넣어서 state와 dispatch 함수를 반환받는다.

useState는 useReducer의 대안이다. useReducer는 복잡한 상태 관리를 좀 더 구조화하고 예측 가능하게 만든다. 상태 관리를 하나의 centralized 된 함수인 `reducer`에서 진행할 수 있다.

그리고 다음과 같이 사용한다.

```javascript
const [state, dispatch] = useReducer(reducer, initialArg, init?)
```

1. reducer

   `reducer`는 상태가 어떻게 업데이트되는지 정의하는 함수이다. 렌더링시에 실행되기 때문에 반드시 `pure`해야 하며, `state`와 `action`을 argument로 받고, 다음 상태를 반환해야 한다.

2. initialArgs

   초기 상태를 계산할 때 사용된다.

3. init?

   초기 상태를 계산하는 함수이다. initialArgs를 인자로 받아 초기 상태값을 만드렁진다. 만약 전달하지 않으면 initialArg가 초기 상태가 된다. 만약 initialArgs가 필요 없다면 null을 할당한다. 만약 init을 이용하지 않고 아래와 같이 initialArgs에서 init 함수를 호출한다면, 매 렌더링마다 init 함수가 실행될 것이므로 분리하는 것이 좋다.

   ```javascript
   function createInitialTodoState(username) {
     // ...
   }

   function TodoList({ username }) {
     const [state, dispatch] = useReducer(
       reducer,
       createInitialTodoState(username)
     );
     // ...
   }
   ```

참고로 javascript의 `reduce` 메서드에 영향을 받아 `reducer`라는 이름이 지어지게 됐다. `reduce` 메서드는 accumulator와 current item을 받아 next result를 만들어 내는데, `reducer`도 current state와 action을 받아 new state를 만들어내는게 이와 비슷하여 명명하게 되었다.

## 제네릭 타입

두 가지 제네릭 타입을 받는다. 첫 번째는 init 함수의 제네릭 타입을, 두 번째는 초기값의 타입을 받는다.

```typescript
function useReducer<R extends Reducer<any, any>, I>(
  reducer: R,
  initializerArg: I,
  initializer: (arg: I) => ReducerState<R>
): [ReducerState<R>, Dispatch<ReducerAction<R>>];
```

## 주의할 점

1. reducer 함수는 함수 컴포넌트 밖에 배치한다.
   가독성을 위해서이다.

   ```javascript
   import { useReducer } from "react";

   function reducer(state, action) {
     // Define how the state should change based on the action
   }

   function MyComponent() {
     const [state, dispatch] = useReducer(reducer, {
       theme: "dark",
       language: "eng",
     });
     // ...
   }
   ```

2. immutable state update를 진행한다
   이는 useState와 동일하다.

3. 다음 상태 접근 방법
   useState처럼 상태를 업데이트하고 바로 업데이트 된 상태에 접근 가능한게 아니다.

   ```javascript
   function handleClick() {
     console.log(state.age); // 27
     dispatch({ type: "incremented_age" });
     console.log(state.age); // Still 27, but why ??!
   }
   ```

   다음 상태에 곧바로 접근하려면 아래와 같이 `reducer`를 이용해야한다.

   ```javascript
   function handleClick() {
     console.log(state.age); // 27
     dispatch({ type: "incremented_age" }); // Request a re-render with 43
     const nextState = reducer(state, action);
     console.log(nextState.age); // 28
   }
   ```

4. switch 문 작성시 항상 default 케이스(fallback)를 작성해놓자
   이는 의도치않은 상태 변경 시도 케이스를 잡을 수 있다.

   ```javascript
   function reducer(state, action) {
     switch (action.type) {
       case "SWITCH_THEME":
         return { ...state, theme: state.theme === "dark" ? "light" : "dark" };
       case "CHANGE_LANGUAGE":
         return { ...state, language: action.langChoice };
       default:
         console.error("unexpected action type: " + action.type);
         return state;
     }
   }
   ```

5. 상태를 항상 객체로 다룬다.

   보통 useReducer로 다루는 상태는 매우 복잡한 상태이다. 만약 복잡한 상태가 아니라면 useState를 사용하자.

6. `if` 문 대신 `switch` 문을 사용한다.

7. 가능한한 모든 action을 핸들링한다.

8. `action`과 `user interaction`은 1:1 매핑이다.

   5개의 입력 필드가 존재하는 form을 reset할 때, set field action을 다섯 번 호출하는 것이 아닌, reset action을 한 번만 호출한다.

9. `action`은 `user interaction`을 잘 분명하게 표현해야한다.

10. dispatch의 형태는 전통적으로 `type`과 `payload`로 구성된다.

## DEBT

1. 전역 상태에서 useReducer를 사용하려면?

- [React Best Practices : using useReducer() | by Manish Sharma | Medium](https://medium.com/@mansha99/react-best-practices-using-usereducer-667422051610)

- [useThe Ultimate Guide to useReducer: Best Practices and Pitfalls | by Emrebener | Medium](https://emrebener.medium.com/the-ultimate-guide-to-usereducer-best-practices-and-pitfalls-01912c711b4f)

- [Using the useReducer Hook in React with TypeScript - DEV Community](https://dev.to/craigaholliday/using-the-usereducer-hook-in-react-with-typescript-27m1)

- [React 18.x & TypeScript | how to safely type the useReducer hook](https://www.fabiobiondi.dev/blog/2023-01/how-to-safely-type-usereducer-in-react-and-typescript/)
