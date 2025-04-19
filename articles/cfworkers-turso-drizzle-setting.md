---
title: "ReactRouter × Cloudflare Workers × turso の環境に Drizzle を設定したい"
emoji: "🤡"
type: "tech"
topics: ["reactrouter", "cloudflareworkers", "drizzle", "turso"]
published: false
---

## はじめに

最近、ReactRouter を使って個人開発を進めているのですが、
Drizzle と Turso を使った DB 構築に少し詰まったので備忘録として残しておこうと思います。

主な技術スタックは以下のとおりです。

- [ReactRouter v7](https://reactrouter.com/)（フレームワーク）
- [Drizzle](https://orm.drizzle.team/)（ORM）
- [Turso](https://docs.turso.tech/introduction)（データベース）
- Cloudflare Workers（デプロイ）

## やりたいこと

- Cloudflare Workers 上のアプリから、Drizzle で Turso 上のデータにアクセスする
- ローカル環境と本番環境で、異なるデータベースで管理できる状態を作る

## セットアップ

私の場合、Cloudflare が提供する[公式テンプレート](https://github.com/cloudflare/templates/tree/staging/react-router-starter-template)を使いました。
`npm run deploy`で ReactRouter アプリが Cloudflare Workers に簡単にデプロイできます。

```bash:
npm create cloudflare@latest -- --template=cloudflare/templates/react-router-starter-template
```

## 1. データベース作成と接続情報の取得

今回、データベースに Turso を使用します。

Turso にログインした状態で DB を作成します。

```bash
turso db create <データベース名>
```

DB が作成できたら[公式の手順](https://docs.turso.tech/sdk/ts/quickstart)に従って、2 つの資格情報を取得しましょう。

データベース URL の取得

```bash
turso db show --url <データベース名>
```

認証トークンの取得

```bash
turso db tokens create <データベース名>
```

取得できたら、ルートディレクトリ直下に`.dev.vars`を作成して、環境変数として記載します。
ただし、開発環境でのみ使用するファイルなので、`.gitignore`に追記しておきましょう。

```bash:.dev.vars
TURSO_DATABASE_URL=<取得したデータベースURL>
TURSO_AUTH_TOKEN=<取得した認証トークン>
```

実際に `npm run dev` で起動してみると、`.dev.vars`を読み込んでいることがわかるはずです。

```bash
npm run dev

> dev
> react-router dev

Using vars defined in .dev.vars 👈️👈️ これ！！！
  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
  ➜  Debug:   http://localhost:5173/__debug
  ➜  press h + enter to show help
```

環境変数の取り扱いについては、
[こちらの記事](https://zenn.dev/matty5791/articles/d15dbf5a80921b)や[公式ドキュメント](https://developers.cloudflare.com/workers/local-development/environment-variables/)に詳しく記載されているので参考にしてみてください。

## 2. Workers 環境で扱う型定義の生成

[公式ドキュメント](https://developers.cloudflare.com/workers/languages/typescript/#2-generate-runtime-types-using-wrangler)によれば、自身の Worker に合った型定義を生成することを推奨しています。

> We recommend you generate types for your Worker by running wrangler types

npm スクリプトを見てみると、`wrangler types`が設定されており、
こちらを実行することで、`⁠worker-configuration.d.ts`という型定義ファイルが生成できます。

```json:package.json
 "scripts": {
   "typegen": "wrangler types && react-router typegen",
},
```

※初回時は下記のようになっています

```ts:worker-configuration.d.ts（旧）
declare namespace Cloudflare {
  interface Env {
    VALUE_FROM_CLOUDFLARE: "Hello from Cloudflare";
  }
}
interface Env extends Cloudflare.Env {}
```

そして`typegen`コマンド実行後は、`.dev.vars` に設定した Turso の接続情報２つが `Cloudflare` という名前空間の`Env`に設定されていることがわかるはずです。

```ts:worker-configuration.d.ts（新）
declare namespace Cloudflare {
	interface Env {
		VALUE_FROM_CLOUDFLARE: "Hello from Cloudflare";
		TURSO_URL: string; 👈️ これ！
		TURSO_AUTH_TOKEN: string; 👈️ これ！
	}
}
interface Env extends Cloudflare.Env {}
(このあとも続くが省略)
```

ここで作成した型を、以降の Drizzle の設定で使用します。

https://developers.cloudflare.com/workers/languages/typescript/#2-generate-runtime-types-using-wrangler

## 3. Drizzle の設定

次に、アプリケーションと DB を繋ぐ ORM の設定を進めていきます。
今回は Drizzle を使用するので、[公式](https://orm.drizzle.team/docs/connect-turso)に従って必要なパッケージをインストールします。

```bash
npm i drizzle-orm @libsql/client
npm i -D drizzle-kit
```

次に Turso に接続するためのクライアントを作成します。
私の場合、app ディレクトリの中に`db/client.server.ts`ファイルを配置し、記述しました。

こちらの記事を参考にしています。
[React Router × Cloudflare Workers × Supabase で Drizzle ORM を試す](https://www.gaji.jp/blog/2025/02/13/22355/)

```ts: app/db/client.server.ts
import { drizzle } from "drizzle-orm/libsql";
import { createClient } from "@libsql/client";

export const db = (env: Env) => {
  const client = createClient({
    url: env.TURSO_URL,
    authToken: env.TURSO_AUTH_TOKEN,
  });
  return drizzle(client);
};
```

ここで重要なのが、引数の`env`の型`Env`です。
先ほど`worker-configuration.d.ts`に生成した型定義を利用しており、Worker 環境内の環境変数に対して、型安全にアクセスできるようになっています。

そして環境変数をもとに作成した DB クライアントに対して、データベース操作をシンプルにするため、`drizzle`関数でラップしているといった感じになります。

https://orm.drizzle.team/docs/connect-turso

## 4. マイグレーション設定〜実行

```ts:drizzle.config.ts
import type { Config } from "drizzle-kit";
import dotenv from "dotenv";

// 開発環境用の環境変数の読込
dotenv.config({ path: ".dev.vars" });

export default {
  schema: "./app/db/schema.ts",
  out: "./drizzle",
  dialect: "turso",
  dbCredentials: {
    url: process.env.TURSO_URL,
    authToken: process.env.TURSO_AUTH_TOKEN,
  },
} as Config;
```

## おわりに

## 参考サイト

https://orm.drizzle.team/docs/connect-turso
https://docs.turso.tech/sdk/ts/orm/drizzle#drizzle-turso
https://www.gaji.jp/blog/2025/02/13/22355/
https://zenn.dev/matty5791/articles/d15dbf5a80921b
