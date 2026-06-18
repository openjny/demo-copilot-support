---
title: "Microsoft Entra ID 条件付きアクセスで特定アプリだけサインインがブロックされる場合の切り分け"
category: "security"
tags: ["entra-id", "conditional-access", "sign-in-logs", "what-if", "report-only", "audience-reporting"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [27]
sources:
  - "https://learn.microsoft.com/en-us/entra/identity/conditional-access/troubleshoot-conditional-access"
  - "https://learn.microsoft.com/en-us/entra/identity/conditional-access/what-if-tool"
  - "https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-report-only"
  - "https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-cloud-apps"
  - "https://learn.microsoft.com/en-us/entra/identity/conditional-access/service-dependencies"
  - "https://learn.microsoft.com/en-us/entra/identity/monitoring-health/how-to-view-applied-conditional-access-policies"
---

# Microsoft Entra ID 条件付きアクセスで特定アプリだけサインインがブロックされる場合の切り分け

## 概要

条件付きアクセス (Conditional Access) を追加した直後に、**特定のクラウドアプリだけ**
サインインがブロックされる場合でも、その場で「設定ミス」とは断定できない。公式ドキュメントでは、
**サインイン ログの Conditional Access タブで、どのポリシーが適用されたか / されなかったか、
その理由を確認する**ことが最優先の切り分けとして案内されている。

また、ユーザーには「そのアプリだけの問題」に見えていても、実際には**依存先リソース
（service dependency）や Audience** に対する条件付きアクセス評価が原因のことがある。
そのため、アプリ名だけでなく **Resource / Audience / 適用ポリシー / 未適用理由** を
セットで確認する必要がある。

## 詳細

### まず確認できる事実

- 条件付きアクセスの想定外の結果を調べる公式の起点は、**Microsoft Entra のサインイン ログ**。
- サインイン イベントの **Conditional Access** タブでは、適用されたポリシーと
  適用されなかったポリシー、その理由を確認できる。
- **What If** ツールでは、指定したユーザー・対象リソース・端末条件・クライアントアプリに対して、
  どのポリシーが適用される見込みかをシミュレートできる。
- **Report-only** モードでは、ポリシーを強制せず評価結果だけをサインイン ログへ残せる。
- **Policy impact (preview)** では、過去 24 時間 / 7 日 / 1 か月のポリシー影響の傾向を確認できる。
- 条件付きアクセスは、見えているクライアントアプリだけでなく、その背後で呼び出される
  **依存サービス**にも影響を受けることがある。たとえば Teams の利用時に SharePoint や Exchange
  へのポリシーが影響することがある。

### 特定アプリだけブロックされる場合の主な原因候補

本事象だけでは原因を一意に断定できないが、公式ドキュメント上は以下の候補を優先的に切り分ける。

1. **Users / Groups / Exclusions の割り当て差分**
   - 一部ユーザーだけ発生するなら、対象ユーザーが別グループに所属している、
     例外グループに入っていない、という差分が考えられる。
2. **Target resources（旧 cloud apps）の設定差分**
   - 対象アプリの選択が想定より広い / 狭い、あるいは個別アプリではなく
     Microsoft 365 全体をまとめて扱うべきケースがある。
3. **Service dependency / Audience による想定外の評価**
   - 見えているアプリではなく、背後の依存先リソースが条件付きアクセスの評価対象になっている可能性。
4. **Conditions / Grant controls の不一致**
   - 準拠デバイス、Hybrid join、場所、クライアントアプリ、サインイン リスク、
     アプリ保護ポリシーなどが一部ユーザーでだけ満たされていない可能性。
5. **ポリシー変更直後の設定ミス**
   - 変更前は発生していなかったなら、Audit logs で直近のポリシー変更差分を確認する価値が高い。

### よく参照するエラー コード

条件付きアクセス関連では、サインイン ログやブラウザー エラーに
`AADSTS53000` ～ `AADSTS53009` 系が出ることがある。特に以下が切り分けの手掛かりになる。

- `AADSTS53000` / `DeviceNotCompliant`
- `AADSTS53001` / `DeviceNotDomainJoined`
- `AADSTS53002` / `ApplicationUsedIsNotAnApprovedApp`
- `AADSTS53003` / `BlockedByConditionalAccess`
- `AADSTS53009` / `Application needs to enforce Intune protection policies`

## 手順

### 1. 失敗したサインイン イベントを特定する

1. 対象ユーザー、発生時刻、対象アプリ名を整理する。
2. 可能ならエラー画面の **Request ID / Correlation ID / Timestamp** を控える。
3. Microsoft Entra 管理センターで **Entra ID > Monitoring & health > Sign-in logs** を開く。
4. 以下のフィルターで対象イベントを絞る。
   - Username
   - Date
   - Resource
   - Conditional Access = Failure
   - Correlation ID（分かる場合）

### 2. Conditional Access タブで「どのポリシーが、なぜ効いたか」を確認する

1. 該当イベントを開き、**Conditional Access** タブを確認する。
2. 以下を記録する。
   - 適用されたポリシー名
   - 適用されなかったポリシー名と理由
   - Block access か、未達成の Grant / Session control か
3. **Policy Name** を開き、実際の設定を確認する。
4. **Basic info / Location / Device info / Authentication details / Additional details** も合わせて確認し、
   どの条件が一致したかを突き合わせる。

### 3. Resource / Audience / 依存サービスを確認する

1. サインイン イベントの **Resource** と **Audience** を確認する。
2. ユーザーが「特定アプリだけ」と認識していても、背後の依存サービスに対するポリシーが
   適用されていないか確認する。
3. Microsoft 365 系アプリでは、個別アプリではなく **Office 365** グループで
   一貫したポリシーにした方がよいケースがある。

### 4. ポリシー割り当てを見直す

以下の確認結果と設計意図を照らし合わせる。

- **Users / Groups**: 影響ユーザーが本当に対象か
- **Exclusions**: 除外すべき例外ユーザー / グループが漏れていないか
- **Target resources**: 想定したアプリ / サービスだけが対象か
- **Conditions**: Device platform / Locations / Client apps / Risk などが過剰でないか
- **Grant controls**: MFA、準拠デバイス、Hybrid join、アプリ保護要件が現実の端末条件に合っているか

### 5. What If と Report-only で再確認する

1. **Conditional Access > Policies > What If** を開く。
2. 実際の事象に合わせて、以下を入力して評価する。
   - User
   - Target resource
   - Device platform
   - Client app
   - 必要に応じて Location / Risk など
3. 「適用されるポリシー」「適用されないポリシーと理由」が設計意図と一致するか確認する。
4. ただし **What If は service dependencies を評価しない**ため、実サインイン ログの結果も必ず併用する。
5. 修正版ポリシー案は **Report-only** で検証し、サインイン ログの **Report-only** タブや
   **Policy impact** で影響を確認してから有効化する。

### 6. 直近変更の影響を Audit logs で確認する

1. **Entra ID > Monitoring & health > Audit logs** を開く。
2. **Service = Conditional Access**
3. **Category = Policy**
4. **Activity = Update Conditional Access policy**
5. 発生日時の前後で、どのポリシーにどの変更が入ったかを確認する。

## よくある原因

- 対象ユーザーが想定外のグループに含まれていた、または除外設定が不足していた
- Target resources の対象アプリ選択が広すぎる / 狭すぎる
- Teams などのアプリで、依存先の SharePoint / Exchange へのポリシーが効いていた
- 準拠デバイスや Hybrid join などの要件を一部ユーザーの端末が満たしていなかった
- ポリシー変更時に Block access や厳しい Grant control を誤って追加していた

## 参考リンク

- [Troubleshoot sign-in problems with Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/troubleshoot-conditional-access)
- [The Conditional Access What If tool](https://learn.microsoft.com/en-us/entra/identity/conditional-access/what-if-tool)
- [Evaluate Conditional Access policies in report-only mode](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-report-only)
- [Target resources in Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-cloud-apps)
- [Service dependencies in Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/service-dependencies)
- [View applied Conditional Access policies in activity logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/how-to-view-applied-conditional-access-policies)
