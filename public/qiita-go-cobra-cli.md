---
title: 【Go】Cobraで業務自動化CLIツールを30分で作る
tags:
  - Go
  - CLI
  - cobra
  - 自動化
  - GitHub
private: false
---

## TL;DR

- Go + Cobra で CLI ツールを作成
- `biz-tools media draft article.md -p zenn` で記事PRを自動作成
- シングルバイナリで配布が楽

## 背景

複数のメディア（Zenn、Qiita、ブログ）に記事を投稿する際、毎回手動でブランチ作成 → コミット → PR作成をするのが面倒でした。

そこで Go + Cobra で CLI ツールを作り、1コマンドで完結させることにしました。

## 環境

- Go 1.18+
- cobra v1.10.2
- Windows / WSL

## 完成形

```bash
# 記事ファイルを指定してPR作成
biz-tools media draft my-article.md -p zenn

# 出力
Creating branch: draft/zenn-20260614-143000 in /path/to/zenn-repo
Pushing to remote...
Creating Pull Request...
✅ Draft PR created successfully!
   PR URL: https://github.com/user/repo/pull/1
```

## 実装手順

### 1. プロジェクト作成

```bash
mkdir biz-tools && cd biz-tools
go mod init github.com/yourname/biz-tools
go get github.com/spf13/cobra
go get gopkg.in/yaml.v3
```

### 2. ディレクトリ構成

```
biz-tools/
├── main.go
├── go.mod
├── go.sum
├── config.yaml        # プラットフォーム設定
├── config.yaml.example
└── cmd/
    ├── root.go
    └── media.go
```

### 3. main.go

```go
package main

import "github.com/yourname/biz-tools/cmd"

func main() {
    cmd.Execute()
}
```

### 4. cmd/root.go

```go
package cmd

import (
    "fmt"
    "os"
    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "biz-tools",
    Short: "Business automation CLI tool",
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

### 5. cmd/media.go（抜粋）

```go
var mediaDraftCmd = &cobra.Command{
    Use:   "draft [file]",
    Short: "Create a draft and PR on GitHub",
    Args:  cobra.MinimumNArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        platform, _ := cmd.Flags().GetString("platform")
        file := args[0]
        return runDraft(file, platform)
    },
}

func runDraft(file, platform string) error {
    // 1. config.yaml から対象リポジトリを取得
    // 2. ブランチ作成
    // 3. ファイルコピー & コミット
    // 4. プッシュ
    // 5. gh pr create でPR作成
    return nil
}
```

### 6. config.yaml

```yaml
platforms:
  zenn:
    repo: C:\Users\yourname\Documents\git_repo\zenn-content
  qiita:
    repo: C:\Users\yourname\Documents\git_repo\qiita-content
```

## ポイント

### Cobra のサブコマンド構造

```
biz-tools
├── media
│   ├── draft    ← 記事ドラフト作成
│   └── publish  ← 記事公開（予定）
├── video        ← 動画関連（予定）
└── fba          ← FBA関連（予定）
```

`rootCmd.AddCommand(mediaCmd)` でサブコマンドを追加するだけ。

### プラットフォームごとのディレクトリ対応

```go
switch platform {
case "zenn":
    destPath = filepath.Join("articles", filepath.Base(file))
case "qiita":
    destPath = filepath.Join("public", filepath.Base(file))
default:
    destPath = filepath.Base(file)
}
```

### 外部コマンド実行

```go
func gitCommand(args ...string) (string, error) {
    cmd := exec.Command("git", args...)
    output, err := cmd.CombinedOutput()
    return string(output), err
}
```

## Windows でのビルド

WSL から Windows 向けにクロスコンパイル：

```bash
GOOS=windows GOARCH=amd64 go build -o biz-tools.exe
```

## まとめ

| 項目 | 内容 |
|------|------|
| フレームワーク | Cobra |
| 実装時間 | 約30分 |
| コード量 | 約200行 |
| 機能 | PR自動作成 |

Cobra を使えばサブコマンド構造の CLI が簡単に作れます。業務自動化の第一歩としておすすめです。

## 参考リンク

- [spf13/cobra](https://github.com/spf13/cobra)
- [Go by Example](https://gobyexample.com/)
