---
title: 【コピペOK】KiroのSteering設定で「質問攻め」を止める方法
tags:
  - AI
  - VSCode
  - プロンプトエンジニアリング
  - Kiro
  - 開発効率化
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## この記事でできること

KiroのSteering設定をコピペするだけで、以下が実現できます。

- AIの「質問攻め」を止める
- 定型作業を一言で開始できる
- チャットごとに同じ説明を繰り返さなくて済む

## 環境

- Kiro（VS Code拡張）
- `.kiro/steering/` ディレクトリ

## Steeringとは

Kiroに対する永続的な指示を書くMarkdownファイルです。

```
.kiro/
└── steering/
    ├── user-preferences.md  # 常に読み込む設定
    └── my-workflow.md       # 呼び出し式の設定
```

## 1. 基本設定（常に読み込む）

まずは全チャットで適用される基本設定を作成します。

```markdown:.kiro/steering/user-preferences.md
---
inclusion: auto
---

# ユーザー設定

## 作業ルール
- 不明点があっても質問せず、推論して進める
- 迷ったら `memo/` フォルダを確認する
- それでも不明ならより安全な選択肢を取る

## コミュニケーション
- 敬語を使用すること
- 簡潔に回答すること

## 禁止事項
- メモファイルへの追記は、必ずユーザーに確認してから行う
- 勝手にファイルを削除しない
```

### 設定のポイント

| 項目 | 説明 |
|---|---|
| `inclusion: auto` | 常に読み込まれる |
| 作業ルール | 「質問せず推論」がキモ |
| 禁止事項 | 書き込み系は確認必須にする |

## 2. 作業別設定（呼び出し式）

次に、特定の作業用の設定を作成します。

```markdown:.kiro/steering/my-workflow.md
---
inclusion: manual
---

# 定型作業

「作業を進めて」と言われたら、以下の手順で作業を開始する。

## 前提
- 作業内容は `memo/memo.YYYYMMDD.md` に記載されている
- 前回の進捗も同ファイルに記録されている

## 作業の流れ
1. `memo/memo.YYYYMMDD.md` で当日の作業を確認
2. 前回の進捗を把握
3. 続きから作業を進める
4. 質問せず、メモに記載されている情報で判断して進める

## 完了条件
- 作業が完了したら、完了した内容を報告する
- 追加の指示がなければ次のタスクに進む
```

### 設定のポイント

| 項目 | 説明 |
|---|---|
| `inclusion: manual` | `#ファイル名` で呼び出したときだけ読み込む |
| 前提 | AIが参照すべきファイルを明記 |
| 作業の流れ | 具体的な手順を書く |

## 3. 使い方

### 基本の呼び出し

```
# チャットで入力
#my-workflow 作業を進めてください
```

`#my-workflow` でSteering設定を読み込み、続けて指示を出します。

### 複数のSteeringを組み合わせる

```
#my-workflow #code-review PRをレビューしてください
```

## 4. inclusion の種類と使い分け

| 値 | 動作 | 用途 |
|---|---|---|
| `auto` | 常に読み込む | 基本設定、コミュニケーションルール |
| `manual` | `#ファイル名` で呼び出し | 作業別の設定、プロジェクト固有の設定 |

### auto を使いすぎない

`auto` が多いとコンテキストを圧迫します。基本設定以外は `manual` にして、必要なときだけ呼び出すのがおすすめです。

## 5. よくあるパターン

### コードレビュー用

```markdown:.kiro/steering/code-review.md
---
inclusion: manual
---

# コードレビュー

## レビュー観点
- バグの可能性
- パフォーマンス問題
- セキュリティリスク
- 可読性

## 出力形式
- 問題の重要度（高/中/低）
- 該当箇所
- 修正案
```

### Git操作用

```markdown:.kiro/steering/git-workflow.md
---
inclusion: manual
---

# Git操作

## コミットメッセージ
- 日本語で書く
- プレフィックスを付ける（feat:, fix:, refactor:, docs:）

## ブランチ名
- feature/機能名
- fix/バグ名

## 禁止事項
- mainへの直接push禁止
- force push禁止
```

### ドキュメント作成用

```markdown:.kiro/steering/documentation.md
---
inclusion: manual
---

# ドキュメント作成

## スタイル
- 敬語で書く
- 箇条書きを活用
- コードブロックにはファイル名を付ける

## 構成
1. 概要
2. 前提条件
3. 手順
4. トラブルシューティング
```

## 6. トラブルシューティング

### AIが指示を忘れる

長いチャットになると、AIがSteeringの内容を忘れることがあります。

```
# 再読み込み
#user-preferences 上記の設定を思い出してください
```

### 設定が反映されない

1. ファイルパスを確認（`.kiro/steering/` 配下にあるか）
2. front matter（`---`で囲まれた部分）の書式を確認
3. `inclusion` の値を確認

### auto設定が多すぎてコンテキストが溢れる

不要な `auto` 設定を `manual` に変更します。

```markdown
# Before
---
inclusion: auto
---

# After
---
inclusion: manual
---
```

## 7. Tips

### ファイルパスは具体的に書く

```markdown
# ❌ AIが「どのファイル？」と聞いてくる
設定ファイルを確認してから作業

# ✅ 質問されない
`memo/memo.YYYYMMDD.md` で当日の作業を確認
`config/settings.json` で設定値を確認
```

### 禁止事項は明確に

```markdown
# ❌ 曖昧
慎重に作業する

# ✅ 明確
- 確認なしにファイルを削除しない
- 本番環境への変更は必ず確認を取る
```

### 出力形式を指定する

```markdown
## 報告形式
- 完了したタスク
- 変更したファイル
- 次にやるべきこと
```

## まとめ

Kiro Steeringの設定ポイントは以下の3点です。

1. **基本設定は `auto`、作業別は `manual`**
2. **「質問せず推論して進める」を明記**
3. **ファイルパスは具体的に書く**

コピペして使えるテンプレートを用意したので、ぜひ活用してください。

## 参考

- [Kiro公式](https://kiro.dev)
