---
title: "Workerを別々の環境に分けてデプロイする（Workers Builds編）"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea:
topics: ["cloudflareworkers"]
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

## TODO: Workers Builds を使って環境をわけてみる

今回は Workers Builds を使って、1 つのリポジトリから 3 つの環境を Worker で分離できるようにしてみます。本来だと 3 つも必要ないかもしれませんが。

| 環境       | 用途                         | Worker 名                    |
| ---------- | ---------------------------- | ---------------------------- |
| production | 実際のユーザーが使う本番環境 | **`test_worker_production`** |
| staging    | 動作確認・QA を行う環境      | **`test_worker_staging`**    |
| develop    | 開発環境                     | **`test_worker`**            |

Workers Builds で環境を分けるには、以下の 2 軸で設定を管理する必要があります。

1. **wrangler.json**: 環境ごとの設定値（環境変数・シークレット・バインディングなど）
2. **Cloudflare ダッシュボード**: ブランチとデプロイ環境のマッピング

## `wrangler.jsonc` を設定する

Wrangler の設定ファイル（`wrangler.jsonc`）については、
[こちら](https://developers.cloudflare.com/workers/wrangler/configuration/)に詳細に書かれているのでぜひ参考にしてみてください。

またファイル形式は toml でしたが、json または jsonc が推奨されているので、jsonc で説明していきます。

### 基本構造

`wrangler.json` では、ルートに共通設定を書き、`env.*` で環境ごとの上書き設定を定義します。`env.*` の設定はルートの設定を上書き（マージ）します。

```json
{
  "name": "my-worker",
  // 共通設定（全環境に適用）
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

[継承可能なキー](https://developers.cloudflare.com/workers/wrangler/configuration/#inheritable-keys)と

### name（Worker 名）

環境ごとに Worker 名を分けることで、ダッシュボード上で区別しやすくなります。
name を省略した場合は`<name>-<env.環境名>`となる仕様ですが、
もし明示的に指定したい場合は、name を指定することも可能です。

```json
{
  "name": "test-worker",
  "env": {
    "staging": {
      "name": "" // 省略すると "test-worker-staging"
    },
    "production": {
      // "name": "test-worker-production"
    }
  }
}
```

### vars（環境変数）

また、[継承不可能なキー](https://developers.cloudflare.com/workers/wrangler/configuration/#non-inheritable-keys)に該当します。
トップレベルの `vars`を継承することができないため、`env.<環境>`ごとに`vars`を設定する必要があります。

```json
{
  "vars": {
    "ENV": "development"
  },
  "env": {
    "staging": {
      "vars": {
        "ENV": "staging",
        "API_BASE_URL": "https://staging.example.com"
      }
    },
    "production": {
      "vars": {
        "ENV": "production",
        "API_BASE_URL": "https://example.com"
      }
    }
  }
}
```

### バインディング（KV / D1 / R2 など）

KV や D1 などのバインディングも環境ごとに別リソースを紐づけられます。

```json
{
  "env": {
    "staging": {
      "kv_namespaces": [{ "binding": "MY_KV", "id": "<STAGING_KV_ID>" }],
      "d1_databases": [{ "binding": "MY_DB", "database_id": "<STAGING_DB_ID>" }]
    },
    "production": {
      "kv_namespaces": [{ "binding": "MY_KV", "id": "<PRODUCTION_KV_ID>" }],
      "d1_databases": [
        { "binding": "MY_DB", "database_id": "<PRODUCTION_DB_ID>" }
      ]
    }
  }
}
```

### 共通設定と env.\* の優先順位 のめも

- ルートに書いた設定は**全環境のデフォルト値**として機能します
- `env.*` に同じキーがあれば**env.\* の値で上書き**されます
- `env.*` にしか存在しないキーは**その環境にのみ追加**されます

## Workers Builds のダッシュボード設定

### TODO: ブランチ → デプロイ環境のマッピング

- 設定画面で、ブランチとデプロイ先の環境名（`staging` / `production`）を対応づけます。

- Build・Deploy コマンドを管理画面から設定する

## TODO: Workers Build の課題

- ブランチ制御が細かく行えない問題
  - main ブランチの無駄ビルドが走ってしまう問題があるかも？
  - github actions を使うことでこれ解消できうかも？

## おわりに

Workers Builds を使った環境分離のポイントをまとめます。

- `wrangler.jsonc` のルートに共通設定、`env.*` に環境固有の設定を書く
- 継承可能なキーと継承不可能なキーを把握して、必要に応じてオーバーライドする
- シークレットなどの機密情報は `wrangler secret put <key>`などをセットする
- Workers Builds のダッシュボードでブランチと環境をマッピングさせる

Workers Build を使った Worker 環境の分離について、基本的な手順は理解できたので、
次回は、GitHub Actions のパターンも整理しようと思います。

## 参考

https://developers.cloudflare.com/workers/wrangler/environments/

https://developers.cloudflare.com/workers/wrangler/configuration/
