---
title: "generateStaticParams() 에서 supabase client 를 호출하면"
categories:
  - Bookshelf Project
  - Next.js
  - Supabase
tags:
  - generateStaticParams
  - Supabase createServerClient()
toc: true
toc_sticky: true
share: true
published: false
---

# 문제 상황

dynamic route 에 generateStaticParams() 를 사용해서 빌드타임에 정적으로 라우트를 생성하려는데

1. posts 리스트를 fetch 하기 위해서 supabase createClient 호출 → createClient 는 내부적으로 cookies() 호출

   - 🚫 request scope 이외에 cookies 를 사용하면 Next.js 에러
     ([Next.js 에러](https://nextjs.org/docs/messages/next-dynamic-api-wrong-context))

2. Route Handler 로 supabase 쿼리를 조회하는데 상대 경로를 인식하지 못함  
   `route handler failed to parse url`

3. Route Handler 에서 상대 경로를 사용했더니 런타임에는 문제 없는데, 빌드 실패함

   ```
    Collecting page data  ..TypeError: fetch failed
       at async Object.s [as generateStaticParams] (.next/server/app/(with-layout)/post/[id]/page.js:1:5448) {
     [cause]: [AggregateError: ] { code: 'ECONNREFUSED' }
   }

   > Build error occurred
   [Error: Failed to collect page data for /post/[id]] { type: 'Error' }
   ```

# 해결

- 권장되는 방식은 아닌거 같지만 @supabase/supabase-js 패키지에서 쿠키 설정 없이 createClient 메소드를 호출해서 임시 클라이언트 생성. 해당 클라이언트로 supabase 쿼리 조회

---

## Troubleshooting

### 1. generateStaticParams() 내부에서 createServerClient() 호출하지 않도록

`cookies()` 는 request scope 내에서만 동작하도록 설계되었다.

즉, `coookies()` 는 요청(request) 객체가 있어야 작동하는 함수다. 요청 객체 안에 담겨있는 쿠키를 읽어서 동작하기 때문이다.

하지만, `generateStaticParams()` 는 빌드 타임에 실행하는 함수로서, 어떠한 HTTP 요청에 의해서 실행되는 것이 아니라, 빌드 과정에서 자동으로 실행된다. HTTP 요청이 없었기 때문에, 요청 객체도 생겨나지 않고, 이에 따라 `cookies()` 도 데이터를 가지지 못한다.

따라서, `generateStaticParams()` 내부에서 `cookies()` 호출하지 않도록 해야 함. → 내부적으로 쿠키 설정이 필요한 `createServerClient()` 를 사용할 수 없어서 Route Handler 로 fetch 하는 방법을 생각함

### 2. Route Handler 를 호출할때 절대 경로를 사용하도록

```tsx
// app/post/[id]/page.tsx

export async function generateStaticParams() {
  const res = await fetch(`${process.env.NEXT_PUBLIC_BASE_URL}/api/posts`);
  const data: BookDetails[] = await res.json();
  return data.map((book) => ({
    id: String(book.id),
  }));
}
```

**_⛔ 하지만 이건 Anti Pattern 임!! 🔼_**

- dev mode 에는 서버가 http://localhost:3000 에서 실행되므로 위 코드를 실행하는 데에 무리가 없다. 하지만, 빌드 타임에는 실행중인 서버가 없기 때문에! (빌드 타임에는 HTML, JS 등을 생성하는 작업만 할 뿐, 아직 서버가 켜진 것은 아님) fetch 는 실패할 수 밖에 없다.

### 3. 서버 컴포넌트 내부에서 Route Handelr 로 데이터를 fetch 하는 것은 BAD PRACTICE 이다.

- [참고 1](https://github.com/vercel/next.js/discussions/63183)
- [참고 2](https://nextjs-faq.com/fetch-api-in-rsc)

```tsx
export async function generateStaticParams() {
  const res = await fetch(`${process.env.NEXT_PUBLIC_BASE_URL}/api/posts`);
  const data: BookDetails[] = await res.json();
  return data.map((book) => ({
    id: String(book.id),
  }));
}
```

- `Route Handler` 는 클라이언트 사이드 fetch 할때 사용해야 한다.
  - 클라이어트 사이드에서는 HTTP 요청을 하는 것이 자연스럽다. → HTTP 요청을 할때 route handler, 혹은 server action 을 호출하면 됨
- 서버 환경에서 fetch 하는 것은 같은 서버 환경에서 불필요한 HTTP 요청을 만드는 것이다. (A 서버에서 실행중인 코드가 A 서버에게 HTTP 요청을 보내서 응답을 받아오는 구조)
  - 서버 환경에서는 fetch 로 불필요한 HTTP 를 또 요청하지 말고, 서버 코드를 직접 호출하자.

따라서, `cookies()` 를 호출하지 않는 supabase client를 생성함

```tsx
import { Database } from "@/database.types";
import { createClient } from "@supabase/supabase-js";

type BookDetails = Database["public"]["Tables"]["book_details"]["Row"];

export async function generateStaticParams() {
  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );

  const { data } = await supabase
    .from("book_details")
    .select("*")
    .overrideTypes<BookDetails[], { merge: false }>();

  return data?.map((d) => ({
    id: String(d.id),
  }));
}

export default async function Page({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  return <div>This is {id}</div>;
}
```

- 결과: 런타임 OK, 빌드 실패

```
app/(with-layout)/post/[id]/page.tsx
Type error: Page "app/(with-layout)/post/[id]/page.tsx" has an invalid export:
  "Promise<{ id: string; }[] | undefined>" is not a valid generateStaticParams return type:
    Expected "any[] | Promise<any[]>", got "Promise<{ id: string; }[] | undefined>".
      Expected "Promise<any[]>", got "Promise<{ id: string; }[] | undefined>".
        Expected "any[]", got "{ id: string; }[] | undefined".
          Expected "any[]", got "undefined".
```

- 빌드가 실패한 이유:
  - `generateStaticParams` 함수가 **`undefined`를 반환할 수도 있어서 빌드를 실패했다**.
    `generateStaticParams()`는 반드시 빈배열이라도 반환해야 하므로, 따라서 `undefined`를 반환할 가능성은 허용되지 않는다
    ‼️ 이는 빌드 타임에 정적으로 **모든 경로를 생성**하기 위함

> You must return [an empty array from `generateStaticParams`](https://nextjs.org/docs/app/api-reference/functions/generate-static-params#all-paths-at-build-time) or utilize

에러 발생시 빈배열을 리턴하도록 하니 빌드 성공함!

## 새로 알게된 점

- 서버 환경에서 Route Handler 로 내 end point 를 호출하는 것은 권장되지 않는 패턴이다!
- Route Handerl 는 클라이언트 컴포넌트에서 사용하거나, 다른 서버에서 내 서버를 호출하는 end point 로 사용
