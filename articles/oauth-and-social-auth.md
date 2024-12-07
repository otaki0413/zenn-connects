---
title: "ソーシャル認証とOAuthの仕組みをざっくり知る"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["oauth", "openid"]
published: false
---

これは「Happiness Chain Advent Calendar 2024」の 11 日目の記事です。
https://adventar.org/calendars/10341

## はじめに

現在 Django を使った Twitter クローン作成に取り組んでおり、Github 認証を実装する場面がありました。実装自体はできたものの、どんな仕組みで成り立っているのかよく分からなかったので、基本的な仕組みを理解したいと思ったのが本記事を書くに至った経緯です。

:::message
専門的な解説ではないので、もしかすると間違った表現をしている可能性があります。
:::

## そもそも OAuth ってなに？

OAuth とは Open Authorization の略であり、

OAuth は現在、Google、X、Facebook、Microsoft など多くのプロバイダーによってサポートされているため、何かしらのサービスにログインする際使われることが多いです。

下記がとてもわかりやすいので、事前に読むことをおすすめします！

https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be

- aaa
- aaa
- aaa

ここで重要なのが、OAuth は「**認可**」に特化した仕組みだということです。
似た表現に「**認証**」がありますが、意味がことなっています。

### 認可と認証

OAuth が認可

- 認可：
- 認証：あああああ

### OAuth と OpenID Connect の比較

|          | OAuth2.0 | OpenID Connect                       |
| -------- | -------- | ------------------------------------ |
| **目的** | 認可     | 認証（この人が誰であるかを確認する） |
| A2       | B2       | C2                                   |
| A3       | B3       | C3                                   |

## GitHub 認証について

Github 認証において、OAuth と OpenID Connect がどのように関係しているかを見ていく。

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant App as アプリ
    participant GitHub as GitHub

    User->>App: 「GitHubでログイン」ボタンをクリック
    App->>GitHub: 認証リクエストを送信
    GitHub->>User: ログイン画面を表示
    User->>GitHub: ユーザー名/パスワードでログイン
    GitHub->>User: アプリへのアクセス許可を確認
    User->>GitHub: 許可を承認
    GitHub->>App: 認証コードを送信
    App->>GitHub: 認証コードを使用してアクセストークンをリクエスト
    GitHub->>App: アクセストークンを送信
    App->>GitHub: APIリクエスト（アクセストークンを添付）
    GitHub->>App: リクエストされたデータを返す
    App->>User: 必要なデータを表示
```

## Google 認証について

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant App as アプリ
    participant Google as Google

    User->>App: 「Googleでログイン」ボタンをクリック
    App->>Google: 認可リクエストを送信
    Google->>User: ログイン画面を表示
    User->>Google: ログイン認証
    Google->>User: アプリへの許可を確認
    User->>Google: 許可を承認
    Google->>App: IDトークンとアクセストークンを送信
    App->>Google: 必要に応じてAPIリクエストを送信（アクセストークンを添付）
    Google->>App: 必要なデータを返す
    App->>User: データを表示
```

## おわりに

本記事では、ソーシャル認証と OAuth の仕組みの基本について、説明しました。
OAuth 自体、すべてを理解しようとすると奥が深い分野ですが、全体像さえ理解できれば、ソーシャル認証を実装する際のなどで活かせる場面があるなと感じました。これからももっと理解を深めていきたいです。

ここまで読んでいただきありがとうございます！

## 参考

https://apidog.com/jp/blog/github-oauth-2-process/
