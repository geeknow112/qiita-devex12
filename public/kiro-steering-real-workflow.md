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

KiroのSteering設定をコピペするだけで、AIの「質問攻め」を止められます。

## 環境

- Kiro（VS Code拡張）
- `.kiro/steering/` ディレクトリ

## 1. 基本設定（常に読み込む）

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
```

**ポイント**: `inclusion: auto` で常に読み込まれます。

## 2. 作業別設定（呼び出し式）

```markdown:.kiro/steering/my-workflow.md
---
inclusion: manual
---

# 定型作業

「作業を進めて」と言われたら、以下の手順で作業を開始する。

## 作業の流れ
1. `memo/memo.YYYYMMDD.md` で当日の作業を確認
2. 前回の続きから作業を進める
3. 質問せず、メモに記載されている情報で判断して進める
```

**ポイント**: `inclusion: manual` でチャットに `#my-workflow` と入力したときだけ読み込まれます。

## 3. 使い方

```
# チャットで入力
#my-workflow 作業を進めてください
```

これでAIが質問せずに作業を開始します。

## inclusion の種類

| 値 | 動作 |
|---|---|
| `auto` | 常に読み込む |
| `manual` | `#ファイル名` で明示的に呼び出し |

## Tips

### AIが忘れたら再読み込み

長いチャットになるとAIが指示を忘れることがあります。

```
# 再読み込み
#user-preferences
```

### ファイルパスは具体的に

```markdown
# ❌ 曖昧
設定ファイルを確認してから作業

# ✅ 具体的
`memo/memo.YYYYMMDD.md` で当日の作業を確認
```

## 参考

- [Kiro公式](https://kiro.dev)
