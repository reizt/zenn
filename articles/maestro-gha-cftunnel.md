---
title: 'Cloudflare Tunnelを使ってGithub ActionsからMacbookにMaestroを実行させる'
emoji: '🚇'
type: 'tech'
topics: ['Cloudflare Tunnel', 'Github Actions', 'Maestro', 'E2E']
published: false
---

## やりたいこと

**Webアプリの自動E2EテストをCI上で動かしたい**

そこで[Maestro](https://maestro.dev)

> Playwrightとの比較はここでは省きます🙏

Maestroのメリット
- 👍 YAMLでシンプルに書ける
- 👍 Maestro StudioのGUIが使いやすい

![Maestro Studio](/images/maestro-gha-cftunnel/maestro-studio.png)

Maestroのデメリット
- Github Actionsで実行するにはMaestro Cloudが必要で、Webだと定額の$125/moとやや高額
- Maestro CLIを使って実行するとChromeDriver周りのバグで失敗する(https://github.com/mobile-dev-inc/maestro/issues/2576)

でもなんとか無料でやりたい。。

## そこで本記事の解決策
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)でローカルPCのWebサーバーをパブリックドメインに公開
- ローカルPCでNodeサーバーを立ち上げMaestroを実行するエンドポイントを作成

![引用：Cloudflare Tunnelの仕組み](/images/maestro-gha-cftunnel/cloudflare-tunnel.webp)

![アーキテクチャ](/images/maestro-gha-cftunnel/arch.png)

## 手順

### 手順1. Cloudflare Zero Trustの設定

#### 手順1.1. Service Tokenを作成する

Access>Service authで「Create Service Token」

> Github Actionsからcurlするときにヘッダーに付与する

![](/images/maestro-gha-cftunnel/screen-service-token.png)

名前と有効期限を入力して「Generate Token」

![](/images/maestro-gha-cftunnel/screen-service-token-create.png)

作成された認証情報を保存

![](/images/maestro-gha-cftunnel/screen-service-token-created.png)

#### 手順1.2. Policyを作成する

Access>Policiesから「Add a policy」

![](/images/maestro-gha-cftunnel/screen-policy.png)

Actionに「Service Auth」、Add rules>IncludeのSelectorは「Service Token」を選択肢Valueに先ほど作成したトークンを設定

![](/images/maestro-gha-cftunnel/screen-policy-create.png)

#### 手順1.3. Applicationを作成する

Access>Applicationsから「Add an application」

![](/images/maestro-gha-cftunnel/screen-application.png)

TypeはSelf-hostedを選択

![](/images/maestro-gha-cftunnel/screen-application-type.png)

Add public hostnameをクリック

![](/images/maestro-gha-cftunnel/screen-application-info.png)

ローカルサーバーをホストするドメインを設定

![](/images/maestro-gha-cftunnel/screen-application-info-hostname.png)

下に行くとPoliciesがあるので「Select existing policies」で先ほど作成したPolicyを選択

![](/images/maestro-gha-cftunnel/screen-application-info-policies.png)

その他はデフォルトの設定で保存

### 手順2. Maestro実行用PCの設定
`cloudflared`というCLIを使ってCloudflare Tunnelに接続する

⚠️ NAME, DOMAIN, TUNNEL_ID, USERNAMEは適宜置き換えてください

```sh
# cloudflaredをインストール
brew install cloudflared
# ログイン
cloudflared tunnel login
# トンネル作成
cloudflared tunnel create <NAME> # IDが出るのでメモ
# ルーティング設定
cloudflared tunnel route dns <NAME> <DOMAIN>
```

`~/.cloudflared/config.yml`に下記を記述

```yml
tunnel: <TUNNEL_ID>
credentials-file: /Users/<USERNAME>/.cloudflared/<TUNNEL_ID>.json
ingress:
  - hostname: <DOMAIN>
    service: http://127.0.0.1:8000
  - service: http_status:404
```

Cloudflare Tunnelに接続する
```sh
cloudflared tunnel run <NAME>
```

あとはlocalhost:8000で起動するサーバーを作るだけ

## 手順3. Maestroを実行するNodeサーバーを作る

server.jsの実装
- ブランチを受け取ってcloneする
- Webアプリを立ち上げる
- maestro実行

:::details 実際のコード

```js: server.js
import { execSync } from 'node:child_process';
import http from 'node:http';
import path from 'node:path';
import { fileURLToPath, URL } from 'node:url';

const PORT = 8000;
const REPO_URL = process.argv[2];
if (!REPO_URL) {
  console.error('Repository URL is required');
  process.exit(1);
}
console.log(`Repository URL: ${REPO_URL}`);

const server = http.createServer((req, res) => {
  try {
    const url = new URL(req.url, `http://localhost:${PORT}`);
    const branch = url.searchParams.get('branch');
    const isBranchValid = branch && /^[a-zA-Z0-9_-]+$/.test(branch);
    if (!isBranchValid) {
      res.statusCode = 400;
      res.end('Branch is invalid');
      return;
    }

    console.log(`Received request: branch=${branch}`);
    const __dirname = path.dirname(fileURLToPath(import.meta.url));
    const shellPath = path.join(__dirname, 'exec.sh');
    execSync(`${shellPath} "${REPO_URL}" "${branch}"`, { stdio: 'inherit' });
    res.statusCode = 200;
    res.end('Test completed');
  } catch (err) {
    console.error(err);
    res.statusCode = 500;
    res.end('Internal server error');
  }
});

