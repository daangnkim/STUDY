##### fillna
- 결측값(NaN)을 다른 값으로 채운다.

##### value_counts
- 인덱스가 0, 1이 아닌 실제 값들을 사용한다. 그러므로 df.index[0]을 사용하면 첫 번째 값을 얻는다.
- 컬럼의 . 각값들이 몇 번씩 나타내는지 개수를 세는 함수
- 결과는 Series
- 빈도 순으로 정렬

##### reset_index(drop=True)
- 인덱스 초기화

##### iloc
- iloc[:10]
	- 10개의 행, 10개의 열을 가져온다.
- iloc[:10, -1]
	- :10 처음 10개 행
	- -1 마지막 컬럼
- iloc[-1]
	- 마지막 행
- iloc은 인덱싱 방식이라 그런지 [:10]에서 끝인 10은 포함하지 않음. 0부터 9까지임.
- loc[:9]는 끝이 포함됨.

##### nlargest(10)
- 상위 10개의 값을 내림차순으로 선택
- .iloc[-1]하면 마지막 행 선택

##### idxmax()
- 가장 큰 값의 인덱스 찾기

##### mode()
- 최빈값 찾기

##### df.isnull()
- 결측치가 있는 위치는 True, 없는 위치는 False로 만든다.

##### dropna(subset=[])
- 특정 컬럼에서 결측값이 있는 행을 삭제한다.