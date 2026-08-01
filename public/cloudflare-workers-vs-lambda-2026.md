---
title: 【2026年版】Cloudflare Workers vs AWS Lambda エッジコンピューティング比較
tags:
  - CloudflareWorkers
  - AWSLambda
  - Edge
  - Serverless
  - Cloud
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

サーバーレスの選択肢として長年AWS Lambdaが定番でしたが、Cloudflare Workersのようなエッジコンピューティング基盤も実務で無視できない存在になっています。この記事では、レイテンシ・コールドスタート・料金体系の観点で両者を比較します。

## 実行モデルの違い

**AWS Lambda** はリージョンベースで、指定したリージョンのデータセンターで関数が実行されます。

```javascript
export const handler = async (event) => {
  return { statusCode: 200, body: JSON.stringify({ message: "hello" }) };
};
```

**Cloudflare Workers** はグローバルに分散された300以上のエッジロケーションで実行され、リクエストは最寄りのロケーションにルーティングされます。

```javascript
export default {
  async fetch(request, env, ctx) {
    return new Response(JSON.stringify({ message: "hello" }));
  },
};
```

「ユーザーの近くで実行する」ことが前提のWorkersに対し、Lambdaは「特定リージョンで確実に実行する」ことが前提という設計思想の違いがあります。

## コールドスタート

- **Lambda**: V8/Node.jsランタイムでコンテナ起動が必要なため、数百ms〜1秒程度のコールドスタートが発生しうる（Provisioned Concurrencyで緩和可能だが追加コスト）
- **Workers**: V8 Isolateという軽量な分離単位で実行されるため、コールドスタートは実質ゼロに近い（数ms単位）

低レイテンシが要求されるAPIのフロントに置く処理（認証チェック、A/Bテスト振り分けなど）は、Workersの方が体感差が大きく出やすい領域です。

## 実行時間・リソース制限

| 項目 | AWS Lambda | Cloudflare Workers |
|---|---|---|
| 最大実行時間 | 最大15分 | 標準プランは数十ms〜数秒（Cron Triggersなど別枠あり） |
| メモリ | 最大10,240MB | 128MB前後 |
| 対応言語 | Node.js, Python, Go, Java, Rust等 | JavaScript/TypeScript, Rust(WASM), Python(beta) |

Lambdaは「重い処理を長時間かけて実行する」ことも可能ですが、Workersは「軽量な処理を大量のリクエストに対して高速に返す」ことに最適化されています。バッチ処理や機械学習推論のような重量級ワークロードはLambda、APIゲートウェイ的な軽量処理はWorkersという住み分けが自然です。

## 料金体系

- **Lambda**: リクエスト数 + 実行時間（GB秒）で課金。無料枠は月100万リクエスト
- **Workers**: リクエスト数ベースの課金がメインで、実行時間はCPU時間のみ計測（I/O待ちは課金対象外）。無料枠は1日10万リクエスト

I/O待ち（外部API呼び出しなど）が多い処理では、Workersの「CPU時間のみ課金」というモデルがコスト面で有利に働くことがあります。

## エコシステムとの統合

LambdaはAWSサービス（S3, DynamoDB, SQS, EventBridge等）とのネイティブ統合が豊富で、既存のAWS基盤に組み込みやすい構成です。WorkersはCloudflare D1（SQLite互換DB）、KV、R2（オブジェクトストレージ）、Durable Objectsなど独自エコシステムを持ち、Cloudflare完結でフルスタックを組める設計になっています。

## 選定基準

| 観点 | Lambdaが向く | Workersが向く |
|---|---|---|
| 既存のAWSインフラとの統合が必要 | ◎ | △ |
| グローバルな低レイテンシが最優先 | ○ | ◎ |
| 重量級のバッチ・推論処理 | ◎ | △ |
| 軽量なAPIゲートウェイ・認証処理 | ○ | ◎ |

## まとめ

- LambdaはAWSエコシステムとの統合と重量級処理への対応力が強み
- Workersはグローバル分散実行とほぼゼロのコールドスタートが強み
- 「どこで、どれくらいの重さの処理を、誰の近くで実行したいか」で選定するのが実践的

両者を併用し、認証やリダイレクトのような軽量処理はWorkers、本体のビジネスロジックはLambda、というハイブリッド構成をとるアーキテクチャも増えています。
