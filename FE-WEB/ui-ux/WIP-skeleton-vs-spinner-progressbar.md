스피너와 스켈레톤, 프로그래스바 각각을 쓰는 기준은 다음과 같다.
- 로딩 시간이 짧을 것으로 예상된다면 로딩 스피너를 사용한다.
- 로딩 시간이 길 것으로 예상된다면 로딩 스켈레톤을 사용한다.
- 버튼 액션을 기다릴때는 스피너를 사용한다.
- 레이아웃을 미리 알 수 있는 경우 스켈레톤을 사용한다.
- 여러 요소가 동시에 로딩될 때, 예를들면 소셜미디어 피드 페이지처럼 사용자 프로필 이미지, 게시글 텍스트, 댓글 목록, 좋아요 수, 광고 배너 등이 동시에 로드될 때 각각을 로딩 스피너를 보여주는 것 보단 스켈레톤을 보여주는 것이 낫다.
- 업로드, 다운로드 등과 관련한 진행 상태를 나타낼때는 progressbar가 적당하다.

솔직히 시간은 공감이 안간다.





로딩 스켈레톤의 장점은 다음과 같다.
1. 컨텐츠가 어떻게 보여질지 미리 인지 가능하다.
2. 로딩에서 컨텐츠로의 부드러운 전환이 가능하다.

로딩 스켈레톤의 단점은 다음과 같다.
1. 디자인과 개발에 시간이 더 들어간다.
2. 로딩 스켈레톤의 복잡도는 UX에 좋은 경험을 미치지 못한다.


[☠️Loading spinners and loading skeletons | Productboard engineering](https://medium.com/productboard-engineering/%EF%B8%8F-spinners-versus-skeletons-in-the-battle-of-hasting-b51b9c6574ef)

[Skeleton loading screen design — How to improve perceived performance - LogRocket Blog](https://blog.logrocket.com/ux-design/skeleton-loading-screen-design/)

