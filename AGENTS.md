# AGENTS.md

このリポジトリは、GitHub Copilot Coding Agent を使った IT サポート問い合わせの
自動調査・ナレッジ蓄積ワークフローのデモ環境です。

## エージェント構成

```
Issue（問い合わせ） → auto-triage ラベル付与 → Support Agent → ナレッジ作成/更新（PR） → Issue 回答 → PR マージで Issue 自動クローズ
```

| エージェント | 定義ファイル | 役割 |
|---|---|---|
| Support Agent | `.github/agents/support.agent.md` | Issue の問い合わせを調査し、ナレッジを作成して回答する |

## ファイルレイアウト

- `.github/copilot-instructions.md` — Coding Agent のグローバル指示（ワークフロー定義、命名規則、フォーマット規定）
- `.github/agents/support.agent.md` — サポートエージェント定義
- `.github/ISSUE_TEMPLATE/inquiry.yml` — 問い合わせ用 Issue テンプレート
- `.github/workflows/auto-triage.yml` — `auto-triage` ラベル付与で Copilot cloud agent を Issue に自動アサイン（要 `COPILOT_ASSIGN_TOKEN` Secret）
- `.github/workflows/auto-merge-knowledge.yml` — `approved` ラベル付与または 7 日放置で PR を自動マージ（Issue を自動クローズ）
- `.vscode/mcp.json` — Microsoft Learn MCP 設定（ローカル VS Code 用）
- `knowledge/` — 再利用可能なナレッジベース（カテゴリ別）
- `knowledge/_template.md` — ナレッジ記事テンプレート
- `investigations/` — Issue 固有の調査記録
- `investigations/_template.md` — 調査記録テンプレート
- `docs/DESIGN.md` — 設計思想・存在理由

## ファイル命名規則

### ナレッジ (`knowledge/`)

```
knowledge/{category}/{YYYY-MM-DD}-{slug}.md
```

- `category`: `azure` | `m365` | `security`
- `slug`: 内容を表す kebab-case の短い識別子
- 例: `knowledge/azure/2026-06-18-app-service-503-error.md`

### 調査記録 (`investigations/`)

```
investigations/{issue-number}-{slug}.md
```

- `issue-number`: 対応する GitHub Issue 番号（ゼロ埋めなし）
- `slug`: 問い合わせ内容を表す kebab-case の短い識別子
- 例: `investigations/1-app-service-503-error.md`

## 不変条件

- `knowledge/` の記事は Issue 横断で再利用される。特定 Issue に依存する情報は
  `investigations/` に書く。
- すべてのナレッジ・調査記録は YAML frontmatter を持つ（テンプレート参照）。
- `knowledge/` のファイルは作成後も更新される（新しい情報の追記、関連 Issue の追加）。
- `investigations/` のファイルは原則として対応完了後は更新しない（履歴として保持）。
- 成果物は Issue ごとに 1 つの PR として提出し、PR 本文に `Closes #<issue>` を記載する。
- PR は `approved` ラベル付与または 7 日放置で自動マージされ、Agent 自身はマージしない。
- 日付が必要な場合は `date -I` で取得する。
