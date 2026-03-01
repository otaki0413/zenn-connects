---
title: "Worker を複数環境に分けてデプロイする（Workers Builds 編）"
emoji: "🔥"
type: "tech"
topics: ["cloudflareworkers"]
published: false
publication_name: "frontendflat"
---

## はじめに

最近社内で Cloudflare Workers のプロジェクトを改修していました。これまで 1 つの Worker で運用していたのですが、環境を分ける必要が出てきたのでやり方をまとめようと思います。

以降の説明では下記のように省略して記載しますのでご了承ください。

- Cloudflare Pages → **Pages**
- Cloudflare Workers → **Workers**

### 対象読者

- Pages から Workers に乗り換えて環境の切り替えに困っている方
- Workers で複数の環境（本番・プレビューなど）に分けて運用したい方
- Workers Builds の基本的な仕組みを理解したい方

## Pages と Workers の環境の扱い方の違い

Pages から Workers に移行するケースも多いと思うので、まず Git 統合を使用する前提で、両者の環境管理の違いを簡単に見てみましょう。

### Pages の場合

Pages では、1 つのプロジェクト内部で環境（**プロダクション** / **プレビュー**）を分けて管理できます。環境変数や KV・D1 などのバインディングも個別に設定できます。

![image](/images/workers-build-deploy/2.png)
_Pages プロジェクトの設定画面にて環境を選択できるイメージ_

### Workers の場合

一方、Workers の場合、1 つの Worker 内でプロダクションとプレビューなど環境を分けて管理する仕組みがありません。そのため、環境を分けるには別々の Worker としてデプロイする必要があります。その選択肢として 2 つほど考えられそうです。

- **Workers Builds**
- **Github Actions + Wrangler**

本記事ではシンプルな Workers Builds のほうを取り扱います。

## Workers Builds とは？

https://developers.cloudflare.com/workers/ci-cd/builds/

