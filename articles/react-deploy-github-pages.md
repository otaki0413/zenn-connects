---
title: 'Viteで作成したReactアプリをGitHub Pagesにデプロイする'
emoji: '🚀'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [react, vite, githubactions, githubpages]
published: true
---

## はじめに

本記事では、Vite を用いて作成した React アプリを GitHub Pages にデプロイする手順を解説します。前提として、`npm create vite` で React 環境を構築済みであることを想定しています。
https://ja.vite.dev/guide/#%E6%9C%80%E5%88%9D%E3%81%AE-vite-%E3%83%95%E3%82%9A%E3%83%AD%E3%82%B7%E3%82%99%E3%82%A7%E3%82%AF%E3%83%88%E3%82%92%E7%94%9F%E6%88%90%E3%81%99%E3%82%8B

### GitHub Actions による自動ビルドについて

今回の手順では、ローカルで `npm run build` を実行せずに、GitHub Actions による自動ビルド＆デプロイを行います。ただし、ビルドの流れを理解しやすくするために、あえてローカルでのビルド手順も最初に紹介します。

## 手順

以下の流れで進めます。

1. React アプリのローカルでのビルド手順の確認
2. `vite.config.js` の設定
3. GitHub Actions のワークフロー作成
4. GitHub Pages にデプロイ

https://ja.vite.dev/guide/static-deploy.html#github-pages

## 1. React アプリのローカルでのビルド手順の確認

`package.json` に記載されている `build` コマンドを使います。

```json: package.json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build", 👈️ これ！
    "lint": "eslint .",
    "preview": "vite preview"
  },
}
```

ターミナルにてこちらの`build`コマンドを実行します。

```bash:terminal
npm run build
```

実行後、下記のようにビルド結果がプロジェクト直下の`dist`ディレクトリに出力されます。
こちらを github Pages 等のプラットフォームにデプロイすることでアプリを公開できるようになります。

```bash:terminal
dist/
├── assets
│   ├── index-BD5lPyyF.css
│   └── index-DrXMdGPY.js
├── index.html
```

それではビルドのイメージが付いたところで、実際のデプロイ手順を見ていきましょう。

## 2. `vite.config.js` の設定

GitHub Pages の URL に対応するように `base` にリポジトリ名を設定します。
下記[公式の手順](https://ja.vite.dev/guide/static-deploy.html#github-pages)に従って、私の場合は`/react_todo_list/`としました。

> `https://<USERNAME>.github.io/<REPO>/` にデプロイする場合（例: リポジトリーは `https://github.com/<USERNAME>/<REPO>`）、base を `'/<REPO>/'` と設定してください。

```js: vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

// https://vite.dev/config/
export default defineConfig({
  base: '/react_todo_list/',  👈️ リポジトリ名を設定
});
```

## 3. GitHub Actions のワークフロー作成

プロジェクト直下に`.github/workflows`ディレクトリを作成して、`deploy.yml`を配置します。

```bash:terminal
.github/
└── workflows
    └── deploy.yml
```

そして`deploy.yml`にワークフローを記載します。[こちら](https://ja.vite.dev/guide/static-deploy.html#github-pages)のコードを参考にしてください。

また、今回`main`ブランチではなく、トピックブランチである`react-todo`ブランチへのプッシュ時にワークフローの実行を想定していたので、対象ブランチを修正しています。

```yml: deploy.yml
# 静的コンテンツを GitHub Pages にデプロイするためのシンプルなワークフロー
name: Deploy static content to Pages

on:
  # 対象ブランチプッシュ時に実行されます
  push:
    branches: ['react-todo'] 👈️ プッシュ先のブランチ指定

  # Actions タブから手動でワークフローを実行できるようにします
  workflow_dispatch:

# GITHUB_TOKEN のパーミッションを設定し、GitHub Pages へのデプロイを許可します
permissions:
  contents: read
  pages: write
  id-token: write

# 1 つの同時デプロイメントを可能にする
concurrency:
  group: 'pages'
  cancel-in-progress: true

jobs:
  # デプロイするだけなので、単一のデプロイジョブ
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Build
        run: npm run build
      - name: Setup Pages
        uses: actions/configure-pages@v4
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          # dist フォルダーのアップロード
          path: './dist'
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## 4. GitHub Pages にデプロイ

Github Actions のワークフローが作成できたので、対象ブランチにプッシュしましょう。
GitHub Actions の実行が完了すると、GitHub Pages にデプロイされます。
デプロイ結果を確認するには、GitHub リポジトリの「Settings」→「Pages」に移動し、表示されている URL にアクセスしてください。

### デプロイ対象のブランチが許可されていない場合の対応

私の場合、`react-todo`ブランチにプッシュしたのですが、なぜかデプロイに失敗しました。
「Actions」に記載していたエラー文を確認したところ、`react-todo`ブランチはルールによりデプロイが許可されていないことが原因のようでした。

> Branch "react-todo" is not allowed to deploy to github-pages due to environment protection rules.

![](/images/react-deploy-github-pages/1.png)

こちらの対処として、「Settings」>「Environments」より protection rules の変更を行う必要があります。デフォルトだと`main`ブランチのみデプロイが許可されているため、他のブランチを対象にする場合は追加する必要があります。

![](/images/react-deploy-github-pages/2.png)

下記のように`react-todo`ブランチを追加しました。

![](/images/react-deploy-github-pages/3.png)

そして再度、ワークフローを実行したところ無事デプロイできるようになりました。

## おわりに

本記事では、Vite 製 React アプリを GitHub Pages にデプロイする手順を紹介しました。

デプロイ後、変更を加えたい場合は `react-todo` ブランチにプッシュすれば、GitHub Actions により自動的に再デプロイされるようになっています。

最後に私がデプロイしたアプリを共有しておきます！🚀

https://otaki0413.github.io/react_todo_list/
