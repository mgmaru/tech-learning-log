# MCPサーバの仕組みと配置パターンまとめ

## 1. このドキュメントの目的

このドキュメントは、MCPサーバの基本的な仕組みと、MCPサーバをどこに配置すべきかを整理した学習用メモです。

特に、以下の疑問を解消することを目的にしています。

- MCPサーバとは何か
- MCPクライアントとの関係は何か
- MCPサーバを自作すると何が理解できるのか
- 個人利用とチーム利用で、MCPサーバの置き場所はどう変わるのか
- プロジェクト内にMCPサーバを置くべきか
- 共有サーバとして運用する場合はどう考えるべきか

---

## 2. MCPとは何か

MCPは **Model Context Protocol** の略です。

ざっくり言うと、MCPは **AIアプリに外部ツールや外部データを接続するための共通規格** です。

通常、AIは会話上で与えられた情報しか扱えません。しかしMCPを使うと、AIクライアントは外部の機能にアクセスできるようになります。

例えば、以下のようなことが可能になります。

- ローカルファイルを読む
- プロジェクトのMarkdownドキュメントを検索する
- GitHub Issueを取得する
- DBを検索する
- NotionやSlackなどの外部サービスと連携する
- 独自の開発補助ツールをAIから呼び出す

---

## 3. MCPの登場人物

MCPの基本構造は以下のようになります。

```text
ユーザー
  ↓
MCP対応クライアント
例: Claude Desktop / Claude Code / Cursor / VS Code など
  ↓
MCPサーバ
例: GitHub MCP / Filesystem MCP / 自作MCP
  ↓
外部リソース
例: ファイル、DB、API、GitHub、Notion、Slack など
```

重要なのは、**MCPサーバ自体がAIモデルではない** という点です。

MCPサーバは、AIに対して、

```text
私はこういう機能を提供できます
```

と伝える外部ツール提供サーバです。

---

## 4. MCPサーバが提供するもの

MCPサーバは、主に以下のような機能を提供します。

| 種類 | 役割 | 例 |
|---|---|---|
| Tools | AIから呼び出せる関数 | `add`, `search_docs`, `create_issue` |
| Resources | AIが参照できるデータ | ファイル内容、DBの情報、APIレスポンス |
| Prompts | 定型プロンプト | レビュー用プロンプト、設計相談プロンプト |

最初に学ぶなら、まずは **Tools** だけで十分です。

例えば、以下のようなToolを作るとMCPの仕組みを理解しやすくなります。

| Tool名 | 内容 |
|---|---|
| `hello` | 名前を受け取って挨拶文を返す |
| `add` | 2つの数値を足す |
| `read_note` | 指定したメモファイルを読む |
| `search_notes` | notesフォルダ内のMarkdownを検索する |

---

## 5. MCPサーバを自作する意味

MCPサーバの仕組みを理解するには、**自作してみるのがかなり有効**です。

理由は、以下の関係を体感できるからです。

```text
MCPクライアント
  ↓ Toolを呼び出す
MCPサーバ
  ↓ 実際の処理をする
外部リソース
```

自作すると、以下のことが理解しやすくなります。

| 理解できること | 内容 |
|---|---|
| MCPクライアントとは何か | Claude CodeやClaude Desktopなど、MCPサーバを呼び出す側 |
| MCPサーバとは何か | 外部機能をAIに公開する側 |
| Toolとは何か | AIから呼び出せる関数 |
| Resourceとは何か | AIが読み取れる情報 |
| なぜ権限管理が大事か | ファイル、DB、APIキーにアクセスできてしまうため |
| なぜMCPが便利か | 外部ツール連携を共通規格で扱えるため |

---

## 6. 最小のMCPサーバ実装例

以下は、TypeScriptで作る最小のMCPサーバの例です。

この例では、`hello` と `add` という2つのToolを提供します。

### ディレクトリ構成

```text
my-first-mcp-server/
├─ package.json
├─ tsconfig.json
└─ src/
   └─ index.ts
```

### package.json

