## 기본적인 개념

route-level data fetching이 가능하게한다. 다만 `data router mode` 내에서만 동작한다.

네비게이션이 시작되자마자 컴포넌트가 그려지길 기다리지 않고 데이터 패칭을 시작한다. 이는 부모 컴포넌트가 그려지고 데이터를 패칭한뒤 자식 컴포넌트가 그려지고 데이터를 패칭하는 waterfall 문제를 제거한다.

****

## loader의 장점

##### decoupling data fetching from ui rendering
컴포넌트는 데이터를 보여주고 user interaction만 관리한다.

##### 로딩스피너가 보여지지 않고 완성된 UI가 보여진다

#### react-router의 Link 태그를 이용하여 prefetch가 가능해진다.

****

## loader의 단점

##### 1. 설계 복잡도 증가
##### 2. 컴포넌트 상태에 따른 dynamic fetching 불리

****

## 궁금증

1. prefetch를 사용하지 않더라도 loader를 사용하는게 더 빠른 이유는 뭘까?
2. loader 내에서 요청을 defer한 다음에 로딩 스피너를 보여줄 수는 없을까?

어떤 페이지에 대한 로딩 처리는 크게 세곳에서 가능하다.

1. 컴포넌트 내에서 isLoading을 처리한다.
2. Suspense에서 isLoading을 처리한다. Suspense를 사용하므로 데이터가 로드될 때까지 UI가 보여지지 않는다.
3. loader를 호출하고 Suspense에서 처리한다. Suspense를 사용하므로 데이터가 로드될 때까지 UI가 보여지지 않는다.


[React Query meets React Router | TkDodo's blog](https://tkdodo.eu/blog/react-query-meets-react-router)
[What a Data Router/Loader is and why you need it! | Velopack](https://docs.velopack.io/blog/2024/05/24/seemless-router-preloading)
[Framework Adoption from RouterProvider | React Router](https://reactrouter.com/upgrading/router-provider#prerequisites)