---
title: '【コピペOK】Kiroを「完全オート」で動かす設定ファイル集'
tags:
  - AI
  - Kiro
  - 自動化
  - VSCode
  - 開発効率化
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## この記事の内容

Kiroを「完全オート」で動かすための設定ファイルをコピペできる形で提供します。

**実現できること：**
- 質問なしで作業を進める
- 承認なしでファイル編集
- エラーも自動で修正
- 完了まで止まらない

## 設定ファイル一覧

```
.kiro/steering/
├── full-auto-workflow.md    # 完全自動モード
├── investigate-repo.md      # リポジトリ調査モード
└── user-preferences.md      # 基本設定
```

## 1. 完全自動ワークフロー

```md:full-auto-workflow.md
---
inclusion: manual
---

# 完全自動ワークフロー

`#full-auto` で呼び出された場合、以下のルールで作業を進める。

## 基本方針
- **質問しない**。memoと既存コードから推論する
- **承認を求めない**。必要な作業はすべて実行する
- **完了まで止まらない**。エラーが出たら自力で修正する

## 情報の取得先
1. `memo/memo.YYYYMMDD.md` で当日のタスクを確認
2. 前回のコミット履歴で作業の続きを把握
3. 既存コードのパターンに従う

## 作業完了の定義
- コードが動作する状態
- テストがパスする（テストがある場合）
- lintエラーがない（lintがある場合）
- PRが作成されている（push可能な場合）

## エラー時の対応
- ビルドエラー → 修正して再試行（最大3回）
- テスト失敗 → 失敗内容を分析して修正
- 依存関係エラー → npm install を実行
- 型エラー → 型定義を修正
- 3回試行しても解決しない → その時点での状態を報告して停止

## 禁止事項
- mainブランチへの直接push
- rm -rf、git push -f、DROP TABLE 等の破壊的操作
- 本番環境の設定ファイル変更
```

**使い方：**
```
#full-auto 作業を進めて
```

## 2. リポジトリ調査モード

```md:investigate-repo.md
---
inclusion: manual
---

# リポジトリ調査モード

`#investigate` で呼び出された場合、以下のルールで調査を進める。

## 基本方針
- **質問しない**。調査対象を自分で判断する
- **網羅的に調べる**。関連しそうなファイルはすべて確認する
- **レポートにまとめる**。調査結果は構造化して出力する

## 調査の進め方
1. まずディレクトリ構造を把握する
2. エントリーポイント（main, index, app）から追跡する
3. 依存関係を再帰的に辿る
4. 設定ファイル（config, env, yaml）も確認する
5. テストコードからも仕様を読み取る

## レポートの形式

### 1. 概要
- 対象機能の目的
- 主要なファイル一覧

### 2. アーキテクチャ
- データフロー
- 依存関係

### 3. 重要な実装詳細
- 核となるロジック
- 注意すべき点

### 4. 関連ファイル一覧
- ファイルパスと役割の対応表

## 出力先
- 調査結果は memo/investigation-YYYYMMDD-{対象}.md に保存
```

**使い方：**
```
#investigate 認証周りの実装を調査して
```

## 3. 基本設定（常に読み込み）

```md:user-preferences.md
---
inclusion: auto
---

# ユーザー設定

## コミュニケーション
- 敬語を使用すること
- 簡潔に回答すること

## メモファイルへの追記ルール
- メモファイルへの追記は、必ずユーザーに確認してから行う
```

## 4. Hooks設定（オプション）

### ファイル保存時に自動lint

```json:.kiro/hooks/lint-on-save.json
{
  "name": "Lint on Save",
  "version": "1.0.0",
  "when": {
    "type": "fileEdited",
    "patterns": ["*.ts", "*.tsx", "*.js", "*.jsx"]
  },
  "then": {
    "type": "runCommand",
    "command": "npm run lint:fix"
  }
}
```

### 調査の進捗報告

```json:.kiro/hooks/investigation-progress.json
{
  "name": "Investigation Progress",
  "version": "1.0.0",
  "when": {
    "type": "postToolUse",
    "toolTypes": ["read"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "10ファイル以上読み込んだ場合、現在の調査進捗を簡潔に報告してください。"
  }
}
```

## クイックスタート

### 1. ディレクトリ作成

```bash
mkdir -p .kiro/steering
mkdir -p .kiro/hooks
```

### 2. ファイル配置

上記の設定ファイルを`.kiro/steering/`に配置

### 3. Autopilot ON

Kiroのチャット欄上部のトグルで「Autopilot」をONにする

### 4. 使用開始

```
#full-auto 作業を進めて
```

## よくあるカスタマイズ

### プロジェクト固有のルールを追加

```md
## プロジェクト固有ルール
- コミットメッセージは日本語で
- PRタイトルは「feat:」「fix:」で始める
- テストは必ず追加する
```

### 特定ファイルへのアクセス制限

```md
## アクセス制限
- .env ファイルは読み取らない
- production/ 配下は変更しない
```

## まとめ

| 設定 | 用途 |
|------|------|
| `#full-auto` | 定型作業を自動実行 |
| `#investigate` | リポジトリ調査 |
| Hooks | 自動lint等 |

設定ファイルをコピペして、Autopilot ONにするだけで使えます。
