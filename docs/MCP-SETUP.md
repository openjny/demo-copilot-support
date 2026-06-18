# MCP サーバー設定ガイド

このリポジトリで使用する MCP (Model Context Protocol) サーバーの設定方法をまとめる。

## 概要

Microsoft Learn MCP サーバーを利用し、GitHub Copilot が公式ドキュメントを参照できるようにする。

| 環境 | 設定場所 | トップレベルキー |
|------|---------|--------------|
| VS Code | `.vscode/mcp.json` | `"servers"` |
| Copilot CLI | `.github/mcp.json` または `~/.copilot/mcp-config.json` | `"mcpServers"` |
| Cloud Agent (Coding Agent) | GitHub.com リポジトリ設定 | `"mcpServers"` |

## VS Code 向け設定

ファイル: `.vscode/mcp.json`

```json
{
  "servers": {
    "microsoft-learn": {
      "type": "http",
      "url": "https://learn.microsoft.com/api/mcp"
    }
  }
}
```

## GitHub Copilot CLI 向け設定

ファイル: `.github/mcp.json`（プロジェクトレベル）

```json
{
  "mcpServers": {
    "microsoft-learn": {
      "type": "http",
      "url": "https://learn.microsoft.com/api/mcp"
    }
  }
}
```

ユーザーレベルで設定する場合は `~/.copilot/mcp-config.json` に同じ内容を記述する。

### 確認方法

Copilot CLI セッション内で以下を実行:

```
/mcp show
```

## GitHub Copilot Cloud Agent (Coding Agent) 向け設定

Cloud Agent の MCP 設定はリポジトリファイルではなく、**GitHub.com のリポジトリ設定画面**で行う。

### 設定手順

1. GitHub.com でリポジトリのメインページに移動
2. **Settings** タブをクリック
3. サイドバーの「Code & automation」セクションから **Copilot** → **MCP servers** を選択
4. 「MCP configuration」セクションに以下の JSON を入力:

```json
{
  "mcpServers": {
    "microsoft-learn": {
      "type": "http",
      "url": "https://learn.microsoft.com/api/mcp",
      "tools": [""]
    }
  }
}
```

5. **Save MCP configuration** をクリック

### 注意事項

- Cloud Agent の MCP 設定では `tools` プロパティ（文字列配列）が**必須**。`[""]` で全ツールを許可
- この設定は Copilot Cloud Agent と Copilot Code Review の両方で共有される
- Code Review で MCP を無効にしたい場合は、Settings > Copilot > Code review から「Allow Copilot to use MCP tools when reviewing pull requests」をオフにする
- GitHub MCP Server と Playwright MCP Server はデフォルトで有効

### デフォルトで利用可能なサーバー

Cloud Agent は以下の MCP サーバーが事前設定済み:

- **GitHub MCP Server** — リポジトリ内の Issue、PR、コード検索
- **Playwright MCP Server** — ブラウザ自動化

## 設定ファイルの違い

| 項目 | VS Code | Copilot CLI | Cloud Agent |
|------|---------|-------------|-------------|
| キー名 | `servers` | `mcpServers` | `mcpServers` |
| 設定場所 | リポジトリ内ファイル | リポジトリ内ファイル or ユーザーホーム | GitHub.com UI |
| 認証 | CodeLens で OAuth | `/mcp add` で対話設定 | Agents secrets |
| `type` の値 | `http` | `http` | `http`, `sse`, `local` |

## 参考リンク

- [Configure MCP servers for your repository](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/configure-mcp-servers)
- [Adding MCP servers for GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/use-copilot-cli/overview)
- [Extending GitHub Copilot Chat with MCP](https://docs.github.com/en/copilot/customizing-copilot/using-model-context-protocol/extending-copilot-chat-with-mcp)
