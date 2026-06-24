---
title: 【コピペOK】Kiro/Claude Codeの「信頼済みコマンド」設定で調査作業の承認を完全自動化する
tags:
  - 自動化
  - AI
  - VSCode
  - 開発効率化
  - Kiro
private: false
updated_at: '2026-06-24T21:19:33+09:00'
id: 894b16b584ab0e83dce5
organization_url_name: null
slide: false
ignorePublish: false
---

## この記事で解決できる問題

AIエージェント（Kiro / Claude Code）を使っていて、こんな経験ありませんか？

```
AI: grep -r "auth" src/ を実行してもいいですか？
私: OK
AI: ls -la src/auth/ を実行してもいいですか？
私: OK
AI: cat src/auth/login.ts を実行してもいいですか？
私: OK
AI: head -50 src/auth/middleware.ts を実行してもいいですか？
私: OK
（以下、延々と続く）
```

**調査系コマンドは読み取り専用で破壊的ではない**のに、毎回承認を求められます。

1回の調査タスクで20回以上の承認。これでは効率化どころか、逆に面倒です。

この記事では、**調査系コマンドの承認を完全にゼロにする設定**をコピペできる形で提供します。

## 解決策：信頼済みコマンド（Trusted Commands）を設定する

Kiro / Claude Codeには「このコマンドは信頼できる」と事前に登録する機能があります。

登録されたコマンドは**承認なしで自動実行**されます。

## Claude Code：settings.json の設定

### 設定ファイルの場所

| スコープ | ファイルパス | 用途 |
|---------|-------------|------|
| グローバル | `~/.claude/settings.json` | 全プロジェクト共通で適用 |
| プロジェクト | `.claude/settings.json` | 特定プロジェクトのみ適用 |
| ローカル | `.claude/settings.local.json` | 個人設定（gitignore推奨） |

### コピペ用：調査系コマンド許可リスト

`~/.claude/settings.json` に以下を貼り付けてください。

```json
{
  "permissions": {
    "allow": [
      "Bash(grep:*)",
      "Bash(rg:*)",
      "Bash(ag:*)",
      "Bash(ls:*)",
      "Bash(cat:*)",
      "Bash(head:*)",
      "Bash(tail:*)",
      "Bash(find:*)",
      "Bash(wc:*)",
      "Bash(du:*)",
      "Bash(df:*)",
      "Bash(which:*)",
      "Bash(whereis:*)",
      "Bash(type:*)",
      "Bash(file:*)",
      "Bash(stat:*)",
      "Bash(ps:*)",
      "Bash(top -b -n 1:*)",
      "Bash(free:*)",
      "Bash(diff:*)",
      "Bash(sort:*)",
      "Bash(uniq:*)",
      "Bash(cut:*)",
      "Bash(awk:*)",
      "Bash(sed -n:*)",
      "Bash(jq:*)",
      "Bash(yq:*)",
      "Bash(tree:*)",
      "Bash(aws * describe-*)",
      "Bash(aws * get-*)",
      "Bash(aws * list-*)",
      "Bash(aws logs filter-log-events:*)",
      "Bash(aws logs get-log-events:*)",
      "Bash(aws logs describe-log-groups:*)",
      "Bash(aws logs describe-log-streams:*)",
      "Bash(aws s3 ls:*)",
      "Bash(aws cloudwatch get-metric-statistics:*)",
      "Bash(git status:*)",
      "Bash(git log:*)",
      "Bash(git diff:*)",
      "Bash(git show:*)",
      "Bash(git branch:*)",
      "Bash(git remote -v:*)",
      "Bash(git config --list:*)",
      "Bash(git blame:*)",
      "Bash(git rev-parse:*)",
      "Bash(gh pr list:*)",
      "Bash(gh pr view:*)",
      "Bash(gh pr diff:*)",
      "Bash(gh pr checks:*)",
      "Bash(gh issue list:*)",
      "Bash(gh issue view:*)",
      "Bash(gh run list:*)",
      "Bash(gh run view:*)",
      "Bash(gh repo view:*)",
      "Bash(npm list:*)",
      "Bash(npm outdated:*)",
      "Bash(npm audit:*)",
      "Bash(yarn list:*)",
      "Bash(yarn outdated:*)",
      "Bash(yarn why:*)",
      "Bash(pip list:*)",
      "Bash(pip show:*)",
      "Bash(pip freeze:*)",
      "Bash(go list:*)",
      "Bash(go mod graph:*)",
      "Bash(go version:*)",
      "Bash(docker ps:*)",
      "Bash(docker images:*)",
      "Bash(docker logs:*)",
      "Bash(docker inspect:*)",
      "Bash(docker compose ps:*)",
      "Bash(docker compose logs:*)",
      "Bash(kubectl get:*)",
      "Bash(kubectl describe:*)",
      "Bash(kubectl logs:*)",
      "Bash(curl -s:*)",
      "Bash(curl -I:*)",
      "Bash(ping -c:*)",
      "Bash(nslookup:*)",
      "Bash(dig:*)",
      "Bash(netstat -an:*)",
      "Bash(ss -an:*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(rm -r *)",
      "Bash(rmdir *)",
      "Bash(git push -f *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)",
      "Bash(git clean -fd *)",
      "Bash(DROP *)",
      "Bash(DELETE FROM *)",
      "Bash(TRUNCATE *)",
      "Bash(chmod 777 *)",
      "Bash(chown *)",
      "Bash(mkfs *)",
      "Bash(dd *)",
      "Bash(curl * | sh)",
      "Bash(curl * | bash)",
      "Bash(wget * | sh)",
      "Bash(wget * | bash)"
    ]
  }
}
```

