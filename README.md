# 📊 SQL Analysis for User Behavior, Savings, and Withdrawals

## 📁 Database Tables Overview

This SQL script analyzes user behavior and financial activities across the following tables:

- `users_customuser`: Stores user profile data such as email, join date, activity status, risk appetite, salary, etc.
- `savings_savingsaccount`: Tracks the amount each user has saved.
- `plans_plan`: Represents financial or investment plans associated with users.
- `withdrawals_withdrawal`: Logs all user withdrawal transactions.

---

## 🔍 Query Explanations

### 1. **Retrieve Recently Joined Users (Last 60 Days)**
```sql
SELECT id, email, date_joined, is_active
FROM users_customuser
WHERE is_active = 1 AND date_joined >= CURDATE() - INTERVAL 60 DAY;
```
Retrieves active users who joined in the last 60 days. Useful for tracking new user acquisition.

---

### 2. **Total Savings Per User**
```sql
SELECT u.id, u.email, COALESCE(SUM(s.amount), 0) AS total_savings
FROM users_customuser u
LEFT JOIN savings_savingsaccount s ON u.id = s.id
GROUP BY u.id, u.email;
```
Calculates total savings for each user. `COALESCE` ensures users with no savings still appear with `0`.

---

### 3. **Users Without Withdrawals**
```sql
SELECT u.id, u.email FROM users_customuser u
WHERE u.id NOT IN (
  SELECT DISTINCT w.id FROM withdrawals_withdrawal w
);
```
Identifies users who have never made any withdrawals.

---

### 4. **Average Salary by Gender**
```sql
SELECT gender_id, AVG(monthly_salary) AS avg_salary
FROM users_customuser
GROUP BY gender_id;
```
Shows average monthly salary across different genders.

---

### 5. **Users with Plans Created in 2025**
```sql
SELECT u.id, u.email, p.name, p.created_on
FROM users_customuser u
JOIN plans_plan p ON u.created_on = p.created_on
WHERE YEAR(p.created_on) = 2025;
```
Displays users who created plans in 2025. Join might be improved by matching user `id` to plan `owner_id` if available.

---

### 6. **Top Users by Maximum Savings**
```sql
SELECT u.id, u.email, MAX(s.amount) AS max_savings
FROM users_customuser u
JOIN savings_savingsaccount s ON u.id = s.id
GROUP BY u.id, u.email
ORDER BY max_savings DESC
LIMIT 10;
```
Lists users with the highest single savings entries.

---

### 7. **Withdrawal Frequency Per User**
```sql
SELECT u.id, u.email, COUNT(w.id) AS withdrawal_count
FROM users_customuser u
LEFT JOIN withdrawals_withdrawal w ON u.id = w.id
GROUP BY u.id, u.email;
```
Counts number of withdrawals per user.

---

### 8. **Users With Both Savings and Withdrawals**
```sql
SELECT u.id, u.email
FROM users_customuser u
WHERE EXISTS (SELECT 1 FROM savings_savingsaccount s WHERE s.id = u.id)
  AND EXISTS (SELECT 1 FROM withdrawals_withdrawal w WHERE w.id = u.id);
```
Identifies users who have both saved and withdrawn.

---

### 9. **Update Inactive Users**
```sql
UPDATE users_customuser
SET is_active = 0
WHERE id IN (
  SELECT id FROM (
    SELECT id FROM users_customuser
    WHERE is_active = 1 AND last_login < CURDATE() - INTERVAL 6 MONTH
  ) AS temp_table
);
```
Deactivates users who haven't logged in for 6+ months.

---

### 10. **Monthly User Sign-Up Trend**
```sql
SELECT YEAR(date_joined) AS signup_year,
       MONTH(date_joined) AS signup_month,
       COUNT(id) AS new_users
FROM users_customuser
GROUP BY YEAR(date_joined), MONTH(date_joined)
ORDER BY signup_year DESC, signup_month DESC;
```
Tracks number of user registrations per month.

---

### 11. **Users with Above-Average Savings**
```sql
SELECT u.id, u.email, SUM(s.amount) AS total_savings
FROM users_customuser u
JOIN savings_savingsaccount s ON u.id = s.id
GROUP BY u.id, u.email
HAVING total_savings > (
  SELECT AVG(total_savings) 
  FROM (
    SELECT SUM(amount) AS total_savings
    FROM savings_savingsaccount
    GROUP BY id
  ) AS avg_savings
);
```
Highlights users whose total savings are above average.

