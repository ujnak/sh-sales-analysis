---
applyTo: "**"
---

# Oracle SHスキーマ 分析ガイドライン

このプロジェクトでは Oracle Database の **SH（Sales History）スキーマ** を分析対象とする。
以下のルールに従ってSQLを構築し、分析結果を提示すること。

## スキーマ構成

SHスキーマはスタースキーマ構造。中心にファクトテーブル `SH.SALES` があり、
4つのディメンションテーブルと結合して使う。

### ファクトテーブル

| テーブル | 主な用途 |
|---------|---------|
| `SH.SALES` | 販売トランザクション。売上金額・数量の集計の起点 |
| `SH.COSTS` | 商品コスト。収益性分析（粗利）に使う |

**SH.SALES の主要カラム**
- `PROD_ID` — 商品ID
- `CUST_ID` — 顧客ID
- `TIME_ID` — 日付（DATE型）
- `CHANNEL_ID` — チャネルID
- `PROMO_ID` — プロモーションID（999 = プロモーションなし）
- `QUANTITY_SOLD` — 販売数量（NUMBER）
- `AMOUNT_SOLD` — 販売金額（NUMBER）

### ディメンションテーブル

| テーブル | 結合キー | 主な分析軸 |
|---------|---------|-----------|
| `SH.PRODUCTS` | `PROD_ID` | 商品名、カテゴリ、サブカテゴリ、定価 |
| `SH.CUSTOMERS` | `CUST_ID` | 顧客名、都市、都道府県、所得レベル |
| `SH.TIMES` | `TIME_ID` | 年月（`CALENDAR_MONTH_DESC`）、月番号、会計月 |
| `SH.CHANNELS` | `CHANNEL_ID` | チャネル区分（Direct Sales / Internet / Partners / Tele Sales） |
| `SH.PROMOTIONS` | `PROMO_ID` | プロモーション名、カテゴリ |

## SQL作成ルール

### 必ずスキーマ修飾子をつける
```sql
-- OK
SELECT * FROM SH.SALES s JOIN SH.TIMES t ON s.TIME_ID = t.TIME_ID

-- NG（スキーマ名を省略しない）
SELECT * FROM SALES s JOIN TIMES t ON s.TIME_ID = t.TIME_ID
```

### 件数制限には ROWNUM を使う（Oracle構文）
```sql
-- OK: Oracle
SELECT * FROM (...) WHERE ROWNUM <= 10

-- NG: Oracle では LIMIT は使えない
SELECT * FROM ... LIMIT 10
```

### 日付フィルタのパターン
```sql
-- 月指定（CALENDAR_MONTH_DESC は 'YYYY-MM' 形式の文字列）
WHERE t.CALENDAR_MONTH_DESC = '2001-06'

-- 年指定
WHERE EXTRACT(YEAR FROM s.TIME_ID) = 2001

-- 日付範囲
WHERE s.TIME_ID BETWEEN DATE '2001-01-01' AND DATE '2001-12-31'
```

### ウィンドウ関数（分析関数）はOracle標準構文で
```sql
-- 前月比
LAG(売上) OVER (ORDER BY 月)

-- 構成比
SUM(AMOUNT_SOLD) / SUM(SUM(AMOUNT_SOLD)) OVER () * 100

-- ランク
RANK() OVER (ORDER BY SUM(AMOUNT_SOLD) DESC)
```

### 粗利の計算（SH.COSTSを使う場合）
```sql
SELECT p.PROD_NAME,
       SUM(s.AMOUNT_SOLD) AS 売上合計,
       SUM(s.QUANTITY_SOLD * c.UNIT_COST) AS コスト合計,
       SUM(s.AMOUNT_SOLD) - SUM(s.QUANTITY_SOLD * c.UNIT_COST) AS 粗利
FROM SH.SALES s
JOIN SH.PRODUCTS p ON s.PROD_ID = p.PROD_ID
JOIN SH.COSTS c
  ON s.PROD_ID = c.PROD_ID
  AND s.TIME_ID = c.TIME_ID
  AND s.CHANNEL_ID = c.CHANNEL_ID
  AND s.PROMO_ID = c.PROMO_ID
GROUP BY p.PROD_NAME
ORDER BY 粗利 DESC
```
`SH.COSTS` との結合キーは `PROD_ID + TIME_ID + CHANNEL_ID + PROMO_ID` の4列。

### プロモーション分析の注意
`PROMO_ID = 999` は「プロモーションなし」を意味する特別値。
プロモーション効果を見るときは除外するか、明示的に区別すること。

```sql
-- プロモーションあり/なしで比較
SELECT CASE WHEN s.PROMO_ID = 999 THEN 'プロモーションなし' ELSE 'プロモーションあり' END AS プロモ区分,
       SUM(s.AMOUNT_SOLD) AS 売上合計
FROM SH.SALES s
GROUP BY CASE WHEN s.PROMO_ID = 999 THEN 'プロモーションなし' ELSE 'プロモーションあり' END
```

## 出力フォーマット

- 金額は **カンマ区切り**（例: `1,234,567`）で表示する
- 変化率は **符号付き**（例: `+12.3%` / `-5.6%`）で表示する
- テーブル形式で整理し、**ポイントコメント**を末尾に添える
- ランキングは特段の指定がなければ **上位10件** に絞る
- 実行したSQLはユーザーが求めた場合や確認が必要な場合に開示する

## スキーマ確認が必要な場合

カラム名やデータ型が不明なときは `get_schema` ツールで確認してからSQLを構築する。
推測でSQLを実行してエラーを出すより、スキーマ確認を優先する。
