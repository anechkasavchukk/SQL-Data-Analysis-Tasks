# Revenue Prediction vs. Actual Performance

## 📌 Project Overview
This analysis evaluates the accuracy of revenue predictions by comparing them against actual daily revenue data. The project calculates the percentage of fulfillment over time to identify how closely the business met its targets.

## 🛠 Tools & Technologies
* **Language:** SQL (Google BigQuery)
* **Techniques Used:** 
    * `UNION ALL` for combining disparate datasets (revenue and predictions).
    * `SUM() OVER(ORDER BY ...)` for calculating cumulative totals.
    * `DATE_TRUNC` for temporal aggregation.

## 📊 Key Insights
* The analysis provides a daily fulfillment percentage, helping stakeholders understand if the revenue projections are over- or underestimated[cite: 2].

## 💻 SQL Query Logic
1. **Data Unification:** Used `UNION ALL` to merge separate tables containing actual daily revenue and daily predicted revenue.
2. **Daily Aggregation:** Grouped the unified data by day to get total actual revenue and total predicted revenue per day.
3. **Cumulative Metrics:** Applied window functions with `ORDER BY extr_day` to calculate running totals for both revenue and predictions.
4. **Percentage Calculation:** Divided the cumulative actual revenue by the cumulative predicted revenue to derive a daily fulfillment percentage.

### SQL Code
```sql
SELECT 
    extr_day, 
    (SUM(daily_revenue) OVER(ORDER BY extr_day) / SUM(daily_predict) OVER(ORDER BY extr_day)) * 100 AS perc
FROM (
    SELECT
        extr_day,
        SUM(revenue) AS daily_revenue,
        SUM(total_predict) AS daily_predict
    FROM(
        SELECT 
            DATE_TRUNC(date, DAY) AS extr_day,
            SUM(price) AS revenue,
            0 AS total_predict
        FROM `data-analytics-mate.DA.product` p
        JOIN `data-analytics-mate.DA.order` o ON p.item_id = o.item_id
        JOIN `data-analytics-mate.DA.session` s ON o.ga_session_id = s.ga_session_id
        GROUP BY 1
        UNION ALL
        SELECT 
            DATE_TRUNC(date, DAY) AS extr_day,
            0 AS revenue,  
            SUM(predict) AS total_predict
        FROM `data-analytics-mate.DA.revenue_predict` rp
        GROUP BY 1
    ) raw_union
    GROUP BY 1 
) AS daily_grouped
ORDER BY extr_day;
