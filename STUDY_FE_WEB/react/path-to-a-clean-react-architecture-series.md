## UI와 API를 분리한다.

UI와 API를 분리한다. UI는 API의 구체적인 구현 사항에 대해서 몰라도 되기 때문이다. UI 관점에서는 axios를 쓰는지 fetch를 쓰는지, get을 하는지 post를 하는지, response, request 타입은 무엇인지, endpoint는 무엇인지 알 필요가 없다.

```typescript
// 1
apiClient.get<UserResponse>(`/user/${handle}`)

// 2
UserApi.getUser(handle)
```

1을 2로 리펙토링하면 좋은 점은, UI와 API 요청이 분리된다는 점이다. 

## Data Transformation은 API layer에서 처리한다.

```typescript
import { useEffect, useState } from "react";
import { Navigate, useParams } from "react-router";

import UserApi from "@/api/user";
import { LoadingSpinner } from "@/components/loading";
import { ShoutList } from "@/components/shout-list";
import { UserResponse, UserShoutsResponse } from "@/types";

import { UserInfo } from "./user-info";

export function UserProfile() {
  const { handle } = useParams<{ handle: string }>();

  const [user, setUser] = useState<UserResponse>(); // ❗️
  const [userShouts, setUserShouts] = useState<UserShoutsResponse>(); // ❗️
  const [hasError, setHasError] = useState(false);

  useEffect(() => {
    if (!handle) {
      return;
    }

    UserApi.getUser(handle)
      .then((response) => setUser(response)) // ❗️
      .catch(() => setHasError(true));

    UserApi.getUserShouts(handle)
      .then((response) => setUserShouts(response)) // ❗️
      .catch(() => setHasError(true));
  }, [handle]);

  if (!handle) {
    return <Navigate to="/" />;
  }

  if (hasError) {
    return <div>An error occurred</div>;
  }
  if (!user || !userShouts) {
    return <LoadingSpinner />;
  }

  return (
    <div className="max-w-2xl w-full mx-auto flex flex-col p-6 gap-6">
      <UserInfo user={user.data} />
      <ShoutList
        users={[user.data]} // ❗️
        shouts={userShouts.data} // ❗️
        images={userShouts.included} // ❗️
      />
    </div>
  );
}
```

위 코드에서 ❗️가 남겨진 주석들은 다음 문제점들이 존재한다.

1. UI는 User 객체가 response.data로 부터 나온다는 것을 알고 있어야한다.
2. 마찬가지로 shouts가 response.data에 있거나, images가 response.included로부터 나온다는 것을 알고 있어야한다.
3. 만약 백엔드에서 response.data를 response.payload로 혹은 response.included를 response.attchments로 이름을 바꾸면 호출하는 모든 곳을 찾아가 수정해야한다.

즉, UI와 API 로직이 강하게 결합되어 유지보수가 힘들어진다. 그러므로 data transformation 로직을 API layer에 넣어서 다음과 같이 코드를 작성하여 해결한다.

```typescript
export function UserProfile() {
  const { handle } = useParams<{ handle: string }>();

  const [user, setUser] = useState<User>();
  const [shouts, setShouts] = useState<Shout[]>();
  const [images, setImages] = useState<Image[]>([]);
  const [hasError, setHasError] = useState(false);

  useEffect(() => {
    if (!handle) {
      return;
    }

    UserApi.getUser(handle)
      .then((user) => setUser(user))
      .catch(() => setHasError(true));

    UserApi.getUserShouts(handle)
      .then(({ shouts, images }) => {
        setShouts(shouts);
        setImages(images);
      })
      .catch(() => setHasError(true));
  }, [handle]);

  ...

  return (
    <div className="max-w-2xl w-full mx-auto flex flex-col p-6 gap-6">
      <UserInfo user={user} />
      <ShoutList users={[user]} shouts={shouts} images={images} />
    </div>
  );
}
```

두번째 예는 다음과 같다.

