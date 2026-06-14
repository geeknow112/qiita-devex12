---
title: 【実践】Go + Cobraで業務自動化CLIツールを作る - 設計から実装まで
tags:
  - Go
  - CLI
  - cobra
  - 自動化
  - 効率化
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## この記事で得られること

- Go + Cobra で **実用的な CLI ツール** を作る方法
- 複数の処理対象に対応する **設定ファイル設計**
- `os/exec` を使った **外部コマンド連携**
- WSL + Windows 環境での **ハマりポイント** と解決策

サンプルコードは汎用的に書いているので、自分のユースケースに合わせてカスタマイズできます。

## 背景：定型作業を自動化したい

業務で繰り返し発生する作業、ありませんか？

- 複数のディレクトリに対して同じ操作をする
- 決まったフォーマットでファイルを作成する
- 外部ツール（git, aws cli など）を組み合わせた処理

こうした作業を **1コマンドで終わらせる** CLIツールを Go + Cobra で作ります。

## 完成イメージ

```bash
$ mytool process input.txt --target production

Processing input.txt for target: production
Step 1: Validating input...
Step 2: Executing task...
Step 3: Finalizing...

✅ Completed successfully!
   Output: /path/to/output
```

## 技術スタック

| 項目 | 選定 | 理由 |
|------|------|------|
| 言語 | Go 1.18+ | シングルバイナリ、クロスコンパイル |
| CLIフレームワーク | [spf13/cobra](https://github.com/spf13/cobra) | kubectl, gh と同じ構造 |
| 設定ファイル | YAML | 人間が読みやすい |

### Cobra を選ぶ理由

Cobra は以下のツールで採用されている実績あるフレームワークです：

- **kubectl** (Kubernetes CLI)
- **hugo** (静的サイトジェネレーター)
- **gh** (GitHub CLI)

サブコマンド構造を簡単に作れるのが最大の魅力です。

## プロジェクト構成

```
mytool/
├── main.go              # エントリーポイント
├── go.mod
├── go.sum
├── config.yaml          # 環境別設定（gitignore推奨）
├── config.yaml.example  # 設定テンプレート
├── .gitignore
└── cmd/
    ├── root.go          # ルートコマンド
    ├── config.go        # 設定読み込み
    └── process.go       # サブコマンド
```

## 実装

### 1. プロジェクト初期化

```bash
mkdir mytool && cd mytool
go mod init github.com/yourname/mytool
go get github.com/spf13/cobra
go get gopkg.in/yaml.v3
```

### 2. main.go

```go
package main

import "github.com/yourname/mytool/cmd"

func main() {
    cmd.Execute()
}
```

Cobra の流儀に従ってシンプルに。

### 3. cmd/root.go

```go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "mytool",
    Short: "業務自動化CLIツール",
    Long: `mytool は定型業務を自動化するCLIツールです。

Available commands:
  process  - ファイル処理を実行
  validate - 入力ファイルの検証
  report   - レポート生成`,
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func init() {
    rootCmd.Flags().BoolP("version", "v", false, "バージョン情報を表示")
}
```

### 4. 設定ファイル設計

複数の処理対象（環境、プロジェクトなど）に対応するため、設定をYAMLで管理します。

**config.yaml.example**:
```yaml
# 処理対象ごとの設定
targets:
  development:
    path: /path/to/dev
    options:
      verbose: true
  staging:
    path: /path/to/staging
    options:
      verbose: false
  production:
    path: /path/to/prod
    options:
      verbose: false
      dry_run: true
```

**cmd/config.go**（設定読み込み用ファイル）:

```go
package cmd

import (
    "fmt"
    "os"

    "gopkg.in/yaml.v3"  // go get gopkg.in/yaml.v3
)

type Config struct {
    Targets map[string]TargetConfig `yaml:"targets"`
}

type TargetConfig struct {
    Path    string                 `yaml:"path"`
    Options map[string]interface{} `yaml:"options"`
}

func loadConfig() (*Config, error) {
    data, err := os.ReadFile("config.yaml")
    if err != nil {
        return nil, fmt.Errorf("config.yaml not found: %w", err)
    }

    var config Config
    if err := yaml.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("failed to parse config: %w", err)
    }
    return &config, nil
}
```

### 5. cmd/process.go（サブコマンド実装）

```go
package cmd

import (
    "fmt"
    "os"
    "os/exec"
    "path/filepath"

    "github.com/spf13/cobra"
)

var processCmd = &cobra.Command{
    Use:   "process [file]",
    Short: "ファイル処理を実行",
    Long: `指定されたファイルに対して処理を実行します。

Example:
  mytool process input.txt --target production
  mytool process data.csv --target staging --dry-run`,
    Args: cobra.MinimumNArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        target, _ := cmd.Flags().GetString("target")
        dryRun, _ := cmd.Flags().GetBool("dry-run")
        file := args[0]
        return runProcess(file, target, dryRun)
    },
}

func init() {
    rootCmd.AddCommand(processCmd)
    processCmd.Flags().StringP("target", "t", "development", "処理対象 (development, staging, production)")
    processCmd.Flags().Bool("dry-run", false, "実際の処理を実行せずに確認のみ")
}

func runProcess(file, target string, dryRun bool) error {
    // 1. 設定読み込み
    config, err := loadConfig()
    if err != nil {
        return err
    }

    targetConfig, ok := config.Targets[target]
    if !ok {
        return fmt.Errorf("target '%s' not found in config.yaml", target)
    }

    // 2. 入力ファイル確認
    if _, err := os.Stat(file); os.IsNotExist(err) {
        return fmt.Errorf("file not found: %s", file)
    }

    fmt.Printf("Processing %s for target: %s\n", file, target)
    
    if dryRun {
        fmt.Println("[DRY RUN] 以下の処理を実行します:")
    }

    // 3. 処理実行
    fmt.Println("Step 1: Validating input...")
    fmt.Println("Step 2: Executing task...")
    fmt.Println("Step 3: Finalizing...")

    fmt.Printf("\n✅ Completed successfully!\n")
    fmt.Printf("   Output: %s\n", filepath.Join(targetConfig.Path, filepath.Base(file)))

    return nil
}
```

### 6. 外部コマンド実行

git や aws cli など、外部ツールと連携する場合：

```go
func runCommand(name string, args ...string) (string, error) {
    cmd := exec.Command(name, args...)
    output, err := cmd.CombinedOutput()
    if err != nil {
        return string(output), fmt.Errorf("%w: %s", err, string(output))
    }
    return string(output), nil
}

func gitCommit(message string) error {
    if _, err := runCommand("git", "add", "."); err != nil {
        return err
    }
    if _, err := runCommand("git", "commit", "-m", message); err != nil {
        return err
    }
    return nil
}
```

## ハマったポイント

### 1. WSL と Windows で認証情報が別

**症状**: WSL でビルドしたバイナリが Windows の認証情報を使えない

**解決策**:

```bash
# Windows 向けにクロスコンパイル
GOOS=windows GOARCH=amd64 go build -o mytool.exe
```

### 2. os.Chdir() の罠

**症状**: ディレクトリ移動後の処理が終わっても元に戻らない

**解決策**: defer で確実に戻す

```go
func processInDirectory(dir string) error {
    originalDir, _ := os.Getwd()
    if err := os.Chdir(dir); err != nil {
        return err
    }
    defer os.Chdir(originalDir)
    // 処理...
    return nil
}
```

### 3. エラーメッセージが不親切

**症状**: `exit status 1` だけで何が起きたかわからない

**解決策**: CombinedOutput で標準エラーも取得

```go
output, err := cmd.CombinedOutput()
if err != nil {
    return fmt.Errorf("%w: %s", err, string(output))
}
```

## ビルドとインストール

```bash
# 開発中（プロジェクトルートで実行）
go build -o mytool

# ローカルにインストール（プロジェクトルートで実行）
go install .

# Windows 向け（WSL から）
GOOS=windows GOARCH=amd64 go build -o mytool.exe

# Mac 向け
GOOS=darwin GOARCH=amd64 go build -o mytool-mac
```

## まとめ

| ポイント | 内容 |
|----------|------|
| フレームワーク | Cobra でサブコマンド構造を簡単に作成 |
| 設定管理 | YAML で環境別設定を外出し |
| 外部連携 | os/exec で git, aws cli 等と連携 |
| クロスコンパイル | GOOS, GOARCH で各OS向けビルド |

Go + Cobra なら、シングルバイナリで配布も楽。ぜひ自分の業務に合わせたCLIツールを作ってみてください。

## 参考リンク

- [spf13/cobra - GitHub](https://github.com/spf13/cobra)
- [Go by Example](https://gobyexample.com/)
