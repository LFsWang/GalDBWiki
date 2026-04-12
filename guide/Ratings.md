# 評分系統

GalDB 目前顯示的評分資料來自 [VNDB](https://vndb.org/) 每日資料庫 dump。未來 GalDB 將建立自有評分系統。

## 數值欄位

| 欄位 | 說明 | 儲存格式 |
|---|---|---|
| vote_average（算術平均） | 所有投票的簡單平均 | ×10 整數（823 = 82.3 分） |
| rating（貝氏平均） | 經 Bayesian average 修正後的分數 | ×10 整數（823 = 82.3 分） |
| vote_count | 有效投票數 | 整數 |
| vote_stddev | 投票的樣本標準差 | ×10 整數 |
| vote_dist | 投票分布直方圖 | 10 格陣列（1★ ~ 10★） |

VNDB 的投票範圍是 10–100（對應顯示分數 1.0–10.0）。GalDB 儲存時直接沿用此值。

## 貝氏平均分數（Bayesian Average）

VNDB 的 `c_rating` 欄位並非單純的算術平均，而是使用**貝氏平均**（Bayesian average）來避免低票數作品因少數極端投票而排名異常偏高或偏低。

### 公式

$$
\text{rating} = \frac{p \cdot a + \sum(\text{votes})}{p + n}
$$

其中：

- **n** = 該作品的投票數（`COUNT(votes)`）
- **Σ(votes)** = 該作品所有投票的總和（`SUM(votes)`）
- **a** = 全站「各作品平均分」的平均值（average of per-VN averages）
- **p** = 虛擬投票數（pseudo-count），用來控制向全站均值收斂的強度

### 虛擬投票數 p 的計算

$$
p = \left(1 - \min\left(1,\; \frac{n}{100}\right)\right) \times 7
$$

p 是 n 的線性遞減函數：

| 投票數 n | p 值 | 效果 |
|---|---|---|
| 0 | 7.0 | 完全由全站均值決定 |
| 50 | 3.5 | 一半權重來自全站均值 |
| 100+ | 0.0 | 完全由實際投票決定 |

當投票數達到 100 以上，p = 0，rating 退化為純算術平均。投票數越少，全站均值的影響越大。

### 全站均值 a 的定義

```sql
SELECT AVG(a) FROM (
    SELECT AVG(vote) FROM votes GROUP BY vid
) x(a)
```

a 是「先對每部作品求平均，再對所有作品的平均值取平均」。這確保每部作品的權重相等，不會因為某部作品票數特別多而主導全站均值。

### 分數上限（Rating Cap）

為了防止低票數作品靠少量高分衝上排行榜頂端，VNDB 對 rating 施加了上限：

| 投票數 | 上限規則 |
|---|---|
| < 5 | 不計算 rating（顯示為 null） |
| 5 ≤ n < 50 | 不超過「≥ 50 票作品」中排名第 102 的 rating |
| 50 ≤ n < 100 | 不超過「≥ 100 票作品」中排名第 52 的 rating |
| ≥ 100 | 無上限 |

### 有效投票的過濾

計算時會排除被標記為 `ign_votes`（忽略投票）的使用者。這是 VNDB 管理員用來處理灌票帳號的機制。

### 完整 SQL（VNDB 原始碼）

以下為 VNDB 的 `update_vnvotestats()` 函式，擷取自 [`sql/func.sql`](https://code.blicky.net/yorhel/vndb/src/branch/master/sql/func.sql)：

```sql
CREATE OR REPLACE FUNCTION update_vnvotestats() RETURNS void AS $$
  WITH votes(vid, uid, vote) AS (
    -- 所有有效投票（排除被忽略的使用者）
    SELECT vid, uid, vote
      FROM ulist_vns
     WHERE vote IS NOT NULL
       AND uid NOT IN(SELECT id FROM users WHERE ign_votes)

  ), avgavg(avgavg) AS (
    -- 全站「各作品平均」的平均值
    SELECT AVG(a)::real
      FROM (SELECT AVG(vote) FROM votes GROUP BY vid) x(a)

  ), ratings(vid, count, average, rating) AS (
    SELECT vid, COUNT(uid), (AVG(vote)*10)::smallint,
           -- Bayesian average: B(a, p, votes) = (p*a + sum(votes)) / (p + count(votes))
           --   p = (1 - min(1, count/100)) * 7
           --   a = avgavg（全站均值）
           ( (1 - LEAST(1, COUNT(uid)::real/100))*7
               * (SELECT avgavg FROM avgavg)
             + SUM(vote)
           ) / ( (1 - LEAST(1, COUNT(uid)::real/100))*7 + COUNT(uid) )
      FROM votes
     GROUP BY vid

  ), capped(vid, count, average, rating) AS (
    -- 對低票數作品施加分數上限
    SELECT vid, count, average, CASE
      WHEN count <   5 THEN NULL
      WHEN count <  50 THEN LEAST(rating,
        (SELECT rating FROM ratings WHERE count >= 50
         ORDER BY rating DESC LIMIT 1 OFFSET 101))
      WHEN count < 100 THEN LEAST(rating,
        (SELECT rating FROM ratings WHERE count >= 100
         ORDER BY rating DESC LIMIT 1 OFFSET 51))
      ELSE rating END
    FROM ratings

  ), stats(vid, count, average, rating, rat_rank, pop_rank) AS (
    SELECT v.id, COALESCE(r.count, 0), r.average, (r.rating*10)::smallint
         , CASE WHEN r.rating IS NULL THEN NULL
           ELSE rank() OVER(ORDER BY hidden, r.rating DESC NULLS LAST) END
         , rank() OVER(ORDER BY hidden, r.count DESC NULLS LAST)
      FROM vn v
      LEFT JOIN capped r ON r.vid = v.id
  )
  UPDATE vn
     SET c_rating    = rating,    -- ×10 整數，100–1000
         c_votecount = count,
         c_pop_rank  = pop_rank,
         c_rat_rank  = rat_rank,
         c_average   = average    -- ×10 整數，100–1000
    FROM stats
   WHERE id = vid;
$$ LANGUAGE SQL;
```

### 數值範例

假設全站均值 a = 68.5（對應顯示分 6.85），某作品有 20 票、算術平均 85.0、票數總和 1700：

1. 計算 p：`(1 - min(1, 20/100)) × 7 = 0.8 × 7 = 5.6`
2. 計算 rating：`(5.6 × 68.5 + 1700) / (5.6 + 20) = (383.6 + 1700) / 25.6 ≈ 81.4`
3. 最終儲存值：`814`（×10 整數）

算術平均是 85.0，但 Bayesian average 把它拉向全站均值，修正為 81.4。

## GalDB 的顯示規則

- 投票數 < 10 時不顯示評分（前端 `MIN_VOTES = 10`）
- 評分以 ×10 刻度顯示（如 82.3）
- 評分依分數分為 7 個等級（D / C / B / A / S / SS / SSS）

| 等級 | 分數門檻 |
|---|---|
| SSS | ≥ 90 |
| SS | ≥ 80 |
| S | ≥ 75 |
| A | ≥ 70 |
| B | ≥ 65 |
| C | ≥ 60 |
| D | < 60 |

## 投票分布直方圖

投票分布以 10 格直方圖呈現，每格對應一個分數段：

| 格索引 | 投票範圍 | 顯示標籤 |
|---|---|---|
| 0 | 10–19 | 1★ |
| 1 | 20–29 | 2★ |
| … | … | … |
| 9 | 90–100 | 10★ |

分桶公式：`bucket_index = floor(vote / 10) - 1`

投票分布與標準差由 GalDB 自行從 VNDB votes dump 計算（使用 [Welford's online algorithm](https://en.wikipedia.org/wiki/Algorithms_for_calculating_variance#Welford's_online_algorithm)），而非直接從 VNDB 的資料庫 dump 取得。
