react-router가 지원하는 prefetch는 스플릿된 코드를 prefetch 하는 것 그리고 server data를 fetch 하는 것 모두를 의미한다.

`Link`태그를 prefetch 옵션과 함께 사용하면 스플릿된 코드나 server data를 prefetch한다. 다만 server data를 prefetch 하는 경우에는 무조건 loader가 들어가있어야한다.

[Link | React Router](https://reactrouter.com/api/components/Link#prefetch)