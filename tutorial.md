# はじめてのBigQueryハンズオン

## ハンズオンの概要

あなたは、東京と横浜に店舗を持つ小売チェーン Next Drag の CS 担当者です。

Next Drag は、顧客満足度を向上させるために、店舗に来店した顧客からアンケートを収集しています。しかし、アンケートの回答は CSV 形式で保存されており、そのデータ量は膨大です。

あなたは、この大量のアンケートデータを効率的に分析し、具体的なサービス改善につなげたいと考えています。

そのために、あなたは BigQuery と Gemini を活用することで、これらの課題を解決できると考えました。

### このラボの内容
* CSV 形式の店舗データ、アンケートデータを BigQuery にインポートする
* Gemini in BigQuery を用いてデータを理解する
* BigQueryML と Gemini を用いてアンケートデータを分析する
* DataCanvas を用いてクイックにデータを可視化する
* Looker Studio を用いてダッシュボードを作成する


## ハンズオンの開始
<walkthrough-tutorial-duration duration=5></walkthrough-tutorial-duration>
まずはハンズオンに利用するファイルをダウンロードします。
<walkthrough-info-message>既に実施済の手順はスキップできます。</walkthrough-info-message>

Cloud Shell 
<walkthrough-cloud-shell-icon></walkthrough-cloud-shell-icon> を開き、次のコマンドを実行します。

### 1. ハンズオン資材をダウンロードする
```bash
git clone https://github.com/tichiba/next24-handson.git
```

### 2. ハンズオン資材があるディレクトリに移動する
```bash
cd ~/next24-handson
```

### 3. チュートリアルを開く
```bash
teachme tutorial.md
```

## Google Cloud プロジェクトの設定
次に、ターミナルの環境変数にプロジェクトIDを設定します。
```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
```
<walkthrough-info-message>**Tips:** コードボックスの横にあるボタンをクリックすることで、クリップボードへのコピーおよび Cloud Shell へのコピーが簡単に行えます。</walkthrough-info-message>

次に、このハンズオンで利用するAPIを有効化します。

<walkthrough-enable-apis apis=
  "bigquery.googleapis.com, cloudaicompanion.googleapis.com, aiplatform.googleapis.com, dataplex.googleapis.com">
</walkthrough-enable-apis>

## **[シナリオ1] CSV 形式の店舗データ、アンケートデータを BigQuery にインポートする**

<walkthrough-tutorial-duration duration=15></walkthrough-tutorial-duration>

あなたはまず、Next Dragの各店舗から送られてきたCSV形式のデータを扱います。

店舗データには店舗名、住所などが、売上データには商品ID、売上金額、販売日時などが、アンケートデータには店舗IDと定性的なコメントが含まれています。

これらのデータを手作業で分析するのは困難なため、あなたはBigQueryにデータをインポートすることにしました。BigQueryのインターフェースを使ってそれぞれのCSVファイルをアップロードし、分析の準備を整えます。

## GCS バケットの作成とファイルのアップロード

1. ファイルのアップロード先となる GCS バケットを作成します。
```bash
gcloud storage buckets create gs://${PROJECT_ID}_bigquery_handson --project=$PROJECT_ID --location=asia-northeast1
```

2. 作成した GCS バケットにハンズオン資材の CSV をアップロードします。
```bash
gcloud storage cp store.csv order.csv order_items.csv customer_voice.csv gs://${PROJECT_ID}_bigquery_handson
```

ハンズオンに必要な CSV データを GCS バケットにアップロードすることができました。

## BigQuery の Dataset 準備
ここからはより直感的に理解しやすいよう Cloud Console 上で操作を行います。

まずは BigQuery の Dataset を作成します。

1. ナビゲーションメニュー <walkthrough-nav-menu-icon></walkthrough-nav-menu-icon> から [**BigQuery**] に移動します。
<walkthrough-menu-navigation sectionId="BIGQUERY_SECTION"></walkthrough-menu-navigation>
2. エクスプローラペインの **自身の プロジェクト ID** の右側に表示されている
<walkthrough-spotlight-pointer cssSelector="[instrumentationid=bq-dataset-explorer-resource-list] button.node-context-menu" single="true">**︙ (三点リーダー)**</walkthrough-spotlight-pointer> をクリックし、[**データセットを作成**] を選択します。
3. [**データセットを作成する**] ペインで下記の情報を入力します。

  フィールド  | 値
  ------- | --------
  データセット ID | `next_drag`
  ロケーションタイプ | リージョン
  データのロケーション | `asia-northeast1`

