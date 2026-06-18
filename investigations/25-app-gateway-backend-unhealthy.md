---
issue_number: 25
title: "Application Gateway (Standard_v2) のバックエンドが Unhealthy になる"
status: "open"
created: "2026-06-18"
resolved: ""
category: "azure"
related_knowledge:
  - "knowledge/azure/2026-06-18-app-gateway-backend-unhealthy.md"
---

# Application Gateway (Standard_v2) のバックエンドが Unhealthy になる

## 問い合わせ内容

- 対象サービス: Azure Application Gateway（SKU: Standard_v2、リージョン: Japan East）
- 発生契機: 構成変更後
- 事象: バックエンドプールに登録した VM が「バックエンド正常性」で Unhealthy と表示され、
  サイトにアクセスできない。ヘルスプローブ設定は確認したが、どこを見ればよいか分からない。
- 期待結果: バックエンドが Healthy になり、サイトにアクセスできること。

## 調査プロセス

1. Issue #25 の本文を読み、事象・SKU・発生契機（構成変更後）を整理した。
2. `knowledge/` を検索したが、Application Gateway のバックエンド正常性に関する既存ナレッジは
   なかった（App Service 503 のナレッジのみ）。
3. Microsoft Learn の公式トラブルシューティングドキュメントを参照し、Unhealthy の
   メッセージ別の原因・対処、既定プローブの挙動、確認ツール（portal / PowerShell）を確認した。
   - Troubleshoot backend health issues in Azure Application Gateway
   - Application Gateway health probes overview / Create a custom probe
4. 調査結果をナレッジ `knowledge/azure/2026-06-18-app-gateway-backend-unhealthy.md` に整理した。

## 調査結果

- **確認できた事実（公式ドキュメント根拠）**
  - 全バックエンドが Unhealthy になると Application Gateway は 502 Bad Gateway を返す。
    「サイトにアクセスできない」という症状と整合する。
  - Unhealthy の原因は単一ではなく、ネットワーク到達性（TCP connect error: NSG/UDR/FW）、
    DNS 解決、アプリの応答コード（status code mismatch）、応答本文、TLS/証明書（CN 不一致・
    期限切れ・チェーン不備・信頼されない CA）など複数層に分かれる。
  - 既定プローブは `<protocol>://127.0.0.1:<port>/` 形式で、プロトコル/ポートは HTTP 設定を
    継承し、200〜399 のみを Healthy とみなす。別のホスト名/パス/コードを許容するには
    カスタムプローブが必要。
  - 公式の定石は、**バックエンド正常性の Details 列のメッセージを起点に切り分ける**こと。

- **未確認（情報不足のため断定不可）**
  - Details 列に実際に表示されているメッセージ。
  - 「構成変更後」に何を変更したか（NSG/UDR、プローブ、HTTP 設定の HTTPS 化、証明書、
    バックエンドターゲットの種別など）。
  - バックエンドプールの種別（IP / FQDN / App Service）とプローブのプロトコル（HTTP/HTTPS）。

  → 以上が不明なため原因を一意に断定できない。回答では原因候補と切り分け手順を提示する方針とした。

## 回答内容

Issue #25 に対し、原因を断定せず以下を回答した（断定回避・切り分け方針）。

- まず「バックエンド正常性」の **Details メッセージ**を確認するよう案内（最優先・原因特定の起点）。
- 想定原因を可能性順に提示し、各候補に根拠の有無を明記:
  TCP connect error（NSG/UDR/FW）、status code mismatch（401/403/404 等）、
  DNS resolution error、timeout、TLS/証明書（CN 不一致・期限切れ・チェーン）。
- メッセージ別の切り分け手順（確認項目と期待結果の対応づけ）と、PowerShell による
  有効 NSG/ルートの確認、`netstat` での Listen 確認、直接アクセスでの応答確認を提示。
- 「構成変更後」という契機から、変更点（NSG/UDR、HTTPS 化、プローブのホスト名/パス）の
  ロールバック観点での確認も提案。
- 参考 Microsoft Learn URL を明記。
- 追加情報（Details メッセージ、変更内容、プール種別/プロトコル）を Issue で依頼。

## 備考

- Microsoft Learn MCP が当該セッションで未提供だったため、公式ドキュメントを直接参照して
  根拠 URL を確定した（参照ドキュメントの ms.date は 2026-04-08 / updated_at 2026-04-27）。
- Details メッセージや構成変更の内容が判明すれば、原因を一意に特定できる可能性が高い。