---

### 12. **High-Risk, Low-Savings Users**
```sql
SELECT u.id, u.email, u.risk_apetite, COALESCE(SUM(s.amount), 0) AS total_savings
FROM users_customuser u
LEFT JOIN savings_savingsaccount s ON u.id = s.id
WHERE u.risk_apetite > 2
GROUP BY u.id, u.email, u.risk_apetite
HAVING total_savings < 10000;
```
Detects users with high risk tolerance but low financial savings.

---

### 13. **Withdrawal-to-Savings Ratio**
```sql
SELECT u.id, u.email, COALESCE(SUM(w.amount), 0) AS total_withdrawals,
       COALESCE(SUM(s.amount), 0) AS total_savings,
       CASE 
         WHEN COALESCE(SUM(s.amount), 0) = 0 THEN 0 
         ELSE (SUM(w.amount) / SUM(s.amount)) * 100 
       END AS withdrawal_ratio_percent
FROM users_customuser u
LEFT JOIN savings_savingsaccount s ON u.id = s.id
LEFT JOIN withdrawals_withdrawal w ON u.id = w.id
GROUP BY u.id, u.email;
```
Measures how much users are withdrawing relative to their savings.

---

### 14. **Top Withdrawal Dates**
```sql
SELECT DATE(w.transaction_date) AS withdrawal_date,
       COUNT(w.id) AS total_withdrawals,
       SUM(w.amount) AS total_amount
FROM withdrawals_withdrawal w
GROUP BY withdrawal_date
ORDER BY total_withdrawals DESC
LIMIT 5;
```
Displays dates with the highest withdrawal activity.

---

### 15. **Individual Withdrawal Summary**
```sql
SELECT w.id, DATE(w.transaction_date) AS withdrawal_date,
       COUNT(w.id) AS total_withdrawals,
       SUM(w.amount) AS total_amount
FROM withdrawals_withdrawal w
GROUP BY w.id
ORDER BY total_withdrawals DESC
LIMIT 5;
```
Summarizes withdrawals per transaction ID.

---

### 16. **High-Value Users by Location**
```sql
SELECT u.address_country, u.address_city,
       COUNT(DISTINCT u.id) AS total_users,
       SUM(s.amount) AS total_savings
FROM users_customuser u
JOIN savings_savingsaccount s ON u.id = s.id
GROUP BY u.address_country, u.address_city
ORDER BY total_savings DESC;
```
Identifies top cities/countries with high savers.

---

### 17. **Recurring Withdrawals (>3 Months)**
```sql
SELECT u.id, u.email, YEAR(w.transaction_date) AS yr,
       MONTH(w.transaction_date) AS mnth,
       COUNT(w.id) AS withdrawals_count
FROM users_customuser u
JOIN withdrawals_withdrawal w ON u.id = w.owner_id
GROUP BY u.id, u.email, yr, mnth
HAVING withdrawals_count > 2;
```
Identifies users with frequent withdrawals over time.

### 18. **Running Total of Savings per User**
```sql
SELECT savings_id, transaction_date, amount AS daily_savings,
  SUM(amount) OVER (
    PARTITION BY savings_id 
    ORDER BY transaction_date 
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS cumulative_savings
FROM savings_savingsaccount
ORDER BY savings_id, transaction_date;
```
- Uses a window function to calculate cumulative savings over time for each user.
- Useful for trend analysis or chart plotting.

### 19. **Categorize Users by Risk Appetite**
```sql
SELECT id, email, risk_apetite,
  CASE 
    WHEN risk_apetite >= 7 THEN 'High Risk'
    WHEN risk_apetite BETWEEN 4 AND 6 THEN 'Medium Risk'
    ELSE 'Low Risk'
  END AS risk_category
FROM users_customuser;
```
- Classifies users into risk segments which can drive product targeting or advisory strategies.

