---
issue_number: 24
title: "AKS の Pod が CrashLoopBackOff で起動しない"
status: "open"
created: "2026-06-18"
resolved: ""
category: "azure"
related_knowledge:
  - "knowledge/azure/2026-06-18-aks-pod-crashloopbackoff.md"
---

# AKS の Pod が CrashLoopBackOff で起動しない

## 問い合わせ内容

- 対象サービス: Azure Kubernetes Service (AKS)
- 環境: Japan East / Kubernetes 1.29 / 昨日のデプロイ後から発生
- 内容: 新しいイメージをデプロイしたところ、一部の Pod が `CrashLoopBackOff` になり起動しなくなった。
  `kubectl get pods` では RESTARTS が増え続ける。原因の切り分け方が分からない。
- 期待する結果: Pod が正常に Running になること。原因の特定方法を知りたい。

## 調査プロセス

1. Issue #24 本文を読み、カテゴリ（Azure / AKS）と症状（CrashLoopBackOff・RESTARTS 増加）を把握した。
2. `knowledge/` を検索し、関連する既存ナレッジが無いことを確認した
   （既存は App Service 503 のみ）。
3. Microsoft Learn の公式トラブルシューティング記事を調査し、原因候補と切り分け手順の根拠を確認した。
   - Pod is stuck in CrashLoopBackOff mode
   - Troubleshoot pod workload restarts in AKS
4. 調査結果を `knowledge/azure/2026-06-18-aks-pod-crashloopbackoff.md` に汎用ナレッジとして作成した。

## 調査結果

- `CrashLoopBackOff` は「コンテナが起動直後に異常終了し、Kubernetes が再起動を繰り返している」
  という**症状**であり、単一の原因を示すものではない。本問い合わせの情報だけでは原因を一意に
  断定できない（確認できた事実: 新イメージのデプロイ後に発生／RESTARTS が増加。未確認: Exit Code、
  describe の Events、--previous ログ、プローブ設定、リソース設定）。
- 公式ドキュメントが挙げる原因候補: アプリ起動失敗、リソース制限不足（OOM）、ConfigMap/Secret
  欠落、イメージ問題、Init コンテナ失敗、liveness/readiness プローブ失敗、依存サービス未準備、
  ネットワーク、不正な command/args。
- 新イメージのデプロイ直後という状況からは、可能性として「アプリ変更によるクラッシュ」「イメージの
  タグ/破損」「新バージョンでのリソース要件増加（OOM）」「プローブ設定と起動時間の不整合」が
  相対的に高い（いずれも未確認の推測。describe の Exit Code / Reason と --previous ログで裏付けが必要）。
- 切り分けの起点は `kubectl describe pod`（Last State の Exit Code / Reason、Events）と
  `kubectl logs --previous`。Exit Code 137 / OOMKilled ならメモリ、Events のプローブ失敗なら
  プローブ設定、というように出力と原因を対応づけられる。

## 回答内容

Issue #24 本体にコメントで回答した。原因は断定せず、以下を提示:

- 状況整理（確認できた事実と未確認事項の区別）
- 想定される原因候補（可能性の高い順・各候補の根拠の有無）
- 切り分け手順（`kubectl get pods` → `kubectl describe pod`（Exit Code/Reason/Events）→
  `kubectl logs --previous` → `kubectl get events`）と、出力ごとの判断の対応づけ
- 参考 Microsoft Learn URL

## 備考

- 原因特定には describe の Exit Code / Reason と `--previous` ログが鍵。追加情報（これらの出力）が
  得られれば原因をさらに絞り込める旨を回答に含めた。
- 認証情報・機密値は Issue コメントに記載していない。
