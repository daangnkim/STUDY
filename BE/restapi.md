## batch delete rest api

1. RFC7231에 근거하면 DELETE는 URI에 기재하여 단 하나의 리소스만 삭제. 이는 BULKY한 삭제가 불가능한 구조라는 단점 존재.

2. URI에 id를 쿼리스트링으로 보낼 수도 있지만, id를 담은 URL이 너무 길어지면 클라이언트와 서버 사이에 있는 프록시나 게이트웨이가 내용을 잘라버리거나 아얘 414 상태코드가 떨어질 수 있음.

3. DELETE 메서드에 id를 body에 담아보내면 RFC7231에 의하면 서버나 프록시가 이러한 요청을 거절할 수 있으며 OPENAPI나 SWAGGER도 DELETE 요청에 PAYLOAD를 보내는 것을 허용하지 않음. 또한

[Mass delete via HTTP/Rest how do you do it? | by Heiko W. Rupp | ITNEXT](https://itnext.io/mass-delete-via-http-rest-how-do-you-do-it-1bff0f5eb72d)