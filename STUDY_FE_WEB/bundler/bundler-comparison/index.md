![[Pasted image 20250523225417.png]]

크게 비교해볼만한 번들러는 rollup, tsup, vite, webpack, esbuild정도인 것 같다.

webpack은 small app에 대해서 트리 쉐이킹이 잘 안되는 이슈가 존재한다.
`
esbuild는 타입스크립트 정의 파일이 자동으로 만들어지지 않는다. 별도로 만들어주어야한다.

CLI 길이는 esbuild > tsup > rollup = vite = webpack 순으로 길다.

Config 파일 필요 여부는 rollup, vite, webpack이 필요하다.

번들 사이즈는 라이브러리 사이즈가 작은 경우 esbuild, rollup, vite, webpack이 비슷하고, tsup이 살짝 많다. 라이브러리 사이즈가 큰 경우 tsup = esbuild = rollup > vite = wepack 으로 사이즈가 작다.

번들링 속도는 라이브러리 사이즈가 작은 경우 webpack > rollup = tsup, vite, esbuild 순으로 느리고, 라이브러리 사이즈가 큰 경우 webpack > vite = rollup = tsup = esbuild 순으로 느리다.

esbuild, tsup은 css experimental이다.


[2024 JavaScript bundlers comparison | Tony Cabaye](https://tonai.github.io/blog/posts/bundlers-comparison/)

[esbuild vs parcel vs rollup vs tsup vs vite vs webpack | npm trends](https://npmtrends.com/esbuild-vs-parcel-vs-rollup-vs-tsup-vs-vite-vs-webpack)