Workers Builds とは、GitHub / GitLab のリポジトリと Workers を連携できる CI/CD の機能です。
ブランチへの push をトリガーにしてビルド → デプロイまでを自動化できるので、[Github Actions での CI/CD パイプライン](https://developers.cloudflare.com/workers/ci-cd/external-cicd/github-actions/)を組まずともシンプルなセットアップで導入できるのが良いポイントです。

また下記は GitHub の例ですが、PR 上にビルド結果が表示され、[プレビュー URL](https://developers.cloudflare.com/workers/configuration/previews/) も出すことができるので、レビュー時の動作確認などに役立ちます。

![image](/images/workers-build-deploy/1.png)
_PR 上にビルド結果が表示されるイメージ_

## Workers Builds を使って環境を分けてみる

今回は Workers Builds を使う前提で、1 つのリポジトリから下記 3 つの Worker に分離してデプロイできるようにします。

| 環境       | 用途のイメージ               | Worker 名                       |
| ---------- | ---------------------------- | ------------------------------- |
| develop    | 開発環境                     | **`workers-builds`**            |
| staging    | 動作確認・QA を行う環境      | **`workers-builds-staging`**    |
| production | 実際のユーザーが使う本番環境 | **`workers-builds-production`** |

検証用のリポジトリは下記になります。

https://github.com/otaki0413/workers-builds-sample

## `wrangler.jsonc` の設定

各 Worker の環境構成には、Wrangler の設定ファイル（`wrangler.jsonc`）を使用します。Cloudflare のダッシュボード上からも設定を変更できるのですが、[公式ドキュメント](https://developers.cloudflare.com/workers/wrangler/configuration/#source-of-truth)では設定ファイルを **Source of truth**（信頼できる唯一の情報源）として管理することを推奨しています。

> We recommend treating your Wrangler configuration file as the source of truth for your Worker configuration, and to avoid making changes to your Worker via the Cloudflare dashboard if you are using Wrangler.

以降の説明で、ダッシュボード上での設定を行う箇所は一部ありますが、運用フローとしても`wrangler.jsonc` や Wrangler CLI 経由で設定を行うのがよいと思われるため、そちらに焦点を当てて説明します。

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

### 設定イメージ

今回の検証用リポジトリの `wrangler.jsonc` の設定イメージです。

- 環境ごとに `vars` や `kv_namespaces` を切り替えている
- `binding` 名は全環境で統一し、`id` で接続先を分けている
- これらの継承不可能なキーは `env.*` ごとに設定が必要（一部省略しています）

```jsonc:wrangler.jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "workers-builds",

  "vars": {
    "ENVIRONMENT": "development",
    "LOG_LEVEL": "debug"
  },
  "kv_namespaces": [{ "binding": "MY_KV", "id": "<DEVELOP_KV_ID>" }],

  // vars やバインディングなどの継承不可能なキーは、env.* ごとに設定が必要
  "env": {
    "staging": {
      "vars": {
        "ENVIRONMENT": "staging",
        "LOG_LEVEL": "debug"
      },
      "kv_namespaces": [{ "binding": "MY_KV", "id": "<STAGING_KV_ID>" }],
    },
    "production": {
      "vars": {
        "ENVIRONMENT": "production",
        "LOG_LEVEL": "error"
      },
      "kv_namespaces": [{ "binding": "MY_KV", "id": "<PRODUCTION_KV_ID>" }],
    }
  }
}
```

## Workers を手動デプロイする

Workers Builds で Git リポジトリと連携するには、既存の Worker から Git リポジトリを選択する必要があるため、事前に Worker が Cloudflare 上に存在している必要があります。そのため、[`wrangler deploy`](https://developers.cloudflare.com/workers/wrangler/commands/#deploy) で 3 環境分を手動デプロイしておきます。

`--env` で環境を指定すると、`wrangler.jsonc` の `env.*` に対応した Worker としてデプロイされます。

```bash
wrangler deploy                    # デフォルト環境
wrangler deploy --env staging      # staging 環境
wrangler deploy --env production   # production 環境
```

下記、Staging 環境向けの手動デプロイログですが、`wrangler.jsonc` に設定した `vars` や KV バインディングが正しく反映されていることが確認できます。

```bash
pnpm wrangler deploy --env staging

 ⛅️ wrangler 4.68.1 (update available 4.69.0)
─────────────────────────────────────────────
Total Upload: 7.46 KiB / gzip: 2.12 KiB
Worker Startup Time: 4 ms
Your Worker has access to the following bindings:
Binding                                                    Resource
env.MY_KV (1a8a4c5339bb458b8411a6638ba86d01)      KV Namespace
env.ENVIRONMENT ("staging")      Environment Variable
env.LOG_LEVEL ("debug")          Environment Variable

Uploaded workers-builds-staging (6.18 sec)
Deployed workers-builds-staging triggers (3.78 sec)
  https://workers-builds-staging.otaki0413-it.workers.dev
```

## シークレットの登録

`wrangler.jsonc` で指定した`vars` は公開しても問題ない値でしたが、API キーや DB 接続情報のような機密情報は [`wrangler secret put`](https://developers.cloudflare.com/workers/wrangler/commands/#secret-put) で Worker ごとに登録する必要があります。

デプロイしたときと同様に、`--env` で環境を指定してシークレットを登録します。

```bash
wrangler secret put <SECRET>                    # デフォルト環境
wrangler secret put <SECRET> --env staging      # staging 環境
wrangler secret put <SECRET> --env production   # production 環境
```

もし複数のシークレットをまとめて登録したい場合は [`wrangler secret bulk`](https://developers.cloudflare.com/workers/wrangler/commands/#secret-bulk) が便利です。
JSON または `.env` 形式のファイルを渡すと、一括登録できます。

```bash
wrangler secret bulk secrets.json
wrangler secret bulk secrets.json --env staging
wrangler secret bulk secrets.json --env production
```

## Workers Builds のダッシュボード設定

Cloudflare ダッシュボードに入り、Workers Builds の設定を進めます。

### ブランチとデプロイ設定

まず、手動デプロイした各 Worker を開き、**Settings > Builds**からリポジトリを接続します。
そして、ブランチとデプロイコマンドを設定します。この設定で**ブランチへの push をトリガーに対応 Worker へ自動デプロイが行われる**ようになります。

![image](/images/workers-build-deploy/3.png)
_Workers Builds の設定画面にてブランチとデプロイ設定を行うイメージ_

### Worker 別の Workers Builds の設定

| Worker 名                   | ブランチ  | 非本番ブランチのビルド | デプロイコマンド                           | バージョンコマンド                 |
| --------------------------- | --------- | ---------------------- | ------------------------------------------ | ---------------------------------- |
| `workers-builds`            | `develop` | ビルドあり             | **`npx wrangler deploy`**                  | **`npx wrangler versions upload`** |
| `workers-builds-staging`    | `staging` | ビルドなし             | **`npx wrangler deploy --env staging`**    | -                                  |
| `workers-builds-production` | `main`    | ビルドなし             | **`npx wrangler deploy --env production`** | -                                  |

Workers Builds のデプロイコマンドはデフォルトで `npx wrangler deploy` ですが、1 つのリポジトリから複数の Worker へデプロイするため、`--env` で環境を明示する必要があります。
指定しないと、3 つの Worker がすべて `wrangler.jsonc` のトップレベル設定（develop 環境）へ上書きデプロイされてしまうため、この考慮が必要になります。

![image](/images/workers-build-deploy/4.png)
_Workers Builds ビルド設定_

:::details 非本番ブランチのビルドに関して

非本番ブランチのビルドを有効にすると、その Worker 上では、**非本番ブランチへの push をトリガーに**プレビューデプロイコマンドが実行されます。
デフォルトは `npx wrangler versions upload`で、バージョンのみがアップロードされ本番トラフィックには影響しない [PreviewURL](https://developers.cloudflare.com/workers/configuration/previews/) が発行されます。

https://developers.cloudflare.com/workers/ci-cd/builds/build-branches/
:::

## Workers Builds の課題

Workers Builds はシンプルに使える反面、ブランチ制御の柔軟性に限界があります。

**ブランチ制御の柔軟性に限界がある**

設定できるのは「本番ブランチ」と「非本番ブランチ」の 2 種類のみです。
そのため、「`staging` ブランチのみ staging 環境へデプロイし、`main` ブランチはビルドをスキップしたい」といった細かい制御はできません。結果として、意図しないブランチでもビルドが走ってしまうことを避けられません。

## おわりに

Workers Builds を使った環境分離のポイントをまとめます。

- `wrangler.jsonc` のルートに共通設定、`env.*` に環境固有の設定を書く
- 継承可能なキーと継承不可能なキーを把握して、必要に応じてオーバーライドする
- 機密情報は `wrangler secret put <KEY> --env <ENV>` で環境ごとに登録する
- Workers Builds のダッシュボードでブランチとデプロイコマンドをマッピングする

細かいブランチ制御が必要な場合は Workers Builds だけでは難しいケースもあるため、次回は GitHub Actions + Wrangler のパターンも整理しようと思います。

## 参考

https://developers.cloudflare.com/workers/ci-cd/builds/git-integration/github-integration/

https://developers.cloudflare.com/workers/wrangler/environments/

https://developers.cloudflare.com/workers/wrangler/configuration/
