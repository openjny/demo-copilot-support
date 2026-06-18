---
title: "新規ユーザーの Microsoft Authenticator MFA 登録が QR 読み取り後に完了しないときの切り分け"
category: "security"
tags: ["entra-id", "mfa", "authenticator", "onboarding", "troubleshooting"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [15]
sources:
  - "https://learn.microsoft.com/en-us/entra/identity/authentication/concept-registration-mfa-sspr-combined"
  - "https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods-manage"
  - "https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-authenticator-app"
  - "https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-userdevicesettings"
  - "https://learn.microsoft.com/en-us/entra/identity/authentication/howto-authentication-temporary-access-pass"
  - "https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/mfa/troubleshoot-azure-mfa-issue"
---

# 新規ユーザーの Microsoft Authenticator MFA 登録が QR 読み取り後に完了しないときの切り分け

## 概要

Microsoft Entra ID の MFA 登録は、現在は主に統合登録（Security info）で実施され、利用可能な登録方法は Authentication methods policy と（移行中の場合）レガシー MFA/SSPR 設定の両方の影響を受けます。

QR 読み取り後に承認が完了しない事象は、利用者端末側要因とテナント設定要因の両方で発生しうるため、初動で原因を断定せず、サインインログとポリシー有効範囲の両面で切り分けるのが有効です。

## 詳細

ドキュメントで確認できる事実:

- 認証方法の可否は Authentication methods policy が推奨管理方式で、移行状態によってはレガシー MFA/SSPR 設定も有効です。
- Microsoft Authenticator は MFA プッシュ通知とコード方式の両方に利用されます。
- ユーザー単位で **Require re-register MFA** を実行すると、既存の Authenticator 登録が削除され、再登録を強制できます。
- 新規オンボーディング時の代替導線として Temporary Access Pass (TAP) を使うと、追加の強認証がない状態でも Security info で登録開始できます。

未確認（環境依存）:

- 一部ユーザーだけ失敗する場合、グループベースの認証方法ポリシー差分、端末通知設定差分、古い登録情報残存などの可能性があります。

## 手順

1. **失敗ユーザーと成功ユーザーを 1 名ずつ比較**
   - Entra 管理センターで対象ユーザーの Authentication methods とグループ所属を確認する。
   - 期待結果: 失敗ユーザーのみ Authenticator 利用可否や対象ポリシーが異なる場合、設定起因の可能性が高い。

2. **Authentication methods policy を確認**
   - Entra ID > Authentication methods > Policies で Microsoft Authenticator が対象ユーザー/グループに対して有効か確認する。
   - 併せて移行状態（Pre-migration / Migration in Progress / Migration Complete）を確認し、レガシー設定影響の有無を整理する。
   - 期待結果: 失敗ユーザーが対象外なら、対象グループへ追加して再試行で改善。

3. **ユーザー単位の再登録を実施**
   - ユーザー > Authentication methods > **Require re-register MFA** を実行し、`https://aka.ms/mfasetup` から再登録する。
   - 期待結果: 旧端末登録や不整合が原因なら再登録で解消。

4. **端末側の通知経路を確認**
   - Microsoft Authenticator の通知許可、OS 日時自動設定、アプリ更新状態を確認する。
   - （中国拠点 Android など）プッシュ通知制約がある環境では、通知以外の方法（TOTP コード等）を一時利用する。
   - 期待結果: 端末要因なら、通知設定修正またはコード方式で登録完了。

5. **オンボーディング運用として TAP を準備（必要時）**
   - Temporary Access Pass を有効化し、失敗ユーザーに一時配布して Security info 登録を完了させる。
   - 期待結果: 初期登録時の認証詰まりを回避し、業務開始遅延を最小化。

## よくある原因

- **ポリシー適用差分（高）**: 失敗ユーザーが Authenticator 有効グループ外、または移行中設定差分の影響を受けている。
- **既存登録情報の不整合（中）**: 過去端末情報が残存し、プッシュ承認検証で失敗する。
- **端末通知/時刻同期問題（中）**: QR 登録後の承認通知が到達しない、またはタイムアウトする。
- **オンボーディング時の認証ブートストラップ不足（中）**: 初回登録前の認証導線が弱く、登録完了まで到達できない。

## 参考リンク

- [Combined security information registration in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-registration-mfa-sspr-combined)
- [Manage authentication methods for Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods-manage)
- [Microsoft Authenticator authentication method](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-authenticator-app)
- [Manage user authentication methods for Microsoft Entra multifactor authentication](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-userdevicesettings)
- [Configure Temporary Access Pass in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-authentication-temporary-access-pass)
- [Troubleshoot common issues with Azure Multi-Factor Authentication](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/mfa/troubleshoot-azure-mfa-issue)
