---
title: "【AppRouter】Next.js × Supabase 認証機能について"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "nextjs"]
published: false
---

## はじめに

Next.js の **AppRouter** と Supabase の **Auth Helpers** による認証機能について気になったのでまとめます。対象者はこれから AppRouter と Supabase を連携させて何か認証機能を搭載したアプリケーションを作ってみたい方です。
また、本記事は主に下記公式ドキュメントの内容を元に作成しております。
https://supabase.com/docs/guides/auth/auth-helpers/nextjs

## Auth Helpers（認証ヘルパー）とは？

**Auth Helpers**とは、 Supabase の認証機能を操作するための**便利な関数やユーティリティをまとめたもの**です。これを使うことで比較的簡単に認証機能を実装することができます。

Next.js の他に SvelteKit, Remix, Nuxt といった JS フレームワークをサポートしているようです。
https://github.com/supabase/auth-helpers

以下 Auth Helpers の例です。

- **signUp**（ユーザを新規登録するための関数）
- **signIn**（ユーザがログインするための関数、メールやパスワードでの認証をサポート）
- **signOut**（ユーザをログアウトさせるための関数）
- **getSession**（現在のユーザ情報を取得するための関数）

補足ですが、現在 Auth Helpers は**ベータ版**となっています。

> The Auth Helpers are in beta. They are usable in their current state, but it's likely that there will be breaking changes.

https://supabase.com/docs/guides/auth/auth-helpers

## App Router の場合、どのように使う？

次に Auth Helpers を **App Router**の環境 でどのように使えばよいのか見ていきます。

ドキュメントによれば、Auth Helpers は**セッションを localStorage ではなく Cookie に保持する仕組み**になっていることから**Cookie ベースの認証機能**を提供していることが分かります。

また、 Next.js の AppRouter からは`Client Components`,`Server Components`, `Server Actions`, `Route Handlers`, `Middleware`などクライアントとサーバーの両方を意識した開発ができるようになっているため、このあたりも考慮した作りになってそうですね。

> The Next.js Auth Helpers package configures Supabase Auth to store the user's session in a cookie, rather than localStorage. This makes it available across the client and server of the App Router - Client Components, Server Components, Server Actions, Route Handlers and Middleware. The session is automatically sent along with any requests to Supabase.

https://supabase.com/docs/guides/auth/auth-helpers/nextjs

### Supabase クライアント を作成するには？

実際にアプリ側から Supabase に接続するにはどうすればよいのでしょうか？
**Next.js Auth Helpers** は、Supabase のクライアントにアクセスする 5 つの関数を提供しています。

1. `createClientComponentClient`
   - クライアントコンポーネント側から使用するメソッド
2. `createServerComponentClient`
   - サーバーコンポーネント側から使用するメソッド
3. `createServerActionClient`
   - サーバーアクション で使用するメソッド
4. `createRouteHandlerClient`
   - ルートハンドラー で使用するメソッド
5. `createMiddlewareClient`
   - ミドルウェアで使用するメソッド

`create〇〇〇Client` の形式は共通で、アプリ内で呼び出したい場所に応じて`〇〇〇`の部分を変更する書き方となっています。
まさに **App Router の環境に適応した命名**になっているため、分かりやすいですね！

