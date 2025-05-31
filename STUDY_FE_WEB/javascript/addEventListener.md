## addEventListener는 핸들러 함수의 파라미터에 어떤 타입을 줘야할까?

`WindowEventMap`을 보면 이벤트와 핸들러 파라미터 타입간의 매핑이 꽤 구체적으로 작성돼있다. 이를 참고하면 된다.

## addEventListener의 옵션

다음 세 가지 옵션이 존재한다.

1. `once`
2. `capture`
3. `passive`

## `once` (boolean)

단 한번만 호출되고 삭제되도록 만든다.

## `capture` (boolean)

옵션을 적용하면 자손 DOM 요소에 전달되기 전에 `capture`옵션이 적용된 listener가 먼저 트리거된다.

```javascript
var parent = document.getElementById("parent"),
  child = document.getElementById("child");

parent.addEventListener("click", function () {
  console.log("click on parent element");
});

child.addEventListener("click", function () {
  console.log("click on child element");
});
```

위 코드를 실행하면 click on child element, click on parent element 순으로 콘솔이 찍히지만, 부모 요소에 capture 옵션을 적용하면 click on parent element, click on child element 순으로 콘솔이 찍힌다.

## `passive` (boolean)

옵션이 적용되면 `preventDefault`가 절대 실행되지 않도록 만든다. 설명 핸들러 내에서 e.preventDefault()를 호출하더라도 아무 동작도 하지 않는다. safari는 기본적으로 false이고, 나머지 브라우저는 true이다.

브라우저는 `touchstart`, `touchmove`같은 이벤트 핸들러의 결과를 기다리고 스크롤을 실행한다. 왜냐하면 e.preventDefault()가 호출되면 스크롤 자체를 방지할 수 있기 때문이다. 이로인해 스크롤 동작의 지연이 발생한다. 그러므로 `passive` 옵션을 true로 만들어 이벤트 핸들러의 결과를 기다리지 않고 스크롤이 되도록 만든다.

## References

[javascript - How to properly type event handler in wrapped addEventListener call in TypeScript - Stack Overflow](https://stackoverflow.com/questions/59940863/how-to-properly-type-event-handler-in-wrapped-addeventlistener-call-in-typescrip)

[How to handle events in Javascript and tips on keeping them organised. | CodyHouse](https://codyhouse.co/blog/post/handle-events-javascript)

[EventTarget: addEventListener() method - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener)

[javascript - What are passive event listeners? - Stack Overflow](https://stackoverflow.com/questions/37721782/what-are-passive-event-listeners)
