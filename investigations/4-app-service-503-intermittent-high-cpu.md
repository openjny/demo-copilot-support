---
issue_number: 4
title: "[問い合わせ] Azure App Service で断続的に 503 エラーが発生する"
status: "resolved"
created: "2026-06-18"
resolved: "2026-06-18"
category: "azure"
related_knowledge:
  - "knowledge/azure/2026-06-18-app-service-503-intermittent-high-cpu.md"
---

# [問い合わせ] Azure App Service で断続的に 503 エラーが発生する

## 問い合わせ内容

Azure App Service (Linux/B2) 上の Node.js アプリで、アクセス集中時に断続的な 503 が発生。Application Insights 上は特定インスタンスの CPU 高騰が見えるが原因は未確定。再起動で一時回復。

## 調査プロセス

1. `knowledge/` を確認し、既存ナレッジ未整備であることを確認。  
2. Microsoft Learn の公式ドキュメント（502/503 トラブルシュート、Diagnostics、Health check、スケール、ベストプラクティス）を確認。  
3. 断定を避け、事実（公式ドキュメント）と未確認事項（実環境依存）を分離した回答方針を整理。

## 調査結果

- 公式ドキュメント上、503 は高 CPU/高メモリ/長時間リクエスト/例外などアプリレベル要因で発生しうる。  
- 調査手順は「監視 → 診断データ収集 → 緩和」の順が推奨。  
- Health check を設定すると不健康インスタンスを配信対象から除外できる。  
- Always On 無効時のアンロードによる遅延リスク、CPU 高騰時の scale up/out 推奨が明示されている。  
- 本件は「特定インスタンス CPU 高騰」「ピーク時発生」「再起動で一時回復」から、CPU ボトルネック/スケール不足/不健康インスタンス残留が主要候補。

## 回答内容

Issue への回答として、以下を提示する想定で整理。

- **確認できた事実**: App Service 503 の代表原因、診断導線、Health check/scale/Always On の挙動。  
- **未確認（切り分け必要）**: CPU 高騰の根本原因（コード、依存先遅延、接続枯渇など）は現状情報だけでは断定不可。  
- **原因候補（優先順）**:  
  1. 特定インスタンス CPU ボトルネック  
  2. スケール不足またはスケール反応遅延  
  3. Health check 未整備による不健康インスタンス残留  
  4. Node.js 実装由来（推測）  
- **推奨対応**:  
  1. Metrics と 503 時系列相関の確認  
  2. Diagnose and solve problems で異常インスタンス/推奨事項確認  
  3. Health check 有効化と閾値見直し  
  4. ピーク帯に向けた scale out、必要時 scale up  
  5. Always On 有効化、ステートレスなら Session affinity Off 検討

## 備考

- セキュリティ設定変更（認証経路・IP 制限）に触れる場合は影響範囲を事前評価する。  
- 再起動は暫定回避であり、恒久対応は計測根拠に基づくスケール/実装改善で実施する。
