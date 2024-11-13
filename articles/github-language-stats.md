---
title: 'Githubで使用してる言語のチャートを出力するAPIを作った'
emoji: '📊'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [Github, TypeScript]
published: true
---

## やりたいこと

こういうのをGithubのプロフィールに表示したい！

![](/images/github-language-stats/want.png)

## 技術選定

### データ取得
Github APIでリポジトリごとの言語の行数を取得できる。
具体的には以下のGraphQLクエリで取得する。
```graphql
query userInfo($login: String!, $limit: Int!) {
  user(login: $login) {
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
Githubのプロフィールに表示するには画像にする必要がある。

そこでVercelが開発した[Satori](https://github.com/vercel/satori)というライブラリがあり、ReactをSVGに変換することができるらしいので、これを使ってみる。

## できたもの

![](/images/github-language-stats/output.png)

## 課題

satoriでSVG化を行う際、フォントをファイルで指定する必要があるのだが、本家のフォントはシステムフォントを使っているためでファイルを手にいれることができず、近しいフォントを使うしかなさそうだったが、何か方法があるかもしれない。

## 最後に

VercelでAPIを公開しているので、https://github.com/reizt/gh-stats-api のREADMEを読んで是非プロフィールに表示してみてください！
