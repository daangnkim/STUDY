`Declarative` --> `Data` --> `Framework`로 갈수록 제어권을 react-router가 가져간다.

##### Declarative
URL에 매핑되는 컴포넌트 렌더링, `<Link>`, `useNavigate`, `useLocation`등 가장 기본적인 API들만 제공한다.

##### Data
route 설정을 react rendering 바깥으로 옮기며 `loader`, `action`, `useFetch` 등이 제공된다.

##### Framework
- typesafe `href`
- typesafe Route Module API
- intelligent code splitting
- SPA, SSR, and static rendering strategies
- and more

[Picking a Mode | React Router](https://reactrouter.com/start/modes)