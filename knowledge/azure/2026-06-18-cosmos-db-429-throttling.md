---
title: "Azure Cosmos DB for NoSQL の 429 (Request rate too large) スロットリングの原因切り分けと対応"
category: azure
tags: [cosmos-db, nosql, throttling, 429, request-units, partition-key, autoscale]
created: 2026-06-18
updated: 2026-06-18
issue_ref: [28]
sources:
  - https://learn.microsoft.com/azure/cosmos-db/troubleshoot-request-rate-too-large
  - https://learn.microsoft.com/azure/cosmos-db/monitor-normalized-request-units
  - https://learn.microsoft.com/azure/cosmos-db/use-metrics
  - https://learn.microsoft.com/azure/cosmos-db/diagnostic-queries
  - https://learn.microsoft.com/azure/cosmos-db/scaling-provisioned-throughput-best-practices
  - https://learn.microsoft.com/azure/cosmos-db/how-to-redistribute-throughput-across-partitions
  - https://learn.microsoft.com/azure/cosmos-db/synthetic-partition-keys
---

# Azure Cosmos DB for NoSQL の 429 (Request rate too large) スロットリングの原因切り分けと対応

## 概要

Azure Cosmos DB for NoSQL で発生する HTTP 429「Request rate too large（要求レートが大きすぎます）」は、リクエストがプロビジョニング済みスループット（RU/s）を超過してレート制限されたことを示す。429 の原因は大きく「全体的な RU/s 不足」と「ホットパーティション（特定パーティションへの偏り）」に分けられ、対応策が異なる。**RU/s を上げる前に根本原因を特定すること**が公式に推奨されている。

なお、本番ワークロードでは **429 が全リクエストの 1〜5% 程度かつエンドツーエンドのレイテンシが許容範囲内であれば、RU/s を使い切れている健全な状態**とされ、必ずしも対処は不要。SDK は 429 を受けると `x-ms-retry-after-ms` ヘッダーに従って自動リトライするため、アプリ側で 429 が観測されない場合もある（Azure Monitor では計上される）。

> 本ナレッジは API for NoSQL を対象とする。API for MongoDB はエラーコード体系が異なるため別ドキュメントを参照。

## 詳細

### 429 が返る主なパターン

公式ドキュメントでは 429 のエラーメッセージにより原因が分かれる。

1. **Request rate is large.（要求レートが大きい）** — 最も一般的。RU/s の絶対的不足、またはホットパーティションが原因。
2. **高頻度のメタデータ要求** — コンテナー/データベースの作成・読み取り・一覧取得などの制御プレーン操作が多発。RU/s を上げても解決しない（システム予約の制限）。
3. **一時的なサービスエラー** — リトライで回復するトランジェントなもの。
4. `TXN_WAIT_FOR_TRANSACTION_END` — トランザクション関連。

### RU/s 不足 vs ホットパーティション

- Cosmos DB は**プロビジョニング済み RU/s を全物理パーティションに均等配分**する。
- **ホットパーティション**は、一部の論理パーティションキーに要求が集中し、その物理パーティションの RU/s を使い切る状態。症状として「コンテナー全体の消費 RU/s はプロビジョニング値より低いのに、特定パーティションへの要求が 429 になる」。
- このため「RU/s を上げる」対応はホットパーティションには効果が薄く（または非効率）、パーティションキー設計の見直しが必要になる。

### autoscale 利用時の注意

autoscale では、**正規化使用率（Normalized RU Consumption）が 100% に達したパーティションがある**と、コンテナー全体が最大 RU/s までスケールしていなくても 429 が発生し得る。正規化使用率は「全パーティションキーレンジ中の最大使用率」で定義されるため、1 つのパーティションがホットだと全体値が高く出る。

## 切り分け手順

原因を断定せず、以下の順で確認する。各ステップの結果で分岐を判断する。

### ステップ 1: 429 の割合とレイテンシを確認

- Azure portal → 対象 Cosmos DB アカウント → **Insights** → **Requests**（ステータスコード別リクエスト数）。
- 判断基準:
  - 429 が **1〜5% かつレイテンシ許容内** → 健全。対処不要の可能性が高い。
  - それ以上、またはレイテンシが許容外 → ステップ 2 以降へ。
- アプリ側で 429 が見えなくても Azure Monitor で計上されることがある（SDK の自動リトライのため）。

### ステップ 2: ホットパーティションの有無を確認（最重要の分岐点）

- Azure portal → **Insights** → **Throughput** → **Normalized RU Consumption (%) By PartitionKeyRangeID**。対象 DB / コンテナーでフィルター。
- 各 `PartitionKeyRangeId` は 1 つの物理パーティションに対応。
- 判断基準:
  - **特定の PartitionKeyRangeId だけ常時 100% 近く、他は 30% 以下** → **ホットパーティション**（→ ステップ 4 のキー設計見直しへ）。
  - **全 PartitionKeyRangeId が均等に高い** → **全体的な RU/s 不足**（→ ステップ 3 の RU/s 増へ）。

### ステップ 3: どの操作・どのキーが 429/高 RU を出しているか特定

