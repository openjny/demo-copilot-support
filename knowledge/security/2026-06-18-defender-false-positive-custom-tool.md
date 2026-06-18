---
title: "Microsoft Defender が内製ツールを誤検知した場合の安全な許可手順（MDE/Intune）"
category: "security"
tags: ["defender-for-endpoint", "false-positive", "intune", "allow-indicator", "antivirus-exclusion"]
created: "2026-06-18"
updated: "2026-06-18"
issue_ref: [13]
sources:
  - "https://learn.microsoft.com/en-us/defender-endpoint/address-unwanted-behaviors-mde"
  - "https://learn.microsoft.com/en-us/defender-endpoint/defender-endpoint-false-positives-negatives"
  - "https://learn.microsoft.com/en-us/defender-endpoint/admin-submissions-mde"
  - "https://learn.microsoft.com/en-us/defender-endpoint/indicator-file"
  - "https://learn.microsoft.com/en-us/defender-endpoint/configure-exclusions-microsoft-defender-antivirus"
  - "https://learn.microsoft.com/en-us/defender-endpoint/navigate-defender-endpoint-antivirus-exclusions"
  - "https://learn.microsoft.com/en-us/defender-endpoint/common-exclusion-mistakes-microsoft-defender-antivirus"
---

# Microsoft Defender が内製ツールを誤検知した場合の安全な許可手順（MDE/Intune）

## 概要

Microsoft Defender for Endpoint (MDE) で正規ツールが誤検知された場合、Microsoft 公式ガイダンスでは
**いきなり広い除外を入れず**、まず検知ソースの確認・誤検知提出を行い、必要最小限の許可を段階的に適用することが推奨される。

特に内製 EXE（署名なし）では、証明書ベース許可は使えないため、暫定対応はファイルハッシュ許可または限定的な除外になる。
ただし除外は保護レベルを下げるため、範囲を最小化し、恒久化しない運用が重要。

## 詳細

### 公式ドキュメントで確認できる事実

- Defender の誤検知対応は、検知ソース確認 → 必要に応じた抑止/提出/指標/除外の順で進める運用が示されている。
- 誤検知時は Microsoft へのファイル/ハッシュ提出（Clean として提出）が推奨される。
- File indicator（Allow/Block）には前提条件があり、Windows では Defender AV active mode、クラウド保護、Behavior Monitoring などが必要。
- 除外設定は Defender 保護を低下させ、マルウェア保護・IOC・ASR などに影響し得るため、最小範囲で運用し定期監査が必要。
- 拡張子 `.exe` や広範囲フォルダーの除外は推奨されない。

### この問い合わせに当てはめた判断

- 検出名 `Trojan:Win32/Wacatac.B!ml` は機械学習系の検知名であり、これだけで誤検知/真陽性を断定できない（未確認）。
- ただし「定義更新後に複数端末で同時発生」「社内で長期利用の同一ツール」という状況は、誤検知シナリオと整合する（推測）。
- 署名なし EXE のため、証明書 IOC による許可は適用不可。実運用上はハッシュ許可または限定除外が候補。

## 手順

1. **検知ソースと対象ファイルを特定する**
   - Defender ポータルで該当アラートの detection source（Antivirus/EDR/ASR など）を確認。
   - 影響端末で対象 EXE の SHA-256 を採取し、全端末で同一バイナリか確認する。

2. **Microsoft へ誤検知（False Positive）を提出する**
   - `security.microsoft.com` の **Actions & submissions > Submissions** から file または file hash を **Clean** として提出する。
   - 緊急度、検知名、定義バージョン、影響台数を添付し、提出履歴を追跡する。

3. **暫定回避として「最小範囲」の許可を適用する**
   - 第一候補: MDE の **File hash Allow indicator** を作成する（設定: Endpoints > Indicators）。
   - 影響端末だけにスコープを限定し、期限（expiration）を設定する。
   - 注意: ツール更新でハッシュが変わると再登録が必要。

4. **ハッシュ許可で運用できない場合のみ、Intune で限定除外を使う**
   - Intune Endpoint security > Antivirus で、特定ファイル/特定パスの除外を作成する。
   - 拡張子除外（例: `.exe`）や広範囲パス除外（例: `C:\`、`C:\Users\*`）は避ける。
   - 影響範囲: 除外対象では Defender の検査/ブロック能力が低下し、関連機能にも波及し得る。

5. **復旧確認とロールバック**
   - 業務影響が解消したことを確認し、Microsoft 側の再判定・定義更新後に暫定許可/除外を段階的に削除する。
   - 再発監視のため、同一ハッシュ・同一パスの検知を継続監視する。

## よくある原因

- 定義更新やクラウド判定ロジック更新後に、特定バイナリが一時的に誤検知される
- 署名なし/更新頻度の高い内製 EXE で、判定が不安定になる
- 暫定除外を広く入れすぎて、別リスクを増やしてしまう

## 参考リンク

- [Address unwanted behaviors in Microsoft Defender for Endpoint](https://learn.microsoft.com/en-us/defender-endpoint/address-unwanted-behaviors-mde)
- [Address false positives/negatives in Microsoft Defender for Endpoint](https://learn.microsoft.com/en-us/defender-endpoint/defender-endpoint-false-positives-negatives)
- [Submit files using the unified submissions portal in Defender for Endpoint](https://learn.microsoft.com/en-us/defender-endpoint/admin-submissions-mde)
- [Create indicators for files](https://learn.microsoft.com/en-us/defender-endpoint/indicator-file)
- [Exclusions overview](https://learn.microsoft.com/en-us/defender-endpoint/navigate-defender-endpoint-antivirus-exclusions)
- [Configure custom exclusions for Microsoft Defender Antivirus](https://learn.microsoft.com/en-us/defender-endpoint/configure-exclusions-microsoft-defender-antivirus)
- [Common mistakes to avoid when defining exclusions](https://learn.microsoft.com/en-us/defender-endpoint/common-exclusion-mistakes-microsoft-defender-antivirus)
