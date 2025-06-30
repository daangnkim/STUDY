## loader와 lazy의 등장 배경

route-level data fetching이 가능하게한다. 다만 `data router mode` 내에서만 동작한다.

네비게이션이 시작되자마자 컴포넌트가 그려지길 기다리지 않고 데이터 패칭을 시작한다. 이는 부모 컴포넌트가 그려지고 데이터를 패칭한뒤 자식 컴포넌트가 그려지고 데이터를 패칭하는 waterfall 문제를 제거한다. 

즉 데이터 패칭 -> 렌더링 순서로 진행된다.

loader는 render cycle 바깥에서 패치한다. (🤔 react router가 페이지에 필요한 데이터를 알수 있게된다?)

waterfall 문제를 해결하고나서 만나는 다음 문제는 거대한 js bundle이다. js bundle은 당장 보지 않는 페이지들과 페이지들에 대한 loader까지 한 번에 로드한다. 그렇다면 React.lazy를 쓰면 되지 않을까? 라는 생각이든다.

우리가 아는 React.lazy는 데이터 패칭 -> Lazy 컴포넌트 요청 -> 렌더링 이라는 waterfall과 같은 체이닝 문제가 여전히 존재한다.

react-router에서 만든 Lazy를 적용하게 되면, Loader와 Component를 하나의 번들 안에서 load하는 것이 아닌 Loader와 Component를 병렬로 다운로드한다.


****

## loader의 장점

##### parallel data fetching
router level에서 데이터 패칭이 일어나므로, sequential data fetching으로 인한 waterfall 현상을 제거할 수 있다.
##### decoupling data fetching from ui rendering
컴포넌트는 데이터를 보여주고 user interaction만 관리한다.
##### 로딩스피너가 보여지지 않고 완성된 UI가 보여진다

#### react-router의 Link 태그를 이용하여 prefetch가 가능해진다.

****

## loader의 단점

##### 설계 복잡도 증가

##### 컴포넌트 상태에 따른 dynamic fetching 불리




****

## 🤔

1. prefetch를 사용하지 않더라도 loader를 사용하는게 더 빠른 이유는 뭘까?
2. loader 내에서 요청을 defer한 다음에 로딩 스피너를 보여줄 수는 없을까?

어떤 페이지에 대한 로딩 처리는 크게 세곳에서 가능하다.

1. 컴포넌트 내에서 isLoading을 처리한다.
2. Suspense에서 isLoading을 처리한다. Suspense를 사용하므로 데이터가 로드될 때까지 UI가 보여지지 않는다.
3. loader를 호출하고 Suspense에서 처리한다. Suspense를 사용하므로 데이터가 로드될 때까지 UI가 보여지지 않는다.


[React Query meets React Router | TkDodo's blog](https://tkdodo.eu/blog/react-query-meets-react-router)
[What a Data Router/Loader is and why you need it! | Velopack](https://docs.velopack.io/blog/2024/05/24/seemless-router-preloading)
[Framework Adoption from RouterProvider | React Router](https://reactrouter.com/upgrading/router-provider#prerequisites)
[Faster Lazy Loading in React Router v7.5+ | Remix](https://remix.run/blog/faster-lazy-loading#background)
[Lazy Loading Routes in React Router 6.4+ | Remix](https://remix.run/blog/lazy-loading-routes)