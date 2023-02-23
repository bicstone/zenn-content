---
title: "Gatsby でページ読み込み時にデザインがチラつく現象を一発解決するプラグインを作りました"
emoji: "😵‍💫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "gatsby", "plugin", "fouc", "ssg"]
published: true
published_at: 2023-03-09 10:00
---

Gatsby Plugin Fix FOUC は、ページ読み込み時にスタイルが変化してちらつく問題を解決するプラグインを作りました。

## SSG で発生するちらつき (FOUC) について

FOUC - flash of unstyled content とは、主に SSG によって構成されたサイトにおいて発生する問題で、ページ読み込み時にスタイルが変化してちらつく現象のことです。

クライアント側でレンダリングされる本来のスタイルと、生成された HTML に含まれるスタイルが異なるために発生します。

ユーザーがページを読み込んだ直後では生成された HTML に含まれるスタイルが表示されますが、その後クライアント側で給水 (Hydrate) が行われ、本来のスタイルがレンダリングされます。このスタイルの変化により、ページ読み込み時にちらつきが発生します。

:::details ちらつきの例 (注意: gif アニメを使用しています。点滅にご注意ください)

![崩れたスタイルが一瞬表示されている画面](/images/gatsby-plugin-fix-fouc/image01.gif)

:::

## 解決方法

原因療法で考えた場合、本来は実行環境によりスタイルが異なるという実装が問題なので、プログラムの修正が最優先です。

ですが、CSS in JS を採用している場合など、<!-- textlint-disable ja-technical-writing/no-doubled-joshi -->なかなか修正が難しい場合があります<!-- textlint-enable ja-technical-writing/no-doubled-joshi -->。その場合の特効薬として、クライアント側で Gatsby がレンダリングされるまでページ全体を隠すことで、ユーザーにちらつきを表示されないようにするという対症療法で解決する手法を取ってみました。

`<body>` 要素に data 属性を追加し、クライアント側で Gatsby のレンダリングが行われるまでページを非表示 ( `opacity: 0` ) にする CSS を適用することで、ちらつきが表示されないようにします。

このアプローチは、[Google optimize](https://developers.google.com/optimize/) でも採用されていた方法です。CSS で透明度を変更するだけなので再レンダリングが不要で、パフォーマンスやアクセシビリティに影響を与えません。

:::details ビフォーアフターの画面収録 (注意: gif アニメを使用しています。点滅にご注意ください)

| ビフォー                                                                              | アフター                                                                                                                                   |
| ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| ![崩れたスタイルが一瞬表示されている画面](/images/gatsby-plugin-fix-fouc/image01.gif) | ![Gatsby のレンダリングが行われるまでページを隠すことで、崩れたスタイルが表示されていない画面](/images/gatsby-plugin-fix-fouc/image02.gif) |

:::

## トレードオフ

Lighthouse のスコア (First Contentful Paint, Largest Contentful Paint) が低下します。 結果として、SEO に影響を与える可能性があります。 (ただし、Cumulative Layout Shift のスコアは改善します)。

`minWidth` を設定すると、画面を隠さない幅を指定できます。

Gatsby の HTML 生成においては画面の横幅が 0 px と設定されています。そのため、画面の横幅によってデザインが変わる場合、<!-- textlint-disable ja-technical-writing/no-doubled-joshi -->スマートフォンサイズではスタイルの差異が起こらない場合があります<!-- textlint-enable ja-technical-writing/no-doubled-joshi -->。そのような場合に、スマートフォンサイズではページを隠さず、PC サイズではページを隠すというような使い方ができます。

Google はモバイルファーストインデックスを行っているため、クローラーはスマートフォンからアクセスする可能性が高いです。この機能を活用することで、SEO への影響の軽減が可能です。

https://developers.google.com/search/docs/crawling-indexing/mobile/mobile-sites-mobile-first-indexing?hl=ja

## インストール

https://github.com/bicstone/gatsby-plugin-fix-fouc

```bash
yarn add gatsby-plugin-fix-fouc
# or
npm install gatsby-plugin-fix-fouc
```

## 使用方法

プラグインをロードするだけで、デフォルトの設定で動作します。

```js
// gatsby-config.js
module.exports = {
  plugins: [`gatsby-plugin-fix-fouc`],
};
```

## 高度な使用方法

設定の変更が可能です。

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
