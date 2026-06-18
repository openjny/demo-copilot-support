---
issue_number: 1
title: "[問い合わせ] Azure App Service でデプロイ後に 503 エラーが発生する"
status: "resolved"
created: "2026-06-18"
resolved: "2026-06-18"
category: "azure"
related_knowledge:
  - "knowledge/azure/2026-06-18-app-service-503-startup-failure.md"
---

# [問い合わせ] Azure App Service でデプロイ後に 503 エラーが発生する

## 問い合わせ内容

.NET 8 の Web API を Azure App Service（B1 / Japan East）にデプロイしたところ、
デプロイは成功するが、アクセスすると 503 Service Unavailable が返る。

- デプロイログにエラーなし、App Service のステータスは「Running」
- ローカルでは正常動作、前回（1 週間前）のデプロイは問題なし
- Kudu のログストリームに以下のスタートアップ例外:

```text
Application startup exception: System.IO.FileNotFoundException:
Could not load file or assembly 'Microsoft.Extensions.Configuration.Abstractions, Version=8.0.0.0'
```

## 調査プロセス

1. 既存ナレッジ（`knowledge/azure`）を確認 → 該当する再利用可能ナレッジは存在せず。
2. 公式ドキュメントで原因と対処を調査。
   - HTTP 502/503 のトラブルシューティング（起動失敗と 503 の関係）
   - App Service への ZIP デプロイ時のファイル更新挙動（旧ファイル残留）
   - Run From Package によるアトミックデプロイの利点
3. 症状（デプロイ成功・Running 表示・ローカル正常・前回正常・起動時のアセンブリロード失敗）
   から原因仮説を絞り込み。

## 調査結果

- 503 は必ずしもプラットフォーム障害ではなく、アプリのワーカーが起動時例外で
  繰り返しクラッシュしている場合にも返る。今回は Kudu ログの
  `FileNotFoundException: Could not load file or assembly` が決定的シグナルで、
  アプリが起動できていない（＝アプリ／デプロイ成果物側の問題）。
- 既定の ZIP デプロイは成果物を実行ディレクトリ `wwwroot` に展開し、
  **ファイルはタイムスタンプが既存と異なる場合のみコピーされる**。このため
  旧バージョンの依存 DLL が残留し、新旧アセンブリ混在によるバージョン不一致を
  起こし得る。「前回は問題なかった」「今回もデプロイは成功」という状況と整合する
  最有力の原因と判断。
- 副次的な原因候補として、NuGet パッケージのバージョン不整合／依存の取りこぼし、
  発行形式（framework-dependent / self-contained）とランタイムスタックの不一致がある。
- 恒久対策として `WEBSITE_RUN_FROM_PACKAGE=1` によるアトミックデプロイが有効。
  ZIP を読み取り専用 wwwroot としてマウントするため、旧ファイル残留やファイルロック
  競合を構造的に防げる。

詳細は作成したナレッジを参照:
`knowledge/azure/2026-06-18-app-service-503-startup-failure.md`

## 回答内容

Issue #1 に以下を回答:

- 原因の説明（起動時アセンブリロード失敗による 503、最有力は旧アセンブリ残留）
- 対処手順（Kudu でログ／wwwroot 確認 → クリーン再デプロイ → Run From Package 有効化 →
  依存・ランタイム整合確認）
- 参考リンク（公式ドキュメント 3 件）とナレッジへの参照

## 備考

- 各対処は公式ドキュメントに基づく。最終的な確定にはユーザー環境での
  Kudu ログ／wwwroot の実物確認が必要。
- `WEBSITE_RUN_FROM_PACKAGE=1` は wwwroot を読み取り専用にするため、
  アプリが実行時に同ディレクトリへ書き込む処理がある場合は事前検証が必要（影響範囲）。
