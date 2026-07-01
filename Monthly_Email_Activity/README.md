# Monthly Email Activity Analysis

## 📌 Project Overview
This analysis explores user engagement patterns in email marketing on a monthly basis. It calculates the percentage of messages sent by each user relative to the total volume for that month and tracks the first and last interaction dates for each user.

## 🛠 Tools & Technologies
* **Language:** SQL (Google BigQuery)
* **Techniques Used:** 
    * `PARTITION BY` with `COUNT()` to calculate relative shares.
    * `MIN()` and `MAX()` window functions to track user interaction lifecycles.
    * `DATE_TRUNC` to aggregate data by month.

## 📊 Key Insights
* The analysis identifies individual user contribution to monthly email volume, which is essential for segmenting active versus passive users[cite: 3].
* It provides clear visibility into the "first" and "last" email dates for every account within each month.

## 💻 SQL Query Logic
1. **Data Joining:** Combined email sent data with account and session tables to associate messages with specific accounts and timeframes.
2. **Monthly Truncation:** Used `DATE_TRUNC` to group events by month[cite: 3].
3. **Relative Percentage:** Calculated the ratio of messages sent by an account to the total messages sent in that month using window functions.
4. **Lifecycle Tracking:** Used `MIN()` and `MAX()` window functions partitioned by account and month to identify the first and last sent dates per user.

### SQL Code
```sql
SELECT DISTINCT 
    sent_month, 
    id_account,
    (COUNT(id_message) OVER (PARTITION BY id_account, sent_month) / COUNT(id_message) OVER(PARTITION BY sent_month)) * 100 AS sent_msg_percent_from_this_month,
    MIN(sent_date) OVER (PARTITION BY id_account, sent_month) AS first_sent_date,
    MAX(sent_date) OVER (PARTITION BY id_account, sent_month) AS last_sent_date
FROM (
    SELECT 
        ac.id AS id_account,
        es.id_message,
        DATE_ADD(s.date, INTERVAL es.sent_date DAY) AS sent_date,
        DATE_TRUNC(DATE_ADD(s.date, INTERVAL es.sent_date DAY), MONTH) AS sent_month
    FROM `data-analytics-mate.DA.email_sent` es
    JOIN `data-analytics-mate.DA.account_session` acs ON es.id_account = acs.account_id
    JOIN `data-analytics-mate.DA.session` s ON acs.ga_session_id = s.ga_session_id
    JOIN `data-analytics-mate.DA.account` ac ON acs.account_id = ac.id
) sub
ORDER BY sent_month, id_account;