4. [**データセットを作成**] をクリックします。
5. エクスプローラペインの自身のプロジェクト ID の下に、データセット `next_drag` が作成されていることを確認します。

## Store テーブルの作成
次に、作成した Dataset に新しいテーブルを作成します。

1. エクスプローラーペインで `next_drag` の横にある **︙** をクリックし、続いて [**テーブルを作成**] をクリックします。
2. [**ソース**] の [**テーブルの作成元**] に [**Google Cloud Storage**] を選択します。
3. [**参照**] をクリックして、Cloud Storage から [**store.csv**] ファイルを選択します。
4. [**送信先**] の [**テーブル**] の名前に `store` を入力します。
5. [**スキーマ**] > [**自動検出**] のチェックをオンにして、[**テーブルを作成**] をクリックします。

作成したテーブルのデータをプレビューで確認します。

1. エクスプローラーペインから **プロジェクト ID** > `next_drag` > `store` テーブルを選択します。
2. <walkthrough-spotlight-pointer cssSelector="[instrumentationid=bq-table-preview-tab]" single="true">[**プレビュー**]</walkthrough-spotlight-pointer> をクリックします。

3つの店舗のデータが正しく取り込まれていることが分かります。

## その他のテーブルの作成
続けて、あと3つのテーブルを作成します。

### Order テーブル

1. エクスプローラーペインで `next_drag` の横にある **︙** をクリックし、続いて [**テーブルを作成**] をクリックします。
2. [**ソース**] の [**テーブルの作成元**] に [**Google Cloud Storage**] を選択します。
3. [**参照**] をクリックして、Cloud Storage から [**order.csv**] ファイルを選択します。
4. [**送信先**] の [**テーブル**] の名前に `order` を入力します。
5. [**スキーマ**] > [**自動検出**] のチェックをオンにして、[**テーブルを作成**] をクリックします。

### Order items テーブル

1. エクスプローラーペインで `next_drag` の横にある **︙** をクリックし、続いて [**テーブルを作成**] をクリックします。
2. [**ソース**] の [**テーブルの作成元**] に [**Google Cloud Storage**] を選択します。
3. [**参照**] をクリックして、Cloud Storage から [**order_items.csv**] ファイルを選択します。
4. [**送信先**] の [**テーブル**] の名前に `order_items` を入力します。
5. [**スキーマ**] > [**自動検出**] のチェックをオンにして、[**テーブルを作成**] をクリックします。

### Customer voice テーブル

1. エクスプローラーペインで `next_drag` の横にある **︙** をクリックし、続いて [**テーブルを作成**] をクリックします。
2. [**ソース**] の [**テーブルの作成元**] に [**Google Cloud Storage**] を選択します。
3. [**参照**] をクリックして、Cloud Storage から [**customer_voice.csv**] ファイルを選択します。
4. [**送信先**] の [**テーブル**] の名前に `customer_voice` を入力します。
5. [**スキーマ**] > [**自動検出**] のチェックをオンにして、[**テーブルを作成**] をクリックします。

BigQUery へ CSV データをインポートすることができました。

次に BigQuery のデータに対するクエリの実行方法を学びます。

## **[シナリオ2] Gemini in BigQuery を用いてデータを理解する**

<walkthrough-tutorial-duration duration=15></walkthrough-tutorial-duration>

膨大なデータの中身を確認するために、Gemini in BigQueryを活用します。「2024 年 7 月の各店舗の売上金額は?」といった質問を自然言語で投げかけると、Geminiが自動的にSQLクエリを生成し、結果を表示してくれます。これにより、データの全体像を素早く把握し、分析の方向性を定めることができます。

<!-- TODO:Data Insights が public になったら追加 -->

## BigQuery で簡単なクエリを実行

