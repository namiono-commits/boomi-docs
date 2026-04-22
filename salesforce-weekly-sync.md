# Googleカレンダー → Salesforce 週次同期 手順書

> 毎週金曜日実施 / 作成: 2026年4月

---

## 1. 概要

Googleカレンダーに登録されているその週の予定を、Salesforceの Event (Activity) オブジェクトに一括登録する手順です。毎週金曜日に、その週（月曜日〜金曜日）の予定をまとめて処理します。

### 使用ツール

- Googleカレンダー（予定取得元）
- Claude（CSV作成・整形）
- [Salesforce Workbench](https://workbench.developerforce.com/)（CSVインポート）
- Salesforce（本番環境）

### 所要時間の目安

1回あたり約10〜15分（イベント件数による）

---

## 2. 事前準備（初回のみ）

以下は初回セットアップ済み。新しいPCに移行した場合などのみ再実施。

### Salesforceユーザー設定

1. Salesforceにログイン → 右上アイコン → **Settings**
2. 左メニュー **Personal Information** → **Language & Time Zone**
3. Time Zone を **(GMT+09:00) Japan Standard Time (Asia/Tokyo)** に設定
4. **Save**

### 重要な設定値

変更時はこの手順書も更新すること。

| 項目 | 値 |
|---|---|
| **Related To (WhatId)** | `0061W00000rnP4dQAE` (Japan Presales Generic Activities) |
| **Default Activity Type** | `Internal Meeting/Training/Enablement` |
| **Salesforce Object** | `Event` |
| **Time Zone** | Asia/Tokyo (JST, UTC+9) |

---

## 3. 毎週の作業フロー（金曜日実施）

### Step 1: ClaudeにCSV作成を依頼

Claudeのチャットに以下のフォーマットで依頼。対象期間を変えるだけでOK。

> Googleカレンダーから今週（◯月◯日〜◯月◯日）の予定を取り出して、Salesforce Event インポート用のCSVを作ってください。
>
> - WhatId = `0061W00000rnP4dQAE`
> - Activity_Type は自動分類（不明なら Internal Meeting/Training/Enablement）
> - Location列は含めない（ピックリストでエラーになるため）

Claudeが以下を含むCSVを生成する：

- `Subject`, `StartDateTime`, `EndDateTime`, `IsAllDayEvent`
- `Description`, `Type`, `ShowAs`
- `WhatId`, `Activity_Type__c`
- 参考用メモ列：`Response_Status__Note`, `Organizer__Note`, `Attendee_Count__Note`

### Step 2: CSVの確認・編集

ダウンロードしたCSVをExcelで開き、以下を確認・削除する。

| 削除対象 | 判断基準 |
|---|---|
| 未返信の予定 | `Response_Status__Note = needsAction` で実際に出席しなかったもの |
| 個人ブロック | ランチ、移動時間の仮押さえ、プライベートな予定など |
| 重複・取消 | 同じ予定が2つ入っている、結局キャンセルになったもの |
| Activity_Typeの誤分類 | Claudeの自動分類が違っていたら値を直接修正（Demo / Partner Facing Presales など） |

編集後、上書き保存（CSV UTF-8形式のまま）。

### Step 3: Workbenchでインポート

1. <https://workbench.developerforce.com/> にアクセス
2. Environment: **Production** → **Login with Salesforce**（SSOでログイン）
3. 上部メニュー **data → Insert**
4. 設定画面で以下を選択：
   - Object Type: `Event`
   - From File: 編集済みCSVを選択
   - Process records asynchronously via Bulk API: ✅チェック
   - Assign null values to mapped fields...: ✅チェック
5. **Next** → フィールドマッピング画面

#### 必須マッピング（9項目）

| Salesforce Field | CSV Column | 必須 |
|---|---|---|
| `Subject` | `Subject` | ★必須 |
| `StartDateTime` | `StartDateTime` | ★必須 |
| `EndDateTime` | `EndDateTime` | ★必須 |
| `WhatId` | `WhatId` | ★必須 |
| `Activity_Type__c` | `Activity_Type__c` | ★必須 |
| `IsAllDayEvent` | `IsAllDayEvent` | 推奨 |
| `Description` | `Description` | 推奨 |
| `ShowAs` | `ShowAs` | 任意 |
| `Type` | `Type` | 任意（組織で非表示の場合あり） |

> メモ列（`Response_Status__Note` / `Organizer__Note` / `Attendee_Count__Note`）は未マッピングのままでOK。

6. **Map Fields** → **Confirm Insert** をクリック
7. 実行。Bulk APIの場合、結果が出るまで1〜2分

### Step 4: Salesforceで確認

1. Workbenchの結果画面で全件 **Success / Created** を確認
2. Salesforceにログイン → カレンダーを開く
3. 対象週にイベントが表示されているか目視確認
4. 時刻が正しい（JST）ことを1〜2件サンプル確認
5. Activity Typeが適切か確認。違うものは個別に編集

---

## 4. トラブルシューティング

| 症状 | 対処 |
|---|---|
| 時刻が9時間ずれる | SalesforceユーザーのTime Zoneが `Asia/Tokyo` になっているか確認（第2章参照） |
| `REQUIRED_FIELD_MISSING` エラー | 必須マッピング（Subject / Start / End / WhatId / Activity_Type__c）が漏れている |
| `INVALID_CROSS_REFERENCE_KEY` | WhatIdの値が間違っている、または該当Opportunityが削除された。URLから最新IDを取り直す |
| Activity Typeでエラー | ピックリスト値が変更されている可能性。SalesforceのEvent新規作成画面で最新の選択肢を確認 |
| Workbenchにログインできない | ブラウザを変える、キャッシュクリア、またはSalesforceに先にログインしてから試す |
| 間違えて重複登録した | Workbench → **data → Delete** で該当EventのIDを指定して削除可能 |

---

## 5. 効率化のコツ

- 金曜日の午後に固定化（例：17:00）すると習慣化しやすい
- 件数が少ない週（〜10件）は、CSV経由より Salesforce画面で直接登録する方が早い場合も
- Activity Typeの分類ルールで不満があれば、Claudeに「Demoと判定する条件を増やして」など調整依頼可能
- 管理者に **Einstein Activity Capture (EAC)** の有効化を相談すれば、この作業自体が不要になる可能性あり

---

## 6. 参考情報

### Activity Type 選択肢一覧

2026年4月時点でSalesforceで使える主な Activity Type：

- `Internal Meeting/Training/Enablement`（デフォルト）
- `Admin/Misc`, `Account Research`, `Content Create`
- `Demo`, `Demo Prep`, `Discovery`
- `GTM Campaign`, `Internal Initiative`, `Marketing Event/Support`
- `Partner Facing Presales`, `Product Evaluation`
- `RFI/RFP`, `Solution Create`, `Stage Five Handoff`
- `Technical Strategy Session`, `Travel`, `Strategic Review`, `Strategic Pursuit`
- `CSE – Enablement` / `Evangelism` / `Partner Management` / `Partner Opportunity` / `Tech Assistance` / `Tech Proof & Content`
- `Partner Tech Success`, `Customer Tech Success`, `Custom Solution Internal Build`, `Custom Solution External Build`

### 変更履歴

| 日付 | 変更者 | 内容 |
|---|---|---|
| 2026-04-22 | Nami Ono / Claude | 初版作成 |
