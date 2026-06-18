---
title: "Azure Application Gateway (Standard_v2) でバックエンドが Unhealthy になる原因の切り分け"
category: "azure"
tags: ["application-gateway", "backend-health", "health-probe", "unhealthy", "502", "nsg", "udr", "tls"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [25]
sources:
  - "https://learn.microsoft.com/en-us/troubleshoot/azure/application-gateway/application-gateway-backend-health-troubleshooting"
  - "https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview"
  - "https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-create-probe-portal"
  - "https://learn.microsoft.com/en-us/powershell/module/az.network/get-azapplicationgatewaybackendhealth"
---

# Azure Application Gateway (Standard_v2) でバックエンドが Unhealthy になる原因の切り分け

## 概要

Application Gateway は、バックエンドプールに登録されたサーバーへ定期的にヘルスプローブを
送信し、応答が正常なサーバーにのみトラフィックを転送する。プローブが失敗すると、その
サーバーは **Unhealthy** とマークされ、リクエストは転送されなくなる。プール内の全サーバーが
Unhealthy になると、Application Gateway はクライアントへ **502 Bad Gateway** を返し、
サイトにアクセスできなくなる（出典:
[Troubleshoot backend health issues](https://learn.microsoft.com/en-us/troubleshoot/azure/application-gateway/application-gateway-backend-health-troubleshooting)）。

Unhealthy には複数の原因があり、設定（プローブ・HTTP 設定）だけを見ても一意に特定できない。
**まず「バックエンド正常性（Backend Health）」の Details 列に表示されるメッセージを確認し、
そのメッセージごとに切り分ける**のが公式に示された定石である。本ナレッジでは、
そのメッセージと原因・対処を整理する。

## 詳細

### 既定プローブの挙動（重要な前提）

カスタムプローブを設定していない場合、既定プローブは
`<protocol>://127.0.0.1:<port>/` の形式で送信される。プロトコルと宛先ポートは
**HTTP 設定（Backend HTTP settings）から継承**される。正常と見なされるのは
**HTTP ステータスコード 200〜399** のみ。別のホスト名・パス・ステータスコードを正常として
扱いたい場合は、カスタムプローブを作成して HTTP 設定に関連付ける必要がある（出典:
バックエンド正常性トラブルシューティング）。

> 確認できた事実: 「ヘルスプローブ設定は確認した」とのことだが、Unhealthy の原因は
> プローブ設定そのものとは限らず、ネットワーク経路・バックエンドの応答コード・TLS/証明書
> など複数の層に分かれる。次の「メッセージ別の原因」で切り分ける。

### バックエンド正常性の Details メッセージ別の原因と対処

公式ドキュメントで定義されている主なメッセージは以下のとおり。**Details 列の文言が
そのまま原因の手がかり**になる。

| Details のメッセージ（要旨） | 想定原因 | 主な対処 |
| --- | --- | --- |
| Backend server timeout（応答がタイムアウト閾値超） | バックエンド/アプリの応答遅延、依存先（DB 等）の遅延 | バックエンドへ直接アクセスし応答時間を計測。カスタムプローブで timeout を応答時間より大きく設定 |
| DNS resolution error（FQDN を解決できない） | バックエンドプールが FQDN 型で、DNS 解決に失敗 | FQDN の正当性確認、VNet のカスタム DNS 設定確認、ローカル/同一 VNet の VM から名前解決を検証 |
| TCP connect error（指定ポートに接続不可） | **NSG / UDR / ファイアウォールによる遮断**、バックエンドが当該ポートで待ち受けていない | NSG（バックエンド NIC/サブネット、AppGw サブネットのアウトバウンド）、UDR、OS ファイアウォール、`netstat` での Listen 確認 |
| HTTP status code mismatch（期待 200-399 以外を受信） | バックエンドが 401/403/404/405/500/503 等を返している、プローブのパスが不適切 | 受信コードに応じて対処（下表）。正当なコードなら status code match を持つカスタムプローブを作成 |
| HTTP response body mismatch（本文が一致しない） | カスタムプローブの本文一致条件と実際の応答本文が不一致 | プローブパスへ直接アクセスして本文を確認し、一致文字列を修正 |
| CN does not match（証明書 CN/SNI 不一致） | HTTPS バックエンドで、プローブ/HTTP 設定のホスト名が証明書の CN/SAN と不一致 | HTTP 設定で「特定ホスト名で上書き」/「バックエンドターゲットからホスト名を選択」、またはカスタムプローブの host を CN に合わせる |
| Backend certificate has expired（証明書期限切れ） | バックエンドのサーバー証明書（または中間/ルート）が失効 | 証明書を更新し、`Leaf > Intermediate > Root` の完全なチェーンで再インストール |
| Intermediate/Leaf certificate missing（チェーン不備） | バックエンドの証明書チェーンが不完全・順序不正 | 完全なチェーンを正しい順序でバックエンドにインストール |
| Server cert not issued by well-known CA（信頼されない CA） | プライベート CA 発行の証明書だがルート CA を AppGw に未登録 | HTTP 設定で「well-known でない CA」を選び、ルート CA 証明書（.CER）をアップロード |

#### HTTP ステータスコード不一致時の受信コード別対処

| 受信コード | 対処 |
| --- | --- |
| 401 | バックエンドが認証を要求。プローブは資格情報を渡せないため、status code match に 401 を許可するか認証不要のパスをプローブ対象にする |
| 403 | アクセス禁止。プローブパスへのアクセス可否を確認 |
| 404 | ページなし。ホスト名/パスをアクセス可能な値へ変更 |
| 405 | プローブは HTTP GET。サーバーが GET を許可しているか確認 |
| 500 | 内部エラー。バックエンドのアプリ/サービス稼働状況を確認 |
| 503 | サービス利用不可。バックエンドのアプリ/サービス稼働状況を確認 |

## 手順

### 1. バックエンド正常性の Details メッセージを確認する（最優先）

1. Azure portal で対象 Application Gateway を開き、**[バックエンド正常性 (Backend health)]** を表示する。
2. Unhealthy のサーバー行の **Details 列**のメッセージを記録する。これが原因切り分けの起点になる。
3. portal の代わりに PowerShell でも取得できる:

   ```azurepowershell
   Get-AzApplicationGatewayBackendHealth -Name "appgw1" -ResourceGroupName "rgOne"
   ```

   > 補足: ユーザーにバックエンド正常性の閲覧権限がない場合、`No results.` と表示される。

### 2. メッセージに対応する層を検証する

- **TCP connect error** の場合（ネットワーク到達性）:

  ```azurepowershell
  # AppGw サブネットの NSG/サブネット構成を確認
  $vnet = Get-AzVirtualNetwork -Name "vnetName" -ResourceGroupName "rgName"
  Get-AzVirtualNetworkSubnetConfig -Name appGwSubnet -VirtualNetwork $vnet

  # バックエンド NIC の有効な NSG / 有効なルート（UDR）を確認
  Get-AzEffectiveNetworkSecurityGroup -NetworkInterfaceName "nic1" -ResourceGroupName "testrg"
  Get-AzEffectiveRouteTable -NetworkInterfaceName "nic1" -ResourceGroupName "testrg"
  ```

  バックエンドサーバー上では `netstat` で対象ポートを Listen しているか、OS ファイアウォールで
  当該ポートの受信が許可されているかを確認する。

- **HTTP status code / body mismatch** の場合（アプリ層）:
  バックエンドへ**直接**（Application Gateway を経由せず）プローブパスにアクセスし、
  返るステータスコード・本文を確認する。

- **CN/証明書系**の場合（TLS 層）:
  バックエンドへ直接アクセスして証明書の CN/SAN・有効期限・チェーンを確認し、
  HTTP 設定のホスト名や信頼ルート CA の登録を見直す。

### 3. 必要に応じてカスタムプローブで調整する

タイムアウト延長、許可ステータスコードの追加、ホスト名/パスの指定が必要な場合は
カスタムプローブを作成し HTTP 設定へ関連付ける（出典:
[Create a custom probe by using the portal](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-create-probe-portal)）。

## よくある原因

- NSG（バックエンド NIC/サブネット、または AppGw サブネットのアウトバウンド）・UDR・OS ファイアウォールによるプローブポートの遮断（TCP connect error）
- バックエンドが当該ポートで待ち受けていない（バインド/サービス停止）
- プローブのパス/ホスト名が不適切で 404/403 等が返る、または認証要求で 401 が返る
- バックエンド/依存先の応答遅延によるタイムアウト
- HTTPS バックエンドでの証明書 CN/SNI 不一致、証明書の期限切れ・チェーン不備・信頼されない CA

## 参考リンク

- [Troubleshoot backend health issues in Azure Application Gateway](https://learn.microsoft.com/en-us/troubleshoot/azure/application-gateway/application-gateway-backend-health-troubleshooting)
- [Application Gateway health probes overview](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview)
- [Create a custom probe for Application Gateway by using the portal](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-create-probe-portal)
- [Get-AzApplicationGatewayBackendHealth](https://learn.microsoft.com/en-us/powershell/module/az.network/get-azapplicationgatewaybackendhealth)
