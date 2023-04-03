---
title: "TypeScript で書いたパッケージを npm に公開する 2023"
emoji: "🌸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["npm", "typescript"]
published: true
---

新しいツールの登場や CommonJS / ES Module 両対応の必要性などによって npm パッケージの開発まわりが数年前と比べて変わってきているので書きました。

## バンドラの選定

バンドラには [webpack](https://github.com/webpack/webpack) や [Vite](https://github.com/vitejs/vite)、[Turbopack](https://github.com/vercel/turbo)、[Rspack](https://github.com/web-infra-dev/rspack) などありますが、この記事では筆者がおすすめする [tsup](https://github.com/egoist/tsup) を使います。tsup は内部で [esbuild](https://github.com/evanw/esbuild)（と [SWC](https://swc.rs/)[^1]）が使われているバンドラで、`.js`、`.json`、`.mjs`、`.ts`、`.tsx`（experimental で`.css`）をサポートしています。esbuild が使われているため webpack と比べて高速で、また Zero-config で使うこともできます（今回はコンフィグを書きますが）。tsup が使われているプロジェクトとして代表的なものには [Chakra UI](https://github.com/chakra-ui/chakra-ui)、[Storybook](https://github.com/storybookjs/storybook)、[Redux](https://github.com/reduxjs/redux)、[Turbo](https://github.com/vercel/turbo)、[Satori](https://github.com/vercel/satori) などがあります。

## パッケージの開発

ディレクトリを切って `npm init` なり `yarn init` なりをしたら、使用するツールをインストールします。今回は最低限のパッケージのみインストールします。Prettier や ESLint などはお好みでインストールしてください。[Jest](https://github.com/facebook/jest) でテストを書く場合は [@swc/jest](https://github.com/swc-project/jest) を使うのがおすすめです。

```bash
$ npm i -D typescript tsup
$ npx tsc --init
```

この記事では変数 foo を export するだけのパッケージをつくります。

```ts:src/index.ts
export const foo = 42;
```

tsup 用の設定ファイルを書きます。`.ts` を使うことができ、TypeScript で型安全に書くことができます[^2]。今回は target は ES2020 とします。バージョンを下げすぎると Polyfill が必要になる場合があることに注意してください。また、CommonJS / ES Module のどちらにも対応するために 2 つのフォーマットで出力します。`clean: true` はビルド前に dist ディレクトリを削除するため、`dts: true` は型ファイルを生成するために設定します。

```ts:tsup.config.ts
import { defineConfig } from "tsup";

export default defineConfig({
  target: "es2020",
  format: ["cjs", "esm"],
  clean: true,
  dts: true,
});
```

設定ファイルを書いたら、ビルドを実行します。

```bash
$ npx tsup ./src
```

すると dist ディレクトリに結果が出力されます。

```
dist
├── index.d.ts
├── index.js
└── index.mjs
```

package.json に以下を書き加えます。files フィールドには公開するための最低限のファイルのみ記述してますが、README.md や LICENSE など必要に応じてファイルを追加してください。exports フィールドではパッケージのエントリポイントを CommonJS、ES Modules、型のためにそれぞれ指定しています。main、module、types フィールドは exports フィールドに対応していないバージョンの Node.js を利用している人向けのフォールバックです。他のフィールドは必要に応じて書き加えてください。

```json
{
  "files": ["dist", "package.json"],
  "exports": {
    ".": {
      "require": "./dist/index.js",
      "import": "./dist/index.mjs",
      "types": "./dist/index.d.ts"
    }
  },
  "main": "./dist/index.js",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts"
}
```

ここまできたらパッケージの公開準備は完了です。必要であれば、[`npm pack`](https://docs.npmjs.com/cli/v9/commands/npm-pack?v=true) でパッケージをテストすることもできます。下記のコマンドでパッケージを公開します。

```bash
$ npm publish
```

お疲れ様でした。

[^1]: ES5 に変換するときに使われる https://tsup.egoist.dev/#es5-support
[^2]: `.js` や、package.json に書くこともできます https://tsup.egoist.dev/#using-custom-configuration
