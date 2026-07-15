![[Pasted image 20260715085856.png]]

- 기존에는 JIT를 이용하는 Dalvik Virtual Machine 이용. 
	- Lollpop 부터 ART + AOT 이용. 이때부터 `.dex` (Dalvik Executable) 바이트 코드는 `dex2oat` (native machine code) 로 컨버팅됨.
	- ART는 성능 향상, 메모리 최적화 등 여러 이점 존재.

### 안드로이드 설치 과정

apk 파일에 다음 네 가지 존재
	- .dex 파일
	- 이미지, 레이아웃, 번역 등의 리소스
	- manifest

package manager는 app의 signature를 확인함.
ART 내의 dex2oat 툴이 앱의 바이트 코드를 네이티브 코드로 변경.
Zygote process 가 새로운 프로세스를 만듬

이후 과정은 모르겠음.

### 앱이 내부에서 돌아가는 원리

Launcher 가 `Intent`를 Activity Manager에게 보냄
Activity Manager가 실행중인 프로세스를 가지고 있는지 하ㅗㄱ인하고, 망냑 없다면 Zygote로부터 프로세스를 포크뜸
메인스레드가 실행되고, Activity나 Service의 라이프사이클 실행
시스템 서비스 (위치, 알람 등)는 Binder IPC 메카니즘을 통해서 접근되며, 앱과 시스템 컴포넌트들간의 통신이 안전하게 이뤄지도록함.

### 성능 최적화

- AOT + JIT 하이브리드 : AOT는 설치시에 발생. ART는 자주 변경되는 코드에 대해서는 여전히 실시간으로 JIT 컴파일 수행
- Profiel-guided optimizaiton : ART는 당신이 앱을 어떻게 사용하는지 추적하고, 핸드폰이 사용되는 동안 자주 사용되는 코드들을 최적화함
- Garbage Collection : ART는 사용되지 않는 객체들의 메모리를 회수함


### App components

안드로이드 앱을 구성하는 네가지 요소로 구성된다.

- 액티비티
- 서비스
- 브로드 케스트 리시버
- 콘텐츠 프로바이더

각각은 목적과 라이프사이클이 정해져있다. 
컴포넌트는 독립되어 코드 결합이 발생하지 않는다. 
리스트 액티비티가 챗 액티비티를 직접 실행하지 않고, 안드로이드 시스템을 거친 후 실행된다.
더불어서 앱의 실행 시점은 다양하기 때문에 단일 진입점인 메인 함수 개념이 없다.
앱 내에서 다른 앱을 라이브러리로 호출한다. 

### 클래스

클래스는 크게 일반 클래스, 컴포넌트 클래스 두 가지로 나뉨

### 리소스

정적인 데이터들을 관리하고 재사용한다. 


### References

- [Android 액티비티(Activity)란?](https://www.tosspayments.com/blog/articles/mobile-pay-2)
- [Deep Dive: How Android Works Under the Hood | by Aswinsrinivasan | Medium](https://medium.com/@aswinsrinivasan2004/deep-dive-how-android-works-under-the-hood-fd1184382db2)
- [Application fundamentals  |  App architecture  |  Android Developers](https://developer.android.com/guide/components/fundamentals#Components)