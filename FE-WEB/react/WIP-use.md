`Promise`나 `context`의 값에 접근이 가능하며 `use`는 훅이 아니다.

몇 가지 특징은 다음과 같다

1. 훅과의 공통점과 차이점

- 훅처럼 함수형 컴포넌트와 훅 내에서 호출되야하지만, if문과 반복문 내에서도 호출이 가능하다. 그렇기에 `useContext`보다 유연하게 사용이 가능하여 useContext 대체가 가능하다.

2. `use`에 전달되는 `Promise`는 캐시돼야한다.

- 안그러면 무한 루프에 빠진다.

3. `Promise`는 serialized 돼야한다.

4. `Suspense`와 `Error Boundary`와 통합이 가능하다.

## 예시

### 1. `Context`의 사용

```typescript
"use client";

import React, { use } from "react";
import ExampleSection from "../shared/ExampleSection";
import { ThemeContext } from "./themeContext";

const UseApiClientContext: React.FC = () => {
  const theme = use(ThemeContext);
  return (
    <ExampleSection>
      <code>System theme is: {theme}</code>
    </ExampleSection>
  );
};
export default UseApiClientContext;
```

### 2. 서버에서 클라이언트로의 Streaming

리액트의 Suspense와 Concurrent 기능을 이용하여 데이터를 서버에서 클라이언트로 streaming한다.

먼저 다음과 같이 RSC에서 promise를 Client Component에 전달한다.

```typescript
import React, { Suspense } from "react";
import UseApiClientComponent from "./use-api-client-component";
import ErrorBoundary from "./ErrorBoundry";
import ExampleSection from "../shared/ExampleSection";

const UseApiServerComponent: React.FC = () => {
  // promise is created on the server component
  const promise = fetch(`https://jsonplaceholder.typicode.com/posts/1`, {
    cache: "no-store",
  }).then((res) => res.json());
  return (
    <ExampleSection>
      {/* 
                we have an suspense around client component.
                This makes sure, we see a fallback UI until 
                the promise is resolved.
            */}

      <ErrorBoundary>
        <Suspense fallback={<p>Streaming data...</p>}>
          <UseApiClientComponent data={promise} />
        </Suspense>
      </ErrorBoundary>
    </ExampleSection>
  );
};
export default UseApiServerComponent;
```

그리고 다음과 같이 `use`를 사용한다.

```typescript
"use client";

import React, { use } from "react";

const UseApiClientComponent: React.FC<{ data: any }> = ({ data }) => {
  // we pass the promise to use.
  const response = use(data);
  // and we get data.
  return <pre>{JSON.stringify(response, undefined, 2)}</pre>;
};
export default UseApiClientComponent;
```

## Reference

[use – React](https://react.dev/reference/react/use)

[Use API— React 19 Deep dive: Part 1 | by Harshana Abeyaratne | Medium](https://harshq.medium.com/use-api-react-19-deep-dive-part-1-5e6a8383127e)

[개발자 단민 | React 19의 새로운 API, use](https://www.jeong-min.com/64-react-19-use/)