```typescript
import FeedApi from "@/api/feed";
import { LoadingView } from "@/components/loading";
import { ShoutList } from "@/components/shout-list";
import { FeedResponse, Image, User } from "@/types";

export function Feed() {
  const [feed, setFeed] = useState<FeedResponse>(); // ❗️
  const [hasError, setHasError] = useState(false);

  useEffect(() => {
    FeedApi.getFeed()
      .then((feed) => setFeed(feed))
      .catch(() => setHasError(true));
  }, []);
  if (hasError) {
    return <div>An error occurred</div>;
  }
  if (!feed) {
    return <LoadingView />;
  }

  const users = feed.included.filter((u): u is User => u.type === "user"); // ❗️
  const images = feed.included.filter((i): i is Image => i.type === "image"); // ❗️
  return (
    <div className="w-full max-w-2xl mx-auto flex flex-col justify-center p-6 gap-6">
      <ShoutList shouts={feed.data} users={users} images={images} />
    </div>
  );
}
```

마찬가지로 data transformation 로직을 API layer에 넣는다.

```typescript
import { FeedResponse, Image, User } from "@/types";

import { apiClient } from "./client";

async function getFeed() {
  const response = await apiClient.get<FeedResponse>("/feed");
  const shouts = response.data.data;
  const users = response.data.included.filter(
    (u): u is User => u.type === "user"
  );
  const images = response.data.included.filter(
    (i): i is Image => i.type === "image"
  );
  return { shouts, users, images };
}

export default { getFeed };
```

세번째 예는 다음과 같다.

```typescript
import MediaApi from "@/api/media";
import ShoutApi from "@/api/shout";
import UserApi from "@/api/user";

...

export function ReplyDialog({ children, shoutId }: ReplyDialogProps) {
  const [open, setOpen] = useState(false);
  const [isLoading, setIsLoading] = useState(true);
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [hasError, setHasError] = useState(false);

  ...
  
  async function handleSubmit(event: React.FormEvent<ReplyForm>) {
    event.preventDefault();
    setIsLoading(true);
    try {
      const message = event.currentTarget.elements.message.value;
      const files = event.currentTarget.elements.image.files;
      let imageId = undefined;
      if (files?.length) {
        const formData = new FormData();
        formData.append("image", files[0]);
        const image = await MediaApi.uploadImage(formData);
        imageId = image.data.id;
      }
      const newShout = await ShoutApi.createShout({
        message,
        imageId,
      });
      await ShoutApi.createReply({
        shoutId,
        replyId: newShout.data.id,
      });
      setOpen(false);
    } catch (error) {
      console.error(error);
    } finally {
      setIsLoading(false);
    }
  }
  
  // a form using the handleSubmit function is rendered here
  return (
    ...
  );
}

```

MediaApi는 FormData object를 전달받아야 한다는 것을 UI가 알 필요가 없다.  그러므로 다음과 같이 수정한다.

```typescript
export function ReplyDialog({ children, shoutId }: ReplyDialogProps) {
  const [open, setOpen] = useState(false);
  const [isLoading, setIsLoading] = useState(true);
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [hasError, setHasError] = useState(false);

  ...
  
  async function handleSubmit(event: React.FormEvent<ReplyForm>) {
    event.preventDefault();
    setIsLoading(true);
    try {
      const message = event.currentTarget.elements.message.value;
      const files = event.currentTarget.elements.image.files;
      let image;
      if (files?.length) {
        image = await MediaApi.uploadImage(files[0]);
      }
      const newShout = await ShoutApi.createShout({
        message,
        imageId,
      });
      await ShoutApi.createReply({
        shoutId,
        replyId: newShout.data.id,
      });
      setOpen(false);
    } catch (error) {
      console.error(error);
    } finally {
      setIsLoading(false);
    }
  }
  
  // a form using the handleSubmit function is rendered here
  return (
    ...
  );
}
```

## UI와 API를 분리하는 이유

1. 백엔드 구조가 바뀌더라도 API 레이어만 고치면 된다. UI를 건들지 않아도 된다.
2. .data.data, .included와 같은 래퍼 필드를 보지 않아도 된다.


## References

1. [Path To A Clean(er) React Architecture - A Shared API Client](https://profy.dev/article/react-architecture-api-client)
2. [Path To A Clean(er) React Architecture - API Layer & Fetch Functions](https://profy.dev/article/react-architecture-api-layer-and-fetch-functions)
3. [Path To A Clean(er) React Architecture - API Layer & Data Transformations](https://profy.dev/article/react-architecture-api-layer-and-data-transformations)

