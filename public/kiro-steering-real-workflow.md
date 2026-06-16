---
title: 【実録】Kiroに「質問するな、推論しろ」と教えたら作業時間が1/3になった
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

## 結論から

Kiroの設定ファイル（Steering）を整備したら、**同じ作業が1/3の時間で終わるようになった**。

具体的な数字で言うと：

| 作業 | Before | After |
|---|---|---|
| メディア記事投稿 | 45分 | 15分 |
| 毎回の説明 | 5〜10回 | 0回 |
| 「どのリポジトリ？」 | 毎回聞かれる | 聞かれない |

この記事では、実際に使っているSteering設定を**そのまま公開**する。

## 前提：Kiroとは

AWS公式のAIコーディングアシスタント。Claude搭載で、VS Code上で動く。

https://kiro.dev

特徴は「Steering」という仕組み。`.kiro/steering/` に置いたMarkdownファイルが、AIへの永続的な指示になる。

## Before：何が問題だったか

毎日、Zenn/Qiita/note/WordPressに記事を投稿している。

Kiroに「メディア更新して」と言うと、こうなる：

```
私：メディア更新して
Kiro：どのプラットフォームですか？
私：Zenn
Kiro：リポジトリはzenn-contentとzenn-devex12のどちらですか？
私：zenn-devex12
Kiro：何を更新しますか？新規記事？既存記事の修正？
私：新規
Kiro：どんなテーマで書きますか？
```

**毎回これ。**

AIは丁寧に確認してくれるが、作業効率は最悪。

## After：Steeringで「質問禁止」にした

`.kiro/steering/media-update.md` を作成：

```markdown
---
inclusion: manual
---

# メディア更新作業

「メディア更新」と言われたら、以下の手順で作業を開始する。

## 対象リポジトリ
- Zenn: `zenn-devex12`
- Qiita: `qiita-devex12`
- note: 手動投稿（下書きは `memo/draft.note.*.md`）
- WordPress: 手動投稿

## 作業の流れ
1. `memo/` フォルダの最新メモと下書き(`draft.*.md`)を確認
2. `memo.zenn.*.md`, `memo.qiita.*.md` で前回の進捗を把握
3. 前回の続きから執筆・投稿作業を進める
4. **質問せず、メモに記載されている情報で判断して進める**
```

ポイントは最後の一行。

**「質問せず、推論して進める」**

これを書くだけで、AIの動きが激変する。

## 実際の会話（After）

```
私：#media-update メディア更新
Kiro：（memoフォルダを確認）
Kiro：（前回の進捗を確認）
Kiro：前回はZenn/QiitaのGo記事が完了。noteの下書きがあるので投稿を進めます。
```

**質問ゼロ。**

## Steeringの書き方：3つのコツ

### 1. 「質問するな」を明記する

AIは親切心で質問してくる。それを止めるには明示的に書く。

```markdown
## 作業ルール
- 不明点があっても質問せず、推論して進める
- 迷ったらmemoフォルダを確認する
- それでも不明ならより安全な選択肢を取る
```

### 2. ファイルパスは具体的に

❌ 悪い例：
```markdown
設定ファイルを確認してから作業
```

✅ 良い例：
```markdown
`memo/memo.YYYYMMDD.md` で当日の作業を確認
`memo/memo.zenn.*.md` でZennの進捗を確認
```

AIが「どのファイル？」と聞く理由は、パスが曖昧だから。

### 3. inclusion: manual で呼び出し式に

```markdown
---
inclusion: manual
---
```

これを書くと、チャットで `#ファイル名` と打たないと読み込まれない。

常に読み込む設定（`inclusion: auto`）もあるが、作業別の指示は `manual` が良い。コンテキストを無駄に消費しない。

## 実際に使っているSteering（全文公開）

### user-preferences.md（auto）

```markdown
---
inclusion: auto
---

# ユーザー設定・作業ルール

## メモファイルへの追記ルール
- メモファイル（`memo.*.md`など）への追記は、必ずユーザーに確認してから行う
- 勝手に追記しない

## コミュニケーション
- 敬語を使用すること
- 簡潔に回答すること
```

### media-update.md（manual）

```markdown
---
inclusion: manual
---

# メディア更新作業

## 対象リポジトリ
- Zenn: `zenn-devex12`
- Qiita: `qiita-devex12`
- note: 手動投稿（下書きは `memo/draft.note.*.md`）
- WordPress: 手動投稿

## 作業の流れ
1. `memo/` フォルダの最新メモと下書きを確認
2. 前回の続きから執筆・投稿作業を進める
3. 質問せず、メモに記載されている情報で判断して進める

## 記事の品質基準
- 一般論のゴミ記事は書かない
- 一球入魂で1000PV狙える記事を書く
- 他にない独自の視点・実体験を入れる
- 具体的な数字を含める
```

## よくある質問

### Q. Claude CodeのCLAUDE.mdと何が違う？

Claude Codeは `CLAUDE.md` 1ファイルに全部書く。

Kiroは複数ファイルに分けられる＋ `inclusion` で読み込みタイミングを制御できる。

作業別に分けたいならKiroのSteeringのほうが柔軟。

### Q. AIが指示を無視することはある？

ある。特に長いチャットになると忘れる。

対策：
- 重要な指示は冒頭に書く
- 作業ごとにチャットをリセットする
- `#ファイル名` で明示的に再読み込みさせる

### Q. どこまで書けばいい？

「毎回説明していること」をすべて書く。

2回以上説明したことは、Steeringに書くべき。

## まとめ

Kiroを賢く使うコツ：

1. **「質問するな、推論しろ」を明記する**
2. **ファイルパスは具体的に書く**
3. **作業別にSteeringを分けて、manualで呼び出す**

これだけで、作業時間が1/3になる。

設定に30分かけても、1週間で元が取れる計算だ。

まだSteering書いてないなら、今日から始めてみてほしい。
