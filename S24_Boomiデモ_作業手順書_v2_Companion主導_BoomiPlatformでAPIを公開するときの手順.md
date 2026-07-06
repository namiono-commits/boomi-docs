# S24 Boomiデモ 作業手順書 v2（Boomi Companion 主導版）

各手順を **A：Companionへの指示（貼り付け用プロンプト）** → **B：手作業の設定（Companionではできない部分）** の2段で構成しています。 台本v2（案B・ダミーAPI・パターンA）／付属データ `inventory_dummy.csv`・`inventory_api_sample.json` に対応。

**考え方**：処理（フロー）の骨組み＝Companionに作らせる。接続の資格情報・エンドポイント値・デプロイ・API公開・通知先の認証・実行テストは**手作業**。Companionは"配線図"まで、"電源を入れる"のは人、というイメージ。 **注意**：Companionの生成範囲・AgentStudioのツール登録方式はバージョン依存です。**要確認**の箇所は実機で調整してください。プロンプトは日本語で記載していますが、英語でも可。

---

## 0\. 全体像（v1と同じ）

```
[ダミーデータ] → [在庫照会API(公開)] ──┬─► [業務プロセス] 呼び出し→分岐→通知
                                        └─► [AIエージェント] 同じ公開APIをツール化(パターンA)
```

業務プロセスとエージェントが**同じ公開API**を指すこと（＝S24の主張の核）。

---

## 手順1. ダミーデータ（SQL Server 投入済み）

**完了**：`dbo.inventory`（8件）を SQL Server に投入済み。以降は Database V2 connector で参照する。

- テーブル定義・INSERT・単一照会用SP・全件SELECT は付録B。  
- 単一照会は SP `dbo.usp_get_inventory_by_code(@code)` を使う（V2でパラメータ付きSQLを直接投げると弾かれるため）。全件は付録BのパラメータなしSELECTを直接パススルー。

---

## 手順2. 在庫照会API（ダミー）

### 2-A. Companionへの指示

```
Boomi Integrationで、在庫照会のリアルタイムREST APIを公開する処理を作成してください。
■ オペレーション2つ
 1) GET /inventory?code={product_code} … product_code で1件検索し、単一JSONを返す
 2) GET /inventory … 全件をJSONで返す
■ データソース: SQL Server のテーブル dbo.inventory（Database V2 connector で接続）
   列: product_code, product_name, unit, stock_qty, reorder_point, warehouse
■ SQLの組み方（Database V2 プロファイルの検証仕様に合わせること）
 ・全件(GET /inventory) … パラメータなしSQLなので、SELECT を JDBC パススルーでそのまま実行
 ・単一照会(GET /inventory?code=) … パラメータありなので、ストアドプロシージャ
   dbo.usp_get_inventory_by_code(@code) でラップして呼び出す
   （V2ではパラメータ付きSQLを直接投げるとプロファイル検証で弾かれるため、SP必須）
■ status は DB に保存せず、SQL側の CASE で導出する:
   CASE WHEN stock_qty <= reorder_point THEN 'REORDER' ELSE 'OK' END AS status
■ as_of（現在時刻）は単一照会レスポンスにのみ、処理側で付与する（DBには持たない）
■ 返却JSON
 ・単一照会: { "product_code","product_name","unit","stock_qty","reorder_point","warehouse","status","as_of" }
 ・全件: { "count": <件数>, "items": [ ...単一と同じ要素（as_of除く）... ] }
Web Services Server(listen) で公開し、リクエスト/レスポンスのプロファイルも生成してください。
```

### 2-B. 手作業

- **Database V2 connection** の資格情報（SQL Server 接続）を設定。ドライバ（`mssql-jdbc-*.jar`）が入ったアトムを使う（今回は Test/SnowflakeDemo）。  
- **Atom（またはクラウドAtom）へデプロイ**し、エンドポイントURLを控える。  
- 単体テスト：全件が `count:8` で返るか、単一が7列フル＋as\_of で返るか。

### 2-C. Database V2 で単一照会（code で1件）を作るときの重要な落とし穴 ★実地で確定した知見

このAPIで一番ハマったのが単一照会。Database V2 の仕様上、下記を守らないと動かない。