まずは、Gemini を使わずに SQL クエリを実行する方法を試します。

店舗ごとの売上を集計するクエリを実行します。

1. <walkthrough-spotlight-pointer cssSelector="[instrumentationid=bq-sql-code-editor] button[name=addTabButton]" single="true">[**SQL クエリを作成**] アイコン</walkthrough-spotlight-pointer> をクリックして、新しいタブを開きます。

2. 下記の SQL を入力し、[**実行**] をクリックして実行結果を確認します。
```SQL
SELECT 
  store,
  SUM(price) as sales_total
FROM `next_drag.sales_data`
GROUP BY store
```

## Gemini で SQL クエリを生成

次に、Gemini を用いて SQL クエリを生成します。

<!-- Geminiの有効化ができてなかったら、以下の手順を追加

まず、Gemini in BigQuery を有効化します。

1. クエリエディタの横にある <walkthrough-spotlight-pointer cssSelector="[instrumentationid=bq-sql-code-editor] button#_1rif_Gemini" single="true">[**Gemini**] アイコン</walkthrough-spotlight-pointer> にマウスカーソルを合わせ、表示されたツールチップの [**続行**] をクリックします。

2. [**Geminiを有効にする**]ペインで、Trusted Tester プログラムに関する利用規約への同意にチェックを入れ、[**次へ**] をクリックします。

3. [**Cloud AI Companion API**] が無効になっている場合は [**有効にする**] をクリックし、[**閉じる**] をクリックします。
-->

1. <walkthrough-spotlight-pointer cssSelector="[instrumentationid=bq-sql-code-editor] button[name=addTabButton]" single="true">[**SQL クエリを作成**] アイコン</walkthrough-spotlight-pointer> をクリックして、新しいタブを開きます。

2. <walkthrough-spotlight-pointer cssSelector="sqe-duet-trigger-overlay" single="true">[**コーディングをサポート**]アイコン</walkthrough-spotlight-pointer> をクリックするか、Ctrl + Shift + P を入力して、[**コーディングをサポート**]ツールを開きます。

3. 次のプロンプトを入力します。
```
next_drag データセットから、販売金額トップ 10 の商品とそのカテゴリを調べるクエリを書いて
```

4. [**生成**] をクリックします。
  Gemini は、次のような SQL クエリを生成します。
```terminal
-- next_drag データセットから、販売金額トップ 10 の商品とそのカテゴリを調べるクエリを書いて
SELECT
    oi.item,
    oi.category,
    SUM(oi.total_price) AS total_sales
  FROM
    `bq-handson-427902.next_drag.order_items` AS oi
  GROUP BY 1, 2
ORDER BY
  total_sales DESC
LIMIT 10;
```

<walkthrough-info-message>**注:** Gemini は、同じプロンプトに対して異なる SQL クエリを提案する場合があります。必要に応じてクエリを修正してください。</walkthrough-info-message>

5. 生成された SQL クエリを受け入れるには、[**挿入**] をクリックして、クエリエディタにステートメントを挿入します。[**実行**] をクリックして、提案された SQL クエリを実行します。

3. 実行したクエリを保存して、チームへの共有や次回に再利用することができます。 [**保存**] をクリックし、続いて [**クエリを保存**] をクリックします。

4. [**名前**] に `販売トップ10` と入力し、[**保存**] をクリックします。

5. 保存されたクエリはエクスプローラペインの **プロジェクト ID** > [**クエリ**] の下で確認ができます。

## BigQueryでクエリの定期実行を設定
次に、定期的にクエリを実行する スケジュールの作成をします。

1. エクスプローラーペインから **プロジェクト ID** > [**クエリ**] > `販売トップ10` を選択します。
2. クエリを以下のように修正し、[**クエリを保存**] をクリックします。1行目を追加し、実行結果を新しいテーブル `top10_items` に保存するようにしています。
```sql
CREATE OR REPLACE TABLE `next_drag.top10_items` AS
SELECT
    oi.item,
    oi.category,
    SUM(oi.total_price) AS total_sales
  FROM
    `bq-handson-427902.next_drag.order_items` AS oi
  GROUP BY 1, 2
ORDER BY
  total_sales DESC
LIMIT 10;
```

