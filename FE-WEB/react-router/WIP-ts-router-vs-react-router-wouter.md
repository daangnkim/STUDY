## react-router를 쓰면서 느꼈던 점

- 문서가 구리다.
	- Outlet의 최적화에 대한 설명이 없음.
		- Layout 컴포넌트를 딱딱하게 만듬.
	- loader의 유효성
		- 
	- validation 방법의 부재?
- 타이핑이 잘 안된다. 특히 query parameter와 엮이는 순간 코드가 매우 지저분해진다.
- 파일 경로 설정이 애매하다.
- loader가 가져오는 성능 개선과 벨리데이션이 매력적이다.

## ts-router

- 타입스크립트가 라우트에 대한 타입을 가능한한 최대한 추론하게 하는 것이 목표다.
- 



## react-router에 대한 사람들의 평가


내가 공식 문서 읽을 줄을 모르나 싶을 정도로 공식 문서를 읽기가 어려웠어서 사람들 평을 좀 찾아봤다. 더불어서, 얼핏보니 react-router에 대한 안좋은 평가도 보이는 것 같아 어떤 부분이 안좋게 평가되는지 찾아봤다. 괄호치고 쓴건 필자 개인의견

- react-router-dom이 단순한 라우팅 도구에서 nextjs와 같은 풀스택 프레임워크로 발전하려고 시도하는 중
- 라우팅 라이브러리가 데이터 로딩을 하지 않았으면 좋겠음. 그냥 라우팅 역할만 충실히 해줬으면 좋겠음.
- major 버전이 바뀔 때마다 breaking change좀 안만들었으면 좋겠음.
- v3에서 v6으로 버전업을 하는 것이 v4에서 v5보다 버전업을 하는 것 보다 쉬움. v6에서 v7로 버전업 하는 것은 그렇게 어렵지 않았음. (공식문서도 브레이킹 체인지를 의식한듯 v6 - v7 버전업은 브레이킹 체인지 없다고한다)


[Is there any quality React Router v7 guide with Vite SPA yet? : r/reactjs](https://www.reddit.com/r/reactjs/comments/1hco21n/is_there_any_quality_react_router_v7_guide_with/)

[React Router Official Documentation](https://reactrouter.com/)

[Decisions on Developer Experience | TanStack Router React Docs](https://tanstack.com/router/latest/docs/framework/react/decisions-on-dx)

[molefrog/wouter: 🥢 A minimalist-friendly ~2.1KB routing for React and Preact](https://github.com/molefrog/wouter)