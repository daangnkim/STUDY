## REST, RESTful

REST를 만든 로피 필딩은 발표 당시 웹이 HTTP를 제대로 사용하고 있지 않아 HTTP의 장점을 최대한 활용할 수 있는 아키텍처인 REST를 제공했다.

REST(Representation State Transfer)는 HTTP를 기반으로 클라이언트가 서버의 리소스에 접근하는 방식을 규정한 아키텍트다.

REST를 지킨 서비스 디자인을 RESTful이라고 부르며, REST API는 REST를 기반으로 API를 구현한 것을 의미한다. REST API는 크게 `자원(uri)`, `행위(http methods)`, `표현(payload)`로 구성된다.

REST는 지켜야할 제약이 6가지 존재한다.

## [1] Uniform Interface

인위적인 제약을 두지 않은 `principle of generality`(Y2K 사태처럼 소프트웨어를 설계할 때 미래에 발생할 수 있는 변화까지 고려해야한다.)를 적용함으로서 시스템을 간소화하고 동작을 예측 가능하게 만든다. 핵심 원칙 네 가지가 존재한다

1. 모든 자원은 고유하게 식별된다.
2. `표현`을 통한 자원 조작이 가능하다.
   DB는 JSON으로 표현되지 않지만, JSON이나 XML등의 표현을 응답받고 이 표현을 수정하여 DB를 수정할 수 있다.
3. 자기 서술적 메세지
   표현은 메세지를 처리하는 방법에 대한 충분한 정보를 포함해야한다. `Content-Type: application/json`을 받으면 클라이언트는 응답을 JSON으로 해석한다.
4. HATEOAS (Hypermedia as the engine of application state)
   초기 URI만 알고있어야하며, 하이퍼링크를 통해 다른 모든 자원과 상호작용 가능해야한다.

   ```javascript
   {
   "id": 123,
   "name": "홍길동",
   "_links": {
    "self": { "href": "/users/123" },
    "orders": { "href": "/users/123/orders" },
    "update": { "href": "/users/123", "method": "PUT" }
    }
   }
   ```

쉽게말하면 클라이언트와 서버간 인터랙션을 위한 일관된 인터페이스를 제공하는 것이다. 예를들면 HTTP 메소드를 활용하거나 URI를 자원 식별자로 사용하는 것이다.

## [2] Client-Server

클라이언트와 서버간 관심사가 분리되어 각 컴포넌트가 독립적으로 개발 가능하도록 만든다. 이렇게하여 UI를 여러 플랫폼에 이식 가능하도록 만든다.

## [3] Stateless

서버는 상태를 저장하지 않기 때문에 클라이언트에서 서버로의 요청은 요청에 필요한 모든 정보가 담겨져있어야 한다. 만약 세션 ID를 생성하는 방식등의 Stateful한 API를 구현한다면, 완전한 RESTful 아키텍처가 아니다.

## [4] Cacheable

응답은 cacheable하거나 non-cacheable한지에 대한 라벨링이 존재해야한다. 캐시가 가능하다면 클라이언트는 이 캐시 데이터를 재사용할 권리를 가지고 있다.

## [5] Layered Architecture

클라이언트가 서버와 통신할 때 여러 계층(로드 밸런서 계층, 인증 계층, 애플리케이션 서버 개층 등)을 통과할 수 있다.

## [6] 코드 온 디맨드

서버가 클라이언트에게 실행 가능한 코드를 전송하여 클라이언트의 기능을 확장한다.

## 이러한 원칙은 클라이언트 URL을 설계할 때도 적용돼야 할까?

URL을 `자원 식별자`로 쓰잔느 철학은 API나 웹 페이지 모두에 유효하다.

## References

- [What Is a REST API? Examples, Uses & Challenges | Postman Blog](https://blog.postman.com/rest-api-examples/)

- [What is REST?: REST API Tutorial](https://restfulapi.net/)

- 모던 자바스크립트 Deep Dive
