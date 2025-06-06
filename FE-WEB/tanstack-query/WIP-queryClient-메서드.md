
##### 1. fetchQuery vs prefetchQuery

- 캐시가 있으면 캐싱된 데이터 사용
- 결과를 반환받길 원하면 prefetchQuery 사용

##### 2. invalidateQueries vs refetchQueries vs cancelQueries vs removeQueries vs resetQueries

refetch는 말 그대로 query를 다시 fetch한다.
cancel은 서버에 보낸 요청을 취소한다. 낙관적 업데이트 사용시 이용 가능하다.
remove는 캐싱된 데이터를 지운다.
reset은 데이터를 초기값으로 돌린다. initialData가 있다면 그 값으로 돌아간다.






[QueryClient | TanStack Query Docs](https://tanstack.com/query/v5/docs/reference/QueryClient)
