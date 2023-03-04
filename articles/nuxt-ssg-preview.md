---
title: "Jamstack な Nuxt.js + microCMS サイトにプレビューページを追加してみた"
emoji: "👀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue", "nuxtjs", "microcms", "jamstack", "ssg"]
published: true
published_at: 2023-03-13 11:00
publication_name: "hacobell_dev"
---

ハコベルシステム開発部の大石貴則です。普段はフロントエンドエンジニアとして物流 DX SaaS プロダクトの開発を行なっています。

この記事では、 Nuxt.js (static mode) + microCMS で構成した Jamstack なサイトにおいて、デプロイ不要でコンテンツをプレビューするページを追加し、執筆者が下書きを即時プレビューできるようにする方法を解説します。

(Vue.js 2, Nuxt.js 2 を使用しています)

## ハコベルサービスサイトの技術構成について

[ハコベルのサービスサイト](https://www.hacobell.com/)は、Nuxt.js の static mode と、ヘッドレスCMSである microCMS を活用した Jamstack 構成で作られています。

https://nuxtjs.org/ja/docs/concepts/static-site-generation/

デプロイ環境において、microCMS の API を呼び出してコンテンツを埋め込んだ静的な HTML と JavaScript を生成します。これらの静的ファイルを CDN に保存し、CDN からユーザーに配信するというアーキテクチャとなっています。

クライアントサイドのアクセスに対し、静的ファイルを CDN から配信するだけで完結します。パフォーマンスの高さや、セキュリティリスクを軽減できる利点があります。

また、静的ファイルにすでにコンテンツが埋め込まれているため、クライアントサイドから microCMS の API を呼び出す必要がなくなります。

## 課題

このアーキテクチャでは、microCMS で管理しているコンテンツの変更がある度にデプロイを行わなければ、クライアントサイドで最新のコンテンツを反映できません。

ハコベルでは、本番環境に新しいコンテンツを反映する前に検証環境へデプロイし、検証環境で関係者が確認後に本番環境へデプロイするというフローを踏んでいます。

しかし、組織の拡大に伴い、お知らせやセミナー情報などコンテンツの更新頻度が増加しています。デプロイするまで検証環境でプレビューを表示できないことは、運用において大きな課題になっていました。

## Nuxt.js の Preview Mode について

このような課題に対して、 Nuxt.js では Preview Mode という機能が用意されています。 microCMS でも紹介されています。

https://nuxtjs.org/ja/docs/features/live-preview/

https://blog.microcms.io/nuxt-preview-mode/

ユーザーから特定のクエリが渡された場合、クライアントサイドで新たにコンテンツを再取得するという手法です。

しかし、この手法では新たなコンテンツを作成しても、ルーティングが生成されておらずページが存在しないので 404 エラーになるという課題があります。

そのため、CDN 側でルーティングを構成する必要があります。将来的なインフラの移行を予定しており、移動コスト軽減の観点から、この手法は採用できませんでした。

## 動的にコンテンツを表示するプレビュー専用ページを作成

そこで、私たちは URL クエリから content ID を取得し、クライアントサイドでコンテンツを取得するプレビュー専用ページを作成することにしました。

mounted を使用した単純な Vue ファイルです。 asyncData を使用してしまうと、 静的ファイル生成時にコンテンツを取得しようとして 404 エラーが発生し失敗してしまいます。

/pages/preview/news.vue

```html
<template>
  <NewsPage v-if="content" :content="content" />
</template>

<script lang="ts">
  import Vue from "vue";
  export default Vue.extend({
    components: { NewsPage },
    data() {
      return {
        content: null as NewsContent | null,
      };
    },
    async mounted() {
      const { id } = this.$route.query;
      const content = await this.$news.get(id);
      this.content = content;
    },
  });
</script>
```

このページにアクセスされたら、クライアントサイドで毎回 mounted が呼び出されます。 URL クエリの content ID を microCMS のプラグインに渡すことで最新のコンテンツが取得されます。

### 本番環境からアクセスできないようにする

プレビューができるページを作成できました。しかし、このファイルを本番環境に公開することは、下記の理由から避ける必要があります。

- microCMS のアクセストークンが露出する
- 下書きのコンテンツが漏洩する
- プレビューページが検索エンジンのクローラーにインデックス付けされてしまう

そのため、本番環境ではページを生成しないように制御します。

nuxt.config.js

```js
const {
  CMS_BASE_URL,
  CMS_API_KEY,
  ENVIRONMENT,
} = process.env

export default {
  target: 'static',  // static モード

  // 検証環境のみ、下書きにアクセス可能な microCMS アクセストークンを
  // クライアントサイドに出荷する
  publicRuntimeConfig: ENVIRONMENT === "development"
  ? {
      CMS_BASE_URL,
      CMS_API_KEY,
    }
  : {},

  generate: {
    // プレビューモードが無効な場合にプレビューページを作らない
    exclude: ENVIRONMENT === "development" ? [] : [/^\\/preview/],
  },
}
```

### microCMS の設定

最後に、コンテンツの執筆者がより容易にプレビューページを開けるよう、microCMS 管理画面の設定を変更します。

「API 設定」→「画面プレビュー」から遷移先 URL を設定します。

![microCMS のスクリーンショット。「API 設定」、「画面プレビュー」の「遷移先 URL」に「https://example.com/preview/news?id={CONTENT_ID}」を設定し、「画面プレビューボタンの表示」を活性にしている](/images/nuxt-ssg-preview/image01.png)

執筆画面に「画面プレビュー」ボタンが追加されました。このボタンをクリックすると、即時にプレビューページが開きます。

![microCMS のスクリーンショット。「お知らせ」の記事を編集する画面に「画面プレビュー」というボタンが表示されている](/images/nuxt-ssg-preview/image02.png)

この検証環境でのプレビューページは、社内環境であれば URL からアクセスが可能なので、デプロイせずとも関係者がサイト上の表示を確認することが可能になりました。

![microCMS のスクリーンショット。先程編集した記事の内容で、ハコベルのサービスサイトサイト上に「お知らせ」が表示されている](/images/nuxt-ssg-preview/image03.png)

## まとめ

プレビューページを新たに作ることで、Jamstack を維持したまま運用の負荷を大きく下げることができました。

## 最後に

私たちは「物流の次を発明する」をミッションに掲げ、荷主と運送会社をつなぐマッチングプラットフォーム事業と、荷主向けのオペレーション DX を支援する SaaS 事業を運営しています。

2022 年 8 月に、ラクスルの事業部からハコベル株式会社として分社化し、第二創業期を迎えました。

そのため、さらなる事業拡大を目指し、エンジニアを始めとした全方向で一緒に働くメンバーを募集しています。面白いフェーズなので、興味がある方はお気軽にご応募ください！

<!-- textlint-disable ja-technical-writing/ja-no-redundant-expression -->いきなりの応募はちょっと...という方は大石個人へ DM やメールして頂ければざっくばらんにお話することも可能です！<!-- textlint-enable -->

ぜひお気軽にご連絡ください。

https://hrmos.co/pages/hacobell
