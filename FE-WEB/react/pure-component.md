`pure component`를 작성하여 컴포넌트를 예측 가능하게 만든다. 그리고 리액트는 작성하는 모든 컴포넌트가 `pure component`이길 기대한다.

즉, 동일한 입력에 대해서 동일한 jsx가 반환되고, 이전의 변수나 객체를 변경하지 않아야한다.

```javascript
function Recipe({ drinkers }) {
  return (
    <ol>
      <li>Boil {drinkers} cups of water.</li>
      <li>
        Add {drinkers} spoons of tea and {0.5 * drinkers} spoons of spice.
      </li>
      <li>Add {0.5 * drinkers} cups of milk to boil and sugar to taste.</li>
    </ol>
  );
}

export default function App() {
  return (
    <section>
      <h1>Spiced Chai Recipe</h1>
      <h2>For two</h2>
      <Recipe drinkers={2} />
      <h2>For a gathering</h2>
      <Recipe drinkers={4} />
    </section>
  );
}
```

위 drinkers에 2를 넣으면 `2 cups of water`가 렌더링되고, 4를 넣으면 `4 cups of water`가 보여진다.

이러한 규칙을 깨는 예제는 다음과 같다.

```jsx
let guest = 0;

function Cup() {
  // Bad: changing a preexisting variable!
  guest = guest + 1;
  return <h2>Tea cup for guest #{guest}</h2>;
}

export default function TeaSet() {
  return (
    <>
      <Cup />
      <Cup />
      <Cup />
    </>
  );
}
```

호출할 때마다 다른 jsx를 반환한다는 점에서 pure하지 않다. 위 코드는 함수 스코프 바깥에, 함수가 호출되기 전부터 존재하며 함수를 impure하게 만든다. 만약 렌더링 과정에 변수나 객체를 생성한다면 이러한 문제가 발생하지 않는다.

```jsx
function Cup({ guest }) {
  return <h2>Tea cup for guest #{guest}</h2>;
}

export default function TeaGathering() {
  let cups = [];
  for (let i = 1; i <= 12; i++) {
    cups.push(<Cup key={i} guest={i} />);
  }
  return cups;
}
```

순수 함수는 외부 상태를 변경하지 않는다. 하지만 화면을 업데이트하거나, 애니메이션을 시작하거나, 데이터를 변경하는 등의 변경하는 `side effect`가 필요하다. 이러한 side effect는 보통 이벤트 핸들러 안에 배치한다.

rendering phase는 pure 해야하기 때문이다. rendering phase에서 이벤트 핸들러가 실행되지 않기 때문에, side effect는 이벤트 핸들러 안에서 만든다. rendering phase가 아닌 `on the side`에서 실행되기 때문에 `side effect`라고 부른다.

그렇다면 리액트는 왜 `순수성`을 중요하게 여길까?

1. 항상 동일한 입력에 대해서 동일한 결과를 만들기 때문에 서버에서 실행이 가능하고, 하나의 컴포넌트가 여러 사용자의 요청에 대응이 가능하다.
2. 동일한 결과를 만들어낸다면 캐싱이 가능하다.
3. 렌더링을 하는 도중 데이터가 업데이트되면 기존 렌더링을 중단하고 재시작이 가능하다.

3번은 동시성 모드(Concurrent Mode)에서 더 알아보자.

## `pure`한지 여부를 판단하는 `<StrictMode/>`

`<StrictMode/>`를 래핑하면 개발 모드에 한해서 function component 내부의 코드를 두 번 실행하여 component의 순수성을 판단한다. function component가 순수하다면 코드를 두 번 실행해도 항상 동일한 입력에 대해서 동일한 결과가 나오기 때문에 변화가 없을 것이다.

- [Keeping Components Pure – React](https://react.dev/learn/keeping-components-pure)
