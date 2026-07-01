# Email Marketing & Account Acquisition Analysis

## 📌 Project Overview
This project analyzes the performance of email marketing campaigns and user account acquisition across different countries. The goal was to aggregate raw event data, prevent data duplication (fan-out), and rank the top-performing countries based on the number of generated accounts and emails sent. The final dataset was visualized to track campaign dynamics over time.

## 🛠 Tools & Technologies
* **Database:** Google BigQuery
* **Language:** SQL
* **Visualization:** Looker Studio / Tableau (or specify your BI tool)
* **Techniques Used:** CTEs, `LEFT JOIN`, `UNION ALL`, Window Functions (`SUM() OVER()`, `DENSE_RANK()`), Data Aggregation, Handling Fan-out issues.

## 📊 Key Insights & Conclusion
Based on the visualized metrics and the Top-10 ranking:
1. **Market Leadership:** The United States is the absolute leader in both the number of new accounts created (12,384) and the volume of emails sent (233,503).
2. **Campaign Dynamics:** The time-series analysis reveals a clear peak in email distribution during **December 2020**, followed by a gradual and consistent decline into early 2021.

![Dashboard Preview](посилання_на_твою_картинку_тут.png) 
*(Note: Upload your screenshot to the GitHub repo and replace the link above with the actual image path).*

## 💻 SQL Query Logic
The data extraction and transformation process was broken down into several logical steps using Common Table Expressions (CTEs):

1. **Email Metrics Aggregation:** Gathered email-related events (sent, opened, visited). Used `COUNT(DISTINCT)` to prevent fan-out when joining multiple event tables.
2. **Account Metrics Aggregation:** Aggregated account creation data with placeholder zeros for email metrics to align table structures.
3. **Data Merging:** Used `UNION ALL` to combine email and account datasets.
4. **Final Aggregation:** Grouped the merged data by date, country, and account statuses.
5. **Window Functions:** Calculated global country totals using `SUM() OVER(PARTITION BY country)`.
6. **Ranking:** Applied `DENSE_RANK()` to rank countries by total accounts and total emails sent, ensuring no gaps in ranking values.
7. **Filtering:** Outputted the final dataset filtered to show only the Top 10 countries in either category.

### SQL Code

```sql
-- Step 1: Gathering basic metrics for emails and statuses
WITH email_metrics AS (
    SELECT 
        DATE_ADD(s.date, INTERVAL es.sent_date DAY) AS date, 
        sp.country,  
        COUNT(DISTINCT es.id_message) AS sent_msg,
        COUNT(DISTINCT eo.id_message) AS open_msg,
        COUNT(DISTINCT ev.id_message) AS visit_msg,  -- Using DISTINCT to prevent Fan-out
        acc.send_interval,
        acc.is_verified,
        acc.is_unsubscribed,
        0 AS account_cnt
    FROM `data-analytics-mate.DA.email_sent` AS es
    LEFT JOIN `data-analytics-mate.DA.email_open` eo 
        ON es.id_message = eo.id_message             -- LEFT JOIN to keep unopened emails
    LEFT JOIN `data-analytics-mate.DA.email_visit` ev 
        ON es.id_message = ev.id_message
    JOIN `data-analytics-mate.DA.account_session` acs 
        ON es.id_account = acs.account_id
    JOIN `data-analytics-mate.DA.account` acc 
        ON acc.id = acs.account_id
    JOIN `data-analytics-mate.DA.session` s 
        ON acs.ga_session_id = s.ga_session_id
    JOIN `data-analytics-mate.DA.session_params` sp 
        ON acs.ga_session_id = sp.ga_session_id
    GROUP BY 1, 2, 6, 7, 8, 9
),

-- Step 2: Calculating the number of created accounts
cte_accounts AS (
    SELECT   
        s.date, 
        sp.country, 
        0 AS sent_msg, 
        0 AS open_msg, 
        0 AS visit_msg,   -- Placeholders to align structure for UNION ALL
        send_interval,
        is_verified,
        is_unsubscribed, 
        COUNT(DISTINCT id) AS account_cnt
    FROM `data-analytics-mate.DA.account` acc
    JOIN `data-analytics-mate.DA.account_session` acs 
        ON acc.id = acs.account_id
    JOIN `data-analytics-mate.DA.session` s 
        ON acs.ga_session_id = s.ga_session_id
    JOIN `data-analytics-mate.DA.session_params` sp 
        ON acs.ga_session_id = sp.ga_session_id
    GROUP BY 1, 2, 3, 4, 5, 6, 7, 8
),

-- Step 3: Merging the datasets
acc_email AS (
    SELECT date, country, sent_msg, open_msg, visit_msg, send_interval, is_verified, is_unsubscribed, account_cnt
    FROM email_metrics
    UNION ALL
    SELECT date, country, sent_msg, open_msg, visit_msg, send_interval, is_verified, is_unsubscribed, account_cnt
    FROM cte_accounts
),

-- Step 4: Grouping data after UNION
grouped_metrics AS (
    SELECT
        date,
        country,
        send_interval,
        is_verified,
        is_unsubscribed,
        SUM(account_cnt) AS account_cnt,
        SUM(sent_msg) AS sent_msg,
        SUM(open_msg) AS open_msg,
        SUM(visit_msg) AS visit_msg
    FROM acc_email
    GROUP BY 1, 2, 3, 4, 5
),

-- Step 5: Calculating global totals per country using Window Functions
cnt_step AS (
    SELECT 
        date, country, sent_msg, open_msg, visit_msg, send_interval, is_verified, is_unsubscribed, account_cnt,  
        SUM(account_cnt) OVER(PARTITION BY country) AS total_country_account_cnt, 
        SUM(sent_msg) OVER (PARTITION BY country) AS total_country_sent_cnt
    FROM grouped_metrics
),

-- Step 6: Ranking Top Countries
ranking AS (
    SELECT *,
        DENSE_RANK() OVER(ORDER BY total_country_account_cnt DESC) AS rank_total_country_account_cnt,
        DENSE_RANK() OVER(ORDER BY total_country_sent_cnt DESC) AS rank_total_country_sent_cnt  
    FROM cnt_step
)

-- FINAL: Outputting results with specific column order and Top-10 filtering
SELECT 
    date,
    country,
    send_interval,
    is_verified,
    is_unsubscribed,
    account_cnt,
    sent_msg,
    open_msg,
    visit_msg,
    total_country_account_cnt,
    total_country_sent_cnt,
    rank_total_country_account_cnt,
    rank_total_country_sent_cnt
FROM ranking
WHERE rank_total_country_account_cnt <= 10  -- Keeping only Top 10 for the final dashboard
   OR rank_total_country_sent_cnt <= 10;
