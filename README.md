# 📊 SQL JOINs & Window Functions Project

**Course:** Database Development with PL/SQL (INSY 8311)  
**Instructor:** Eric Maniraguha | eric.maniraguha@auca.ac.rw  
**Student:** [Your Full Name] | [Student ID]  
**Group:** [A / B / C / D]  
**Submission Date:** [Date]

---

## 📌 Business Problem

### Business Context
[Describe the type of company, department, and industry — e.g., "A regional e-commerce retailer operating across East Africa with a dedicated sales and marketing department."]

### Data Challenge
[Write 2–3 sentences explaining the data problem — e.g., "The business lacks visibility into which products drive revenue across different regions. Without systematic analysis, the marketing team cannot identify high-value customers or track sales trends over time. This project addresses these gaps using relational SQL queries and analytical window functions."]

### Expected Outcome
[State the decision or insight expected — e.g., "Identify the top 5 products per region, segment customers into spending quartiles, and detect month-over-month sales growth to guide targeted marketing campaigns."]

---

## 🎯 Success Criteria

| # | Goal | Window Function |
|---|------|-----------------|
| 1 | Rank top 5 products per region or quarter | `RANK()` |
| 2 | Calculate running monthly sales totals | `SUM() OVER()` |
| 3 | Measure month-over-month revenue growth | `LAG()` / `LEAD()` |
| 4 | Segment customers into spending quartiles | `NTILE(4)` |
| 5 | Compute three-month moving average of sales | `AVG() OVER()` |

---

## 🗄️ Database Schema

### Tables

**1. Customers**
| Column | Type | Description |
|--------|------|-------------|
| customer_id | INT (PK) | Unique customer identifier |
| customer_name | VARCHAR | Full name of the customer |
| region | VARCHAR | Geographic region |
| email | VARCHAR | Contact email |
| join_date | DATE | Date customer registered |

**2. Products**
| Column | Type | Description |
|--------|------|-------------|
| product_id | INT (PK) | Unique product identifier |
| product_name | VARCHAR | Name of the product |
| category | VARCHAR | Product category |
| unit_price | DECIMAL | Price per unit |

**3. Transactions**
| Column | Type | Description |
|--------|------|-------------|
| transaction_id | INT (PK) | Unique transaction identifier |
| customer_id | INT (FK) | References Customers |
| product_id | INT (FK) | References Products |
| quantity | INT | Number of units sold |
| transaction_date | DATE | Date of transaction |
| total_amount | DECIMAL | Total sale value |

### ER Diagram

```
CUSTOMERS                TRANSACTIONS              PRODUCTS
+-------------+          +------------------+       +-------------+
| customer_id |◄────────►| transaction_id   |◄─────►| product_id  |
| name        |  1    M  | customer_id (FK) |  M  1 | name        |
| region      |          | product_id (FK)  |       | category    |
| email       |          | quantity         |       | unit_price  |
| join_date   |          | transaction_date |       +-------------+
+-------------+          | total_amount     |
                         +------------------+
```

> 📎 *[Insert your ER diagram image here]*

---

## 🔗 Part A — SQL JOINs Implementation

### 1. INNER JOIN — Transactions with Valid Customers and Products

```sql
-- Retrieve all transactions that have matching customer and product records
SELECT 
    t.transaction_id,
    c.customer_name,
    c.region,
    p.product_name,
    t.quantity,
    t.total_amount,
    t.transaction_date
FROM Transactions t
INNER JOIN Customers c ON t.customer_id = c.customer_id
INNER JOIN Products p  ON t.product_id  = p.product_id
ORDER BY t.transaction_date DESC;
```

**Screenshot:**  
> 📸 *[Insert screenshot here]*

**Business Interpretation:**  
This query returns only transactions where both the customer and product exist in the database. It provides a clean view of confirmed sales activity and forms the foundation for revenue reporting.

---

### 2. LEFT JOIN — Customers Who Have Never Made a Transaction

```sql
-- Identify registered customers with no purchase history
SELECT 
    c.customer_id,
    c.customer_name,
    c.region,
    c.email,
    t.transaction_id
FROM Customers c
LEFT JOIN Transactions t ON c.customer_id = t.customer_id
WHERE t.transaction_id IS NULL
ORDER BY c.customer_name;
```

**Screenshot:**  
> 📸 *[Insert screenshot here]*

**Business Interpretation:**  
Customers returned with NULL transaction values have registered but never purchased. This information is critical for re-engagement campaigns and helps the marketing team target dormant accounts with personalized incentives.

---

### 3. RIGHT JOIN — Products with No Sales Activity

