---
title: 【Claude Code】IDEなしで快適に！CLI作業環境おすすめ設定まとめ
tags:
  - 開発環境
  - CLI
  - AI
  - Claude
  - ClaudeCode
private: false
updated_at: '2026-06-28T00:34:33+09:00'
id: 8f8d2238d3600f0413c9
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Claude CodeはCLIツールです。VS CodeやJetBrainsのようなGUIのIDEはありません。

最初は「IDEがないと不便じゃないか？」と感じるかもしれません。でも、適切に環境を整えれば、むしろ快適に開発できます。

この記事では、Claude CodeをCLIで使う際のおすすめ設定を紹介します。

## ターミナル環境を整える

### Windows Terminalのペイン分割

WindowsユーザーはWindows Terminalを使いましょう。ペイン分割が使えます。

```
縦分割（左右）: Alt + Shift + +
横分割（上下）: Alt + Shift + -
ペイン間移動:   Alt + 矢印キー
```

左ペインでClaude Codeを動かし、右ペインでファイルの確認やコマンド実行、という使い方が基本になります。

### エディタはビューワーとして使う

IDE代わりにVS Codeをファイルビューワーとして開いておくのがおすすめです。

```bash
# 現在のディレクトリをVS Codeで開く
code .
```

VS Codeは**編集はしない**、ファイルツリーの確認とコードの閲覧専用として使います。実際の編集はClaude Codeに任せます。

## CLAUDE.md でプロジェクトを理解させる

CLAUDE.mdはプロジェクトルートに置く設定ファイルです。Claude Codeが最初に読み込み、プロジェクトの文脈を把握します。

```markdown
# プロジェクト概要
○○向けのWebアプリケーション

## 技術スタック
- バックエンド: Go (Gin)
- フロントエンド: React + TypeScript
- インフラ: AWS ECS Fargate

## 開発ルール
- PRのベースブランチはdevelop
- コミットメッセージはConventional Commits形式
- テストは必ず書く

## 禁止事項
- 本番環境への直接操作
- .envファイルのコミット
```

書けば書くほどClaude Codeの理解が深まり、的外れな提案が減ります。

## settings.json のおすすめ設定

`~/.claude/settings.json`（グローバル設定）または`.claude/settings.json`（プロジェクト設定）で動作を細かく制御できます。

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(ls *)",
      "Bash(cat *)",
      "Bash(grep *)",
      "Bash(find *)",
      "Bash(npm run *)",
      "Bash(go *)"
    ],
    "deny": []
  }
}
```

よく使うコマンドをあらかじめ許可しておくことで、実行のたびに承認を求められるストレスがなくなります。

:::note warn
`deny`に本番環境のコマンドを入れておくと、誤操作防止になります。
:::

## 信頼済みコマンドの設定

`ls`、`grep`、`git log`などの読み取り系コマンドは、毎回承認するのは手間です。まとめて許可しておきましょう。

```json
{
  "permissions": {
    "allow": [
      "Bash(ls *)",
      "Bash(ls)",
      "Bash(cat *)",
      "Bash(grep *)",
      "Bash(find *)",
      "Bash(git log *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git branch *)"
    ]
  }
}
```

破壊的な操作（`rm`、`git reset --hard`など）は許可せず、都度確認するのがおすすめです。

## Hooks で自動化する

Hooksは特定のタイミングで自動的にコマンドを実行する仕組みです。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint --silent 2>/dev/null || true"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '✅ Claude Code 作業完了'"
          }
        ]
      }
    ]
  }
}
```

ファイル編集後に自動でlintを走らせる、作業完了時に通知するなど、繰り返し手動でやっていた操作を自動化できます。

## CLAUDE.md の階層構造を活用する

CLAUDE.mdはディレクトリごとに置けます。

```
project/
├── CLAUDE.md          # プロジェクト全体のルール
├── frontend/
│   └── CLAUDE.md      # フロントエンド固有のルール
└── backend/
    └── CLAUDE.md      # バックエンド固有のルール
```

フロントエンドディレクトリで作業するとき、Claude Codeはプロジェクトルートと`frontend/`の両方のCLAUDE.mdを読み込みます。サブディレクトリに固有のルールを書けます。

## MCP サーバーで機能を拡張する

MCP（Model Context Protocol）を使うと、外部ツールとの連携ができます。

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "your_token"
      }
    }
  }
}
```

GitHubと連携すれば、Claude Codeが直接PRの作成やIssueの確認ができるようになります。

:::note warn
トークンをプロジェクトローカルの`.claude/settings.json`に書く場合は、`.gitignore`に追加してコミットを防いでください。

```
# .gitignore
.claude/settings.json
```

グローバル設定（`~/.claude/settings.json`）に書くほうが安全です。
:::

## 実際の作業フロー

環境が整ったら、作業フローはこうなります。

```
1. Windows Terminalを開く（ペイン分割）
   左: Claude Code起動
   右: ファイル確認 / コマンド実行用

2. プロジェクトディレクトリに移動
   $ cd path/to/project
   $ claude

3. 作業内容を指示
   > このAPIのバリデーション処理を追加して

4. Claude Codeが自動でファイルを編集
   右ペインで変更を確認

5. 問題なければコミット
   > git add と commit して
```

IDE的な操作（ファイルを探す、関数にジャンプする）は全部Claude Codeに頼めるので、意外と不自由しません。

## まとめ

Claude Code CLI環境を快適にするポイント：

| 設定 | 効果 |
|------|------|
| Windows Terminalペイン分割 | 左でAI、右で確認の2画面運用 |
| VS Codeをビューワーとして使用 | ファイルツリーの把握が楽 |
| CLAUDE.mdの充実 | プロジェクト理解が深まり精度向上 |
| 信頼済みコマンドの設定 | 承認作業のストレスを削減 |
| Hooksで自動化 | lint・通知など繰り返し作業をゼロに |

「IDEがない」は欠点ではなく、**AIに全部任せる**という割り切りです。慣れると、IDE時代よりも速く開発できます。
