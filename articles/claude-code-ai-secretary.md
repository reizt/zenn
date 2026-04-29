---
title: 'Claude Code + MCPで作る「AIパーソナル秘書」— 毎日の行動を長期目標に自動接続する'
emoji: '🤖'
type: 'tech'
topics: [ClaudeCode, MCP, Obsidian, Pushcut, Todoist]
published: false
---

## はじめに

TODO管理ツールはいろいろ試しました。Todoist、Notion、スマホのショートカット。タスクを整理するのは得意なんですが、どれも長期目標との接続が続きませんでした。

通勤時間は毎日あります。でも「何をするか決めていない」状態で電車に乗ると、気づけばスマホを見ています。夜も大差なく、疲れた頭では「明日やろう」が正直なところです。

そこで Claude Code + MCP群 + Pushcut を使って、行動を長期目標に接続し続ける仕組みを作りました。この記事はその設計と実装の記録です。


## システム全体像

![システム構成図](/images/secretary-system-architecture.png)

システムは4つのレイヤーで構成されています。

**スケジューラ（cron）** が時刻に応じて `claude -p` を起動します。`local-schedule-mcp` を使ってlaunchdジョブとして登録しており、MacBookが起動している限り、7:00・8:00・21:00 に自動実行されます。

**Claude Code（エージェント）** が判断脳です。各時刻に対応したプロンプトファイルを読み込み、MCPからデータを取得して通知内容を組み立て、Obsidianのdailyページに書き込みます。

**データソースMCP群** は現実状態のセンサーです。[Todoist](https://todoist.com)（タスク）、Google Calendar（予定）、[Hevy](https://hevy.com)（トレーニング）、[Macrofactor](https://macrofactorapp.com)（食事・体重）から当日の状態を取得します。

**Pushcut** がiPhoneに通知する役割を担っています。通知には当日のdailyページを開くボタンが付いており、ObsidianのURLスキームで直接開けます。

:::message
Claude Routines（クラウド側の定期実行）も検討しましたが、Hevy・MacrofactorなどのローカルMCPにアクセスできないため、今回はMacBookローカルで完結する構成にしました。
:::


## Phase別実装ウォーク

実装はPhase 1〜3に分けて進めました。各フェーズで作成したのは `agents/` 配下のMarkdownプロンプトファイルで、`claude -p` にそのまま渡しています。

### Phase 1: Daily Brief（7:00）

一番情報量が多いエージェントです。Calendar・Todoist・Hevy・Macrofactorから情報を並列取得し、通勤時の勉強テーマと短中長期目標への接続の説明をObsidian dailyページに書き込んでから通知します。

通勤テーマは曜日ルールで決まります（月・水・金=株、火・木=キャリア）。ただしその日期限のTodoistタスクがあれば、曜日ルールより優先して採用します。

### Phase 2: Commute Brief（8:00）

Daily Briefから1時間後、1行に絞った再通知を送ります。「今日やること：〇〇」だけです。電車に乗る直前に届くので、何をするかで迷わず開始できます。

### Phase 3: Night Theme（21:00）

翌日の通勤テーマを翌日のdailyページに予約します。朝の開始コストをゼロにするのが目的です。明日期限のTodoistタスクがあればそれを優先、なければ曜日ルールから選びます。

---

各エージェントの登録は `local-schedule-mcp` を通じてlaunchdジョブとして行います。`allowed_tools` でRead/Write/Edit/MCPを明示し、`add_dirs` でObsidian vaultのパスを渡すのがポイントです。

```python
create_job(
    name="daily-brief",
    cron="0 7 * * *",
    prompt_file="agents/daily-brief.md",
    allowed_tools=["Read", "Write", "Edit", "mcp__todoist__*", ...],
    add_dirs=["/path/to/obsidian/vault"],
)
```

## まとめ

意志力に頼って長期目標を実行しようとすると、疲れた夜や忙しい朝に必ず負けます。このシステムは「頑張らなくても接続が続く状態」を作ることを目標にしました。

Claude Codeをエージェントとして使う利点は、MCPで現実のデータ（カレンダー・タスク・トレーニング）を参照しながら判断できる点です。単なるリマインダーではなく、その日の状況を踏まえた提案が出てくるのが実感として違います。

まだPhase 3までですが、毎朝通知が届くだけで「今日何をするか迷う時間」が減りました。

## 今後のフェーズ

Phase 1〜3で基本的な通知サイクルは動いています。今後は以下を予定しています。

**Phase 4: 退勤前通知（17:30）**
Hevy（直近ジム日）とGoogle Calendarを見て、「今日ジムに行くか」「何時に寝るか」を判断して通知します。夜を無秩序にしないための仕組みです。

**Phase 5: 週次レビュー（日曜 21:00）**
通勤学習回数・ジム回数・Todoistの完了タスクをまとめてスコア化します。「来週の1改善」を具体的なアクションとして出力し、Obsidianに週次レポートとして保存します。

**Phase 6: 双方向コールバック**
現状のPushcutボタンはdailyページを開くだけです。将来的には「再提案」ボタンでClaudeが別テーマを生成して通知し直す仕組みを作りたいと思っています。MacにHTTPサーバーを立ててCloudflare Tunnelでトンネリングする構成を想定しています。
