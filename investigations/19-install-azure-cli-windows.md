---
issue_number: 19
title: "Azure CLI を Windows PC にインストールする方法"
status: "resolved"
created: "2026-06-18"
resolved: "2026-06-18"
category: "azure"
related_knowledge:
  - "knowledge/azure/2026-06-18-install-azure-cli-windows.md"
---

# Azure CLI を Windows PC にインストールする方法

## 問い合わせ内容

開発用の Windows PC に Azure CLI をインストールしたい。標準的な手順を知りたい。
期待する結果は、Azure CLI がインストールでき、`az login` が使えるようになること。

## 調査プロセス

1. Issue #19 の本文を読み、カテゴリ（Azure）・対象（Azure CLI）・要望（Windows へのインストール手順）を把握。
2. `knowledge/` 配下を検索し、既存ナレッジ（App Service 503 の記事のみ）に該当がないことを確認。
3. Microsoft Learn の公式ドキュメント
   「[Install the Azure CLI on Windows](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows)」
   を取得し、WinGet / MSI / PowerShell の各インストール方法と注意点を確認。

## 調査結果

- 問い合わせは公式ドキュメントで一意に回答できる標準的な手順であり、原因の切り分けは不要（断定回答が可能）。
- Windows での標準的なインストール方法は WinGet、MSI、PowerShell の 3 通り。
  - WinGet: `winget install --exact --id Microsoft.AzureCLI`（Windows 11 / 新しい Windows 10 で既定利用可、推奨）
  - MSI: 公式ページからダウンロードして実行、または `https://aka.ms/installazurecliwindowsx64`
  - PowerShell（管理者）: `Invoke-WebRequest` で MSI を取得し `msiexec /quiet` でサイレントインストール
- 重要な注意点: インストール後はターミナルを閉じて開き直す必要がある（PATH 反映のため）。
- 検証コマンド: `az version`、サインイン: `az login`、更新: `az upgrade`。
- 上記をナレッジ `knowledge/azure/2026-06-18-install-azure-cli-windows.md` として作成。

## 回答内容

Issue #19 本体に、断定的な回答コメントを投稿した。WinGet を推奨手順として提示しつつ
MSI / PowerShell の代替手順、インストール後にターミナルを開き直す注意点、`az version` / `az login`
での確認手順、参考 Microsoft Learn URL を含めた。

## 備考

- Azure CLI のバージョンは継続的に更新されるため、本記事のコマンドは最新版を取得する形（バージョン未指定）を基本とした。
- WSL 上での利用は Linux 向けパッケージを使う旨をナレッジに補足済み。
