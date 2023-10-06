---
title: "Next.js - RSC PayloadとComposition Pattern"
emoji: "💭"
type: "tech"
topics: ["Nextjs","React","Composition"]
published: true
publication_name: "frontendflat"
---
今回は「Composition Patternにより、なぜクライアントコンポーネント（CC）の中にサーバーコンポーネント（SC）を配置できるのか？」について二つの観点から説明します。

- RSC Payload
- レンダリングの流れ

この二つの観点を理解することで、Composition Patternへの理解を深めることができます。

## React Server Component Payload（RSC Payload）とは

レンダリングは、サーバーサイドへのリクエストにより始まりますが、サーバーサイドではまずSCからRSC Payloadをレンダリングします。

RSC Payloadには以下の内容が含まれます。

- SCのレンダリング結果
- CCをレンダリングする場所のplaceholderとJavaScriptファイルへの参照
- SCからCCに渡されたprops

詳細は後述しますが、RSC PayloadにはSCをレンダリングするために必要な情報が含まれており、サーバーサイドとクライアントサイドそれぞれでHTMLをレンダリングする際に利用されます。

## レンダリングの流れ

Next.jsはザックリ以下のような流れでSCとCCをレンダリングします。

1. サーバーサイド
   1. SCからRSC Payloadをレンダリングする
   2. RSC PayloadとCCから初期表示のためのHTMLをレンダリングする（SSR）
2. クライアントサイド
   1. 1-2で生成されたHTMLを元にすぐに非インタラクティブな画面を表示する
   2. RSC Payloadを元にSCとCCをツリー上で紐づけ、レンダリングする
   3. JavaScriptをhydrateし、CCをインタラクティブにする

要点は三つ

- SCのレンダリングは1-1でしか行われない
- CCのレンダリングは1-2と2-2で行われる
- RSC Payloadは1-2と2-2で使用される

ページの初期ロードやユーザーアクションによりサーバーサイドへのアクセスが必要になったときは、常に新しいリクエストがサーバーに送信され、新しいレンダリングが始まります。つまり常にレンダリングの流れは不変であり、サーバーサイド→クライアントサイドの順番で行われるということです。

## CCからSCを読み込むことはできない

前述した通り、CCがレンダリングされるとき（1-2, 2-2）にはすでにSCのレンダリング（1-1）が完了しています。
CCからSCを読み込めてしまうと、その時点で新たなサーバーへのリクエストが発生してしまうので、CCにSCを読み込むことはできません。

```jsx:NG
'use client'
import ServerComponent from './Server-Component'
 
export default function ClientComponent() {
  return (
    <>
      <ServerComponent />
    </>
  )
}
```

## Composition Pattern

一方で、SCをpropsとしてCCに渡すこと（Composition Pattern）はできます。

```jsx:OK
import ClientComponent from './client-component'
import ServerComponent from './server-component'
 
export default function Page() {
  return (
    <ClientComponent>
      <ServerComponent />
    </ClientComponent>
  )
}
```

`<ClientComponent>`は`children`propsが何によって埋められるかについては関心がありません。言い換えると`<ClientComponent>`は`children`を配置する場所にのみ責任を持っているということです。
つまり、`<ServerComponent />`を事前にレンダリングし、`<ClientComponent>`をレンダリングするときにその結果を`children`の場所に埋め込むということができます。これがSCをpropsとしてCCに渡すことができる理由です。

また、SCからCCを読み込むことができる理由も同じように考えることができます。

```jsx:OK
import ClientComponent from './client-component'
 
export default function ServerComponent() {
  const props = {};
  return (
    <>
      <ClientComponent {...props} />
    </>
  )
}
```

1-1では、`<ClientComponent>`をplaceholderに置き換えてしまい、`<ServerComponent />`を事前にレンダリングします。そうして生成されたSCのレンダリング結果と合わせ、`<ClientComponent>`を配置する場所、`<ClientComponent>`をレンダリングする際に必要なpropsとJavaScriptをクライアントサイドに返すことで、クライアントサイドでのレンダリングが可能になります。

## クライアントサイドのレンダリングに必要なもの

以上の話を踏まえると、クライアントサイドのレンダリングに必要なものは以下の4つです。

- CC
- SCのレンダリング結果
- CCをレンダリングする場所のplaceholderとJavaScriptファイルへの参照
- SCからCCに渡されたprops

ここでRSC Payloadに含まれる内容を再掲します。

- SCのレンダリング結果
- CCをレンダリングする場所のplaceholderとJavaScriptファイルへの参照
- SCからCCに渡されたprops

CCとRSC Payloadにより、4つの必要条件を満たすことができます。

## 最後に

サーバーサイド→クライアントサイドというレンダリングの流れにおいて、クライアントコンポーネントの中にサーバーコンポーネントを配置するためにはRSC Payloadが必要不可欠です。

Next.jsでサーバーコンポーネントを活用するためにはComposition Patternが必須になってくると思うので、その原理を理解してComposition Patternを使っていきましょう！