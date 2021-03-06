# Lesson 20. EC ログ分析クエリ

## デシル分析
デシル分析は，何らかの指標を10等分割し，それぞれの分割に入る人数や平均値を見比べる手法です。
### 人数で等分割
期間を固定して（例えば直近から2年間における）ユーザーごとの購入総額が大きい順に並べ，それを人数で等分割してみましょう。それぞれの分割ごとの平均購入額がどれだけ違うかを見てみます。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH stat AS
(
  SELECT SUM(two_years_sales) AS total_sales, COUNT(1) AS total_cnt
  FROM
  (
    SELECT member_id, SUM(price*amount) AS two_years_sales
    FROM sales_slip
    WHERE TD_INTERVAL(time, '-730d', 'JST')
    AND member_id IS NOT NULL
    GROUP BY member_id
  )
),
sales_table AS(
  SELECT member_id, SUM(price*amount) AS two_years_sales
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST')
  AND member_id IS NOT NULL
  GROUP BY member_id
), 
tile_table AS
(
  SELECT tile, COUNT(1) AS cnt, AVG(two_years_sales) AS sales_avg, 
    1.0*SUM(two_years_sales)/MAX(total_sales) AS sales_ratio
  FROM
  (
    SELECT NTILE(10)OVER(ORDER BY two_years_sales DESC) AS tile, member_id, two_years_sales
    FROM sales_table
  ), stat
  GROUP BY tile
)

SELECT tile, cnt, sales_avg, sales_ratio,
  SUM(sales_ratio)OVER(ORDER BY tile) AS cum_sales_ratio
FROM tile_table
ORDER BY tile
```
|tile      |cnt   |sales_avg               |sales_ratio         |cum_sales_ratio|
|----------|------|------------------------|--------------------|---------------|
|1         |868   |922889.3917050691       |0.3431727655309763  |0.3431727655309763|
|2         |868   |452626.7281105991       |0.1683074563810683  |0.5114802219120447|
|3         |867   |336523.27566320647      |0.12499068280414563 |0.6364709047161903|
|4         |867   |270205.9411764706       |0.10035925455326965 |0.7368301592694599|
|5         |867   |218811.01153402537      |0.08127027077935292 |0.8181004300488128|
|6         |867   |175734.354094579        |0.06527084008420783 |0.8833712701330206|
|7         |867   |136468.13610149943      |0.05068667383770041 |0.9340579439707211|
|8         |867   |98327.76931949251       |0.036520668597532065|0.9705786125682532|
|9         |867   |59429.34371395617       |0.022073106933749573|0.9926517195020027|
|10        |867   |19807.259815242494      |0.007348280497997277|1.0            |

購入額が一番多いタイル1と一番少ないタイル10を比較すれば，平均購入額92万円と2万円で大きな差が出ています。sales_ratioでは，そのタイルの人が全体の総購入額の何%を占めているかを計算しています。以下の円グラフはその内訳を表したものです。このグラフは，購入額の多いたった20%（タイル1と2）のユーザーで，全体の売上の半分以上を占めているという事実を表しています。

### 購入比率で等分割
全員の総購入額を均等に10分割し，購入額の大きい人から順に積み上げていくと，総購入額の10%までに何人が入るか，さらに次の20%までに何人が入るかを見ていきます。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH stat AS
(
  SELECT SUM(two_years_sales) AS total_sales, COUNT(1) AS total_cnt
  FROM
  (
    SELECT member_id, SUM(price*amount) AS two_years_sales
    FROM sales_slip
    WHERE TD_INTERVAL(time, '-730d', 'JST')
    AND member_id IS NOT NULL
    GROUP BY member_id
  )
),
sales_table AS
(
  SELECT member_id, SUM(price*amount) AS two_years_sales
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST')
  AND member_id IS NOT NULL
  GROUP BY member_id
),
tile_table AS
(
  SELECT 
    CEIL( 1.0*SUM(two_years_sales)OVER(ORDER BY two_years_sales DESC)/total_sales*10 ) AS tile, member_id, two_years_sales
  FROM sales_table, stat
)

SELECT tile, COUNT(1) AS cnt, AVG(two_years_sales) AS sales_avg, SUM(two_years_sales) AS sales_sum
FROM tile_table
GROUP BY tile
ORDER BY tile
```
|tile      |cnt   |sales_avg               |sales_sum           |
|----------|------|------------------------|--------------------|
|1.0       |112   |2078484.5267857143      |232790267           |
|2.0       |242   |966748.6694214876       |233953178           |
|3.0       |339   |688485.7374631269       |233396665           |
|4.0       |433   |538803.2424942263       |233301804           |
|5.0       |540   |432428.45185185183      |233511364           |
|6.0       |662   |352691.336858006        |233481665           |
|7.0       |804   |290399.51243781095      |233481208           |
|8.0       |994   |234838.8722334004       |233429839           |
|9.0       |1336  |174689.02844311378      |233384542           |
|10.0      |3210  |72785.57214085385       |233568901           |

この結果からは，ロイヤル（購入額の大きい）なユーザーが112人で全体の売上の10%を占めること（平均200万円層），次のロイヤルが241人であること（平均100万円層）がわかります。最も少額購入のユーザー層の3212人が束になって，やっと最上位の112人と同じ売上総額になります。

