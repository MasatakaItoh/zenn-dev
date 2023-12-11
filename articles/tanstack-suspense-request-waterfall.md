---
title: "useSuspenseQueryでリクエストウォーターフォールに気をつけよう"
emoji: "🐉"
type: "tech"
topics: ["React","TanstackQuery","Suspense"]
published: true
publication_name: "frontendflat"
---

Tanstack Query v5 から Suspense によるデータ取得が安定板となったことで`useSuspenseQuery`を使い始めた方もいるのではないでしょうか。

今回はそんな`useSuspenseQuery`を使う上で、リクエストウォーターフォールを発生させないために気をつけるべきことをご紹介します。

https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5#new-hooks-for-suspense

## コンポーネント内で複数のuseSuspenseQueryを実行しない

`useQuery`と同じような感覚でコンポーネント内で複数のクエリを実行すると、リクエストウォーターフォールが発生します。一つ目のクエリが内部的にPromiseをthrowするため、二つ目以降のクエリが実行される前にコンポーネントがサスペンドするためです。

```jsx:NG
const firstQuery = useSuspenseQuery({ queryKey: ['first'], queryFn: fetchFirst });
const secondQuery = useSuspenseQuery({ queryKey: ['second'], queryFn: fetchSecond });
```

またリクエストウォーターフォールというと、ネストしたコンポーネント間で発生することが多いですが、`useSuspenseQuery`では上記と同様の理由により、クエリを親コンポーネントに移動するだけではリクエストウォーターフォールを解消することができません。

リクエストウォーターフォールを解消するには、主に二つの方法が考えられます。

- `useSuspenseQueries`を使用する
- コンポーネントを分割する

## useSuspenseQueriesを使用する

コンポーネント内で複数のクエリを実行する場合は、`useSuspenseQueries`を使用するだけでリクエストウォーターフォールを解消できます。

```jsx:OK
const [firstQuery, secondQuery] = useSuspenseQueries({
  queries: [
    { queryKey: ['first'], queryFn: fetchFirst },
    { queryKey: ['second'], queryFn: fetchSecond },
  ]
});
```

## コンポーネントを分割する

コンポーネントを分割し、それぞれのコンポーネントを異なるSuspense境界に配置することでもリクエストウォーターフォールを解消できます。

```jsx:OK
// First.tsx
const firstQuery = useSuspenseQuery({ queryKey: ['first'], queryFn: fetchFirst });

// Second.tsx
const secondQuery = useSuspenseQuery({ queryKey: ['second'], queryFn: fetchSecond });

<Suspense fallback={<Loading />}>
  <First />
</Suspense>
<Suspense fallback={<Loading />}>
  <Second />
</Suspense>
```

コンポーネントを分割しても複数のクエリが同じSuspense境界に配置されていると、その境界内全体がサスペンドされるのでご注意ください。

```jsx:NG
<Suspense fallback={<Loading />}>
  <First />
  <Second />
</Suspense>
```

## まとめ

今回の例はシンプルでSuspense境界がわかりやすかったですが、実際のアプリケーションは複雑で気付かぬうちにリクエストウォーターフォールを発生していることがあります。

レイテンシの大きいクライアントサイドフェッチにおいては、リクエストウォーターフォールがパフォーマンスに与える影響は大きいです。
リクエストウォーターフォールの有無は Developer Tools の Network タブを確認すればすぐにわかるのでチェックしつつ、`useSuspenseQuery`と`useSuspenseQueries`を活用していきましょう！