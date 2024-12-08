---
title: "OAuthとソーシャルログインの仕組みをざっくり知る"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["oauth", "openid"]
published: false
---

これは「Happiness Chain Advent Calendar 2024」の 11 日目の記事です。
https://adventar.org/calendars/10341

## はじめに

現在、Django を使った Twitter クローンを作成しており、その中で GitHub ログイン機能を実装しました。しかし、OAuth を使った仕組みがどのように動いているのかが具体的にわからず、もっと理解を深めたいと思いました。本記事では、その基本的な仕組みについて学んだ内容を整理し、初心者向けに解説します。

:::message
専門的な解説ではないので、もしかすると間違った表現をしている可能性があります。
:::

## そもそも OAuth とは？

こちらの記事がとてもわかりやすいので、一度読むことをおすすめします！

https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be

**OAuth**（**=Open Authorization**）とは、アプリや Web サービスがユーザーのデータや機能にアクセスできるようにする仕組みのことです。現在 **OAuth 2.0** が主流となっています。

OAuth で実現できることには、以下のようなものが挙げられます。

- 自身の GitHub アカウントを使って、他のアプリケーションにログインする
- 外部サービスから X の API を通じて、特定ユーザーのプロフィールやポストを取得する
- Slack 連携して、ユーザーが指定したチャンネルに対して通知を送信する

## OAuth は認可のプロトコルである

ここで重要なのが **OAuth** は「**認可（Authorization）**」のプロトコルであるということです。似た表現に「**認証（Authentication）**」がありますが、意味は全く異なるので注意してください。

- **認証（Authentication）**: ユーザーが誰であるかを確認すること
- **認可（Authorization）**: 他のサービスやアプリのデータや機能へのアクセスを許可すること

https://zenn.dev/tanaka_takeru/articles/aecd36a805886d

もし「**認証**」を取り扱う場合には **OpenID Connect** という拡張仕様が使われます。
本記事では取り扱いませんが、OpenID Connect を使うことで、**ユーザーが誰であるかの確認（認証）** もできるため、OAuth と合わせて使う場面がある仕組みでもあります。下記記事が分かりやすいので合わせて読むことを

https://qiita.com/TakahikoKawasaki/items/498ca08bbfcc341691fe

## OAuth の仕組み

それでは OAuth の具体的な流れについて見ていきます。

OAuth は主に 以下 4 つの登場人物で構成されています。
| 登場人物 | 説明 |
| -------------------------------- | --------------------------------------------- |
| **リソースオーナー（ユーザー）** | データを所有しているユーザー |
| **アプリ** | リソースオーナーが利用するアプリケーション |
| **認可サーバー** | ユーザーのアクセス許可を管理するサーバー |
| **リソースサーバー** | ユーザーデータを格納し API で提供するサーバー |

そしてもう 1 つ重要な要素に「**アクセストークン**」があります。
これは認可サーバーが発行するものなのですが、アプリからリソースサーバ上の特定のリソースに対して安全にアクセスする上で必要になります。

### OAuth のフロー

[OAuth のフローは複数あります](https://qiita.com/TakahikoKawasaki/items/200951e5b5929f840a1f#1-%E8%AA%8D%E5%8F%AF%E3%82%B3%E3%83%BC%E3%83%89%E3%83%95%E3%83%AD%E3%83%BC)が、標準的なフローである「**認可コードフロー（Authorization Code Flow）**」を見ていきます。GitHub ログインにおいてもこちらが利用されています。

基本的なステップとしては以下です。

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant App as アプリ
    participant OServer as 認可サーバー
    participant RServer as リソースサーバー

    User->>App: ユーザーのリソースにアクセスしたい
    App->>OServer: 認可リクエストを送信
    OServer->>User: アプリへのアクセス許可を確認
    User->>OServer: アプリへのアクセス許可を承認
    OServer->>App: 認証コードを送信
    App->>OServer: 認証コードを使ってアクセストークンをリクエスト
    OServer->>OServer: アクセストークン生成
    OServer->>App: アクセストークン発行
    App->>RServer: APIリクエスト（アクセストークン添付）
    RServer->>RServer: アクセストークン検証
    RServer->>App: リクエストデータを返す

```

### GitHub ログインのフロー

GitHub ログインの場合、**基本的には OAuth のみで認証が完結します**。
ここで特徴的なのは、GitHub サーバーが認可サーバー&リソースサーバーの両方の役割を担っていることです。
GitHub は、ユーザーにアクセスを許可するための OAuth フローを使用し、認証情報はアプリ側で別途確認しません。

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant App as アプリ
    participant GitHub as GitHub

    User->>App: ①「GitHubでログイン」ボタンをクリック
    App->>GitHub: ②認可リクエストをエンドポイントに送信
    GitHub->>User: ③ログイン画面を表示
    User->>GitHub: ④ユーザー名/パスワードでログイン
    GitHub->>User: アプリへのアクセス許可を確認
    User->>GitHub: 許可を承認
    GitHub->>App: 認証コードを送信
    App->>GitHub: 認証コードを使用してアクセストークンをリクエスト
    GitHub->>App: アクセストークンを送信
    App->>GitHub: APIリクエストを送信（アクセストークン添付）
    GitHub->>App: リクエストされたデータを返す
    App->>App: 取得したGitHubのユーザー情報で認証・ログイン
```

## おわりに

本記事では、ソーシャル認証と OAuth の仕組みの基本について、説明しました。
OAuth 自体、すべてを理解しようとすると奥が深い分野ですが、全体像さえ理解できれば、ソーシャル認証を実装する際のなどで活かせる場面があるなと感じました。これからももっと理解を深めていきたいです。

ここまで読んでいただきありがとうございます！

## 参考

https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be
https://zenn.dev/takamin55/articles/538711ed6fd48d
