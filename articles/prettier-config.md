---
title: "Prettier の設定はデフォルトのままがいいのでは？"
emoji: "🔧"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["prettier"]
published: true
publication_name: "pamxy_tech"
---

筆者は Prettier の設定を（ほぼ）デフォルトのまま使っており、この記事では筆者がそうする理由を挙げています。また、この記事にはデフォルト以外の設定で Prettier を利用することを非難する意図はありません。

## 万人が読みやすいフォーマットに最も近い

Prettier の設定は増やせば増やすほど利用者間で共通項は少なくなります。つまり、設定が多いほどコードのフォーマットが乖離するといって差し支えないのではないでしょうか。推奨される Prettier の設定があるのであれば話は別ですが、筆者が調べた限りではそのようなものは見つかりませんでした。Go や Rust では標準のフォーマッタが存在しており、コードのフォーマットに差が生まれないことをメリットの一つとしています。「モダンっぽいから」などの理由で `"semi": false` や `"singleQuote": true` を追加するのは理にかなっていないと筆者は思います。たかが Prettier の設定ごときで可読性が変わることはないだろうという意見はごもっともですが、だからといって Prettier の設定をカスタマイズする理由もありません。個人開発であればどのような設定であろうと自由かもしれませんが、人に見られるようなコードを書く場合は万人が最も読みやすいであろうデフォルトの設定に則ってコードを書くのがベターなのではないでしょうか。

## `"semi": false` 時には Automatic Semicolon Insertion を理解する必要がある

JavaScript は文末にセミコロン（`;`）がなくても文法的には正しい言語ですが、これは JavaScript のパーサーが文末だと判断したときにセミコロンを自動で補完してくれるためです。この仕様を Automatic Semicolon Insertion（以下 ASI）といいます。

しかし、セミコロンなしではコードが意図するように動かない場合があります（Prettier のドキュメントではこれを ASI failures と呼んでいます）。例えば次のようなコードは一見正しそうに見えますが:

<!-- prettier-ignore -->
```js
// 引用: https://prettier.io/docs/en/rationale.html#semicolons

console.log('Running a background task')
(async () => {
  await doBackgroundWork()
})()
```

上記のコードは次のように解釈されるため、エラーになります。

<!-- prettier-ignore -->
```js
console.log("Running a background task")(async () => {
  await doBackgroundWork()
})()
```

これは `console.log()` の末尾（改行後の先頭）にセミコロンを入れることで防ぐことができます。TypeScript や現代のエディタの力を借りればエラーを出してくれるので気づくことはできると思いますが、それよりセミコロンを挿入することで統一した方が無難なのではないでしょうか。

## デフォルトの設定が推奨になる可能性がある

先日、Prettier がデフォルトでインデントにタブを使うべきかという記事が話題になりました[^1]。インデントにタブを使うことはアクセシビリティ上の利点が存在が存在します[^2]。現実的にはこのような変更を嫌う人たちが存在するためこの変更を取り入れることは難しいようです。しかし、タブを使うことに利点があることは間違いありませんし、取り入れる未来もなくはなかったでしょう。そんなときに、もし競合する設定を取り入れていたら恩恵を受けられない可能性があります。

## おわりに

ちなみに筆者は `"trailingComma": "all"` だけ設定しています。これには「無駄な差分が生まれなくなる」、「可読性にほとんど影響を与えない」といった明確な理由があります。みなさんもデフォルト以外の設定で Prettier を使う意図があれば是非主張をお聞かせください。

## 追記

### printWidth は 80 より大きい値を設定することが推奨されていない

長い行は読みやすくするために空白や改行が使われることが多く、平均値は printWidth を大きく下回ることがほとんどだからとのこと。

https://prettier.io/docs/en/options.html#print-width

### Prettier 開発チームがこれ以上オプションを追加しない意向を示している

Prettier の開発者目線の話ですが、Prettier の開発チームはこれ以上オプションを追加しない意向を示しています。Prettier の最大の目的はスタイルをめぐる論争を止めることなのに、オプションを増やせば増やすほどその目的から遠ざかるとのこと。

https://prettier.io/docs/en/option-philosophy.html

[^1]: [Prettier はデフォルトでインデントのためにタブを使うべきなのだろうか](https://sosukesuzuki.dev/posts/prettier-uses-tabs/)
[^2]: [インデントにタブを使うアクセシビリティ上の利点](https://sosukesuzuki.dev/posts/tabs-for-a11y/)
