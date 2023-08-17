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

まず**Auth Helpers**とは、 Supabase の認証機能を操作するための便利な関数やユーティリティをまとめたものです。以下が AuthHelpers の例です。

- **signUp**（ユーザを新規登録するための関数）
- **signIn**（ユーザがログインするための関数、メールやパスワードでの認証サポート）
- **signOut**（ユーザをログアウトさせるための関数）
- **getSession**（現在のユーザ情報を取得するための関数）

https://supabase.com/docs/guides/auth/auth-helpers

これから、AuthHelpers を AppRouter でどのように使えばよいのか見ていきます

> The Next.js Auth Helpers package configures Supabase Auth to store the user's session in a cookie, rather than localStorage. This makes it available across the client and server of the App Router - Client Components, Server Components, Server Actions, Route Handlers and Middleware. The session is automatically sent along with any requests to Supabase.

## おわりに

## 参考サイト

https://supabase.com/docs/guides/auth/auth-helpers/nextjs
