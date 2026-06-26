---
title: "OneDrive / SharePoint の 1 ファイルあたり最大アップロードサイズ（250 GB）"
category: "m365"
tags: ["onedrive", "sharepoint", "file-upload", "limits"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [20]
sources:
  - "https://learn.microsoft.com/en-us/office365/servicedescriptions/sharepoint-online-service-description/sharepoint-online-limits"
  - "https://support.microsoft.com/office/restrictions-and-limitations-in-onedrive-and-sharepoint-64883a5d-228e-48f5-b3d2-eb39e07630fa"
---

# OneDrive / SharePoint の 1 ファイルあたり最大アップロードサイズ（250 GB）

## 概要

OneDrive for Business（および SharePoint in Microsoft 365）における 1 ファイルあたりのアップロード上限は **250 GB** です。これは個々のファイルに適用される上限で、OneDrive のフォルダー、SharePoint ドキュメントライブラリ、Microsoft Teams の「ファイル」タブ、Viva Engage の会話にアップロードされる各ファイルが対象です。

この値は公式ドキュメント「SharePoint limits」（Microsoft 365 / Office 365 サービス説明）に明記されており、本ナレッジ作成時点（2026-06-18）の最新値です。制限値は変更される可能性があるため、利用時には必ず参考リンクの公式ドキュメントで最新値を確認してください。

## 詳細

公式ドキュメント「Service limits for all plans > File size and file path length」より、ファイルサイズ・パス長に関する主な上限は次のとおりです。

| 項目 | 上限 | 適用範囲 |
| --- | --- | --- |
| ファイルアップロード上限 | **250 GB** | OneDrive フォルダー、SharePoint ドキュメントライブラリ、Teams「ファイル」タブ、Viva Engage |
| リスト項目への添付ファイル | 250 MB | Microsoft Lists / SharePoint リスト |
| ZIP 一括ダウンロード | 20 GB | 複数ファイルダウンロード時に自動生成される ZIP |
| ファイルパス長 | 400 文字 | フォルダーパス + ファイル名（デコード後）の合計 |

補足:
- ストレージ容量はバイナリ GB（1 GB = 2^30 バイト）で計算されます。
- 1 ファイルが 250 GB 以下でも、ユーザー／テナントのストレージ割り当て（OneDrive は標準 1 TB/ユーザーなど）を超えるとアップロードできません。ファイルサイズ上限とストレージ容量上限は別の制限です。
- かつて言及されていた「2 GB」や「100 GB」といった旧上限は古い情報です。現在の上限は 250 GB です。

## 手順

大容量ファイルをアップロードする際の確認ポイント:

1. アップロードするファイルが 1 ファイルあたり 250 GB 以下であることを確認する。
2. ユーザーの OneDrive ストレージ残量が十分か（割り当て上限を超えないか）を確認する。
3. ファイルパス（フォルダーパス + ファイル名）が 400 文字以内であることを確認する。
4. 大容量ファイルは Web ブラウザーよりも OneDrive 同期アプリ（OneDrive.exe）経由のアップロードが安定する場合がある。
5. ネットワークの安定性・帯域に注意する（数十 GB 規模はアップロードに時間を要する）。

## よくある原因

250 GB 未満なのにアップロードできない場合に考えられる原因:

- ユーザー／テナントのストレージ割り当てを使い切っている。
- ファイルパスが 400 文字を超えている、または使用できない文字／ファイル名が含まれている。
- ネットワークの中断・タイムアウト。

## 参考リンク

- [SharePoint limits - Service Descriptions | Microsoft Learn](https://learn.microsoft.com/en-us/office365/servicedescriptions/sharepoint-online-service-description/sharepoint-online-limits)
- [Restrictions and limitations in OneDrive and SharePoint - Microsoft Support](https://support.microsoft.com/office/restrictions-and-limitations-in-onedrive-and-sharepoint-64883a5d-228e-48f5-b3d2-eb39e07630fa)
