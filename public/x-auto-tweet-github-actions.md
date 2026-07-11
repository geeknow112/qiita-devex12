---
title: 【初心者向け】GitHub ActionsでXに自動投稿する方法 - 5ステップで完成
tags:
  - GitHub
  - GitHubActions
  - Twitter
  - X
  - 自動化
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## この記事でできるようになること

- GitHubにファイルを置くだけでXに自動投稿
- 予約投稿（指定日時に自動でツイート）
- **完全無料**（外部サービス不要）

:::note info
所要時間: 約30分
必要なもの: GitHubアカウント、Xアカウント
:::

## なぜGitHub Actionsなのか？

「Xに定期投稿したいけど、月額サービスは使いたくない」

GitHub Actionsなら**無料枠だけで十分**です。

| サービス | 月額 | 制限 |
|---------|------|------|
| Buffer（無料） | 0円 | 3投稿/日 |
| Hootsuite（無料） | 0円 | 2アカウント、5投稿 |
| **GitHub Actions** | **0円** | **2000分/月**（余裕） |

エンジニアならコードで管理できるGitHub Actionsが便利です。

---

## Step 1: X Developer Portalでアプリを作成

### 1-1. アクセス

https://developer.x.com/en/portal/dashboard

Xアカウントでログインしてください。

:::note warn
初回は「Developer Agreement」への同意が必要です。
用途を聞かれたら「Making a bot」を選んでください。
:::

### 1-2. プロジェクト作成

1. 「Projects & Apps」をクリック
2. 「+ Create Project」をクリック
3. プロジェクト名: `auto-tweet`（何でもOK）
4. Use case: 「Making a bot」を選択
5. Description: 適当に入力（「自動投稿用」など）
6. アプリ名: `my-tweet-bot`（何でもOK）

### 1-3. 認証情報を取得

「Keys and Tokens」タブを開いて、以下の4つをメモ帳にコピー：

- API Key
- API Key Secret
- Access Token（「Generate」を押して生成）
- Access Token Secret

:::note alert
この4つは**絶対に公開しない**でください。
GitHubのコードに直接書くのもNGです。
:::

### 1-4. 書き込み権限を設定

**ここを忘れると投稿できません！**

1. 「User authentication settings」の「Edit」をクリック
2. App permissions: **「Read and write」を選択**
3. Type of App: 「Web App, Automated App or Bot」
4. Callback URI: `https://example.com`（ダミーでOK）
5. Website URL: 自分のGitHubプロフィールURL

設定を保存したら、**Access Tokenを再生成**してください（権限変更後は必須）。

---

## Step 2: GitHubリポジトリを準備

### 2-1. リポジトリ作成

GitHubで新規リポジトリを作成：

- Repository name: `x-auto-tweet`（何でもOK）
- Public or Private: どちらでも可
- 「Add a README file」にチェック

### 2-2. Secretsを登録

リポジトリの「Settings」→「Secrets and variables」→「Actions」

「New repository secret」をクリックして、4つ登録：

| Name | Value |
|------|-------|
| `X_API_KEY` | さっきメモしたAPI Key |
| `X_API_KEY_SECRET` | API Key Secret |
| `X_ACCESS_TOKEN` | Access Token |
| `X_ACCESS_TOKEN_SECRET` | Access Token Secret |

---

## Step 3: 投稿スクリプトを作成

### 3-1. フォルダ構成

```
x-auto-tweet/
├── .github/
│   └── workflows/
│       └── tweet.yml      ← GitHub Actions設定
├── scripts/
│   └── tweet.js           ← 投稿スクリプト
├── tweets/
│   └── (ここにツイートを置く)
└── package.json
```

### 3-2. package.json

```json:package.json
{
  "name": "x-auto-tweet",
  "version": "1.0.0",
  "dependencies": {
    "twitter-api-v2": "^1.15.0"
  }
}
```

### 3-3. 投稿スクリプト

