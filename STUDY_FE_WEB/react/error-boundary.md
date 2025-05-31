## class component lifecycle

lifecycle 메서드 관련하여 이미지를 검색해보면 각양각색으로 만들어놓은 이미지들이 나오는데, 리액트 16.3 이후 다음 세 가지 메서드들은 사라졌음을 참고하여 이미지를 찾자.

1. `componentWillMount`
2. `componentWillReceiveProps`
3. `componentWillUpdate`

이중 Error Boundary에 주로 사용되는 `getDerivedStateFromError`, `componentDidCatch`에 대한 설명은 다음과 같다.

1. `getDerivedStateFromError`
   Error를 Catch한 이후에 fallback ui를 보여주기 위해 사용된다. rendering phase에서 호출되므로, 사이드 이펙트를 만드는 것을 허용하지 않는다.

2. `componentDidCatch`
   에러 정보를 로깅하기 위해서 사용된다. commit phase에서 호출되므로 사이드 이펙트를 만들 수 있다.

- [React Lifecycle | GeeksforGeeks](https://www.geeksforgeeks.org/reactjs-lifecycle-components/)

- [Intro to React component lifecycle | by Rafael Cruz | Medium](https://medium.com/@ralph1786/intro-to-react-component-lifecycle-ac52bf6340c)

## `Error Boundary`

Error Boundary는 컴포넌트 트리에서 발생하는 에러를 캐치한다음, 로깅하고, fallback UI를 보여줌으로써 gracefully하게 handling하여 app crash의 발생을 막는다. component tree를 타고 에러가 전파되는 것을 막는다.

Error Boundary는 다음 네 가지 에러는 캐치하지 못한다.

1. 이벤트 핸들러 에러 (이때는 `try/catch`를 사용해야한다)
2. 비동기 에러 (setTimeout or requestAnimationFrame)
3. 서버 사이드 렌더링
4. Error Boundary 내부에서 발생한 에러

function component로는 만드는게 불가능하다. 에러 핸들링을 위한 다음 두 가지 lifecycle 메서드가 class component에만 존재하기 때문이다.

1. `getDerivedStateFromError`
2. `componentDidCatch`

Error Boundary 구현을 위해서는 위 두개 메서드 중 하나 이상을 이용하면 되며, 예시 코드는 다음과 같다.

```typescript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // Update state to display the fallback UI on the next render.
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.log(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // Render any custom fallback UI
      return <h1>Oops! Something went wrong.</h1>;
    }

    return this.props.children;
  }
}

// Using in a component
class App extends React.Component {
  render() {
    return (
      <ErrorBoundary>
        <MyComponent />
      </ErrorBoundary>
    );
  }
}
```

## Error Boundary의 종류

필요에 따라서 Component Level, Layout Level, Top Level Error Error Boundary 세 가지로 나눌 수 있다.

1. Component Level

   컴포넌트 트리 하위에 배치. 개별 컴포넌트를 Error Boundary로 래핑하는 접근으로, 높은 세분화 수준(a high level of granularity)을 제공한다. 상태를 공유하지 않는 개별적인 컴포넌트들을 래핑할때 적합하다. 다만 컴포넌트가 많은 경우 중복 코드가 많이 발생할 수 있다.

2. Layout Level

   컴포넌트 트리 상위에 배치. Sidebar나 Dashboard처럼 상태를 공유하는 컴포넌트들을 래핑할때 적합하다.

3. Top Levels

   컴포넌트 트리 최상위에 배치. 애플리케이션에서 발생할 수 있는 어떠한 에러든 핸들링할 때 적합하다. 개별 컴포넌트 혹은 컴포넌트 그룹에서 발생한 에러인데, Top Level에서 핸들링하는 경우 사용자에게 사라져보이는 영역이 불필요하게 커보일 수 있게된다.

직접 구현도 가능하지만 [bvaughn/react-error-boundary: Simple reusable React error boundary component](https://github.com/bvaughn/react-error-boundary) 패키지를 이용할 수도 있다.

## REFERENCES

- [Error Boundary in React js - DEV Community](https://dev.to/imashwani/error-boundary-in-react-js-1n5o)

- [React Error Boundary: A More Powerful Way to Handle React Errors | ILLA Cloud](https://illacloud.com/blog/react-error-boundary/#component-level-error-boundaries)

- [Suspense and Error Boundary in React Explained | Reetesh Kumar](https://reetesh.in/blog/suspense-and-error-boundary-in-react-explained)

- [선언적 비동기 처리로 사용자 경험 향상시키기 (Suspense, ErrorBoundary)](https://enjoydev.life/blog/frontend/12-suspense-errorboundary)

- [Suspense, Error Boundary와 선언적 데이터 패칭 | 5kdk 개발 블로그](https://5kdk.github.io/blog/2023/10/18/suspense-for-data-fetching)

- [How to Create a Functional Error Boundary in React (with some limitations ) | by Dzmitry Ihnatovich | Medium](https://medium.com/@ignatovich.dm/how-to-create-a-functional-error-boundary-in-react-with-some-limitations-a0fd6a598d81)

- [bvaughn/react-error-boundary: Simple reusable React error boundary component](https://github.com/bvaughn/react-error-boundary)

- [(KOR)[React] API 에러 로깅은 어디서 처리하는 것이 좋을까? (feat. React-Query) | by Jaewang Lee | Medium](https://medium.com/@jnso5072/kor-react-api-%EC%97%90%EB%9F%AC-%EB%A1%9C%EA%B9%85%EC%9D%80-%EC%96%B4%EB%94%94%EC%84%9C-%EC%B2%98%EB%A6%AC%ED%95%98%EB%8A%94-%EA%B2%83%EC%9D%B4-%EC%A2%8B%EC%9D%84%EA%B9%8C-feat-react-query-fe67feec8e4c)
