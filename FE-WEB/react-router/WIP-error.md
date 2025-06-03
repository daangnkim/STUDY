## createBrowserRouter 내에서 errorElement vs ErrorBoundary prop의 차이

errorElement는 loaders나 actions, 컴포넌트 렌더링 중 발생하는 에러에 대한 처리를 한다.

```tsx
<Route path="/for-sale" errorElement={<Properties/>}/>
<Rotue path="/for-sale" ErrorBoundary={Properties}/>
```

댓글이 제대로 안달리는 것을 보니 차이가 없는 듯 하다. 다만 대댓글에서 말하는 것처럼 라우터 specific한 에러는 errorElement에서, 포괄적인 에러의 경우 ErrorBoundary에서 해결하면 될듯.


[What is the difference between errorElement and ErrorBoundary? · remix-run/react-router · Discussion #11456](https://github.com/remix-run/react-router/discussions/11456)

[reactjs - Error boundaries vs errorElement from react-router-dom package - Stack Overflow](https://stackoverflow.com/questions/75630048/error-boundaries-vs-errorelement-from-react-router-dom-package)