---
title: "5 分から 1 分に！ yarn install の時間を劇的に短縮した話"
emoji: "⌛"
type: "tech"
topics: ["frontend", "npm", "yarn", "nodesass", "sass"]
published: true
published_at: 2023-03-23 11:00
---

yarn install (フロントエンドの依存ライブラリインストール) の時間が長くなってしまった原因の調査と、解決した記録です。

(今回は yarn v1 を使用していますが、npm では `yarn why` の代わりに `npm ls` を用いることで同様に調査可能です)

## 背景

とあるプロダクトで、いつの間にか yarn install (フロントエンドの依存ライブラリインストール) の時間に 5 分ほどかかるようになってしまいました。

一見はローカル環境での初回インストール時しか影響がなく感じるので気にしなくていいやと思いがちですが、よく考えると結構大きな問題です。

- ローカル環境の構築に時間がかかることにより、オンボーディングの手間が増加
- dependencies 更新時に各ローカル環境で yarn install が必要なため、開発効率の低下
- CI の結果がすぐに分からないことにより、レビューの速度が低下
- CD の時間がかかることにより、マージからリリースまでの速度が低下
- CI / CD のコストが増大

そこで、時間をかけて調査と修正をすることにしました。

## インストールに時間がかかるライブラリの特定

すべてのライブラリを再インストールするため、 node_modules をまるごと削除します。

```bash
rm -rf node_modules
```

そして、再度インストールします。

```bash
yarn install --immutable
```

インストール中はターミナルを目視で眺めます。

そして、 `Building fresh packages...` というステータスになったら注視します。今回のプロダクトでは下記の画面の状態で 2 分かかっていました。

![yarn でインストール中の画面。後述のコードブロックの内容が表示されている。](/images/yarn-install-5-minutes/image01.png)

```bash
[4/4] Building fresh packages...
[-/11] waiting...
[2/11] fsevents
[-/11] waiting...
[-/11] waiting...
[8/11] node-sass
```

その後、下記の画面になり、さらに 2 分ほどかかっていました。

```bash
[4/4] Building fresh packages...
[-/11] waiting...
[-/11] waiting...
[-/11] waiting...
[-/11] waiting...
[10/11] grpc
```

以上の結果から、 `node-sass` と `grpc` のビルドに 4 分かかっていることがわかります。

verbose モードにしてログを保存して分析することも最初は考えていたのですが、インストールに時間のかかるライブラリは目視で十分確認できました。

`node-sass` については `node-sass@4.13.1` が dependencies に指定されていました。

しかし、 `grpc` は指定していません。そのため、どのライブラリの依存関係になっているのか調べました。

```bash
yarn why grpc

yarn why v1.22.19
[1/4] Why do we have the module "grpc"...?
[2/4] Initialising dependency graph...
[3/4] Finding dependency...
[4/4] Calculating file sizes...
=> Found "grpc@1.24.2"
info Reasons this module exists
   - "firebase#@firebase#firestore" depends on it
   - Hoisted from "firebase#@firebase#firestore#grpc"
Done in 0.44s.
```

`firebase` が依存していることがわかりますね。たしかに `firebase@7.8.1` はインストールしています。

## 該当ライブラリの README の確認

<!-- textlint-disable ja-technical-writing/ja-no-weak-phrase -->

原因は `node-sass@4.13.1` と `firebase@7.8.1` であることがわかったところで、ライブラリ該当のバージョンの README を見に行ってみましょう。何かヒントがあるかもしれません。

<!-- textlint-enable -->

### node-sass

まずは node-sass を調べてみます。

当時の [README](https://github.com/sass/node-sass/tree/v4.13.1) を参照すると、下記のように記載があります。

> Supported Node.js versions vary by release, please consult the releases page.

リリースページを要確認とのことで、[リリースページ](https://github.com/sass/node-sass/releases/tag/v4.13.1)を見てみると原因がわかりました。つい最近 Node を 14 系にアップデートしたばかりだったのです。

<!-- textlint-disable ja-technical-writing/max-comma -->

> Supported Environments
> Node 0.10, 0.12, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13

<!-- textlint-disable -->

> Install runs only two Mocha tests to see if your machine can use the pre-built LibSass which will save some time during install. If any tests fail it will build from source.

通常はビルド済みのものをコピーするだけですが、今回サポート外の Node バージョンだったため、毎回ソースコードからビルドしていたことがわかりました。

今回、影響を最小限度にするべく、Node 14 をサポートしている最も近いバージョン 4.14.1 にアップデートすることにしました。プロダクトに影響のある差分がなかったので、もっと早く動いていれば良かったと反省しました。

https://github.com/sass/node-sass/compare/v4.13.1...v4.14.1

### firebase

次に grpc に依存している firebase のリリースノートを調べてみます。

7.14.0 に次のような[リリースノート](https://firebase.google.com/support/release-notes/js#version_7140_-_april_9_2020)がありました。

> Replaced grpc with @grpc/grpc-js in the Node.js builds. As a result, the minimum supported NodeJS version is now 8.13.0.

7.14.0 未満は `grpc` のソースコードを毎回ビルドしていましたが、7.14.0 以降はビルド済みの `@grpc/grpc-js` に依存するようになったようです。

こちらも、7.14.0 にアップデートすることにしました。

## まとめ

yarn install の時間が 5 分から 1 分に短縮することができました 🎉

特に CI の速度が速くなったことによる恩恵が大きかったです。開発効率を大きく向上させることができました。

node-sass から dart-sass の移行など、他にも改善が必要なため着々と進めていく所存です。