### 売上積み上げグラフ
もっと単純に，総購入額が多いユーザーからの積み上げグラフを作ってみるとわかりやすいかもしれません。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH stat AS
(
  SELECT SUM(two_years_sales) AS total_sales, COUNT(1) AS total_cnt
  FROM
  (
    SELECT member_id, SUM(price*amount) AS two_years_sales
    FROM sales_slip
    WHERE TD_INTERVAL(time, '-730d', 'JST')    
    AND member_id IS NOT NULL
    GROUP BY member_id
  )
),
sales_table AS
(
  SELECT member_id, SUM(price*amount) AS two_years_sales
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST')  
  AND member_id IS NOT NULL
  GROUP BY member_id
)

SELECT member_id, 
  RANK()OVER(ORDER BY two_years_sales DESC) AS rnk,
  two_years_sales, 
  SUM(two_years_sales)OVER(ORDER BY two_years_sales DESC) AS cum_sales,
  1.0*SUM(two_years_sales)OVER(ORDER BY two_years_sales DESC)/total_sales AS cum_ratio
FROM sales_table, stat
ORDER BY cum_ratio
```
|member_id |rnk   |two_years_sales         |cum_sales           |cum_ratio           |
|----------|------|------------------------|--------------------|--------------------|
|1385684   |1     |11323360                |11323360            |0.004850860108142775|
|1833816   |2     |7118372                 |18441732            |0.007900328355175503|
|647431    |3     |7084217                 |25525949            |0.010935164803255128|
|1916325   |4     |6271414                 |31797363            |0.013621801278139624|
|1111523   |5     |5426092                 |37223455            |0.01594630683355009 |


このような積み上げで見れば，全体の売上の25%がたった600人程度のロイヤルユーザーによって占められていることがよくわかります。もしこのグラフが直線に近ければロイヤルなユーザーは少なく，前半が急激に膨らんでいればいるほど，とても大きな購入額の少数ユーザーが存在するということが示唆されます。

## RFM分析
### Recency
Recencyは，ユーザーを直近の「購入日」によって区分する方法です。Recencyの意味でのユーザーの良さは，直近で購入のあったユーザーほど「高く」，何年も前に購入がさかのぼるユーザーほど「低い」と見ることができます。Recencyは，いわばユーザーの「新鮮度」で，Recencyの高いユーザーほど次のイベントへの感度や新製品の購入意欲が高いものと判断できます。

Recencyでは， Rから始まるもう1つの指標であるRegistration Day（登録日）も考慮することが重要です。はるか昔に登録してから直近も購入してくれている息の長いユーザーは，最近登録したユーザーよりも貴重だと考えられます。直近登録のユーザーが，より時間が経ったときも常にRecencyが高くいられるかはわかりません。そういう意味で，同時に登録日も求めておくことが非常に有用です。

「直近」を考える際，現在進行系で蓄積されているデータに対しては今日または昨日でかまわないのですが，もっと過去のデータでは「直近」がいつかを定める必要があります。これまで何度か登場しているTD_SCHEDULED_TIME()関数を使うことで，実行ごとに「直近」の値を定めることができます。

以下のクエリでは，recent_daysはユーザーごとの（直近以前からの）最新購入日（最新を1として何日前か），registration_daysは一番最初の購入日（初購入の意味での登録日が何日前か）としています。sales_periodは購入期間を意味し，これらの差を意味します。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH recency_inner_table AS
(
  SELECT member_id, 
    FLOOR(1.0*(TD_SCHEDULED_TIME()-MAX(time))/(60*60*24))+1 AS recent_days,
    FLOOR(1.0*(TD_SCHEDULED_TIME()-MIN(time))/(60*60*24))+1 AS registration_days
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, NULL, TD_DATE_TRUNC('day', TD_SCHEDULED_TIME(), 'JST'), 'JST')
  AND member_id IS NOT NULL
  GROUP BY member_id
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, 
  member_id,
  RANK()OVER(ORDER BY recent_days ASC, registration_days DESC) AS rnk,
  recent_days,
  registration_days, 
  registration_days-recent_days+1 AS sales_period
FROM recency_inner_table
ORDER BY recent_days ASC, registration_days DESC
```
|d         |member_id|rnk                     |recent_days         |registration_days   |sales_period|
|----------|---------|------------------------|--------------------|--------------------|------------|
|2013-12-19|433014   |1                       |1.0                 |3282.0              |3282.0      |
|2013-12-19|114543   |1                       |1.0                 |3282.0              |3282.0      |
|2013-12-19|70031    |3                       |1.0                 |3281.0              |3281.0      |
|2013-12-19|46514    |4                       |1.0                 |3280.0              |3280.0      |
|2013-12-19|517094   |5                       |1.0                 |3279.0              |3279.0      |


