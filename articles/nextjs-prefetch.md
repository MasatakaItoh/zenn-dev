---
title: "App Routerにおけるプリフェッチの挙動"
emoji: "📘"
type: "tech"
topics: ["Next.js", "React", "router"]
published: true
publication_name: "frontendflat"
---
## Prefetchとは

プリフェッチとは、ユーザーが特定のページを訪れる前に、バックグラウンドでそのページを事前取得（プリロード）することです。
Next.jsでは遷移先のRouteをプリフェッチしキャッシュします。
クライアントサイドレンダリング（CSR）において、遷移先のページをシームレスに表示するためにはプリフェッチが必要不可欠です。

ちなみにNext.jsでは、Linkコンポーネントまたは`router.prefetch()`を使うことでプリフェッチを実行できます。
`router.push()`におけるナビゲーションではプリフェッチされないので、ナビゲーションにはLinkコンポーネントを使用することが推奨されています。

https://nextjs.org/docs/app/api-reference/functions/use-router

## DevelopmentとProductionによる挙動の違い

> Prefetching is only enabled in production.

公式ドキュメントには「プリフェッチはproductionでのみ有効」と記載がありますが、正確にはDevelopmentでもプリフェッチされます。
以下のようにプリフェッチされるタイミングが異なり、Productionでは本番環境向けにプリフェッチの挙動が強化されるイメージです。

- Development: Linkコンポーネントをhoverした際にプリフェッチが実行されます。
- Production: 最初のページロードやスクロールによって、Linkコンポーネントがビューポートに入ると自動的にプリフェッチが行われます。

https://nextjs.org/docs/app/api-reference/components/link#prefetch

ちなみにLinkコンポーネントの`prefetch`propsを`false`にすることでこの挙動を無効化できますが、これはDevelopmentとProductionどちらの挙動にも適用されます。

## Static RoutesとDynamic Routesによる挙動の違い

プリフェッチの挙動は、ルーティングによっても異なります。

- Static Routes: Linkコンポーネントの`prefetch`propsのデフォルト値は`true`で、そのルート全体がプリフェッチされ、キャッシュされます。
- Dynamic Routes: 直近の`loading.js`ファイルまでの共有レイアウトのみをプリフェッチし30秒間キャッシュします。つまり、ルートレイアウトやネストされた`layout.js`のみがプリフェッチの対象となり、Dynamic Routes全体をfetchするコストを削減しつつ、ユーザーに対して即座にロード状態を提供することができることを意味します。

## まとめ

プリフェッチはデフォルトで有効なのであまり意識したことがない人もいるかもしれませんが、CSRにおけるパフォーマンスやユーザー体験を支える重要なテクニックです。
DevelopmentとProduction、Static RoutesとDynamic Routesにおける挙動の違いを正しく理解し、必要に応じてプリフェッチの挙動を調整できるようになると良いですね。