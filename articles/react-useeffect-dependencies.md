---
title: "依存関係の不足したuseEffectを作らないための考え方"
emoji: "🐙"
type: "tech"
topics: ["React","useEffect"]
published: true
publication_name: "frontendflat"
---

useEffectの依存関係（dependencies）は以下のように定義されており、依存関係にはsetupコード内で参照されるリアクティブな値をすべて指定する必要があります。

> The list of all reactive values referenced inside of the setup code. Reactive values include props, state, and all the variables and functions declared directly inside your component body.
> 
> setupコードの内部で参照されるすべてのリアクティブな値のリスト。リアクティブな値には、props、state、そしてコンポーネントボディの内部で直接宣言されたすべての変数と関数が含まれます。

https://react.dev/reference/react/useEffect

しかし、useEffectは「ある値が変わったときにある処理を実行する」ような実装ができてしまうので、依存関係が不足した誤ったコードを書いてしまいがちです。

## 依存関係が不足したコードとは

```jsx
useEffect(() => {
  const diff = prevValue - value;
  doSomething(diff);
}, [value]);
```

`value`が変わったときに`doSomething`を実行するような実装です。これは仮に動作に問題がなかったとしても、useEffectの使い方としては誤っています。

正しくは`prevValue` `value` `doSomething`の三つの値を依存関係に指定しますが、一度完成したロジックに対してあとから不足した依存関係を追加すると、意図しないタイミングで処理が実行されてバグってしまう可能性があります。

この誤った実装のダメポイントは、**useEffectありきでロジックを作ってしまっていること**です。useEffectの依存関係を使ってプログラムの条件（上の例では「`value`が変わったときに」という条件）を作ろうとしてしまうと依存関係が不足した実装になってしまいます。

そこで**一度useEffectなしでロジックを作ってしまい、あとからuseEffectでラップする**という考え方をすると、依存関係を正しく指定することができます。

## 正しいuseEffectの作り方

1. 「`value`が変わったときに`doSomething`を実行する」というロジックをuseEffectなしで作る

```jsx
if (prevValue !== value) {
  const diff = prevValue - value;
  doSomething(diff);
}
```

2. あとからuseEffectでラップする

```jsx
useEffect(() => {
  if (prevValue !== value) {
    const diff = prevValue - value;
    doSomething(diff);
  }
}, [prevValue, value, doSomething]);
```

この2ステップを踏むことで、1でロジックを作り、2ではただuseEffectでラップし依存関係をすべて指定するだけというシンプルな考え方になり、useEffectを正しく使うことができます。

## まとめ

- ドキュメントにも書いてありますが、useEffectの依存関係にはsetupコード内で参照されるリアクティブな値をすべて指定しましょう
- useEffectありきでロジックを作ってしまわないように、ロジックを2ステップに分けて考えると良いでしょう
- そもそもESLintが依存関係の不足を検知しエラーを出してくれるので、リンターは必ず導入しましょう