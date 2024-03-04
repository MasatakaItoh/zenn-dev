---
title: "Tanstack Queryで実装する楽観的更新"
emoji: "🐷"
type: "tech"
topics: ["React","TanstackQuery"]
published: true
publication_name: "frontendflat"
---

Tanstack Queryにはv5から楽観的更新（Optimistic Updates）を手軽に実装できる新たな方法が追加されています。
楽観的更新を実装するUIが一つ（ないし少ない）の場合は特に有効な手法なので、今回はその方法を紹介します。

## `variables`を使った楽観的更新の実装

以下はTODOを追加するmutationの例です。

```tsx
const { isPending, variables, mutate, isError } = useMutation({
  mutationFn: (newTodo: string) => axios.post('/api/data', { text: newTodo }),
  onSettled: async () => {
    return await queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
});

return (
  <ul>
    {todoQuery.items.map((todo) => (
      <li key={todo.id}>{todo.text}</li>
    ))}
    {isPending && <li style={{ opacity: 0.5 }}>{variables}</li>}
  </ul>
);
```

`mutate`を実行すると即座に、リスト最後尾に半透明のアイテムが追加され、mutationが完了すると通常のアイテムとして表示されるという挙動を取ります。

具体的には以下の通り

1. `mutate`を実行すると、mutationの実行中を表す`isPending`は`true`になる。
2. `useMutation`から返される`variables`には`mutationFn`の引数が格納されているため、リスト最後尾に新たに追加されるTODOアイテムが表示される。
3. mutationが完了（`onSettled`）すると、`invalidateQueries`を実行しTODOをリフェッチする。`await`しているのでリフェッチが終わるまで`isPending`は`true`のままで2は表示され続ける。
4. リフェッチが終わると2は消え、追加されたアイテムが含まれた最新の状態でリストが再レンダリングされる。

`invalidateQueries`は`onSettled`で実行されるので、mutationが成功しても失敗してもリフェッチされます。これによりmutationが失敗したときは2で追加されたアイテムは4で消えてくれます。

もしmutateが失敗したときのリトライ機能を実装したければ、例えば以下のように実装することができます。mutationがエラーになっても`variables`はクリアされません。

```tsx
{
  isError && (
    <li style={{ color: 'red' }}>
      {variables}
      <button onClick={() => mutate(variables)}>Retry</button>
    </li>
  )
}
```

また、mutationの`variables`に他のコンポーネントからアクセスしたい場合は`useMutationState`を使用することができます。これにより`mutate`を実行したコンポーネント以外で`variables`を参照し、楽観的更新を実装することができます。

```tsx
const { mutate } = useMutation({
  mutationFn: (newTodo: string) => axios.post('/api/data', { text: newTodo }),
  onSettled: () => queryClient.invalidateQueries({ queryKey: ['todos'] }),
  mutationKey: ['addTodo'],
})

// 他のコンポーネントでvariablesを参照できる
const variables = useMutationState<string>({
  filters: { mutationKey: ['addTodo'], status: 'pending' },
  select: (mutation) => mutation.state.variables,
})
```

## さいごに

今回は`variables`を使ってUIを操作することで楽観的更新を実装する方法を紹介しました。

冒頭にも記載しましたが、この手法は楽観的更新を実装するUIが一つ（ないし少ない）の場合に有効です。もし複数UIを操作する必要がある場合は、従来通りキャッシュを直接操作する方法を使うことをおすすめします。

https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates#via-the-cache

## 参考

https://tanstack.com/query/latest/docs/framework/react/guides/migrating-to-v5#simplified-optimistic-updates

https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates