## 특징

fsd 아키텍쳐 구조에서 public api와 private api를 구분하기 위해 사용

## 장점

import 문의 라인 수가 비교적 줄어들어 코드가 깔끔해진다

## 단점

- 순환 참조 발생 가능성
- IDE에서 파일 경로 변경에 따른 자동 import가 되지 않음
- bundler 내에서 tree shaking을 enabled 하지 않으면 사용하지 않는 코드까지 번들에 포함됨

## References

- [JavaScript에서 Barrel 파일을 사용할 때의 장단점 | flaming.codes](https://flaming.codes/ko/posts/barrel-files-in-javascript/)

- [Barrel files and why you should STOP using them now - DEV Community](https://dev.to/tassiofront/barrel-files-and-why-you-should-stop-using-them-now-bc4)

- [Why you should avoid Barrel Files in JavaScript Modules?](https://laniewski.me/blog/pitfalls-of-barrel-files-in-javascript-modules/)
