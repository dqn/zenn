---
title: "Vercel + Next.js + Apollo Server で GraphQL サーバを立てる 2023"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vercel", "nextjs", "graphql"]
published: false
publication_name: "pamxy_tech"
---

掲題でググると deprecated になった [apollo-server-micro](https://www.npmjs.com/package/apollo-server-micro) を使った方法ばかりヒットするので書きました。

## パッケージのインストール

apollo-server-micro ではなく [@apollo/server](https://github.com/apollographql/apollo-server) を使います。`yarn add` の部分は各々のパッケージマネージャに合わせて書き換えてください。今回は TypeScript を使うので型定義や [GraphQL Code Generator](https://github.com/apollographql/apollo-server) などもインストールします。

```bash
$ yarn add next react react-dom graphql @apollo/server
$ yarn add -D typescript @types/react @types/node @graphql-codegen/cli @graphql-codegen/typescript @graphql-codegen/typescript-resolvers prettier
```

インストールしたら package.json の scripts に以下を書き加えてください。codegen.ts は後ほど作成します。

```json:package.json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "codegen": "graphql-codegen --config codegen.ts"
  }
}
```

## Route Handler の定義

今回は Next.js v13 の app ディレクトリを使います。app ディレクトリは執筆時では experimental なので next.config.js で下記のように `experimental.appDir: true` を指定する必要があります。

```js:next.config.js
/**
 * @type {import("next").NextConfig}
 */
const nextConfig = {
  experimental: {
    appDir: true,
  },
};

module.exports = nextConfig;
```

`/graphql` というエンドポイントを作成するには src/app/graphql/ に route.ts という名前でファイルを作成し、handler を定義します。

```bash
$ mkdir -p src/app/graphql
$ touch src/app/graphql/route.ts
```

以下に route.ts の中身のサンプルを示します。GraphQL のリクエストは POST メソッドで受け付けるので、`POST` という名前の関数を export してください。 `resolvers` の中身は型を生成してから書きたいので一旦空で構いません。ApolloServer のインスタンスを生成するところまでは一般的な使い方と同じですが、今回は Serverless Functions で実行するので HTTP サーバを立てる代わりに `executeOperation` を使います。サンプルなのでエラーハンドリングはしていませんが、必要に応じて書き換えてください。

```ts:route.ts
import assert from "node:assert";
import { ApolloServer } from "@apollo/server";
import { NextResponse } from "next/server";

export const typeDefs = /* GraphQL */ `
  type Post {
    id: ID!
    title: String!
  }

  type Query {
    posts: [Post!]!
  }
`;

const resolvers = {};

const server = new ApolloServer({
  typeDefs,
  resolvers,
});

export async function POST(request: Request): Promise<NextResponse> {
  const json = await request.json();
  const res = await server.executeOperation({ query: json.query });
  assert(res.body.kind === "single");
  return NextResponse.json(res.body.singleResult);
}
```

定義したら `yarn dev` で dev サーバを立てます。

## 型の生成と resolvers の定義

プロジェクトのルートに以下のような codegen.ts を作成します。hooks や config はお好みで書き換えてください。

```ts:codegen.ts
import type { CodegenConfig } from "@graphql-codegen/cli";

const config: CodegenConfig = {
  overwrite: true,
  schema: "http://localhost:3000/graphql",
  hooks: {
    afterAllFileWrite: "prettier --write",
  },
  generates: {
    "src/generated/graphql.ts": {
      plugins: ["typescript", "typescript-resolvers"],
      config: {
        useIndexSignature: true,
        useTypeImports: true,
        enumsAsConst: true,
        immutableTypes: true,
        strictScalars: true,
      },
    },
  },
};

export default config;
```

作成したら、dev サーバは起動したまま `yarn codegen` を実行します。成功すると src/generated/graphql.ts が生成されます。最後に、生成された型定義を使って resolvers を定義しましょう。

```ts:route.ts
import { Post, Resolvers } from "../../generated/graphql";

const posts: Post[] = [
  {
    id: "42",
    title: "The Awakening",
  },
  {
    id: "43",
    title: "City of Glass",
  },
];

const resolvers: Resolvers = {
  Query: {
    posts: () => posts,
  },
};
```

リクエストを投げるとクエリに応じたレスポンスが返されるはずです。お疲れ様でした。

```bash
$ curl -X POST -H 'Content-Type: application/json' -d '{"query": "query { posts { id title }}"}' http://localhost:3000/graphql | jq
# {
#   "data": {
#     "posts": [
#       {
#         "id": "42",
#         "title": "The Awakening"
#       },
#       {
#         "id": "43",
#         "title": "City of Glass"
#       }
#     ]
#   }
# }
```
