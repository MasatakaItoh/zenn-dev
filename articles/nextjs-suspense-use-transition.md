---
title: "URLクエリパラメータによる検索画面をSuspenseで実装する3パターン"
emoji: "3️⃣"
type: "tech"
topics: ["Nextjs","RSC","React","Suspense","useTransition"]
published: true
publication_name: "frontendflat"
---

今回はNext.jsと`Suspense`を使ったURLクエリパラメータによる検索画面の実装方法を3つご紹介します。

1. Suspense + RSC 
2. Suspense + CC
3. Suspense + useTransition

3番目のパターンのサンプルコードを用意したのでよければご参照ください。

https://github.com/MasatakaItoh/nextjs-resas

## 1. Suspense + RSC

RSC（サーバーコンポーネント）でのデータフェッチには`async await`が使えます。後述する`use`を使うこともできますが、パフォーマンスの観点から`async await`を使うことが推奨されています。

また検索パラメータはURLクエリパラメータから取得しますが、Next.jsの`useSearchParams`はCC（クライアントコンポーネント）でしか使えません。RSCでは`page.tsx`の`searchParams`propsからURLクエリパラメータにアクセスできるので、それをpropsとしてRSCに渡します。

```tsx:RegionalEmployTable.tsx
// RSCではasync awaitが使える
export const RegionalEmployTable = async ({
  // page.tsxのsearchParamsをpropsとして渡す 
  searchParams
}) => {
  const params = new URLSearchParams();
  params.set("prefCode", searchParams.prefCode ?? "1");

  const { result } = await getRegionalEmploy(params.toString());
};
```

https://react.dev/reference/react/use#caveats

https://nextjs.org/docs/app/api-reference/functions/use-search-params

https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional


そしてRSCを`Suspense`でラップします。ポイントは`key`を指定していることで、URLクエリパラメータのみ変更による画面遷移では`Suspense`が再実行されないため、`searchParams`ごとに異なる識別子を`Suspense`に与えます。

> URLクエリパラメータのみ変更による画面遷移では`Suspense`が再実行されない

ちなみにこちらの問題の対策として、`router.push`や`<Link />`によるSPA画面遷移ではなく、`<a />`によるMPA画面遷移にしてしまうという方法も考えられます。

```tsx:RegionalEmployContainer.tsx
// Suspenseに識別子を与える
<Suspense key={JSON.stringify(searchParams)} fallback={<Loading />}>
  <RegionalEmployTable searchParams={searchParams} />
</Suspense>
```

https://react.dev/learn/preserving-and-resetting-state#option-2-resetting-state-with-a-key

これにより、URLクエリパラメータが変更されるたびに`Suspense`のフォールバックUIを表示する検索画面が実装できました。

