---
title: "AKS の Pod がデプロイ後に CrashLoopBackOff になり再起動を繰り返す"
category: "azure"
tags: ["aks", "kubernetes", "crashloopbackoff", "troubleshooting", "kubectl", "liveness-probe", "deployment"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [24]
sources:
  - "https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/create-upgrade-delete/pod-stuck-crashloopbackoff-mode"
  - "https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/availability-performance/troubleshoot-pod-workload-restart"
  - "https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-states"
  - "https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/"
---

# AKS の Pod がデプロイ後に CrashLoopBackOff になり再起動を繰り返す

## 概要

Azure Kubernetes Service (AKS) で `kubectl get pods` の STATUS が `CrashLoopBackOff` になり、
RESTARTS のカウントが増え続ける状態は、**コンテナが起動直後に異常終了（exit code が 0 以外）し、
Kubernetes がバックオフ間隔を空けながら再起動を繰り返している**ことを示す。

`CrashLoopBackOff` は「症状」であって特定の単一原因を表すものではない。新しいイメージのデプロイ後に
発生したという事実は「変更点があった」ことを示すが、それだけでは原因（アプリのクラッシュ／設定不足／
プローブ誤設定／リソース不足など）を一意に特定できない。したがって、まず `kubectl describe pod` の
`Last State` の **Exit Code / Reason** と `kubectl logs --previous` の出力を確認し、
原因を切り分けることが対応の起点になる。

## 詳細

### CrashLoopBackOff が示す状態

公式ドキュメント
[Pod is stuck in CrashLoopBackOff mode](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/create-upgrade-delete/pod-stuck-crashloopbackoff-mode)
によれば、`CrashLoopBackOff` の Pod は「起動後まもなく失敗または予期せず終了し、ログに 0 以外の
終了コードが含まれている」可能性が高い。コンテナの状態遷移については Kubernetes の
[Pod Lifecycle - Container states](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-states)
を参照。

> 補足: 終了コードが 0（正常終了）でも、`restartPolicy: Always` の下で短命なプロセス
> （例: 引数なしの busybox）が起動→完了→再起動を繰り返すと CrashLoopBackOff になることがある。
> この場合はクラッシュではなく「常駐しないプロセスを常駐前提で動かしている」設定の問題。

### 想定される原因（公式ドキュメントに基づく候補）

[Pod is stuck in CrashLoopBackOff mode](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/create-upgrade-delete/pod-stuck-crashloopbackoff-mode)
が挙げる主な原因は次のとおり。いずれも「候補」であり、describe / logs の確認で裏付けが必要。

1. **アプリケーションの起動失敗** — コンテナ内アプリが設定ミス・依存欠落・環境変数の誤りで
   起動直後にクラッシュする。
2. **リソース制限の不適切な設定** — CPU / メモリの requests・limits が低すぎ、コンテナが
   OOM などで kill される（メモリ起因の場合 Exit Code 137 が出やすい）。
3. **ConfigMap / Secret の欠落・誤設定** — 参照する設定値やファイルが見つからずアプリが落ちる。
4. **イメージの問題** — イメージの破損やタグ誤り。新イメージのデプロイ直後という状況と整合しやすい。
5. **Init コンテナの失敗** — Init コンテナが失敗すると Pod が再起動する。
6. **Liveness / Readiness プローブの失敗** — プローブが誤設定だと正常なコンテナでも
   unhealthy と判定され再起動される。
7. **依存サービスの未準備** — DB・キュー・他 API などの依存先がまだ利用できない。
8. **ネットワークの問題** — 必要なサービスへ通信できずアプリが失敗する。
9. **不正なコマンド/引数** — `ENTRYPOINT`・command・args の誤りでクラッシュする。

### RESTARTS が増え続ける場合（プローブ起因の切り分け）

[Troubleshoot pod workload restarts in AKS](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/availability-performance/troubleshoot-pod-workload-restart)
は、再起動が繰り返される代表的な原因としてコンテナのヘルスプローブ設定を挙げている。
`kubectl describe pod` の Events に `Liveness probe failed` / `Readiness probe failed` が出る場合は
こちらの線が濃い。代表的な原因:

- liveness / startup プローブのタイムアウト・失敗しきい値・初期遅延が、アプリの正常な
  起動時間や応答時間を考慮できていない。
- 起動の遅いアプリに startup プローブが無く、起動完了前に liveness プローブが走ってしまう。
- プローブのエンドポイントが外部リソース（DB・ストレージ・下流サービス）に依存しており、
  アプリ自体は健全でも一時的に失敗する。
- CPU スロットリング／リソース競合でプローブ要求がタイムアウトする。
- ヘルスチェックのロジックが厳しすぎ、liveness プローブで依存検証まで行っている。

## 手順

### 1. 状態と再起動回数を確認する

```bash
kubectl get pods -n <namespace> -o wide
```

`STATUS` が `CrashLoopBackOff`、`RESTARTS` が増加していることを確認する。

### 2. describe で Exit Code / Reason / Events を確認する（最重要）

```bash
kubectl describe pod <pod-name> -n <namespace>
```

確認ポイントと判断の対応づけ:

| 確認項目 | 出力例 | 示唆される原因 |
| --- | --- | --- |
| `Last State: Terminated` の `Exit Code` | `1` 等の一般的エラー | アプリ起動失敗・設定不正 |
| 同上 `Exit Code` | `137`（OOMKilled） | メモリ limit 不足／リーク |
| 同上 `Reason` | `OOMKilled` | メモリリソース不足 |
| Events: `Liveness probe failed` | プローブ失敗ログ | プローブ誤設定・起動遅延 |
| Events: `Failed to pull image` / `ErrImagePull` | イメージ取得失敗 | タグ誤り・レジストリ認証 |
| `Init Containers` の State | Init が Error | Init コンテナ失敗 |

### 3. 直前にクラッシュしたコンテナのログを確認する

再起動後の現行コンテナではなく、クラッシュした「前回」のコンテナのログを見る:

```bash
# 直前にクラッシュしたインスタンスのログ（CrashLoopBackOff では特に重要）
kubectl logs <pod-name> -n <namespace> --previous

# 現行コンテナのログ
kubectl logs <pod-name> -n <namespace>

# 複数コンテナの場合はコンテナを指定
kubectl logs <pod-name> -c <container-name> -n <namespace> --previous
```

スタックトレースや「config not found」「connection refused」などのメッセージから原因を絞り込む。

### 4. Pod 定義と Events 全体を確認する

```bash
# termination message を含む Pod の完全な定義
kubectl get pod <pod-name> -n <namespace> --output=yaml

# namespace 全体のイベント（時系列）
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

### 5. 原因に応じた対処

- **アプリ／設定起因**（Exit Code 1、ログに例外）: 新イメージのアプリ変更・環境変数・
  ConfigMap/Secret を見直す。直前の正常イメージへロールバックして切り分けるのも有効。
- **イメージ起因**（ErrImagePull / ImagePullBackOff）: タグ・ダイジェスト・レジストリ認証
  （ACR の場合は AKS への attach / imagePullSecret）を確認する。
- **メモリ起因**（Exit Code 137 / OOMKilled）: `resources.limits.memory` を見直す。
- **プローブ起因**（Events にプローブ失敗）: 起動の遅いアプリには startup プローブを追加し、
  `failureThreshold` / `periodSeconds` / `timeoutSeconds` を実際の起動・応答時間に合わせる。
  liveness プローブは軽量に保ち、依存検証は readiness プローブへ移す。

## よくある原因

- 新イメージのアプリ変更・依存欠落・環境変数誤りによる起動直後のクラッシュ（Exit Code 1）
- メモリ limit 不足による OOMKilled（Exit Code 137）
- ConfigMap / Secret の欠落・誤設定
- イメージのタグ誤り・レジストリ認証失敗（ImagePullBackOff 経由）
- Init コンテナの失敗
- liveness / startup プローブの誤設定や起動遅延の考慮不足による不要な再起動

## 参考リンク

- [Pod is stuck in CrashLoopBackOff mode - Azure | Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/create-upgrade-delete/pod-stuck-crashloopbackoff-mode)
- [Troubleshoot pod workload restarts in AKS - Azure | Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/availability-performance/troubleshoot-pod-workload-restart)
- [Pod Lifecycle - Container states | Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-states)
- [Debug Running Pods | Kubernetes](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Configure Liveness, Readiness and Startup Probes | Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