(https://supabase.com/docs/guides/auth/auth-helpers/nextjs#creating-a-supabase-client)

それでは、実際に簡単なプロジェクトを作成しながら、AppRouter と auth-helpers の使い方を見ていきましょう。

## 環境構築

`npx create-next-app -e with-supabase` コマンドを使えば、Auth Helpers による、cookie ベースの認証が設定されたテンプレを生成できますが、今回はマニュアルでセットアップします。
ただし、Supabase のプロジェクトは作成できている前提です。

### Next.js のプロジェクト作成

```:terminal
npx create-next-app@latest
```

### Next.js Auth Helpers のパッケージのインストール

```:terminal
npm i @supabase/auth-helpers-nextjs @supabase/supabase-js
```

https://github.com/supabase/auth-helpers

### 環境変数の設定

Supabase プロジェクトに関する、環境変数を`.env.local`に設定します。

```ts:.env.local
NEXT_PUBLIC_SUPABASE_URL=<プロジェクトURL>
NEXT_PUBLIC_SUPABASE_ANON_KEY=<プロジェクトAPIの非公開キー>
```

ダッシュボードより[Project Setting] > [API]で上記の環境変数を確認することが出来るので、
それらをコピぺしましょう。
![](/images/supabase-api-setting.png)

### テーブル作成とデータ準備

Supabase の **SQL Editor** で下記のクエリを実行し、テーブルとデータを作成します。

#### posts テーブルの作成

```sql
create table if not exists posts (
  id uuid default gen_random_uuid() primary key,
  created_at timestamp with time zone default timezone('utc'::text, now()) not null,
  title text not null
);
```

#### RLS ポリシー の設定

- [RLS](https://supabase.com/docs/guides/auth/row-level-security)の有効化
- select によるデータ取得を行えるようにポリシーを作成する

```sql
alter table posts
  enable row level security;

create policy "anyone can select posts" ON "public"."posts"
as permissive for select
to public
using (true);
```

#### データを posts テーブルに挿入

```sql
insert into
  posts(title)
values
  ('first post'),
  ('second post'),
  ('third post')
```

ここまでで下準備は整ったのでいよいよ **auth-helpers** での実装の方に移っていきましょう。

## 実装

### 1. サーバーコンポーネント上でデータ取得する

まず、Supabase の`posts`テーブルからデータを取得し`app/pages.tsx` で表示します。

今回はサーバーコンポーネント側でデータ取得を行うので`createServerComponentClient` 関数を用いてクライアントを作成します。

```ts:app/page.tsx
import { createServerComponentClient } from "@supabase/auth-helpers-nextjs";
import { cookies } from "next/headers";

export default async function Home() {
  // クライアント作成
  const supabase = createServerComponentClient({ cookies });
  // postsテーブルからデータ取得
  const { data: posts } = await supabase.from("posts").select();

  return <pre>{JSON.stringify(posts, null, 2)}</pre>;
}
```

下記のような見た目になれば大丈夫です！

ここにスクショ

### 2. Github OAuth でユーザー認証できるようにする

今回は、Github によるソーシャルログインを実装します。
::: details Github 認証の設定
実際に Github 認証 を行うためには アプリケーションの認証情報を Github と Supabase に追加する必要があります。公式に手順が記載されていますので参考にして下さい。
(https://supabase.com/docs/guides/auth/social-login/auth-github)
:::
ここでは`AuthButton.tsx`を作成して、ログインボタンとログアウトボタンを作っています。

今回はボタン押下時に認証処理を行いたいので、`use client`を冒頭に記載します。
つまり、クライアントコンポーネントになるので`createClientComponentClient`関数を用いて Supabase クライアントを作成します。

サインイン処理については、[SignInWithOAuth](https://supabase.com/docs/reference/javascript/auth-signinwithoauth) 関数で Github 認証を実装しており、ユーザーがサインインに成功すると、`http://localhost:3000/auth/callback` にリダイレクトする想定です。

```ts:AuthButton.tsx
"use client";

import { createClientComponentClient } from "@supabase/auth-helpers-nextjs";

export const AuthButton = () => {
  // Supabaseクライアント作成
  const supabase = createClientComponentClient();

  // サインイン処理
  const handleSignIn = async () => {
    // GitHub OAuthで認証する
    await supabase.auth.signInWithOAuth({
      provider: "github",
      options: {
        redirectTo: "http://localhost:3000/auth/callback",
      },
    });
  };
  // サインアウト処理
  const handleSignOut = async () => {
    await supabase.auth.signOut();
  };

  return (
    <>
      <button onClick={handleSignIn}>Login</button>
      <button onClick={handleSignOut}>Logout</button>
    </>
  );
};
```

作成した `AuthButton`コンポーネントを `app/page.tsx`でインポートします。

```diff ts:app/page.tsx
import { createServerComponentClient } from "@supabase/auth-helpers-nextjs";
import { cookies } from "next/headers";
+ import { AuthButton } from "./_components/AuthButton";

export default async function Home() {
  const supabase = createServerComponentClient({ cookies });
  const { data: posts } = await supabase.from("posts").select();

  return (
    <>
+     <AuthButton />
      <pre>{JSON.stringify(posts, null, 2)}</pre>;
    </>
  );
}
```

### 3. Route Handlers の作成

サインインに成功すると、`http://localhost:3000/auth/callback` にリダイレクトするので、Router Handler を作成する必要があります。

今回は、app ディレクトリ配下に`auth/callback/route.ts`を作成します。
このルートで実行しているのは、**Code Exchange(コード交換)** です。

まず、リクエスト URL からクエリパラメータ`code`を指定して、認証コードを取得します。
認証コード が存在する場合は、`createRouteHandlerClient`関数でクライアントを作成し、
`exchangeCodeForSession`で アプリ側と Supabase との間でセッションを確立しています。

:::details exchangeCodeForSession
文字だけみると「認証コードをセッションに交換する」を指しますが、
実際は「**認証コードを使用してセッションを確立する**」が正確です。
:::

```ts:app/auth/callback/route.ts
import { createRouteHandlerClient } from "@supabase/auth-helpers-nextjs";
import { cookies } from "next/headers";
import { NextRequest, NextResponse } from "next/server";

// Code Exchange用のルートハンドラ
export async function GET(request: NextRequest) {
  const requestUrl = new URL(request.url);
  const code = requestUrl.searchParams.get("code");

  // 認証コードを使用して、セッションを確立する
  if (code) {
    const supabase = createRouteHandlerClient({ cookies });
    await supabase.auth.exchangeCodeForSession(code);
  }

  // サインイン後にリダイレクトするURLを指定
  return NextResponse.redirect(requestUrl.origin);
}
```

ここまでの内容をまとめると、GitHub 認証を通じてセッションが確立され、ユーザの認証とアプリケーションと Supabase の間でセキュアに通信が行えるになりました。セッション情報には認証情報が管理されるため、今後 Supabase へアクセスする際にはセッション情報が使用されます。

https://nextjs.org/docs/app/building-your-application/routing/route-handlers#cookies

### 4. ミドルウェアの設定

ここまでの実装で 1 つ問題があるためそれを解決します。

#### 問題点

**Cookie の有効期限が切れた場合、更新時に Cookie が削除されてログアウト状態になる**

サーバーコンポーネントは、Cookie を読み取り可能だが、書き戻すことはできないようです。

> Next.js Server Components allow you to read a cookie but not write back to it. Middleware on the other hand allow you to both read and write to cookies.

`serverComponentClient.ts`の実装を見ても、サーバーコンポーネント側からだと`cookies`を設定・削除できない旨がコメントで記載されています。
https://github.com/supabase/auth-helpers/blob/main/packages/nextjs/src/serverComponentClient.ts

#### 解決策

**[Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware) 関数を作成する**

今回はミドルウェア関数で解決します。`middleware.ts`をプロジェクトのルート直下に配置することで、**ルートが読み込まれる直前に何かしらの処理を実行**することができます。

今回の場合だと、**サーバーコンポーネントが読み込まれて、Supabase からデータ取得する時点までにセッションを有効化**させておきます。

[getSession](https://supabase.com/docs/reference/javascript/auth-getsession)関数を実行して、有効期限が切れたセッションを更新させます。

```ts:middleware.ts
import { createMiddlewareClient } from "@supabase/auth-helpers-nextjs";
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

// ルートが読み込まれる直前に実行
export async function middleware(req: NextRequest) {
  const res = NextResponse.next();
  // クライアント作成
  const supabase = createMiddlewareClient({ req, res });
  // 有効期限が切れたセッションを更新
  await supabase.auth.getSession();
  return res;
}
```

この middleware 関数は、サーバーコンポーネントのルートをロードするレスポンスを返します。これにより下記サーバーコンポーネント(app/pages.tsx)の `cookies` 関数には新しいセッションを含む更新後の Cookie が含まれるので、Cookie の期限切れの問題を解決できます。

```ts: app/page.tsx
import { createServerComponentClient } from "@supabase/auth-helpers-nextjs";
import { cookies } from "next/headers";
import { AuthButton } from "./_components/AuthButton";

export default async function Home() {
  const supabase = createServerComponentClient({ cookies });
  const { data: posts } = await supabase.from("posts").select();

  return (
    <>
      <AuthButton />
      <pre>{JSON.stringify(posts, null, 2)}</pre>;
    </>
  );
}
```

https://nextjs.org/docs/app/building-your-application/routing/middleware

## おわりに

最後まで読んでいただきありがとうございます。
Supabase は今非常に勢いのある

## 参考サイト

https://supabase.com/docs/guides/auth/auth-helpers/nextjs