![ ](https://gyazo.com/0ea3e1ae769332ce3509340d1fce2b03/raw)

## 2. Suspense + CC

先ほど Suspense + RSC で作ったものを Suspense + CC で作ってみます。

今回はCCなのでURLクエリパラメータは`useSearchParams`から取得できます。また`Promise`からのデータ取得には`use`を使ってみます。

```tsx:RegionalEmployTable.tsx
export const RegionalEmployTable = () => {
  // CCなのでuseSearchParamsを使える
  const searchParams = useSearchParams()
  // useでPromiseからのデータ取得が可能
  const { result: regionalEmploy } = use(getRegionalEmploy(searchParams.toString()));
};
```

https://ja.react.dev/reference/react/use

RSCで実行されない`fetch`（クライアントサイドフェッチ）にはキャッシュ機構が備わっていないので、キャッシュ機構が必要な場合はTanstack Queryなどのライブラリを組み合わせると良いでしょう。

https://tanstack.com/query/latest/docs/framework/react/guides/suspense

セキュリティやパフォーマンスなど、Suspense + RSC のメリットを教授できなくなりますが、RSC と CC の使い分けコストや複雑性を考慮して、従来通り Suspense + CC で実装することも十分選択肢に入ると思います。

## 3. Suspense + useTransition

これは RSC と CC ともに実装できますが、今回は RSC のパターンでご紹介します。まずRSCの実装は Suspense + RSC と同様です。

```tsx:RegionalEmployTable.tsx
// RSCの実装はSuspense + RSCと同様
export const RegionalEmployTable = async ({ searchParams }) => {
  const params = new URLSearchParams();
  params.set("prefCode", searchParams.prefCode ?? "1");

  const { result } = await getRegionalEmploy(params.toString());
};
```

続いて`Suspense`の実装ですが、`Suspense`は初回データフェッチのフォールバックUIとしてのみ利用するので`key`の指定は不要です。今回はNext.jsの`loading.tsx`を使います。

```tsx:loading.tsx
export default function Loading() {
  return (
    <p>Loading ....</p>
  );
}
```

そしてここからがポイントで、以下のように実装することでページ遷移状態を取得することができます。 

1. `router.push`を`startTransition`でラップ（画面遷移をトランジションとしてマーク）し、トランジション中かどうかを表す`isSearching`フラグを取得する。
2. アプリケーション全体をラップするcontextに対して、1で取得したフラグを渡す。
3. 2のcontextから各CCに対してフラグを渡し、「トランジション中はローディング中のUIを表示する」という実装をする。

```tsx:RouterTransitionProvider
"use client";
import { createContext, PropsWithChildren, useContext, useState } from "react";

export type RouterTransitionContext = {
  isPending: boolean;
  setIsPending: (isLoading: boolean) => void;
};

const RouterTransitionContext = createContext<RouterTransitionContext>({
  isPending: false,
  setIsPending: () => {},
});

// アプリケーション全体をラップするcontextを実装する
export const RouterTransitionProvider = ({ children }: PropsWithChildren) => {
  const [isPending, setIsPending] = useState(false);
  return (
    <RouterTransitionContext.Provider value={{ isPending, setIsPending }}>
      {children}
    </RouterTransitionContext.Provider>
  );
};

// トランジション状態を各CCに提供できるようにする
export const useRouterTransition = () => useContext(RouterTransitionContext);
```

```tsx:useChangeParams.ts
"use client";
import { useRouter, useSearchParams } from "next/navigation";
import { useEffect, useTransition } from "react";

export const useChangeParams = (
  key: "prefCode" | "year" | "matter" | "class",
) => {
  const router = useRouter();
  const searchParams = useSearchParams();
  const [isSearching, startTransition] = useTransition();
  const { setIsPending } = useRouterTransition();

  const handleChange = (v: string) => {
    const params = new URLSearchParams(searchParams);
    params.set(key, v);
    // 画面遷移をトランジションとしてマークする
    startTransition(() => {
      router.push(`/?${params.toString()}`);
    });
  };

  useEffect(() => {
    setIsPending(isSearching);
  }, [isSearching]);

  return handleChange;
};
```

```tsx:ClassSelect.tsx
// URLクエリパラメータを変更するプルダウンUI
export const ClassSelect = () => {
  const handleChange = useChangeParams("class");
  // contextからトランジション状態を取得する
  const { isPending } = useRouterTransition();

  return (
    // プルダウンのonChangeで、トランジションでマークされた画面遷移を実行する
    <Select defaultValue={CLASSES[0].value} onValueChange={handleChange}>
      {/* トランジション中はdisableにする */}
      <SelectTrigger disabled={isPending}>
        <SelectValue />
      </SelectTrigger>
      <SelectContent>
        {CLASSES.map(({ label, value }) => (
          <SelectItem key={value} value={value}>
            {label}
          </SelectItem>
        ))}
      </SelectContent>
    </Select>
  );
};
```

上記のようにトランジション状態をプルダウンやテーブルなど各UIに適用することで、このようなアプリケーションを実装することができました。
初期表示としてNext.jsの`loading.tsx`による画面全体のフォールバックUIの表示、その後検索条件を変えたときはテーブルを非表示にせずにローディング中を表すUI（CSSによる透過）を表現できています。

![](/images/nextjs-suspense-use-transition/suspense-use-transition.gif)

通常、検索結果画面のようなUIにおいては、再検索によってテーブル自体が非表示になることは好まれません。
今回はURLクエリパラメータのみ変更による画面遷移だったので上述の通りそもそも`Suspense`が再実行されませんが、`pathname`が変化する画面遷移においては`Suspense`が再実行されフォールバックUIが表示されます（Suspense + RSC と同様の挙動）が、`startTransition`により画面遷移をトランジションとしてマークすることで、`Suspense`のフォールバックUIを表示しないということが可能です。そういう意味でも Suspense + useTransition はセットで使うことでより真価を発揮します。

https://ja.react.dev/reference/react/useTransition

https://zenn.dev/uhyo/books/react-concurrent-handson-2

## さいごに

今回はURLクエリパラメータを使った検索画面の実装方法を3つご紹介しました。

RSCを使うメリットデメリット、`Suspense`単体でできること、`Suspense`と`useTransition`を組み合わせることでできること、`<a />`遷移によるMPAなど、データフェッチするにもいくつかの選択肢があります。それぞれのメリットデメリットを考慮して、適切な実装方法を選択しましょう。