```json
{
  "name": "my-first-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "bin": {
    "my-first-mcp-server": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsx src/index.ts"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "latest",
    "zod": "latest"
  },
  "devDependencies": {
    "tsx": "latest",
    "typescript": "latest"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*.ts"]
}
```

### src/index.ts

```ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// MCPサーバ本体を作成
const server = new McpServer({
  name: "my-first-mcp-server",
  version: "1.0.0",
});

// Tool 1: hello
server.tool(
  "hello",
  "名前を受け取って、挨拶文を返します。",
  {
    name: z.string().describe("挨拶したい相手の名前"),
  },
  async ({ name }) => {
    return {
      content: [
        {
          type: "text",
          text: `こんにちは、${name}さん！`,
        },
      ],
    };
  }
);

// Tool 2: add
server.tool(
  "add",
  "2つの数値を足し算します。",
  {
    a: z.number().describe("1つ目の数値"),
    b: z.number().describe("2つ目の数値"),
  },
  async ({ a, b }) => {
    const result = a + b;

    return {
      content: [
        {
          type: "text",
          text: `${a} + ${b} = ${result}`,
        },
      ],
    };
  }
);

// stdio経由でMCPクライアントと接続
const transport = new StdioServerTransport();
await server.connect(transport);
```

このコードで重要なのは、以下の部分です。

```ts
server.tool(...)
```

これは、AIクライアントに対して、

```text
このMCPサーバには、この名前・説明・入力形式のToolがあります
```

と公開している部分です。

---

## 7. MCPサーバの通信方式

MCPサーバの接続方式には、主に以下の2パターンがあります。

| 方式 | 概要 | 向いている用途 |
|---|---|---|
| stdio | クライアントがMCPサーバをローカルプロセスとして起動し、標準入出力で通信する | 個人利用、ローカル開発、プロジェクト同梱 |
| HTTP | MCPサーバをWebサーバのように起動し、URL経由で通信する | チーム共有、社内ツール、クラウド運用 |

最初に学ぶなら、**stdio方式** が分かりやすいです。

---

## 8. MCPサーバを置く場所の考え方

MCPサーバは、利用目的によって置き場所が変わります。

主な配置パターンは以下の3つです。

```text
1. 自分のローカルPCに置く
2. プロジェクト内に同梱して、各メンバーがローカルで起動する
3. HTTP MCPサーバとして社内・クラウド上に配置する
```

---

## 9. パターン1：自分のローカルPCに置く

学習用や個人利用なら、自分のPCにMCPサーバを置けば十分です。

```text
自分のPC
├─ Claude Code
└─ MCPサーバ
```

Claude CodeやClaude Desktopが、ローカルのMCPサーバを起動して利用します。

### 向いている用途

- MCPの学習
- 個人用ツール
- 自分のローカルファイルを読むMCP
- 自分だけが使う開発補助

### メリット

- 最も簡単に始められる
- 外部公開しなくてよい
- セキュリティリスクを小さくできる
- 手元のファイルを扱いやすい

### デメリット

- 他の人はそのまま使えない
- チーム共有には向かない
- 各PCごとに設定が必要

---

## 10. パターン2：プロジェクト内にMCPサーバを同梱する

チーム開発で最初におすすめなのは、この方式です。

```text
project-root/
├─ app/
├─ docs/
├─ tools/
│  └─ mcp-server/
│     ├─ package.json
│     ├─ tsconfig.json
│     └─ src/
├─ .mcp.json
└─ README.md
```

この方式では、MCPサーバのコードをGitHubリポジトリに含めます。

各メンバーはリポジトリをcloneし、自分のPC上でMCPサーバを起動します。

```text
GitHubリポジトリ
  ↓ clone
AさんのPC: Claude Code + MCPサーバ
BさんのPC: Claude Code + MCPサーバ
CさんのPC: Claude Code + MCPサーバ
```

つまり、MCPサーバは各自のPCで動きますが、**実装と設定をチームで共有できる** という形です。

### 向いている用途

