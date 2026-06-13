
### Data Modeling
The Data modeling DDL is below, basically to enable us calculate the the LTV:CAC by each join date/week/month/year cohort, I create two fact table one stores all the related cohort information for predicted_ltv which attributed to join_date, and the other is to attribute the daily ads campaigns spends per user sign up to the user join_date as well.

once we have those two table, we can compute the final LTV:CAC value.

fct_cohort_date_ltv DDL:
```
CREATE TABLE IF NO EXISTS "fct_cohort_date_ltv"
 ( "user_id" string,
  "join_date" date,
   "acquisition_channel" string,
   "listing_id" string,
   "listed_date" date,
   "host_ltv" decimal,
   "join_date_year" date,
   "join_date_month" date,
   "join_date_week" date )
```

fct_cohort_date_cac DDL:
```
CREATE TABLE IF NO EXISTS "fct_cohort_date_cac" ( "user_id" string, "join_date" date, "acquisition_channel" string, "total_spend" decimal, "total_attributed_signups" integer, "cac" decimal )
```

DML sql to create the two fact table:
fct_cohort_date_ltv:

```
-- get unique list information
WITH listings AS (
SELECT listing_id, host_user_id, listed_date
FROM listings_daily_snapshot
GROUP BY 1,2,3
),

--get unique user join information
users AS (
SELECT user_id, join_date, acquisition_channel
FROM users_daily_snapshot
GROUP BY 1,2,3
),

--get each predicted host ltv value
pred_ltv AS (
SELECT listing_id, host_ltv, predicted_at
FROM listing_predictions
)

--join everything to attributed the --predicted ltv to user join_date, create --the different cohort based on join_date
SELECT
A.user_id, A.join_date, A.acquisition_channel, B.listing_id, B.listed_date, C.host_ltv,
strftime('%Y-01-01', A.join_date) AS join_date_year,
strftime('%Y-%m-01', A.join_date) AS join_date_month,
date(A.join_date, 'weekday 0', '-6 days') AS join_date_week
FROM users A
LEFT JOIN listings B
ON A.user_id = B.host_user_id
LEFT JOIN pred_ltv C
ON B.listing_id = C.listing_id
AND DATE(B.listed_date) = Date(C.predicted_at)
```

fct_cohort_date_cac:
```
--same unique user join date
WITH users AS (
    SELECT user_id, join_date, acquisition_channel
    FROM users_daily_snapshot
    GROUP BY 1,2,3
    ),
    --aggrate the CAC by date and channel
    spend_by_channel_date AS (
    SELECT date, channel, SUM(spend_amount) AS total_spend,
    SUM(attributed_signups) AS total_attributed_signups,
    SUM(spend_amount) / NULLIF(SUM(attributed_signups), 0) AS cac
    FROM ad_spend_daily
    GROUP BY 1,2
    )
    --beacause each user acuiqistion_channel is unique so we attributed the channel level cac to individual user level
    SELECT A.user_id, A.join_date, A.acquisition_channel, B.total_spend, B.total_attributed_signups, B.cac
    FROM users A LEFT JOIN spend_by_channel_date B
    ON A.join_date = B.date
    AND A.acquisition_channel = B.channel
```

### Acquisition-cpost attribution approach
The main issues is from the ad_spend_daily table, the ads spend is only attributed to daily campaign level, rather than individual user level, so we take the average ads spends per each users in one channel cohort as a proxy for individual CAC.


### Assumptions I made aboue edge case

The average acquisition-cost limition:

1. different user might actually have different acquisition path and acquisition values but we treat the acquisition costs all the same.

2.if there is a significant variablity in the cost per sign up, this will make the value very noisy.

3. we assume for each join_date and the same channel, all the user cac is the same for now

Even though the attributed_signups are noisy but we only take whatever value map in the channel to the user acquisition channel is correct, ignore if there is any overlap attributed to different channels.

There are cohort join date which does not have LTV values buty do have CAC values, so we are also assume that acquisition users don't necessary have to have listing property value.

### tooling
Colab notebook for python
sqlite import in python to create table and write sql


### AI usage prompts
Claude prompts:

1. let's say we have a ad_spend_daily table as below:
Daily ad spend per campaign, with an impressions count and a count of users the marketing platform attributed to that campaign on that day.
in this case how should we understand the acquisition cost attributed to user's join date?

answer:
Summary of the Logic
attributed_cac per user =
    total spend on that day for that channel
    ÷
    total platform-attributed signups on that day for that channel

...assigned to every user who joined on that day via that channel
This is a simplification but it's the standard industry approach when you don't have individual-level ad attribution data — and it's exactly what the statement "acquisition cost is attributed to the user's join date" means in practice.

2.the metric we want is LTV : CAC, what does it mean exactly? and what is the range of this value and what each value represents for?

Great question! Here's a comprehensive breakdown:

LTV:CAC Benchmarks by Business Type
RatioSignalTypical Action< 1:1Destroying valuePause spending, fix fundamentals1:1 – 2:1UnsustainableOptimize urgently3:1Industry benchmarkHealthy, maintain4:1 – 5:1Very efficientConsider scaling spend> 5:1Under-investingLikely leaving growth on table

