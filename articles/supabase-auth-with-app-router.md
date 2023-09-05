---
title: "【AppRouter】Next.js × Supabase 認証機能の実装"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "nextjs"]
published: false
---

## はじめに

Next.js の **AppRouter** と Supabase の **Auth Helpers** による認証機能について簡単にまとめます。
対象者はこれから AppRouter と Supabase を連携させて何かアプリケーションを作ってみたい方にとって参考になるかもしれません。
また、本記事は主にドキュメント記載の内容を元に作成しております。
(https://supabase.com/docs/guides/auth/auth-helpers/nextjs)

## Auth Helpers（認証ヘルパー）とは？

**Auth Helpers**とは、 Supabase の認証機能を操作するための便利な関数やユーティリティをまとめたものです。これを使うことで比較的簡単に認証機能を実装することができます。

以下 **Auth Helpers** の例です。

- **signUp**（ユーザを新規登録するための関数）
- **signIn**（ユーザがログインするための関数、メールやパスワードでの認証をサポート）
- **signOut**（ユーザをログアウトさせるための関数）
- **getSession**（現在のユーザ情報を取得するための関数）

補足ですが、現在 Auth Helpers は**ベータ版**となっています。

> The Auth Helpers are in beta. They are usable in their current state, but it's likely that there will be breaking changes.

https://supabase.com/docs/guides/auth/auth-helpers

::: message
もし Next.js pages ディレクトリを使用されたい場合は、下記が参考になります。
https://supabase.com/docs/guides/auth/auth-helpers/nextjs-pages
:::

## App Router の場合はどのように使う？

次に Auth Helpers を **App Router**の環境 でどのように使えばよいのか見ていきます。

公式によると、Auth Helpers はセッションを**Cookie**に保持する仕組みになっていることから、**Cookie ベースの認証機能**を提供することが分かります。また Next.js の AppRouter からは`Client Components`,`Server Components`, `Server Actions`, `Route Handlers`, `Middleware`などクライアントとサーバーの両方を意識した開発ができるようになっているため、このあたりも考慮する必要がありそうです。

> The Next.js Auth Helpers package configures Supabase Auth to store the user's session in a cookie, rather than localStorage. This makes it available across the client and server of the App Router - Client Components, Server Components, Server Actions, Route Handlers and Middleware. The session is automatically sent along with any requests to Supabase.

## 環境構築

`npx create-next-app -e with-supabase` コマンドを使えば、Auth Helpers による、cookie ベースの認証が設定されたテンプレを生成できますが、今回はマニュアルでセットアップします。

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

Supabase プロジェクトに関する、環境変数を設定します。
supabase プロジェクトの立ち上げ方については詳しくは述べませんが、下記記事が参考になると思います。

```ts:.env.local
NEXT_PUBLIC_SUPABASE_URL=<プロジェクトURL>
NEXT_PUBLIC_SUPABASE_ANON_KEY=<プロジェクトAPIの非公開キー>
```

> The Next.js Auth Helpers package configures Supabase Auth to store the user's session in a cookie, rather than localStorage. This makes it available across the client and server of the App Router - Client Components, Server Components, Server Actions, Route Handlers and Middleware. The session is automatically sent along with any requests to Supabase.

## 実装

### Github OAuth でユーザー認証できるように

今回はボタンによる認証処理を行いたいので、`use client`を記載する必要があります。
つまり、クライアントコンポーネントになるので`createClientComponentClient`関数で Supabase クライアントを作成しましょう。

```ts:AuthButton.tsx
"use client";

import { createClientComponentClient } from "@supabase/auth-helpers-nextjs";

export const AuthButton = () => {
  // Supabaseクライアント作成
  const supabase = createClientComponentClient();

  // サインアウト処理
  const handleSignOut = async () => {
    await supabase.auth.signOut();
  };

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

  return (
    <>
      <button onClick={handleSignIn}>Login</button>
      <button onClick={handleSignOut}>Logout</button>
    </>
  );
};
```

### Route Handlers の作成

サインインに成功すると、auth/callback にリダイレクト処理が走るので、Router Handler で受け取ってあげる必要があります。

```ts:app/auth/callback/route.ts

import { createRouteHandlerClient } from "@supabase/auth-helpers-nextjs";
import { cookies } from "next/headers";
import { NextRequest, NextResponse } from "next/server";

// Code Exchange用のルートハンドラ
export async function GET(request: NextRequest) {
  const requestUrl = new URL(request.url);
  const code = requestUrl.searchParams.get("code");

  /**
   * 認証CodeとユーザーSessionを交換
   * そして今後Supabaseにリクエストする際のCookieとして設定
   */
  if (code) {
    const supabase = createRouteHandlerClient({ cookies });
    await supabase.auth.exchangeCodeForSession(code);
  }

  // サインイン後にリダイレクトするURLを指定
  return NextResponse.redirect(requestUrl.origin);
}
```

:::details Route Handlers

https://nextjs.org/docs/app/building-your-application/routing/route-handlers#cookies
:::

## おわりに

最後まで読んでいただきありがとうございます。

## 参考サイト

https://supabase.com/docs/guides/auth/auth-helpers/nextjs
