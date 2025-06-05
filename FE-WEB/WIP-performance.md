## code spliting

JS 비용은 파싱 비용 + 실행 비용으로 구성된다.
실제로는 실행 비용이 파싱 비용보다 비싸기 때문에 파싱 최적화보다는 실행 최적화에 집중해야한다.
코드 스플리팅, 레이지 로드등으로 불필요한 코드 실행을 피하는 것이 좋다.
단순히 번들 크기를 줄이는 것이 아닌 실행되는 코드의 양을 줄이는 것이다

실제로 webpack의 eager 모드는 chunk를 생성하지 않고 한 번에 js를 받아온 뒤 실행은 시키지 않는 옵션이다.

[✂️ Code splitting - What, When and Why - DEV Community](https://dev.to/thekashey/code-splitting-what-when-and-why-59op)
