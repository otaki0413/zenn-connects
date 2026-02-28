---
title: "Worker を別々の環境に分けてデプロイする（Workers Builds 編）"
emoji: "🔥"
type: "tech"
topics: ["cloudflareworkers", "cloudflarepages"]
published: false
publication_name: "frontendflat"
---

## はじめに

最近、社内で Cloudflare Workers プロジェクトを改修していました。
これまで 1 つの Worker で運用していたのですが、環境を分ける必要がでたのでまとめようと思います。本記事では、Workers Builds を使ったパターンを整理していきます。

以降、下記のように省略して記載しますのでご了承ください。

- Cloudflare Pages → Pages
- Cloudflare Workers → Workers

### 対象読者

- Pages から Workers に乗り換えて環境の切り替えに困っている方
- Workers で複数の環境（本番・プレビューなど）に分けて運用したい方
- Workers Builds の基本的な仕組みを理解したい方

## Pages と Workers の環境の扱い方の違い

まず、Git 統合を使用する前提ですが、Pages と Workers の違いを簡単に見てみましょう。

### Pages の場合

Pages では、1 つのプロジェクト内部で環境（プロダクション / プレビュー）を分けて管理できます。環境変数や KV・D1 などのバインディングも個別に設定できます。

![image](/images/workers-build-deploy/2.png)
_Pages プロジェクトの設定画面にて環境を選択できるイメージ_

### Workers の場合

一方、Workers の場合、1 つの Worker 内でプロダクションとプレビューの環境を分けて管理する仕組みがありません。そのため、環境をわけるには別々の Worker としてデプロイする必要があります。その選択肢として 2 つほど考えられそうです。

- **Workers Builds**
- **Github Actions + Wrangler**

本記事では Workers Builds の方を取り扱います。

## Workers Builds とは？

https://developers.cloudflare.com/workers/ci-cd/builds/

