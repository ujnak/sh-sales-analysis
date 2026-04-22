---
name: custom-analysis
description: >
  ユーザーが自由な条件を指定して柔軟に販売データを分析する。
  「〇〇を調べて」「SQLを実行して」「詳しく分析して」「チャネルごとの商品別売上は？」
  「収益性を見せて」「プロモーション効果は？」「都市別の売上は？」
  など、既存スキルのパターンに当てはまらない自由な分析リクエストに使用する。
user-invocable: true
argument-hint: "[分析したい内容を自由に記述]"
---

# カスタム分析スキル

ユーザーが自由な質問や条件を指定し、最適なSQLを組み立ててOracle SHスキーマを分析する。

## 実行手順

1. ユーザーの質問を理解し、必要なテーブルと結合条件を特定する

   利用可能なテーブル（SHスキーマ）:
   - `SH.SALES` — PROD_ID, CUST_ID, TIME_ID, CHANNEL_ID, PROMO_ID, QUANTITY_SOLD, AMOUNT_SOLD
   - `SH.PRODUCTS` — PROD_ID, PROD_NAME, PROD_CATEGORY, PROD_SUBCATEGORY, PROD_LIST_PRICE, PROD_STATUS
   - `SH.CUSTOMERS` — CUST_ID, CUST_FIRST_NAME, CUST_LAST_NAME, CUST_CITY, CUST_STATE_PROVINCE, CUST_INCOME_LEVEL
   - `SH.TIMES` — TIME_ID, CALENDAR_MONTH_DESC, CALENDAR_MONTH_NUMBER
   - `SH.CHANNELS` — CHANNEL_ID, CHANNEL_DESC, CHANNEL_CLASS
   - `SH.COSTS` — PROD_ID, TIME_ID, CHANNEL_ID, PROMO_ID, UNIT_COST, UNIT_PRICE
   - `SH.PROMOTIONS` — PROMO_ID, PROMO_NAME, PROMO_CATEGORY

   スキーマが不明な場合は `get_schema` ツールで確認してから実行する。

2. 最適なSQLを構築して `run_sql` ツールで実行する
   - 分析の意図を理解して適切な集計・結合・フィルタを選択
   - 結果が多い場合はROWNUM等で適切に件数を制限
   - 複数のクエリが必要な場合は順番に実行

3. 結果をMarkdownテーブルまたは箇条書きで分かりやすく表示する

4. 分析結果から洞察をコメントする

## 分析アイデア例

- 「チャネル×カテゴリのクロス集計を見せて」
- 「収益性が高い商品（AMOUNT_SOLD - UNIT_COST）を教えて」（SH.COSTSを使用）
- 「プロモーション効果を比較して」（SH.PROMOTIONSを使用）
- 「都市別の売上分布は？」（SH.CUSTOMERSを使用）
- 「特定商品の販売推移をチャネル別に見せて」
- 「所得レベル別の顧客の購買行動の違いは？」

## 注意事項

- 質問の意図が不明な場合は、どの観点で分析するか確認してから実行する
- 実行したSQLをユーザーに見せることで透明性を確保する（必要に応じて）
- エラーが発生した場合はスキーマを確認して修正する
