---
name: sales-analysis
description: >
  Oracle Database（SHスキーマ）の販売データを分析するための包括的なスキル。
  ユーザーが「売上を調べて」「販売データを見せて」「売上分析して」「どの商品が売れてる？」
  「顧客ランキングを出して」「月ごとの売上は？」「前月比を教えて」などと言ったときに使用する。
  SH.SALES, SH.PRODUCTS, SH.CUSTOMERS, SH.TIMES, SH.CHANNELS テーブルを活用する。
---

# 販売データ分析スキル

このスキルはOracle SHスキーマの販売データにアクセスして分析を行う。

## データベース構造

### 主要テーブル

**SH.SALES**（販売トランザクション）
- `PROD_ID` — 商品ID（SH.PRODUCTSと結合）
- `CUST_ID` — 顧客ID（SH.CUSTOMERSと結合）
- `TIME_ID` — 日付（SH.TIMESと結合）
- `CHANNEL_ID` — チャネルID（SH.CHANNELSと結合）
- `PROMO_ID` — プロモーションID
- `QUANTITY_SOLD` — 販売数量
- `AMOUNT_SOLD` — 販売金額

**SH.PRODUCTS**（商品マスタ）
- `PROD_ID`, `PROD_NAME`, `PROD_CATEGORY`, `PROD_SUBCATEGORY`, `PROD_LIST_PRICE`, `PROD_STATUS`

**SH.CUSTOMERS**（顧客マスタ）
- `CUST_ID`, `CUST_FIRST_NAME`, `CUST_LAST_NAME`, `CUST_CITY`, `CUST_STATE_PROVINCE`, `CUST_INCOME_LEVEL`

**SH.TIMES**（日付ディメンション）
- `TIME_ID`, `CALENDAR_MONTH_DESC`（例: "2001-01"）, `CALENDAR_MONTH_NUMBER`, `FISCAL_MONTH_DESC`

**SH.CHANNELS**（販売チャネル）
- `CHANNEL_ID`, `CHANNEL_DESC`（例: "Direct Sales", "Internet"）

## 分析パターン

### 期間別売上集計
```sql
SELECT t.CALENDAR_MONTH_DESC AS 月,
       SUM(s.AMOUNT_SOLD) AS 売上合計,
       SUM(s.QUANTITY_SOLD) AS 販売数量
FROM SH.SALES s
JOIN SH.TIMES t ON s.TIME_ID = t.TIME_ID
WHERE t.CALENDAR_MONTH_DESC BETWEEN '2001-01' AND '2001-12'
GROUP BY t.CALENDAR_MONTH_DESC
ORDER BY t.CALENDAR_MONTH_DESC
```

### 商品カテゴリ別売上
```sql
SELECT p.PROD_CATEGORY AS カテゴリ,
       SUM(s.AMOUNT_SOLD) AS 売上合計,
       RANK() OVER (ORDER BY SUM(s.AMOUNT_SOLD) DESC) AS ランク
FROM SH.SALES s
JOIN SH.PRODUCTS p ON s.PROD_ID = p.PROD_ID
GROUP BY p.PROD_CATEGORY
ORDER BY 売上合計 DESC
```

### 前月比トレンド
```sql
SELECT t.CALENDAR_MONTH_DESC AS 月,
       SUM(s.AMOUNT_SOLD) AS 当月売上,
       LAG(SUM(s.AMOUNT_SOLD)) OVER (ORDER BY t.CALENDAR_MONTH_DESC) AS 前月売上,
       ROUND((SUM(s.AMOUNT_SOLD) - LAG(SUM(s.AMOUNT_SOLD)) OVER (ORDER BY t.CALENDAR_MONTH_DESC))
             / LAG(SUM(s.AMOUNT_SOLD)) OVER (ORDER BY t.CALENDAR_MONTH_DESC) * 100, 1) AS 前月比率
FROM SH.SALES s
JOIN SH.TIMES t ON s.TIME_ID = t.TIME_ID
GROUP BY t.CALENDAR_MONTH_DESC
ORDER BY t.CALENDAR_MONTH_DESC
```

## 出力ガイドライン

- 数値はカンマ区切りで読みやすく表示する（例: 1,234,567）
- 金額の単位を明示する
- ランキングは上位10件程度に絞る
- 結果はMarkdownテーブルで整理して表示する
- 分析のポイントや気づきを簡潔にコメントする

詳細なSQLパターンは `references/sql-patterns.md` を参照。
