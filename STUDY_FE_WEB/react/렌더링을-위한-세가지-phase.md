UI를 그리기 위해 세 가지 `phase`로 나뉜다.

1. `trigger` a render
2. `render` the component
3. `commit` to the dom`

`trigger`는 크게 initial render와 state 변경에 의한 render 트리거 두가지로 나뉜다. initial render의 경우 `createRoot` 호출 이후 `render`를 호출한다.

`render`는 컴포넌트를 호출하는 것이다. initial render의 경우 루트 컴포넌트를 호출한다. 그리고 DOM node를 생성한다. subsequent render의 경우 상태가 변경된 함수를 `recursive`하게 호출한다. 그리고 이전 render와 비교해서 어떤 property가 변경됐는지 계산만해둔다. 만약 변경되지 않았다면 `commit` phase에서 아무것도 하지 않는다.

**`render` 과정은 항상 `pure` 해야한다.** 동일한 prop, state가 입력되면 동일한 jsx가 나와야한다. 또한 `render`는 이전에 존재하던 객체나 변수를 변경하면 안된다.

`commit`은 DOM을 실제로 수정하는 것이다. initial render의 경우 `appendChild()`DOM API를 이용해서 생성된 DOM node를 모두 적용한다. subsequent render의 경우 변경된 DOM node만 수정하도록 최소한의 작업을 진행한다.

```jsx
export default function Clock({ time }) {
  return (
    <>
      <h1>{time}</h1>
      <input />
    </>
  );
}
```

위 코드에서 input을 입력해도 입력한 텍스트가 사라지지 않는 이유는 `<h1/>` 태그만 수정되고 있다고 판단하기 때문이다. DOM을 실제로 수정하고 `browser rendering`과정이 진행된다.

- [Render and Commit – React](https://react.dev/learn/render-and-commit)
