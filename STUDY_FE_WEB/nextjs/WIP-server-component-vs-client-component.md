## 클라이언트와 서버 각각의 환경이 지닌 능력과 제약

각각의 환경은 `각각의 능력과 제약`이 존재한다. 만약 렌더링과 데이터 패칭의 위치를 서버로 옮긴다면, 클라이언트로 전달되는 코드의 양을 줄이고 웹사이트의 성능을 올릴 수 있을 것이다. 하지만 interactive ui를 구현하는데 제약이 있을 수 있다.

## 네트워크 바운더리

환경을 구분짓는 conceptual line이다. 컴포넌트 트리에서 `network boundary`를 어디다둘지 선택할 수 있다.

데이터를 패칭해서 포스트를 그리는 것을 서버 컴포넌트를 이용해서 서버에서할 수 있고, 좋아요 버튼과 같은 interactive ui는 클라이언트 컴포넌트를 이용해서 클라이언트에서 렌더링할 수 있다.

이때 컴포넌트들은 두 가지 모듈 그래프로 분리된다. 첫 번째는 서버 컴포넌트를 모두 포함하는 서버 모듈 그래프(혹은 트리) 두 번째는 클라이언트 컴포넌트를 모두 포함하는 클라이언트 모듈 그래프(트리) 두 가지로 나뉘게 된다.

서버 컴포넌트는 그려진 후 클라이언트에게 React Server Component Payload (RSC)를 보내게 된다. 그리고 RSC는 다음 두 가지를 포함한다.

1. 서버 컴포넌트의 렌더링된 결과
2. 클라이언트 컴포넌트가 렌더링될 위치와 클라이언트 컴포넌트에 대한 자바스크립트 레퍼런스 파일들

이 정보를 이용하여 서버 컴포넌트와 클라이언트 컴포넌트를 consolidate 하여 최종적으로 dom을 업데이트한다.

## REFERENCES

- [React Foundations: Server and Client Components | Next.js](https://nextjs.org/learn/react-foundations/server-and-client-components)

- [Understanding Astro islands architecture - LogRocket Blog](https://blog.logrocket.com/understanding-astro-islands-architecture/)
