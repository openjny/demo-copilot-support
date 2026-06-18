---
title: "Azure App Service (Linux/Node.js) でアクセス集中時に断続的な 503 が出る場合の切り分け"
category: "azure"
tags: ["app-service", "503", "linux", "nodejs", "performance", "scale"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [4]
sources:
  - "https://learn.microsoft.com/en-us/azure/app-service/troubleshoot-http-502-http-503"
  - "https://learn.microsoft.com/en-us/azure/app-service/overview-diagnostics"
  - "https://learn.microsoft.com/en-us/azure/app-service/monitor-instances-health-check"
  - "https://learn.microsoft.com/en-us/azure/app-service/manage-scale-up"
  - "https://learn.microsoft.com/en-us/azure/app-service/app-service-best-practices"
  - "https://learn.microsoft.com/en-us/azure/app-service/configure-common"
---

# Azure App Service (Linux/Node.js) でアクセス集中時に断続的な 503 が出る場合の切り分け

## 概要

Azure App Service の 503 は、Microsoft Learn 上で「アプリ側の応答不能（高 CPU/高メモリ、長時間リクエスト、例外など）」が主要因として整理されています。再起動で一時回復する場合は、恒久対応ではなくワーカープロセスの状態リセットで回復している可能性があります。

本ナレッジは、アクセス集中時に 503 が断続的に出るケースを対象に、原因候補を断定せず、公式ドキュメントで確認できる事実と実運用で有効な切り分け手順を分けて整理します。

## 詳細

### ドキュメントで確認できる事実

- App Service の 502/503 では、アプリレベル要因（高 CPU、高メモリ、応答遅延、例外）が代表的な原因。
- 調査は「監視（Metrics）→ データ収集（Diagnose and solve problems / Kudu）→ 緩和（スケール、再起動、Auto-heal）」の順で進める。
- Health check は 1 分間隔で指定パスを監視し、不健康判定のインスタンスをロードバランサーから除外できる。
- Always On を無効にすると、無通信 20 分でアンロードされ、ウォームアップ遅延が発生しうる。
- CPU が継続的に高い/スパイクする場合、スケールアップまたはスケールアウトを検討する。

### 本ケースでの原因候補（断定しない）

1. **特定インスタンスの CPU ボトルネック**（根拠あり）  
   - 事象: Application Insights で特定インスタンスの CPU 高騰を観測。  
   - 503 と同時間帯に CPU 使用率・応答時間が連動していれば可能性が高い。
2. **スケール不足/スケール反応遅延**（根拠あり）  
   - 事象: 朝方のアクセス集中時に断続発生。  
   - B2 で同時接続/処理量が増えた場合、一時的に処理能力不足となる可能性。
3. **非健全インスタンスが配信系に残留**（根拠あり）  
   - Health check 未設定、または妥当でないパス設定だと、不健康インスタンスへ配信継続しうる。
4. **アプリ実装由来の CPU 偏りや接続枯渇**（推測）  
   - Node.js の外部向け HTTP 接続再利用不足や、重い同期処理でイベントループが詰まると、インスタンス偏在時に 503 を誘発しうる。

## 手順

1. **時系列相関の確認（必須）**  
   - App Service Metrics で `CPU Time` / `Memory Working Set` / `Requests` / `Response Time` を 503 発生時間帯で重ねて確認。  
   - 判定: 503 と同時に CPU が張り付くなら、CPU 起因の可能性が高い。

2. **Diagnose and solve problems で根拠を補強**  
   - `Availability and Performance` の `Web App Down` / `CPU Usage` / `Memory Usage` を確認。  
   - 判定: 異常インスタンスや構成リスクの指摘が出るかを確認。

3. **Health check の有効化/見直し**  
   - `/health` などの軽量エンドポイントを作成し、200/500 を適切に返す。  
   - `WEBSITE_HEALTHCHECK_MAXPINGFAILURES` などの閾値を要件に合わせ調整。  
   - 判定: 不健康インスタンス除外後に 503 が減るか確認。

4. **スケール戦略の適用**  
   - まずは短期緩和として **Scale out**（インスタンス増）を優先。  
   - CPU が全体的に高止まりする場合は **Scale up**（上位プラン）を実施。  
   - 判定: ピーク帯の 503 率、P95/P99 応答時間が改善するか確認。

5. **設定・実装の再確認**  
   - `Always On` を有効化（本番）。  
   - ステートレス構成なら `Session affinity` を Off 検討。  
   - Node.js で外部 HTTP の keep-alive 利用や重い同期処理の有無を点検。

6. **暫定回避策と恒久対応を分離**  
   - 再起動は暫定回避として扱い、原因特定後にスケール/実装改善を恒久対応とする。

## よくある原因

- ピーク時のみ CPU が上限近くに達し、処理待ちで 503 が発生
- Health check 未設定で不健康インスタンスへの振り分けが継続
- 単一インスタンス運用や余裕の少ないプランでトラフィックスパイクに追従できない
- Node.js アプリで外部接続・同期処理がボトルネック化

## 参考リンク

- [Fix HTTP 502 and HTTP 503 Errors - Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/troubleshoot-http-502-http-503)
- [Troubleshoot with Diagnostics - Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/overview-diagnostics)
- [Monitor App Service instances by using Health check](https://learn.microsoft.com/en-us/azure/app-service/monitor-instances-health-check)
- [Scale up an app in Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/manage-scale-up)
- [Best practices for Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/app-service-best-practices)
- [Configure common settings for App Service](https://learn.microsoft.com/en-us/azure/app-service/configure-common)
