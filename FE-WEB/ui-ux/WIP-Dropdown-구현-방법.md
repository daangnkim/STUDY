다음과 같은 UI를 구현해야 한다고 가정해보자.

1. 사이드바 내 리스트에 마우스를 호버하면 더보기 메뉴가 나온다.
2. 더보기 메뉴에 마우스를 호버하면 드랍다운이 나온다.

구현 방법은 크게 세 가지가 존재한다.

1. summary, detail
2. popupAPI
3. 상태를 통한 제어
4. 의사 클래스 (pseudo class) 이용

## [1] summary, detail을 이용한 방식

native html element를 이용해서 구현하는 방법이다.

```html
<details className="dropdown">
  <summary className="btn m-1">open or close</summary>
  <ul
    className="menu dropdown-content bg-base-100 rounded-box z-1 w-52 p-2 shadow-sm"
  >
    <li>
      <a>Item 1</a>
    </li>
    <li>
      <a>Item 2</a>
    </li>
  </ul>
</details>
```

이 방법은 사이드바 내 리스트의 더보기 메뉴가 한 번에 하나씩만 열려야 하는데, 그럴려면 `name` 어트리뷰트를 사용해야한다. 하지만 `name` 어트리뷰트는 비교적 최신 브라우저([Chrome 기준 2023년 12월부터](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/details#browser_compatibility))에서만 동작을 한다.

## [2] popupAPI

```jsx
<!-- change popover-1 and --anchor-1 names. Use unique names for each dropdown -->
<!-- For TSX uncomment the commented types below -->
<button className="btn" popoverTarget="popover-1" style={{ anchorName: "--anchor-1" } /* as React.CSSProperties */}>
  Button
</button>

<ul className="dropdown menu w-52 rounded-box bg-base-100 shadow-sm"
  popover="auto" id="popover-1" style={{ positionAnchor: "--anchor-1" } /* as React.CSSProperties */ }>
  <li><a>Item 1</a></li>
  <li><a>Item 2</a></li>
</ul>
```

이 API 역시도 최신 브라우저([Chrome 기준 2023년 5월](https://developer.mozilla.org/en-US/docs/Web/API/Popover_API))에서만 동작을 한다.

## [3] 상태를 통한 제어

단순히 js를 이용하는 방법이다.

## [4] 의사 클래스 (pseudo class) 이용

요소의 특정 상태를 나타내는 선택자들은 콜론(:)을 이용하여 총칭하며 `의사 클래스`라고 부름

```html
<li className="group">
  <a>Thread Title</a>
  <div>
    <!-- group이 focus되거나 hover되면 메뉴 버튼이 나타난다. -->
    <div className="peer hidden group-hover:block group-focus:block">
      메뉴 버튼
    </div>

    <!-- 메뉴 버튼이 focus되면 드롭 다운이 나타난다. -->
    <ul className="hidden peer-focus:block">
      <li>제목 편집</li>
      <li>스레드 삭제</li>
    </ul>
  </div>
</li>
```