- **診断ログ（Diagnostic Logs）**を一時的に有効化し、Log Analytics で分析する（取り込み量に応じて課金されるため、デバッグ時のみ有効化を推奨）。
- 429 のリクエストを特定する Kusto（resource-specific テーブル）:

  ```kusto
  CDBDataPlaneRequests
  | where StatusCode == "429"
  | summarize count() by OperationName, bin(TimeGenerated, 1m)
  | render timechart
  ```

- 物理パーティション別の RU 消費:

  ```kusto
  CDBPartitionKeyRUConsumption
  | where TimeGenerated >= ago(1d)
  // | where DatabaseName == "MyDB" and CollectionName == "MyCollection"
  | summarize sum(todouble(RequestCharge)) by toint(PartitionKeyRangeId)
  | render columnchart
  ```

- レスポンスヘッダー `x-ms-request-charge` で個々の操作の RU コストを確認できる。重いクエリ（フルスキャン、クロスパーティションクエリ、大きなレスポンス）が原因のこともある。

### ステップ 4: 原因に応じた対応（下記「手順」参照）

## 手順

### A. 全体的に RU/s が不足している場合

1. 対象コンテナー/データベースの RU/s を増やす（手動またはautoscale 上限の引き上げ）。
2. スケールアップの注意:
   - 「distinct な PartitionKeyRangeId 数 × 10,000 RU/s」以下への引き上げは**即時**完了する。
   - それを超える引き上げは物理パーティション追加を伴い、**非同期で最大 5〜6 時間**かかる場合がある。
3. クエリ最適化で RU 消費自体を下げる（インデックス設計、必要な列のみ取得、クロスパーティションクエリの削減）。
4. SDK 側のリトライ設定を確認（`MaxRetryAttemptsOnRateLimitedRequests` / `ThrottlingRetryOptions` 等）。アクセス増に対しバックオフが追いつかない場合は調整。

### B. ホットパーティションの場合

1. **パーティションキー設計を見直す**。要求量とストレージを均等分散するキーを選ぶ。
   - 悪い例: 書き込み中心の IoT データを `date` で分割 → 当日のデータが 1 パーティションに集中。
   - 改善: `id`（GUID やデバイス ID）や、`id` + `date` を組み合わせた[合成パーティションキー](https://learn.microsoft.com/azure/cosmos-db/synthetic-partition-keys)でカーディナリティを上げる。
   - マルチテナントで `tenantId` 分割し巨大テナントがある場合 → 大口テナントは専用コンテナー＋より粒度の細かいキー（例 `userId`）にする。
2. 既存コンテナーのパーティションキーは変更できないため、**新コンテナーを作成してデータ移行**する（[パーティションキー変更手順](https://devblogs.microsoft.com/cosmosdb/how-to-change-your-partition-key/)）。
3. 暫定対応として[パーティション間のスループット再配分（プレビュー）](https://learn.microsoft.com/azure/cosmos-db/how-to-redistribute-throughput-across-partitions)でホット物理パーティションに RU/s を寄せる（物理パーティションあたり最大 10,000 RU/s）。
   - 注意: **単一のホットパーティションキー**が原因の場合は、パーティション分割や再配分をしても同一キーのデータは同じ物理パーティションに残るため効果がない。この場合はキー設計の見直しが本質的な解決策。

### C. メタデータ要求のレート制限の場合

1. `CosmosClient` / `DocumentClient` は**静的なシングルトン**としてアプリのライフタイム全体で再利用する（初期化時に多くの RU を消費するメタデータ取得が走るため）。
2. データベース名・コンテナー名はキャッシュし、`ReadContainerAsync` 等のメタデータ呼び出しを毎回行わない。
3. メタデータ操作が必要な場合はバックオフを実装して送信レートを下げる。

## よくある原因

- プロビジョニング RU/s がアクセス増に追いついていない（全体不足）。
- パーティションキーの偏りによるホットパーティション（特定キー・特定日付・特定テナントへの集中）。
- 重いクエリ（クロスパーティション、フルスキャン、大きなレスポンス）による RU 消費の増大。
- `CosmosClient` を都度生成している、メタデータ操作の多発によるメタデータレート制限。
- autoscale で 1 パーティションだけ 100% に張り付き、全体は上限未満でも 429。

## 参考リンク

- [Diagnose and troubleshoot "Request rate too large" (429) exceptions](https://learn.microsoft.com/azure/cosmos-db/troubleshoot-request-rate-too-large)
- [Monitor normalized request units in Azure Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/monitor-normalized-request-units)
- [Monitor and debug with insights in Azure Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/use-metrics)
- [Advanced diagnostics queries (Kusto)](https://learn.microsoft.com/azure/cosmos-db/diagnostic-queries)
- [Best practices for scaling provisioned throughput (RU/s)](https://learn.microsoft.com/azure/cosmos-db/scaling-provisioned-throughput-best-practices)
- [Redistribute throughput across partitions (preview)](https://learn.microsoft.com/azure/cosmos-db/how-to-redistribute-throughput-across-partitions)
- [Create a synthetic partition key](https://learn.microsoft.com/azure/cosmos-db/synthetic-partition-keys)
