번들을 하는 경우 다양한 포맷으로 생성이 가능하다.

- CJS
- ESM
- Browser Compatible Format
- UMD, AMD

Browser Compatible Format이란, 아래와 같이 `<script>`태그에 바로 넣어서 쓸 수 있는 javascript 파일을 일컬음.

```typescript
<script src="library.min.js"></script>
<script>
  MyLibrary.doSomething(); // ✅ 바로 사용 가능
</script>
```

이러한 구조는 IIFE를 통해서, 필요한 모듈만 전역 객체에 노출함으로써 만들어질 수 있다.

```typescript
(function() {
  // 이 안의 변수들은 외부에서 접근 불가 (private)
  var myFunction = function() {
    console.log('Hello!');
  };
  
  var helperFunction = function() {
    // 이 함수는 외부에서 못 봄
  };
  
  // 필요한 것만 전역으로 노출
  window.MyLibrary = {
    myFunction: myFunction
    // helperFunction은 노출 안 함
  };
})(); // 즉시 실행!
```


[How to publish packages that can be used in browsers and Node | by Zell Liew | We’ve moved to freeCodeCamp.org/news | Medium](https://medium.com/free-code-camp/how-to-publish-packages-that-can-be-used-in-browsers-and-node-c51274dca77c)

[Create a Simple npm Library to Use in and Out of the Browser | by Danny Perez | Better Programming](https://medium.com/better-programming/create-a-simple-npm-library-to-use-in-and-out-of-the-browser-700f207eb73)

[2024 JavaScript bundlers comparison | Tony Cabaye](https://tonai.github.io/blog/posts/bundlers-comparison/)
