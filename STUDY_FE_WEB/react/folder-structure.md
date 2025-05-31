크게 다음과 같이 분류할 수 있을 것 같다.

1. Grouping by file type

	직관적이어서 폴더 구조 이해에 대한 부담이 적다는 장점이 있지만, 어떤 파일이 어떤 feature를 다루는지 잘 드러나지 않는다는 단점이 존재한다.

	개인적으로 로그인을 제외한 feature가 단 하나만 존재하는 경우에 적합하다고 생각한다. 예를들면, 본인이 만들었더 `PC-SDM`은 로그인 후 설문을 작성하고 결과를 보여주는, 오로지 설문 feature만 존재하는 웹사이트였다.

2. Grouping by file type and then Grouping by feature type

	파일 타입을 먼저 탐색하고, 기능을 탐색하는 방식으로 구성된다.

	파일 유형별 일관성 유지가 용이하며 feature에 대한 고려가 가능해지지만 feature 관련 로직이 폴더 전반에 퍼지게되고, 공통 폴더에 대한 위치가 모호해진다.
	
3. Grouping by feature type and then Grouping by file type

	기능을 탐색하고 파일 타입을 탐색하는 방식으로 구성된다.

	feature 관련 로직이 뭉치게된다.

4. Grouping by the scope of influence, Grouping by feature type, Grouping by file type (FSD)


## Feature 관련한 로직을 한데 뭉치는 이유가 뭘까?



## index.ts as public API

Feature Sliced Design에서 소개되는 내용인데, 특정 폴더가 제공하는 public API를 index.ts에 둠으로써 다음 네 가지 이점을 누릴 수 있다.

1. import 경로가 짧아진다
2. api의 내부 폴더 구조를 신경쓰지 않아도 된다.
3. local api와 public api 구분이 가능해진다.

## QUESTIONS

1. 비지니스를 고려하여 폴더 구조를 잡으면 어떤 점이 좋지?
2. 프로젝트가 복잡해질수록 domain 따라 나누게 되는 이유는 머지?


[⚛️ Folder Structures in React Projects - DEV Community](https://dev.to/itswillt/folder-structures-in-react-projects-3dp8)

[feature-sliced/documentation: 🍰 Architectural methodology for frontend projects](https://github.com/feature-sliced/documentation)

[Screaming Architecture - Evolution of a React folder structure - DEV Community](https://dev.to/profydev/screaming-architecture-evolution-of-a-react-folder-structure-4g25)