```sql
-- Detect products that have never appeared in any transaction
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.unit_price,
    t.transaction_id
FROM Transactions t
RIGHT JOIN Products p ON t.product_id = p.product_id
WHERE t.transaction_id IS NULL
ORDER BY p.category;
```

**Screenshot:**  
> 📸 *[Insert screenshot here]*

**Business Interpretation:**  
Products with no associated transactions represent zero-revenue inventory. These items may need promotional pricing, repositioning, or removal from the catalog to optimize stock management and reduce carrying costs.

---

### 4. FULL OUTER JOIN — Compare All Customers and Products Including Unmatched Records

```sql
-- Compare customers and products, exposing records with no match on either side
SELECT 
    c.customer_id,
    c.customer_name,
    p.product_id,
    p.product_name,
    t.total_amount
FROM Customers c
FULL OUTER JOIN Transactions t ON c.customer_id = t.customer_id
FULL OUTER JOIN Products p     ON t.product_id  = p.product_id
ORDER BY c.customer_id, p.product_id;
```

**Screenshot:**  
> 📸 *[Insert screenshot here]*

**Business Interpretation:**  
A FULL OUTER JOIN surfaces all gaps in the data — inactive customers and unsold products alike. This holistic view supports data quality audits and ensures that no business entity falls through the cracks of standard reporting.

---

### 5. SELF JOIN — Compare Customers Within the Same Region

```sql
-- Identify pairs of customers from the same region for regional benchmarking
SELECT 
    a.customer_name  AS customer_a,
    b.customer_name  AS customer_b,
    a.region
FROM Customers a
INNER JOIN Customers b 
    ON  a.region      = b.region
    AND a.customer_id < b.customer_id
ORDER BY a.region, a.customer_name;
```

**Screenshot:**  
> 📸 *[Insert screenshot here]*

**Business Interpretation:**  
By joining the Customers table to itself on region, we can identify peer customers sharing the same market. This enables regional sales comparisons, peer benchmarking, and targeted group promotions within geographic segments.

---

## 📈 Part B — Window Functions Implementation

### Category 1: Ranking Functions

```sql
-- Rank customers by total revenue using ROW_NUMBER, RANK, DENSE_RANK, and PERCENT_RANK
SELECT 
    c.customer_name,
    c.region,
    SUM(t.total_amount)                                          AS total_revenue,
    ROW_NUMBER()  OVER (ORDER BY SUM(t.total_amount) DESC)      AS row_num,
    RANK()        OVER (ORDER BY SUM(t.total_amount) DESC)      AS rank_position,
    DENSE_RANK()  OVER (ORDER BY SUM(t.total_amount) DESC)      AS dense_rank,
    ROUND(
        PERCENT_RANK() OVER (ORDER BY SUM(t.total_amount) DESC) * 100, 2
    )                                                            AS percent_rank
FROM Transactions t
INNER JOIN Customers c ON t.customer_id = c.customer_id
GROUP BY c.customer_id, c.customer_name, c.region
ORDER BY total_revenue DESC;
```

**Screenshot:**  
> 📸 *[Insert screenshot here]*

**Interpretation:**  
`RANK()` assigns the same position to customers with equal revenue, while `DENSE_RANK()` avoids gaps in ranking. `PERCENT_RANK()` reveals where each customer falls in the percentile distribution, enabling the business to identify its top 10% of revenue contributors.

---

### Category 2: Aggregate Window Functions

```sql
-- Running monthly sales total (ROWS frame) and monthly average (RANGE frame)
SELECT 
    TO_CHAR(t.transaction_date, 'YYYY-MM')             AS month,
    SUM(t.total_amount)                                AS monthly_sales,
    SUM(SUM(t.total_amount)) OVER (
        ORDER BY TO_CHAR(t.transaction_date, 'YYYY-MM')
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )                                                  AS running_total,
    AVG(SUM(t.total_amount)) OVER (
        ORDER BY TO_CHAR(t.transaction_date, 'YYYY-MM')
        RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )                                                  AS cumulative_avg
FROM Transactions t
GROUP BY TO_CHAR(t.transaction_date, 'YYYY-MM')
ORDER BY month;
```

**Screenshot:**  
> 📸 *[Insert screenshot here]*

**Interpretation:**  
The running total (`ROWS` frame) accumulates revenue month by month, clearly showing growth momentum. The cumulative average (`RANGE` frame) smooths out seasonal fluctuations, giving management a stable long-term benchmark for monthly performance.

---

### Category 3: Navigation Functions — LAG & LEAD

