---
title: "Next.jsの4つのキャッシュ"
emoji: "4️⃣"
type: "tech"
topics: ["Nextjs","React","ReactCache"]
published: true
publication_name: "frontendflat"
---
今回はNext.jsの4つのキャッシュについて、それぞれの特性と役割を説明します。

:::message
一つ目のReact Cacheは、後述する他の3つのキャッシュの説明のために必要なので紹介しますが、Next.jsの機能ではなくReactの機能です。React Cache以外のキャッシュはNext.jsの機能に当たります。
:::

## React Cache

一つ目のキャッシュは[React Cache](https://react.dev/reference/react/cache)によるもので、同一レンダリング内における同じURLとオプションを持つリクエストをメモ化します。React Cacheの役割は、**重複リクエストを排除し、リクエスト数を減らすこと**です。
Next.jsでは、[fetch](https://nextjs.org/docs/app/api-reference/functions/fetch)を利用するだけでこれを実現できますし、fetchを使えない場合はReact Cacheを直接使用することで実現できます。

具体的には以下のような挙動を取ります。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Frequest-memoization.png&w=3840&q=75&dpl=dpl_8ryKRR41mbmmJprfJDzPycmyzNEP)

1. 特定のリクエストが初めて呼び出される。キャッシュは存在しないのでデータソースへのリクエストを実行し、レスポンスをメモリに保存する。
2. 同一レンダリング内で同一リクエストが実行されると、データソースへのリクエストは実行されずメモリからデータが返される。
3. レンダリングが終了し、メモリのキャッシュは破棄される。

## Data Cache

Next.jsは、fetchした結果をData Cacheとして保存します。fetchによりデータソースにリクエストした結果はData Cacheに保存され、以降同一リクエストはData Cacheから返されます。つまりData Cacheの役割は、**オリジンデータソースへのリクエスト数を減らすこと**です。

また、Data Cacheは永続的で複数デプロイ間にわたって有効なので、再デプロイしてもキャッシュは破棄されません。なので一定期間や特定のタイミングでキャッシュを破棄するために[fetchオプション](https://nextjs.org/docs/app/api-reference/functions/fetch#optionsnextrevalidate)などによる再検証が用意されています。先ほどのReact Cacheはレンダリングごとに自動的にキャッシュが破棄されましたが、キャッシュされる期間が大きく異なるので混同しないように注意しましょう。

React Cacheと合わせると、以下のような挙動を取ります。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Fdata-cache.png&w=3840&q=75&dpl=dpl_8ryKRR41mbmmJprfJDzPycmyzNEP)

- メモリのキャッシュの有無を確認し、あればメモリからデータを返却し、なければData Cacheの有無を確認します。
- 同様にData CacheがあればData Cacheからデータを返却し、なければオリジンデータソースへのリクエストを実行します。
- リクエストの結果はメモリとData Cacheそれぞれに保存されます。
- fetchオプションとして`{ cache: 'no-store' }`を指定すると、Data Cacheへの保存はスキップされます。
- メモリのキャッシュはレンダリングごとに破棄されるので、レンダリングごとに必ず一回はData Cacheへのリクエストを実行します。
- 一方、Data Cacheは再検証されない限り有効なので、オリジンデータソースへのリクエストは実行されません。

## Full Route Cache

Next.jsは、ビルドまたは再検証時に静的にレンダリングされたルートに対して、サーバーサイドでRSC PayloadとHTMLを生成し、それらをFull Route Cacheとして保存します。
Next.jsはRSC PayloadとHTMLを利用してレンダリングを行うので、クライアントサイドで静的ルートにアクセスしたときには既にData Cacheやデータソースへのリクエストは完了していることを意味します。つまり、Full Route Cacheの役割は、**静的ルート自体をキャッシュし、不要なリクエスト、つまりレンダリングコストを削減すること**です。

またFull Route Cacheは、Data Cacheと同様に永続的で再検証によりキャッシュを破棄することができますが、Data Cacheとは異なり再デプロイによってもキャッシュは破棄されます。

ちなみに動的ルートはリクエスト時にレンダリングされるので、Full Route Cacheへの保存はスキップされます。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Fcaching-overview.png&w=3840&q=75&dpl=dpl_8ryKRR41mbmmJprfJDzPycmyzNEP)

RSC PayloadとHTMLの詳細についてはこちらの記事を参照ください。

https://zenn.dev/frontendflat/articles/nextjs-rscpayload-composition

## Router Cache

Router Cacheは4つのキャッシュの中で唯一クライアントサイドのキャッシュに該当します。
`<Link />`のprefetchによりルートを事前に取得し、訪問済みのルートをクライアントサイドで静的/動的ルート問わずキャッシュします。これによりフォワード/バックナビゲーションによる高速な画面遷移を実現することができます。つまり、Router Cacheの役割は、**ナビゲーションのUX向上**です。

![](https://nextjs.org/_next/image?url=%2Fdocs%2Fdark%2Frouter-cache.png&w=3840&q=75&dpl=dpl_8ryKRR41mbmmJprfJDzPycmyzNEP)

Router Cacheはユーザーセッション中で有効なので、リロードするとキャッシュが破棄されます。また、時間経過によっても破棄されます。静的レンダリングは5分、動的レンダリングは30秒、キャッシュに保存されます。

## まとめ

表にまとめると

| キャッシュの種類 | キャッシュされる場所 | キャッシュ期間       | 役割                       |
|----------|------------|---------------|--------------------------|
| React Cache | サーバーサイド    | レンダリング中       | リクエスト数を減らすこと             |
| Data Cache | サーバーサイド    | 複数デプロイ（再検証可能） | オリジンデータソースへのリクエスト数を減らすこと |
| Full Route Cache | サーバーサイド    | 単一デプロイ（再検証可能） | 静的ルートのレンダリングコストを削減すること   |
| Router Cache | クライアントサイド  | ユーザーセッション中    | ナビゲーションのUX向上 |

要点をまとめると

1. 前提として、React Cacheによりサーバーサイドでのリクエストの総数を削減（＝リクエストをメモ化）している。
2. リクエストをメモ化した上で、Data Cacheによりオリジンデータリソースへのリクエスト数を削減している。
3. そもそも静的ルートに関しては、1と2をビルドと再検証時に実行し、その結果をFull Route Cacheとして保存しているのでレンダリングコストが少なくて済む。
4. Router Cacheにより、静的ルートも動的ルートもナビゲーションのUXが良い。