1. **`{ call sp(...) }` を Standard Get のSQL文に埋め込む方式はNG** … `@code` がパラメータとしてバインドされず、SPが引数なしで走る → 0行 → 空ドキュメント → `as_of` だけ返る。  
2. **SPを「Stored Procedure」オペレーション型で取り込む（GUI Import）と、今度は Response が Unstructured になり、7列を構造化JSONにマッピングできない** … V2の仕様。SP READ は構造化レスポンスにできない。  
3. **結論＝単一照会はSPを使わず実装する。** 有効だったのは次のいずれか：  
   - **(推奨・実採用) 全件GET＋プロセス内フィルタ**：全件を取得し、プロセス内で `code` で1件に絞る。全件が動く仕組みをそのまま流用でき、確実。  
   - パラメータ付き Standard Get（`WHERE product_code = ?` に `?` で**パラメータバインド**。SQL文への文字列連結ではない）。  
4. **全件の集約は Combine Documents が効かないことがある** … コネクタが行ごとに個別ドキュメントを生成し Combine で1つに畳めない場合、**Groovy スクリプトで1ドキュメント化**してから items\[\] と count を作る。  
5. SP `dbo.usp_get_inventory_by_code` 自体はSQL Server側に残してOK（今回は最終的に未使用だが無害）。`sp_describe_first_result_set` で結果セット7列が返ることは確認済み。

要点：**V2でSPを構造化レスポンスAPIにするのは相性が悪い。code絞り込みは「全件＋プロセス内フィルタ」が現実解。**

---

## 手順3. API Management で公開

ここはポータル操作が中心で手作業主体。Companionは公開対象処理の準備までなので、Aは軽め。

### 3-A. Companionへの指示（任意）

```
手順2で作った在庫照会APIについて、API Managementで公開するための
API仕様（OpenAPI）を出力できる形に整えてください。
```

### 3-B. 手作業

- API Management で手順2のエンドポイントを**API登録 → ゲートウェイ経由の公開URL発行**。  
- 認証は収録用途なら簡易（APIキー等）。**画面に映すなら顧客に見せられる値**に。  
- 公開URL（例 `https://<gateway>/inventory`）を控える。以降、手順4・5はこの公開URLを使う。  
- **要確認**：ゲートウェイのデプロイ先・ポリシーは実機バージョンで確認。

---

## 手順4. 業務プロセス（呼び出し → 分岐 → 通知）＝台本①

### 4-A. Companionへの指示

```
Boomi Integrationで、在庫の要補充チェック処理を作成してください。
1) HTTP Client で 公開API GET /inventory（全件）を呼び出す
2) レスポンスの items から status == "REORDER" の品目だけ抽出する
3) 抽出結果を Slack に「要補充リスト」として通知する
   （本文に product_code / product_name / stock_qty / reorder_point を含める）
4) 該当が0件のときは通知しない分岐も入れる
Slackが難しければ Mail connector でも可。
```

### 4-B. 手作業

- HTTP Client の**接続先を手順3の公開URL**に設定（APIキー等の認証も）。  
- **Slack（または Mail）接続の認証**・通知先チャンネルを設定。  
- 実行テスト：要補充4件（P-1002/1004/1006/1008）が通知に出るか確認。  
- （単一品目デモにするなら `GET /inventory?code=P-1002` を叩く版に差し替え）

---

## 手順5. AIエージェント（AgentStudio・パターンA）＝台本②

**最重要の要確認ポイント**：AgentStudioのツール登録方式（OpenAPI仕様インポート / コネクタ・MCP Source経由）。ここで5-Aの実現度が変わる。

### 5-A. Companionへの指示（方式が仕様インポートの場合）

```
AgentStudioで在庫問い合わせ用のエージェントを1体作成してください。
手順3で公開した在庫照会API（GET /inventory?code={product_code}）を
ツールとして登録し、在庫に関する質問が来たらこのツールを呼ぶよう指示してください。
回答には 在庫数(stock_qty)・補充点(reorder_point)・状態(status) を含めること。
```

### 5-B. 手作業

- **公開APIをツールとして登録**（方式は実機で確認。仕様インポートなら手順3のOpenAPIを使用）。  
- エージェントのシステムプロンプトに「在庫はこのツールで確認する」を明記。  
- 使用モデル・接続を設定。  
- 固定質問でテスト：「梱包用段ボール（P-1002）の在庫は？」→ 在庫12・要補充。潤沢例は P-1007。  
- **手順4と同じ公開URLを指しているか**を必ず確認（主張の核）。

