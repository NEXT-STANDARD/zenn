---
title: 'ClaudeやCursorにリアルタイム世論を接続する「WebMCPハブ」を個人で作った話'
emoji: '🔮'
type: 'tech'
topics:
  - 'mcp'
  - 'claude'
  - 'cursor'
  - '個人開発'
  - 'cloudflare'
published: true
---

# はじめに

こんにちは、フェニックスです。

「未来レーダー（[MiraiRadar.com](https://mirairadar.com)）」という、世界の予測市場（Polymarket）のリアルマネー確率と日本の世論をリアルタイムに比較・可視化するWebサービスを個人で作って公開しました。

そして今回、このサービスを **Cloudflare上でWebMCP（Model Context Protocol）に対応させ、Claude DesktopやCursor、ChatGPTなどの生成AIから直接呼び出せるオープンなデータハブ** として開放しました。

この記事は、
- なぜ「世界のスマートマネー × 日本の世論」というオルタナティブデータを作ったのか
- なぜそれをWebMCPで公開したのか
- Cloudflare Pages FunctionsでどうやってMCPエンドポイントを構築したのか
- ClaudeやCursorから1行で接続する仕組み

について書いた開発記録です。

---

# 1. なぜ「世界のスマートマネー × 日本の世論」なのか

2024年の米大統領選以降、Polymarketをはじめとする分散型予測市場（Prediction Market）は、世界の金融・テック界隈で「最も歪みのない先行指標」として扱われるようになりました。

世論調査（アンケート）は「本音を言わない」「サンプルが偏る」というバイアスを抱えがちですが、予測市場では **世界中のスマートマネーやクォンツが身銭を切って（リアルマネーで）オッズを形成する** ため、圧倒的な情報の純度とスピードを持ちます。

しかし、日本で生活していると、ひとつ強烈な違和感を感じることがありました。

「**世界の大口マネーが冷徹に弾き出した確率**」と、「**日本のお茶の間やファンの直感**」の間には、必ず **巨大な見解のギャップ（世論スプレッド）** が存在するということです。

たとえば、
- **大谷翔平の60本塁打**: 海外のクォンツは敬遠リスクや投手復帰を警戒して「50%」と慎重に見積もる一方、日本のファンは「88%」が確信している。
- **日銀の追加利上げ**: 海外ヘッジファンドは円安インフレ圧力から「65%」を織り込む一方、日本国内の生活者は「35%」と慎重。

| 観測テーマ | 世界スマートマネー（Polymarket） | 日本の生活者世論 | 世論ギャップ（乖離） |
|---|:---:|:---:|:---:|
| ⚾ 大谷翔平 60本塁打達成 | 50% | 88% | **38%** |
| 📊 日銀の年内追加利上げ | 62% | 35% | **27%** |
| 🎮 Nintendo Switch 2 発売 | 85% | 92% | **7%** |

この「38%のギャップ」こそが、最も面白いオルタナティブデータです。

### 理念：世論を誘導するのではなく、客観的ギャップを提示する

大事にしているのは、「**世論を特定の方向に誘導する意図は1ミリもない**」ということです。
「世界が正しい」とも「日本のファンが正しい」とも主張しません。「世界とお茶の間の間に、今これだけの見解の差があるかもしれない」という客観的事実を提示する。

これが未来レーダーの核となる設計思想です。

---

# 2. なぜ WebMCP（Model Context Protocol）で公開したのか

ChatGPTやClaudeなどのLLM（生成AI）を使っていて、誰もが一度は感じるもどかしさがあります。

> 「**AIは過去の知識は完璧だけど、今この瞬間の世界のコンセンサスや世論を知らない**」

検索プラグインを使えばWeb記事は読めますが、記事に書かれているのは「記者の個人的な見解」であって、「市場参加者の確率」や「大衆のリアルタイムな比率」ではありません。

そこで、**生成AIが外部ツールとして未来レーダーの世論スプレッドを直接叩けるプロトコル（MCP）** を実装しました。

```
┌──────────────────┐               ┌────────────────────────┐               ┌──────────────────┐
│  ユーザーの指示  │  ─────────▶   │ Claude / Cursor / AI   │  ─────────▶   │   未来レーダー   │
│「日銀利上げについて│               │ (Model Context         │ (WebMCP 経由) │ (MiraiRadar.com) │
│ 世界と日本の差は？」│  ◀─────────   │  Protocol コネクタ)    │  ◀─────────   │ リアルタイムオッズ│
└──────────────────┘               └────────────────────────┘               └──────────────────┘
```

Claude DesktopやCursorに未来レーダーを接続すると、AIは自律的にツールを呼び出し、以下のような高解像度な回答を瞬時に生成できるようになります。

```text
ユーザー: 「大谷翔平の60本塁打について、海外オッズと日本の世論を比較分析して」

Claude:
（未来レーダーMCPの get_market_detail を自動実行）
「現在、世界の大口予測市場（Polymarket）では達成確率が【50%】と慎重な見方が優勢です。
一方、日本の世論調査では【88%】が達成を支持しており、【38%の巨大なスプレッド】が発生しています。

世界のスマートマネーは後半戦の敬遠四球の急増を警戒しているのに対し、日本のファンは
打球初速と直近の量産ペースを高く評価している点が最大の分岐点となっています。」
```

---

# 3. 技術アーキテクチャ ＆ WebMCPの実装

未来レーダーの全体アーキテクチャは以下の通りです。

```
[クライアント (ブラウザ)] ──┐
                          ├──▶ [Cloudflare Edge (Pages Functions)]
[Claude / Cursor (MCP)] ──┘         │
                                     ├──▶ [GET/POST /api/mcp] (JSON-RPC 2.0)
                                     ├──▶ [GET /market/:slug] (動的OGPエッジルーター)
                                     │
                                    ▼
                          [Polymarket CLOB API] ＋ [Supabase Database]
```

### Cloudflare Pages Functions による MCP エンドポイント

MCPはAnthropicが策定したJSON-RPC 2.0ベースのオープンプロトコルです。
Cloudflare Pages Functionsの `functions/api/mcp.ts` に、MCP規格に完全準拠したエンドポイントを配備しました。

```typescript:functions/api/mcp.ts
// 提供するMCPツール一覧の定義
const MCP_TOOLS = [
  {
    name: 'get_top_spread_discrepancies',
    description: '世界オッズと日本世論の間で、見解が最も乖離している注目銘柄TOPランキングを取得します。',
    inputSchema: {
      type: 'object',
      properties: {
        limit: { type: 'number', description: '取得件数 (最大10)' },
      },
    },
  },
  {
    name: 'get_market_detail',
    description: '特定銘柄のリアルタイム世界オッズ、日本世論支持率、AIカタリスト、YES/NO論拠を取得します。',
    inputSchema: {
      type: 'object',
      properties: {
        slug: { type: 'string', description: '銘柄スラッグ (例: ohtani-60-home-runs)' },
      },
      required: ['slug'],
    },
  },
  {
    name: 'search_radar_topics',
    description: 'キーワード（大谷翔平、日銀、AIなど）で未来レーダーの観測銘柄を検索します。',
    inputSchema: {
      type: 'object',
      properties: {
        query: { type: 'string', description: '検索キーワード' },
      },
      required: ['query'],
    },
  },
];

export const onRequest = async ({ request }: { request: Request }) => {
  const corsHeaders = {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    'Content-Type': 'application/json; charset=utf-8',
  };

  if (request.method === 'OPTIONS') {
    return new Response(null, { headers: corsHeaders });
  }

  // GETリクエスト: サーバー情報 ＆ ツール一覧
  if (request.method === 'GET') {
    return new Response(JSON.stringify({
      name: 'mirairadar-webmcp',
      version: '1.0.0',
      description: '未来レーダー WebMCP Server - 世界のスマートマネー × 日本の世論',
      tools: MCP_TOOLS,
    }, null, 2), { headers: corsHeaders });
  }

  // POSTリクエスト: MCP JSON-RPC 2.0 ツール実行
  const body = await request.json() as any;
  const { method, params, id } = body;

  if (method === 'tools/list') {
    return new Response(JSON.stringify({
      jsonrpc: '2.0',
      id: id ?? 1,
      result: { tools: MCP_TOOLS },
    }), { headers: corsHeaders });
  }

  if (method === 'tools/call') {
    const toolName = params?.name;
    const toolArgs = params?.arguments || {};
    
    // ...各ツールのデータ返却ロジック...
  }
};
```

### レートリミットとエッジキャッシュ（300秒）

AIエージェントからの大量アクセスによるオリジンサーバー（DBやPolymarket API）の枯渇を防ぐため、Cloudflare CDNエッジで **300秒（5分）のHTTPキャッシュ** を効かせています。
世界中どこから叩かれても、**50ms未満の超低遅延** で即座にJSONが返ります。

---

# 4. Claude Desktop / Cursor からの接続方法（1秒ノーコード）

ローカルのCLIコマンドを打つ必要はありません。アプリの設定画面にURLを貼るだけで完了します。

### 方法 1: 設定画面でURLを入れるだけ（推奨）

Claude DesktopやCursorの **「Settings ➔ MCP (Model Context Protocol)」** を開き、以下を登録します。

| 項目 | 入力値 |
|---|---|
| **Server Name** | `mirairadar` |
| **URL (Remote MCP)** | `https://mirairadar.com/api/mcp` |

これだけで、ClaudeやCursorが未来レーダーを認識します。

### 方法 2: `claude_desktop_config.json` に書く場合

```json
{
  "mcpServers": {
    "mirairadar": {
      "command": "npx",
      "args": ["-y", "@mirairadar/mcp-server"],
      "env": {
        "MIRAIRADAR_API_URL": "https://mirairadar.com/api/mcp"
      }
    }
  }
}
```

---

# 5. 法務・コンプライアンスの設計（Regulatory Moat）

予測市場や世論調査を扱うWebサービスにおいて、法務防御は最も重要なインフラです。

1. **刑法185条・186条（賭博罪）の完全排除**:
   - 金銭・暗号資産・ポイントの授受を100%排除。完全無料・登録不要のオルタナティブデータメディアに徹する。
2. **公職選挙法第138条の3（人気投票の公表禁止）自動ロック**:
   - 衆院選や参院選などの選挙期間中は、該当銘柄の投票受付を全画面で自動停止（`isElectionBlackout`）。Polymarketの閲覧専用モードへと安全に切り替える。
3. **管理画面のローカル完全隔離**:
   - 管理画面（`/admin`）は本番環境では一切露出せず、`localhost` 環境でのみ動作するようコードレベルで隔離。

---

# おわりに

AIエージェントが普及した世界では、Webサイトのユーザーは「人間」だけではなくなります。
「**人間が画面を見て楽しむUI**」と「**AIエージェントがプロトコル経由で知性を取得するAPI（WebMCP）**」の両方を備えていることが、これからのWebサービスの標準になっていくはずです。

もしよろしければ、ClaudeやCursorに接続して遊んでみてください。

- サービスURL: [https://mirairadar.com](https://mirairadar.com)
- WebMCP設定ガイド: [https://mirairadar.com/ai-connector](https://mirairadar.com/ai-connector)
- MCP エンドポイント: `https://mirairadar.com/api/mcp`