Workers Builds とは、GitHub や GitLab 上のリポジトリ と Workers を連携できる CI/CD の機能です。ブランチへの push をトリガーにしてビルド → デプロイまでを自動化できるので、[Github Actions での CI/CD パイプライン](https://developers.cloudflare.com/workers/ci-cd/external-cicd/github-actions/)を組まずともシンプルなセットアップで導入できるのが良いポイントです。

また下記は GitHub の例ですが、PR 上にビルド結果が表示され、[プレビュー URL](https://developers.cloudflare.com/workers/configuration/previews/) も出すことができるので、レビュー時の動作確認などに役立ちます。

![image](/images/workers-build-deploy/1.png)
_PR 上にビルド結果が表示されるイメージ_

## Workers Builds を使って環境を分けてみる

今回は Workers Builds を使う前提で、1 つのリポジトリから下記 3 つの Worker に分離してデプロイできるようにします。

| 環境       | 用途のイメージ               | Worker 名                              |
| ---------- | ---------------------------- | -------------------------------------- |
| develop    | 開発環境                     | **`workers-builds-sample`**            |
| staging    | 動作確認・QA を行う環境      | **`workers-builds-sample-staging`**    |
| production | 実際のユーザーが使う本番環境 | **`workers-builds-sample-production`** |

検証用のリポジトリは下記になります。

https://github.com/otaki0413/workers-builds-sample

## `wrangler.jsonc` の設定

各 Worker の環境構成には、Wrangler の設定ファイル（`wrangler.jsonc`）を使用します。Cloudflare のダッシュボード上から設定はいじれるのですが、[公式ドキュメント](https://developers.cloudflare.com/workers/wrangler/configuration/#source-of-truth)では下記のように設定ファイルを **Source of truth**（信頼できる唯一の情報源）として管理することを推奨しています。

> We recommend treating your Wrangler configuration file as the source of truth for your Worker configuration, and to avoid making changes to your Worker via the Cloudflare dashboard if you are using Wrangler.

以降の説明で、一部ダッシュボード設定を行う箇所はありますが、運用フローとしても`wrangler.jsonc` や Wrangler CLI 経由で設定を行うのがよいと思われるため、そちらに焦点を当てて説明します。

### 基本的な構造

`wrangler.jsonc` の設定ポイントは以下のとおりです。

- **トップレベルに共通設定を書く**
- **`env.*` で環境ごとの上書き設定を書く**
  - [継承可能なキー](https://developers.cloudflare.com/workers/wrangler/configuration/#inheritable-keys)はトップレベルの値が各環境に引き継がれ、`env.*` で上書きできます
  - [継承不可能なキー](https://developers.cloudflare.com/workers/wrangler/configuration/#non-inheritable-keys)はトップレベルの値は継承されないため、各環境で個別設定が必要です

```jsonc:wrangler.jsonc
{
  // トップレベルの共通設定
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "workers-builds-sample",

  "env": {
    "staging": {
      // ステージング環境の上書き設定
    },
    "production": {
      // 本番環境の上書き設定
    }
  }
}
```

### 全体の設定例

実際に今回の 3 環境構成で設定した例はこちらになります。

```jsonc:wrangler.jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "workers-builds-sample",

  // vars は継承不可能なキーのため、env.* ごとに設定が必要
  "vars": {
    "ENVIRONMENT": "development",
    "LOG_LEVEL": "debug"
  },

  "env": {
    "staging": {
      // name を省略すると "workers-builds-sample-staging" が自動で付与される
      "vars": {
        "ENVIRONMENT": "staging",
        "LOG_LEVEL": "debug"
      },
      "kv_namespaces": [{ "binding": "MY_KV", "id": "<STAGING_KV_ID>" }],
    },
    "production": {
      "name": "workers-builds-sample-production",
      "vars": {
        "ENVIRONMENT": "production",
        "LOG_LEVEL": "error"
      },
      "kv_namespaces": [{ "binding": "MY_KV", "id": "<PRODUCTION_KV_ID>" }],
    }
  }
}
```

### シークレットの登録

上記の wrangler.jsonc の `vars` はコード管理されますが、API キーなどの機密情報はコード管理できません。
API キーなどの機密情報は `wrangler secret put` で環境ごとに登録します。

```bash
wrangler secret put SECRET_KEY
wrangler secret put SECRET_KEY --env staging
wrangler secret put SECRET_KEY --env production
```

ダッシュボードの「Settings > Variables and Secrets」からも確認・追加できます。

## Workers Builds のダッシュボード設定

### ブランチ → デプロイ環境のマッピング

Cloudflare ダッシュボードで対象の Worker を開き、**Settings > Builds** からリポジトリを接続します。接続後、ブランチとデプロイ先の環境を対応づけます。

| ブランチ     | デプロイコマンド例                     |
| ------------ | -------------------------------------- |
| `main`       | `npx wrangler deploy`                  |
| `staging`    | `npx wrangler deploy --env staging`    |
| `production` | `npx wrangler deploy --env production` |

ビルドコマンド（例: `npm run build`）とデプロイコマンドは管理画面から設定できます。これにより、ブランチへの push をトリガーに対応する Worker へ自動デプロイされるようになります。

## Workers Builds の課題

Workers Builds はシンプルに使える反面、ブランチ制御の柔軟性に限界があります。

**ブランチ制御が粗い**

設定できるのは「プロダクションブランチ」と「プレビューブランチ（それ以外すべて）」の 2 種類のみです。そのため、「`staging` ブランチのみ staging 環境へデプロイし、`main` ブランチはビルドをスキップしたい」といった細かい制御はできません。結果として、意図しないブランチでもビルドが走ってしまうことがあります。

このような細かいブランチ制御が必要な場合は、GitHub Actions + Wrangler の構成が有効です。次回記事でそちらも整理する予定です。

## おわりに

Workers Builds を使った環境分離のポイントをまとめます。

- `wrangler.jsonc` のルートに共通設定、`env.`\* に環境固有の設定を書く
- 継承可能なキーと継承不可能なキーを把握して、必要に応じてオーバーライドする
- 機密情報は `wrangler secret put <KEY> --env <ENV>` で環境ごとに登録する
- Workers Builds のダッシュボードでブランチとデプロイコマンドをマッピングする

細かいブランチ制御が必要な場合は Workers Builds だけでは難しいケースもあるため、次回は GitHub Actions + Wrangler のパターンも整理しようと思います。

## 参考

[https://developers.cloudflare.com/workers/wrangler/environments/](https://developers.cloudflare.com/workers/wrangler/environments/)

[https://developers.cloudflare.com/workers/wrangler/configuration/](https://developers.cloudflare.com/workers/wrangler/configuration/)
