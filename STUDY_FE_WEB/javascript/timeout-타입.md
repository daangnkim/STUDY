아래와 같이 setTimeout의 리턴 타입을 number로 작성하는 경우,

```typescript
let timeout: number = setTimeout(() => {}, 3000);
```

다음과 같은 에러가 발생한다.

> Type 'Timeout' is not assignable to type 'number'.ts(2322)

setTimeout은 서버에서 실행될 수도 있고, 브라우저에서도 실행될 수 있는데, 만약 프로젝트 내에 `@types/node`가 존재한다면 typescript 컴파일러가 이를 서버에서 실행할 것이라고 생각하는 것이다.

이 에러를 해결하려면 두 가지 방법이 존재한다.

첫 번째는 애초부터 어떤 모듈을 사용할 것인지 명시하는 것이다.

```typescript
// 서버 사이드에서 사용하는 경우
import { setTimeout } from "timer";
const timeId = setTimeout(() => {}, 3000);
// 혹은
const timerId = global.setTimeout(() => {}, 3000);

// 브라우저에서 사용하는 경우
const timerId = window.setTimeout(() => {}, 3000);
```

두 번째는 타입스크립트 유틸리티 타입을 이용하는 것이다.

```typescript
let timer: ReturnType<typeof setTimeout> = setTimeout(() => { ... });
clearTimeout(timer);
```

isomorphic한 환경에서도 대응이 되기 때문에 두 번째 방법이 선호된다.
그렇다면 NodeJS.Timeout은 무슨 타입일까? 타입 파일을 까보면 다음과 같다.

```typescript
class Timeout implements Timer {
  /**
   * When called, requests that the Node.js event loop _not_ exit so long as the`Timeout` is active. Calling `timeout.ref()` multiple times will have no effect.
   *
   * By default, all `Timeout` objects are "ref'ed", making it normally unnecessary
   * to call `timeout.ref()` unless `timeout.unref()` had been called previously.
   * @since v0.9.1
   * @return a reference to `timeout`
   */
  ref(): this;
  /**
   * When called, the active `Timeout` object will not require the Node.js event loop
   * to remain active. If there is no other activity keeping the event loop running,
   * the process may exit before the `Timeout` object's callback is invoked. Calling `timeout.unref()` multiple times will have no effect.
   * @since v0.9.1
   * @return a reference to `timeout`
   */
  unref(): this;
  /**
   * If true, the `Timeout` object will keep the Node.js event loop active.
   * @since v11.0.0
   */
  hasRef(): boolean;
  /**
   * Sets the timer's start time to the current time, and reschedules the timer to
   * call its callback at the previously specified duration adjusted to the current
   * time. This is useful for refreshing a timer without allocating a new
   * JavaScript object.
   *
   * Using this on a timer that has already called its callback will reactivate the
   * timer.
   * @since v10.2.0
   * @return a reference to `timeout`
   */
  refresh(): this;
  [Symbol.toPrimitive](): number;
  /**
   * Cancels the timeout.
   * @since v20.5.0
   */
  [Symbol.dispose](): void;
}
```

타이머가 활성화되어 있는 동안 Node.js의 이벤트 루프가 계속 실행되게 만들거나, 타이머가 있더라도 다른 작업이 없는 경우 Node.js 프로세스를 종료하게 만드는 메서드들이 존재한다.

## References

[TypeScript - use correct version of setTimeout (node vs window) - Stack Overflow](https://stackoverflow.com/questions/45802988/typescript-use-correct-version-of-settimeout-node-vs-window)
