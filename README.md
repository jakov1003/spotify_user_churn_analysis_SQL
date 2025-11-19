# Introduction
Previous projects I uploaded on this profile were both challenging and fun, but I wanted to do something more business-oriented this time.

# The Dataset
I downloaded [this dataset](https://www.kaggle.com/datasets/nabihazahid/spotify-dataset-for-churn-analysis) from Kaggle. It contains 8000 rows of Spotify user data.

For better clarity, I renamed three columns: 'listening_time' to 'mins_listened_per_day', 'skip_rate' to 'song_skip_rate', and 'device_type' to 'platform.'

Click the dropdown arrow below if you would like to view a 5-row preview of the dataset.
<details>
<summary> Dataset preview </summary>

| user_id | gender | age | country | subscription_type | mins_listened_per_day | songs_played_per_day | song_skip_rate | platform | ads_listened_per_week | offline_listening | is_churned |
|:------:|:------:|:---:|:-------:|:----------------:|:--------------------:|:-------------------:|:--------------:|:--------:|:--------------------:|:----------------:|:----------:|
| 1      | Female | 54  | CA      | Free             | 262                  | 30                  | 0.2            | Desktop  | 3                    | 1                | 1          |
| 2      | Other  | 33  | DE      | Family           | 141                  | 620                 | 0.34           | Web      | 0                    | 1                | 0          |
| 3      | Male   | 38  | AU      | Premium          | 199                  | 380                 | 0.04           | Mobile   | 0                    | 1                | 1          |
| 4      | Female | 22  | CA      | Student          | 36                   | 20                  | 0.31           | Mobile   | 0                    | 1                | 0          |
| 5      | Other  | 29  | US      | Family           | 250                  | 570                 | 0.36           | Mobile   | 0                    | 1                | 1          |
</details>

# Tools I Used
- SQL
- PostgreSQL
- pgAdmin 4
- Visual Studio Code
- Git & Github (for version control and sharing)

# Churn
Churn refers to users who have either stopped subscribing to paid plans, stopped using the app entirely, or both.

The dataset labeled some free users as churned, so I used the term in the broadest sense of the word.

I analyzed Spotify user churn from 3 perspectives:

**1.Age**

**2.Platform**

**3.Culture**

# The Analysis
## 1.Age churn

```sql
with churned_age as (
    SELECT
        -- 16 is the minimum value of user age in this dataset, 59 is the maximum 
        CASE
        WHEN age BETWEEN 16 AND 24 THEN '16–24'
        WHEN age BETWEEN 25 AND 34 THEN '25-34'
        WHEN age BETWEEN 35 AND 44 THEN '35–44'
        WHEN age BETWEEN 45 AND 54 THEN '45–54'
        WHEN age BETWEEN 55 AND 59 THEN '55–59'
        ELSE 'Unknown'
        END AS age_groups,
    COUNT(user_id) as churned_count
    FROM 
        spotify_user_analysis
    WHERE 
        is_churned = TRUE
    GROUP BY
        age_groups
),

not_churned_age as (
    SELECT
        CASE 
        WHEN age BETWEEN 16 AND 24 THEN '16–24'
        WHEN age BETWEEN 25 AND 34 THEN '25-34'
        WHEN age BETWEEN 35 AND 44 THEN '35–44'
        WHEN age BETWEEN 45 AND 54 THEN '45–54'
        WHEN age BETWEEN 55 AND 59 THEN '55–59'
        ELSE 'Unknown'
        END AS age_groups,
    COUNT(user_id) as not_churned_count
    FROM 
        spotify_user_analysis
    WHERE 
        is_churned = FALSE
    GROUP BY
        age_groups
)

SELECT
    cte_1.age_groups,
    cte_2.not_churned_count as current_users,
    cte_1.churned_count as former_users,
-- I decided to use ratios of current to former users instead of calculating differences 
-- because one age group has significantly fewer users than the others in this dataset.
-- I had to apply the CAST function due to situation-specific constraints
-- of the division operation and the ROUND function.
    ROUND((cte_2.not_churned_count::float / cte_1.churned_count)::numeric, 2) AS current_to_former_ratio
FROM
    churned_age as cte_1
INNER JOIN
    not_churned_age as cte_2 ON cte_2.age_groups = cte_1.age_groups
GROUP BY
    cte_1.age_groups, cte_1.churned_count, cte_2.not_churned_count
ORDER BY
    current_to_former_ratio DESC
```
### Query output
Larger ratio = better, smaller ratio = worse.

| age_groups   | current_users | former_users | current_to_former_ratio |
|:-------------:|:-------------:|:-------------:|:-----------------------:|
| 16–24        |     1251      |      416      |          3.01           |
| 35–44        |     1355      |      460      |          2.95           |
| 45–54        |     1391      |      486      |          2.86           |
| 55–59        |      668      |      245      |          2.73           |
| 25–34        |     1264      |      464      |          2.72           |

### Insight
One data-driven business decision could be to improve the song suggestion algorithm for users aged 25–34 and 55–59, as age is an important factor in Spotify’s song recommendation algorithm.  

## 2.Platform churn

```sql
with churned as (
    SELECT
        platform, 
        COUNT (user_id) as churned
    FROM 
        spotify_user_analysis
    WHERE 
        is_churned = TRUE
    GROUP BY
        platform
),

not_churned as (
    SELECT
        platform,
        COUNT(user_id) as not_churned
    FROM
        spotify_user_analysis
    WHERE
        is_churned = FALSE
    GROUP BY
        platform
)

SELECT
    cte_1.platform,
    cte_2.not_churned as current_users,
    cte_1.churned as former_users,
-- Here, I decided to use the difference between former and current users
-- because the user base is of the similar size across all three platforms in the context of this dataset. 
-- Using ratios would not yield useful insights in this case.
    cte_2.not_churned - cte_1.churned as difference
FROM
    churned as cte_1
INNER JOIN
    not_churned as cte_2 ON cte_2.platform = cte_1.platform
GROUP BY
    cte_1.platform, cte_1.churned, cte_2.not_churned
ORDER BY
    difference DESC
```

### Query output
Larger difference = better, smaller difference = worse.

| platform | current_users | former_users | difference |
|---|:-:|:-:|:-:|
| Desktop | 2063 | 715 | 1348 |
| Web | 1966 | 657 | 1309 |
| Mobile | 1900 | 699 | 1201 |

### Insights

Data-driven business decisions could include investigating the sound quality in mobile apps and improving their overall app design and user experience. 

If we approach user retention from the platform perspective, technical and UX/UI issues are one of the main reasons why they churn.

## 3.Cultural churn

```sql
with churned_culture as (
    SELECT
        -- Since this dataset only includes six countries,
        -- I thought that grouping them by 
        -- cultural similarities and/or geographical proximity
        -- could help derive more actionable insights.
        CASE
        WHEN country IN ('US', 'UK', 'CA', 'AU') THEN 'Anglosphere'
        WHEN country IN ('FR', 'DE') THEN 'France-Germany'
        WHEN country IN ('IN', 'PK') THEN 'India-Pakistan'
        ELSE 'Unknown'
        END AS cultural_groups,
        COUNT(user_id) as churned_count
    FROM 
        spotify_user_analysis
    WHERE 
        is_churned = TRUE
    GROUP BY
        cultural_groups
),

not_churned_culture as (
    SELECT
        CASE 
        WHEN country IN ('US', 'UK', 'CA', 'AUS') THEN 'Anglosphere'
        WHEN country IN ('FR', 'DE') THEN 'France-Germany'
        WHEN country IN ('IN', 'PK') THEN 'India-Pakistan'
        ELSE 'Unknown'
        END as cultural_groups,
    COUNT(user_id) as not_churned_count
    FROM 
        spotify_user_analysis
    WHERE 
        is_churned = FALSE
    GROUP BY
        cultural_groups
)

SELECT
    cte_1.cultural_groups,
    cte_2.not_churned_count as current_users,
    cte_1.churned_count as former_users,
-- I decided to use ratios of current to former users instead of calculating differences 
-- because one cultural group has significantly more users than the others in this dataset.
-- I had to apply the CAST function due to situation-specific constraints
-- of the division operation and the ROUND function.
    ROUND((cte_2.not_churned_count::float / cte_1.churned_count)::numeric, 2) AS current_to_former_ratio
FROM
    churned_culture as cte_1
INNER JOIN
    not_churned_culture as cte_2 ON cte_2.cultural_groups = cte_1.cultural_groups
GROUP BY
    cte_1.cultural_groups, cte_1.churned_count, cte_2.not_churned_count
ORDER BY
    current_to_former_ratio DESC
```
### Query output
Larger ratio = better, smaller ratio = worse.
            
| cultural_groups | current_users | former_users | current_to_former_ratio |
|:---------------:|:-------------:|:------------:|:----------------------:|
| Anglosphere     | 2982          | 1004         | 2.97                   |
| India-Pakistan  | 1489          | 521          | 2.86                   |
| France-Germany  | 1458          | 546          | 2.67                   |


### Insight - it's never enough

One data-driven business decision could be marketing investments specifically targeted at France and Germany, because the users from that group appear more difficult to retain based on their listening habits and general behavior.

## Time for some critical thinking 

Several aspects of this dataset bothered me:

1.No distinction between Android and iOS users (both labeled as 'Mobile').

2.No distinction between Windows and macOS users (both labeled as 'Desktop').

3.Absence of time-based data.

However, despite these problems, it was still quite useful for what I wanted to achieve with this project.

## A very simple conclusion

I improved my business-oriented thinking and, once again, refreshed my knowledge of SQL.














