## CJS

- ESM보다 먼저 나온 Nodejs를 위한 모듈 시스템으로, ServerJS라고도 부르며 주로 서버 사이드 애플리케이션에서 사용됨
- 파일 확장자는 `cjs`
- 모듈을 작성하는 옛날 방식으로, `synchronous`하며 `dynamic`함. 무엇보다 표준이 아님
- `module.exports`와 `require` 사용
- `synchronous` 하게만 동작하기 때문에 어떤 모듈을 다 불러오기 전까지 다음 코드는 실행되지 않음
-  `dynamic` 하게 동작함. 

	모듈을 코드 맨 위에서 미리 불러올 필요가 없음. 실행 중(run-time)에 필요할 때 `require`할 수 있음.

	```typescript
	if (someCondition) {
	  const debug = require('debug'); // 조건이 맞을 때만 로드됨
	}	
	```

## ESM

- ESM이 있기 전까지는 브라우저에는 모듈 시스템이 없었음. 
- ESM은 ES6와 함께 등장함.
- 파일 확장자는 `mjs`
-  모듈을 작성하는 표준 방식으로, `asynchronous`하며 `static`함
-  `export`와 `import`사용
- `asynchronous`

	다른 모듈을 불러올 때 까지 프로세스나 작업들의 실행을 차단하지 않음

	```typescript
	// 이 import가 실행되어도 다른 코드의 실행을 막지 않음
	import { someFunction } from './utils.js';
	
	// 다른 코드들이 계속 실행될 수 있음
	console.log('다른 작업 실행 중...');
	```

- `static`

	모듈의 구조와 의존성이 컴파일 시점에 미리 분석되고 결정됨. 

	아래와 같이 dynamic하게 작성 불가

	```typescript
	// ✅ 정적 - 컴파일 시점에 분석 가능
	import { utils } from './utils.js';
	
	// ❌ 동적 - ESM에서는 불가능
	if (condition) {
	  import { something } from './conditional.js'; // 이런 식으로 사용 불가
	}
	```

	그리고 위와 같은 static한 특성 때문에 컴파일 타임에 `tree-shake`가 가능하다.

CJS나 ESM이 없다면 모듈화가 안되어 코드를 파일 하나에 작성했을 것임.

## ESM은 왜 나오게 됐을까?

그렇다면 ESM은 왜 나왔을까? 모듈 시스템에 대한 표준이 필요했고, 프론트엔드 애플리케이션에서 비동기 함수에 대한 요구가 있었기 때문임.

## CJS는 제거하고 ESM만 사용하는게 어떨까?

옛날 엔지니어들은 ESM을 받아들이길 거부했음. 

Node.js는 CJS에 강하게 의존하기 때문에 ESM을 사용하려면 별도의 세팅이 필요했음. 이미 수많은 프로젝트들이 CJS를 사용하기도 했음.

브라우저 입장에서는 CJS를 사용하려면 transpiler가 필요했음.

그래서 함부로 제거할 수가 없음

## AMD (Asynchronous Module Definition)

- 모듈을 asynchronous하게 import한다.
- 문법이 직관적이지 않다.
- frontend 진영을 위해 개발됐다.

## UMD (Universal Module Definition)

- 프론트엔드 백엔드 모두 잘 동작한다.
- UMD는 모듈 시스템 구성을 위한 패턴에 가깝다.


## 외부 패키지 설치의 문제점

과거 페이스북 인스타그램이 리액트에 의존 -> 리액트는 바벨에 의존 -> 바벨은 left-pad 모듈에 의존.

left-pad 모듈 개발자가 자신의 npm 패키지들 모두 삭제 -> 바벨 실패 -> react 빌드 실패 -> 페이스북 인스타그램 다운으로 이어졌던 이슈가 존재.

[Snapstromegon - CommonJS vs. ESM](https://www.hoeser.dev/blog/2023-02-21-cjs-vs-esm/)

[The Ongoing War Between CJS & ESM: A Tale of Two Module Systems - DEV Community](https://dev.to/greenteaisgreat/the-ongoing-war-between-cjs-esm-a-tale-of-two-module-systems-1jdg)

[How to publish packages that can be used in browsers and Node | by Zell Liew | We’ve moved to freeCodeCamp.org/news | Medium](https://medium.com/free-code-camp/how-to-publish-packages-that-can-be-used-in-browsers-and-node-c51274dca77c)

[What the heck are CJS, AMD, UMD, and ESM in Javascript? - DEV Community](https://dev.to/iggredible/what-the-heck-are-cjs-amd-umd-and-esm-ikm)