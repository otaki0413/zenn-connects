---
title: "Cloudflare Workers × Drizzle × Turso で開発環境と本番環境を分けたDB設計をしてみる"
emoji: "🤡"
type: "tech"
topics: ["reactrouter", "cloudflareworkers", "drizzle", "turso"]
published: true
---

## はじめに

最近、ReactRouter を使って個人開発を進めているのですが、 Cloudflare Workers と Drizzle と Turso を使った DB 環境構築に少し詰まったので備忘録として残しておこうと思います。

使用する技術スタックは以下のとおりです。

- [ReactRouter v7](https://reactrouter.com/)（フレームワーク）
- [Drizzle](https://orm.drizzle.team/)（ORM）
- [Turso](https://docs.turso.tech/introduction)（データベース）
- [Cloudflare Workers](https://developers.cloudflare.com/workers/)（デプロイ）

まだ、制作をはじめたばかりですが下記リポジトリになります。
https://github.com/otaki0413/gym-memo

## やりたいこと

- ローカル環境と本番環境で、データベースを分けて管理できるようにする（どちらもリモート上に作成）
- Cloudflare 上のアプリから Drizzle を使って Turso 上のデータにアクセスできるようにする

## 0. セットアップ

私の場合、Cloudflare が提供する[公式テンプレート](https://github.com/cloudflare/templates/tree/staging/react-router-starter-template)を使いました。
また、Remix 公式 が提供する[こちらのテンプレート](https://github.com/remix-run/react-router-templates/tree/main/cloudflare)でも問題ないと思います。

どちらのテンプレートも`npm run deploy`で ReactRouter アプリが Cloudflare Workers に簡単にデプロイできるようになっています。

```bash:
npm create cloudflare@latest -- --template=cloudflare/templates/react-router-starter-template
```

## 1. データベース作成と接続情報の取得

今回、データベースに Turso を使用するので、DB を作成します。

開発環境と本番環境で DB を分けたいので、2 個作ります。
（※DB 作成〜認証トークン取得までを 2 回行う）

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

取得できたら、ルートディレクトリ直下に`.dev.vars`を作成して記載します。
`.dev.vars`は、ローカル開発時に優先的に読み込まれるファイルです。
[公式](https://developers.cloudflare.com/workers/local-development/environment-variables/)に詳しく書いてあるので参考にしてみてください。

```bash:.dev.vars
TURSO_URL=<開発用のデータベースURL>
TURSO_AUTH_TOKEN=<開発用の認証トークン>
```

一方、本番環境で使用する環境変数は、`.prod.vars`というファイルを作成して記載しました。
ファイル名は何でも良いですが、`.dev.vars`と対照的にしたい意図でこうしています。

```bash:.prod.vars
TURSO_URL=<本番用のデータベースURL>
TURSO_AUTH_TOKEN=<本番用の認証トークン>
```

本来、本番環境では Cloudflare のダッシュボードで環境変数を設定しますが、
DB スキーマ変更時にマイグレーションを本番 DB に適用したいときには、ローカルのマイグレーションファイルを適用する必要があると思い、このような形をとっています。
（※もし他に良い方法があればぜひ教えていただきたいです 👏）

どちらのファイルも `.gitignore`に追記しておきましょう。

```bash:.gitignore
.dev.vars*
.prod.vars*
```

ここで`npm run dev` を実行してみましょう。
先ほど説明した通り、開発時には`.dev.vars`が自動的に読み込まれることがわかるはずです。

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

Workers における環境変数の取り扱いについては、
[こちらの記事](https://zenn.dev/matty5791/articles/d15dbf5a80921b)や[公式ドキュメント](https://developers.cloudflare.com/workers/local-development/environment-variables/)に詳しく記載されているので参考にしてみてください。

https://zenn.dev/matty5791/articles/d15dbf5a80921b
https://developers.cloudflare.com/workers/local-development/environment-variables/

## 2. Workers 環境で扱う型定義の生成

[公式ドキュメント](https://developers.cloudflare.com/workers/languages/typescript/#2-generate-runtime-types-using-wrangler)によれば、自身の Worker に合った型定義を生成することを推奨しています。

> We recommend you generate types for your Worker by running wrangler types

npm スクリプトを見てみると、`wrangler types`が設定されているかと思います。
こちら実行することで、`⁠worker-configuration.d.ts`という型定義ファイルが生成されます。

```json:package.json
 "scripts": {
   "typegen": "wrangler types && react-router typegen",
},
```

※初回時は下記

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

## 3. Drizzle のセットアップ

次に、アプリケーションと DB を繋ぐ ORM の設定を進めていきます。
今回は Drizzle を使用するので、[公式](https://orm.drizzle.team/docs/connect-turso)に従って必要なパッケージをインストールします。

```bash
npm i drizzle-orm @libsql/client
npm i -D drizzle-kit
```

次にスキーマの作成を行います。
`app/db/schema.ts`を作成して、下記のようなイメージで作成しました。

```ts:app/db/schema.ts
import { int, sqliteTable, text } from "drizzle-orm/sqlite-core";
import { cuid, createdAt, updatedAt } from "./helpers";

export const userTable = sqliteTable("user", {
  id: cuid(),
  username: text("username", { mode: "text" }).notNull().unique(),
  displayName: text("display_name", { mode: "text" }),
  email: text("email", { mode: "text" }).notNull().unique(),
  avatarUrl: text("avatar_url", { mode: "text" }),
  bio: text("bio", { mode: "text" }),
  createdAt,
  updatedAt,
});

... (省略) ...
```

このあと、このスキーマファイルを元にマイグレーション設定を行います。

## 4. マイグレーション設定

マイグレーションの設定内容は`drizzle.config.ts`に記載するのですが、
今回マイグレーション適用時に`.dev.vars`と`.prod.vars`のどちらの環境変数を読み込むかを切り替えたいので、下記のような実装をしました。
`process.env.ENV`に渡る値は、`package.json`の npm スクリプトで調整します。（以降参照）

```ts:drizzle.config.ts
import type { Config } from "drizzle-kit";
import dotenv from "dotenv";

// ENV に応じて読み込む環境変数ファイルを切替え
const currentEnv = process.env.ENV ?? "development";
console.log(`Current environment: ${currentEnv}`);
dotenv.config({
  path: currentEnv === "production" ? ".prod.vars" : ".dev.vars",
});

if (!process.env.TURSO_URL || !process.env.TURSO_AUTH_TOKEN) {
  console.error("環境変数が設定されていません。");
  process.exit(1);
}

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

package.json に記載する npm スクリプトは下記になります。
マイグレーションの適用コマンドは、`npm run db:migrate` ですが
`ENV`を用いて、開発用 DB と本番用 DB のどちらに適用するか切り替えられるようにしました。

```json:package.json
"db:generate": "drizzle-kit generate",
"db:migrate": "drizzle-kit migrate",
"db:migrate:dev": "ENV=development npm run db:migrate", 👈️ 開発用DBに適用
"db:migrate:prod": "ENV=production npm run db:migrate", 👈️ 本番用DBに適用
"db:studio": "drizzle-kit studio",
```

## 5. Drizzle クライアント作成

次に Turso に接続するためのクライアントを作成します。
私の場合、app フォルダの中に`db/client.server.ts`ファイルを配置し、記述しました。
こちらの方の記事が参考になりました！

https://www.gaji.jp/blog/2025/02/13/22355/

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
ここで使っている `Env` 型は、先ほど `typegen` コマンドで生成された `worker-configuration.d.ts` 内でグローバルに定義されているため、import せずにどこでも使用できます。

そして DB クライアントに対して`drizzle`関数でラップしているといった感じになります。

https://orm.drizzle.team/docs/connect-turso

## 6. データ取得のイメージ

実際の ReactRouter のコードでのデータ取得イメージです。

`loader`関数の引数に渡る`context`から、環境変数にアクセスできるので
`context.cloudflare.env`を db 関数に渡して、DB クライアントを作成してデータ取得を行っています。

```tsx:menus.tsx
import type { Route } from "./+types/menus";
import { db } from "~/db/client.server";
import { trainingMenuTable } from "~/db/schema";

export async function loader({ context }: Route.LoaderArgs) {
   // DBクライアント作成
  const dbClient = db(context.cloudflare.env);
  // データ取得
  const myMenus = await dbClient.select().from(trainingMenuTable);
  return { myMenus };
}

export default function Menus({ loaderData }: Route.ComponentProps) {
  const { myMenus } = loaderData;

  return (
    <div className="space-y-3 p-3">
      <div className="flex items-center justify-between">
        <div className="text-2xl font-semibold">マイメニューの管理</div>
        <div>
          <Button variant="outline" className="aspect-square max-sm:p-0">
            <PlusIcon size={16} aria-hidden="true" />
            新規作成
          </Button>
        </div>
      </div>

      {/* マイメニューリスト */}
      <MenuList initialMenus={myMenus} />
    </div>
  );
}
```

## おわりに

本記事では、個人開発をする中で CloudflareWorkers × Drizzle × Turso の DB 環境構築で詰まった部分についてまとめてみました。
DB を開発用と本番用に分けるやり方は、本当にこれで適切なのかわかりませんがもっと良い方法があればぜひ教えていただきたいです。
ここまで読んでいただきありがとうございました。

## 参考サイト

https://orm.drizzle.team/docs/connect-turso
https://docs.turso.tech/sdk/ts/orm/drizzle#drizzle-turso
https://www.gaji.jp/blog/2025/02/13/22355/
https://zenn.dev/matty5791/articles/d15dbf5a80921b
