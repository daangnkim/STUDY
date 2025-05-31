Error UI를 위한 built-in component는 없지만 Loading UI를 위한 built-in component는 존재하는데 이것이 Suspense다.

`Suspense`는 데이터가 준비되기 전까지 fallback content를 보여주는 것을 의미한다. Suspense는 단순히 데이터를 패칭하는 컴포넌트를 `<Suspense>`로 래핑한다고해서 끝나는게 아니다. `suspense-enabled`한 데이터 소스들에 대해서만 Suspense가 동작한다. 이러한 데이터 소스들은 다음과 같이 존재한다.

1. `tanstack query`, `swr`
2. react 19 버전의 `use` api
3. nextjs의 `server component`

그리고 `suspense-enabled`한 데이터 소스는 `error boundary-enabled`한 데이터 소스이기도 하다.

## `<Suspense>`의 동작 원리

가장 가까운 `Suspense`에게 Promise를 throw한다. 그리고 해당 Promise가 `pending`인 경우 fallback props에 전달된 컴포넌트를 렌더링하고, `resolve`가 되면 컴포넌트를 렌더링한다.

## Unresolved Questions

1. Error Boundary가 이벤트 핸들러 에러, 비동기 에러 등에 대해서는 캐치하지 못하는 이유는 무엇인지?
2. Suspense의 구체적인 동작 원리는 무엇인지?
3. componentDidCatch와 getDerivedStateFromError가 무엇인지?

## References

- [Suspense and Error Boundary in React Explained | Reetesh Kumar](https://reetesh.in/blog/suspense-and-error-boundary-in-react-explained)

- [선언적 비동기 처리로 사용자 경험 향상시키기 (Suspense, ErrorBoundary)](https://enjoydev.life/blog/frontend/12-suspense-errorboundary)

- [Suspense, Error Boundary와 선언적 데이터 패칭 | 5kdk 개발 블로그](https://5kdk.github.io/blog/2023/10/18/suspense-for-data-fetching)

- [How to Create a Functional Error Boundary in React (with some limitations ) | by Dzmitry Ihnatovich | Medium](https://medium.com/@ignatovich.dm/how-to-create-a-functional-error-boundary-in-react-with-some-limitations-a0fd6a598d81)

- [bvaughn/react-error-boundary: Simple reusable React error boundary component](https://github.com/bvaughn/react-error-boundary)

- [Understanding React Suspense with Visuals and Code](https://prismic.io/blog/what-is-react-suspense)

- [react.dev의 suspense](https://react.dev/reference/react/Suspense)
