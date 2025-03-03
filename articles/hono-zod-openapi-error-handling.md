---
title: "Hono Zod OpenAPIのエラーハンドリング"
emoji: "🐡"
type: "tech"
topics: ["Hono","Zod","OpenAPI","TypeScript"]
published: true
publication_name: "frontendflat"
---

## はじめに

今回は、Honoとその3rd-party MiddlewareであるZod OpenAPIを使ったときのAPIエラーハンドリングについて紹介します。APIのエラーハンドリングというと、主に以下のようなケースが考えられます。

- パスやボディ、クエリなどのリクエストパラメータが不正な場合のバリデーションエラー（400 Bad Request）
- トークンが存在しなかったり、期限切れになっているなどの認証エラー（401 Unauthorized）
- データベース接続の失敗や外部サービスとの連携時に起こるサーバーエラー（500 Internal Server Error）

HonoのミドルウェアやZod OpenAPIを活用することで、エラーハンドリングは簡単に実装することができますが、今回はそれらの機能を組み合わせ、**エラー処理を一元管理し、レスポンス形式を統一する**ことを目指します。

## サンプルリポジトリ

今回の記事の内容を実装したサンプルリポジトリも用意しました。記事と合わせてご覧ください。

https://github.com/MasatakaItoh/hono-zod-openapi-error-handling

また、各ライブラリの導入手順については、公式ドキュメントやGitHubのガイドの通りなので、ここでは詳細は割愛します。必要な場合は、以下のリンクを参考にしてください。

https://hono.dev/docs/getting-started/basic

https://hono.dev/examples/zod-openapi

https://hono.dev/examples/swagger-ui

## エラーハンドリングの実装手順

ここからが本題です。Honoでは、下記のようにエラーハンドリングを実装することができます。

- ミドルウェアやルートでは、Honoが提供する`HTTPException`を使ってエラーをスローする。 
- トップレベルの`app.onError`により、スローされたエラーをキャッチし、共通のエラーハンドリングを行う。

それでは、実装例を見ていきましょう。

### 1. トップレベルの共通エラーハンドリングを実装する

今回は、共通して利用するAPIのエラーレスポンス型`ErrorResponse`を以下のように定義します。`message`はエラーメッセージ、`errors`には各フィールドごとのバリデーションエラー等の詳細を格納します。

```typescript
type ErrorResponse = {
  message: string;
  errors?: {
    [key: string]: string[];
  }
}
```

Honoでは、`app.onError`を使うことで、各ミドルウェアやルートでスローされたエラーをキャッチし、トップレベルで共通の処理を実装することが可能です。

https://github.com/MasatakaItoh/hono-zod-openapi-error-handling/blob/main/src/index.ts#L17-L35

後ほど`HTTPException`をスローする際に、`cause`オプションとしてZodのエラー`ZodError`が渡ってくるため、`errors`の形式に変換して返しています。

https://hono.dev/docs/api/hono#error-handling

https://hono.dev/docs/api/exception

### 2. バリデーションエラーを実装する

Zod OpenAPIでは、各ルートの処理後に実行されるフックを登録することで、バリデーションに失敗した場合のエラーハンドリングが可能です。さらに、`OpenAPIHono`の`defaultHook`を利用することで、各ルートごとにフックを個別に設定せずに、共通のハンドリングを実現できます。

https://github.com/MasatakaItoh/hono-zod-openapi-error-handling/blob/main/src/hono.ts#L8-L17

バリデーションが失敗した場合、`result.error`に含まれる`ZodError`を`HTTPException`の`cause`にセットして400エラーをスローしています。これにより、バリデーションエラーを`app.onError`でキャッチし、`ErrorResponse`型に変換してクライアントに返すことができます。

https://github.com/honojs/middleware/tree/main/packages/zod-openapi#a-dry-approach-to-handling-validation-errors

実際のアプリケーションではルートをファイル分割して管理したくなりますが、その場合は各`OpenAPIHono`インスタンスに対して個別に`defaultHook`を設定する必要があります。サンプルリポジトリでは、`createOpenApiHono`という関数を用意してインスタンスの生成を共通化しています。

https://github.com/honojs/middleware/issues/323

### 3. 認証エラーを実装する

Honoでは、`createMiddleware`を利用して独自のミドルウェアを作成することが可能です。今回の例では、Authorizationヘッダを簡単に検証するミドルウェアを実装しています。

https://github.com/MasatakaItoh/hono-zod-openapi-error-handling/blob/main/src/middleware.ts#L8-L28

https://github.com/MasatakaItoh/hono-zod-openapi-error-handling/blob/main/src/route/protected.ts#L60-L61

Authorizationヘッダが存在しない場合や、アクセストークンの検証に失敗した場合は、それぞれ`HTTPException`を使って401エラーをスローしています。このようにミドルウェアでも、先ほどのバリデーションエラーと同様の形式で`HTTPException`をスローするだけで、共通のエラーハンドリングが実現できます。

https://hono.dev/docs/helpers/factory#createmiddleware

### 4. APIドキュメントにエラーレスポンスを追加する

Honoでは、Zod OpenAPIとSwagger UIを組み合わせることで、自動生成されたOpenAPI仕様のドキュメントをSwagger UIとして表示できます。

https://github.com/honojs/middleware/tree/main/packages/swagger-ui#with-openapihono-usage

この際、OpenAPIにエラーレスポンスの型を定義することで、Swagger UI上にもエラー時のスキーマが表示されるようになりますが、各ルートに個別に記述するのは面倒なため、共通化しておきます。また、手順1で定義した`ErrorResponse`型は、`createErrorResponseSchema`から算出できるため、`z.infer`を利用してここで定義しておきます。

https://github.com/MasatakaItoh/hono-zod-openapi-error-handling/blob/main/src/error-schema.ts#L5-L66

続いて、各ルートに`errorResponses`を追加していきます。サンプルリポジトリでは、認証のミドルウェアを実装した`/protected`ルートと、認証不要の`/public`ルートを用意しており、`/public`ルートには401エラーを除いた`publicErrorResponses`を指定しています。

https://github.com/MasatakaItoh/hono-zod-openapi-error-handling/blob/main/src/error-schema.ts#L74-L77

https://github.com/MasatakaItoh/hono-zod-openapi-error-handling/blob/main/src/route/public.ts#L6-L45

![](/images/hono-zod-openapi-error-handling/swagger-ui.png)

## さいごに

以上の手順により、各種エラーの処理を`app.onError`で一元管理し、APIのエラーレスポンスを`ErrorResponse`形式に統一することができました。Zod OpenAPIを利用したエラーハンドリングの一例として、ぜひ参考にしてみてください。