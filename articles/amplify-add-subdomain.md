---
title: "ドメイン取得サービスで取得したサブドメインだけをAmplifyアプリケーションに割り当てる"
emoji: "🅰️"
type: "tech"
topics: ["Amplify", "AWS", "DNS"]
published: true
---
## はじめに

ドメイン取得サービスで取得したドメインのサブドメインだけをAmplifyアプリケーションに割り当てたいという場合、サブドメインにCNAMEレコードを設定することで実現できます。
今回はムームードメインをドメイン取得サービスとして利用します。

## 手順

ムームードメインでサブドメインを取得している前提で話を進めます。

### Amplify Consoleでカスタムドメインを追加

まずはムームードメインで取得したドメインを設定します。 今回はサブドメインだけを割り当てたいので、ルートを除外し、取得したサブドメインを追加します。

![](/images/amplify-add-subdomain/flow1.png)

## ムームードメインでCNAMEレコードを設定

カスタムドメインを追加すると、SSLの設定が始まります。画面中央に表示されるサブドメインとCNAMEレコードをムームードメイン側で設定します。

![](/images/amplify-add-subdomain/flow2.png)

![](/images/amplify-add-subdomain/flow3.png)

追加でCNAMEレコードの追加を促されるので、アクション→DNSレコードを表示をクリックします。

![](/images/amplify-add-subdomain/flow4.png)

CloudFrontのCNAMEレコードをムームードメイン側で設定します。

![](/images/amplify-add-subdomain/flow5.png)

![](/images/amplify-add-subdomain/flow6.png)

以上で設定は完了です。

ドメインがアクティベートされたら、ムームードメインで取得したサブドメインにアクセスし、Amplifyアプリケーションが表示されることを確認しましょう。

## 最後に

現時点では、デフォルトドメインである`xxx.amplifyapp.com`からもアプリケーションにアクセスできてしまいます。
Amplifyではデフォルトドメインを無効化することはできませんが、デフォルトドメインへのアクセス時にカスタムドメインへリダイレクトさせる設定をしておくと良いと思います。
詳しい手順は以下の記事を参照してみてください。

https://dev.classmethod.jp/articles/tsnote-amplify-how-do-i-disable-the-default-domain-for-amplify-hosting/

