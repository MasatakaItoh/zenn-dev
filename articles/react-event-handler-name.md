---
title: "Reactにおけるイベントハンドラの命名規則"
emoji: "📌"
type: "tech"
topics: ["React", "命名規則"]
published: true
publication_name: "frontendflat"
---
Responding to Events では、イベントハンドラの追加方法だけではなく、その命名規則についても触れられているので良いですね。 今回はそれを紹介します。

https://react.dev/learn/responding-to-events

## 基本的な命名規則

>Have names that start with handle, followed by the name of the event. By convention, it is common to name event handlers as handle followed by the event name. You’ll often see onClick={handleClick}, onMouseEnter={handleMouseEnter}, and so on.

>By convention, event handler props should start with on, followed by a capital letter.

まとめるとシンプルに以下のルールで良いです。

- propsとしてのイベントハンドラは`onEventname`（`onClick`など）
- それ以外は`handleEventname`（`handleClick`など）

イベントハンドラの命名用のESLintルールもあるみたいなので、チームで命名規則を統一したい場合は使ってみると良いかもしれません。

https://github.com/jsx-eslint/eslint-plugin-react/blob/master/docs/rules/jsx-handler-names.md

## handleButtonClick？handleClickButton？

個人的にはデフォルトイベントの命名に習って、最近は`handleButtonClick`派です。

https://developer.mozilla.org/ja/docs/Web/Events#%E3%82%A4%E3%83%99%E3%83%B3%E3%83%88%E3%81%AE%E4%B8%80%E8%A6%A7

ちなみにReactのドキュメントでは、`handlePlayClick`や`onPlayMovie`という命名が使われています。
`handlePlayClick`は`handleButtonClick`パターンで、`onPlayMovie`は`handleClickButton`パターンのように見え、命名規則が統一されていない印象を受けます。 と思ったのですが、`click`や`touch`のようなEvent名を命名に使わない場合は`playMovie`のように命名するのが自然なんですかね。
英語がわからなくて「英語の文法的に〜」というのが言えないので、詳しい方がいたら教えてください🫠

## Event名に依存した命名規則

>Here, Toolbar passes them down as onClick handlers to its Buttons, but it could later also trigger them on a keyboard shortcut. Naming props after app-specific interactions like onPlayMovie gives you the flexibility to change how they’re used later.

```tsx
export default function App() {
  return (
    <Toolbar onPlayMovie={() => alert('Playing!')} />
  );
}

function Toolbar({ onPlayMovie }) {
  return (
    <Button onClick={onPlayMovie}>Play Movie</Button>
  );
}
```

上記の例では、`onPlayMovie`は`onPlayClick`のように命名することもできます。
しかし、Event名にちなんだ命名は、簡単で命名規則を統一できる一方で、そのEvent名が該当するイベントハンドラに使用するに限られます。

つまり`onPlayClick`というpropsは、clickイベントハンドラ以外に渡すことはできないということです。 仮にキーボードイベントでも同様の処理を実行したくなったときに、`onPlayClick`という命名のままだと困るという状況が発生する可能性があります。

そこで、あらかじめ`onPlayMovie`のようにアプリケーション固有のインタラクションにちなんだ命名をすることで、propsをあらゆるイベントハンドラに渡しやすくなり、より柔軟に使うことができるようになります。

## まとめ

基本的には`handleEventname` or `onEventname`でEvent名を元に命名をすれば良いと思いますが、アプリケーション固有のインタラクションにちなんだ命名という引き出しを持っておくと、より改修に強いコードを書くことができそうです。