---
title: 【Claude Code】MCPサーバー構築実践入門 - 独自ツールをAIに接続する
tags:
  - MCP
  - ClaudeCode
  - Claude
  - AI
  - TypeScript
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

MCP（Model Context Protocol）は、AIアシスタントと外部ツール・データソースを接続するためのオープンプロトコルです。Claude Codeは標準でMCPクライアントとして動作し、社内API・DB・独自ツールをMCPサーバーとして公開すれば、そのままAIから呼び出せるようになります。この記事では、最小構成のMCPサーバーを作りながら仕組みを理解します。

## MCPの全体像

MCPは大きく3つの役割で成り立っています。

- **Host**: Claude Codeなど、AIとMCPサーバーを仲介するアプリケーション
- **Client**: Hostの内部でMCPサーバーと通信するコンポーネント
- **Server**: 実際にツールやリソースを提供するプロセス

サーバーは「Tools」（AIが呼び出せる関数）、「Resources」（AIが読めるデータ）、「Prompts」（定型プロンプトのテンプレート）の3種類を公開できます。

## 最小構成のサーバーを書く

TypeScript SDKを使うと、数十行でツールを1つ持つMCPサーバーが書けます。

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "weather-internal", version: "1.0.0" });

server.tool(
  "get_incident_status",
  "社内インシデント管理システムから現在のステータスを取得する",
  { incidentId: z.string() },
  async ({ incidentId }) => {
    const res = await fetch(`https://internal-api/incidents/${incidentId}`);
    const data = await res.json();
    return {
      content: [{ type: "text", text: JSON.stringify(data) }],
    };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

標準入出力（stdio）でHostと通信するのが最もシンプルな構成です。ツールの説明文とZodスキーマがそのままAIへのインターフェース定義になります。

## Claude Codeへの登録

作成したサーバーを `.mcp.json` に登録すると、Claude Codeが起動時に自動接続します。

```json
{
  "mcpServers": {
    "internal-tools": {
      "command": "node",
      "args": ["./mcp-servers/internal-tools/dist/index.js"]
    }
  }
}
```

`.mcp.json` はプロジェクト直下に置けばチーム全体で共有でき、`enableAllProjectMcpServers` や `enabledMcpjsonServers` で自動承認するかどうかを制御できます。

## Resourcesでコンテキストを渡す

Toolsが「AIが実行できる操作」なのに対し、Resourcesは「AIが読み込めるデータ」です。社内ドキュメントやログをResourceとして公開すれば、AIが必要なときに参照できます。

```typescript
server.resource(
  "runbook",
  "internal://runbooks/{service}",
  async (uri, { service }) => ({
    contents: [
      {
        uri: uri.href,
        text: await fetchRunbook(service),
      },
    ],
  })
);
```

## 認証情報の扱い

社内APIのトークンをMCPサーバーに持たせる場合、環境変数経由で渡すのが基本です。

```json
{
  "mcpServers": {
    "internal-tools": {
      "command": "node",
      "args": ["./mcp-servers/internal-tools/dist/index.js"],
      "env": { "INTERNAL_API_TOKEN": "${INTERNAL_API_TOKEN}" }
    }
  }
}
```

トークンをリポジトリに直書きせず、実行環境の環境変数から展開する構成にしておくことで、`.mcp.json` 自体は安全にコミットできます。

## まとめ

- MCPはTools/Resources/Promptsの3種類でAIに機能を公開する
- TypeScript SDKなら数十行で最小構成のサーバーが書ける
- `.mcp.json` に登録すればチーム全体でClaude Codeから利用可能になる
- 認証情報は環境変数展開で扱い、設定ファイル自体はコミット可能な状態に保つ

社内システムをMCPサーバー化しておくと、Claude Codeが「聞けば答えてくれる・動かせる」対象がどんどん広がっていきます。