server.listen(PORT, () => {
  console.log(`Server listening on http://localhost:${PORT}`);
});
```

```sh: exec.sh
#! /bin/sh

set -e

REPO_URL=$1
BRANCH=$2

CLONE_DIR="tmp/clone-${BRANCH}"

rm -rf ${CLONE_DIR}
mkdir -p ${CLONE_DIR}
echo "📦 Cloning repository to ${CLONE_DIR}..."
git clone --depth=1 -b ${BRANCH} ${REPO_URL} ${CLONE_DIR}
cd ${CLONE_DIR}
echo "📦 Installing dependencies..."
pnpm install
echo "🌱 Starting next dev server..."
nohup pnpm run dev > next.log 2>&1 &
NEXT_PID=$!

echo "🌱 next dev started (pid: $NEXT_PID). Waiting for server..."
until curl -sf http://localhost:3000 >/dev/null 2>&1; do
  sleep 1
done
echo "🚀 next dev ready!"

echo "🎬 Running maestro..."
maestro test e2e/tests --exclude-tags=skip
echo "🎬 Maestro test completed"

echo "🧹 Cleaning up..."
kill $NEXT_PID
rm -rf ${CLONE_DIR}
echo "🧹 Cleanup completed"
```

:::

> 💡 macOSならNodeサーバーを`launchctl`を使って常時稼働状態にするなどの運用方法が現実的と思われる

## 動かしてみる

まずは自分のPCでCloudflare Tunnelにcurlしてみる
![](/images/maestro-gha-cftunnel/local.gif)

Cloudflare Tunnel経由でMaestroを動かせていることがわかる🔥

あとはGithub ActionsのWorkflow fileにこれを書くだけ

Github Action Secretsにドメインと認証情報を登録する

```sh
gh secret set MAESTRO_TUNNEL_HOST --body "<DOMAIN>" # 先ほど設定したドメイン
gh secret set SERVICE_TOKEN_ID --body "..."
gh secret set SERVICE_TOKEN_SECRET --body "..."
```

Workflow fileはcurlするのみ

```yml
name: Maestro Web E2E
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
defaults:
  run:
    working-directory: e2e
jobs:
  maestro-web-cli:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Run maestro test on MacBook via Cloudflare Tunnel
        env:
          BRANCH: ${{ github.head_ref || github.ref_name }}
        run: |
          curl -fsSL "${{ secrets.MAESTRO_TUNNEL_HOST }}?branch=${{ env.BRANCH }}" \
            -H "CF-Access-Client-Id: ${{ secrets.SERVICE_TOKEN_ID }}" \
            -H "CF-Access-Client-Secret: ${{ secrets.SERVICE_TOKEN_SECRET }}"
```

これでmainにプルリクを作った際Maestroが動くようになりました🙌

## まとめ

ローカルサーバーをパブリックドメインにホストできるCloudflare Tunnelを使えば、自宅サーバー的なことが簡単にできるので、いろいろ活用方法が考えられますね！

今回はMaestroをGithub ActionsからローカルPC経由で実行するトリッキーなユースケースを紹介しました。

何かの参考になれば幸いです！
