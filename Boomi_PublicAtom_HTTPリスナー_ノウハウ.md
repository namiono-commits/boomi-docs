# Boomi パブリックAtomクラウドで HTTPリスナー（Web Services Server）を立てるノウハウ

作成日: 2026-08-01

Boomi の Web Services Server（Listen）でシンプルなHTTP APIエンドポイントを作り、パブリックAtomクラウドをランタイムとして公開・呼び出しするまでの手順と、実際にハマったポイントをまとめたもの。

---

## 0. 全体像（最短ルート）

1. Web Services Server オペレーションを作成（Listen / GET / Simple URL パス / 応答プロファイル）
2. プロセスを作成: **開始シェイプ = Web Services Server** → メッセージ → リターンドキュメント
3. Shared Web Server で **Basic認証** を設定し、APIユーザーを作成
4. プロセスを**最新リビジョンでパッケージ化 → パブリッククラウド環境へデプロイ**
5. `https://<Base>/<Simple URLパス>` を Basic認証付きで呼び出し

これらが1つでも欠けると、401 / 404 / 文字化けなどでハマる。以下、要点。

---

## 1. 認証（パブリッククラウドは None 不可）

**パブリックランタイムクラウドは認証タイプ「None」をサポートしない。** 最低でも Basic 認証が必須。

- 設定場所は**オペレーション画面ではない**。
  `Manage → Runtime Management → 対象クラウド → Settings & Configuration → Shared Web Server → Listening Port Configuration → Authentication Type = Basic`
- 設定変更後は**ランタイムへの適用（反映）が必要**。
- 編集には **Runtime Management 権限**が必要（読み取り専用だと変更不可）。

### ユーザー名は「完全修飾名」が必須

クラウドはユーザー名でどのアカウントのデプロイに振り分けるかを判断するため、**`user@accountID` 形式の完全修飾名**を使う。

- 例: `demouser@boomi_japan_workshop-XXXXXX.XXXXXX`
- User Management の入力欄には短縮名（`demouser`）を入れるが、**認証に使うのは薄いグレーで表示される完全修飾名**。
- 短縮名だけだと **401**。

### トークン = パスワード

- Basic認証の パスワードには User Management で発行した**トークン**を使う。
- トークンを**再生成すると旧トークンは即無効**。配布された古い値だと401になる。
- **コピー時の末尾スペース/改行の混入**が401の隠れ原因として非常に多い。`.Trim()` で除去して確認する。

---

## 2. デプロイ必須（テストモードは使えない）

- **リスナー型（リアルタイム）プロセスはキャンバスのテストモードで実行できない。** 実際のHTTPリクエストを受けて動くため。
- **デプロイして外部からエンドポイントを叩くのが唯一のテスト方法。**
- デプロイ前は認証段階で弾かれ **401**、デプロイ後にアカウントが解決されるようになる（今回、401 → 404 に変わったのがこの証拠）。

### パッケージは「最新リビジョン」で作り直す

- パッケージは**作成時点のスナップショット**を固める。
- 「開始シェイプやパスを設定する前に一度パッケージ化 → その後修正 → 古いパッケージをデプロイ」だと、実行版が古いままで **404**。
- **設計を変えたら: 保存 → 最新リビジョンで新規パッケージ作成 → 環境へ再デプロイ（上書き）**。

### デプロイ先の環境

- デプロイ先の Environment が、狙っているパブリッククラウド（例: `c01-jp`）に対応しているか確認。
- デプロイ直後は**リスナー登録に数十秒**かかることがある。

---

## 3. オペレーションのパス設計（Object 欄に注意）

- **Simple URL パスは Object 欄の値で変わる。** ここが今回の最大の落とし穴だった。
  - Object = `test` → パス `/ws/simple/getTest`
  - Object = 空 → パス `/ws/simple/get`
- **呼び出しURL = Base + Simple URLパス。**
  - 例: `https://c01-jp.integrate-test.boomi.com/ws/simple/get`
- パスは**画面の「シンプルURLパス」欄に表示される文字列をそのままコピー**するのが確実（綴り・大文字小文字ミスを防ぐ）。
- Object を変更したら**パスも変わるので、呼び出し側URLも合わせて修正 + 再デプロイ**。

---

## 4. プロセス構成

正しい構成:

```
[開始シェイプ: Web Services Server (オペレーション)] → [メッセージ or マップ] → [リターンドキュメント]
```

- **Web Services Server が「開始シェイプ」であること。** 途中のステップに置くとパスが登録されず404。
- デプロイ対象は**オペレーション単体ではなくプロセス本体**。
- レスポンス（Return する値）の作り方:
  - **固定値のテスト → メッセージシェイプ**にJSONを直接記述（一番手軽）。
  - **実データ・変換あり → マップシェイプ**で応答プロファイルへマッピング。
- 「レスポンスの出力タイプ / 応答プロファイル」で定義したものが、そのまま**呼び出し元への戻り値（Return）**になる。
- 「期待される入力タイプ = なし」は**リクエスト側**の話。レスポンス側とは別物。

メッセージシェイプの例（応答プロファイル: object > data1, data2）:

```json
{
  "object": {
    "data1": "テスト値1",
    "data2": "テスト値2"
  }
}
```

---

## 5. 呼び出し方（PowerShell）

```powershell
$user  = "demouser@boomi_japan_workshop-XXXXXX.XXXXXX".Trim()
$token = "<トークン>".Trim()
$url   = "https://c01-jp.integrate-test.boomi.com/ws/simple/get"

$pair    = "$($user):$($token)"
$b64     = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($pair))
$headers = @{ Authorization = "Basic $b64" }

Invoke-RestMethod -Uri $url -Method Get -Headers $headers
```

- GETならボディ不要。HTTPS必須。
- 送信ヘッダーの中身を目視確認したいとき:
  `[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($b64))`

---

## 6. 文字化け対策（日本語）

BoomiのレスポンスはUTF-8だが、**Windows PowerShell 5.1 は既定でISO-8859-1としてデコード**するため日本語が化ける（例: `テスト値1` → `ãã¹ãå¤1`）。デコード側の問題。

**方法A: Invoke-WebRequest で生バイトをUTF-8デコード（PS5.1でも確実）**

```powershell
$resp = Invoke-WebRequest -Uri $url -Method Get -Headers $headers
$json = [Text.Encoding]::UTF8.GetString($resp.RawContentStream.ToArray())
$data = $json | ConvertFrom-Json
$data.object.data1
```

**方法B: PowerShell 7（pwsh）で実行** — 既定でUTF-8扱いのため `Invoke-RestMethod` のまま化けない。

コンソール表示自体もUTF-8に:

```powershell
[Console]::OutputEncoding = [Text.Encoding]::UTF8
```

補足: `@{data1=...; data2=...}` という表示は文字化けではなく、**PowerShellが入れ子オブジェクトを表示する標準形式**。値は `$data.object.data1` で取得、全体は `$data | ConvertTo-Json` で確認できる。

---

## 7. エラーコードで切り分け

| コード | 意味 | 主な原因と対処 |
|--------|------|----------------|
| **401 Unauthorized** | 認証NG | 認証タイプがBasicか / ユーザー名が完全修飾名か / トークンが最新か / 末尾スペース混入 / 未デプロイでアカウント未解決 |
| **404 Unknown operation for the given URL path** | パスにリスナー未登録 | パス違い（Object/綴り）/ Web Services Serverが開始シェイプでない / プロセス未デプロイ・古いパッケージ / メソッド不一致 |
| **404（別環境）** | デプロイ先違い | 狙ったクラウドに対応する環境へデプロイしたか |
| **500** | プロセス内部エラー | プロセス（メッセージ/マップ/プロファイル）の設定を確認 |

---

## 8. チェックリスト（配布用）

- [ ] Shared Web Server の Authentication Type = Basic（保存・反映済み）
- [ ] APIユーザーを User Management で作成、トークンを控えた
- [ ] Basic認証のユーザー名は**完全修飾名**を使用
- [ ] プロセスの**開始シェイプ = Web Services Server**
- [ ] Object 欄の値を確定し、**Simple URLパスと呼び出しURLを一致**させた
- [ ] **最新リビジョンでパッケージ化**し、対象クラウド環境へ**デプロイ**
- [ ] 変更後は**再デプロイ**した
- [ ] 呼び出しは **HTTPS + Basic認証**、メソッドはオペレーションと一致
- [ ] 文字化けは UTF-8デコード or PS7 で対処
```
