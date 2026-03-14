---
title: スマホから全自動開発：Mac mini と OpenClaw で作る全自動開発環境
emoji: 📱
type: tech
topics: [OpenClaw, Claude, AI, 自動化, Github]
published: false
---

## 1. はじめに

近年、LLM を利用した開発支援ツールの普及により、開発作業の一部をエージェントに委任するワークフローが現実的になってきました。
コード編集・テスト実行・Pull Request 作成といった一連の作業を自動化し、開発者はタスクの指示とレビューに集中する形です。

本記事では、自宅の **Mac mini** を常時稼働させ、以下のツールを組み合わせて **リモートから操作可能な AI エージェント型開発環境**を構築する方法を解説します。

* OpenClaw
* Claude Code
* Firecrawl
* Cloudflare Tunnel

これにより、スマートフォンから Discord や Linear を通じてタスクを指示し、コード変更から PR 作成までを自動化できます。

---

# 2. システム構成

この構成では、クラウドではなく **ローカルの Mac mini を実行環境として利用**します。

主な理由は以下の通りです。

* Git リポジトリ操作のレイテンシを抑えられる
* 任意のローカル開発環境をそのまま利用できる
* 長時間稼働するエージェントの実行環境を自由に管理できる

構成は次のとおりです。

| コンポーネント        | 役割                         | 接続方法             |
| --------------------- | ---------------------------- | -------------------- |
| **Discord**           | リモート操作インターフェース | OpenClaw Discord Bot |
| **Linear**            | タスク管理・トリガー         | Webhook              |
| **OpenClaw**          | エージェントの実行基盤       | Mac mini             |
| **Claude Code**       | コード生成・編集             | CLI                  |
| **Firecrawl**         | ドキュメント取得             | API                  |
| **Cloudflare Tunnel** | 外部アクセスの公開           | `cloudflared`        |

外部からのリクエストは **Cloudflare Tunnel** を経由して Mac mini に到達します。

---

# 3. セットアップ

ここでは、Mac mini 上でエージェント環境を構築する手順を説明します。

## 3.1 Cloudflare Tunnel の設定

自宅環境を外部に公開する場合、通常は以下の設定が必要になります。

* 固定 IP
* ルーターのポート開放

Cloudflare Tunnel を使用すると、これらの設定なしで安全にローカルサービスを公開できます。

### インストール

```bash
brew install cloudflare/cloudflare/cloudflared
```

### トンネル作成

```bash
cloudflared tunnel create <tunnel-name>
```

DNS を設定します。

```bash
cloudflared tunnel route dns <tunnel-name> <your-domain>
```

### 設定ファイル

`~/.cloudflared/config.yml`

```yaml
tunnel: <tunnel-name>
credentials-file: /Users/<user>/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: <your-domain>
    service: http://localhost:3000
  - service: http_status:404
```

トンネルを起動します。

```bash
cloudflared tunnel run <tunnel-name>
```

このドメインを **Discord / Linear / GitHub Webhook** のエンドポイントとして利用します。

---

# 3.2 OpenClaw のセットアップ

OpenClaw は、AI エージェントを実行するランタイムです。

初期化します。

```bash
npx openclaw init
```

これにより `openclaw.json` が生成されます。

主な設定項目:

* Discord Bot Token
* 利用するチャンネル
* 実行可能コマンド
* アクセス可能ディレクトリ

例:

```json
{
  "channels": {
    "discord": {
      "token": "DISCORD_BOT_TOKEN"
    }
  },
  "security": {
    "allowed_commands": [
      "npm",
      "git",
      "claude"
    ]
  }
}
```

AI が実行可能なコマンドやディレクトリを制限することで、安全性を確保できます。

---

# 3.3 Claude Code の統合

Claude Code は、ターミナル上でコード解析・編集を行う CLI ツールです。
OpenClaw のスキルとして登録することで、AI から呼び出せるようになります。

例:

```json
{
  "skills": [
    {
      "name": "claude_code",
      "description": "コードの解析・修正・テスト実行",
      "command": "claude \"<prompt>\"",
      "working_directory": "/path/to/project"
    }
  ]
}
```

この設定により、エージェントは次の処理を自動実行できます。

* コード修正
* テスト実行
* Git 操作
* PR 作成

---

# 3.4 Firecrawl の連携

LLM はトレーニングデータ以降のライブラリ更新を認識できません。
Firecrawl を利用すると、Web 上のドキュメントを取得して LLM に提供できます。

典型的な用途:

* 最新 API ドキュメント取得
* ライブラリ変更点の確認
* README の解析

Firecrawl は対象ページをクロールし、Markdown 形式に変換します。
この結果をエージェントのコンテキストとして渡すことで、最新仕様に基づいたコード生成が可能になります。

---

# 4. 開発フロー

環境構築後の典型的なワークフローは以下の通りです。

1. 開発者が **Linear** にタスクを作成
2. Webhook により **OpenClaw** がイベントを受信
3. 必要に応じて **Firecrawl** が関連ドキュメントを取得
4. **Discord** に実行確認を通知
5. 承認後、**Claude Code** がコード修正とテストを実行
6. GitHub に Pull Request を作成
7. 開発者がレビューしマージ

---

# 5. まとめ

Mac mini を常時稼働するローカル実行環境として利用することで、AI エージェントによる開発自動化を比較的シンプルな構成で実現できます。

* Discord / Linear をインターフェースとして利用
* OpenClaw がエージェント実行を管理
* Claude Code がコード変更を担当
* Firecrawl が最新ドキュメントを取得

これにより、タスク作成から Pull Request 作成までの一連の作業を自動化できます。
