# SQLパターン集（SHスキーマ販売分析）

## 売上集計クエリ

### 年間売上サマリ
```sql
SELECT EXTRACT(YEAR FROM s.TIME_ID) AS 年,
       SUM(s.AMOUNT_SOLD) AS 売上合計,
       SUM(s.QUANTITY_SOLD) AS 販売数量合計,
       COUNT(DISTINCT s.CUST_ID) AS ユニーク顧客数
FROM SH.SALES s
GROUP BY EXTRACT(YEAR FROM s.TIME_ID)
ORDER BY 年
```

### 月次売上推移
```sql
SELECT t.CALENDAR_MONTH_DESC AS 年月,
       SUM(s.AMOUNT_SOLD) AS 売上合計,
       SUM(s.QUANTITY_SOLD) AS 数量
FROM SH.SALES s
JOIN SH.TIMES t ON s.TIME_ID = t.TIME_ID
GROUP BY t.CALENDAR_MONTH_DESC
ORDER BY t.CALENDAR_MONTH_DESC
```

## ランキングクエリ

### 商品別売上ランキング（上位N件）
```sql
SELECT ROWNUM AS ランク, 商品名, 売上合計, 販売数量
FROM (
  SELECT p.PROD_NAME AS 商品名,
         SUM(s.AMOUNT_SOLD) AS 売上合計,
         SUM(s.QUANTITY_SOLD) AS 販売数量
  FROM SH.SALES s
  JOIN SH.PRODUCTS p ON s.PROD_ID = p.PROD_ID
  GROUP BY p.PROD_NAME
  ORDER BY 売上合計 DESC
)
WHERE ROWNUM <= 10
```

### カテゴリ別売上ランキング
```sql
SELECT p.PROD_CATEGORY AS カテゴリ,
       SUM(s.AMOUNT_SOLD) AS 売上合計,
       ROUND(SUM(s.AMOUNT_SOLD) / SUM(SUM(s.AMOUNT_SOLD)) OVER () * 100, 1) AS 構成比率
FROM SH.SALES s
JOIN SH.PRODUCTS p ON s.PROD_ID = p.PROD_ID
GROUP BY p.PROD_CATEGORY
ORDER BY 売上合計 DESC
```

### 顧客別売上ランキング（上位N件）
```sql
SELECT ROWNUM AS ランク, 顧客名, 売上合計, 購入回数
FROM (
  SELECT c.CUST_FIRST_NAME || ' ' || c.CUST_LAST_NAME AS 顧客名,
         SUM(s.AMOUNT_SOLD) AS 売上合計,
         COUNT(*) AS 購入回数
  FROM SH.SALES s
  JOIN SH.CUSTOMERS c ON s.CUST_ID = c.CUST_ID
  GROUP BY c.CUST_FIRST_NAME, c.CUST_LAST_NAME
  ORDER BY 売上合計 DESC
)
WHERE ROWNUM <= 10
```

### チャネル別売上ランキング
```sql
SELECT ch.CHANNEL_DESC AS チャネル,
       SUM(s.AMOUNT_SOLD) AS 売上合計,
       ROUND(SUM(s.AMOUNT_SOLD) / SUM(SUM(s.AMOUNT_SOLD)) OVER () * 100, 1) AS 構成比率
FROM SH.SALES s
JOIN SH.CHANNELS ch ON s.CHANNEL_ID = ch.CHANNEL_ID
GROUP BY ch.CHANNEL_DESC
ORDER BY 売上合計 DESC
```

## トレンド分析クエリ

### 前月比トレンド
```sql
SELECT 月, 当月売上,
       LAG(当月売上) OVER (ORDER BY 月) AS 前月売上,
       ROUND((当月売上 - LAG(当月売上) OVER (ORDER BY 月))
             / LAG(当月売上) OVER (ORDER BY 月) * 100, 1) AS 前月比率
FROM (
  SELECT t.CALENDAR_MONTH_DESC AS 月,
         SUM(s.AMOUNT_SOLD) AS 当月売上
  FROM SH.SALES s
  JOIN SH.TIMES t ON s.TIME_ID = t.TIME_ID
  GROUP BY t.CALENDAR_MONTH_DESC
)
ORDER BY 月
```

### 前年同月比
```sql
SELECT 年, 月番号, 年月売上,
       LAG(年月売上) OVER (PARTITION BY 月番号 ORDER BY 年) AS 前年同月売上,
       ROUND((年月売上 - LAG(年月売上) OVER (PARTITION BY 月番号 ORDER BY 年))
             / LAG(年月売上) OVER (PARTITION BY 月番号 ORDER BY 年) * 100, 1) AS 前年比率
FROM (
  SELECT EXTRACT(YEAR FROM s.TIME_ID) AS 年,
         EXTRACT(MONTH FROM s.TIME_ID) AS 月番号,
         SUM(s.AMOUNT_SOLD) AS 年月売上
  FROM SH.SALES s
  GROUP BY EXTRACT(YEAR FROM s.TIME_ID), EXTRACT(MONTH FROM s.TIME_ID)
)
ORDER BY 年, 月番号
```

### 累計売上（年内）
```sql
SELECT t.CALENDAR_MONTH_DESC AS 年月,
       SUM(s.AMOUNT_SOLD) AS 月次売上,
       SUM(SUM(s.AMOUNT_SOLD)) OVER (
         PARTITION BY EXTRACT(YEAR FROM t.TIME_ID)
         ORDER BY t.CALENDAR_MONTH_DESC
       ) AS 累計売上
FROM SH.SALES s
JOIN SH.TIMES t ON s.TIME_ID = t.TIME_ID
GROUP BY t.CALENDAR_MONTH_DESC, EXTRACT(YEAR FROM t.TIME_ID)
ORDER BY t.CALENDAR_MONTH_DESC
```

## フィルタリングパターン

### 特定期間の絞り込み
```sql
-- 月で絞り込む例
WHERE t.CALENDAR_MONTH_DESC = '2001-06'

-- 年で絞り込む例
WHERE EXTRACT(YEAR FROM s.TIME_ID) = 2001

-- 日付範囲で絞り込む例
WHERE s.TIME_ID BETWEEN DATE '2001-01-01' AND DATE '2001-12-31'
```

### 特定チャネルで絞り込む
```sql
WHERE ch.CHANNEL_DESC = 'Direct Sales'
-- チャネル一覧: Direct Sales, Internet, Partners, Tele Sales
```
