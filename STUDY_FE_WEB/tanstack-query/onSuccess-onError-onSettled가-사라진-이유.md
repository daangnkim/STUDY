tanstack query `v5` 버전부터 onSuccess, onError, onSettled callback 함수가 사라진 이유에 대해서 알아보자.

useQuery의 onSuccess, onError, onSettled는 몇 가지 문제점이 존재한다.

1.  useEffect와 다른 동작을 기대하지만 useEffect와 동일하게 동작한다.

    useGetTodoList는 서버에서 데이터를 가져온다. 에러가 존재하면 토스트 메세지를 보여준다.
    만약 하나의 페이지에서 useGetTodoList를 두 번 호출한다면, 토스트 메세지가 두 번 보여진다.

    onError 콜백에서 이러한 에러를 핸들링해도 마찬가지이다. 한 번만 보여지지 않는다.

2.  additional render cycle

    ```jsx
    export function useTodos() {
      const [todoCount, setTodoCount] = React.useState(0);
      const { data: todos } = useQuery({
        queryKey: ["todos", "list"],
        queryFn: fetchTodos,
        //😭 please don't
        onSuccess: (data) => {
          setTodoCount(data.length);
        },
      });

      return { todos, todoCount };
    }
    ```

3.  the intermediate render cycle will contain false values

    1. todos will be undefined and length will be 0. That's the initial state while the Query is fetching. This is correct.

    2. todos will be an Array of length 5 and todoCount will be 0. That's the in-between render cycle where useQuery has already run (and so has onSuccess), but setTodoCount was scheduled. This is wrong because values are not in-sync.

    3. todos will be an Array of length 5 and todoCount will be 5. That is the final state and is correct again.

## 그래서 어떻게 핸들링 해야할까?

1. global callback 이용하기

   ```jsx
   const queryClient = new QueryClient({
     queryCache: new QueryCache({
       onError: (error, query) => {
         if (query.meta.errorMessage) {
           toast.error(query.meta.errorMessage);
         }
       },
     }),
   });

   export function useTodos() {
     return useQuery({
       queryKey: ["todos", "list"],
       queryFn: fetchTodos,
       meta: {
         errorMessage: "Failed to fetch todos",
       },
     });
   }
   ```

2. `Error Boundary` 이용하기

3. `error` 프로퍼티 이용하기

- [Breaking React Query's API on purpose | TkDodo's blog](https://tkdodo.eu/blog/breaking-react-querys-api-on-purpose#react-query-v5)

- [React Query Error Handling | TkDodo's blog](https://tkdodo.eu/blog/react-query-error-handling)
