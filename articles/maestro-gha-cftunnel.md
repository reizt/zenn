---
title: 'Cloudflare Tunnelを使ってGithub ActionsからMacbookにMaestroを実行させる'
emoji: '🚇'
type: 'tech'
topics: ['Cloudflare Tunnel', 'Github Actions', 'Maestro']
published: false
---

## やりたいこと

- Webアプリの自動E2EテストをCI上で動かしたい
- Why maestro
  - 👍 YAMLでシンプルに書ける
  - 👍 Maestro StudioのGUIが使いやすい
- Maestroの欠点 : Github Actionsで実行できない(ChromeDriver周りのバグ、とだけ言って詳細は語らない、Issueは貼っとくか？)

## ソリューション
- Cloudflare TunnelでローカルPCのWebサーバーをパブリックドメインに公開
- Webサーバーの中でサブプロセスを立ち上げてmaestroを実行する
- アーキ図貼る

### Cloudflare Tunnelの概要

- ローカルPCのWebサーバーをパブリックドメインに公開できる(ngrokみたいな？とか言おうかな)
- 他にもSSHなども可能(他のユースケースも詳細に調査する？)

## 手順 - Cloudflare Tunnel周り

- ダッシュボード
  - Service Tokenを作成
  - Policyを作成してService Tokenを許可
  - Access Applicationを作成してPolicyをアタッチ
- 常時稼働PC
  - brew install cloudflared でインストール
  - cloudflared tunnel create でトンネル作成
  - cloudflared tunnel route dns でルーティング作成
  - ~/.cloudflared/config.ymlに認証情報とingress設定
  - cloudflared tunnel run xxx で起動
  - Honoサーバーを作って立ち上げる

## 手順 - Maestroを実行するHonoサーバー周り

やること
- push時やPR作成時に実行する
- 最新のコードを取ってくる
  - 最低限のファイルさえ取れればいい
- サブプロセスでmaestro実行

## 動かしてみる
- まずはターミナルからcurlでTunnelを呼び出してみる
- 動いた！
- Github ActionsのWorkflow fileにこれを書くだけ
  - secretの設定はする
- プルリク作ってみる
- 動いた！

## まとめ
