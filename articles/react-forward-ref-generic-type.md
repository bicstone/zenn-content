---
title: "React.forwardRef で TypeScript のジェネリック型が扱えない問題の対処方法"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "typescript", "frontend", "component"]
published: false
publication_name: "hacobell_dev"
---

ハコベルシステム開発部の大石貴則です。普段はフロントエンドエンジニアとして物流 DX SaaS プロダクトの開発を行なっています。

この記事では、`React.forwardRef` で TypeScript のジェネリック型を持った Props を扱うことができないという問題の対処方法について紹介します。

(React 18, TypeScript 4.9 を使用しています)

## 背景

ハコベルでは、デザインシステムの構築を進めています。UI コンポーネントの実装にあたり、ref を転送する必要があったのですが、TypeScript のジェネリック型を持った Props を扱うことができないという問題に直面しました。

例として、下記のようなジェネリック型を持った Props を渡す必要がある Component1 を Component2 から呼び出したい場合を考えます。

```tsx
const Component2 = <T,>(props: Component1Props<T>) => (
  <Component1<T> {...props} />
);
```

ref も渡す必要があったため、`React.forwardRef` を用いて実装しようとしたのですが、Component1Props にジェネリクスを渡すことができません。

```tsx
const Component2 = React.forwardRef<HTMLDivElement, Component1Props>(
  (props, ref) => <Component1 ref={ref} {...props} />
);
```

`React.forwardRef` は下記のように型が定義されており、拡張ができないためです。

```tsx
function forwardRef<T, P = {}>(
  render: ForwardRefRenderFunction<T, P>
): ForwardRefExoticComponent<PropsWithoutRef<P> & RefAttributes<T>>;
```

## 対処方法

この問題に対して 3 つの対処方法があります。それぞれのメリット、デメリットを解説します。

### 1. キャストする

まずはキャストする方法を解説します。

```tsx
const Component2 = React.forwardRef<HTMLDivElement, Component1Props<any>>(
  (props, ref) => <Component1 ref={ref} {...props} />
) as <T>(
  p: Component1Props<T> & { ref?: React.Ref<HTMLDivElement> }
) => JSX.Element;
```

```tsx
<Component2<number> ref={ref} value={2} />
```

プログラムとしてわかりやすく、使う側も理解しやすい利点があります。

一方、変更に弱かったり、Type Error が発生しなくなってしまう可能性があるなど、あまり TypeScript の恩恵を受けられない方法でもあります。

### 2. forwardRef を使用しない

次に forwardRef を使用せず、ref を転送する prop を別途作成する方法です。forwardRef が実装される前は、prop として ref を渡していました。現バージョンでも動作し、[React の公式ドキュメント](https://ja.reactjs.org/docs/refs-and-the-dom.html#exposing-dom-refs-to-parent-components) でも紹介されている方法です。

```tsx
const Component2 = <T,>(
  props: Component1Props<T> & { customRef: Ref<HTMLDivElement> }
): JSX.Element => <Component1<T> ref={customRef} {...props} />;
```

```tsx
<Component2<number> customRef={ref} value={2} />
```

プログラム上では一番シンプルで変更にも強いという大きな利点があります。

一方、ドキュメントや実装を読まないと ref を渡す方法がわからないため、コンポーネントを利用する側で混乱を引き起こす可能性があります。

### 3. forwardRef の型を上書きして型推論を効かせる

次に高階関数推論を活用する方法です。

TypeScript 3.4 から [高階関数推論](https://devblogs.microsoft.com/typescript/announcing-typescript-3-4/#higher-order-type-inference-from-generic-functions) が導入されました。今回渡したいジェネリクスを prop を経由して伝搬させることができます。

```tsx
// @types/react.d.ts
import React from "react";

declare module "react" {
  function forwardRef<T, P = {}>(
    render: (props: P, ref: ForwardedRef<T>) => JSX.Element | null
  ): (props: P & RefAttributes<T>) => JSX.Element | null;
}
```

```tsx
interface Component1Props<T> {
  value: T;
}

const Component2Wrapped = <T, P = {}>(
  props: Component1Props<P>,
  ref: React.ForwardedRef<T>
): JSX.Element => <Component1 ref={ref} {...props} />;

const Component2 = React.forwardRef(Component2Wrapped);
```

```tsx
<Component2 ref={ref} value={2} />
```

[StackOverflow](https://stackoverflow.com/a/58473012) Answered by [ford04](https://stackoverflow.com/users/5669456/ford04) (CC BY SA 4.0)

元々の forwardRef の型と一見変わらなく見えます。しかし、TypeScript のアーキテクトである Anders Hejlsberg さんによると、高次関数型推論が行えるのは単一の呼び出しシグネチャと他のメンバーを持たない純粋関数の場合のみであると説明しています[^1]。

元々の型は純粋関数でないため、高階関数推論が動作していませんでした。この型の置き換えによって、純粋関数になったため高階関数推論が動作するようになります。

この方法のメリットは、1 箇所のみの修正で済むためプログラムが煩雑にならず変更にも強くなります。使用する側も ref prop で渡せるので内部実装を意識する必要がありません。

一方デメリットとして、`declare module` による型定義は、私の過去の経験上トラブルが起きやすく、後々維持していく上で問題になる可能性があります。

## まとめ

<!-- textlint-disable ja-technical-writing/no-doubled-joshi -->`React.forwardRef` ではジェネリック型を持った props を扱うことができない問題があり、3 つの対処方法を紹介しました。<!-- textlint-enable ja-technical-writing/no-doubled-joshi -->

今回、私達は UI コンポーネントライブラリとして提供するため、使用する側の混乱を軽減するために「1. キャストする」方法を用いることにしました。

ref を用いないで実装をするのがベストですが、どうしても使う必要がある場合、ドキュメントを整えた上で「2. forwardRef を使用しない」方法を用いるのが、保守性を考えてベターな方法だと考えています。

## 最後に

ハコベルでは運輸大手セイノーホールディングスとのジョイントベンチャー化に伴い、2022 年 8 月に第二創業期を迎えました。

そのため、エンジニアを始めとして、全方向で一緒に働くメンバーを募集しています。面白いフェーズなので、興味がある方はお気軽にご応募ください！

<!-- textlint-disable ja-technical-writing/ja-no-redundant-expression -->いきなりの応募はちょっと...という方は大石個人へ DM やメールして頂ければざっくばらんにお話することも可能です！<!-- textlint-enable ja-technical-writing/ja-no-redundant-expression -->

ぜひお気軽にご連絡ください。

https://hrmos.co/pages/hacobell

[^1]: [https://github.com/microsoft/TypeScript/issues/30650#issuecomment-486680485](https://github.com/microsoft/TypeScript/issues/30650#issuecomment-486680485)
