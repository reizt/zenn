---
title: '【Githubキラキラプロフィール計画】使っている言語の割合チャートを自作する'
emoji: '📊'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [Github, TypeScript]
published: true
---

## やりたいこと

リポジトリにあるこういうやつと同じチャートをGithubのプロフィールにも表示したい！

![](/images/github-profile-language-stats/want.png)

## 技術選定

### データ取得

Github APIでリポジトリごとの言語の行数を取得できます。

具体的には以下のGraphQLクエリで取得します。

```graphql
query userInfo($username: String!, $limit: Int!) {
  user(login: $username) {
    repositories(ownerAffiliations: OWNER, isFork: false, first: 100) {
      nodes {
        name
        languages(first: $limit, orderBy: {field: SIZE, direction: DESC}) {
          edges {
            size
            node {
              color
              name
            }
          }
        }
      }
    }
  }
}
```

### 画像生成

Githubのプロフィールに表示するには画像にする必要があります。

そこでVercelが開発した[Satori](https://github.com/vercel/satori)というライブラリがあり、ReactをSVGに変換することができるので、これを使いました。

同時にAPIもTypeScript+Reactで作ることが決定しました。

### デプロイ先

Edge runtimeでカジュアルにTypeScriptのAPIをデプロイできるのでVercelを選択しました。

### 自動更新

README.mdの画像にAPIのURLを指定してしまうと、パフォーマンスとAPI制限の懸念があるので、Github Actionsをcronで毎日実行してSVGをリポジトリにコミットするようにしました。

```yaml: .github/workflows/update.yml
name: Update Language Stats
on:
  schedule:
    - cron: 0 0 * * *
jobs:
  # ...
```

## できたもの

馴染んでていい感じです！


ライトテーマ

![](/images/github-profile-language-stats/profile-light.png)

ダークテーマ

![](/images/github-profile-language-stats/profile-dark.png)

プロフィールページはこちら

https://github.com/reizt

## 課題

satoriでSVG化を行う際、フォントをファイルで渡す必要があるのですが、本家のフォントはシステムフォントを使っているためファイルを手に入れることができず、近しいフォントを使うしかなかったです。

何か方法があればいいのですが、、

## 最後に

APIは公開しているので、 https://github.com/reizt/gh-stats-api のREADMEを読んで是非あなたのプロフィールにも表示してみてください！
