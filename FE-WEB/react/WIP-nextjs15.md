## [Getting Started: Fetching Data | Next.js](https://nextjs.org/docs/app/getting-started/fetching-data)

- `fetch`는 기본적으로 캐싱되지 않지만, nextjs는 라우트를 프리렌더링하고 성능 개선을 위해 캐싱한다. 동적 렌더링을 원한다면 `{ cache: 'no-store' }` 를 사용하자.
- 개발하는 동안에는 `fetch`에 대한 로깅이 가능하다.


### 클라이언트 컴포넌트에서 데이터 패칭

두 가지 방법이 존재한다.

1. `use` 훅을 사용한다.
2. `SWR`, `tanstack query`를 이용한다.

### Streaming

이용하려면 `dynamicIO` 옵션이 켜져야한다. 

서버 컴포넌트에서 `async/await`를 사용한다면 nextjs는 동적 렌더링을 진행한다. 만약 데이터를 받아오는데 오래 걸린다면, 렌더링이 그만큼 막히게된다. 그래서 `streaming`을 사용한다.

`streaming`은 페이지를 몇 가지의 청크로 쪼개서 클라이언트에게 보내는 것이다. `streaming`을 구현하는 방법은 크게 두가지다.

1. loading.js를 이용하는 방법

	페이지와 동일한 폴더 위치에 loading.js 파일을 배치한다. 그러면 유저는 페이지에 진입했을 때 loading state를 보게된다. 렌더링이 끝나게 되면 loading state는 렌더링 결과물로 대체된다. route 단위로 잘 동작하는 접근 방식이다.

2. Suspense를 이용하는 방법

	조금 더 세분화된 스트리밍이 가능하다. 예를들면, `Suspense` 바운더리 바깥에 있는 컨텐츠를 즉시 보여줄수 있다. 


### Sequential Data Fetching vs Parallel Data Fetching

`async/await`를 나란히 배치하는 것이 아닌, `Promise.all`을 이용한다.

```typescript

  // 잘못된 접근
  const { username } = await params
  const artist = await getArtist(username)
  const albums = await getAlbums(username)

  // 올바른 접근
  const { username } = await params
  const artistData = getArtist(username)
  const albumsData = getAlbums(username)
 
  // Initiate both requests in parallel
  const [artist, albums] = await Promise.all([artistData, albumsData])
 

```

### Preload

