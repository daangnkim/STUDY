
## Tanstack Query의 효용성

****

## QueryClient

![[Pasted image 20250528001531.png]]

인스턴스가 생성되면 QueryClientProvider를 통해 어디서든 사용이 가능하다.

QueryCache와 MutationCache를 위한 컨테이너다.

QueryCache와 MutationCache에 대한 기본값 설정이 가능하다.

****
## QueryCache

QueryCache는 In-memory 객체이다.

serialize된 queryKey를 키로 가지며, Query 클래스의 인스턴스를 값으로 가진다.

****
## Query

Query는 useQuery를 특정 쿼리키와 함께 호출할 때 생성된다. 만약 쿼리 키에 해당하는 쿼리가 이미 존재한다면 생성되지 않는다.

그리고 각 쿼리는

- 키에 매핑되는 고유한 데이터가 존재한다.
- 데이터의 상태를 가지고있다.
- 데이터를 가지고있다.
- 메타데이터를 가지고있다.

****

## QueryObserver

Observer는 Query와 컴포넌트를 연결한다.

useQuery를 호출할때 생성된다.

QueryObserver는 하나의 Query를 구독한다.

쿼리의 어떤 프로퍼티를 컴포넌트가 사용하는지에 따라 리렌더링한다던지 등의 최적화도 여기서 발생한다. select 옵션이나 타이머에 대한 관리도 Observer가 담당한다.

****

## Query와 Query Instance의 One to Many 관계

하나의 Query는 여러개의 QueryObserver를 가질 수 있다.

예를들면 `["user", 1]`을 쿼리 키로 여러 useQuery를 호출한다면, Query는 하나인데, Observer는 여러개가 될수있다.

****

## 생성 과정

1. `["user", 1]`쿼리 키와 함께 useQuery를 호출시 해당 키에 매핑되는 Query가 존재하지 않으므로 Query가 생성된다.
2. QueryObserver가 생성되고 해당 Query를 구독한다.
3. 데이터 패칭이 진행된다.
4. `["user", 1]` 쿼리 키와 함께 useQuery를 호출시 해당 키에 매핑되는 Query를 이용한다.
5. QueryObserver가 또 생성되고 해당 Query를 구독한다.
6. 데이터가 stale하지 않는다면 네트워크 요청없기 기존 데이터를 이용한다.
7. 데이터가 업데이트 된다면 QueryObserver에게 notifiy되고, QueryObserver는 각 컴포넌트를 리렌더링한다.
8. 만약 컴포넌트가 unmount되면 QueryObserver는 제거된다.
9. QueryObserver가 남아있지 않으면, gc time에 의존하여 Query가 가비지 컬렉션된다.

****
## structural sharing

필요할 때만 렌더링하기 위해 `structural sharing`을 사용한다. 원래 react는 참조가 바뀌면 리렌더링하지만, tanstack query는 아니다.

structural sharing은 `deep compareison`을 의미한다.

일반적으로 서버로부터 데이터를 받아오면 JSON을 파싱하는 과정에서 새로운 참조가 생성되지만, 리렌더링은 일어나지 않는다. `데이터가 변경되지 않았다면 이전 참조를 사용하고, 일부만 변경됐다면 변경된 부분만 교체한다. 그래서 불필요한 컴포넌트 렌더링을 방지한다.`

아래 코드에서 name만 변경된 경우, name 필드만 변경한다.

```typescript
const userData = {
	id: 1,
	name: 'kim',
	details: {
		age: 30,
		location: 'seoul'
	}
}

const newUserData = {
	id: 1,
	name: 'park',
	details: userData.details
}
```

****
## tracked properties

tanstcak query는 `proxy object`를 이용하여 오직 useQuery가 반환하는 프로퍼티가 실제로 사용될때만 리렌더링한다. isFetching이나 isStale은 자주 변경되지만 컴포넌트에서 사용되지 않는 경우 리렌더링을 유발하지 않는다. 동작원리는 다음과 같다.

