1.  `refetchInterval`

    일정 주기로 refetch를 진행한다.

2.  `refetchIntervalInBackground`

    웹이 백그라운드에 있을 때도 일정 주기로 refetch를 할지 결정한다.

3.  `refetchOnMount` : `boolean` | `always` | `((query: Query) => boolean | "always")`

    `true`인 경우, 데이터가 `stale`한 경우, mount 됐을 때 refetch 한다.
    `always`인 경우, 데이터 `stale`여부에 관계 없이, mount 됐을 때 refetch 한다.

4.  `refetchOnWindowFocus` : `boolean` | `always` | `((query: Query) => boolean | "always")`

    `true`인 경우, 데이터가 `stale`한 경우, focus 됐을 때 refetch 한다.
    `always`인 경우, 데이터 `stale`여부에 관계 없이, focus 됐을 때 refetch 한다.

5.  `refetchOnReconnect` : `boolean` | `always` | `((query: Query) => boolean | "always")`

    `true`인 경우, 데이터가 `stale`한 경우, reconnect 됐을 때 refetch 한다.
    `always`인 경우, 데이터 `stale`여부에 관계 없이, reconnect 됐을 때 refetch 한다.

- [useQuery | TanStack Query React Docs](https://tanstack.com/query/latest/docs/framework/react/reference/useQuery)