3. [**スケジュール**] をクリックします。
4. **新たにスケジュールされたクエリ** ペインで次のとおり入力します。

フィールド | 値
---------------- | ----------------
クエリの名前 | `販売トップ10`
繰り返しの頻度 | 日
時刻 | `01:00`
すぐに開始 | 選択する
ロケーションタイプ | リージョン
リージョン | `asia-northeast1`

5. 他はデフォルトのまま [**保存**] をクリックし、スケジュールを保存します。認証を求められた場合は、ハンズオン用のユーザーを選んで認証します。
6. すぐに開始 を選択したため、エクスプローラーペインの **プロジェクト ID** > `next_drag` の下に新しいテーブル `top10_items` が作成されていることが確認できます。
7. スケジュールされたクエリの実行結果を、ナビゲーションペインの **スケジュールされたクエリ**  から確認します。

BigQuery のデータに対するクエリの実行方法を学びました。

<!-- 
Gemini in BigQuery のアシスタント機能を学びました。これ以外のプロンプトも自由に試してみてください。
-->

## **[シナリオ3] BigQueryML と Gemini を用いてアンケートデータを分析する**

<walkthrough-tutorial-duration duration=15></walkthrough-tutorial-duration>

顧客満足度を向上させるためには、アンケートデータの詳細な分析が必要です。BigQueryMLとGeminiを組み合わせることで、アンケートの自由記述回答を分析し、顧客の不満や要望を抽出します。例えば、「品揃えが悪い」「レジ待ち時間が長い」といった不満や、「オーガニック商品の拡充」「セルフレジの導入」といった要望を特定することができます。

## BigQueryからVertex AIへの接続を作成
<walkthrough-tutorial-duration duration=15></walkthrough-tutorial-duration>
まず、BigQuery から Vertex AI への接続を作成します。

1. ナビゲーションメニュー <walkthrough-nav-menu-icon></walkthrough-nav-menu-icon> から [**BigQuery**] に移動します。
1. 接続を作成するには、エクスプローラペインの [**+データを追加**] をクリックし、続いて [**外部データソースへの接続**] をクリックします。
2. **外部データソース** ペインで、次の情報を入力します。

フィールド | 値
--- | ---
接続タイプ | **Vertex AI リモートモデル、リモート関数、BigLake（Cloud リソース）**
接続 ID | `gemini-connect`
ロケーションタイプ | **リージョン**
リージョン | `asia-northeast1`

4. [**接続を作成**] をクリックします。
5. [**接続へ移動**] をクリックします。
6. [**接続情報**] ペインで、次の手順で使用する **サービス アカウント ID** をコピーします。

## Vertex AIへの接続で用いるサービスアカウントにアクセス権限を付与

次に、作成した接続で用いるサービスアカウントにアクセス権限 (IAM) を付与します。

1. ナビゲーションメニュー <walkthrough-nav-menu-icon></walkthrough-nav-menu-icon> から [**IAMと管理**] > [**IAM**] に移動します。

2. [**アクセス権を付与**] をクリックします。

3. [**新しいプリンシパル**] に、前の手順でコピーしたアカウント ID を入力します。

4. [**ロールを選択**] をクリックし、[**Vertex AI**] > [**Vertex AI ユーザー**] を選択します。

5. [**保存**] をクリックし、アクセス権を付与します。

## BigQuery から生成 AI モデル Gemini へ接続
続いて、作成した接続を用いて BigQuery から生成 AI モデル Gemini へ接続します。

1. ナビゲーションメニュー <walkthrough-nav-menu-icon></walkthrough-nav-menu-icon> から [**BigQuery**] に移動します。

1. [**SQLクエリを作成**] をクリックして新しいタブを開き、以下の SQL を実行します。
```sql
CREATE OR REPLACE MODEL next_drag.gemini_model
  REMOTE WITH CONNECTION `asia-northeast1.gemini-connect`
  OPTIONS(ENDPOINT = 'gemini-1.5-flash')
```
ここでは、生成 AI モデルの Gemini 1.5 Flash を指定しました。

