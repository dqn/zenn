---
title: "TypeScript で union to tuple をするのが難しい理由"
emoji: ""
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: false
---

次のように、union の構成要素を 1 つずつ持つ tuple の型を定義したいときの話です。

```ts
type T1 = UnionToTuple<"a" | "b" | "c">;
// => ["a", "b", "c"];
```

TypeScript で union to tuple をする方法として 2 通りの方法がよく知られていますが、この記事ではそれらの方法が不安定である理由を解説しています。

## なぜ難しいのか

TypeScript のリポジトリに union to tuple の機能を提案する issue があり、そこに理由がよくまとめられたコメントが書かれています。

https://github.com/microsoft/TypeScript/issues/13298#issuecomment-514082646

このコメントでは 2 通りの方法について言及しています。

### 1. 関数のオーバーロードを利用した方法

複数の Function Type Expressions からオーバーロードされた関数の型を作るときは次のようにしてできます。

```ts
type T1 = () => string;
type T2 = () => number;

type T3 = T1 & T2;
```

このとき、Type Inferernce を使って `T3` の返り値の型を取り出すと、最後にオーバーロードした関数の返り値の型である `number` を返します[^1]。

```ts
type T4 = T3 extends () => infer R ? R : never;
// => number
```

この仕様を利用すると、次のようなステップを踏めば union を tuple に変換できそうです。

1. 型 `A | B | C` を `() => A | () => B | () => C` に変換する
2. union から intersection (`() => A & () => B & () => C`) に変換する
3. 先述の方法で型を 1 つだけ取り出し、tuple に追加する
4. 構成要素がなくなるまで「3」を繰り返す

コードにすると次のようになります。

<!-- prettier-ignore -->
```ts
type UnionToIntersection<U> =
  (U extends unknown ? (x: U) => void : never) extends (x: infer I) => void ? I : never;

type LastOf<U> =
  UnionToIntersection<U extends unknown ? () => U : never> extends () => infer R ? R : never;

type UnionToTuple<T, L = LastOf<T>> =
  [T] extends [never] ? [] : [...UnionToTuple<Exclude<T, L>>, L];

type T1 = UnionToTuple<"a" | "b" | "c">;
// => ?
```

`UnionToTuple<"a" | "b" | "c">` の結果は `["a", "b", "c"]` のようになるかもしれません。一見うまくいっているように見えますが、この順序は**コンパイルごとに安定ではありません**。先述の issue で、TypeScript の現在の実装では union の順序は保証していないことと、これを変更することは難しいことが述べられています。

この `UnionToTuple` で得られる tuple 型の要素の順序が崩壊する様子を確認したい場合はこちらのリポジトリを clone して試すことができます（環境によっては再現しないかも）。

https://github.com/dqn/broken-union-to-tuple

### 2. union がとり得るすべての順列を列挙する方法

次のような方法で、union がとり得る順列をすべて列挙した型をつくることができます。

<!-- prettier-ignore -->
```ts
type UnionToTuple<T, Orig = T> =
  [T] extends [never] ? [] : T extends unknown ? [T, ...UnionToTuple<Exclude<Orig, T>>] : never;

type T1 = UnionToTuple<"a" | "b" | "c">;
// => ["a", "b", "c"] | ["a", "c", "b"] | ["b", "a", "c"] | ["b", "c", "a"] | ["c", "a", "b"] | ["c", "b", "a"]

const t1: T1 = ["a", "b", "c"]; // OK
const t2: T1 = ["b", "c", "a"]; // OK
```

この方法は先述のコンパイルごとに要素の順序が異なる可能性があるという問題を解決できているように見えます。しかし、この方法には組合せ爆発を引き起こすという問題があります。TypeScript は、**構成要素数が 1,000,000 以上になることが推測されるとコンパイルできなくなります**[^2]。生成される順列の総数は union の構成要素数が 10 のときに 1,000,000 を超える（10! = 3,628,800）ので、構成要素数が 9 以下の union でしか使うことができません。9 以下であっても language server やコンパイラには大きな負荷がかかるので使うべきではないでしょう。

<!-- prettier-ignore -->
```ts
type T1 = UnionToTuple<"a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" | "i" | "j">;
// ERROR: Expression produces a union type that is too complex to represent.
// 10! (= 3,628,800) 通り
// 構成要素数が 1,000,000 以上になることが推定されたのでエラー
```

## おわりに

本当に tuple へ変換する必要があるのかまず検討しましょう。順序を持つ必要がない場合は、少し冗長ですが下記のような方法で対処できる場合があります。

```ts
function getValuesOf<T extends string>(values: { [K in T]: K }): T[] {
  return Object.values(values);
}

type Foo = "a" | "b" | "c";

// OK
const values1 = getValuesOf<Foo>({ a: "a", b: "b", c: "c" });
// => ("a" | "b" | "c")[]

// ERROR
const values2 = getValuesOf<Foo>({ a: "a", b: "b" });
const values3 = getValuesOf<Foo>({ a: "a", b: "b", c: "c", d: "d" });
const values4 = getValuesOf<Foo>({ a: "a", b: "b", c: "b" });
```

