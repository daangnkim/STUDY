loader 내에서 렌더링에 급하게 필요한 데이터와 급하게 필요하지 않은 데이터를 구분지을 수 있다.

##### `critical data`
화면에 바로 보여주어야 하는 부분으로, loader에서 awaited된 data를 반환한다.

##### `non critical data`
화면에 바로 보여주지 않아도 되는 부분으로, loader에서 unresolved된 data를 반환한다. await하고 반환하지 않기 때문에 렌더링 먼저 가능하다.

예를들면, 상품 상세 페이지에서 상품에 대한 설명은 `critical data`, 상품에 대한 리뷰는 `non critical data`가 된다. 

```tsx
import {
  defer,
  useLoaderData,
  Await,
} from "react-router-dom";
import React from "react";
import { getProduct, getProductReviews } from "./api";

export async function loader({ params }) {
  // Await critical data immediately:
  const product = await getProduct(params.productId);
  // Return non-critical data as a promise (deferred):
  const reviewsPromise = getProductReviews(params.productId);

  return defer({ product, reviews: reviewsPromise });
}

export default function ProductPage() {
  const { product, reviews } = useLoaderData();

  return (
    <main>
      <h1>{product.name}</h1>
      <p>{product.description}</p>

      <React.Suspense fallback={<p>Loading reviews…</p>}>
        <Await resolve={reviews}>
          {(reviewsData) => (
            <ul>
              {reviewsData.map((review) => (
                <li key={review.id}>{review.comment}</li>
              ))}
            </ul>
          )}
        </Await>
      </React.Suspense>
    </main>
  );
}
```

## 언제 사용하면 좋을까?

##### 1. 느린 API 요청
리뷰 정보, 공급자 정보 등 중요하지 않으면서도 느린 API 요청에서 사용한다.
##### 2.엄청나게 큰 데이터를 보여주어야 할때
##### 3. 데이터가 로드되는동안 fallback ui를 보여주고 싶을 때



[Speed Up Data Fetching With React Router Defer](https://www.dhiwise.com/blog/design-converter/how-to-improve-data-loading-with-react-router-defer)