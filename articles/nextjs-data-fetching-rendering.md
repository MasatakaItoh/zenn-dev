---
title: "App RouterにおけるData FetchingとRenderingのバリエーション整理"
emoji: "💬"
type: "tech"
topics: ["Nextjs", "React", "AppRouter"]
published: true
publication_name: "frontendflat"
---

Next.jsでは、App RouterでServer Component（SC）とClient Component（CC）が区別されるようになってから、Data FetchingとRenderingのバリデーションが増えより複雑になりました。

今回はmicroCMSテンプレートを元に、各Data FetchingとRenderingの挙動を整理します。編集する対象ファイルは`app/articles/[slug]/page.tsx`とします。

https://github.com/microcmsio/nextjs-simple-blog-template/blob/main/app/articles/%5Bslug%5D/page.tsx

このテンプレートはmicroCMSにログインすることで誰でも使用することができるので、よかったら実際に手元で動作確認をしてみてください。

https://templates.microcms.io/templates/detail/a530e59f-d66d-4b85-9ef5-4cf4288adb09

## SCでSG

対象のファイルから`revalidate`の記述を削除することで、SCでSGのような挙動を取ります。

```jsx:app/articles/[slug]/page.tsx
export default async function Page({ params, searchParams }: Props) {
  const data = await getDetail(params.slug, {
    draftKey: searchParams.dk,
  });

  return <Article data={data} />;
}
```

App RouterではStatic Renderingがデフォルトなので、fetchはビルド時に一度しか実行されません。
例えば、一度ビルドしたあとにmicroCMSで記事を更新しても、表示は古いままとなります。これはビルド時に取得したデータが返され続けていて、リクエストごとにデータが取得されていないことを意味します。

https://nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default

https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating#fetching-data-on-the-server-with-fetch

リクエストごとにデータを取得する（＝SSRする）ためには、Dynamic Renderingを有効にする必要があります。

## SCでSSR

[Dynamic Functions](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-functions)または[Uncached Data Request](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating#opting-out-of-data-caching)を使用すると、Dynamic Renderingが有効になります。今回は[Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)の指定でUncached Data Requestを実行し、Dynamic Renderingを有効にします。

```jsx:app/articles/[slug]/page.tsx
export const fetchCache = 'default-no-store';

export default async function Page({ params, searchParams }: Props) {
  const data = await getDetail(params.slug, {
    draftKey: searchParams.dk,
  });

  return <Article data={data} />;
}
```

これでSSRの挙動となり、リクエストごとにデータを取得するようになります。一度ビルドしたあとにmicroCMSで記事を更新しても、常に最新のデータが表示されます。

ちなみに今回はRoute Segment Configを使用しましたが、microcms-js-sdkでもfetchリクエストオプションが追加できるようになっています。実際には、キャッシュの制御はセグメント単位ではなくリクエスト単位で行いたいと思うので、fetchオプションとして`cache: 'no-store'`を指定するのがおすすめです。

https://blog.microcms.io/microcms-js-sdk-2_5_0/

## SCでISR

Route Segment Configの[revalidate](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#revalidate)を使用すると、ISRを実現できます。

```jsx:app/articles/[slug]/page.tsx
export const revalidate = 60;

export default async function Page({ params, searchParams }: Props) {
  const data = await getDetail(params.slug, {
    draftKey: searchParams.dk,
  });

  return <Article data={data} />;
}
```

これで60秒のISRとなります。 
SSRと同様に、fetchオプションとしてrevalidateを指定し、リクエストごとにキャッシュを制御することもできます。

```jsx
fetch('https://...', { next: { revalidate: 60 } })
```

https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating#time-based-revalidation

## SCでOn-demand ISR

[revalidatePath](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)または[revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)を使用することで、On-demand ISRを実現できます。

https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating#on-demand-revalidation

説明が長くなるので参考記事を添付します。

https://zenn.dev/ynot/articles/dc27182e5cc263

## CCでSSR

対象のファイルをSCからCCに変更するために、以下のように編集します。

```jsx:app/articles/[slug]/page.tsx
'use client';

export default function Page({ params, searchParams }: Props) {
  const [data, setData] = useState();

  const fetchDetail = async () => {
    const res = await getDetail(params.slug, {
      draftKey: searchParams.dk,
    });
    setData(res);
  };

  useEffect(() => {
    fetchDetail();
  }, []);

  return data ? <Article data={data} /> : <p>loading...</p>;
}
```

要点は3つ

- ファイルの先頭に`'use client';`ディレクティブを付ける
- Pageコンポーネントからasyncを削除する
- fetchは`useEffect`内で実行する

従来のFunctional Componentですね。

Next.jsはデフォルトでSSRの挙動を取るので、`loading...`が埋め込まれた状態で、サーバーからクライアントにレスポンスが返されます。
その後、クライアント側で`useEffect`が実行されるので、しばらく`loading...`が表示されたあとに`<Article />`に切り替わる挙動になります。

実際にクライアント側でfetchする場合には、Tanstack QueryやSWRに代表されるキャッシュ機構を備えたライブラリを使用することになりますし、APIキーが表側に露出しないように[Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)などを活用することになると思います。


## CCでSSRしない

next/dynamicを使うことで、SSRでは読み込ませず、クライアントでのみ実行するコンポーネントを作成できます。

https://nextjs.org/docs/app/building-your-application/optimizing/lazy-loading#nextdynamic

こちらも説明が長くなるので参考記事を添付します。

https://zenn.dev/euxn23/articles/cecab60f672d2d

## Static Exports

詳細は割愛しますが、静的生成も健在です。

https://nextjs.org/docs/app/building-your-application/deploying/static-exports