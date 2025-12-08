## react-native-gesture-handler을 사용하는 이유
- 복잡한 제스쳐 구현, native level에서 구현되므로 더 스무스하게 동작하고 응답에 대한 딜레이나 제스쳐 충돌이 없음.
	- https://www.linkedin.com/pulse/what-react-native-gesture-handler-why-its-essential-mobile-mehta-vrcjf

## animation 서능 최적화
- useNativeDrive 쓰기 (단 layout style 변경하지 않는 선에서 동작함)
	- useNativeDrive가 없다면 대부분의 작업이 js thread에서 진행되며, 진행되지 못한다면 프레임을 건너뛰고, View를 업데이트하기 위해서 매번 Bridge를 통과해야함
	- NativeDriver는 이러한 모든 작업을 native 쪽으로 넘김. 그러니까 js thread나 bridge가 관여하지 않아도 됨
- https://freedium.cfd/https://medium.com/@mohantaankit2002/solving-performance-issues-in-react-native-animations-f09a9ed076e2

## worklet
- [Worklets and Threading in Reanimated for Smooth Animations in React Native - DEV Community](https://dev.to/ajmal_hasan/worklets-and-threading-in-reanimated-for-smooth-animations-in-react-native-98)
- [Getting started | React Native Worklets: Multithreading engine for your apps and libraries](https://docs.swmansion.com/react-native-worklets/docs/)

## panResponder
- [React Native PanResponder. PanResponder is a built-in gesture… | by Diliplohar | Medium](https://medium.com/@diliplohar204/react-native-panresponder-523b9ad2b9ca)