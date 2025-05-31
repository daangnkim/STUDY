이전에는 `synchronous rendering`이었고 현재는 `concurrent rendering`이 지원된다. `ReactDOM.render()`를 호출하면 synchronous로, `ReactDOM.createRoot().render()`를 호출하면 concurrent로 동작한다.

`synchronous rendering`은 만약 무거운 렌더링 작업을 시작하면 main thread를 blocking하여 ui가 freeze되고 user interaction blocking(스크롤, 클릭 등)이 발생한다.

`concurrent rendering`은 user interaction blocking 문제를 해결한다. 하지만 렌더링을 위한 기존과는 다른 사고방식이 요구된다는 단점이 요구된다.

`concurrent rendering`을 위한 세 가지 메커니즘은 다음과 같다.

1. 렌더링 중단 및 재시작

   기존에는 상태 업데이트가 trigger되면, `trigger`, `rendering`, `commit` 각각의 phase를 중단할 수 없었다. 만약 이 과정에서 다른 상태 업데이트를 진행하려고 시도하면 클릭으로 인한 상태 업데이트는 task queue에 들어가고, 메인 스레드가 완료된 후에 실행된다.

2. 우선순위에 의한 스케쥴링

3. 자동 배칭 처리

특정 작업을 non critical한 작업이라고 마킹하면 background에서 진행된다. 그리고 react는 계속해서 main queue를 확인한다. 만약 main queue에 새로운 작업이 추가된다면 background 작업을 멈추고 main queue 작업을 진행한다.

참고로 여기서 말하는 background는 mental model이다. 자바스크립트는 single thread다.

## `useTransition`의 문제점

`useTransition`과 관련한 dark side는, 렌더링이 두 번 발생하면서 속도가 느려질 수 있다는 점이다. `startTransition`안에서 A 상태를 변경하려고 하는 경우, 먼저 `isPending` 상태가 false에서 true로 되는 과정을 거친 후에 A 상태를 변경한다. 이러한 문제를 해결하려면 컴포넌트를 `React.memo`로 감싸야한다.

혹은 `useTransition`을 잘 사용하려면 nothing to heavy 패턴을 이용한다. 즉, 아무것도 없는 상태에서 무거운 상태를 보여주려고 하는 경우, 이러한 렌더링이 두번 일어나도 크게 문제가 되지 않는다.

`useDeferredValue`는 `useTransition`과 유사하다. 다만, 값을 prop으로 전달받는 것 처럼, setter에 접근하지 않아도 될때 사용된다. 하지만 더블 렌더링은 여전히 발생한다.

`debounce` 대신 `useDeferredValue` 사용은 불가하다. react는 너무 빠르기 때문이다.

## useTransition (v18)

## useDeferredValue (v18)

fresh content가 로드되는 동안 stale content를 보여주기 위해 사용된다.

## QUESTION

1. concurrent rendering은 stable한가?
2. concurrent rendering은 개념이고, 이를 구현하는 feature들이 존재하는게 많나? concurrent rendering이 가능하기 때문에 `startTransition`, `useDeferredValue`등이 동작한다라는 의미인가?
3. concurrent rendering을 실제로 쓰는 경우라면 웹페이지 렌더링이되게 오래 걸리는 페이지여야 할거같은데, 그정도로 오래 걸리는 경우가 있나?
4. It's a result of how the new concurrent renderer works. In concurrent mode, React prepares the new screen (or the updated state of a component) in memory before it updates the DOM. This is known as "rendering in memory" or "offscreen rendering". 메모리에 미리 저장한다는 말이 무슨 말이지?

- [React useTransition: performance game changer or...?](https://www.developerway.com/posts/use-transition)

- [Exploring React Concurrent Mode: Unlocking Its Potential](https://www.dhiwise.com/post/deep-dive-into-react-concurrent-mode-exploring-key-features-and-use-cases)

- [React 18 Concurrent Rendering: What You Need to Know - SDLC Corp](https://sdlccorp.com/post/react-18-concurrent-rendering-what-you-need-to-know/)
