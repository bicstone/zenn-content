---
title: "プライベート npm パッケージを CI/CD でインストールできるようにする方法"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["npm", "githubpackages", "githubactions", "circleci", "codebuild"]
published: true
published_at: 2023-02-20 10:00
---

npm private packages や GitHub Packages などのプライベート npm パッケージをインストールする場合の CI/CD の設定方法を解説します。

## 背景

デザインシステムを構築した際、プライベート npm パッケージを GitHub Packages 経由で配信することにしたのですが、CI/CD での依存関係インストールに苦戦したので、各プラットフォームでの設定方法を解説します。 (2023 年 2 月現在)

## ローカル開発環境

意外と忘れてしまうので、まずはローカル環境での設定について解説します。

package.json と同じディレクトリに .npmrc ファイルを追加し、下記のように設定します。

コミットする前に .gitignore に .npmrc を追加しましょう！

```ini
# @bicstone/ の場合は GitHub Packages を参照するように設定
@bicstone:registry=https://npm.pkg.github.com/
# トークンを設定 (GitHub の場合は read:packages 権限の付いた PAT を使用)
//npm.pkg.github.com/:_authToken=accessToken
```

https://docs.npmjs.com/cli/v9/configuring-npm/npmrc

## GitHub Actions

`actions/setup-node` アクションで、 `registry-url` と `scope` を設定します。

`npm ci` や `yarn install` する際には `NODE_AUTH_TOKEN` を環境変数として渡します。

環境変数を渡す際は GitHub の Environment Secret を使用するようにしましょう。

```yml
steps:
  - uses: actions/checkout@v3
  - uses: actions/setup-node@v3
    with:
      registry-url: "https://npm.pkg.github.com"
      scope: "@bicstone"
  - run: npm ci
    env:
      NODE_AUTH_TOKEN: ${{ secrets.NODE_AUTH_TOKEN }}
```

https://github.com/actions/setup-node/blob/main/docs/advanced-usage.md#use-private-packages

https://docs.github.com/ja/actions/security-guides/encrypted-secrets

## CircleCI

.npmrc を作成することで設定します。

依存関係インストール後、不要になった .npmrc を削除する処理を入れておくようにしましょう。

```yaml
steps:
  - checkout
  - run:
      name: Set npm config
      command: |
        npm config set "@bicstone:registry" "https://npm.pkg.github.com/"
        echo "//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}" >> ~/.npmrc
  - run: npm ci
  - run:
      name: Remove npm config
      command: rm ~/.npmrc
```

https://circleci.com/blog/publishing-npm-packages-using-circleci-2-0/

## CodeBuild (Docker)

通常の変数を使用してしまうとコンテナに保持されてしまいます。 docker history などを用いることでアクセストークンを入手できてしまうので危険です。

代わりに secret 引数を使用します。

Dockerfile

```dockerfile
RUN --mount=type=secret,id=npmrc,dst=.npmrc npm ci
```

```yaml
version: 0.2
env:
  variables:
    DOCKER_BUILDKIT: 1
phases:
  pre_build:
    commands:
      - echo "@bicstone:registry=https://npm.pkg.github.com/" > .npmrc && echo "//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}" >> .npmrc
  build:
    commands:
      - docker build -t $REPOSITORY_URI:$ENV_NAME --secret id=npmrc,src=.npmrc .
```

https://docs.docker.com/engine/reference/builder/#run---mounttypesecret

## まとめ

プライベート npm パッケージを CI/CD でインストールできるようにする方法について解説しました。

マイクロフロントエンドやデザインシステムを適用する場面で必要な機会が増えてきそうです。

なお、有効期限のないアクセストークンを扱う以上、セキュリティのリスクがあります。日々ベストプラクティスは変わっていくので、 1 年に 1 回くらい見直しが必要です。
