---
name: ranking
description: >
  商品・顧客・カテゴリ・チャネルの売上ランキングを表示する。
  ユーザーが「売れ筋商品は？」「優良顧客を教えて」「人気カテゴリを見せて」
  「トップ10を出して」「ランキングを教えて」などと言ったときに使用する。
  Oracle SHスキーマから集計してランキング形式で表示する。
user-invocable: true
argument-hint: "[種別: 商品|カテゴリ|顧客|チャネル] [件数] [期間]"
---

# ランキングスキル

指定した種別・期間の売上ランキングをOracle SHスキーマから取得してチャットに表示する。

## 実行手順

1. ユーザーのメッセージからランキング条件を解釈する
   - 種別: 商品（PROD_NAME）、カテゴリ（PROD_CATEGORY）、顧客（CUST_FIRST_NAME + CUST_LAST_NAME）、チャネル（CHANNEL_DESC）
   - 件数: デフォルト10件
   - 期間: 省略時は全期間

2. 以下のパターンでSQLを組み立てて `run_sql` ツールで実行する

**商品別ランキング（例）:**
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

**カテゴリ別（構成比付き）:**
```sql
SELECT p.PROD_CATEGORY AS カテゴリ,
       SUM(s.AMOUNT_SOLD) AS 売上合計,
       ROUND(SUM(s.AMOUNT_SOLD) / SUM(SUM(s.AMOUNT_SOLD)) OVER () * 100, 1) AS 構成比率
FROM SH.SALES s
JOIN SH.PRODUCTS p ON s.PROD_ID = p.PROD_ID
GROUP BY p.PROD_CATEGORY
ORDER BY 売上合計 DESC
```

3. 結果をMarkdownテーブルで表示する
   - 順位列を先頭に付ける
   - 売上構成比（%）を含める
   - 金額はカンマ区切り

4. 結果の特徴をコメントする（上位集中度、シェアの特徴など）

## 出力例

```
## 商品別売上ランキング TOP10（全期間）

| 順位 | 商品名 | 売上合計 | 販売数量 | 構成比 |
|------|-------|---------|---------|-------|
| 1 | Product A | 2,345,678 | 8,901 | 12.3% |
| 2 | Product B | 1,987,654 | 7,234 | 10.4% |
...

**ポイント:** 上位3商品でシェアの30%以上を占める集中型の販売構造。
```
