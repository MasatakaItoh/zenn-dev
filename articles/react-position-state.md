---
title: "Reactコンポーネントの位置と状態の関係性"
emoji: "🫠"

type: "tech"
topics: ["React"]
published: true
publication_name: "frontendflat"
---

Reactではコンポーネントの状態をそのコンポーネント内部で保持するわけではないので、「この位置にあるコンポーネントの状態はこれ」のように、コンポーネントとその状態を**UI Treeにおける位置**により紐づけます。

これは言い換えると 

- **コンポーネントの位置が変わらなければ状態は保持される**
- **コンポーネントの位置が変われば状態がリセットされる**

のように言うことができます。 

この挙動は（個人的に）なかなか初見殺しな挙動だと思うので、いくつかの例を見ながら、状態が保持またはリセットされるタイミングについて説明します。

:::message
初見殺しな挙動と書きましたが、今回の内容はドキュメントにしっかりと記載されているので合わせて参照してみてください。
:::

https://react.dev/learn/preserving-and-resetting-state

## コンポーネントの位置が変わらなければ状態は保持される

こちらの例では、チェックボックスの状態により二つのカウンターが切り替わりますが、カウンターの状態はリセットされません。
どういうことかというと、ReactはあくまでJSXから生成されたUI Treeにおける位置を参照しているので、`isFancy`という条件が変わっても切り替わったカウンターは同じ位置に配置されるので同じコンポーネントと見なされるということです。

:::details デモ
@[codesandbox](https://codesandbox.io/embed/little-flower-nv8q9x?fontsize=14&hidenavigation=1&module=%2FApp.js&theme=dark)
:::

```jsx
import { useState } from "react";

export default function App() {
  const [isFancy, setIsFancy] = useState(false);
  return (
    <div>
      {isFancy ? <Counter isFancy={true} /> : <Counter isFancy={false} />}
      <label>
        <input
          type="checkbox"
          checked={isFancy}
          onChange={(e) => setIsFancy(e.target.checked)}
        />
        Use fancy styling
      </label>
    </div>
  );
}

function Counter({ isFancy }) {
  const [score, setScore] = useState(0);
  let className = "counter";
  if (isFancy) {
    className += " fancy";
  }
  return (
    <div className={className}>
      <h1>{score}</h1>
      <button onClick={() => setScore(score + 1)}>Add one</button>
    </div>
  );
}
```

ちなみに以下のような例では、条件が変わるごとに`Counter`コンポーネントが破棄されるので状態はリセットされます。

```jsx
{isPaused ? (
  <p>See you later!</p>
) : (
  <Counter />
)}
```

```jsx
{isFancy ? (
  <div>
    <Counter isFancy={true} />
  </div>
) : (
  <section>
    <Counter isFancy={false} />
  </section>
)}
```

あくまで、同じ位置でUI Treeの構造が一致するときに限り、再レンダリング間で状態が保持されると理解しましょう。

## 同じ位置のまま状態をリセットする

前述したとおり、Reactは**コンポーネントの位置が変わらなければ状態は保持される**という挙動を取りますが、`key`を使うことで同じ位置のまま状態をリセットすることができます。

リストレンダリングのときに使用する`key`ですが、`key`をそれぞれ指定することで別のコンポーネントとして区別することができます。
以下の例では、二つの`Counter`コンポーネントは別のものと見なされ、切り替わるごとにコンポーネントは破棄され、状態がリセットされます。

```jsx
{isFancy ? (
  <Counter key={'fancy'} isFancy={true} />
) : (
  <Counter key={'default'} isFancy={false} />
)}
```

ちなみに、三項演算子ではなく論理積を使うことで、同じ位置でも実質コンポーネントを異なる位置でレンダリングし状態をリセットすることもできます。が、制御フローのわかりやすさ的にも`key`を使った方が良いと思います。
```jsx
{isFancy && <Counter isFancy={true} />}
{!isFancy && <Counter isFancy={false} />}
```

## 最後に

ドキュメントでは、同じ位置のコンポーネントの状態をリセットする簡単なユースケースが二つ紹介されています。ぜひ参照して「どういうときにこのような実装ができそうか？」実装イメージを膨らませてみてください。

https://react.dev/learn/preserving-and-resetting-state#resetting-state-at-the-same-position

https://react.dev/learn/preserving-and-resetting-state#resetting-a-form-with-a-key