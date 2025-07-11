## [Getting Started: Fetching Data | Next.js](https://nextjs.org/docs/app/getting-started/fetching-data)

- `fetch`는 기본적으로 캐싱되지 않지만, nextjs는 라우트를 프리렌더링하고 성능 개선을 위해 캐싱한다. 동적 렌더링을 원한다면 `{ cache: 'no-store' }` 를 사용하자.
- 개발하는 동안에는 `fetch`에 대한 로깅이 가능하다.


### 클라이언트 컴포넌트에서 데이터 패칭

두 가지 방법이 존재한다.

1. `use` 훅을 사용한다.
2. `SWR`, `tanstack query`를 이용한다.

