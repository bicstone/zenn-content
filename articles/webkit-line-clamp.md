---
title: "複数行かつ 溢れたら 3 点リーダーを付ける処理を CSS だけで実現したい"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hrml", "css", "w3c", "lineclamp", "webkit"]
published: true
published_at: 2023-02-19 10:00
---

複数行かつ溢れたら 3 点リーダーを付ける処理を CSS だけで実現する方法を解説します。

## 背景

`overflow: hidden`, `whiteSpace: "normal"` と `textOverflow: ellipsis` を組み合わせると、溢れたら 3 点リーダーを付けることができます。

しかし、この方法は 1 行の場合でしか使用できません。意外と 2 行で省略したくなる場面が多いのですが、CSS だけで実現するのは難しく JS で頑張って実装していました。

## `-webkit-line-clamp` vs `line-clamp`

それを踏まえ、W3C では `line-clamp` プロパティが提案されています。一方、先行して Webkit 系で導入された `-webkit-line-clamp` というプロパティがあり、多くのサイトで紹介されています。

https://developer.mozilla.org/en-US/docs/Web/CSS/-webkit-line-clamp

W3C の仕様はまだ作業草案 (Working Draft) であり、ブラウザにはあまり実装されていません。また、仕様はまだまだ変わる可能性があるので、執筆時現在 (2023 年) は `-webkit-line-clamp` を使用してみます。ちなみに `-webkit-line-clamp` も後方互換性のため W3C の草案に正式に定義されているため、しばらくは安心して使用できます。

https://www.w3.org/TR/2022/WD-css-overflow-3-20221231/

`-webkit-line-clamp` は非常に多くのブラウザでサポートされています。一方、少し古いブラウザではサポートされていないため注意が必要です。

もしサポートされていないブラウザで開かれた場合、長い文字列が省略されないので大変なことになってしまいます。

https://caniuse.com/mdn-css_properties_-webkit-line-clamp

## サンプル

バンコクの儀式的正式名称を 3 行で表示するサンプルです。

```css
div {
  width: 300px;
  overflow: hidden;
  display: -webkit-box;
  text-overflow: ellipsis;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 3;
  /* ブラウザがサポートしていない場合のフェールセーフ */
  max-height: 24px;
}
```

```html
<div>
  クルンテープ・マハーナコーン・アモーンラッタナコーシン・マヒンタラーユッタヤー・マハーディロック・ポップ・ノッパラット・ラーチャタニーブリーロム・ウドムラーチャニウェートマハーサターン・アモーンピマーン・アワターンサティット・サッカタッティヤウィサヌカムプラシット
</div>
```

@[codesandbox](https://codesandbox.io/embed/webkit-line-clamp-60uwxy?&theme=light&view=preview)

## まとめ

- ベンダープレフィックスが付いているプロパティが後方互換性のため仕様に登録されることを初めて知りました
- 意外とサポートされているブラウザが多く、安心して使用できそうです
- バンコクの儀式的正式名称を知る切っ掛けとなった動画 [伊沢 vs 須貝（こっそり山本と通話中）(Youtube)](https://www.youtube.com/watch?v=J8Dv9cR74-I)
