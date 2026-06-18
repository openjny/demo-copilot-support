---
issue_number: 15
title: "[問い合わせ] 新規入社者の MFA（Authenticator）登録が完了できない"
status: "resolved"
created: "2026-06-18"
resolved: "2026-06-18"
category: "security"
related_knowledge:
  - "knowledge/security/2026-06-18-entra-mfa-authenticator-registration-timeout.md"
---

# [問い合わせ] 新規入社者の MFA（Authenticator）登録が完了できない

## 問い合わせ内容

新規入社者の一部で、`aka.ms/mfasetup` から Microsoft Authenticator の登録を進めると、QR コード読み取り後の承認が完了せずタイムアウトする。社内ネットワーク/携帯回線の双方で再現し、利用者操作とテナント設定のどちらを優先して疑うべきか不明。

## 調査プロセス

1. リポジトリのテンプレート・命名規則を確認し、security カテゴリで成果物形式を確定。
2. Microsoft Learn の以下を確認し、統合登録、認証方法ポリシー、Authenticator 特性、ユーザー再登録、TAP の公式手順を収集。
3. 事象が「一部ユーザーのみ」である点を重視し、設定差分と端末差分の両面で切り分け手順を整理。

## 調査結果

確認できた事実:

- MFA/SSPR の登録は統合登録（Security info）で実施される。
- 認証方式の有効/無効は Authentication methods policy が推奨で、移行状態によってはレガシー設定も影響する。
- ユーザー単位で MFA 再登録を強制可能。
- 新規ユーザーの初期登録詰まりには Temporary Access Pass を代替導線として利用できる。

未確認事項（テナント依存）:

- 失敗ユーザーが Authenticator の対象外グループに属しているか。
- 失敗端末で通知許可/時刻同期/アプリ更新状態に差分があるか。

## 回答内容

Issue には断定を避け、以下を案内する方針:

- 原因候補（可能性順）
  1. 認証方法ポリシーの適用差分
  2. 既存登録情報の不整合（再登録未実施）
  3. 端末側通知/時刻同期問題
- 切り分け手順
  - 成功ユーザーとのポリシー差分比較
  - Authentication methods policy と移行状態確認
  - Require re-register MFA 実施後の再登録
  - 端末通知設定/OS 時刻/アプリ更新確認
  - 必要に応じ TAP で登録完了まで誘導
- 公式ドキュメント URL を根拠として併記

## 備考

今回の情報だけでは単一原因を特定できないため、運用上は「比較対象を置いた差分確認」と「再登録 + TAP の回復導線」を先に整備すると、オンボーディング遅延を抑えやすい。
