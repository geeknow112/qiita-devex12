---
title: AWS DLT（Distributed Load Testing）で大規模負荷テストを簡単に実現する
tags:
  - AWS
  - JMeter
  - CloudFormation
  - 負荷テスト
  - Fargate
private: false
updated_at: '2026-06-07T22:03:40+09:00'
id: 1ea4e0031203d605e341
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに

https://youtu.be/tgTRH8Ai-Y8

Webアプリケーションをリリースする前に「本番環境で何人のユーザーに耐えられるか」を把握しておきたい場面は多いです。しかし、大規模な負荷テストを自前で構築するのは、サーバーの準備やツールの設定など手間がかかります。

**AWS Distributed Load Testing on AWS（DLT）** は、CloudFormationテンプレート一発でデプロイでき、Webコンソールから簡単に数千〜数万の同時接続をシミュレートできるAWSソリューションです。

この記事では、DLTの仕組みから実際のデプロイ、テスト実行までを解説します。

## DLTの特徴

| 特徴 | 説明 |
|---|---|
| **サーバーレス** | ECS Fargate上でテストを実行。サーバー管理不要 |
| **スケーラブル** | タスク数を増やすだけで負荷を増加可能 |
| **JMeter対応** | 既存のJMeterスクリプトをそのまま使用可能 |
| **Webコンソール付属** | GUIでテスト作成・実行・結果確認が完結 |
| **マルチリージョン** | 複数リージョンからの負荷生成に対応 |

## アーキテクチャ

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Web Console    │────▶│  API Gateway │────▶│  Lambda         │
│  (CloudFront+S3)│     │              │     │  (テスト管理)    │
└─────────────────┘     └──────────────┘     └────────┬────────┘
                                                      │
                                                      ▼
┌─────────────────┐     ┌──────────────┐     ┌─────────────────┐
│  DynamoDB       │◀────│ Step Functions│────▶│  ECS Fargate    │
│  (テスト結果)    │     │  (オーケスト) │     │  (Taurus+JMeter)│
└─────────────────┘     └──────────────┘     └─────────────────┘
```

内部ではオープンソースの**Taurus**が動作しており、JMeter、K6、Locustなどのテストフレームワークをサポートしています。

## デプロイ手順

### 1. CloudFormationでデプロイ

AWSコンソールから以下のURLにアクセスし、「Launch Stack」をクリックします。

https://aws.amazon.com/solutions/implementations/distributed-load-testing-on-aws/

または、AWS CLIでデプロイする場合：

```bash
# テンプレートのダウンロード
curl -O https://s3.amazonaws.com/solutions-reference/distributed-load-testing-on-aws/latest/distributed-load-testing-on-aws.template

# スタックの作成
aws cloudformation create-stack \
  --stack-name distributed-load-testing \
  --template-body file://distributed-load-testing-on-aws.template \
  --parameters \
    ParameterKey=AdminEmail,ParameterValue=your-email@example.com \
    ParameterKey=ExistingVPCId,ParameterValue=vpc-xxxxxxxx \
    ParameterKey=ExistingSubnetA,ParameterValue=subnet-xxxxxxxx \
    ParameterKey=ExistingSubnetB,ParameterValue=subnet-yyyyyyyy \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

**主要なパラメータ：**

| パラメータ | 説明 |
|---|---|
| `AdminEmail` | ログイン用のメールアドレス |
| `ExistingVPCId` | 使用するVPC ID |
| `ExistingSubnetA/B` | インターネット接続可能なサブネット |

デプロイには約15分かかります。

### 2. 初回ログイン

デプロイ完了後、指定したメールアドレスに一時パスワードが届きます。

CloudFormationの出力タブから `ConsoleUrl` を確認し、Webコンソールにアクセスしてログインします。

## テストの作成と実行

### シンプルなHTTPテスト

Webコンソールから「Create Test」をクリックし、以下を設定します。