```typescript
function useCustomQuery(queryFn) {
  // React의 상태 훅
  const [_, forceRender] = React.useState({});

  // 쿼리 데이터와 상태를 저장하는 참조
  const queryInfoRef = React.useRef({
    data: null,
    isLoading: true,
    isError: false,
    isFetching: true,
    // ... 기타 상태
  });

  // 구독된 속성을 추적하는 참조
  const subscribedPropsRef = React.useRef(new Set());

  // 이전 상태 스냅샷을 저장하는 참조
  const prevStateRef = React.useRef({ ...queryInfoRef.current });

  // 상태 업데이트 로직
  const updateQueryInfo = (newInfo) => {
    // 새 상태를 저장
    const oldInfo = queryInfoRef.current;
    queryInfoRef.current = { ...oldInfo, ...newInfo };

    // 구독된 속성 중 변경된 것이 있는지 확인
    const hasSubscribedPropsChanged = Array.from(
      subscribedPropsRef.current
    ).some((prop) => oldInfo[prop] !== queryInfoRef.current[prop]);

    // 구독된 속성이 변경되었을 때만 리렌더링 트리거
    if (hasSubscribedPropsChanged) {
      console.log("구독된 속성 변경 감지, 리렌더링 트리거");
      forceRender({});
    } else {
      console.log("변경된 속성이 구독되지 않음, 리렌더링 없음");
    }

    // 이전 상태 업데이트
    prevStateRef.current = { ...queryInfoRef.current };
  };

  // 데이터 페칭 로직 (간략화)
  React.useEffect(() => {
    const fetchData = async () => {
      try {
        updateQueryInfo({ isLoading: true, isFetching: true });
        const result = await queryFn();
        updateQueryInfo({
          data: result,
          isLoading: false,
          isFetching: false,
          isStale: false,
        });
      } catch (error) {
        updateQueryInfo({
          isError: true,
          error,
          isLoading: false,
          isFetching: false,
        });
      }
    };

    fetchData();

    // 주기적인 백그라운드 리페칭 예시
    const interval = setInterval(() => {
      updateQueryInfo({ isFetching: true, isStale: true });
      fetchData();
    }, 5000);

    return () => clearInterval(interval);
  }, []);

  // 커스텀 게터가 있는 결과 객체 생성
  const result = {};

  // 각 상태 속성에 대한 게터 설정
  Object.keys(queryInfoRef.current).forEach((key) => {
    Object.defineProperty(result, key, {
      get() {
        // 속성에 접근할 때 구독으로 표시
        subscribedPropsRef.current.add(key);
        return queryInfoRef.current[key];
      },
      enumerable: true,
    });
  });

  return result;
}
```

****
## select

데이터를 변형하거나 일부만 선택할 때 사용한다. 가장 중요한 특징은 불필요한 리렌더링을 방지한다는 점이다.

```typescript
export const useTodos = (select) => {
  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    select,
  })
}

export const useTodoCount = () => {
  return useTodos((data) => data.length)
}
```

위 useTodoCount를 사용하는 컴포넌트는 todos 배열의 길이가 변경될 때만 리렌더링된다.

## select와 memoization

select는 인자로 들어오는 data가 바뀌거나 함수의 참조가 바뀔 때 재실행된다.

****

## Query의 상태

`inactive query`란 Observer가 존재하지 않는 Query를 의미한다. Cache에는 존재하지만 어떠한 컴포넌트에 의해서도 사용되지 않는다. 

****

## Stale-While-Validate




[Inside React Query | TkDodo's blog](https://tkdodo.eu/blog/inside-react-query)

[Deep dive in React Query caching mechanism with architecture | by Jordan | Medium](https://medium.com/@yunyoung-jordan-choi/deep-dive-in-react-query-caching-mechanism-with-architecture-89513ceafbf1)

[[React Query] useQuery 동작원리(1) | Time Gambit](https://www.timegambit.com/blog/digging/react-query/01)

[Render Optimizations | TanStack Query React Docs](https://tanstack.com/query/latest/docs/framework/react/guides/render-optimizations)