### Frequency
どれくらい頻繁に購入してくれているかをFrequencyとして，頻度が高いユーザーほど良いと考えます。Frequencyは，いわば「常連度」で，そのECサイトへの愛情を測る度数となります。
Frequencyでは，ユーザーの過去すべて（ライフタイム）における以下の値を考えます。
- 総購入回数
- それを会員期間でならした月間平均購入回数
ただ，期間を固定して例えば「直近から2年間での総購入回数」を考えれば，平均を考える必要はありません。
以下のクエリでは，期間固定（直近2年間）の総購入回数を求めています。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH frequency_inner_table AS
(
  SELECT member_id, 
    COUNT(1) AS two_years_cnt
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST')  
  AND member_id IS NOT NULL
  GROUP BY member_id
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, 
  RANK()OVER(ORDER BY two_years_cnt DESC) AS rnk,
  member_id,
  two_years_cnt
FROM frequency_inner_table
ORDER BY two_years_cnt DESC
```
|d         |rnk   |member_id               |two_years_cnt       |
|----------|------|------------------------|--------------------|
|2013-12-19|1     |2259091                 |326186              |
|2013-12-19|2     |1833816                 |2295                |
|2013-12-19|3     |1963430                 |1575                |
|2013-12-19|4     |1916325                 |1171                |
|2013-12-19|5     |777155                  |1068                |


### Monetary
購入総額を示すMonetaryは，それが高いユーザーほど良いと考えられる，非常にダイレクトな指標です。Monetaryは，いわばECサイト運営サイドへの「貢献度」であり，これが高いユーザーは決して邪険にしてはいけません。購入総額上位の数%のユーザーだけで，全体の購入の20〜50%を占めるようなことも稀ではありません。そういったユーザーは「クジラ（ホエール）」と呼ばれます。
Monetaryでは，ユーザーの過去すべて（ライフタイム）における以下の値を考えます。
- 購入総額
- それを会員期間でならした月間平均購入額
ただ，期間を固定して例えば「直近から2年間での購入総額」を考えれば，平均を考える必要はありません。以下のクエリでは期間固定のMonetaryを求めています。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH monetary_table AS
(
  SELECT member_id, 
    SUM(price*amount) AS two_years_sales
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST') 
  AND member_id IS NOT NULL
  GROUP BY member_id
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, 
  RANK()OVER(ORDER BY two_years_sales DESC) AS rnk,
  member_id,
  two_years_sales
FROM monetary_table
ORDER BY two_years_sales DESC
```
|d         |rnk   |member_id               |two_years_sales     |
|----------|------|------------------------|--------------------|
|2013-12-19|1     |1385684                 |11323360            |
|2013-12-19|2     |1833816                 |7118372             |
|2013-12-19|3     |647431                  |7084217             |
|2013-12-19|4     |1916325                 |6271414             |
|2013-12-19|5     |1111523                 |5426092             |


### RFM 
下記は，先程まで別個で求めていたRFMをユーザーごとに同時に求めるクエリです。FrequencyとMonetaryの指標では，期間固定2年間の集計値を扱います。
また，この結果を「rfm_result」として一度テーブルに書き込んで置きましょう。
```sql
DROP TABLE IF EXISTS rfm_result;
CREATE TABLE rfm_result AS
--TD_SCHEDULED_TIME()=2013-12-19
WITH recency_inner_table AS
(
  SELECT member_id, 
    FLOOR(1.0*(TD_SCHEDULED_TIME()-MAX(time))/(60*60*24))+1 AS recent_days,
    FLOOR(1.0*(TD_SCHEDULED_TIME()-MIN(time))/(60*60*24))+1 AS registration_days
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, NULL, TD_DATE_TRUNC('day', TD_SCHEDULED_TIME(), 'JST'), 'JST')
  AND member_id IS NOT NULL
  GROUP BY member_id
),
recency_table AS
(
  SELECT member_id,
    RANK()OVER(ORDER BY recent_days ASC, registration_days DESC) AS rnk_recency,
    recent_days,
    registration_days, 
    registration_days-recent_days+1 AS sales_period
  FROM recency_inner_table
),
frequency_inner_table AS
(
  SELECT member_id, COUNT(1) AS two_years_cnt
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST') 
  AND member_id IS NOT NULL
  GROUP BY member_id
),
frequency_table AS
(
  SELECT RANK()OVER(ORDER BY two_years_cnt DESC) AS rnk_frequency, member_id, two_years_cnt
  FROM frequency_inner_table
),
monetary_inner_table AS
(
  SELECT member_id, SUM(price*amount) AS two_years_sales
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST') 
  AND member_id IS NOT NULL
  GROUP BY member_id
),
monetary_table AS
(
  SELECT RANK()OVER(ORDER BY two_years_sales DESC) AS rnk_monetary, member_id, two_years_sales
  FROM monetary_inner_table
)

SELECT 
  TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, 
  r.member_id, registration_days, sales_period,
  recent_days     AS recency,
  two_years_cnt   AS frequency,
  two_years_sales AS monetary,
  rnk_recency, rnk_frequency, rnk_monetary
FROM recency_table r 
JOIN frequency_table f
ON r.member_id = f.member_id
JOIN monetary_table m
ON r.member_id = m.member_id
```
|d         |member_id|registration_days       |sales_period        |recency|frequency|monetary|rnk_recency|rnk_frequency|rnk_monetary|
|----------|---------|------------------------|--------------------|-------|---------|--------|-----------|-------------|------------|
|2013-12-19|433014   |3282.0                  |3282.0              |1.0    |192      |497966  |1          |1111         |1056        |
|2013-12-19|114543   |3282.0                  |3282.0              |1.0    |218      |478996  |1          |827          |1138        |
|2013-12-19|70031    |3281.0                  |3281.0              |1.0    |162      |811419  |3          |1560         |351         |
|2013-12-19|46514    |3280.0                  |3280.0              |1.0    |71       |156759  |4          |4752         |5199        |
|2013-12-19|517094   |3279.0                  |3279.0              |1.0    |750      |1639143 |5          |20           |50          |


さて，各々のユーザーのRFMの値を同時に求めるだけならば，上記のクエリでおしまいです。しかしながら，RFM分析の本質は，RFMそれぞれを適切なグループに割り振ることにあります。RFM分析では，そのうち2つを縦横にしたクロス表を見ていきます。そのためのグルーピングとして最も単純な方法は，人間にとってわかりやすい区分を使うものです。

```sql
WITH recency_segment AS
(
  SELECT member_id,
    CASE 
      WHEN recency <=   7 THEN '1. 1week'
      WHEN recency <=  14 THEN '2. 2week'
      WHEN recency <=  30 THEN '3. 1month'
      WHEN recency <=  90 THEN '4. 3month'
      WHEN recency <= 365 THEN '5. 1year'
      WHEN 365 < recency  THEN '6. <1year'
      ELSE 'error'
    END AS seg_recency
  FROM rfm_result
),
frequency_segment AS
(
  SELECT member_id,
    CASE --(times*week)*(month)*(year)
      WHEN (3*4)*12*2 <= frequency THEN '1. 3/week'
      WHEN (2*4)*12*2 <= frequency THEN '2. 2/week'
      WHEN (1*4)*12*2 <= frequency THEN '3. 1/week'
      WHEN (2)  *12*2 <= frequency THEN '4. 2/month'
      WHEN (1)  *12*2 <= frequency THEN '5. 1/month'
      WHEN frequency < 12*2        THEN '6. 1/year'
      ELSE 'error'
    END AS seg_frequency
  FROM rfm_result
),
monetary_segment AS
(
  SELECT member_id,
    CASE --(money)*(month)*(year)
      WHEN ( 15000)*12*2 <= monetary THEN  '1. 15000/month'
      WHEN ( 10000)*12*2 <= monetary THEN  '2. 10000/month'
      WHEN (  5000)*12*2 <= monetary THEN  '3. 5000/month'
      WHEN (  2500)*12*2 <= monetary THEN  '4. 1000/month'
      WHEN (  1000)*12*2 <= monetary THEN  '5. 1000/month'
      WHEN monetary < (  1000)*12*2  THEN  '6. <1000/month'
      ELSE 'error'
    END AS seg_monetary
  FROM rfm_result
)
```
以降では，上記のような人数区分のグルーピングを行います。

### RFマトリクス
下記のクエリはRFマトリクス（RとFのクロステーブル）を求めるものです。RMマトリクス，FM マトリクス，RFM マトリクスに必要なテーブルも掲載しています。
```sql
WITH recency_segment AS
(
  SELECT member_id,
    CASE 
      WHEN recency <=   7 THEN '1. 1week'
      WHEN recency <=  14 THEN '2. 2week'
      WHEN recency <=  30 THEN '3. 1month'
      WHEN recency <=  90 THEN '4. 3month'
      WHEN recency <= 365 THEN '5. 1year'
      WHEN 365 < recency  THEN '6. <1year'
      ELSE 'error'
    END AS seg_recency
  FROM rfm_result
),
frequency_segment AS
(
  SELECT member_id,
    CASE --(times*week)*(month)*(year)
      WHEN (3*4)*12*2 <= frequency THEN '1. 3/week'
      WHEN (2*4)*12*2 <= frequency THEN '2. 2/week'
      WHEN (1*4)*12*2 <= frequency THEN '3. 1/week'
      WHEN (2)  *12*2 <= frequency THEN '4. 2/month'
      WHEN (1)  *12*2 <= frequency THEN '5. 1/month'
      WHEN frequency < 12*2        THEN '6. 1/year'
      ELSE 'error'
    END AS seg_frequency
  FROM rfm_result
),
monetary_segment AS
(
  SELECT member_id,
    CASE --(money)*(month)*(year)
      WHEN ( 15000)*12*2 <= monetary THEN  '1. 15000/month'
      WHEN ( 10000)*12*2 <= monetary THEN  '2. 10000/month'
      WHEN (  5000)*12*2 <= monetary THEN  '3. 5000/month'
      WHEN (  2500)*12*2 <= monetary THEN  '4. 2500/month'
      WHEN (  1000)*12*2 <= monetary THEN  '5. 1000/month'
      WHEN monetary < (  1000)*12*2  THEN  '6. <1000/month'
      ELSE 'error'
    END AS seg_monetary
  FROM rfm_result
),
rf_table AS
(
  SELECT seg_recency, seg_frequency, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_frequency
    FROM recency_segment, frequency_segment
    WHERE recency_segment.member_id = frequency_segment.member_id
  )
  GROUP BY seg_recency, seg_frequency
),
rm_table AS
(
  SELECT seg_recency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_monetary
    FROM recency_segment, monetary_segment
    WHERE recency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_recency, seg_monetary
),
fm_table AS
(
  SELECT seg_frequency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT frequency_segment.member_id, seg_frequency, seg_monetary
    FROM frequency_segment, monetary_segment
    WHERE frequency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_frequency, seg_monetary
),
rfm_table AS
(
  SELECT seg_recency,seg_frequency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_frequency, seg_monetary
    FROM recency_segment, frequency_segment, monetary_segment
    WHERE recency_segment.member_id = monetary_segment.member_id
    AND frequency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_recency, seg_frequency, seg_monetary
)

SELECT seg_recency,
  IF(ELEMENT_AT(kv,'1. 3/week' ) IS NOT NULL, kv['1. 3/week'],  0) AS c1_3week,
  IF(ELEMENT_AT(kv,'2. 2/week' ) IS NOT NULL, kv['2. 2/week'],  0) AS c2_2week,
  IF(ELEMENT_AT(kv,'3. 1/week' ) IS NOT NULL, kv['3. 1/week'],  0) AS c3_1week,
  IF(ELEMENT_AT(kv,'4. 2/month') IS NOT NULL, kv['4. 2/month'], 0) AS c4_2month,
  IF(ELEMENT_AT(kv,'5. 1/month') IS NOT NULL, kv['5. 1/month'], 0) AS c5_1month,
  IF(ELEMENT_AT(kv,'6. 1/year' ) IS NOT NULL, kv['6. 1/year'],  0) AS c6_1year
FROM
(
  SELECT seg_recency, MAP_AGG(seg_frequency, cnt) AS kv
  FROM rf_table
  GROUP BY seg_recency
)
ORDER BY seg_recency
```
|seg_recency|c1_3week|c2_2week                |c3_1week            |c4_2month|c5_1month|c6_1year|
|-----------|--------|------------------------|--------------------|---------|---------|--------|
|1. 1week   |314     |439                     |1067                |584      |153      |48      |
|2. 2week   |38      |81                      |363                 |277      |84       |37      |
|3. 1month  |40      |70                      |446                 |449      |160      |63      |
|4. 3month  |18      |58                      |363                 |571      |312      |139     |
|5. 1year   |19      |39                      |186                 |487      |456      |451     |
|6. <1year  |2       |3                       |39                  |81       |154      |581     |


### RMマトリクス
```sql
WITH recency_segment AS
(
  SELECT member_id,
    CASE 
      WHEN recency <=   7 THEN '1. 1week'
      WHEN recency <=  14 THEN '2. 2week'
      WHEN recency <=  30 THEN '3. 1month'
      WHEN recency <=  90 THEN '4. 3month'
      WHEN recency <= 365 THEN '5. 1year'
      WHEN 365 < recency  THEN '6. <1year'
      ELSE 'error'
    END AS seg_recency
  FROM rfm_result
),
frequency_segment AS
(
  SELECT member_id,
    CASE --(times*week)*(month)*(year)
      WHEN (3*4)*12*2 <= frequency THEN '1. 3/week'
      WHEN (2*4)*12*2 <= frequency THEN '2. 2/week'
      WHEN (1*4)*12*2 <= frequency THEN '3. 1/week'
      WHEN (2)  *12*2 <= frequency THEN '4. 2/month'
      WHEN (1)  *12*2 <= frequency THEN '5. 1/month'
      WHEN frequency < 12*2        THEN '6. 1/year'
      ELSE 'error'
    END AS seg_frequency
  FROM rfm_result
),
monetary_segment AS
(
  SELECT member_id,
    CASE --(money)*(month)*(year)
      WHEN ( 15000)*12*2 <= monetary THEN  '1. 15000/month'
      WHEN ( 10000)*12*2 <= monetary THEN  '2. 10000/month'
      WHEN (  5000)*12*2 <= monetary THEN  '3. 5000/month'
      WHEN (  2500)*12*2 <= monetary THEN  '4. 2500/month'
      WHEN (  1000)*12*2 <= monetary THEN  '5. 1000/month'
      WHEN monetary < (  1000)*12*2  THEN  '6. <1000/month'
      ELSE 'error'
    END AS seg_monetary
  FROM rfm_result
),
rf_table AS
(
  SELECT seg_recency, seg_frequency, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_frequency
    FROM recency_segment, frequency_segment
    WHERE recency_segment.member_id = frequency_segment.member_id
  )
  GROUP BY seg_recency, seg_frequency
),
rm_table AS
(
  SELECT seg_recency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_monetary
    FROM recency_segment, monetary_segment
    WHERE recency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_recency, seg_monetary
),
fm_table AS
(
  SELECT seg_frequency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT frequency_segment.member_id, seg_frequency, seg_monetary
    FROM frequency_segment, monetary_segment
    WHERE frequency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_frequency, seg_monetary
),
rfm_table AS
(
  SELECT seg_recency,seg_frequency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_frequency, seg_monetary
    FROM recency_segment, frequency_segment, monetary_segment
    WHERE recency_segment.member_id = monetary_segment.member_id
    AND frequency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_recency, seg_frequency, seg_monetary
)

SELECT seg_recency,
  IF(ELEMENT_AT(kv,'1. 15000/month') IS NOT NULL, kv['1. 15000/month'], 0) AS c1_15000,
  IF(ELEMENT_AT(kv,'2. 10000/month') IS NOT NULL, kv['2. 10000/month'], 0) AS c2_10000,
  IF(ELEMENT_AT(kv,'3. 5000/month')  IS NOT NULL, kv['3. 5000/month'],  0) AS c3_5000,
  IF(ELEMENT_AT(kv,'4. 2500/month')  IS NOT NULL, kv['4. 2500/month'],  0) AS c4_2500,
  IF(ELEMENT_AT(kv,'5. 1000/month')  IS NOT NULL, kv['5. 1000/month'],  0) AS c5_1000,
  IF(ELEMENT_AT(kv,'6. <1000/month') IS NOT NULL, kv['6. <1000/month'], 0) AS c6_0
FROM
(
  SELECT seg_recency, MAP_AGG(seg_monetary, cnt) AS kv
  FROM rm_table
  GROUP BY seg_recency
)
ORDER BY seg_recency
```
|seg_recency|c1_15000|c2_10000                |c3_5000             |c4_2500|c5_1000|c6_0   |
|-----------|--------|------------------------|--------------------|-------|-------|-------|
|1. 1week   |1081    |665                     |641                 |167    |40     |10     |
|2. 2week   |224     |240                     |297                 |84     |28     |7      |
|3. 1month  |278     |263                     |462                 |166    |49     |10     |
|4. 3month  |203     |257                     |558                 |300    |103    |40     |
|5. 1year   |107     |170                     |438                 |474    |315    |134    |
|6. <1year  |29      |29                      |84                  |153    |238    |327    |

### FMマトリクス
```sql
WITH recency_segment AS
(
  SELECT member_id,
    CASE 
      WHEN recency <=   7 THEN '1. 1week'
      WHEN recency <=  14 THEN '2. 2week'
      WHEN recency <=  30 THEN '3. 1month'
      WHEN recency <=  90 THEN '4. 3month'
      WHEN recency <= 365 THEN '5. 1year'
      WHEN 365 < recency  THEN '6. <1year'
      ELSE 'error'
    END AS seg_recency
  FROM rfm_result
),
frequency_segment AS
(
  SELECT member_id,
    CASE --(times*week)*(month)*(year)
      WHEN (3*4)*12*2 <= frequency THEN '1. 3/week'
      WHEN (2*4)*12*2 <= frequency THEN '2. 2/week'
      WHEN (1*4)*12*2 <= frequency THEN '3. 1/week'
      WHEN (2)  *12*2 <= frequency THEN '4. 2/month'
      WHEN (1)  *12*2 <= frequency THEN '5. 1/month'
      WHEN frequency < 12*2        THEN '6. 1/year'
      ELSE 'error'
    END AS seg_frequency
  FROM rfm_result
),
monetary_segment AS
(
  SELECT member_id,
    CASE --(money)*(month)*(year)
      WHEN ( 15000)*12*2 <= monetary THEN  '1. 15000/month'
      WHEN ( 10000)*12*2 <= monetary THEN  '2. 10000/month'
      WHEN (  5000)*12*2 <= monetary THEN  '3. 5000/month'
      WHEN (  2500)*12*2 <= monetary THEN  '4. 2500/month'
      WHEN (  1000)*12*2 <= monetary THEN  '5. 1000/month'
      WHEN monetary < (  1000)*12*2  THEN  '6. <1000/month'
      ELSE 'error'
    END AS seg_monetary
  FROM rfm_result
),
rf_table AS
(
  SELECT seg_recency, seg_frequency, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_frequency
    FROM recency_segment, frequency_segment
    WHERE recency_segment.member_id = frequency_segment.member_id
  )
  GROUP BY seg_recency, seg_frequency
),
rm_table AS
(
  SELECT seg_recency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_monetary
    FROM recency_segment, monetary_segment
    WHERE recency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_recency, seg_monetary
),
fm_table AS
(
  SELECT seg_frequency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT frequency_segment.member_id, seg_frequency, seg_monetary
    FROM frequency_segment, monetary_segment
    WHERE frequency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_frequency, seg_monetary
),
rfm_table AS
(
  SELECT seg_recency,seg_frequency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_frequency, seg_monetary
    FROM recency_segment, frequency_segment, monetary_segment
    WHERE recency_segment.member_id = monetary_segment.member_id
    AND frequency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_recency, seg_frequency, seg_monetary
)

SELECT seg_frequency,
  IF(ELEMENT_AT(kv,'1. 15000/month') IS NOT NULL, kv['1. 15000/month'], 0) AS c1_15000,
  IF(ELEMENT_AT(kv,'2. 10000/month') IS NOT NULL, kv['2. 10000/month'], 0) AS c2_10000,
  IF(ELEMENT_AT(kv,'3. 5000/month')  IS NOT NULL, kv['3. 5000/month'],  0) AS c3_5000,
  IF(ELEMENT_AT(kv,'4. 2500/month')  IS NOT NULL, kv['4. 2500/month'],  0) AS c4_2500,
  IF(ELEMENT_AT(kv,'5. 1000/month')  IS NOT NULL, kv['5. 1000/month'],  0) AS c5_1000,
  IF(ELEMENT_AT(kv,'6. <1000/month') IS NOT NULL, kv['6. <1000/month'], 0) AS c6_0
FROM
(
  SELECT seg_frequency, MAP_AGG(seg_monetary, cnt) AS kv
  FROM fm_table
  GROUP BY seg_frequency
)
ORDER BY seg_frequency
```
|seg_frequency|c1_15000|c2_10000                |c3_5000             |c4_2500|c5_1000|c6_0   |
|-------------|--------|------------------------|--------------------|-------|-------|-------|
|1. 3/week    |414     |6                       |1                   |1      |0      |8      |
|2. 2/week    |585     |87                      |14                  |3      |1      |0      |
|3. 1/week    |836     |1051                    |554                 |20     |3      |0      |
|4. 2/month   |78      |457                     |1548                |357    |9      |0      |
|5. 1/month   |7       |19                      |343                 |778    |169    |3      |
|6. 1/year    |2       |4                       |20                  |185    |591    |517    |


### RFMマトリクス（R固定）
```sql
WITH recency_segment AS
(
  SELECT member_id,
    CASE 
      WHEN recency <=   7 THEN '1. 1week'
      WHEN recency <=  14 THEN '2. 2week'
      WHEN recency <=  30 THEN '3. 1month'
      WHEN recency <=  90 THEN '4. 3month'
      WHEN recency <= 365 THEN '5. 1year'
      WHEN 365 < recency  THEN '6. <1year'
      ELSE 'error'
    END AS seg_recency
  FROM rfm_result
),
frequency_segment AS
(
  SELECT member_id,
    CASE --(times*week)*(month)*(year)
      WHEN (3*4)*12*2 <= frequency THEN '1. 3/week'
      WHEN (2*4)*12*2 <= frequency THEN '2. 2/week'
      WHEN (1*4)*12*2 <= frequency THEN '3. 1/week'
      WHEN (2)  *12*2 <= frequency THEN '4. 2/month'
      WHEN (1)  *12*2 <= frequency THEN '5. 1/month'
      WHEN frequency < 12*2        THEN '6. 1/year'
      ELSE 'error'
    END AS seg_frequency
  FROM rfm_result
),
monetary_segment AS
(
  SELECT member_id,
    CASE --(money)*(month)*(year)
      WHEN ( 15000)*12*2 <= monetary THEN  '1. 15000/month'
      WHEN ( 10000)*12*2 <= monetary THEN  '2. 10000/month'
      WHEN (  5000)*12*2 <= monetary THEN  '3. 5000/month'
      WHEN (  2500)*12*2 <= monetary THEN  '4. 2500/month'
      WHEN (  1000)*12*2 <= monetary THEN  '5. 1000/month'
      WHEN monetary < (  1000)*12*2  THEN  '6. <1000/month'
      ELSE 'error'
    END AS seg_monetary
  FROM rfm_result
),
rf_table AS
(
  SELECT seg_recency, seg_frequency, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_frequency
    FROM recency_segment, frequency_segment
    WHERE recency_segment.member_id = frequency_segment.member_id
  )
  GROUP BY seg_recency, seg_frequency
),
rm_table AS
(
  SELECT seg_recency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_monetary
    FROM recency_segment, monetary_segment
    WHERE recency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_recency, seg_monetary
),
fm_table AS
(
  SELECT seg_frequency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT frequency_segment.member_id, seg_frequency, seg_monetary
    FROM frequency_segment, monetary_segment
    WHERE frequency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_frequency, seg_monetary
),
rfm_table AS
(
  SELECT seg_recency,seg_frequency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_frequency, seg_monetary
    FROM recency_segment, frequency_segment, monetary_segment
    WHERE recency_segment.member_id = monetary_segment.member_id
    AND frequency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_recency, seg_frequency, seg_monetary
)

SELECT seg_recency, seg_frequency,
  IF(ELEMENT_AT(kv,'1. 15000/month') IS NOT NULL, kv['1. 15000/month'], 0) AS c1_15000,
  IF(ELEMENT_AT(kv,'2. 10000/month') IS NOT NULL, kv['2. 10000/month'], 0) AS c2_10000,
  IF(ELEMENT_AT(kv,'3. 5000/month')  IS NOT NULL, kv['3. 5000/month'],  0) AS c3_5000,
  IF(ELEMENT_AT(kv,'4. 2500/month')  IS NOT NULL, kv['4. 2500/month'],  0) AS c4_2500,
  IF(ELEMENT_AT(kv,'5. 1000/month')  IS NOT NULL, kv['5. 1000/month'],  0) AS c5_1000,
  IF(ELEMENT_AT(kv,'6. <1000/month') IS NOT NULL, kv['6. <1000/month'], 0) AS c6_0
FROM
(
  SELECT seg_recency, seg_frequency, MAP_AGG(seg_monetary, cnt) AS kv
  FROM rfm_table
  GROUP BY seg_recency, seg_frequency
)
ORDER BY seg_recency, seg_frequency
```
|seg_recency|seg_frequency|c1_15000                |c2_10000            |c3_5000|c4_2500|c5_1000|c6_0|
|-----------|-------------|------------------------|--------------------|-------|-------|-------|----|
|1. 1week   |1. 3/week    |308                     |5                   |0      |0      |0      |0   |
|1. 1week   |2. 2/week    |375                     |55                  |7      |2      |0      |0   |
|1. 1week   |3. 1/week    |384                     |461                 |215    |7      |0      |0   |
|...        |             |                        |                    |       |       |       |    |
|6. <1year  |4. 2/month   |5                       |14                  |44     |16     |2      |0   |
|6. <1year  |5. 1/month   |1                       |0                   |32     |94     |26     |1   |
|6. <1year  |6. 1/year    |1                       |0                   |2      |43     |210    |325 |



### FMマトリクス（比率セグメント）
FMについても，デシル分析で見たMonetaryの売上比率を固定したセグメントと同じものを作ってみます。まず，Monetaryについては売上比率を5分割し，最上位の売上20%に入るユーザーを購入額の多い上位から入れていき，それを最もロイヤル度の高いセグメント1とします。次の売上20%に入るユーザーをセグメント2，その次を3として，5までのセグメントを作ります。
Frequencyについても同じ考え方で進め，全体の総頻度に対して頻度が高いほうから20%に入るユーザーをセグメント1とします。ただし，1人だけ圧倒的に頻度の高いユーザー（2259091）が含まれているので，このユーザーは除外して集計します。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH stat AS
(
  SELECT SUM(two_years_sales) AS total_sales, SUM(two_years_cnt) AS total_cnt
  FROM
  (
    SELECT member_id, SUM(price*amount) AS two_years_sales, COUNT(1) AS two_years_cnt
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, TD_TIME_ADD(TD_SCHEDULED_TIME(),'-730d'), TD_SCHEDULED_TIME(), 'JST')
    AND member_id IS NOT NULL AND member_id <> '2259091'
    GROUP BY member_id
  )
),
fm_table AS
(
  SELECT ftile, mtile, COUNT(1) AS cnt, 
    AVG(two_years_cnt) AS cnt_avg, AVG(two_years_sales) AS sales_avg, 
    SUM(two_years_cnt) AS cnt_sum, SUM(two_years_sales) AS sales_sum
  FROM
  (
    SELECT member_id, two_years_cnt, two_years_sales,
      CEIL( 1.0*SUM(two_years_cnt)  OVER(ORDER BY two_years_cnt   DESC)/total_cnt*5 )   AS ftile, 
      CEIL( 1.0*SUM(two_years_sales)OVER(ORDER BY two_years_sales DESC)/total_sales*5 ) AS mtile
    FROM
    (
      SELECT member_id, SUM(price*amount) AS two_years_sales, COUNT(1) AS two_years_cnt
      FROM sales_slip
      WHERE TD_TIME_RANGE(time, TD_TIME_ADD(TD_SCHEDULED_TIME(),'-730d'), TD_SCHEDULED_TIME(), 'JST')
      AND member_id IS NOT NULL AND member_id <> '2259091'
      GROUP BY member_id
    ), stat
  )
  GROUP BY ftile, mtile
)