- プロジェクト専用のMCPサーバ
- `docs/` 配下の仕様書検索
- `architecture.md` の参照
- コーディング規約の参照
- Issueテンプレート生成
- Claude Codeにプロジェクト情報を読ませたい場合

### メリット

- チームで同じMCPサーバ実装を使える
- Git管理できる
- プロジェクト固有の開発支援に向いている
- 外部公開しなくてよい
- ローカルファイルへのアクセスが各自の環境内で完結する

### デメリット

- 各メンバーのPCでセットアップが必要
- Node.jsなどの実行環境を揃える必要がある
- APIキーや環境変数は各自で設定する必要がある

---

## 11. `.mcp.json` によるプロジェクト共有

Claude Codeでは、プロジェクトスコープのMCP設定を使うことで、MCPサーバ設定をプロジェクト内で共有できます。

イメージとしては、プロジェクトルートに `.mcp.json` を置きます。

```json
{
  "mcpServers": {
    "project-docs": {
      "type": "stdio",
      "command": "node",
      "args": ["tools/mcp-server/dist/index.js"]
    }
  }
}
```

この設定をGit管理すれば、チームメンバーが同じMCPサーバ構成を使いやすくなります。

ただし、以下のような情報はGitに含めないように注意します。

- APIキー
- アクセストークン
- パスワード
- 個人の絶対パス
- 本番DB接続情報

必要な値は `.env` や環境変数で管理するのが基本です。

---

## 12. パターン3：HTTP MCPサーバとして共有する

複数人で本当に「同じMCPサーバ」を使いたい場合は、HTTP MCPサーバとしてデプロイします。

```text
AさんのClaude Code ─┐
BさんのClaude Code ─┼→ https://mcp.example.com/mcp
CさんのClaude Code ─┘
                         ↓
                    MCPサーバ
                         ↓
                  DB / GitHub / 社内API
```

この場合、MCPサーバはローカルPCではなく、以下のような場所に置きます。

- 社内サーバ
- VPS
- Dockerコンテナ環境
- AWS / GCP / Azure
- Render / Railway / Fly.io
- Cloudflare Workers など

### 向いている用途

- チーム全員で同じDB/APIを参照したい
- 社内ツールと連携したい
- GitHub、Jira、Notion、Slackなどと共通連携したい
- 会社全体で使うMCPサーバを作りたい

### メリット

- 全員が同じMCPサーバを利用できる
- サーバ側を更新すれば全員に反映できる
- DB/API/社内ツール連携に向いている
- ログ管理や権限管理を中央集約しやすい

### デメリット

- 認証が必要
- HTTPS化が必要
- デプロイ、監視、ログ管理が必要
- セキュリティ設計が重要になる

---

## 13. 配置パターンの比較

| やりたいこと | おすすめの置き場所 |
|---|---|
| 自分だけ学習したい | 自分のPC |
| 自分だけ便利ツールを使いたい | 自分のPC |
| プロジェクトのdocsをClaudeに読ませたい | プロジェクト内 `tools/mcp-server/` |
| チーム全員で同じプロジェクト用MCPを使いたい | プロジェクト内 + `.mcp.json` |
| チーム全員で同じDB/APIを見たい | HTTP MCPサーバ |
| 会社全体で使いたい | 独立リポジトリ + クラウド/社内サーバ |
| 秘密情報や本番DBに接続する | HTTP MCP + 認証 + 読み取り専用 + ログ |

---

## 14. おすすめの進め方

MCPを学びながら実用化するなら、以下の順番がおすすめです。

```text
Step 1: 個人ローカルで最小MCPを作る
  ↓
Step 2: プロジェクト内にMCPサーバを置く
  ↓
Step 3: .mcp.jsonでチーム共有する
  ↓
Step 4: 必要になったらHTTP MCP化する
```

### Step 1：個人ローカルで最小MCPを作る

最初は以下のようなToolだけで十分です。

- `hello`
- `add`

目的は、MCPクライアントとMCPサーバの関係を理解することです。

