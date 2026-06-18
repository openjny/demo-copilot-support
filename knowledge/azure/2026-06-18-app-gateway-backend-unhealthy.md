---
title: "Azure Application Gateway でバックエンドが Unhealthy になる原因の切り分け"
category: "azure"
tags: ["application-gateway", "backend-health", "health-probe", "502", "troubleshooting", "nsg", "tls"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [25]
sources:
  - "https://learn.microsoft.com/en-us/troubleshoot/azure/application-gateway/application-gateway-backend-health-troubleshooting"
  - "https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview"
  - "https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-create-probe-portal"
  - "https://learn.microsoft.com/en-us/powershell/module/az.network/get-azapplicationgatewaybackendhealth"
---

# Azure Application Gateway でバックエンドが Unhealthy になる原因の切り分け

## 概要

Azure Application Gateway (Standard_v2 / WAF_v2) では、ヘルスプローブがバックエンドサーバーへ
定期的にリクエストを送り、応答に基づいてバックエンドの正常性 (Backend health) を判定する。
プローブが期待どおりの応答を得られないとそのバックエンドは **Unhealthy** とマークされ、
トラフィックがルーティングされなくなる。プール内の**全バックエンドが Unhealthy** になると、
Application Gateway はクライアントへ **HTTP 502 Bad Gateway** を返す。

Unhealthy の原因はネットワーク到達性・アプリ応答・TLS/証明書など複数の層にまたがるため、
症状だけで断定はできない。公式トラブルシューティングの定石は、まず
**[バックエンド正常性] の Details 列のメッセージを確認**し、そのメッセージから原因を絞り込むこと。
本記事は Details メッセージ別の原因と切り分け手順を整理する。

## 詳細

### 既定 (default) プローブの仕様

カスタムプローブを設定していない場合、Application Gateway は各バックエンドプール／HTTP 設定の
組み合わせに対して**既定プローブ**を自動的に発行する。既定プローブには次の仕様がある
（出典: [Health probes overview](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview)）。

- リクエスト先は `<protocol>://127.0.0.1:<port>/` 形式。
- **プロトコルとポートは関連付けられた HTTP 設定を継承**する（HTTP 設定が HTTPS なら HTTPS でプローブ）。
- **HTTP ステータスコード 200〜399 を受信した場合のみ Healthy** とみなす。
- ホスト名は既定で `127.0.0.1`。別ホスト名・別パス・別の許容ステータスコードが必要なら
  **カスタムプローブ**を作成する必要がある。

この既定動作を理解していないと、「アプリは動いているのに Unhealthy」というケース
（例: ルートパス `/` が 301/302 以外のリダイレクトや 403/401 を返す、特定ホストヘッダーを要求する）
の原因を見誤りやすい。

### バックエンド正常性 (Backend health) の確認方法

原因特定の起点は **Details 列のメッセージ**。次のいずれかで取得する。

- Azure portal → 対象 Application Gateway → **[バックエンド正常性]** → Unhealthy 行の **Details**。
- PowerShell:

  ```powershell
  Get-AzApplicationGatewayBackendHealth -Name "<appgw>" -ResourceGroupName "<rg>"
  ```

  権限が不足していると `No results.` と表示される
  （出典: [Get-AzApplicationGatewayBackendHealth](https://learn.microsoft.com/en-us/powershell/module/az.network/get-azapplicationgatewaybackendhealth)）。

### Details メッセージ別の原因と切り分け

以下はすべて公式トラブルシューティング
（[Troubleshoot backend health issues](https://learn.microsoft.com/en-us/troubleshoot/azure/application-gateway/application-gateway-backend-health-troubleshooting)）
に記載のあるシナリオ。「このメッセージならこの原因 → この確認」と対応づけて切り分ける。

| Details メッセージ（要旨） | 想定原因 | 次に確認すること（期待される結果） |
| --- | --- | --- |
| **TCP connect error**（指定ポートへ接続不可） | NSG / UDR / ファイアウォールによる遮断、または当該ポートで未 Listen | ① バックエンド NIC・サブネットの NSG でプローブポートのインバウンド許可 ② AppGw サブネットの NSG アウトバウンド許可 ③ UDR が経路を逸らしていないか ④ バックエンド上で対象ポートが LISTENING か・OS ファイアウォール許可（許可・Listen なら正常） |
| **HTTP status code mismatch**（200-399 以外を受信） | プローブのパス／ホスト名が不適切、認証要求 (401)・403/404/500/503 等 | バックエンドへ**直接**プローブパスへアクセスし応答コードを確認。401→認証不要パスへ／コード許可、404→ホスト名・パス見直し、500/503→アプリ稼働確認 |
| **DNS resolution error**（FQDN 解決不可） | バックエンドが FQDN 型で名前解決に失敗（カスタム DNS 設定など） | FQDN の正当性、VNet のカスタム DNS 設定、同一 VNet VM からの名前解決可否を確認 |
| **Backend server timeout** | バックエンドや依存先（DB 等）の応答遅延 | バックエンドへ直接アクセスし応答時間を計測。必要ならカスタムプローブで timeout を延長 |
| **CN does not match / 証明書期限切れ / チェーン不備 / 信頼されない CA** | HTTPS バックエンドでの SNI 用ホスト名不一致、証明書失効・チェーン不正・プライベート CA 未登録 | バックエンドへ直接アクセスし証明書の CN/SAN・有効期限・チェーンを確認。HTTP 設定のホスト名指定や信頼ルート CA(.CER) の登録を見直し |

### ネットワーク到達性 (TCP connect error) の確認に有用なコマンド

NSG / UDR の有効ルールを確認するには次の PowerShell が公式に案内されている。

```powershell
$vnet = Get-AzVirtualNetwork -Name "<vnet>" -ResourceGroupName "<rg>"
Get-AzVirtualNetworkSubnetConfig -Name "<appGwSubnet>" -VirtualNetwork $vnet
Get-AzEffectiveNetworkSecurityGroup -NetworkInterfaceName "<nic>" -ResourceGroupName "<rg>"
Get-AzEffectiveRouteTable -NetworkInterfaceName "<nic>" -ResourceGroupName "<rg>"
```

## 手順

### 1. Details メッセージを確認する（起点）

1. portal の [バックエンド正常性]、または `Get-AzApplicationGatewayBackendHealth` で
   Unhealthy 行の Details メッセージを取得する。

### 2. メッセージに応じて原因を切り分ける

1. 上表の「Details メッセージ → 想定原因 → 次に確認すること」に従って分岐する。
2. **バックエンドへ直接アクセス**（同一 VNet 内の VM などから、プローブと同じプロトコル／ポート／パス／
   ホストヘッダーで）して、Application Gateway を介さない素の応答（ステータスコード・応答時間・証明書）を
   確認すると、ネットワーク層とアプリ層の切り分けが明確になる。

### 3. 既定プローブで不都合なら カスタムプローブを作成する

ルートパス以外をヘルスチェックしたい、特定ホストヘッダーが必要、200-399 以外を許容したい場合は
カスタムプローブを作成する（出典:
[Create a custom probe](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-create-probe-portal)）。

1. [正常性プローブ] でプロトコル・ホスト・パス・間隔・タイムアウト・異常しきい値を設定する。
2. 必要に応じて **正常な HTTP 応答の状態コード範囲**（例: `200-401`）を指定する。
3. HTTP 設定からこのカスタムプローブを参照させる。

### 4. 構成変更後に発生した場合は変更点を切り戻し観点で確認する

直近の変更（NSG/UDR 変更、HTTP 設定の HTTP→HTTPS 化、プローブのホスト名/パス変更、証明書差し替え、
バックエンドターゲット変更）を洗い出し、変更前後で何が壊れたかを突き合わせる。

## よくある原因

- NSG / UDR / OS ファイアウォールによりプローブポートへ到達できない（TCP connect error）。
- 既定プローブがルートパス `/` を叩くが、アプリが 200-399 以外（リダイレクト以外・401/403/404/500/503）を返す。
- HTTPS バックエンドで証明書の CN/SAN 不一致、期限切れ、チェーン不備、プライベート CA 未登録。
- FQDN 型バックエンドの名前解決失敗（カスタム DNS 設定の不備など）。
- バックエンドや依存先の応答遅延によるタイムアウト。

## 参考リンク

- [Troubleshoot backend health issues in Azure Application Gateway](https://learn.microsoft.com/en-us/troubleshoot/azure/application-gateway/application-gateway-backend-health-troubleshooting)
- [Application Gateway health probes overview](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview)
- [Create a custom probe for Application Gateway by using the portal](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-create-probe-portal)
- [Get-AzApplicationGatewayBackendHealth](https://learn.microsoft.com/en-us/powershell/module/az.network/get-azapplicationgatewaybackendhealth)
