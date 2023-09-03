---
title: "【AppRouter】Next.jsとSupabaseによる認証機能の実装"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "nextjs"]
published: false
---

## はじめに

Next.js の **AppRouter** を使って、Supabase の認証機能を実装行こうと思います。
対象者はこれから AppRouter と Supabase を連携させて何かアプリケーションを作ってみたい方にとって参考になるかもしれません。最後まで読んで頂けると喜びます。
また、本記事は主に次のサイトに書かれている内容を元に作成しております。
(https://supabase.com/docs/guides/auth/auth-helpers/nextjs)

## Auth Helpers（認証ヘルパー）とは？

**Auth Helpers**とは、 Supabase の認証機能を操作するための便利な関数やユーティリティをまとめたものです。これを使うことで比較的簡単に認証機能を実装することができます。

以下 **AuthHelpers** の例です。

- **signUp**（ユーザを新規登録するための関数）
- **signIn**（ユーザがログインするための関数、メールやパスワードでの認証をサポート）
- **signOut**（ユーザをログアウトさせるための関数）
- **getSession**（現在のユーザ情報を取得するための関数）

補足ですが、公式ドキュメントによると、AuthHelpers は**ベータ版**のようです。

> The Auth Helpers are in beta. They are usable in their current state, but it's likely that there will be breaking changes.

https://supabase.com/docs/guides/auth/auth-helpers

::: message
もし Next.js pages ディレクトリを使用されたい場合は、下記が参考になります。
https://supabase.com/docs/guides/auth/auth-helpers/nextjs-pages
:::

## Supabase Auth with the Next.js App Router

次に AuthHelpers を **App Router**の環境 でどのように使えばよいのか見ていこうと思います。

下記コマンドで、Auth Helpers を用いた、cookie ベースの認証設定済みのテンプレートを生成できるようですが、今回はマニュアルでセットアップしていきます。

```bash:terminal
npx create-next-app -e with-supabase
```

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

```ts:.env.local
NEXT_PUBLIC_SUPABASE_URL=<プロジェクトURL>
NEXT_PUBLIC_SUPABASE_ANON_KEY=<プロジェクトAPIの非公開キー>
```

> The Next.js Auth Helpers package configures Supabase Auth to store the user's session in a cookie, rather than localStorage. This makes it available across the client and server of the App Router - Client Components, Server Components, Server Actions, Route Handlers and Middleware. The session is automatically sent along with any requests to Supabase.

## おわりに

## 参考サイト

https://supabase.com/docs/guides/auth/auth-helpers/nextjs
