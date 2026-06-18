---
issue_number: 28
title: "Cosmos DB で 429 (Too Many Requests) が多発しスロットリングされる"
status: resolved
created: 2026-06-18
resolved: 2026-06-18
category: azure
related_knowledge:
  - knowledge/azure/2026-06-18-cosmos-db-429-throttling.md
---

# Cosmos DB で 429 (Too Many Requests) が多発しスロットリングされる

## 問い合わせ内容

- 製品: Azure Cosmos DB（API: NoSQL / Core、リージョン: Japan East）
- アクセス増加に伴い、アプリから Cosmos DB へのアクセスで 429 エラーが頻発しスロットリングされる。
- RU 設定不足なのか、パーティション設計の問題なのか切り分けられない。原因の特定方法を知りたい。

## 調査プロセス

1. `knowledge/` 内に Cosmos DB / 429 / throttling 関連の既存ナレッジがないか検索 → 該当なし。
2. Microsoft Learn (MCP) で公式ドキュメントを調査。
   - "Request rate too large (429)" トラブルシュート
   - Normalized RU Consumption による正規化使用率の監視とホットパーティション識別
   - Insights / 診断ログ (Kusto) による 429・RU 消費の分析
   - スループットのスケールアップのベストプラクティス、パーティション間スループット再配分
3. RU/s 不足とホットパーティションを切り分ける手順を整理し、ナレッジ化。

## 調査結果

- **確認できた事実（公式ドキュメント）**:
  - 429 は要求が RU/s を超過してレート制限された状態。原因は「全体的な RU/s 不足」と「ホットパーティション」に大別され、対応が異なる。
  - 本番では 429 が 1〜5% かつレイテンシ許容内なら健全（対処不要のことが多い）。SDK は `x-ms-retry-after-ms` に従い自動リトライするため、アプリ側で 429 が見えない場合もある。
  - ホットパーティションは「コンテナー全体の消費 RU/s はプロビジョニング値未満なのに、特定パーティションへの要求が 429」になる症状。Normalized RU Consumption (%) By PartitionKeyRangeID で識別する。
  - autoscale では 1 パーティションだけ 100% でも全体が上限未満で 429 が発生し得る。
  - 単一のホットパーティションキーが原因の場合、パーティション分割・再配分では解決せず、キー設計（合成パーティションキー等）の見直しが本質的解決策。
- **本 Issue 固有の状況**:
  - 「アクセス増加に伴い」発生という記述のみで、Normalized RU Consumption の偏り・診断ログ・対象コンテナーの RU/s 設定・パーティションキーが未提供。
  - このため**原因（RU/s 不足かホットパーティションか）は現時点で断定不可**。切り分け手順を提示する方針とした。

## 回答内容

- 原因を断定せず、以下の切り分けを案内:
  1. Insights → Requests で 429 の割合とレイテンシを確認（1〜5% かつ許容内なら健全）。
  2. Insights → Throughput → Normalized RU Consumption (%) By PartitionKeyRangeID でホットパーティションの有無を判定（特定 PKRangeId だけ高い＝ホット、全体均等に高い＝RU/s 不足）。
  3. 診断ログ (Kusto) でどの操作・キーが 429／高 RU かを特定。
- 分岐後の対応（RU/s 増、パーティションキー設計見直し／合成キー、スループット再配分、CosmosClient シングルトン化等）をナレッジに整理。
- 追加で確認したい情報（対象コンテナーの RU/s 設定・autoscale 有無、パーティションキー、Normalized RU の偏り、429 の割合とアクセスパターン）を Issue で質問。

## 備考

- パーティションキーは既存コンテナーでは変更不可。設計見直しは新コンテナー作成＋移行を伴う。
- 診断ログは Log Analytics の取り込み課金が発生するため、デバッグ時のみ有効化を推奨。
