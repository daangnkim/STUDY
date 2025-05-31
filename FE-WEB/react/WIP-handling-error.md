프론트엔드에서 발생하는 에러는 세가지가 존재한다.

1. 자바스크립트 에러
	에러 객체를 생성할 수 있는 7가지 Error 생성자 함수로 만들어지는 에러들이다. 예를들면 `SyntaxError`, `ReferenceError`등이 존재한다.

2. 네트워크 에러
	400번대, 500번대 에러, 혹은 connection timeout 등이 존재한다.

3. 입력 에러
	validaton 에러가 해당된다.

## 네트워크 400번대 에러의 종류

- 400 - Bad Request
- 401 - Unauthorized
- 402 - Payment Required
- 403 - Forbidden
- 404 - Not Found
- 405 - Method Not Allowd
- 


## QUESTIONS

- 에러 핸들링은 프론트엔드가 하는게 좋을까? 백엔드가 하는게 좋을까?


[Frontend Error Handling: Best Practices to Log and Fix Bugs | Rollbar](https://rollbar.com/blog/guide-to-frontend-error-handling/)

[HTTP response status codes - HTTP | MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)