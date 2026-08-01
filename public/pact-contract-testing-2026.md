---
title: 【2026年版】Pactで学ぶコントラクトテスト入門 - マイクロサービス間の壊れやすさを解消する
tags:
  - Pact
  - ContractTest
  - Testing
  - Microservices
  - API
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

マイクロサービス構成では「フロントエンドとバックエンドのAPI仕様がいつの間にかズレていた」という事故が起きがちです。E2Eテストで検知することもできますが、サービス数が増えるほどE2E環境の構築・維持コストが跳ね上がります。コントラクトテストは、この問題を「サービス間の約束事（契約）」という単位でテストすることで解決するアプローチです。この記事では代表的なツールPactを使って基本を紹介します。

## コントラクトテストとは何か

従来のE2Eテストは「全サービスを実際に起動して統合的に確認する」アプローチです。これに対しコントラクトテストは、Consumer（利用側、例: フロントエンド）が「このAPIにこう呼びかけたら、こう返ってくるはずだ」という**契約(Contract)**を定義し、Provider（提供側、例: バックエンド）がその契約を満たしているかを個別に検証します。

```
Consumer側テスト → 契約ファイル(pact.json)を生成
                          ↓
Provider側テスト ← 契約ファイルを読んで実際のAPIと照合
```

サービス同士を同時に起動する必要がないため、CIが高速かつ疎結合になります。

## Consumer側: 契約を作る

フロントエンド（Consumer）側では、期待するリクエスト・レスポンスをPactのDSLで定義します。

```javascript
const { PactV3 } = require("@pact-foundation/pact");

const provider = new PactV3({ consumer: "WebApp", provider: "UserService" });

test("ユーザー取得API", async () => {
  provider
    .given("user with id 1 exists")
    .uponReceiving("a request for user 1")
    .withRequest({ method: "GET", path: "/users/1" })
    .willRespondWith({
      status: 200,
      body: { id: 1, name: "Taro" },
    });

  await provider.executeTest(async (mockServer) => {
    const res = await fetch(`${mockServer.url}/users/1`);
    expect((await res.json()).name).toBe("Taro");
  });
});
```

このテストを実行すると、モックサーバーに対する実際のHTTP通信を通じて `pact.json` という契約ファイルが自動生成されます。

## Provider側: 契約を検証する

バックエンド（Provider）側では、生成された契約ファイルを読み込み、実際のAPIサーバーがその契約を満たすか検証します。

```javascript
const { Verifier } = require("@pact-foundation/pact");

new Verifier({
  provider: "UserService",
  providerBaseUrl: "http://localhost:8080",
  pactUrls: ["./pacts/webapp-userservice.json"],
  stateHandlers: {
    "user with id 1 exists": async () => seedUser({ id: 1, name: "Taro" }),
  },
}).verifyProvider();
```

`stateHandlers` で「契約が前提とする状態」をテスト用DBにセットアップしてから、実サーバーに対してリクエストを飛ばし、期待通りのレスポンスが返るか確認します。

## Pact Brokerでチーム間連携

契約ファイルをConsumer/Provider双方のリポジトリで手動共有するのは非効率なので、実運用では **Pact Broker** というハブを使います。

- Consumer側CIが契約を生成 → Pact Brokerにpublish
- Provider側CIがPact Brokerから最新の契約を取得して検証
- 双方の検証結果がBroker上で可視化され、「このバージョンの組み合わせは互換性がある」というマトリクスが自動生成される

これにより「Providerの変更がどのConsumerに影響するか」がデプロイ前に機械的に判定できるようになります（Can I Deploy?機能）。

## E2Eテストとの使い分け

コントラクトテストはAPIの入出力契約を検証しますが、複数サービスをまたいだビジネスフロー全体の正しさは保証しません。実務では以下のように使い分けます。

| 観点 | コントラクトテスト | E2Eテスト |
|---|---|---|
| 実行速度 | 速い（モック中心） | 遅い（全サービス起動） |
| 検知できる問題 | APIスキーマの不整合 | ビジネスフロー全体の不整合 |
| 実行頻度 | PRごとに毎回 | 主要フローのみ厳選して実行 |

## まとめ

- コントラクトテストはConsumer/Provider間の「約束事」を個別に検証する軽量なアプローチ
- Pactは契約ファイルを介してConsumer/Provider双方のテストを疎結合に保つ
- Pact BrokerでCI連携すれば「デプロイして安全か」を自動判定できる
- E2Eテストを完全に置き換えるものではなく、役割分担して併用するのが現実的

マイクロサービスの数が増えてE2E環境の維持がつらくなってきたチームには、導入を検討する価値があるアプローチです。
