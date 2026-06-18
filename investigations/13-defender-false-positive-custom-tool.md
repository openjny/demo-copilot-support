---
issue_number: 13
title: "[問い合わせ] Microsoft Defender が自作の正規ツールを誤検知してブロックする"
status: "resolved"
created: "2026-06-18"
resolved: "2026-06-18"
category: "security"
related_knowledge:
  - "knowledge/security/2026-06-18-defender-false-positive-custom-tool.md"
---

# [問い合わせ] Microsoft Defender が自作の正規ツールを誤検知してブロックする

## 問い合わせ内容

Microsoft Defender for Endpoint 管理下で、社内内製の署名なし EXE が
定義更新後に `Trojan:Win32/Wacatac.B!ml` で複数端末同時に検知・ブロックされる。
業務影響が出ているため、安全性を落とし過ぎない許可方法を確認したい。

## 調査プロセス

1. 既存ナレッジ（`knowledge/security`）を確認し、該当記事が未整備であることを確認。
2. Microsoft Learn の Defender for Endpoint 公式ドキュメントを調査し、誤検知時の標準フロー（検知ソース確認、提出、allow indicator、除外）を確認。
3. Intune 管理端末向けの除外設定手順・除外時のリスク・避けるべき除外（`.exe` 拡張子除外や広範囲フォルダー除外）を確認。
4. 問い合わせ条件（署名なし EXE、複数端末、定義更新直後）を当てはめ、断定を避けつつ原因候補と切り分け・暫定対処を整理。

## 調査結果

- 公式には、除外や許可は「根本原因を把握した後」の最後の手段として扱われる。
- 誤検知が疑われる場合、Defender ポータルから file/hash を **Clean** として提出する手順がある。
- 内製ツールが署名なしの場合、証明書ベース IOC 許可は使えず、実務上は file hash allow が第一候補となる。
- ただし file hash allow はバイナリ更新時に再登録が必要。運用できない場合に限り、Intune で範囲限定の除外を検討する。
- `.exe` 拡張子除外や `C:\` などの広範囲除外は、攻撃面拡大の観点で非推奨。

## 回答内容

Issue #13 には以下方針で回答:

- **状況整理**: 確認済み事実（定義更新後・複数端末同時）と未確認事項（真の検知ソース、同一ハッシュ性）を分離
- **原因候補**: 誤検知の可能性は高いが断定しない
- **切り分け**:
  1. Detection source と SHA-256 を確認
  2. Microsoft へ false positive 提出
  3. 暫定は file hash allow（期限付き・限定スコープ）
  4. 必要時のみ Intune 限定除外
- **注意点**: 除外は保護低下を伴うため最小化・期限設定・後日ロールバック
- **参考リンク**: Defender 公式 7 件を提示

## 備考

- `Wacatac.B!ml` は機械学習系名称であり、名称だけで誤検知を断定できないため、回答では断定表現を避けた。
- セキュリティ影響（除外による保護低下、ASR/IOC への影響可能性）を明示し、暫定措置の恒久化を避ける運用を推奨。
