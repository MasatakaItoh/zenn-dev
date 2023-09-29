---
title: "useCallbackのユースケース3選"
emoji: "3️⃣"
type: "tech"
topics: ["React","useCallback"]
published: true
publication_name: "frontendflat"
---

自分がReactに入門した頃は、メモ化に対して漠然と難しそうなイメージを持っていました。
今思い返すと「メモ化を使うとキャッシュできるのはわかるけど、具体的にどういうときに使うのかわからない」ような感情をいだいていた記憶があります。

ということで今回はメモ化の中でも関数をメモする`useCallback`を対象に、`useCallback`のユースケースは3つしかないという話をします。

1. `memo`化されたコンポーネントのpropsにわたす関数
2. `useEffect`や`useMemo`の依存に含まれる関数 
3. カスタムフック内の関数

ユースケースが3つしかないとわかれば、少なくとも「漠然と難しそう」という状態を脱することができるはずです。

## 1. memo化されたコンポーネントのpropsにわたす関数 

コンポーネントを`memo`化すると、propsが変更しないときに再レンダリングをスキップすることができます。 しかし、以下の例では`memo`が全く意味のないものになります。

```jsx
const Child = memo(({ onClick }) => {});

const Parent = () => {
  const handleClick = () => {};
  return <Child onClick={handleClick} />
};
```

どういうことかというと、関数はレンダリングごとに異なる参照を生成するので、レンダリングごとに`handleClick`つまり`onClick`propsが変化したと見なされます。
propsが変化したときは`memo`の有無に関わらず子コンポーネントが再レンダリングされるので、`memo`が意味のないものになっているということですね。

これを解消するためには`handleClick`を`useCallback`でラップします。

```jsx
const Child = memo(({ onClick }) => {});

const Parent = () => {
  const handleClick = useCallback(() => {}, []);
  return <Child onClick={handleClick} />
};
```

ちなみに子コンポーネントの`memo`を削除すると、今度は`useCallback`が意味のないものになります。

```jsx
const Child = ({ onClick }) => {};

const Parent = () => {
  const handleClick = useCallback(() => {}, []);
  return <Child onClick={handleClick} />
};
```

各レンダリング間で`handleClick`は同一のものと見なされますが、そもそも`memo`化されていないコンポーネントは毎回再レンダリングされるので、`useCallback`を使うだけでは子コンポーネントの再レンダリングを抑制することはできないということです。

## 2. useEffectやuseMemoの依存に含まれる関数

先ほどと同じ理由ですが、`useEffect`や`useMemo`の依存に関数を渡すだけだと、それらは毎回実行されてしまいます。
そのため、依存に渡す関数は`useCallback`でラップする必要があります。

```jsx
const Component = () => {
  const doSomething = useCallback(() => {}, []);
  useEffect(() => {
    doSomething();
  }, [doSomething]);
  // ...
};
```

また`useCallback`は、関数を`useEffect`や`useMemo`の中かコンポーネント外に移動させることで、不要な`useCallback`や依存を減らすことができる可能性があります。

```jsx
const Component = () => {
  useEffect(() => {
    const doSomething = () => {};
    doSomething();
  }, []);
  // ...
};
```

もしくは

```jsx
const doSomething = () => {};

const Component = () => {
  useEffect(() => {
    doSomething();
  }, []);
  // ...
};
```

コードがよりシンプルになりましたね。

## 3. カスタムフック内の関数

自分はカスタムフックから返される関数はすべて`useCallback`でラップするようにしています。

>カスタムフックを作る理由は、普通の関数を作る理由と全く同じであり、すなわち責務の分離とかカプセル化です。 一度カスタムフックとして分離された以上、インターフェースの内側のことはカスタムフック内で完結すべきです。 カスタムフックを使う側はカスタムフックの内側のことを知るべきではなく、その逆も然りです。
 
詳しくは以下の記事を参照することをおすすめします。

https://blog.uhy.ooo/entry/2021-02-23/usecallback-custom-hooks/

カスタムフックから返された関数も結局のところ、ケース1（`memo`化されたコンポーネントのpropsにわたす）または、ケース2（`useEffect`や`useMemo`の依存にわたす）として使われるのですが、カスタムフックが絡むと考え方が変わるのでケース3として紹介しました。

## 最後に

今回は関数をメモ化する`useCallback`のユースケースについて紹介しました。

`useCallback`と似たものに`useMemo`がありますが、`useMemo`は`useCallback`とは異なり、オブジェクト（関数含む）だけではなくプリミティブな値もメモ化しようと思えばできてしまいます。
つまりメモ化する基準や考え方が異なるので、別途学習すると良いでしょう。最後に`useMemo`について、個人的にわかりやすいなと思った参考記事を添付しておきます。

https://zenn.dev/chot/articles/react-when-to-use-memo