By Industry / Business Model
IndustryTypical LTV:CACNotesSaaS / Subscription3:1 – 5:1Most cited benchmarkE-commerce2:1 – 4:1Lower margins compress LTVMarketplace (like Airbnb)3:1 – 6:1Network effects help LTVFintech3:1 – 5:1High LTV if retained longGaming2:1 – 4:1Highly channel dependentMobile Apps2:1 – 3:1High competition = high CACEnterprise SaaS5:1 – 10:1Very high LTV, long contractsConsumer / Retail1.5:1 – 3:1Thin margins

Context Matters More Than the Number
Early Stage Startup
LTV:CAC = 1.5:1  →  might be acceptable
                     still finding product-market fit
                     CAC will improve with scale
Growth Stage
LTV:CAC = 3:1    →  expected benchmark
                     should be optimizing channels
Mature Business
LTV:CAC = 4:1+   →  expected at scale
                     brand recognition lowers CAC
                     retained users raise LTV


Gemini prompts:
1. in the data_dictionary it says the attributed_signups column ins noisy in ad_spend_daily, is it possible that the attribution channel can mis attributed here? for example, in cell 22, I found for join_date at 2021-05-25, 2026-05-24, 2026-05-25 there are no CAC value due to the no matching channel, but could that possible be misleading?

answer:
To address this, you might consider strategies like:

Imputation: Filling NULL CAC values with an average CAC from the same channel or a global average, though this introduces assumptions.
Filtering: Excluding users or cohorts with NULL CAC from LTV:CAC calculations, but this means you're analyzing a subset of your data.
Further Investigation: If these NULLs are frequent, it might warrant a deeper dive into the ad_spend_daily data source to understand why these gaps exist.

2.so acquire user that means the user saw the ads campaign and click into it but does not add any listing? is that the currect understanding? in the industry, how do we identify the attribute user sign up by which action?

answer:
Attribution for user sign-ups is a complex field, but generally revolves around connecting a conversion event (like a sign-up) back to the marketing touchpoints that contributed to it.


### SQL queries compute LTV:CAC


join date Yearly LTV:CAC:
```
  SELECT A.join_date_year, SUM(COALESCE(A.host_ltv, 0)) AS host_ltv,
  SUM(COALESCE(B.cac,0)) AS cac,
  SUM(COALESCE(A.host_ltv, 0)) / NULLIF(SUM(COALESCE(B.cac,0)), 0) AS ltv_cac
  FROM fct_cohort_date_ltv A INNER JOIN fct_cohort_date_cac B
  ON A.user_id = B.user_id
  AND A.join_date = B.join_date
  AND A.acquisition_channel = B.acquisition_channel
  GROUP BY 1
  ORDER BY 1

```

join date Monthly  LTV:CAC:

```
  SELECT A.join_date_month, SUM(COALESCE(A.host_ltv, 0)) AS host_ltv,
  SUM(COALESCE(B.cac,0)) AS cac,
  SUM(COALESCE(A.host_ltv, 0)) / NULLIF(SUM(COALESCE(B.cac,0)), 0) AS ltv_cac
  FROM fct_cohort_date_ltv A INNER JOIN fct_cohort_date_cac B
  ON A.user_id = B.user_id
  AND A.join_date = B.join_date
  AND A.acquisition_channel = B.acquisition_channel
  GROUP BY 1
  ORDER BY 1

```

join date Weekly  LTV:CAC:

```
  SELECT A.join_date_week, SUM(COALESCE(A.host_ltv, 0)) AS host_ltv,
  SUM(COALESCE(B.cac,0)) AS cac,
  SUM(COALESCE(A.host_ltv, 0)) / NULLIF(SUM(COALESCE(B.cac,0)), 0) AS ltv_cac
  FROM fct_cohort_date_ltv A INNER JOIN fct_cohort_date_cac B
  ON A.user_id = B.user_id
  AND A.join_date = B.join_date
  AND A.acquisition_channel = B.acquisition_channel
  GROUP BY 1
  ORDER BY 1
```

join date daily  LTV:CAC:
```
  SELECT A.join_date, SUM(COALESCE(A.host_ltv, 0)) AS host_ltv,
  SUM(COALESCE(B.cac,0)) AS cac,
  SUM(COALESCE(A.host_ltv, 0)) / NULLIF(SUM(COALESCE(B.cac,0)), 0) AS ltv_cac
  FROM fct_cohort_date_ltv A INNER JOIN fct_cohort_date_cac B
  ON A.user_id = B.user_id
  AND A.join_date = B.join_date
  AND A.acquisition_channel = B.acquisition_channel
  GROUP BY 1
  ORDER BY 1
```


### Setup Instructions
1. read the README file to each of the questions with dedicated answers

2.can open the notebook using gogle colab or jupyter notbook.

3. make sure update the file location path and import all the five csv files under the correct file directory

4. run the cell one by one and the final answers should be the last 4 cells, once run it, it will generate the answers in dataframe results, or i created the final csv with all the results stored in each of the tab.
