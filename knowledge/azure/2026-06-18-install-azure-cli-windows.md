---
title: "Windows PC への Azure CLI のインストール方法"
category: "azure"
tags: ["azure-cli", "windows", "install", "winget", "msi", "az-login"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [19]
sources:
  - "https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows"
  - "https://learn.microsoft.com/en-us/cli/azure/install-azure-cli"
  - "https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli"
---

# Windows PC への Azure CLI のインストール方法

## 概要

Azure CLI は、Azure リソースを管理するためのクロスプラットフォームのコマンドラインツールである。
Windows では MSI または ZIP パッケージとして提供され、PowerShell またはコマンドプロンプト
（`cmd.exe`）から利用できる。インストール後に `az login` を実行することで Azure へサインインし、
各種管理コマンドを実行できるようになる。

公式ドキュメント（[Install the Azure CLI on Windows](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows)）
に記載された標準的なインストール方法は次の 3 つである。

- **WinGet（Windows Package Manager）** — Windows 11 / 新しめの Windows 10 で既定で利用可能。更新管理も容易。
- **MSI インストーラー** — ダウンロードしてダブルクリックでインストールする最も基本的な方法。
- **PowerShell（MSI をサイレントインストール）** — 自動化・スクリプト向け。

> 重要: インストール完了後は、**開いているターミナル（PowerShell / コマンドプロンプト）を一度閉じて
> 開き直す**必要がある。これを行わないと `az` コマンドが PATH 上で認識されないことがある。

## 詳細

### 方法 1: WinGet（推奨）

WinGet は Microsoft 製の Windows パッケージマネージャーで、Windows 11 および新しいバージョンの
Windows 10 には既定で含まれている（古い Windows では別途インストールが必要）。
64 ビット OS では既定で 64 ビット版の Azure CLI がインストールされる。

```powershell
winget install --exact --id Microsoft.AzureCLI
```

- `--exact`（`-e`）は公式の Azure CLI パッケージを確実に指定するためのオプション。
- 既定で最新バージョンがインストールされる。バージョンを指定する場合は
  `--version <version>`（例: `--version 2.67.0`）を付与する。

### 方法 2: MSI インストーラー

公式ページ（[Install the Azure CLI on Windows](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows)）
から最新の MSI をダウンロードして実行する。UAC（ユーザーアカウント制御）のダイアログが出たら
「はい」を選択する。既に Azure CLI がインストールされている場合は、MSI 実行により上書き更新される
（事前のアンインストールは不要）。最新版 MSI の直接リンクは `https://aka.ms/installazurecliwindows`
（32 ビット）/ `https://aka.ms/installazurecliwindowsx64`（64 ビット）。

### 方法 3: PowerShell でサイレントインストール

自動化したい場合は、PowerShell を**管理者として**起動し、以下を実行する。

```powershell
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -Uri https://aka.ms/installazurecliwindowsx64 -OutFile .\AzureCLI.msi
Start-Process msiexec.exe -Wait -ArgumentList '/I', 'AzureCLI.msi', '/quiet'
Remove-Item .\AzureCLI.msi
```

上記は 64 ビット版をインストールする。32 ビット版を入れたい場合は URL を
`https://aka.ms/installazurecliwindows` に変更する。

### WSL を使う場合

Windows Subsystem for Linux（WSL）上にインストールする場合は、Windows 用ではなく
Linux ディストリビューション向けのパッケージを使用する。詳細は
[メインのインストールページ](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)を参照。

## 手順

1. 上記いずれかの方法で Azure CLI をインストールする（WinGet が最も簡単で更新も容易）。
2. **開いているターミナルをすべて閉じ、PowerShell またはコマンドプロンプトを開き直す。**
3. インストールとバージョンを確認する。

   ```powershell
   az version
   ```

4. Azure にサインインする。既定ではブラウザーが開き、対話的に認証する。

   ```powershell
   az login
   ```

   - 参考: [Sign in with Azure CLI](https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli)
5. （任意）将来の更新は次のコマンドで行える。

   ```powershell
   az upgrade
   ```

   WinGet で導入した場合は `winget upgrade --id Microsoft.AzureCLI` でも更新できる。

## よくある原因

インストール直後に `az` が「コマンドが見つかりません」となる典型的な原因:

- インストール前から開いていたターミナルをそのまま使っている（PATH が更新されていない）。
  → ターミナルを閉じて開き直す。
- 古い Windows で WinGet 自体が未インストール。→ MSI もしくは PowerShell の方法を使う。

## 参考リンク

- [Install the Azure CLI on Windows](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows)
- [How to install the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [Sign in with Azure CLI](https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli)
- [Azure CLI release notes](https://learn.microsoft.com/en-us/cli/azure/release-notes-azure-cli)