---

## 手順6. テスト＆リハーサル

| ケース | 入力 | 期待結果 |
| :---- | :---- | :---- |
| 要補充の検知 | 業務プロセス実行 | P-1002/1004/1006/1008 が要補充通知 |
| 単一・要補充 | エージェントへ「P-1002の在庫は？」 | 在庫12・reorder 50・REORDER |
| 単一・潤沢 | エージェントへ「P-1007の在庫は？」 | 在庫62・OK |
| 同一API確認 | 手順4と手順5のエンドポイント | 一致していること |

通しリハ1回。②は応答時間がぶれるので質問文固定。画面ズーム・通知オフ・社名/実データ非表示。

---

## 付録A：API契約

**実際の稼働エンドポイント（Test/SnowflakeDemo・共有Webサーバ 9090）**

- 全件：`GET http://<host>:9090/ws/rest/openlegacy/inventory/inventory` → `{"count":8,"items":[...]}`  
- 単一：`GET http://<host>:9090/ws/rest/openlegacy/inventory/{code}` → 7列＋`as_of` の単一オブジェクト  
  - 例：`.../inventory/P-1002`（在庫12・REORDER）、`.../inventory/P-1007`（在庫62・OK）  
  - ※全件は `inventory/inventory` の二重パス（オブジェクト名＋リソースパス）。気になれば REST Config で一本化可（動作優先なら後回し）。

**単一** `GET .../inventory/P-1002`

```json
{ "product_code":"P-1002","product_name":"梱包用段ボール 60サイズ","unit":"箱",
  "stock_qty":12,"reorder_point":50,"warehouse":"東京DC","status":"REORDER",
  "as_of":"2026-07-05T09:00:00+09:00" }
```

**全件** `GET /inventory` → `{ "count":8, "items":[ ... ] }`（`inventory_api_sample.json` 参照） 判定：`status = (stock_qty <= reorder_point) ? "REORDER" : "OK"`

## 付録B：SQL（SQL Server利用時）

```sql
CREATE TABLE dbo.inventory (
  product_code  VARCHAR(20)  NOT NULL PRIMARY KEY,
  product_name  NVARCHAR(100) NOT NULL,
  unit          NVARCHAR(10)  NOT NULL,
  stock_qty     INT           NOT NULL,
  reorder_point INT           NOT NULL,
  warehouse     NVARCHAR(20)  NOT NULL
);
INSERT INTO dbo.inventory VALUES
('P-1001', N'ステンレスボルト M6×20', N'本', 480, 100, N'東京DC'),
('P-1002', N'梱包用段ボール 60サイズ', N'箱',  12,  50, N'東京DC'),
('P-1003', N'PETラベルシール A4',     N'巻', 250,  80, N'大阪DC'),
('P-1004', N'緩衝材 エアクッション 300m', N'本', 45, 60, N'東京DC'),
('P-1005', N'結束バンド 200mm 黒',     N'本',1500, 300, N'大阪DC'),
('P-1006', N'熱転写リボン 110mm',      N'巻',   8,  20, N'東京DC'),
('P-1007', N'樹脂パレット 1100×1100',  N'枚',  62,  30, N'大阪DC'),
('P-1008', N'ニトリル手袋 L 100枚',    N'箱',  90, 100, N'東京DC');
```

**全件取得（パラメータなし → V2で直接パススルー可）**

```sql
SELECT product_code, product_name, unit, stock_qty, reorder_point, warehouse,
       CASE WHEN stock_qty <= reorder_point THEN 'REORDER' ELSE 'OK' END AS status
FROM dbo.inventory;
```

**単一照会（パラメータあり → SPでラップ。V2の直投げ回避）**

```sql
CREATE PROCEDURE dbo.usp_get_inventory_by_code
  @code VARCHAR(20)
AS
BEGIN
  SET NOCOUNT ON;
  SELECT product_code, product_name, unit, stock_qty, reorder_point, warehouse,
         CASE WHEN stock_qty <= reorder_point THEN 'REORDER' ELSE 'OK' END AS status
  FROM dbo.inventory
  WHERE product_code = @code;
END
```

---

*v2 / Companion主導版。処理骨組み=Companion、接続・公開・認証・デプロイ・テスト=手作業。最優先の確認は手順5のツール登録方式。*  