SELECT CAST(mtile AS INTEGER) AS mtile, mtotal,
  IF(ELEMENT_AT(kv,1) IS NOT NULL, kv[1], 0) AS f1,
  IF(ELEMENT_AT(kv,2) IS NOT NULL, kv[2], 0) AS f2,
  IF(ELEMENT_AT(kv,3) IS NOT NULL, kv[3], 0) AS f3,
  IF(ELEMENT_AT(kv,4) IS NOT NULL, kv[4], 0) AS f4,
  IF(ELEMENT_AT(kv,5) IS NOT NULL, kv[5], 0) AS f5
FROM
(
  SELECT mtile, SUM(cnt) AS mtotal, MAP_AGG(ftile, cnt) AS kv
  FROM fm_table
  GROUP BY mtile
)
ORDER BY mtile
```
|mtile     |mtotal|f1                      |f2                  |f3 |f4 |f5     |
|----------|------|------------------------|--------------------|---|---|-------|
|1         |353   |237                     |71                  |25 |16 |4      |
|2         |772   |156                     |356                 |194|53 |13     |
|3         |1201  |25                      |287                 |517|305|67     |
|4         |1797  |4                       |74                  |388|867|464    |
|5         |4548  |9                       |13                  |82 |494|3950   |

上記の結果では，行にmonetaryセグメント，列にfrequencyセグメントとしています。mtotalはそれぞれのmonetaryセグメントの合計人数です。「monetaryセグメント=1，frequencyセグメント=1」の237人は，売上総額および頻度ともに全体の20%の割合を占める上位層です。その右の71人は，frequencyにおいては次の20%に入るユーザーとなります。

対角に位置するセグメントペアは，monetaryとfrequencyが同じセグメント，つまり（全体の割合の意味で）ほぼ同じ順位のユーザーで，この対角上にほとんどの数字が入ってしまう場合は，monetaryとfrequencyの関係が強いことを示しています。

最後におまけとして，monetaryとfrequency（セグメントではなく集計値そのもの）同士の相関を調べてみましょう。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
SELECT CORR(two_years_cnt,two_years_sales) AS cor
FROM
(
  SELECT member_id, SUM(price*amount) AS two_years_sales, COUNT(1) AS two_years_cnt
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, TD_TIME_ADD(TD_SCHEDULED_TIME(),'-730d'), TD_SCHEDULED_TIME(), 'JST')
  AND member_id IS NOT NULL AND member_id <> '2259091'
  GROUP BY member_id
)
```
|cor       |
|----------|
|0.7852289 |