2. 同じタブで以下の SQL を実行し、Gemini からのレスポンスを確認します。
```sql
SELECT ml_generate_text_result as response
FROM ML.GENERATE_TEXT(
    MODEL next_drag.gemini_model,
    (SELECT 'Google Cloud Nextについて教えてください' AS prompt),
    STRUCT(1000 as max_output_tokens, 0.2 as temperature)
  )
```

Gemini からのレスポンスが json 型であることが分かります。
BigQuery では json 型のデータを次のようなクエリで展開することが可能です。
```sql
SELECT JSON_VALUE(ml_generate_text_result.candidates[0].content.parts[0].text) as response
FROM ML.GENERATE_TEXT(
    MODEL next_drag.gemini_model,
    (SELECT 'Google Cloud Nextについて教えてください' AS prompt),
    STRUCT(1000 as max_output_tokens, 0.2 as temperature)
  )
```

## Customer voice データを生成AIを用いて分析
Gemini を使ったアンケートデータの分析を行います。

1. [**クエリ**] > [**新しいタブ**] をクリックし、次の SQL を入力して [**実行**] をクリックします。 
```sql
CREATE OR REPLACE TABLE
  next_drag.customer_voice_category AS
SELECT
  customer_voice,
  store,
  JSON_VALUE(ml_generate_text_result.candidates[0].content.parts[0].text) AS category
FROM
  ML.GENERATE_TEXT( MODEL next_drag.gemini_model,
    (
    SELECT
      CONCAT( '次の[顧客の声]を分類して、[出力形式]に従って出力してください。¥n¥n', '[出力形式]¥n商品・品揃え、価格、スタッフ、店舗環境、その他、のいずれか1つをプレーンテキストで出力。余計な情報は付加しないこと。¥n¥n', '[顧客の声]¥n', customer_voice ) AS prompt,
      customer_voice,
      store
    FROM
      `next_drag.customer_voice` cv
    JOIN
      `next_drag.store` s
    ON
      cv.store_id = s.store_id ),
    STRUCT(1000 AS max_output_tokens,
      0.0 AS temperature) )
```
8. エクスプローラーペインで `next_drag` > `customer_voice_category` を選択し、[**プレビュー**] をクリックします。

生成AIモデル Gemini を用いた顧客の声データの分析ができました。

## **[シナリオ4] DataCanvas を用いてクイックにデータを可視化する**

<walkthrough-tutorial-duration duration=15></walkthrough-tutorial-duration>

分析結果を理解するために、DataCanvasを使ってデータを可視化します。店舗別の売上を比較する棒グラフ、顧客属性と満足度の関係を示す散布図などを簡単に作成できます。これらの視覚的な表現は、分析結果を直感的に理解するのに役立ち、改善策の検討を促進します。

## Data Canvas を用いたデータの探索
ここでは、Data Canvas を用いてクイックにデータを可視化する方法を学びます。

1. 
<walkthrough-spotlight-pointer cssSelector="[instrumentationid=bq-sql-code-editor] button[aria-label='その他の設定項目']" single="true">▼ボタン</walkthrough-spotlight-pointer>
をクリックしてドロップダウンメニューを開き [**データキャンバス**] を選択

リージョンを聞かれたら `asia-northeast1`を選択

2. `next_drag`と入力してデータを検索する 

3. `store` と `order` と `order_items` を選んで [**結合**] をクリック

4. プロンプトとして以下を入力して実行
```
各店舗の販売数量と売上金額を、店舗ごと日ごとカテゴリごとに集計
```

Gemini は、次のような SQL クエリを生成します。
```terminal
# prompt: 各店舗の販売数量と売上金額を、店舗ごと日ごとカテゴリごとに集計

SELECT
  t1.store,
  t3.category,
  EXTRACT(DATE
  FROM
    t2.order_timestamp) AS `order_date`,
  SUM(t3.quantity) AS quantity,
  SUM(t3.total_price) AS total_price
FROM
  `bq-handson-427902.next_drag.store` AS t1
INNER JOIN
  `bq-handson-427902.next_drag.order` AS t2
ON
  t1.store_id = t2.store_id
INNER JOIN
  `bq-handson-427902.next_drag.order_items` AS t3
ON
  t2.order_id = t3.order_id
GROUP BY
  1,
  2,
  3;
```

