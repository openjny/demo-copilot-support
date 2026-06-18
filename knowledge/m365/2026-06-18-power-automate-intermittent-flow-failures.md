---
title: "Power Automate クラウドフローが断続的に失敗する場合の調査手順（SharePoint/メール混在）"
category: "m365"
tags: ["power-automate", "cloud-flow", "sharepoint", "troubleshooting", "throttling", "retry"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [14]
sources:
  - "https://learn.microsoft.com/en-us/power-automate/fix-flow-failures"
  - "https://learn.microsoft.com/en-us/power-automate/guidance/coding-guidelines/error-handling"
  - "https://learn.microsoft.com/en-us/power-automate/limits-and-config"
  - "https://learn.microsoft.com/en-us/sharepoint/dev/general-development/how-to-avoid-getting-throttled-or-blocked-in-sharepoint-online"
---

# Power Automate クラウドフローが断続的に失敗する場合の調査手順（SharePoint/メール混在）

## 概要

同一フローで「成功する回」と「失敗する回」が混在し、かつ失敗ステップが固定されない場合は、単一設定ミスよりも「一時的要因（過負荷・一時障害・認証状態の揺れ）」を優先して切り分ける。

Power Automate 公式ドキュメントでは、まず実行履歴の失敗ランを特定してエラー詳細（HTTP ステータス、コネクタ応答、再試行有無）を確認することが推奨されている。

## 詳細

### まず確認できる事実（公式ドキュメントで確認可能）

- 実行履歴から失敗ランを開き、失敗アクションの詳細を確認できる。
- `401/403` は認証・認可系の失敗として扱われる。
- `500/502` は一時的失敗として再実行で回復する場合がある。
- Retry policy（固定/指数バックオフ）と Run after を使って、一時的失敗に強いフローにできる。
- フローは継続的にスロットリングされると停止対象になり得る（`limits-and-config`）。
- SharePoint Online は負荷時に `429` または `503` を返し、`Retry-After` を尊重した再試行が必要。

### 断続失敗で可能性が高い原因候補（本事象では未確定）

> 以下は **候補**。Issue 記載情報だけでは一意に断定できないため、切り分けで確定する。

1. **一時的なスロットリング/サービス負荷（SharePoint または関連コネクタ）**
   - 根拠: 失敗ステップが固定されず、同一フローでも成功/失敗が混在。
   - 判定キー: 失敗アクションに `429` / `503`、または再試行痕跡があるか。
2. **接続トークンの不安定化（期限切れ・再認証必要・条件付きアクセス影響）**
   - 根拠: 既存フローでも `401/403` が断続発生することがある。
   - 判定キー: 失敗時レスポンスに `Unauthorized` / `Forbidden`。
3. **フロー設計上の同時実行過多（トリガー並列実行、Apply to each 並列）**
   - 根拠: 高並列時に複数コネクタで断続エラーが出やすい。
   - 判定キー: 短時間に失敗が集中し、並列度を下げると改善するか。

## 手順

1. **失敗ランを 10〜20 件抽出して分類**
   - 実行履歴で失敗ランを開き、失敗アクションごとの HTTP ステータス/エラー文言を記録する。
   - `429/503`、`401/403`、`400/404`、`5xx` に分類する。
2. **分類結果で原因を絞る**
   - `429/503` が多い → スロットリング/一時負荷が有力。
   - `401/403` が多い → 接続再認証・権限・CA ポリシー確認が有力。
   - `400/404` が多い → データ不整合やアクション設定不備が有力。
3. **再発防止設定を段階的に適用**
   - 失敗頻度の高いアクションに Retry policy を設定/見直し（指数バックオフ推奨）。
   - Run after + Scope（Try/Catch）で通知・記録・安全終了を実装。
   - トリガー/ループの並列度を下げ、ピーク時の要求集中を緩和。
4. **効果検証**
   - 変更前後で失敗率、`429/503` 件数、再試行後成功率を比較。
   - 2〜3 日の実行履歴で再発有無を確認する。

## よくある原因

- コネクタまたはバックエンドの一時的な負荷上昇（`429/503`）
- 接続情報の失効・再認証不足・権限変更
- 並列実行や高頻度トリガーによる短時間集中アクセス
- エラー時の分岐（Run after）/再試行（Retry policy）未整備

## 参考リンク

- [Troubleshoot a cloud flow - Power Automate](https://learn.microsoft.com/en-us/power-automate/fix-flow-failures)
- [Employ robust error handling - Power Automate](https://learn.microsoft.com/en-us/power-automate/guidance/coding-guidelines/error-handling)
- [Limits and configuration in Power Automate](https://learn.microsoft.com/en-us/power-automate/limits-and-config)
- [Avoid getting throttled or blocked in SharePoint Online](https://learn.microsoft.com/en-us/sharepoint/dev/general-development/how-to-avoid-getting-throttled-or-blocked-in-sharepoint-online)
