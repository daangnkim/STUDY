[Tanstack react query architecture : r/reactjs](https://www.reddit.com/r/reactjs/comments/18z3nsi/tanstack_react_query_architecture/)의 댓글들에 적힌 내용들 + [Separate API Layers In React Apps - 6 Steps Towards Maintainable Code](https://profy.dev/article/react-architecture-api-layer)만 따른다면 어느정도 유지 보수 가능한 폴더가 만들어진다.

1. 서비스를 분리한다

	여러 hook들에서 재사용 가능하게 만든다.

2. custom hook을 만든다

	그렇지 않으면 파일이 커지고 재사용도 힘들어진다.

3. `domain.serivce.ts`, `domain.api.ts`, `domain.type.ts` 형태로 파일을 나눈다.

	하나의 파일 안에 훅들을 같이 둠으로써 query key 관리도 쉬워진다.

4. query-key factory를 만든다

	key의 중복, 네이밍을 걱정하지 않아도 된다. key가 업데이트 되는 경우, key를 사용하는 모든 곳을 찾아서 수정하지 않아도 된다.

5. apiClient를 만든다

	axios.get, axios.post를 직접 호출하는 것이 아닌, apiClient를 호출하게 만듬으로써 fetch API, ky, GraphQL 등으로 라이브러리를 바꿔도 apiClient만 고치면 되도록 만든다.