### 20. **Savings Retention Within 30 Days of Signup**
```sql
WITH signups AS (
  SELECT id AS user_id, DATE(date_joined) AS signup_date FROM users_customuser
),
first_savings AS (
  SELECT owner_id, MIN(DATE(transaction_date)) AS first_saving_date
  FROM savings_savingsaccount GROUP BY owner_id
)
SELECT s.signup_date, COUNT(s.user_id) AS total_signups,
  COUNT(f.owner_id) AS users_with_savings,
  ROUND((COUNT(f.owner_id) * 100.0 / COUNT(s.user_id)), 2) AS retention_rate
FROM signups s
LEFT JOIN first_savings f ON s.user_id = f.owner_id
  AND f.first_saving_date <= s.signup_date + INTERVAL 30 DAY
GROUP BY s.signup_date
ORDER BY s.signup_date;
```
- Measures onboarding effectiveness by tracking how many users saved within 30 days of signup.
- Calculates a daily retention rate based on user behavior.

### 21. **Withdrawals Exceeding 30% of Total Savings**
```sql
WITH savings AS (
  SELECT owner_id, SUM(amount) AS total_savings FROM savings_savingsaccount GROUP BY owner_id
),
withdrawals AS (
  SELECT owner_id, SUM(amount) AS total_withdrawals FROM withdrawals_withdrawal GROUP BY owner_id
),
calculated_data AS ( 
  SELECT u.id, u.email, s.total_savings, w.total_withdrawals,
         (w.total_withdrawals * 100.0 / s.total_savings) AS withdrawal_percent
  FROM users_customuser u
  JOIN savings s ON u.id = s.owner_id
  JOIN withdrawals w ON u.id = w.owner_id
)
SELECT id, email, total_savings, total_withdrawals, withdrawal_percent
FROM calculated_data
WHERE withdrawal_percent > 30;
```
- Highlights users whose withdrawal behavior is unusually high relative to their total savings.
- This can signal liquidity issues or risk of churn.

---

## 🧠 Summary

These queries collectively helps to:
- Understand user lifecycle and behavior
- Track savings and withdrawals
- Identify risk and high-value customers
- Improve targeting, product planning, and user engagement

## 📘 Explanation

This section outlines the theoretical foundations and goals behind the SQL queries used in this analysis. These queries are designed to evaluate user behavior, financial engagement, and operational patterns within a digital financial platform.

### 1. User Activity Monitoring
Understanding user engagement is crucial. Queries targeting:
- **Recent sign-ups** (Query 1)
- **Last login updates** (Query 9)
- **Monthly signup trends** (Query 10)  
help organizations track growth and identify periods of high user acquisition or inactivity.

### 2. Savings Behavior Analysis
By analyzing how much users save (Queries 2, 6, 11, 12), the platform can:
- Segment users into different financial personas.
- Identify high-value savers for reward/retention campaigns.
- Spot users with high risk appetite but low savings, signaling potential financial distress or opportunity for education.

### 3. Withdrawal Behavior Insights
Withdrawal patterns reveal financial liquidity needs. These queries (Queries 3, 7, 13, 14, 15, 17):
- Identify users withdrawing frequently or in large volumes.
- Measure withdrawal-to-savings ratios to monitor cash-out behaviors.
- Detect recurring withdrawal behaviors, useful for subscription or loan risk management.

### 4. Plan and Product Utilization
By analyzing user participation in financial plans (Query 5), companies can:
- Determine which users adopt long-term financial strategies.
- Understand feature adoption rates.
- Optimize marketing efforts based on actual product use.

### 5. Demographic and Location-Based Segmentation
Location-based aggregation (Query 16) and gender-based salary insights (Query 4) enable:
- Geographic market analysis.
- Salary benchmarking by gender, promoting equity awareness.
- Tailored financial product offerings by region or demographic.

### 6. Data Quality and Relational Integrity
Use of joins and subqueries ensures:
- Accurate correlation between users and financial activities.
- Handling of missing data using `COALESCE`.
- Use of `LEFT JOIN` to preserve users even if they lack savings/withdrawals.

### 7. Operational Efficiency
Some queries, like updating inactive users (Query 9), serve direct administrative and operational purposes:
- Keeping data clean and relevant.
- Ensuring system resources focus on active users.

---

### 🔎 Conclusion
These queries form a robust framework for monitoring, segmenting, and managing users in a fintech or digital savings application. They align with key business questions like:
- Who are my most and least active users?
- How are users engaging financially?
- Where do my valuable users come from?
- Which users need engagement interventions?

By combining exploratory and prescriptive SQL queries, businesses can gain deep insights into their user base and make informed, data-driven decisions.
