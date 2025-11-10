# responsive web을 조금 더 쉽게 만드는 방법
- 고정 단위 사용하기
- main content에 대한 maxWidth 제한하기
- emulator를 이용하여 테스트하기

# pixel

용어 정리를 다음과 같이 할 수 있다.
1. device pixel ratio === css pixel ratio === device density
2. device resolution === screen resolution === physical resolution
3. css resolution === logical resolution
4. device pixel === physical pixel
5. css pixel === logical pixel

2x 해상도에서 css로 200px을 적용하면 실제로는 400개의 물리적 픽셀이 사용됨
과거에는 device pixel과 css pixel이 같았음.

pixel density는 크게 ppi, dpi로 나뉜다.

device pixel ratio는 device resolution이 css가 어떻게 해석할지를 결정한다.

- [Understanding the Difference Between CSS Resolution and Device Resolution | by Elad Shechter | Medium](https://elad.medium.com/understanding-the-difference-between-css-resolution-and-device-resolution-28acae23da0b)
- [Screen sizes in DevTools do not match actual device sizes - Stack Overflow](https://stackoverflow.com/questions/69626721/screen-sizes-in-devtools-do-not-match-actual-device-sizes)
- [css - what exactly is device pixel ratio? - Stack Overflow](https://stackoverflow.com/questions/8785643/what-exactly-is-device-pixel-ratio)

# physical device vs chrome devtools
chrome devtools는 physical device에 대한 근사된 view만 볼 수 있음.

- https://www.reddit.com/r/webdev/comments/rgnqmj/why_is_the_responsive_design_dev_tools_on_mobile/