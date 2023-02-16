---
title: "Flex item の overflow: hidden が効かない問題の対処方法"
emoji: "🥲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["html", "css", "flex", "flexbox", "w3c"]
published: true
published_at: 2023-02-18 17:00
---

Flex アイテムで overflow: hidden を設定しているのに Flex アイテムが縮小されずに文字列がはみ出してしまう問題の原因と対処方法を解説します。

## 背景

皆さんは Flex を使っていますか? 私はよく使っています。

便利な Flex ですが、仕様が複雑なため、理解していないと予期しないデザイン崩れが起こってしまうことがあります。

そのうちの 1 つが、長い文字列で Flex アイテムが縮小されない問題です。

例えば下記をご覧ください。

```css
.grid {
  display: flex;
  width: 200px; /* 横幅を200pxに固定 */
}

.item {
  padding: 20px;
  border: 1px solid #000;
}

.text {
  overflow: hidden; /* はみ出さない */
  white-space: nowrap; /* 改行しない */
  text-overflow: ellipsis; /* はみ出す部分は三点リーダーにする */
}

.item:nth-child(1) {
  flex-grow: 1;
  flex-shrink: 1;
}

.item:nth-child(2) {
  flex-shrink: 0;
}
```

```html
<div class="grid">
  <div class="item">
    <p class="text">
      Llanfairpwllgwyngyllgogerychwyrndrobwllllantysiliogogogoch
    </p>
  </div>
  <div class="item">
    <p class="text">Item</p>
  </div>
</div>
```

@[codesandbox](https://codesandbox.io/embed/flex-item-overflow-hidden-439rsu?&theme=light&view=preview)

世界一長い(執筆時現在)村名である「Llanfairpwllgwyngyll-gogerychwyrndrobwll-llantysilio-gogogoch」は本来 3 点リーダーで省略され、Grid 全体は 200 px に収まってほしいのですが、伸びてしまっています。

## 原因

この挙動は一見バグのように見えます。しかし、実は Flex の仕様通りの正しい動作をしています。

W3C Candidate Recommendation, 19 November 2018 の仕様を見てみましょう。 (§7.1.1. Basic Values of flex)

https://www.w3.org/TR/2018/CR-css-flexbox-1-20181119/#flex-common

> By default, flex items won’t shrink below their minimum content size (the length of the longest word or fixed-size element). To change this, set the min-width or min-height property. (See §4.5 Automatic Minimum Size of Flex Items.)

意訳すると下記のとおりです。

> デフォルトでは、Flex Item は最小コンテンツサイズ (最も長い単語または固定サイズの要素の長さ) 未満には縮小しない。
> これを変更するには、 `min-width` または `min-height` プロパティを設定する。 (§4.5 Automatic Minimum Size of Flex Items を参照)

また、 §4.5 Automatic Minimum Size of Flex Items に 5000 文字ぐらいで最小サイズの算出方法についての仕様が書いてあるので、お時間あれば読んでみてください。私は理解するのに 30 分かかりました。

https://www.w3.org/TR/2018/CR-css-flexbox-1-20181119/#min-size-auto

縮められない理由は下記のフローが起因しています。

1. Flex item に `min-width` が設定されていないので、 `min-width: auto` が設定される
1. `auto` の幅を算出するために、コンテンツの幅を算出する
1. コンテンツの width を算出したら Xpx だった
1. `min-width: Xpx` と見なす
1. よって width が期待しているよりも広くなる

## 対処方法

縮められない理由として、 `min-with: auto` が原因であることがわかりました。

W3C のドキュメントでもオススメされていた通り、先程の例に `min-width: 0` を設定してみます。

```diff
 .item {
   padding: 20px;
   border: 1px solid #000;
+  min-width: 0;
 }
```

@[codesandbox](https://codesandbox.io/embed/flex-item-overflow-hidden-with-min-width-0-0mbumq?&theme=light&view=preview)

これで、無事に世界一長い村名が省略されました。

## まとめ

- Flex item が縮小されなくなった原因がわかりました
- 思ったどおりの挙動せずに困った時は、軽く W3C のドキュメントに目を通すと解決することが多いです
- Llanfairpwllgwyngyll-gogerychwyrndrobwll-llantysilio-gogogoch を知るきっかけとなった動画 [問題より答えが長い！極限早押しクイズ(Youtube)](https://www.youtube.com/watch?v=0pPriJMqPEU)
