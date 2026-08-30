---
title: >-
  「AWSアカウント全体を1枚の構成図に」は幻想だった ― IaCジェネレーター×Infrastructure
  Composerで1819→451リソースを可視化して分かった5つの罠
tags:
  - AWS
  - CloudFormation
  - IaC
  - 個人開発
private: false
updated_at: '2026-08-30T13:54:07+09:00'
id: 9cf35726babe77532b39
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## TL;DR

- 個人のAWSアカウントの全体像を1枚の構成図で見たくて、CloudFormationの「IaCジェネレーター」→ Infrastructure Composer（旧Application Composer）を試した
- アカウントスキャンで1819件検出 → 標準リソースを除外して451件に絞っても、Composerは意味のある図にならず**縦一列**に積み上がるだけだった
- 原因はテンプレートに `Ref` / `GetAtt` の依存関係記述がほぼないこと。既存スタックのテンプレートを食わせれば関連リソースはまとまるが、期待していた「矢印で繋がるノードグラフ」ではなく**親カードに子リソースがネストされる**表示だった
- `aws cloudformation get-template --output yaml` はPythonの `!!omap` タグ付きYAMLを吐いて壊れる。`--output json` を使うこと
- 数十万文字級のテンプレートをComposerに流し込むには、クリップボード経由のCtrl+A/Ctrl+V が有効

## 背景・課題

個人でAWSアカウントを何年か使っていると、Lambda・S3・DynamoDB・IAMロールなどがちょこちょこ増えて、コンソールを行き来しないと全体像が把握できなくなる。

そんな折、CloudFormationに「IaCジェネレーター」というアカウント内の既存リソースをスキャンしてテンプレート化してくれる機能があると知った。生成したテンプレートを Infrastructure Composer に読み込ませれば、アカウント全体の構成図が一発で見られるのでは——と思って試した記録。結論から言うと、この組み合わせは「アカウント全体の俯瞰図」用途には向いていなかった。

## やったこと

### 1. 環境準備

AWS CLIが未インストールだったので導入し、個人アカウント（Organizations未使用）のためアクセスキー方式で認証した。

```powershell
winget install -e --id Amazon.AWSCLI
aws configure
```

### 2. アカウント内リソースをスキャン

```bash
aws cloudformation start-resource-scan
aws cloudformation describe-resource-scan --resource-scan-id <arn>
aws cloudformation list-resource-scan-resources --resource-scan-id <arn> --max-items 2000
```

→ **1819件**のリソースを検出。

### 3. 標準リソースを除外して絞り込み

`AWS::SSM::Document` のAWS製ドキュメント、`AWS::RAM::Permission` のAWS所有ARN、`CodeDeployDefault.*`、ElastiCache/RDS/MemoryDB/Neptuneの `default.*` パラメータグループ、`AWSServiceRoleFor*` のサービスリンクロールなど、AWSアカウントに最初から存在する既定リソースを除外。

→ **451件**まで絞り込み。

### 4. テンプレート生成

```bash
aws cloudformation create-generated-template --generated-template-name <name> --resources file://resources.json
aws cloudformation describe-generated-template --generated-template-name <arn>
aws cloudformation get-generated-template --generated-template-name <arn> --format YAML --query TemplateBody --output text
```

1テンプレートあたり最大500リソースという制限があり、451件はギリギリ収まる数だった。

## つまずいたポイント

### ① 「Application Composer」は「Infrastructure Composer」に名称変更されていた

コンソール上でボタン名を探しても見つからなかった。`find` ツールで検索して初めて判明した。旧名前提で書かれた手順書は古くなっている可能性がある。

### ② 451件はComposerで意味のある図にならない

Composer自身が「大量のリソースセットの視覚化については限定的なサポート」という趣旨の警告を出す。実際に読み込ませると、ノードは横に広がらず**縦一列に積み重なるだけ**だった。

### ③ 「実際に課金・稼働しているリソース」（107件）に絞っても変わらない

Lambda / S3 / DynamoDB / CloudFront / API Gatewayなど、実運用中のリソースだけに絞り込んでも縦一列表示は改善しなかった。原因は④。

### ④ 関連図になるかは、テンプレートの依存関係表現に依存する

- IaCジェネレーターが生成したテンプレート（アカウントスキャン由来）は各リソースをフラットに列挙するだけで、`Ref` / `GetAtt` による依存関係の記述がほぼない → Composerは繋ぎようがなく縦一列になる
- 一方、**既存のCloudFormationスタック**（人間 or SAM CLIが書いた本来のテンプレート）を以下のコマンドで取得すると `Ref` / `GetAtt` が保持されている

```bash
aws cloudformation get-template --stack-name <name> --query TemplateBody --output json
```

→ Composerで関連リソースがカードにまとまって表示される。ただし実際の表示は、期待していた「矢印線で繋がるノードグラフ」ではなく、**親リソースのカード内に関連する子リソース（IAM Role、Permission、Deploymentなど）がリスト形式でネストされる**というものだった。

### ⑤ `--output yaml` は使えない

`aws cloudformation get-template --output yaml` は、PythonのYAMLライブラリ由来の `!!omap` タグ付きYAMLを出力する。これはCloudFormation標準テンプレート構文としては不正で、Composerに貼り付けると**エラーも出さず無言で無視される**。`--output json` を使うこと。

### ⑥ 大きいテンプレートをComposerに流し込む方法

ブラウザの「ファイルを開く」はネイティブのファイル選択ダイアログを要求するため、そのままでは扱いにくい。代わりに以下の方法が有効だった。

```powershell
Get-Content -Raw <file> | Set-Clipboard
```

でOSクリップボードにコピーし、Composerのテンプレートエディタでフォーカス→Ctrl+A→Ctrl+Vで流し込む。数十万文字規模でも問題なく反映された。

## 得られた知見・まとめ

- アカウント全体を1枚の俯瞰図にする用途には、IaCジェネレーター＋Infrastructure Composerの組み合わせは不向き
- Composerが役立つのは「1つの既存スタック（アプリ単位）の構成をざっと確認する」用途まで
- 本格的な「矢印で繋がった関連図」が欲しい場合は、CloudFormationテンプレートの `Ref` / `GetAtt` を自前で解析してMermaid/Graphvizで描くほうが確実（今回は未実施、別アプローチとして今後の課題）

## 関連記事

既存リソースからテンプレートを生成する話とは逆に、テンプレートからスタックを作成・更新する基本操作（`aws cloudformation create-stack` / `update-stack`）は、こちらで手順をまとめています。

- [【aws cli】cloudformationスタックの作成と更新](https://hack-note.com/tools/aws_cli_cloudformation/)

## 参考リンク

- [AWS CloudFormation ユーザーガイド - 既存リソースからのIaC生成](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/generate-IaC.html)
- [Infrastructure Composer ドキュメント](https://docs.aws.amazon.com/infrastructure-composer/latest/dg/what-is-composer.html)

