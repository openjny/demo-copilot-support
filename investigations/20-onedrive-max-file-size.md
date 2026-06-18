---
issue_number: 20
title: "OneDrive にアップロードできる 1 ファイルの最大サイズが知りたい"
status: "resolved"
created: "2026-06-18"
resolved: "2026-06-18"
category: "m365"
related_knowledge:
  - "knowledge/m365/2026-06-18-onedrive-max-file-size.md"
---

# OneDrive にアップロードできる 1 ファイルの最大サイズが知りたい

## 問い合わせ内容

大きな動画ファイルを OneDrive for Business にアップロードしたい。1 ファイルあたりのサイズ上限は何 GB までか、仕様を確認したい。

## 調査プロセス

1. Issue #20 本文を確認し、問い合わせカテゴリ（Microsoft 365 / OneDrive for Business）と求める情報（1 ファイルあたりの最大アップロードサイズの数値）を把握した。
2. `knowledge/` 配下を検索し、関連する既存ナレッジがないことを確認した（既存は azure カテゴリのみ）。
3. Microsoft Learn の公式ドキュメントを調査し、「SharePoint limits」サービス説明ページの「Service limits for all plans > File size and file path length」セクションで上限値を確認した。

## 調査結果

- OneDrive for Business（および SharePoint in Microsoft 365）の 1 ファイルあたりのアップロード上限は **250 GB**（確認できた事実）。
- 公式ドキュメント記載: 「**250 GB - File upload limit.** Applies to each individual file uploaded to Microsoft Teams Files tab, SharePoint document libraries, OneDrive folders, and Viva Engage conversations.」
- 出典の `ms.date` は 2026 年で、本調査時点（2026-06-18）の最新値と判断。
- ファイルサイズ上限（250 GB）とストレージ容量上限（OneDrive 標準 1 TB/ユーザー）は別の制限である点を補足した。

## 回答内容

問い合わせに対し断定的に回答した。1 ファイルあたりの最大アップロードサイズは **250 GB**。あわせて、ストレージ割り当てやパス長（400 文字）といった別制限も補足し、根拠として Microsoft Learn「SharePoint limits」の URL を提示した。

## 備考

- 制限値は変更される可能性があるため、最新値は公式ドキュメントで確認すること。
- 作成ナレッジ: `knowledge/m365/2026-06-18-onedrive-max-file-size.md`
