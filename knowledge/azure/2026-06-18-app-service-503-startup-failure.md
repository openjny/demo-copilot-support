---
title: "Azure App Service で .NET アプリのデプロイ後に 503 / FileNotFoundException が発生する"
category: "azure"
tags: ["app-service", "dotnet", "deployment", "503", "assembly-loading", "kudu"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [1]
sources:
  - "https://learn.microsoft.com/en-us/azure/app-service/troubleshoot-http-502-http-503"
  - "https://learn.microsoft.com/en-us/azure/app-service/deploy-zip"
  - "https://learn.microsoft.com/en-us/azure/app-service/deploy-run-package"
---

# Azure App Service で .NET アプリのデプロイ後に 503 / FileNotFoundException が発生する

## 概要

.NET アプリを Azure App Service にデプロイした後、デプロイ自体は成功しているのに
HTTP 503 Service Unavailable が返るケースがある。App Service のステータスが「Running」でも、
アプリのワーカープロセスが起動時に例外で落ちていると 503 になる。

典型的なシグナルは、Kudu のログストリームに記録されるスタートアップ例外で、
特に `System.IO.FileNotFoundException: Could not load file or assembly '...'` のように
アセンブリのロードに失敗しているケース。これはアプリが起動できていないことを示しており、
プラットフォーム側の障害ではなくアプリ／デプロイ成果物側の問題である可能性が高い。

## 詳細

### 503 とアプリ起動失敗の関係

App Service の 503 は「プラットフォームが Running でもアプリのプロセスが応答できていない」
状態を含む。`FileNotFoundException` 等の起動時例外でワーカーが繰り返しクラッシュすると、
リクエストを処理できず 503 が返る。原因の切り分けと対処は公式の
[HTTP 502 / 503 のトラブルシューティング](https://learn.microsoft.com/en-us/azure/app-service/troubleshoot-http-502-http-503)
に沿って行う。

### `Could not load file or assembly` が起きる主な理由

`FileNotFoundException: Could not load file or assembly 'X, Version=8.0.0.0'` は、
実行時に必要なアセンブリ（または依存アセンブリ）が見つからない／バージョンが一致しない
ことを示す。.NET アプリでよくある具体的原因は次のとおり。

1. **wwwroot に前回デプロイの古いアセンブリが残留している（最有力）**
   - 既定の ZIP デプロイ等では成果物が実行ディレクトリ `D:\home\site\wwwroot`
     （Linux は `/home/site/wwwroot`）に展開される。このとき
     **「ZIP 内のファイルはタイムスタンプが既存と異なる場合のみコピーされる」** 動作のため、
     ファイル名が変わった依存 DLL や削除されたはずの旧バージョン DLL が残り、
     新旧アセンブリが混在してバージョン不一致を起こすことがある
     （出典: [Deploy files to App Service](https://learn.microsoft.com/en-us/azure/app-service/deploy-zip)）。
   - 「前回のデプロイでは問題なかった」「今回もデプロイ自体は成功」という状況と整合する。
2. **NuGet パッケージのバージョン不整合 / 依存関係の取りこぼし**
   - ローカルでは復元済みアセンブリで動くが、発行成果物に必要な依存が含まれていない、
     あるいは参照バージョンと実体がずれている。
3. **発行形式とランタイムスタックの不一致**
   - フレームワーク依存（framework-dependent）発行なのに App Service 側の .NET ランタイム
     スタックがアプリのターゲット（例: .NET 8）と異なる、もしくは
     自己完結（self-contained）発行で必要なファイルが欠落している。

### 恒久対策: Run From Package によるアトミックなデプロイ

既定のデプロイは実行ディレクトリへ展開するため、ファイルロック競合や
「一部だけ更新された」中途半端な状態、旧ファイル残留が起こり得る。
`WEBSITE_RUN_FROM_PACKAGE=1` を設定すると、ZIP を読み取り専用の wwwroot として
**そのままマウント**するため、次の利点がある（出典:
[Run your app from a ZIP package](https://learn.microsoft.com/en-us/azure/app-service/deploy-run-package)）。

- デプロイと実行間のファイルロック競合を排除
- 常に「完全にデプロイされたアプリ」だけが動作する（中途半端な状態をなくす）
- 旧ファイル残留による新旧アセンブリ混在を構造的に防止

> 注意: Run From Package は wwwroot を読み取り専用にするため、App Service 上の
> Java ビルトインランタイム（Java SE / Tomcat / JBoss EAP）や Python アプリでは
> 制限・非対応がある。.NET アプリでは利用可能。

## 手順

### 1. 起動例外を特定する

1. Kudu（`https://<app-name>.scm.azurewebsites.net`）の **ログストリーム** /
   `D:\home\LogFiles` でスタートアップ例外を確認する。
2. `FileNotFoundException` のアセンブリ名とバージョンを控える。

### 2. 古いアセンブリ残留を解消する（クリーンデプロイ）

1. Kudu の **DebugConsole** で `D:\home\site\wwwroot`（Linux は `/home/site/wwwroot`）の
   内容を確認し、想定外の古い DLL が残っていないか調べる。
2. クリーンな再デプロイを行う。ローカルでは発行先フォルダーを毎回空にしてから
   `dotnet publish` し、その出力（ディレクトリ自体は含めない）を ZIP 化してデプロイする。

   ```powershell
   # 発行先をクリーンにしてから発行
   Remove-Item -Recurse -Force .\publish -ErrorAction SilentlyContinue
   dotnet publish -c Release -o .\publish
   # publish の中身（ルートディレクトリは含めない）を ZIP 化
   Compress-Archive -Path .\publish\* -DestinationPath .\site.zip -Force
   ```

3. それでも残留が疑われる場合は、Kudu で `D:\home\site\wwwroot` を空にしてから再デプロイする。

### 3. 恒久対策として Run From Package を有効化する

1. アプリ設定に `WEBSITE_RUN_FROM_PACKAGE=1` を追加する（影響範囲: wwwroot が
   読み取り専用になる。アプリが実行中に同ディレクトリへ書き込む処理がある場合は要検証）。
2. ZIP パッケージをデプロイし、アプリを再起動する。

### 4. 依存関係とランタイムを確認する

1. ローカルで `dotnet publish` した出力に、例外に出ているアセンブリ
   （例: `Microsoft.Extensions.Configuration.Abstractions.dll`）が含まれているか確認する。
2. `.csproj` の `TargetFramework` と NuGet パッケージのバージョン整合を確認する。
3. App Service の **構成 > 全般設定** のランタイムスタックがアプリのターゲット
   （例: .NET 8）と一致しているか確認する。

## よくある原因

- 差分 ZIP デプロイで wwwroot に旧バージョンの依存 DLL が残り、新旧アセンブリが混在
- 発行成果物に必要な依存アセンブリが含まれていない（NuGet バージョン不整合・取りこぼし）
- 発行形式（framework-dependent / self-contained）とランタイムスタックの不一致
- ファイルロック競合により一部ファイルだけが更新された中途半端なデプロイ状態

## 参考リンク

- [Fix HTTP 502 and HTTP 503 Errors - Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/troubleshoot-http-502-http-503)
- [Deploy files to Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/deploy-zip)
- [Run your app from a ZIP package - Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/deploy-run-package)
