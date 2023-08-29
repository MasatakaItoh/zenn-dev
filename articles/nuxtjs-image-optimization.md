---
title: "Nuxt.jsにおける画像最適化（画像圧縮とWebP変換）"
emoji: "🎨"
type: "tech"
topics: ["Nuxt.js", "Vue.js", "JavaScript", "WebP"]
published: false
---
## 手順

1. Nuxt Optimized Imagesをインストール
2. nuxt.config.jsに設定を追記
3. WebP変換
4. コンポーネント化（おまけ）

## Nuxt Optimized Imagesをインストール

画像圧縮、WebP変換のために`@aceforth/nuxt-optimized-images`をインストールします。

```
npm install --save-dev @aceforth/nuxt-optimized-images
```

:::message
node >= 10 および nuxt >= 2 が必要です。
:::

画像はそのままでは最適化されません。拡張子ごとに必要な最適化パッケージも合わせてインストールする必要があります。今回は、`jpg`と`png`の圧縮と、`WebP`への変換を行いたいので、下記3つのパッケージをインストールします。

```
npm install --save-dev imagemin-mozjpeg imagemin-pngquant webp-loader
```

`@aceforth/nuxt-optimized-images`にサポートされているパッケージ一覧はこちらです。必要に応じてインストールしてみてください。
https://marquez.co/docs/nuxt-optimized-images#optimization-packages

## nuxt.config.jsに設定を追記

```js:nuxt.config.js
{
  buildModules: [
    '@aceforth/nuxt-optimized-images',
  ],

  optimizedImages: {
    optimizeImages: true
  }
}
```

:::message
nuxtのバージョンが2.9.0未満の場合は、`buildModules`の代わりに`modules`を使用してください。
:::

デフォルトでは開発環境でのビルド時間を短縮するために、画像は本番ビルドのみで最適化されます。開発モードでも最適化を行いたい場合は`optimizeImagesInDev`を有効にしてください。
https://marquez.co/docs/nuxt-optimized-images/configuration#optimizeimagesindev

```js:nuxt.config.js
optimizedImages: {
  optimizeImages: true,
  optimizeImagesInDev: true,
},
```

以上の設定で、`jpg`と`png`の画像最適化が行われました。続いてWebP変換を行います。

## WebP変換

画像パスに`?webp`クエリパラメータを指定することで、`jpg`と`png`画像が自動でWebP形式に変換されます。
余談ですが、WebP対応を行う際は、IE非対応であってもSafariの旧バージョンも考慮し、`<picture>`によるフォールバックの提供を推奨します。（2021/10時点）
https://caniuse.com/webp

```vue
<template>
  <picture>
    <source :srcset="require('~/assets/images/image.jpg?webp')" type="image/webp" />
    <img :src="require('~/assets/images/image.jpg')" />
  </picture>
</template>
```

## コンポーネント化（おまけ）

サイト全体でWebP対応を行う場合の参考程度に。

```vue:BasePicture.vue
<template>
  <picture>
    <!-- media属性による画像出し分け START -->
    <template v-if="sources.length">
      <template v-for="source in sources">
        <source
          :key="source.srcset"
          :srcset="require(`~/assets/images/${source.srcset}?webp`)"
          type="image/webp"
          :media="`(max-width: ${source.media})`"
        />
        <source
          :key="source.srcset"
          :srcset="require(`~/assets/images/${source.srcset}`)"
          :type="source.type"
          :media="`(max-width: ${source.media})`"
        />
      </template>
    </template>
    <!-- media属性による画像出し分け END -->
    <source
      :srcset="require(`~/assets/images/${src}?webp`)"
      type="image/webp"
    />
    <img
      :src="require(`~/assets/images/${src}`)"
      :alt="alt"
      :width="width"
      :height="height"
      :decoding="decoding"
      :loading="loading"
    />
  </picture>
</template>

<script lang="ts">
import Vue from 'vue'

type Source = {
  srcset: string
  type: string
  media: string
}

export default Vue.extend({
  props: {
    sources: {
      type: Array as Vue.PropType<Source[]>,
      required: false,
      default() {
        return []
      },
    },
    src: {
      type: String,
      required: true,
    },
    alt: {
      type: String,
      required: false,
      default: '',
    },
    width: {
      type: Number,
      required: true,
    },
    height: {
      type: Number,
      required: true,
    },
    decoding: {
      type: String,
      required: false,
      default: 'auto',
    },
    loading: {
      type: String,
      required: false,
      default: 'auto',
    },
  },
})
</script>
```

```vue:Parent.vue
<BasePicture
  :sources="[
    {
      srcset: 'sp.jpg',
      type: 'image/jpeg',
      media: '768px',
    },
  ]"
  src="pc.jpg"
  alt="alt値"
  :width="820"
  :height="700"
  decoding="async"
  loading="lazy"
/>
```