### 設定のカテゴリ別解説

#### ファイル検索・表示系

```json
"Bash(grep:*)",      // テキスト検索
"Bash(rg:*)",        // ripgrep（高速検索）
"Bash(ag:*)",        // The Silver Searcher
"Bash(ls:*)",        // ファイル一覧
"Bash(cat:*)",       // ファイル表示
"Bash(head:*)",      // 先頭表示
"Bash(tail:*)",      // 末尾表示
"Bash(find:*)",      // ファイル検索
"Bash(tree:*)"       // ディレクトリツリー
```

これらは**読み取り専用**のコマンドなので、許可しても安全です。

#### テキスト処理系

```json
"Bash(sort:*)",      // ソート
"Bash(uniq:*)",      // 重複除去
"Bash(cut:*)",       // カラム抽出
"Bash(awk:*)",       // テキスト処理
"Bash(sed -n:*)",    // パターン抽出（-n で出力のみ）
"Bash(jq:*)",        // JSON処理
"Bash(yq:*)"         // YAML処理
```

`sed` は `-n` オプション付きのみ許可しています。`sed -i`（ファイル書き換え）は許可していません。

#### AWS CLI系

```json
"Bash(aws * describe-*)",              // リソース情報取得
"Bash(aws * get-*)",                   // 設定取得
"Bash(aws * list-*)",                  // 一覧取得
"Bash(aws logs filter-log-events:*)", // ログ検索
"Bash(aws logs get-log-events:*)",    // ログ取得
"Bash(aws s3 ls:*)"                    // S3一覧
```

AWS CLIの**読み取り系API**のみ許可しています。`aws s3 rm`、`aws ec2 terminate-instances` などは許可していません。

#### Git系

```json
"Bash(git status:*)",    // 状態確認
"Bash(git log:*)",       // 履歴確認
"Bash(git diff:*)",      // 差分確認
"Bash(git show:*)",      // コミット内容
"Bash(git branch:*)",    // ブランチ一覧
"Bash(git blame:*)"      // 行ごとの変更者
```

Git操作は**ローカル参照のみ**を許可しています。`git push`、`git reset`、`git checkout` は許可していません。

#### GitHub CLI系

```json
"Bash(gh pr list:*)",    // PR一覧
"Bash(gh pr view:*)",    // PR詳細
"Bash(gh pr diff:*)",    // PR差分
"Bash(gh issue list:*)", // Issue一覧
"Bash(gh run list:*)",   // Workflow一覧
"Bash(gh run view:*)"    // Workflow詳細
```

これらは**GitHub APIの読み取り**なので安全です。

#### パッケージマネージャー系

