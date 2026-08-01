---
title: 【2026年版】mise vs asdf ランタイムバージョン管理比較 - どちらでNode/Go/Pythonを管理すべきか
tags:
  - mise
  - asdf
  - CLI
  - DevEx
  - VersionManager
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Node.js・Python・Go・Rubyなど複数言語のバージョンをプロジェクトごとに切り替える定番ツールといえば `asdf` でした。しかし近年、Rust製で高速な `mise`（旧rtx）への乗り換えが進んでいます。この記事では両者を実務目線で比較します。

## そもそもの仕組み

両者とも「プラグイン形式で各言語のバージョンマネージャを統合する」という設計は共通しています。プロジェクト直下の設定ファイルにバージョンを書いておけば、ディレクトリに入った瞬間に自動でそのバージョンに切り替わります。

```bash
# asdf
echo "nodejs 20.11.0" >> .tool-versions
echo "golang 1.22.0" >> .tool-versions

# mise
echo "node = \"20.11.0\"" >> .mise.toml
echo "go = \"1.22.0\"" >> .mise.toml
```

miseはは `.tool-versions`（asdf互換フォーマット）もそのまま読めるため、既存asdfプロジェクトへの導入障壁は低く設計されています。

## 速度の違い

ここが乗り換えの最大の動機になっているポイントです。asdfはシェルスクリプトベースで実装されているため、シェル起動のたびにPATH解決のオーバーヘッドが発生します。miseはRust製バイナリで、同じ処理を大幅に高速化しています。

```bash
# 体感速度の目安（環境による）
time asdf current   # 数十〜百数十ms
time mise current    # 数ms程度
```

シェルのプロンプトが表示されるたびに走る処理なので、この差はターミナルを頻繁に開閉する開発者ほど体感しやすくなります。

## 環境変数管理としての機能

miseはバージョン管理だけでなく、`direnv` 相当の環境変数自動読み込み機能を内蔵しています。

```toml
# .mise.toml
[env]
DATABASE_URL = "postgres://localhost/myapp_dev"
NODE_ENV = "development"

[tools]
node = "20.11.0"
python = "3.12.1"
```

これにより「バージョン管理ツール」と「環境変数管理ツール（direnv）」を別々に導入する必要がなく、1つの設定ファイルで完結します。asdfは基本的にバージョン管理に特化しており、環境変数は別途direnvなどと組み合わせるのが一般的です。

## タスクランナー機能

miseにはさらに、簡易的なタスクランナー機能も内蔵されています。

```toml
[tasks.test]
run = "npm test"

[tasks.build]
run = "npm run build"
depends = ["test"]
```

`mise run build` で実行でき、`just` や `make` を別途入れなくても最低限のタスク定義がまかなえます（本格的なタスク管理が必要なら別ツールとの併用が現実的です）。

## プラグインエコシステム

asdfは長年の実績があり、プラグイン数（対応言語・ツール数）では依然として最大手です。miseはasdfプラグインの多くをそのまま利用できる互換レイヤーを持っているため、対応範囲の差は実用上ほぼ埋まってきています。

## 選定基準

| 観点 | asdfが向く | miseが向く |
|---|---|---|
| 既存asdf資産・チーム習熟度が高い | ◎ | ○（互換あり） |
| シェル起動速度を最優先したい | △ | ◎ |
| 環境変数管理も一元化したい | △ | ◎ |
| 枯れた実績・情報量を重視する | ◎ | ○ |

## まとめ

- asdfは実績とプラグインエコシステムの厚みが強み
- miseは速度・環境変数管理・タスクランナーを統合した「オールインワン」設計が強み
- `.tool-versions` 互換のため、asdfからmiseへの移行コストは比較的低い

新規プロジェクトであれば、速度と機能統合の観点でmiseを第一候補にする価値は十分にあります。