```yaml
Name: api-load-test
Description: APIエンドポイントの負荷テスト

# テスト設定
Task Count: 10          # Fargateタスク数
Concurrency: 100        # タスクあたりの同時接続数
Ramp Up: 60             # 負荷を上げるまでの秒数
Hold For: 300           # 負荷を維持する秒数

# ターゲット
HTTP Method: GET
Target URL: https://api.example.com/health
```

この設定では、**10タスク × 100同時接続 = 1,000同時ユーザー**をシミュレートします。

### JMeterスクリプトを使う

より複雑なシナリオはJMeterスクリプトで定義できます。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="API Test">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments"/>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Users">
        <intProp name="ThreadGroup.num_threads">100</intProp>
        <intProp name="ThreadGroup.ramp_time">30</intProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
        <stringProp name="ThreadGroup.duration">300</stringProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="GET /api/users">
          <stringProp name="HTTPSampler.domain">api.example.com</stringProp>
          <stringProp name="HTTPSampler.protocol">https</stringProp>
          <stringProp name="HTTPSampler.path">/api/users</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
        </HTTPSamplerProxy>
        <HeaderManager guiclass="HeaderPanel" testclass="HeaderManager" testname="Headers">
          <collectionProp name="HeaderManager.headers">
            <elementProp name="Authorization" elementType="Header">
              <stringProp name="Header.name">Authorization</stringProp>
              <stringProp name="Header.value">Bearer ${__P(token,)}</stringProp>
            </elementProp>
          </collectionProp>
        </HeaderManager>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

このスクリプトをWebコンソールからアップロードして実行できます。

## テスト結果の見方

テスト完了後、以下のメトリクスが確認できます。

| メトリクス | 説明 |
|---|---|
| **Avg Response Time** | 平均レスポンスタイム |
| **Percentiles (p50, p90, p99)** | パーセンタイル別のレスポンスタイム |
| **Requests per Second** | 秒間リクエスト数 |
| **Success/Error Count** | 成功・エラー数 |
| **Bandwidth** | データ転送量 |

> **ポイント**: p99（99パーセンタイル）が重要です。平均値が良くても、p99が悪いとユーザー体験に影響します。

## コスト最適化のポイント

DLTの主なコストはECS Fargateのタスク実行時間です。

```
月間コスト目安（東京リージョン）:
- 1時間のテスト（10タスク）: 約 $3〜5
- 週1回、1時間のテスト: 約 $15〜20/月
```

**コスト削減Tips：**

1. **Fargate Spotを使う** - テスト環境なら中断されても問題なし
2. **テスト後にリソースを削除** - 使わないときはスタックごと削除
3. **適切なタスク数を設定** - 過剰な負荷は不要なコストに

## トラブルシューティング

### テストが開始されない

```bash
# ECSタスクのログを確認
aws logs filter-log-events \
  --log-group-name /ecs/distributed-load-testing \
  --filter-pattern "ERROR"
```

よくある原因：
- サブネットにインターネット接続がない（NAT Gateway必要）
- セキュリティグループでアウトバウンドが制限されている

### 負荷が期待通りに上がらない

Fargateタスクのリソース制限を確認します。デフォルトは2 vCPU / 4GB メモリですが、大規模テストでは増やす必要があるかもしれません。

## まとめ

AWS DLTを使えば、以下のメリットがあります：

- **15分でデプロイ完了** - CloudFormation一発
- **サーバー管理不要** - Fargateでフルマネージド
- **既存資産を活用** - JMeterスクリプトがそのまま使える
- **スケーラブル** - タスク数を増やすだけで負荷増加

本番リリース前の負荷テストや、定期的なパフォーマンス監視に活用してみてください。

## 参考リンク

- [AWS Solutions - Distributed Load Testing on AWS](https://aws.amazon.com/solutions/implementations/distributed-load-testing-on-aws/)
- [Implementation Guide](https://docs.aws.amazon.com/solutions/latest/distributed-load-testing-on-aws/solution-overview.html)
- [GitHub Repository](https://github.com/aws-solutions/distributed-load-testing-on-aws)
