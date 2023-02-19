---
title: "Gatsby でページ読み込み時にデザインがチラつく現象を一発解決するプラグインを作りました"
emoji: "😵‍💫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "gatsby", "plugin", "fouc", "ssg"]
published: true
published_at: 2023-02-26 10:00
---

Gatsby Plugin Fix FOUC は、ページ読み込み時にスタイルが変化してちらつく問題 (FOUC - flash of unstyled content) を解決するプラグインです。

Gatsby v3 - v5 をサポートしています。

:::details ビフォーアフターの画面収録 (注意: gif アニメを使用しています)

| ビフォー                                                                              | アフター                                                                                                                                   |
| ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| ![崩れたスタイルが一瞬表示されている画面](/images/gatsby-plugin-fix-fouc/image01.gif) | ![Gatsby のレンダリングが行われるまでページを隠すことで、崩れたスタイルが表示されていない画面](/images/gatsby-plugin-fix-fouc/image02.gif) |

:::

https://github.com/bicstone/gatsby-plugin-fix-fouc

## 仕組み

`<body>` 要素に data 属性を追加し、クライアント側で Gatsby のレンダリングが行われるまでページを非表示にすることで、ちらつきが表示されないようにします。

このアプローチは、[Google](https://developers.google.com/optimize/) でも採用されています。再レンダリングが不要で、パフォーマンスやアクセシビリティに影響を与えません。

## トレードオフ

Lighthouse のスコア (First Contentful Paint, Largest Contentful Paint) が低下します。 結果として、SEO に影響を与える可能性があります。 (ただし、Cumulative Layout Shift のスコアは改善します)。

`minWidth` を設定すると、画面を隠さない幅を指定できます。

## インストール

```bash
yarn add gatsby-plugin-fix-fouc
# or
npm install gatsby-plugin-fix-fouc
```

## 使用方法

```js
// gatsby-config.js
module.exports = {
  plugins: [`gatsby-plugin-fix-fouc`],
};
```

## 高度な使用方法

```ts
// gatsby-config.ts
import { breakpoints } from "./src/themes";
import type { GatsbyConfig } from "gatsby";
import type { GatsbyPluginFixFoucRefOptions } from "gatsby-plugin-fix-fouc";
const config: GatsbyConfig = {
  plugins: [
    {
      resolve: `gatsby-plugin-fix-fouc`,
      options: {
        attributeName: "is-loading",
        minWidth: breakpoints.values.sm,
        timeout: 3000,
      } as GatsbyPluginFixFoucRefOptions,
    },
  ],
};
export default config;
```

## 設定

| Property        | Type   | Default                             | Description                                                                                               |
| --------------- | ------ | ----------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `attributeName` | string | `gatsby-plugin-fix-fouc-is-loading` | 追加する data-attribute 名                                                                                |
| `minWidth`      | number | `0`                                 | ページを非表示にする最小幅 (px)。設定しない場合は、幅に関係なく非表示になります                           |
| `timeout`       | number | `4000`                              | Gatsby レンダリングが完了しなかった場合でも、フォールバックとして表示するまでの時間（ミリ秒）を指定します |

## まとめ

もし、便利だと思ったらスター頂ければ幸いです。反響があると、維持するモチベーションに繋がります。

よろしくお願いいたします。

https://github.com/bicstone/gatsby-plugin-fix-fouc