```sql
-- Month-over-month revenue growth using LAG and LEAD
SELECT 
    TO_CHAR(t.transaction_date, 'YYYY-MM')             AS month,
    SUM(t.total_amount)                                AS current_month_sales,
    LAG(SUM(t.total_amount))  OVER (
        ORDER BY TO_CHAR(t.transaction_date, 'YYYY-MM')
    )                                                  AS previous_month_sales,
    LEAD(SUM(t.total_amount)) OVER (
        ORDER BY TO_CHAR(t.transaction_date, 'YYYY-MM')
    )                                                  AS next_month_sales,
    ROUND(
        (SUM(t.total_amount) - LAG(SUM(t.total_amount)) OVER (
            ORDER BY TO_CHAR(t.transaction_date, 'YYYY-MM')
        )) / NULLIF(LAG(SUM(t.total_amount)) OVER (
            ORDER BY TO_CHAR(t.transaction_date, 'YYYY-MM')
        ), 0) * 100, 2
    )                                                  AS mom_growth_pct
FROM Transactions t
GROUP BY TO_CHAR(t.transaction_date, 'YYYY-MM')
ORDER BY month;
```

**Screenshot:**  
> 📸 *[Insert screenshot here]*

**Interpretation:**  
`LAG()` retrieves the previous month's revenue, enabling direct comparison and growth rate calculation. A negative `mom_growth_pct` signals a sales decline and prompts investigation, while `LEAD()` allows the analyst to preview the next period's expected trend.

---

### Category 4: Distribution Functions — NTILE & CUME_DIST

```sql
-- Segment customers into spending quartiles and compute cumulative distribution
SELECT 
    c.customer_name,
    c.region,
    SUM(t.total_amount)                                          AS total_spent,
    NTILE(4)     OVER (ORDER BY SUM(t.total_amount) DESC)       AS spending_quartile,
    ROUND(
        CUME_DIST() OVER (ORDER BY SUM(t.total_amount) ASC) * 100, 2
    )                                                            AS cume_dist_pct
FROM Transactions t
INNER JOIN Customers c ON t.customer_id = c.customer_id
GROUP BY c.customer_id, c.customer_name, c.region
ORDER BY total_spent DESC;
```

**Screenshot:**  
> 📸 *[Insert screenshot here]*

**Interpretation:**  
`NTILE(4)` divides customers into four equal groups by spending, with Quartile 1 representing the top 25% of spenders — prime candidates for loyalty rewards. `CUME_DIST()` shows what percentage of customers each individual outspends, supporting fine-grained tiered marketing strategies.

---

## 🔍 Results Analysis

### 1. Descriptive — What Happened?
[Summarize the key findings from your queries — e.g., "Region X generated the highest cumulative revenue over the analysis period. Product category Y accounted for 40% of total sales. Approximately 15% of registered customers had no transaction history."]

### 2. Diagnostic — Why Did It Happen?
[Explain the causes behind the patterns observed — e.g., "The strong performance in Region X correlates with a higher population density and a targeted promotional campaign run in Q3. Zero-transaction customers likely registered via a referral program but did not complete a purchase."]

### 3. Prescriptive — What Should Be Done Next?
[Recommend actionable steps — e.g., "Launch a re-engagement email campaign targeting the 15% of inactive customers. Introduce bundle promotions for zero-revenue products. Replicate the Region X marketing strategy in Region Z, which has a comparable demographic profile."]

---

## 📚 References

1. Oracle Corporation. (2024). *SQL Window Functions Documentation*. https://docs.oracle.com/en/database/oracle/oracle-database/
2. PostgreSQL Global Development Group. (2024). *Window Function Calls*. https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS
3. Microsoft. (2024). *OVER Clause (Transact-SQL)*. https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql
4. Silberschatz, A., Korth, H. F., & Sudarshan, S. (2020). *Database System Concepts* (7th ed.). McGraw-Hill.
5. [Add any additional tutorials, academic papers, or business resources consulted]

---

## ✅ Integrity Statement

> *"All sources were properly cited. Implementations and analysis represent original work. No AI-generated content was copied without attribution or adaptation."*

---

## 📁 Repository Structure

```
plsql_window_functions_[studentId]_[firstname]/
│
├── README.md                    # This document
├── schema/
│   ├── create_tables.sql        # DDL scripts
│   ├── insert_data.sql          # Sample data
│   └── er_diagram.png           # ER diagram image
│
├── part_a_joins/
│   ├── 01_inner_join.sql
│   ├── 02_left_join.sql
│   ├── 03_right_join.sql
│   ├── 04_full_outer_join.sql
│   └── 05_self_join.sql
│
├── part_b_window_functions/
│   ├── 01_ranking_functions.sql
│   ├── 02_aggregate_window.sql
│   ├── 03_navigation_functions.sql
│   └── 04_distribution_functions.sql
│
└── screenshots/
    ├── joins/
    └── window_functions/
```
