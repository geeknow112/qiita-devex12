---
title: 【2026年版】direnv実践ガイド - プロジェクトごとの環境変数をディレクトリで自動切り替え
tags:
  - direnv
  - CLI
  - DevEx
  - Environment
  - dotenv
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

複数プロジェクトを行き来していると、「このプロジェクトだけAPIキーが違う」「このプロジェクトはstaging環境を向いていてほしい」といった環境変数の切り替えが煩雑になりがちです。`direnv` は、ディレクトリに入った瞬間に環境変数を自動で読み込み・破棄してくれるシェル拡張です。この記事では基本的な使い方から実務でのハマりどころまで紹介します。

## 基本的な使い方

プロジェクト直下に `.envrc` を置くだけです。

```bash
# .envrc
export DATABASE_URL="postgres://localhost/myapp_dev"
export API_KEY="dev-key-12345"
```

初回は明示的な許可が必要です。

```bash
direnv allow
```

これでこのディレクトリに `cd` するたびに環境変数が自動で読み込まれ、ディレクトリを出ると自動で破棄されます。`.bashrc` や `.zshrc` にグローバルな環境変数を書き散らす必要がなくなります。

## dotenvとの連携

`.env` ファイルをすでに運用しているプロジェクトでも、direnvは簡単に統合できます。

```bash
# .envrc
dotenv .env
dotenv .env.local
```

`dotenv_if_exists` を使えば、ファイルが存在しない場合でもエラーにせずスキップできます。CI環境と開発環境で `.env.local` の有無が異なる構成でも安全に運用できます。

## レイヤー構成（standard library関数）

direnvには `layout` という便利な組み込み関数があり、言語ごとの仮想環境を自動アクティベートできます。

```bash
# Python
layout python3

# Node.js（.nvmrcのバージョンを使う）
use node
```

`use node` は `.nvmrc` を読んでnvm経由でバージョンを切り替える、といった連携もできます。miseやasdfと組み合わせる場合は、それらのツール側の `.envrc` フック機能を使う構成が一般的です。

```bash
# miseと連携する場合
eval "$(mise activate bash)"
```

## セキュリティ上の注意点

`.envrc` は任意のシェルコマンドを実行できるため、direnvは**未許可のディレクトリでは自動実行しない**設計になっています。他人のリポジトリをcloneして中に入っただけで勝手にコードが実行される、という事故を防ぐためです。

```bash
direnv: error .envrc is blocked. Run `direnv allow` to approve its content
```

このメッセージが出たら、必ず `.envrc` の中身を確認してから `direnv allow` しましょう。CIで自動化する場合も、事前に `direnv allow` をステップに含める必要があります。

## チーム運用でのTips

`.envrc` 自体はテンプレート的な内容だけをコミットし、秘密情報は `.env.local`（gitignore対象）に分離するのが定石です。

```bash
# .envrc（コミット対象）
dotenv_if_exists .env.local
export NODE_ENV="development"

# .env.local（gitignore対象、各自が用意）
API_KEY="実際のキー"
```

新メンバーのオンボーディングでは「`.env.local.example` をコピーして値を埋めるだけ」という手順書にでき、環境構築の属人化を減らせます。

## まとめ

- direnvはディレクトリ単位で環境変数を自動読み込み・破棄する
- dotenvとの連携で既存の `.env` 資産をそのまま活かせる
- `layout` / `use` で言語ランタイムの仮想環境切り替えも統合できる
- 未許可ディレクトリでは自動実行されないセキュリティ設計になっている

「.bashrcに書いたグローバル環境変数がプロジェクトを跨いで混線する」という悩みがあるなら、導入コストの割に効果が大きいツールです。