### Step 2：プロジェクト内にMCPサーバを置く

次に、プロジェクト内に以下のような構成を作ります。

```text
project-root/
├─ docs/
│  ├─ requirements.md
│  └─ architecture.md
├─ tools/
│  └─ mcp-server/
│     ├─ src/
│     ├─ package.json
│     └─ tsconfig.json
├─ .mcp.json
└─ README.md
```

このMCPサーバには、以下のようなToolを用意すると便利です。

| Tool名 | 内容 |
|---|---|
| `list_project_docs` | プロジェクト内のドキュメント一覧を返す |
| `read_project_doc` | 指定したMarkdownファイルを読む |
| `search_project_docs` | `docs/` 配下のMarkdownを検索する |

### Step 3：`.mcp.json` でチーム共有する

MCP設定をプロジェクトスコープにして、チームで共有します。

この段階では、MCPサーバ自体は各自のPCで動きます。

### Step 4：必要になったらHTTP MCP化する

以下のような要件が出てきたら、共有HTTPサーバ化を検討します。

- 同じDBを全員で見たい
- 社内APIと連携したい
- GitHubやJiraなどの外部サービスを一元管理したい
- 権限やログを中央で管理したい

---

## 15. セキュリティ上の注意点

MCPサーバは、AIから外部ツールを呼び出せるようにする仕組みです。

そのため、以下のような操作を許可する場合は注意が必要です。

- ファイルの読み取り
- ファイルの書き込み
- ファイルの削除
- DBの参照
- DBの更新
- GitHub Issueの作成・更新
- SlackやNotionへの投稿
- APIキーを使った外部サービス連携

特に最初は、以下の方針が安全です。

```text
- 読み取り専用から始める
- 本番DBには接続しない
- 削除系Toolは作らない
- APIキーはGitに含めない
- 必要最小限の権限だけ渡す
- Toolの説明を明確に書く
- 実行ログを残す
```

---

## 16. プロジェクト内MCPサーバの実用例

例えば、プロジェクトに以下のようなドキュメントがあるとします。

```text
docs/
├─ requirements.md
├─ architecture.md
├─ screen_specifications.md
└─ coding_rules.md
```

MCPサーバに以下のToolを用意します。

```text
list_project_docs
read_project_doc
search_project_docs
```

すると、Claude Codeに対して以下のような依頼がしやすくなります。

```text
requirements.mdを読んで、この実装方針が仕様に合っているか確認してください。
```

```text
screen_specifications.mdを参照して、画面仕様に沿ったReactコンポーネントを作ってください。
```

```text
coding_rules.mdに従って、このコードをレビューしてください。
```

つまり、MCPサーバは **プロジェクト固有の知識をClaude Codeに渡すための橋渡し** になります。

---

## 17. MCPサーバの置き場所に関する結論

MCPサーバの置き場所は、利用範囲によって変えるのがよいです。

```text
個人利用:
  自分のローカルPCでOK

プロジェクト利用:
  プロジェクト内 tools/mcp-server/ に置く

チームで同じ実装を共有:
  Git管理 + .mcp.json で共有

チームで同じDB/APIを共有:
  HTTP MCPサーバとしてデプロイ

会社全体で利用:
  独立リポジトリ + 認証付き共有サーバ
```

今回の議論で特に重要な結論は以下です。

> 個人利用ならローカルPCで十分。  
> チーム開発でまず試すなら、MCPサーバのコードをプロジェクト内に置き、`.mcp.json` で共有するのが現実的。  
> 全員が同じDB/API/社内ツールを使いたい段階になったら、HTTP MCPサーバとしてデプロイする。

---

## 18. 参考リンク

- Model Context Protocol 公式ドキュメント  
  https://modelcontextprotocol.io/

- MCP TypeScript SDK  
  https://github.com/modelcontextprotocol/typescript-sdk

- Claude Code MCP ドキュメント  
  https://code.claude.com/docs/ja/mcp

- MCP Inspector  
  https://modelcontextprotocol.io/docs/tools/inspector
