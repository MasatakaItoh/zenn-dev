---
title: "Next.js App Router キャッシュの今"
emoji: "🐕"
type: "tech"
topics: ["Nextjs","React","ReactCache","キャッシュ"]
published: true
publication_name: "frontendflat"
---

先日Vercelから「Next.js App Router Caching: Explained!」というタイトルの動画が公開されていたので、その内容をまとめることでNext.jsのキャッシュの今について整理しておこうと思います。

https://youtu.be/VBlSe8tvg4U?si=HYmOwX9wuxz6S8Oj

## 基本

まずNext.jsでは、静的レンダリングがデフォルトです。RSCを使用していても基本的にはビルド時にページが事前レンダリングされます。これはRoute Handlersも同様です。仮にビルド後にデータを更新してもリビルドしない限り表示は古いままであり、これは静的にレンダリングされていると言えます。
ただし、developmentとproductionでは挙動が異なります。ローカルではコードに変更を加えるたびにデータが再取得・レンダリングされるので、ローカルとビルド後の挙動に違いがあることを理解しておきましょう。

リクエストするたびに最新のデータを取得し表示したい、つまりページやRoute Handlersを動的にレンダリングしたければ、`cookies()`や`headers()`、`noStore()`などの動的レンダリングを有効にする方法を採用する必要があります。

https://nextjs.org/docs/app/building-your-application/rendering/server-components#switching-to-dynamic-rendering

## 静的なfetch

サーバーサイドfetchもそのまま使うとキャッシュされるので、ビルド時に一度だけデータが取得されます。

ローカルではリロードするとキャッシュからデータが返され、スーパーリロードするとキャッシュが破棄されます。
Next.jsではその挙動がわかるようなconfigを提供してくれています。以下を設定するとデータ取得時にキャッシュがヒットしたかどうかがターミナルに出力されます。つまり、リロードすると`cache: HIT`、スーパーリロードすると`cache: SKIP`と表示されるということです。これはfetchのみ対応しているので、fetchを使用している場合は使ってみるとキャッシュの挙動がわかりやすくなると思います。

```js:next.config.js
module.exports = {
  logging: {
    fetches: {
      fullUrl: true,
    },
  },
}
```

https://nextjs.org/docs/app/api-reference/next-config-js/logging

## 動的なfetch

これは「基本」に書いた通りで、`cookies()`や`headers()`、`noStore()`など動的レンダリングを有効にする方法を採用することで、リクエストのたびに最新のデータを取得するサーバーサイドfetchを実装することができます。

## unstable_cache

### 静的なDBリクエスト

Next.jsアプリケーションから直接DBにリクエストする、つまりAPIを介さないリクエストにおいては、fetchを使用することができません。

その場合、Next.jsが提供するunstable_cacheを使用することで、キャッシュやその再検証を実装することになります。第三引数にはオプションを指定可能で、`tags`や`revalidate`を指定することで、サーバーサイドfetchで実現できた任意のタイミングでの再検証や時間による再検証（ISR）も実現できます。

```ts
const getCachedUser = unstable_cache(
  async (id) => getUser(id),
  [`user-${id}`],
  {
    tags: [`user-${id}`],
  }
);
```

https://nextjs.org/docs/app/api-reference/functions/unstable_cache

unstable_cacheはNext.js v14から追加された最新の機能なので、今後APIの構造が多少調整される可能性は高いとのことです。
Next.jsはApp Router導入以降、fetchを拡張する方針をとってきましたが、ユーザーやコミュニティのフィードバックを受けてその方針からの脱却を考えているとのことなので、今後unstable_cacheの活用場面は増えていくと思われます。

### 動的なDBリクエスト

これは単にunstable_cacheを使わないDBリクエストをすれば良いです。

```ts
const getUser = async (id) => getUser(id);
```

### Server Actionsによる再検証

Server Actionsでデータを更新したあとに`revalidateTag()`または`revalidatePath()`を実行することで、fetchと同様にunstable_cacheのキャッシュを破棄することができます。

```ts
'use server'
 
import { revalidateTag } from 'next/cache'
 
export async function createPost() {
  try {
    // ...
  } catch (error) {
    // ...
  }
 
  revalidateTag('posts');
}
```

https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations#revalidating-data

### Webhookによる再検証

CMSなど外部システムの何かしらの操作（例えば記事更新など）によりデータに変更が加えられた場合は、Route HandlersによりWebhookを実装するのが良いでしょう。

```ts
export async function POST() {
  // ...
  
  revalidateTag('posts');
  
  return new Response('Success!');
}
```

### ISRによる再検証

上述の通り、unstable_cacheには第三引数のオプションで`revalidate`を指定することができます。以下の例では60秒ごとに再検証が行われるISRとなります。

```ts
const getCachedUser = unstable_cache(
  async (id) => getUser(id),
  [`user-${id}`],
  {
    revalidate: 60,
  }
);
```

## まとめ

unstable_cacheの登場によりデータ取得とキャッシュにおいても直接DBにリクエストするユースケースがサポートされることになります。これによりApp Routerを採用できるプロジェクトも増えるのではないかと思います。

unstable_cache含め、App Routerの今後に期待しましょう。