```json
"Bash(npm list:*)",      // 依存関係一覧
"Bash(npm outdated:*)",  // 更新確認
"Bash(npm audit:*)",     // 脆弱性確認
"Bash(pip list:*)",      // Python依存関係
"Bash(go list:*)"        // Go依存関係
```

依存関係の**確認のみ**を許可。`npm install`、`pip install` は許可していません。

#### Docker / Kubernetes系

```json
"Bash(docker ps:*)",          // コンテナ一覧
"Bash(docker logs:*)",        // ログ確認
"Bash(docker inspect:*)",     // 詳細情報
"Bash(kubectl get:*)",        // リソース取得
"Bash(kubectl describe:*)",   // 詳細情報
"Bash(kubectl logs:*)"        // Podログ
```

コンテナ・クラスタの**状態確認のみ**を許可。`docker rm`、`kubectl delete` は許可していません。

### 明示的に拒否するコマンド

```json
"deny": [
  "Bash(rm -rf *)",           // 再帰削除
  "Bash(git push -f *)",      // 強制プッシュ
  "Bash(git reset --hard *)", // 強制リセット
  "Bash(DROP *)",             // テーブル削除
  "Bash(DELETE FROM *)",      // レコード削除
  "Bash(curl * | sh)"         // リモートスクリプト実行
]
```

これらは**破壊的な操作**なので、明示的に拒否リストに入れておきます。AIが誤って実行しようとしても、確実にブロックされます。

## Kiro：Trusted Commands の設定

### 設定場所

Settings → **Kiro Agent: Trusted Commands**

設定レベル：
- **User**：全ワークスペースで有効
- **Workspace**：特定プロジェクトのみ有効

### コピペ用：信頼済みコマンドリスト

以下を1行ずつ追加してください。

```
grep
rg
ag
ls
cat
head
tail
find
wc
du
df
which
whereis
type
file
stat
ps
free
diff
sort
uniq
cut
awk
jq
yq
tree
aws
git status
git log
git diff
git show
git branch
git remote -v
git config --list
git blame
git rev-parse
gh pr list
gh pr view
gh pr diff
gh pr checks
gh issue list
gh issue view
gh run list
gh run view
gh repo view
npm list
npm outdated
npm audit
yarn list
yarn outdated
yarn why
pip list
pip show
pip freeze
go list
go mod graph
go version
docker ps
docker images
docker logs
docker inspect
docker compose ps
docker compose logs
kubectl get
kubectl describe
kubectl logs
curl -s
curl -I
ping -c
nslookup
dig
netstat -an
ss -an
```

### Kiroの特徴：前方一致

Kiroは**文字列の前方一致**でマッチします。

| 設定 | マッチする例 |
|------|-------------|
| `grep` | `grep -r "auth" src/`、`grep -n "TODO" *.ts` |
| `aws` | `aws s3 ls`、`aws ec2 describe-instances` |
| `git status` | `git status`、`git status -s` |

`aws` だけ設定すれば、全AWSコマンドが許可されます。読み取り専用に絞りたい場合は `aws ec2 describe`、`aws s3 ls` などを個別に設定してください。

## ワイルドカードの使い方

### Claude Code

Claude Codeでは `:*` がワイルドカードです。

| パターン | マッチ例 |
|---------|---------|
| `Bash(grep:*)` | `grep -r "auth" src/` |
| `Bash(aws * describe-*)` | `aws ec2 describe-instances`、`aws rds describe-db-instances` |
| `Bash(git log:*)` | `git log --oneline -10` |

### 危険なパターンの例

```json
// ❌ 危険：git push -f も許可される
"Bash(git *)"

// ✅ 安全：読み取り系のみ許可
"Bash(git status:*)"
"Bash(git log:*)"
"Bash(git diff:*)"
```

`git *` のような**広すぎるパターンは絶対に避けてください**。`git push -f` や `git reset --hard` も許可されてしまいます。

## 設定の効果：Before / After

### Before（設定前）

```
私: 認証周りのコードを調査して

AI: grep -r "auth" src/ を実行してもいいですか？
私: OK
AI: ls src/auth/ を実行してもいいですか？
私: OK
AI: cat src/auth/login.ts を実行してもいいですか？
私: OK
AI: head -100 src/auth/middleware.ts を実行してもいいですか？
私: OK
AI: git log --oneline -20 -- src/auth/ を実行してもいいですか？
私: OK
AI: aws logs filter-log-events ... を実行してもいいですか？
私: OK
... (20回以上の承認)

所要時間：30分（大半は承認待ち）
```

