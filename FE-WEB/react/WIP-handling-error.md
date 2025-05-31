프론트엔드에서 발생하는 에러는 세가지가 존재한다.

1. 자바스크립트 에러
	에러 객체를 생성할 수 있는 7가지 Error 생성자 함수로 만들어지는 에러들이다. 예를들면 `SyntaxError`, `ReferenceError`등이 존재한다.

2. 네트워크 에러
	400번대, 500번대 에러, 혹은 connection timeout 등이 존재한다.

3. 입력 에러
	validaton 에러가 해당된다.

## 네트워크 400번대 에러의 종류

처리해본 에러는 bold 처리했다.

- **400 - Bad Request**
- **401 - Unauthorized**
- 402 - Payment Required
- **403 - Forbidden**
- **404 - Not Found**
- 405 - Method Not Allowed
- 406 - Not Acceptable
- 407 - Proxy Authentication Required
	- 401과 유사하지만, Proxy가 인증을 해야한다는 점이 다르다.
- 408 -Request Timeout
	- 서버가 클라이언트 요청을 오랫동안 받지 못하는 경우 서버가 연결을 끊는다.
- **409 - Conflict**
	- 서버의 현재 상태와 충돌할 때 발생한다. 이미 존재하는 리소스를 생성하려고 할때, 이미 삭제된 리소스를 수정하려고 할때, 동시 수정을 진행할 때
- 

## QUESTIONS

- 에러 핸들링은 프론트엔드가 하는게 좋을까? 백엔드가 하는게 좋을까?


[Frontend Error Handling: Best Practices to Log and Fix Bugs | Rollbar](https://rollbar.com/blog/guide-to-frontend-error-handling/)

[HTTP response status codes - HTTP | MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)