## おまけ

とはいえ、どうしても tuple へ変換したい場合はあるかもしれないので筆者が思う最善の方法を紹介します。コンパイラに tuple への変換を任せることはやめて、明示的に tuple の型を書くことにします。ただし、その tuple がすべての要素を 1 つずつ持つことはコンパイラに確認させます。

```ts
type Foo = "a" | "b" | "c";

type T1 = UnionToTuple<Foo, ["a", "b", "c"]>;
// => ["a", "b", "c"]

type T2 = UnionToTuple<Foo, ["a", "b"]>;
// => { __missing: "c" }

type T3 = UnionToTuple<Foo, ["a", "b", "c", "d", "e"]>;
// => { __extra: "d" | "e" }

type T4 = UnionToTuple<Foo, ["a", "d"]>;
// => { __extra: "d"; __missing: "b" | "c"　}

type T5 = UnionToTuple<Foo, ["a", "b", "c", "c"]>;
// => { __length: "expected 3, actual 4" }
```

型と値で 2 回 tuple を書くのが煩わしい場合は、次のような関数を定義する方法が使えます。

```ts
function makeTupleOfUnion<T>(): <U extends readonly unknown[]>(
  tuple: UnionToTuple<T, U>,
) => typeof tuple {
  return (tuple) => tuple;
}

const t1 = makeTupleOfUnion<Foo>()(["a", "b", "c"] as const);
// => readonly ["a", "b", "c"]

const t2 = makeTupleOfUnion<Foo>()(["a", "b"] as const);
// ERROR: Argument of type 'readonly ["a", "b"]' is not assignable to parameter of type '{ __missing: "c"; }'.
```

実装例は以下です。union の構成要素数をとるために先述の関数のオーバーロードを利用したハックを使っていますが、順序は見ていないので問題ありません。また、union の構成要素数が 40 程度になるとコンパイルエラーが発生しますが、それは先述の 2 通りの方法でも同様です。どうしてもそれ以上の構成要素を持つ union で使いたい場合は、少し安全性は損ないますが tuple の長さのチェックをなくせば構成要素が 1,000 以上あってもコンパイル可能になります。

<!-- prettier-ignore -->
```ts
type Equals<X, Y> =
  (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false;

type Expand<T> = T extends infer U ? { [K in keyof U]: Expand<U[K]> } : never;

type UnionToIntersection<T> =
  (T extends unknown ? (arg: T) => void : never) extends (arg: infer U) => void ? U : never;

type Extra<Expected, Actual, E = Exclude<Actual, Expected>> =
  [E] extends [never] ? never : { __extra: E };

type Missing<Expected, Actual, M = Exclude<Expected, Actual>> =
  [M] extends [never] ? never : { __missing: M };

type ConstituentsError<Union, Tuple extends readonly unknown[]> =
  Expand<UnionToIntersection<Extra<Union, Tuple[number]> | Missing<Union, Tuple[number]>>>;

type LastInUnion<U> = UnionToIntersection<U extends unknown ? (x: U) => void : never> extends (x: infer L) => void ? L : never;

type CountUnionConstituents<U, Result extends never[] = []> =
  [U] extends [never]
    ? Result["length"]
    : CountUnionConstituents<Exclude<U, LastInUnion<U>>, [...Result, never]>;

type LengthError<Union, Tuple extends readonly unknown[]> = {
  __length: `expected ${CountUnionConstituents<Union>}, actual ${Tuple["length"]}`;
};

type UnionToTuple<Union,Tuple extends readonly unknown[]> =
  Equals<Union, Tuple[number]> extends true
    ? Tuple extends { length: CountUnionConstituents<Union> }
      ? Tuple
      : LengthError<Union, Tuple>
    : ConstituentsError<Union, Tuple>;
```

次のように型を書き捨ててチェックする方法でもいいですが、Linter やコンパイラオプションの設定によっては警告が発生する場合があるのと、そもそも書き忘れると効果がないので先述の方法をお勧めします。

```ts
const tuple = ["a", "b", "c"] as const;
type IsCorrect<T, U> = /* ... */;

type Assert<T extends true> = T;
type _ = Assert<IsCorrect<typeof tuple, Foo>>;
```

[^1]: 最後にオーバーロードした関数が最も寛容なケースだろうと判断されるため。

    > When inferring from a type with multiple call signatures (such as the type of an overloaded function), inferences are made from the last signature (which, presumably, is the most permissive catch-all case).

    https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html#type-inference-in-conditional-types

[^2]:
    ファイルが大きすぎて当該コードへのリンクが作れないため PR のリンクを貼ります。執筆時点では閾値はこの PR と同じ。
    https://github.com/microsoft/TypeScript/pull/42353/files#diff-d9ab6589e714c71e657f601cf30ff51dfc607fc98419bf72e04f6b0fa92cc4b8R13333-R13337