### After（設定後）

```
私: 認証周りのコードを調査して

AI: （自動でgrep, ls, cat, git log, aws logsを実行）

AI: 調査完了しました。

【認証機能の概要】
- JWT + Refresh Token方式
- 主要ファイル: src/auth/login.ts, src/auth/middleware.ts
- 最近の変更: 6/20にトークン有効期限を30分→1時間に変更
- CloudWatchログ: 直近24時間でエラーなし

詳細レポートを出力しますか？

所要時間：5分
```

**承認回数：20回 → 0回**
**所要時間：30分 → 5分**

## プロジェクト別のカスタマイズ例

### フロントエンドプロジェクト

`.claude/settings.json`（プロジェクトルート）

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run lint:*)",
      "Bash(npm run typecheck:*)",
      "Bash(npx tsc --noEmit:*)",
      "Bash(npx eslint:*)",
      "Bash(npx prettier --check:*)"
    ]
  }
}
```

### インフラプロジェクト

```json
{
  "permissions": {
    "allow": [
      "Bash(terraform plan:*)",
      "Bash(terraform validate:*)",
      "Bash(terraform fmt -check:*)",
      "Bash(aws cloudformation validate-template:*)",
      "Bash(aws cloudformation describe-stacks:*)"
    ],
    "deny": [
      "Bash(terraform apply *)",
      "Bash(terraform destroy *)",
      "Bash(aws cloudformation delete-stack *)"
    ]
  }
}
```

### データベース関連プロジェクト

```json
{
  "permissions": {
    "allow": [
      "Bash(psql -c \"SELECT:*)",
      "Bash(psql -c \"EXPLAIN:*)",
      "Bash(mysql -e \"SELECT:*)",
      "Bash(mysql -e \"SHOW:*)",
      "Bash(mysql -e \"DESCRIBE:*)"
    ],
    "deny": [
      "Bash(psql -c \"DROP:*)",
      "Bash(psql -c \"DELETE:*)",
      "Bash(psql -c \"TRUNCATE:*)",
      "Bash(mysql -e \"DROP:*)",
      "Bash(mysql -e \"DELETE:*)"
    ]
  }
}
```

## トラブルシューティング

### 「設定したのに承認が出る」

**原因1：パターンが合っていない**

```json
// ❌ スペースの位置が違う
"Bash(aws logs get-log-events:*)"

// ✅ 正しいパターン
"Bash(aws logs:*)"
```

**原因2：設定ファイルの場所が違う**

- グローバル設定：`~/.claude/settings.json`
- プロジェクト設定：`.claude/settings.json`（リポジトリルート）

設定の優先順位：プロジェクト > グローバル

**原因3：JSON構文エラー**

```bash
# 構文チェック
cat ~/.claude/settings.json | jq .
```

### 「許可しすぎて怖い」

最小限から始めて、必要に応じて追加していくアプローチがおすすめです。

```json
// 最小構成（まずはこれだけ）
{
  "permissions": {
    "allow": [
      "Bash(grep:*)",
      "Bash(ls:*)",
      "Bash(cat:*)",
      "Bash(head:*)",
      "Bash(tail:*)"
    ]
  }
}
```

使っていて「これも許可したい」と思ったら追加していきます。

### 「設定を確認したい」

Claude Codeの場合：

```bash
# 現在の設定を確認
claude /permissions
```

## まとめ

| 設定 | 効果 |
|------|------|
| 調査系コマンドを許可 | 承認ゼロで調査が進む |
| 破壊的コマンドを拒否 | 安全性を担保 |
| プロジェクト別にカスタマイズ | 用途に応じた最適化 |

**1回設定すれば、すべての調査タスクで効果を発揮します。**

この記事の設定をコピペして、快適なAIエージェント生活を始めてください。

## 関連記事

- [【コピペOK】Kiroを「完全オート」で動かす設定ファイル集](https://qiita.com/geeknow112/items/3cb6df52d1b3ce963756)
