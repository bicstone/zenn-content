---
title: "Gatsby でのタイムゾーンを考慮した日時の取り扱い方"
emoji: "🕰"
type: "tech"
topics: ["ssg", "jamstack", "gatsby", "datefns", "react"]
published: true
published_at: 2023-03-09 10:30
---

Gatsby の GraphQL で `formatString` を用いて時刻を扱うとユーザーのタイムゾーンによっては日時が正しく表示されません。タイムゾーンを扱いながら正しい日付を表示する方法を解説します。

(Gatsby は v5 を使用しています)

## 背景

Gatsby 製のポートフォリオサイトの個人ブログでは、作成日時・更新日時を取り扱っていているのですが、タイムゾーンの考慮に苦労しました。

https://github.com/bicstone/portfolio

例えば、下記のようなスキーマを想定してみます。

```graphql
type BlogPost {
  created: DateTime! # "1960-01-01T00:00+00:00"
}

query Query {
  allBlogPost: [BlogPost!]!
}
```

## Gatsby の GraphQL でフォーマットしてみる

まず、Gatsby の GraphQL で DateTime を扱う際、 `formatString` という便利関数が使用できることを知り、使ってみることにしました。

https://www.gatsbyjs.com/docs/graphql-reference/

```graphql
query Query {
  allBlogPost {
    a: created
    b: created(formatString: "YYYY-MM-DD HH:mm:ss")
  }
}
```

<!-- textlint-disable ja-technical-writing/ja-no-weak-phrase -->

この結果、どうなると思いますか？

<!-- textlint-enable -->

このようになります。

```json
{
  "a": "1960-01-01T00:00+09:00",
  "b": "1959-12-31 15:00:00"
}
```

`formatString` は、ビルド環境やユーザー環境にかかわらず、 **UTC** の時間を返します。また、タイムゾーンの指定はできません。

GraphQL で完結するので一見便利そうですが、 UTC で問題がない場合以外には使えませんね。

## タイムゾーン付きで取得し、JS でフォーマットしてみる

そこで、GraphQL にはデータの取得のみ任せて、JS でフォーマットすることにしました。

ライブラリとして date-fns を使います。 (2.29.3 で確認)

https://github.com/date-fns/date-fns

このような便利関数を作ってみました。

formatDateTime.ts

```ts
import { format as formatFn, isValid, parseISO } from "date-fns";

export const formatDateTime = (value: string, format: string): string => {
  const parsedDate = parseISO(value);

  if (!isValid(parsedDate)) return "";

  return formatFn(parsedDate, format);
};
```

ISO 形式の日時を受け取り、ローカルのタイムゾーンで format して返します。

```ts
formatDateTime(post.created, "yyyy/MM/dd HH:mm:ss"); // "1960/01/01 00:00:00"
```

この便利関数を通すことで、無事にユーザーのタイムゾーンに合わせた日時を表示できました。

しかし、ビルド環境とユーザーの閲覧環境でタイムゾーンが異なった場合、ビルドされた HTML と整合性が取れずに hydration error が発生してしまうという問題がありました。

## hydration error を抑制してみる

そこで、まずは hydration error を抑制する方法を考えました。

React では、 `suppressHydrationWarning` を使用することで、 hydration error を抑制できます。

https://beta.reactjs.org/reference/react-dom/client/hydrateRoot#suppressing-unavoidable-hydration-mismatch-errors

ビルドされた HTML では null を入れておき、クライアント側がレンダリングするタイミングでコンポーネントを返すようにします。

```tsx
<div suppressHydrationWarning>
  {typeof window === "undefined" ? null : (
    <time dateTime={post.created}>{createdDate}</time>
  )}
</div>
```

hydration error を抑制できました。しかし、この方法では、クライアントの hydrate が遅れた場合にコンポーネントが表示されないリスクを伴います。

ブログ上部に表示されるコンポーネントだったため、特にちらつきが気になってしまい、この方法は採用できませんでした。

## タイムゾーンを固定してみる

当個人ブログは、JST のタイムゾーンを前提として問題がないブログであるため、最終手段として JST で固定することにしました。

タイムゾーンを固定するライブラリとして date-fns-tz を使います。 (それぞれ 1.3.7 で確認)

https://www.npmjs.com/package/date-fns-tz

先程の便利関数を下記のように書き換えてみました。

formatDateTime.ts

```ts
import { isValid } from "date-fns";
import { format as formatFn, utcToZonedTime } from "date-fns-tz";
import { ja } from "date-fns/locale";

const timeZone = "Asia/Tokyo";

export const formatDateTime = (value: string, format: string): string => {
  const parsedDate = utcToZonedTime(value, timeZone);

  if (!isValid(parsedDate)) return "";

  return formatFn(parsedDate, format, { locale: ja, timeZone });
};
```

この方法では、ビルド環境に関わらず、 JST でフォーマットされた HTML が生成されます。エラー抑制をせずとも hydration error は発生せず、さらに hydrate が遅れてもコンポーネントが表示され、ちらつきが発生しません。

## まとめ

タイムゾーンを持った日時を受け取り、もろもろ処理してクライアントの環境に依存せず、正しい日時を表示することが出来ました。タイムゾーンを持った日時を受け取ったら、何かしらの処理が必要なことを忘れないように気をつけます (自戒)

また、Gatsby のブログを作りたいと考えている方は、私の個人ブログの実装をぜひ参考にしてみてください。

https://github.com/bicstone/portfolio

## (おまけ) タイムゾーンを変更して単体テスト

単体テストでタイムゾーンを固定したい場合、 `process.env.TZ` を変更すれば良いのではと考えてしまいがちなのですが、 node のプロセス実行前にタイムゾーンを設定しなければ動作しないようです。

https://github.com/nodejs/node/issues/3449

そのため、タスクランナーで環境変数 `TZ` を設定した上で jest を呼び出す方法を取る必要があります。

(特定のプラットフォームに依存しないように [cross-env](https://www.npmjs.com/package/cross-env) を使用しています。)

package.json

```json
  "scripts": {
    "test": "jest",
    "test:vietnam": "cross-env TZ=\"Asia/Ho_Chi_Minh\" npm run test"
  },
```
