SOP(Same Origin Policy)를 우회하여 사용자가 원하지 않는 행위를 취하게 만든다.

### How does CSRF work?
1. 사용자가 취약점이 있는 웹사이트에 로그인해있음
2. 이메일, 메시지, SNS를 통해 악성사이트 링크를 전송받음
3. 링크 진입시 악성사이트가 취약점이 있는 웹사이트 쿠키를 이용하여 계정 정보 변경과 관련한 요청을 서버에 전달

### CSRF Header
CSRF 막기 위한 장치로, CSRF 헤더에 값을 실어서 보내면 백엔드가 DB에 저장돼있는 값과 비교함

### CSRF vs X-CSRF-TOKEN 
둘 중 아무거나 하나만 쓰면 된다. 단순히 프레임워크에 따른 이름 차이임

### CSRF 디펜스 방법

1. CSRF Token

2. SameSite
	Strict인 경우 같은 도메인에서의 요청만 쿠키가 전송됨
	Lax인 경우 같은 도메인에서의 요청 + 다른 도메인에서의 GET 요청

3. Referer

### CORS, CSRF, SOP, withCredentials의 관계

SOP는 하나의 도메인에서 다른 도메인에 있는 리소스에 접근이 불가능하도록 막는 정책. CORS는 이 정책에 대한 예외 조항을 두는 것. CSRF는 사용자가 의도치 않은 동작을 수행하도록 하는 웹 공격임. wtihCredentials가 false이면 CSRF는 자연스럽게 발생하지 않음. withCredentials의 기본값이 true가 아닌 이유는 앞선 이유로서 해소가됨.

### SOP 조항이 있으면 withCrendetials는 필요가 없지 않나?

withCrendentials가 False여도 SOP 조항이 막아주는 것 아닌가? 그러니까, 악성 도메인, 취약 도메인이 있다고 할때, 악성 도메인에서 취약 도메인으로 요청을 보내도, SOP 조항이 이를 해결해주는 것 아닌가? 라는 생각이듬. 리소스를 받아오는 것은 막을 수 있으나 요청 자체를 막을 수는 없다는게 중요함. 즉 GET 요청은 막을 수 있더라도, POST, PATCH 등의 요청은 막을 수가 없음.

### withCredentials

서로 다른 도메인에 요청을 보낼 때 요청에 인증 정보를 담아 보낼지를 결정하는 항목. 인증 정보라 함은 (1) 쿠키 혹은 (2) Authorization Header가 존재





[Understanding CORS and CSRF: A Guide for Spring Security | by Suresh | Medium](https://medium.com/@CodeWithTech/understanding-cors-and-csrf-a-guide-for-spring-security-feb34b81a3a4)
[security - Difference between CSRF and X-CSRF-Token - Stack Overflow](https://stackoverflow.com/questions/34782493/difference-between-csrf-and-x-csrf-token)
[What is CSRF (Cross-site request forgery)? Tutorial & Examples | Web Security Academy](https://portswigger.net/web-security/csrf)
[Referer - HTTP | MDN](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/Referer)
[Do SameSite Cookies Solve CSRF?. SameSite attribute was introduced to… | by Airman | Medium](https://airman604.medium.com/do-samesite-cookies-solve-csrf-6dcd02dc9383)[🍪 CORS 쿠키 전송하기 (withCredentials 옵션)](https://inpa.tistory.com/entry/AXIOS-%F0%9F%93%9A-CORS-%EC%BF%A0%ED%82%A4-%EC%A0%84%EC%86%A1withCredentials-%EC%98%B5%EC%85%98)
