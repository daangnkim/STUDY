클라이언트에서 클라이언트로 전달하는 경우와 클라이언트에서 서버로 전달하는 경우가 존재한다.
클라이언트에서 클라이언트로 전달하는 방법은

1. Query Parameter
2. Path Parameter
3. History API
4. Global State
5. Storage

가 존재하고,

클라이언트에서 서버로 전달하는 방법은

1. Query Parameter
2. Path Parameter
3. Body Parameter

등이 존재한다.

클라이언트에서 클라이언트로 전달하는 방법 중 페이지 네비게이션시 전달하는 방법에 대해서 말해보면,

## 페이지 네비게이션시 데이터 전달 방법

크게 세 가지가 존재한다

1. Query Parameter
2. Path Parameter
3. History API

## [1] Query Parameter

특징은

- 소팅이나 필터처럼 데이터 요청의 범위를 변경한다.
- 키, 값 형태로 데이터를 저장한다.

보통은

- 필터링을 위해 사용된다.

## [2] Path Parameter

특징은

- 리소스를 구분하기 위해 사용된다.
- SEO에 영향을 줄수있다.
- 스트링 형태로 데이터를 저장한다.

## [3] History State

[Path Param & Body Param & Query Param | by Ladynobug | Medium](https://medium.com/@LadyNoBug/api-design-f37bb37acf13)
