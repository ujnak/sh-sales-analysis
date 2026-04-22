---
name: trend
description: >
  売上の時系列変化・前月比・前年比などトレンドを分析して表示する。
  ユーザーが「売上の推移を見せて」「前月比は？」「成長率を教えて」「季節性はある？」
  「前年比を出して」「トレンドを分析して」などと言ったときに使用する。
  Oracle SHスキーマのSH.SALESとSH.TIMESを活用する。
user-invocable: true
argument-hint: "[比較軸: 前月比|前年比|累計] [期間] [カテゴリ]"
---

# トレンド分析スキル

売上の時系列変化をOracle SHスキーマから取得し、前月比・前年比・成長率などをチャットに表示する。

## 実行手順

1. ユーザーのメッセージからトレンド分析条件を解釈する
   - 比較軸: 前月比（月次）、前年比（年次）、累計（年度累計）
   - 期間フィルタ
   - 商品カテゴリや顧客セグメントでの絞り込み

2. 分析種別に応じてSQLを組み立てて `run_sql` ツールで実行する

**前月比トレンド（例）:**
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

**前年比トレンド（例）:**
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

3. 結果をMarkdownテーブルで表示する
   - 変化率をプラス/マイナスで示す（+12.3% / -5.6%）
   - 前月比が大きく変動した月をハイライトしてコメント

4. トレンドの特徴をまとめる（成長傾向、季節性、特異点など）

## 出力例

```
## 月次売上トレンド（前月比）

| 年月 | 売上合計 | 前月比 |
|------|---------|-------|
| 2001-01 | 1,234,567 | — |
| 2001-02 | 987,654 | -20.0% |
| 2001-03 | 1,456,789 | +47.5% |
...

**ポイント:** 3月・9月に急上昇するパターンが毎年見られる。季節的な需要変動の可能性あり。
```
