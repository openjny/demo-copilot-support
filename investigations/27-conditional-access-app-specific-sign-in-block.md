---
issue_number: 27
title: "条件付きアクセスで特定のアプリだけサインインがブロックされる"
status: "open"
created: "2026-06-18"
resolved: ""
category: "security"
related_knowledge:
  - "knowledge/security/2026-06-18-conditional-access-app-specific-sign-in-block.md"
---

# 条件付きアクセスで特定のアプリだけサインインがブロックされる

## 問い合わせ内容

- 対象サービス: Microsoft Entra ID 条件付きアクセス
- 発生契機: ポリシー追加後
- 影響範囲: 一部ユーザー
- 事象: 特定のクラウドアプリへのサインインだけがブロックされるようになった。
- 期待結果: 正規ユーザーが対象アプリに正常にサインインでき、原因の特定方法が分かること。

## 調査プロセス

1. Issue #27 の本文を確認し、「ポリシー追加後」「一部ユーザーのみ」「特定アプリのみ」という
   条件から、原因を即断せず切り分け手順中心で回答すべき事案と判断した。
2. `knowledge/` を確認し、Microsoft Entra ID 条件付きアクセスのアプリ単位ブロックに
   関する既存ナレッジが未整備であることを確認した。
3. Microsoft Learn の公式ドキュメントを参照し、以下を確認した。
   - サインイン ログの Conditional Access タブで、適用/未適用ポリシーと理由を確認できること
   - What If / Report-only / Policy impact で事前・事後評価できること
   - Resource / Audience / service dependency の確認が必要なこと
   - Audit logs で直近のポリシー変更差分を追跡できること
4. 調査結果を `knowledge/security/2026-06-18-conditional-access-app-specific-sign-in-block.md`
   にナレッジ化した。

## 調査結果

- **確認できた事実**
  - 条件付きアクセスの想定外の結果は、まず Microsoft Entra の **Sign-in logs** で確認するのが公式手順。
  - サインイン イベントの **Conditional Access** タブで、どのポリシーが適用されたか /
    適用されなかったか、その理由を確認できる。
  - 特定アプリだけの障害に見えても、実際には **service dependency** や **Audience** に含まれる
    別リソースへのポリシーが原因の可能性がある。
  - What If は有用だが、**service dependencies は評価しない**ため、実サインイン ログとの併用が必要。
  - Audit logs では Conditional Access ポリシーの更新履歴を確認できる。

- **未確認（この Issue 情報だけでは断定できない点）**
  - 実際にブロックされたサインイン イベントの Request ID / Correlation ID / エラーコード
  - Conditional Access タブに表示される適用ポリシー名と未適用理由
  - 対象ユーザーの所属グループ、除外設定、端末準拠状態、接続元条件
  - 対象アプリの背後で呼び出されている依存サービス

  → このため、「設定ミス」とは現時点で断定せず、原因候補と切り分け手順を案内するのが適切と判断した。

## 回答内容

Issue #27 への回答案として、以下を提示できる状態に整理した。

- まずサインイン ログで **Conditional Access = Failure** に絞り、該当イベントの
  **Conditional Access** タブで適用/未適用ポリシーと理由を確認すること
- **Resource / Audience** を確認し、見えているアプリ以外の依存リソースが原因でないかを調べること
- **Users / Groups / Exclusions / Target resources / Conditions / Grant controls** を
  設計意図と突き合わせること
- **What If** で再現条件をシミュレートし、修正版は **Report-only** と **Policy impact** で検証すること
- 直近変更が疑わしい場合は **Audit logs** の
  `Service = Conditional Access` / `Activity = Update Conditional Access policy` で差分を見ること
- 必要に応じて、Request ID / Correlation ID / エラーコード / 影響ユーザーの共通点を
  追加で取得するとさらに絞り込めること

## 備考

- 条件付きアクセスは割り当て・条件・依存サービスが組み合わさるため、
  **「特定アプリだけブロックされた」＝対象アプリ設定ミス**とは限らない。
- Microsoft Learn MCP はこのセッションでは利用していないが、参照根拠はすべて
  Microsoft Learn の公開ドキュメント URL に統一した。