<walkthrough-info-message>**注:** Gemini は、同じプロンプトに対して異なる SQL クエリを提案する場合があります。必要に応じてクエリを修正してください。</walkthrough-info-message>

5. [**実行**] をクリックし、[**クエリ結果**]が表示されることを確認します。

## Data Canvas を用いたデータの可視化

Data Canvas には店舗・日・カテゴリごとの売上データが表示されています。

1. [**これらの結果に対してクエリを実行**] をクリックし、以下をプロンプトとして入力して実行します。
```
横浜店の売上金額を日単位で合計して、時系列順に並び替える
```

Gemini は、次のような SQL クエリを生成します。
```terminal
# prompt: 横浜店の売上金額を日単位で合計して、時系列順に並び替える

SELECT
  t1.order_date,
  SUM(t1.total_price) AS sum
FROM
  `bq-handson-427902._f9dbe54ada7edb15276eba6d6774d3a5a1c0124c.anon638d6706621c880c6d031593352a0d2c87c85428bb7c5197cb9c0de1558b52ef` AS t1
WHERE
  t1.store = '横浜店'
GROUP BY
  1
ORDER BY
  t1.order_date;
```

<walkthrough-info-message>**注:** Gemini は、同じプロンプトに対して異なる SQL クエリを提案する場合があります。必要に応じてクエリを修正してください。</walkthrough-info-message>

5. [**実行**] をクリックし、[**クエリ結果**] が表示されることを確認します。

6. [**可視化**] をクリックし、[**折れ線グラフの作成**] を選択します。

横浜店における売上金額の日単位の推移を表すグラフを作成することができました。

## **[シナリオ5] Looker Studio を用いてダッシュボードを作成する**

<walkthrough-tutorial-duration duration=15></walkthrough-tutorial-duration>

最後に、店舗マネージャーや経営陣がいつでもどこでも最新の分析結果を確認できるように、Looker Studio を使ってダッシュボードを共有します。ダッシュボードには、店舗別のKPI、顧客満足度の推移などが表示され、データに基づいた意思決定を支援します。

Data Canvasには、横浜店の売上金額の推移を表すグラフが表示されています。

1. グラフ右上の三点リーダーをクリックして [**Looker Studio にエクスポート**] をクリックします。

Looker Studio が開いて新しいダッシュボードが作成されます。

2. 画面左上の [**無題のレポート**] をクリックし、`横浜店レポート` と名前を変更します。

3. 画面右上の [**保存して共有**] をクリックし、ダイアログが開いたら [**同意して保存する**] を選択します。

このダッシュボードを必要な相手に共有することができるようになりました。


## Looker Studio に BigQuery のデータを追加

次に、Looker Studio に可視化したい BigQuery のデータを追加します。

1. Looker Studio のメニューバーから [**データを追加**] をクリックします。

2. [**データに接続**] セクションで [**BigQuery**] をクリックします。
アクセス権を求められた場合は承認してください。

3. マイプロジェクトからデータソースを次のとおり選択します。

フィールド | 値
---------------- | ----------------
プロジェクト | プロジェクトID
データセット | `next_drag`
表 | `customer_voice_category`

5. [**追加**] をクリックします。続いて [**レポートに追加**] をクリックします。

## Looker Studioにグラフを追加

続いて、追加したデータを元に Looker Studio にグラフを追加します。

1. メニューバーの[**グラフを追加**] > [**棒**] > [**棒グラフ**] をクリックします。
2. 追加されたグラフを選択した状態で、[**グラフ**] ペインでデータの設定をします。

フィールド | 値
---------------- | ----------------
データソース | `customer_voice_category`
ディメンション | `store`
内部ディメンション | `category`
指標 | (AUT)`Record Count`

3. 追加されたグラフをドラッグして任意の位置に移動します。

おめでとうございます！これで Looker Studio のダッシュボードを用いた BigQuery のデータの可視化は完了です。他のテーブルやグラフも自由に追加して試してみてください。


## Congratulations!
<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

おめでとうございます！ハンズオンはこれで完了です。ご参加ありがとうございました。