# Bank_Analytics
## ●	Business Needs:
### Using data to improve business operations, manage risk, and make strategic decisions. 
## ●	Tools used:
### MySQL, Excel, PowerBI.
## ●	Approach:
### Explored data from bank customers in Excel to uncover meaningful trends in loan disbursement.
### Distinguishing based on loan per year and interest rates.
### Tracked key performance metrics with MySQL queries including Total Loan Amount, Loan Count, Cumulative Interest Earned, and Defaulted Loan Count to evaluate loan portfolio health. 
In the banking sector, "real-time" analytics usually refers to **Current Month-to-Date (MTD)** and **Year-to-Date (YTD)** metrics.

Here are the MySQL formulas to generate the specific reports you requested.

---

## 1. State-wise & Month-wise Loan Status

This query provides a matrix of how many loans are in each status (e.g., Fully Paid, Default, Current) broken down by state and month.

```sql
SELECT 
    address_state,
    MONTHNAME(issue_date) AS month_name,
    loan_status,
    COUNT(id) AS total_applications,
    SUM(loan_amount) AS total_funded_amount
FROM bank_loan_data
WHERE YEAR(issue_date) = YEAR(CURDATE()) -- Real-time: Focuses on current year
GROUP BY address_state, MONTH(issue_date), month_name, loan_status
ORDER BY MONTH(issue_date), address_state;

```

## 2. Region-wise Distribution

Most banks map states to regions (North, South, etc.) via a `CASE` statement if a separate regions table isn't available.

```sql
SELECT 
    CASE 
        WHEN address_state IN ('NY', 'NJ', 'PA') THEN 'Northeast'
        WHEN address_state IN ('CA', 'WA', 'OR') THEN 'West'
        WHEN address_state IN ('TX', 'FL', 'GA') THEN 'South'
        ELSE 'Other/Central'
    END AS region,
    COUNT(id) AS total_loans,
    SUM(loan_amount) AS total_volume
FROM bank_loan_data
GROUP BY region
ORDER BY total_volume DESC;

```

## 3. Top 10 States by Loan Volume

This is a standard "Leaderboard" query used to identify high-performing geographical markets.

```sql
SELECT 
    address_state,
    COUNT(id) AS total_loan_applications,
    SUM(loan_amount) AS total_funded_amount,
    SUM(total_payment) AS total_amount_received
FROM bank_loan_data
GROUP BY address_state
ORDER BY total_funded_amount DESC
LIMIT 10;

```

## 4. Year-wise Disbursement Trends

This shows the long-term growth of the bank's lending activities.

```sql
SELECT 
    YEAR(issue_date) AS disbursement_year,
    COUNT(id) AS total_loans_issued,
    SUM(loan_amount) AS total_disbursed_amount,
    AVG(int_rate) AS avg_interest_rate
FROM bank_loan_data
GROUP BY YEAR(issue_date)
ORDER BY disbursement_year DESC;

```

---

### Pro-Tips for "Real-Time" Implementation:

Since it is currently **February 2026**, comparing current performance against the same period in 2025 is the most critical "real-time" insight for a bank analyst. This removes seasonality bias (e.g., comparing a busy December to a slow January).

Here is the SQL logic to calculate **Year-to-Date (YTD)** and **Previous Year-to-Date (PYTD)** to find your growth percentage.

---

## 1. YTD vs. PYTD Comparison

This query calculates the total loan volume for the current year (up to today) and compares it to the exact same date range from the previous year.

```sql
SELECT 
    -- Current YTD (Jan 1, 2026 - Feb 10, 2026)
    SUM(CASE WHEN YEAR(issue_date) = YEAR(CURDATE()) AND issue_date <= CURDATE() 
             THEN loan_amount ELSE 0 END) AS YTD_Volume,
    
    -- Previous YTD (Jan 1, 2025 - Feb 10, 2025)
    SUM(CASE WHEN YEAR(issue_date) = YEAR(CURDATE()) - 1 AND issue_date <= DATE_SUB(CURDATE(), INTERVAL 1 YEAR) 
             THEN loan_amount ELSE 0 END) AS PYTD_Volume,
             
    -- Growth Percentage Formula
    ((SUM(CASE WHEN YEAR(issue_date) = YEAR(CURDATE()) AND issue_date <= CURDATE() THEN loan_amount ELSE 0 END) - 
      SUM(CASE WHEN YEAR(issue_date) = YEAR(CURDATE()) - 1 AND issue_date <= DATE_SUB(CURDATE(), INTERVAL 1 YEAR) THEN loan_amount ELSE 0 END)) / 
      SUM(CASE WHEN YEAR(issue_date) = YEAR(CURDATE()) - 1 AND issue_date <= DATE_SUB(CURDATE(), INTERVAL 1 YEAR) THEN loan_amount ELSE 0 END)) * 100 AS YoY_Growth_Pct
FROM bank_loan_data;

```

---

## 2. Advanced Performance Tuning: Covering Indexes

When you frequently run "State + Month + Status" reports, a standard index is good, but a **Covering Index** is better. A covering index includes all the columns requested in the `SELECT` and `WHERE` clauses, meaning MySQL never has to look at the actual table rows—it gets everything from the index memory.

**The Optimal Index for your Dashboard:**

```sql
CREATE INDEX idx_dashboard_optimized 
ON bank_loan_data (issue_date, address_state, loan_status, loan_amount);

```

---

## 3. Real-Time Logic for "Month-to-Date" (MTD)

---

### Summary Table for Quick Reference

If you're building a view for your BI tool (Power BI/Tableau), use this structure to feed all your charts at once:

```sql
CREATE OR REPLACE VIEW v_loan_realtime_metrics AS
SELECT 
    address_state,
    loan_status,
    COUNT(id) AS loan_count,
    SUM(loan_amount) AS total_amount,
    AVG(int_rate) AS avg_int_rate,
    -- Identifies if the record belongs to the current month for easy filtering in BI
    IF(MONTH(issue_date) = MONTH(CURDATE()) AND YEAR(issue_date) = YEAR(CURDATE()), 1, 0) AS is_mtd
FROM bank_loan_data
GROUP BY address_state, loan_status, is_mtd;

```



## ●	Conclusion:
### Built dynamic dashboards using PowerBI visualize state-wise and month-wise loan status, region wise distribution, top 10 states by loan volume, and year-wise disbursement trends enabling real-time.