```javascript:scripts/tweet.js
const { TwitterApi } = require('twitter-api-v2');
const fs = require('fs');
const path = require('path');

const client = new TwitterApi({
  appKey: process.env.X_API_KEY,
  appSecret: process.env.X_API_KEY_SECRET,
  accessToken: process.env.X_ACCESS_TOKEN,
  accessSecret: process.env.X_ACCESS_TOKEN_SECRET,
});

const tweetsDir = './tweets';
const postedDir = './tweets/posted';

async function main() {
  // postedフォルダ作成
  if (!fs.existsSync(postedDir)) {
    fs.mkdirSync(postedDir, { recursive: true });
  }

  // tweetsフォルダのJSONを取得
  const files = fs.readdirSync(tweetsDir)
    .filter(f => f.endsWith('.json'));

  const now = new Date();

  for (const file of files) {
    const filePath = path.join(tweetsDir, file);
    if (fs.statSync(filePath).isDirectory()) continue;

    const data = JSON.parse(fs.readFileSync(filePath, 'utf-8'));

    // 予約時刻チェック
    if (data.scheduled_at) {
      const scheduledTime = new Date(data.scheduled_at);
      if (scheduledTime > now) {
        console.log(`⏰ まだ時間じゃない: ${file}`);
        continue;
      }
    }

    try {
      // ツイート投稿
      const result = await client.v2.tweet({ text: data.text });
      console.log(`✅ 投稿成功: ${result.data.id}`);

      // 投稿済みに移動
      fs.renameSync(filePath, path.join(postedDir, file));
      console.log(`📁 移動完了: ${file}`);

    } catch (error) {
      console.error(`❌ 失敗: ${file}`, error.message);
    }
  }
}

main().catch(console.error);
```

---

## Step 4: GitHub Actionsを設定

### 4-1. ワークフローファイル

```yaml:.github/workflows/tweet.yml
name: Auto Tweet

on:
  push:
    paths:
      - 'tweets/*.json'
  schedule:
    - cron: '0 0 * * *'  # 毎日9時（日本時間）
  workflow_dispatch:  # 手動実行

jobs:
  tweet:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm install

      - name: Post tweets
        env:
          X_API_KEY: ${{ secrets.X_API_KEY }}
          X_API_KEY_SECRET: ${{ secrets.X_API_KEY_SECRET }}
          X_ACCESS_TOKEN: ${{ secrets.X_ACCESS_TOKEN }}
          X_ACCESS_TOKEN_SECRET: ${{ secrets.X_ACCESS_TOKEN_SECRET }}
        run: node scripts/tweet.js

      - name: Commit changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add -A
          git diff --staged --quiet || git commit -m "chore: move posted tweets"
          git push
```

---

## Step 5: ツイートを投稿してみる

### 5-1. ツイートファイルを作成

`tweets/test.json`を作成：

```json:tweets/test.json
{
  "text": "GitHub Actionsからの自動投稿テスト！"
}
```

### 5-2. プッシュ

```bash
git add .
git commit -m "test tweet"
git push
```

### 5-3. 結果確認

GitHubの「Actions」タブを開いて、ワークフローが実行されているか確認。

成功すると、Xに投稿されています！

---

## 予約投稿するには？

`scheduled_at`を追加するだけ：

```json:tweets/morning.json
{
  "text": "おはようございます！今日も頑張りましょう。",
  "scheduled_at": "2026-07-15T07:00:00+09:00"
}
```

cronで毎日チェックされ、時刻を過ぎていたら投稿されます。

---

## よくあるエラーと解決法

### 403 Forbidden

**原因**: 書き込み権限がない

**解決**:
1. Developer Portalで「Read and write」に変更
2. Access Tokenを**再生成**
3. GitHubのSecretsを更新

### 429 Too Many Requests

**原因**: 投稿しすぎ

**解決**: 
- 無料プランは1日50ツイートまで
- 時間を空けて再試行

### duplicate content

**原因**: 同じ内容を連続投稿

**解決**: 文面を少し変える

---

## まとめ

GitHub Actionsで無料のX自動投稿システムを作りました。

**この方法のメリット**:
- 完全無料
- コードで管理できる
- 予約投稿できる
- GitHubに履歴が残る

ぜひ試してみてください！

---

## 参考リンク

- [X API Documentation](https://developer.x.com/en/docs)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [twitter-api-v2 (npm)](https://www.npmjs.com/package/twitter-api-v2)
