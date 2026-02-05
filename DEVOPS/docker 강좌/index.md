https://www.udemy.com/course/docker-kubernetes-the-practical-guide/ 강의 기록

## docker와 container

도커는 컨테이너를 관리하는 기술

`코드 패키지`라고 불린다.
독립적인, 표준화된 애플리케이션 패키지를 의미한다. 

프로덕션 환경과 정확히 동일한 개발 환경을 보장한다.
컨테이너는 항상 동일한 실행을 보장한다.
예를들면 컨테이너 내 nodejs 버전을 특정 버전으로 lock 시킬 수 있다.
애플리케이션은 

개발자간 동일한 개발 환경을 보장한다.

프로젝트마다 독립적인 컨테이너를 갖게하여. 프로젝트마다 동일한 개발 환경을 보장한다.
글로벌하게 버전을 관리하지 않아도 된다.

### virtual machine과의 비교

virtual machine은 computer 안의 computer, machine 안의 machine
virtual machine은 virtual machine 안에서 돌아가는 virtual os를 가진다.
virtually 하게 존재하지만 어쨋든 여러가지 툴 설치가 가능하다. 
![[Pasted image 20260205223237.png]]
