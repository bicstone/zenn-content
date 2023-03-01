---
title: "普段は https を使って git clone しつつ submodule だけ ssh で接続する方法"
emoji: "🎭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["git", "github", "ssh", "submodule"]
published: true
published_at: 2023-03-02 11:00
---

Git において普段は https で接続しつつも、特定の submodule だけ ssh で接続する方法を解説します。

## 背景

Git で Clone する方法として https と SSH がありますが皆さんはどちらをお使いでしょうか。

私は普段 https で接続しています。最近は GitHub において Fine-grained personal access tokens という新機能が登場し、Personal access token がアクセスできるリポジトリと機能を絞れるようになったので、SSH 接続と比較して安全性が高まっています。

https://github.blog/2022-10-18-introducing-fine-grained-personal-access-tokens-for-github/

一方、執筆時点では SSH 接続をされている方がかなり多いため、 submodule の参照先が SSH に設定されている場面がよくあります。

普段は https 接続を使いつつ、特定のリポジトリの submodule のみ SSH で接続する方法を解説します。

(拙ポートフォリオサイトの[リポジトリ](https://github.com/bicstone/portfolio)で使用している submodule を例に紹介します。)

## 鍵を生成

まずは、特定のリポジトリ用の SSH 鍵を生成します。GitHub のドキュメントにとてもわかりやすいドキュメントがあるので真似しながら設定してみます。

(恥ずかしながら EdDSA というアルゴリズムを使ったことがなかったのですが、ドキュメントで勧められていたので使ってみます。)

https://docs.github.com/ja/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#generating-a-new-ssh-key

```bash
ssh-keygen -t ed25519 -C "github-bicstone-portfolio-submodule" -f "github-bicstone-portfolio-submodule"
Generating public/private ed25519 key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in github-bicstone-portfolio-submodule.
Your public key has been saved in github-bicstone-portfolio-submodule.pub.
The key fingerprint is:
(省略)
```

EdDSA アルゴリズムを用いると噂通り公開鍵長が短くなっていて驚きました。

## 公開鍵を登録

下記を参考に生成された公開鍵を GitHub に登録します。今回の用途だと submodule からプッシュすることはないので "Allow write access" のチェックは外しておきます。

https://docs.github.com/ja/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent

## Git の設定

該当のリポジトリを submodule に追加したいリポジトリを開き、下記のコマンドを実行します。

git -c とは、一時的に config を設定した上で実行するオプションです。親のリポジトリでは SSH 接続をしないので、 submodule を追加するタイミングのみ特定の SSH 鍵を参照するように設定します。

```bash
git -c core.sshCommand="ssh -i ~/.ssh/github-bicstone-portfolio-submodule" submodule add git@github.com:bicstone/portfolio-static.git static
```

無事追加されたら、submodule 内に入ります。

```bash
cd static
```

そして、submodule 内で SSH 鍵の参照設定を恒久化します。

```bash
git config --local core.sshCommand "ssh -i ~/.ssh/github-bicstone-portfolio-submodule"
```

```bash
cd ..
git submodule update --init --recursive
```

以上で、メインのリポジトリでは https を用いた接続、submodule では特定の SSH 鍵を用いた SSH 接続という使い分けを実現できました。

追加された submodule を clone してきた際に初期化する時も同様に行えます。

```bash
git -c core.sshCommand="ssh -i ~/.ssh/github-bicstone-portfolio-submodule" submodule update --init --recursive
```

```bash
cd static
```

```bash
git config --local core.sshCommand "ssh -i ~/.ssh/github-bicstone-portfolio-submodule"
```

## まとめ

submodule は使わないに越したことはないですが、やはり便利なので使ってしまいますね。みなさんも Fine-grained personal access tokens に移行しましょう。

大事なのでもう一度貼っておきます (布教)

https://github.blog/2022-10-18-introducing-fine-grained-personal-access-tokens-for-github/
