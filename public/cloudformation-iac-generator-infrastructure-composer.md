---
title: 「AWSアカウント全体を1枚の構成図に」は幻想だった ― IaCジェネレーター×Infrastructure Composerで1819→451リソースを可視化して分かった5つの罠
tags:
  - AWS
  - CloudFormation
  - IaC
  - 個人開発
private: false
updated_at: ''
id: null
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

## 数字で振り返る

| 段階 | 件数 | 備考 |
|---|---|---|
| アカウントスキャン直後 | 1819件 | `start-resource-scan` |
| 標準リソース除外後 | 451件 | 1テンプレート最大500の制限内に収まったのはたまたま |
| 実運用（課金・稼働）リソースのみ | 107件 | Lambda/S3/DynamoDB/CloudFront/API Gatewayなど |
| Composerで意味のある図になった件数 | 0件 | 451件でも107件でも縦一列は変わらず |

コンソールを手作業で1個ずつ見て回ることを考えれば、1819件→107件まで機械的に絞り込めた時点で十分価値はあった。ただし「1枚の相関図」という当初のゴールには届かなかった、というのが実態。

## 代替手段は検討したか

Composerに見切りをつけた後、以下のような選択肢も存在することを知った（いずれも今回は未検証。次に俯瞰図が欲しくなったら試す候補としてメモしておく）。

| ツール | アプローチ | 想定される強み | 想定される弱み |
|---|---|---|---|
| [Cartography](https://github.com/cartography-cncf/cartography) | AWS APIを直接叩いてNeo4jにグラフ投入 | 実際のリソース間関係（テンプレート由来ではない）をグラフDBで自由にクエリできる | Neo4j環境の構築が必要、個人用途にはオーバースペック気味 |
| Python `diagrams` ライブラリ | コードでアーキテクチャ図を書く | 依存関係を自分で明示するので必ず意図通りの図になる | 結局「手で書く」ので、451件規模の自動生成には向かない |
| Steampipe + Powerpipe | SQLでAWSリソースをクエリしてダッシュボード化 | 相関図というより一覧・集計向き | ノードグラフ的な可視化は主目的ではない |
| 自前でRef/GetAtt解析→Mermaid | テンプレートの `Ref` / `GetAtt` / `DependsOn` を自分でパースしてエッジリスト化 | IaCジェネレーターの出力をそのまま活かせる | 実装コストがかかる（今回は未着手） |

結論として、「今アカウントに何があるか」を知りたいだけなら107件までの絞り込みで十分実用的で、そこから先の「綺麗な相関図」を求めるなら、Composerではなく上記のような専用ツールに乗り換えるべきだった、というのが今回の学びです。

## こんな人にはおすすめ、こんな人にはおすすめしない

**おすすめ**
- 1つの既存スタック（SAMやCDKでデプロイしたアプリ単位）の構成をざっと確認したい人
- 新規にテンプレートを設計する際、リソース同士の関係を視覚的に確認しながら組み立てたい人

**おすすめしない**
- 個人アカウント全体（数百リソース規模）を1枚の図で俯瞰したい人 → Composerでは無理でした
- IaCジェネレーターの出力をそのまま「完成品の構成図」として使いたい人 → あくまで叩き台

## よくあるつまずきQ&A

**Q. `create-generated-template` がエラーで止まる**
A. 1テンプレートあたり最大500リソースの制限に引っかかっていないか確認する。超える場合はリソースをグループ分けして複数テンプレートに分割する。

**Q. Composerに貼り付けても何も表示されない**
A. まず `--output json` で取得し直しているか確認する（`--output yaml` は `!!omap` タグ付きで壊れる、上記⑤参照）。次に、テンプレートのJSON構文自体が正しいか `python3 -m json.tool` などで検証する。

**Q. スキャンにどれくらい時間がかかるか**
A. `start-resource-scan` は非同期実行で、`describe-resource-scan` のステータスが `COMPLETE` になるまでポーリングが必要。筆者の環境（リソース1819件規模）では実行のたびに待ち時間が発生した（具体的な所要時間は未計測、次回計測してこの記事に追記予定）。

## 得られた知見・まとめ

- アカウント全体を1枚の俯瞰図にする用途には、IaCジェネレーター＋Infrastructure Composerの組み合わせは不向き
- Composerが役立つのは「1つの既存スタック（アプリ単位）の構成をざっと確認する」用途まで
- 本格的な「矢印で繋がった関連図」が欲しい場合は、CloudFormationテンプレートの `Ref` / `GetAtt` を自前で解析してMermaid/Graphvizで描くか、Cartographyのような専用ツールに乗り換える方が確実（今回は未実施、別記事で試す予定）
- 1819件→451件→107件という絞り込みの過程自体は、手作業でコンソールを見て回るより圧倒的に速かった。「相関図」は諦めても「棚卸し」としての価値は十分にあった

## 参考リンク

- [AWS CloudFormation ユーザーガイド - 既存リソースからのIaC生成](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/generate-IaC.html)
- [Infrastructure Composer ドキュメント](https://docs.aws.amazon.com/infrastructure-composer/latest/dg/what-is-composer.html)

※本記事は2026年8月時点で個人アカウントを使って検証した実体験です。仕様は変更される可能性があるため、最新の挙動は公式ドキュメントでご確認ください。
