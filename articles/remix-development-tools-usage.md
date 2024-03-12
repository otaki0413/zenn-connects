---
title: "Remix Development Tools(RDT)を使いこなそう！"
emoji: "💿"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["remix"]
published: false
---

# はじめに

先日、**remix-development-tools**のバージョン 4 がリリースされました！ 🎉
ドキュメントも新しく作られたみたいので、詳しくはこちらを確認してみてください。
https://remix-development-tools.fly.dev/

<!-- https://x.com/AlemTuzlak59192/status/1766829096560910738?s=20 -->

## 対象読者

- Remix を使って開発を行われている方全員

# Remix Development Tools とは？

`Remix Development Tools` (以下 RDT) は、Remix の開発フローに役立つツールセットです。

- **Loader data display**（loader 関数によってロードされているデータを確認できる）
- **Route display**（アプリケーションで使用するルートをリスト/ツリー形式で表示できる）
- **Error tracking**（レンダリングされた無効な HTML とその送信元を確認できる）
- **Hydration mismatch tracking**（ハイドレーションの不一致、Client と Server で何がレンダリングされたかを確認できる）
- **Server logs**（ブラウザ上でサーバーログを確認できる）
- **Route boundaries**（ルート境界を確認できる）

# 手順

それではやっていきましょう。

## 1. Remix アプリ初期設定（Vite を使います）

まずは公式の vite テンプレートを使ってプロジェクトを作成します。

```sh
npx create-remix@latest --template remix-run/remix/templates/vite
```

```sh
npx create-remix@latest --template remix-run/remix/templates/vite

 remix   v2.8.1 💿 Let's build a better website...

   dir   Where should we create your new project?
         rdt-sample

      ◼  Template: Using remix-run/remix/templates/vite...
      ✔  Template copied

   git   Initialize a new git repository?
         Yes

  deps   Install dependencies with npm?
         Yes

      ✔  Dependencies installed

      ✔  Git initialized

  done   That's it!

         Enter your project directory using cd ./rdt-sample
         Check out README.md for development and deploy instructions.

         Join the community at https://rmx.as/discord
```

## 2. RDT のセットアップ

RDT の最新バージョン(v4.0.0)ですが、Vite で実行される ESM プロジェクトのみをサポートしています。要件は以下の通りです。

> - Remix のバージョンが 2.8 以上
> - Remix が Vite 上で実行されている
> - プロジェクトで CommonJS を使用している場合、ESM に変換する必要あり

※CommonJS から ESM へ移行する場合は、[ドキュメント](https://remix.run/docs/en/main/future/vite#migrating)や[この記事](https://alemtuzlak.hashnode.dev/migrating-a-v1-cjs-remix-project-to-remix-vite-esm)が参考になると思います。

### 2-1. パッケージのインストール

```sh
npm install remix-development-tools -D
```

### 2-2. ツールの有効化

`vite.config.ts` ファイルにて、プラグインを追加します。配列の順番に注意して下さい。

```ts: vite.config.ts
import { vitePlugin as remix } from "@remix-run/dev";
import { installGlobals } from "@remix-run/node";
import { defineConfig } from "vite";
import tsconfigPaths from "vite-tsconfig-paths";
import { remixDevTools } from "remix-development-tools"; 👈️

installGlobals();

// pluginsの配列において、remix()より前に記載する必要あり
export default defineConfig({
  plugins: [remixDevTools(), remix(), tsconfigPaths()],
});

```

これで有効化したので、アプリを実行すると右下に表示されます。

![](/images/remix-development-tools-position.png)

# 参考記事

https://remix-development-tools.fly.dev/
