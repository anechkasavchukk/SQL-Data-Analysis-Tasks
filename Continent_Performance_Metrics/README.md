# Continent Performance Metrics Analysis

## 📌 Project Overview
This analysis provides a comprehensive view of business performance broken down by continent. It integrates data from orders, accounts, and session parameters to assess revenue distribution, device usage, and account verification status per region.

## 🛠 Tools & Technologies
* **Language:** SQL (Google BigQuery)
* **Techniques Used:** 
    * `WITH` clauses (CTEs) for modular query building.
    * Conditional aggregation (`CASE WHEN`) to split revenue by device.
    * `LEFT JOIN` to consolidate disparate metrics.
    * Window functions for calculating total revenue shares.

## 📊 Key Insights
* The project highlights revenue discrepancies between mobile and desktop users across different continents.
* It provides a clear snapshot of account verification rates and total session activity for each region.

## 💻 SQL Query Logic
1. **Revenue Segmentation:** Created a CTE to aggregate total revenue and split it into 'mobile' and 'desktop' categories using conditional aggregation.
2. **Account Profiling:** Developed a CTE to count distinct accounts and verified accounts per continent.
3. **Session Tracking:** Used a CTE to count distinct sessions per continent.
4. **Data Consolidation:** Joined all CTEs on the `continent` field to provide a final integrated performance report.
5. **Share Calculation:** Used `SUM(revenue) OVER()` to calculate the percentage contribution of each continent to the total global revenue.

### SQL Code
```sql
WITH rev_continent AS (
    SELECT 
        sp.continent, 
        SUM(p.price) AS revenue,
        SUM(CASE WHEN sp.device = 'mobile' THEN p.price END) AS Revenue_from_Mobile,
        SUM(CASE WHEN sp.device = 'desktop' THEN p.price END) AS Revenue_from_Desktop
    FROM `data-analytics-mate.DA.order` o
    JOIN `data-analytics-mate.DA.product` p ON o.item_id = p.item_id
    JOIN `data-analytics-mate.DA.session_params` sp ON o.ga_session_id = sp.ga_session_id
    GROUP BY 1 
),
ac_continent AS (
    SELECT 
        sp.continent, 
        COUNT(DISTINCT ac.id) AS Account_Count,
        COUNT(DISTINCT CASE WHEN ac.is_verified = 1 THEN ac.id END) AS Verified_Account
    FROM `data-analytics-mate.DA.account` ac
    JOIN `data-analytics-mate.DA.account_session` acs ON ac.id = acs.account_id
    JOIN `data-analytics-mate.DA.session_params` sp ON acs.ga_session_id = sp.ga_session_id
    GROUP BY 1
),
ses_continent AS (
    SELECT
        continent,
        COUNT(DISTINCT ga_session_id) AS Session_Count
    FROM `data-analytics-mate.DA.session_params`
    GROUP BY 1
)
SELECT 
    ses_continent.continent, 
    rev_continent.revenue AS Revenue,
    rev_continent.Revenue_from_Mobile, 
    rev_continent.Revenue_from_Desktop,
    (rev_continent.revenue / SUM(rev_continent.revenue) OVER()) * 100 AS Revenue_from_Total_pers,
    ac_continent.Account_Count, 
    ac_continent.Verified_Account, 
    ses_continent.Session_Count
FROM ses_continent
LEFT JOIN rev_continent ON ses_continent.continent = rev_continent.continent
LEFT JOIN ac_continent ON ac_continent.continent = ses_continent.continent;
