---
issue_number: 25
title: "[問い合わせ] Application Gateway のバックエンドが Unhealthy になる"
status: "open"
created: "2026-06-18"
resolved: ""
category: "azure"
related_knowledge:
  - "knowledge/azure/2026-06-18-app-gateway-backend-unhealthy.md"
---

# [問い合わせ] Application Gateway のバックエンドが Unhealthy になる

## 問い合わせ内容

Azure Application Gateway (Standard_v2 / Japan East) のバックエンドプールに登録した VM が、
バックエンド正常性で **Unhealthy** と表示されサイトにアクセスできない。**構成変更後**に発生。
ヘルスプローブの設定は確認したが、どこを見ればよいか分からない。

## 調査プロセス

1. 既存ナレッジ（`knowledge/azure`）を確認 → App Gateway のバックエンド正常性に関する
   再利用可能なナレッジは存在せず（既存は App Service 503 の記事のみ）。
2. 公式トラブルシューティング／ドキュメントで原因と切り分けを調査。
   - Backend health のトラブルシューティング（Details メッセージ別の原因）
   - Health probes overview（既定プローブの仕様: `127.0.0.1`・HTTP 設定継承・200-399 のみ Healthy）
   - Create a custom probe（ホスト名/パス/許容ステータスコードのカスタマイズ）
   - Get-AzApplicationGatewayBackendHealth（PowerShell での正常性確認）
3. 問い合わせ時点で判明している事実と、原因特定に必要な未確認情報を整理。

## 調査結果

- **確認できた事実（公式根拠）**: プール内の全バックエンドが Unhealthy になると
  Application Gateway は **502 Bad Gateway** を返す。今回の「サイトにアクセスできない」と整合する。
- **原因は現時点で一意に特定できない**。Unhealthy の原因はネットワーク到達性・アプリ応答・
  TLS/証明書など複数層にまたがり、問い合わせには切り分けに必要な情報
  （バックエンド正常性の **Details メッセージ**、構成変更の**具体的内容**、
  バックエンド種別／プローブのプロトコル）が含まれていないため。
- 公式の定石は、まず **[バックエンド正常性] の Details 列のメッセージ**を確認し、そこから
  原因を絞り込むこと。Details メッセージ（TCP connect error / HTTP status code mismatch /
  DNS resolution error / Backend server timeout / 証明書系）ごとに原因と次の確認が決まる。
- 既定プローブは `127.0.0.1` のルートパスへ HTTP 設定継承のプロトコル／ポートでアクセスし
  **200-399 のみ Healthy**。別ホスト名/パス/許容コードが必要ならカスタムプローブが必要。
- 「構成変更後」の発生という点から、直近の変更（NSG/UDR、HTTP 設定の HTTPS 化、プローブの
  ホスト名/パス、証明書差し替え、バックエンドターゲット変更）の切り戻し観点での確認も有効
  （これは経験則に基づく提案で、ドキュメントで断定されたものではない）。

詳細は作成したナレッジを参照:
`knowledge/azure/2026-06-18-app-gateway-backend-unhealthy.md`

## 回答内容

Issue #25 に以下を回答済み（断定回避・切り分け重視）:

- 状況整理（確認できた事実: 全 Unhealthy 時の 502 / 未確認: Details メッセージ・変更内容）
- 最優先で確認すべきこと（[バックエンド正常性] の Details メッセージ、`Get-AzApplicationGatewayBackendHealth`）
- 既定プローブの仕様（127.0.0.1・HTTP 設定継承・200-399 のみ Healthy）
- Details メッセージ別の原因候補と切り分け表（各候補に根拠を明記）
- ネットワーク到達性確認の PowerShell（NSG/UDR の有効ルール確認）
- 追加で提供してほしい情報（Details 全文・変更内容・バックエンド種別／プローブプロトコル）
- 参考リンク（Microsoft Learn 4 件）とナレッジ／PR への参照

## 備考

- 原因は未確定のため status は `open`。ユーザーから Details メッセージ等の追加情報が得られ次第、
  原因の特定まで踏み込み、解決時に status を `resolved` へ更新する想定。
- 各切り分けは公式ドキュメントに基づく。最終的な確定にはユーザー環境での Details メッセージと
  直接アクセス（同一 VNet からの素の応答）確認が必要。
- セキュリティ影響: NSG/UDR やプローブ許容ステータスコードの変更は到達範囲・公開範囲に影響し得るため、
  変更時は影響範囲を明示して最小限にとどめる。
