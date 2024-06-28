---
title: "Conformで作るWeb標準なフォーム"
emoji: "🎃"
type: "tech"
topics: ["Conform","React","Remix","Nextjs"]
published: true
publication_name: "frontendflat"
---

## Conformの概要

Conformは、RemixやNext.jsのようなフレームワークをサポートしており、クライアントサイドとサーバーサイドのフォームバリデーションを同じ記述で書くことができるライブラリです。RemixやServer Actionsの台頭により、ZennでもConformを紹介する記事がいくつか投稿されています。

https://zenn.dev/chimame/articles/b10d7e5f5011f9

https://zenn.dev/akfm/articles/server-actions-with-conform

今回は「Conformを使うとWeb標準なフォームを作ることになる」という別の観点でConformについて紹介します。ReactではJavaScriptでフォームの状態を管理することが当たり前になっており、仮にフォームのマークアップが正しくなかったとしても入力〜バリデーション〜送信まで機能を作ることができてしまいます。

SPA時代からエンジニアになった自分のようなエンジニアは、正しいフォームのマークアップを知らないことが多いかもしれません。Conformが生成するコードを見ることで、Web標準なフォームのマークアップを学ぶことができます。

## ヘルパーを使ってコーディングする

Conformには、[getFormProps](https://conform.guide/api/react/getFormProps)や[getInputProps](https://conform.guide/api/react/getInputProps)など、フォームやフォームコントロールのpropsを生成してくれるヘルパーが用意されています。

```tsx
import { useForm, getFormProps, getInputProps } from "@conform-to/react";
import { parseWithZod, getZodConstraint } from "@conform-to/zod";
import { Form } from "@remix-run/react";
import { z } from "zod";

const schema = z.object({
  email: z
    .string({ required_error: 'Email is required' })
    .email('Email is invalid'),
});

export default function SomethingForm() {
  const [form, fields] = useForm({
    constraint: getZodConstraint(schema), 
    onValidate: ({ formData }) => parseWithZod(formData, { schema }),
  });
  
  return (
    <Form method="post" {...getFormProps(form)}>
      <div>
        <label htmlFor={fields.email.id}>Email</label>
        <input {...getInputProps(fields.email, {type: 'email'})} />
        <div id={fields.email.errorId}>{fields.email.errors}</div>
      </div>
    </Form>
  );
}
```

`getFormProps`と`getInputProps`を使ってマークアップすることで、このような出力を得ることができます。

```html
<form method="post" action="/?index" id=":R35:" novalidate="">
  <div>
    <label for=":R35:-email">Email</label>
    <input required="" id=":R35:-email" name="email" form=":R35:" type="email" aria-invalid="true" aria-describedby=":R35:-email-error">
    <div id=":R35:-email-error">Email is required</div>
  </div>
</form>
```

以下のような観点で正しいマークアップがされていると言えるでしょう。 

- フォームコントロールを識別する`name`属性が設定されている
- form要素の`id`属性とinput要素の`form`属性により両者が関連付けられている
- label要素の`for`属性とinput要素の`id`属性により両者が関連付けられている
- input要素の`aria-invalid`属性によりエラー状態であることがわかる
- input要素の`aria-describedby`属性とエラーメッセージの`id`属性により両者が関連付けられている

Confromは自動的に一意な`id`を割り振ってくれるので、`id`の管理が不要な点も嬉しいです。

アクセシビリティに配慮したフォームを自前でコーディングしようとするとボイラープレートが大きくなりがちです。UIライブラリを使わずにネイティブなHTMLでコーディングするときは、Conformのヘルバーを使うことでボイラープレートを最小にしつつアクセシビリティに配慮したマークアップを生成することができます。私は最近[shadcn/ui](https://ui.shadcn.com/)でコーディングするときにConformを使いましたが、非常に体験がよかったです。

## hiddenなフォームコントロールを使う

Conformで実装するフォームでは、フォームコントロールの実態が存在しないデータを保持することはできません。UIとして表示する必要はないが、FormDataとして送信したいデータがある場合は、Web標準に習ってhiddenなフォームコントロールでマークアップしましょう。

```tsx
const [form, fields] = useForm({
  defaultValue: { hiddenData: 'hidden data' },
});

return (
  <input {...getInputProps(fields.hiddenData, {type: 'hidden'})} />
);
```


## 構造化したFormDataを扱う

Web標準なフォームで配列やオブジェクトなど構造化されたデータを扱う場合、フォームコントロールの`name`属性によりそれを表現します。

```html
<form>
  <input name="todos[0].title" value="Todo 1 value">
  <input name="todos[1].title" value="Todo 2 value">
  <input type="submit">
</form>
```

このフォームを送信すると、以下のようなデータ構造で受け取ることになります。

```json
{
  "todos": [
    {"title": "Todo 1 value"},
    {"title": "Todo 2 value"}
  ]
}
```

-----

Conformでは、`getFieldList`と`getFieldset`を使うことで、構造化されたデータを扱うことができます。

https://conform.guide/complex-structures

```tsx
import { useForm, getFormProps, getInputProps } from "@conform-to/react";
import { parseWithZod, getZodConstraint } from "@conform-to/zod";
import { Form } from "@remix-run/react";
import { z } from "zod";

const schema = z.object({
  todos: z.array(z.object({ title: z.string(), notes: z.string() })),
});

export default function SomethingForm() {
  const [form, fields] = useForm({
    constraint: getZodConstraint(schema), 
    onValidate: ({ formData }) => parseWithZod(formData, { schema }),
  });

  // getFieldListを使うことで配列の各要素にアクセスすることができる
  const todos = fields.todos.getFieldList();
  
  return (
    <Form method="post" {...getFormProps(form)}>
      <ul>
        {todos.map((todo) => {
          // getFieldsetを使うことでオブジェクトの子フィールドにアクセスすることができる
          const todoFields = todo.getFieldset();
          return (
            <li key={todo.key}>
              <label htmlFor={todoFields.title.id}>Email</label>
              <input {...getInputProps(todoFields.title, { type: "text" })} />
              <div id={todoFields.title.errorId}>{todoFields.title.errors}</div>

              <label htmlFor={todoFields.notes.id}>Email</label>
              <input {...getInputProps(todoFields.notes, { type: "text" })} />
              <div id={todoFields.notes.errorId}>{todoFields.notes.errors}</div>
            </li>
          );
        })}
      </ul>
    </Form>
  );
}
```

出力はこのようになっており、input要素の`name`属性が`name="todos[0].title"`のように出力されていることがわかります。

```html
<form method="post" action="/?index" id=":R35:" novalidate="">
  <ul>
    <li>
      <label for=":R35:-todos[0].title">Title</label>
      <input required="" id=":R35:-todos[0].title" name="todos[0].title" form=":R35:" type="text" aria-invalid="true" aria-describedby=":R35:-todos[0].title-error">
      <div id=":R35:-todos[0].title-error">Required</div>

      <label for=":R35:-todos[0].notes">Notes</label>
      <input required="" id=":R35:-todos[0].notes" name="todos[0].notes" form=":R35:" type="text" aria-invalid="true" aria-describedby=":R35:-todos[0].notes-error">
      <div id=":R35:-todos[0].notes-error">Required</div>
    </li>
    <!-- ... -->
  </ul>
</form>
```

また、Conformには構造化された値を追加/削除/更新/並び替えする便利なメソッドも提供されています。以下のコードはTODOを追加/削除する例です。

```diff tsx
 <Form method="post" {...getFormProps(form)}>
   <ul>
     {todos.map((todo) => {
       const todoFields = todo.getFieldset();
       return (
         <li key={todo.key}>
           <label htmlFor={todoFields.title.id}>Email</label>
           <input {...getInputProps(todoFields.title, { type: "text" })} />
           <div id={todoFields.title.errorId}>{todoFields.title.errors}</div>

           <label htmlFor={todoFields.notes.id}>Email</label>
           <input {...getInputProps(todoFields.notes, { type: "text" })} />
           <div id={todoFields.notes.errorId}>{todoFields.notes.errors}</div>

+          <button {...form.remove.getButtonProps({ name: fields.todos.name, index })}>
+            Remove todo
+          </button>
         </li>
       );
     })}
   </ul>

+  <button {...form.insert.getButtonProps({ name: fields.todos.name })}>
+    Add todo
+  </button>
 </Form>
```

詳しくは公式ドキュメントに記載があるので確認してみてください。

https://conform.guide/intent-button

## まとめ

以上のようにConformを使うとWeb標準なフォームを作ることができます。配列やオブジェクトなどの構造化されたデータの操作も簡単にできるので、RemixやNext.jsのようなフレームワークを使うときはConformを使ってみると良いかもしれません。