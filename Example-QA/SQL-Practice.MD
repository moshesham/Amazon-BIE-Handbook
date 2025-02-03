## SQL and Redshift Interview Questions and Answers

---
> **Question 1**
> Write a SQL query to find the month-over-month percentage change in revenue for each product. Assume you have a table named `Orders` with columns `order_date`, `product_id`, and `revenue`.

```sql
WITH MonthlyRevenue AS (
    SELECT
        strftime('%Y-%m', order_date) AS order_month,
        product_id,
        SUM(revenue) AS monthly_revenue
    FROM
        Orders
    GROUP BY
        order_month,
        product_id
),
LaggedRevenue AS (
    SELECT
        order_month,
        product_id,
        monthly_revenue,
        LAG(monthly_revenue, 1, 0) OVER (PARTITION BY product_id ORDER BY order_month) AS previous_month_revenue
    FROM
        MonthlyRevenue
)
SELECT
    order_month,
    product_id,
    monthly_revenue,
    previous_month_revenue,
    (monthly_revenue - previous_month_revenue) * 100.0 / previous_month_revenue AS month_over_month_change
FROM
    LaggedRevenue
ORDER BY
    product_id,
    order_month;
```

> **Answer:**
> This query calculates the month-over-month percentage change in revenue for each product.
> 1. **MonthlyRevenue CTE:** Calculates the total revenue for each product per month.
> 2. **LaggedRevenue CTE:** Uses the `LAG` window function to get the revenue from the previous month for each product. The `PARTITION BY` clause ensures that the lag is calculated separately for each product.
> 3. **Final SELECT Statement:** Calculates the percentage change using the formula: `(current_month - previous_month) * 100 / previous_month`. The result is ordered by product and month.
---
> **Question 2**
> You have a table `website_visits` with columns `user_id`, `visit_date`, and `page_url`. Write a query to find the top 5 most visited pages for each user, along with the number of visits.

```sql
WITH RankedVisits AS (
    SELECT
        user_id,
        page_url,
        COUNT(*) AS visit_count,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY COUNT(*) DESC) as rn
    FROM
        website_visits
    GROUP BY
        user_id,
        page_url
)
SELECT
    user_id,
    page_url,
    visit_count
FROM
    RankedVisits
WHERE
    rn <= 5
ORDER BY
 user_id, visit_count DESC;
```

> **Answer:**
> This query determines the top 5 most visited pages per user.
> 1. **RankedVisits CTE:** Ranks pages for each user based on visit count using `ROW_NUMBER`.
> 2. **Final SELECT Statement:** Filters the results to include only the top 5 pages for each user. The result is ordered by user id and then the count of page visits in descending order.

---
> **Question 3**
> Given a table `employees` with columns `employee_id`, `department_id`, `salary`, and `manager_id`, write a query to find the top 3 highest-paid employees in each department.

```sql
WITH RankedEmployees AS (
    SELECT
        employee_id,
        department_id,
        salary,
        DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) as rank
    FROM
        employees
)
SELECT
    employee_id,
    department_id,
    salary
FROM
    RankedEmployees
WHERE
    rank <= 3
ORDER BY
    department_id, salary DESC;
```

> **Answer:**
> This query finds the top 3 highest-paid employees in each department.
> 1. **RankedEmployees CTE:** Ranks employees within each department based on salary using `DENSE_RANK`. `DENSE_RANK` assigns the same rank to employees with the same salary and does not skip ranks.
> 2. **Final SELECT Statement:** Filters for employees ranked within the top 3 in their respective departments. The results are ordered by department then salary in descending order to show the highest paid first.

---
> **Question 4**
> You have a table `transactions` with columns `transaction_id`, `user_id`, `transaction_date`, and `amount`. Write a query to find, for each user, the date of their first transaction and the date of their latest transaction.

```sql
SELECT
    user_id,
    MIN(transaction_date) AS first_transaction_date,
    MAX(transaction_date) AS latest_transaction_date
FROM
    transactions
GROUP BY
    user_id;
```

> **Answer:**
> This query finds the first and latest transaction dates for each user.
> 1. The `MIN` and `MAX` aggregate functions are used with `GROUP BY` to find the earliest and latest transaction dates per user.

---
> **Question 5**
> A table `events` has columns `event_id`, `event_type`, `start_time`, and `end_time`. Write a query to detect overlapping events for the same event type.

```sql
SELECT
    e1.event_id,
    e2.event_id
FROM
    events e1
JOIN
    events e2 ON e1.event_type = e2.event_type
    AND e1.event_id < e2.event_id
    AND e1.end_time > e2.start_time
    AND e1.start_time < e2.end_time;
```

> **Answer:**
> This query identifies overlapping events.
> 1. It uses a self-join on the `events` table, comparing each event with every other event of the same type.
> 2. The join conditions check for overlapping time ranges. `e1.event_id < e2.event_id` ensures we don't compare an event with itself or report duplicates.

---
> **Question 6**
> Assume a table `products` with columns `product_id`, `category`, and `price`. Write a query to categorize products into 'High', 'Medium', and 'Low' price categories based on their price within each category.

```sql
WITH RankedProducts AS (
    SELECT
        product_id,
        category,
        price,
        NTILE(3) OVER (PARTITION BY category ORDER BY price DESC) as price_rank
    FROM
        products
)
SELECT
    product_id,
    category,
    price,
    CASE
        WHEN price_rank = 1 THEN 'High'
        WHEN price_rank = 2 THEN 'Medium'
        WHEN price_rank = 3 THEN 'Low'
    END as price_category
FROM
    RankedProducts;
```

> **Answer:**
> This query categorizes products into price tiers within each category.
> 1. **RankedProducts CTE:** Uses `NTILE(3)` to divide products in each category into three groups based on price.
> 2. **Final SELECT Statement:** Assigns 'High', 'Medium', or 'Low' labels based on the `price_rank`.

---
> **Question 7**
> Given a `sales` table with columns `sale_id`, `sale_date`, and `amount`, write a query to calculate the 7-day rolling average of sales amount for each day.

```sql
SELECT
    sale_date,
    AVG(amount) OVER (ORDER BY sale_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as rolling_average
FROM
    sales
ORDER BY
    sale_date;
```

> **Answer:**
> This query calculates the 7-day rolling average of sales.
> 1. The `AVG` window function with the `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` frame calculates the average over the preceding 6 days and the current day.

---
> **Question 8**
> A table `logs` has columns `log_id`, `user_id`, and `action_time`. Write a query to find users who have performed at least 3 actions within a 5-minute window.

```sql
SELECT DISTINCT
    user_id
FROM
    (
        SELECT
            user_id,
            action_time,
            COUNT(*) OVER (PARTITION BY user_id ORDER BY action_time RANGE BETWEEN INTERVAL '5 minutes' PRECEDING AND CURRENT ROW) as action_count
        FROM
            logs
    ) as ActionCounts
WHERE
    action_count >= 3;
```

> **Answer:**
> This query identifies users with at least 3 actions within a 5-minute window.
> 1. It uses a subquery to count actions within a 5-minute window for each user using the `COUNT` window function with a `RANGE` frame.
> 2. The outer query filters for users with an `action_count` of 3 or more.
---
> **Question 9**
> Assume a table `orders` with columns `order_id`, `customer_id`, `order_date`, and `amount`. Write a query to rank customers by their total spending per month.

```sql
WITH MonthlySpending AS (
    SELECT
        strftime('%Y-%m', order_date) AS order_month,
        customer_id,
        SUM(amount) AS total_spending
    FROM
        orders
    GROUP BY
        order_month,
        customer_id
)
SELECT
    order_month,
    customer_id,
    total_spending,
    RANK() OVER (PARTITION BY order_month ORDER BY total_spending DESC) as spending_rank
FROM
    MonthlySpending
ORDER BY
    order_month, spending_rank;
```

> **Answer:**
> This query ranks customers by their monthly spending.
> 1. **MonthlySpending CTE:** Calculates the total spending per customer for each month.
> 2. **Final SELECT Statement:** Ranks customers within each month based on `total_spending` using the `RANK` window function. The result is ordered by month and spending rank.

---
> **Question 10**
> You have a table `tasks` with columns `task_id`, `assigned_to`, and `completion_date`. Write a query to find tasks that were completed on the same day they were assigned.

```sql
SELECT
    task_id
FROM
    tasks
WHERE
    DATE(assigned_to) = DATE(completion_date);
```

> **Answer:**
> This query identifies tasks completed on the same day they were assigned.
>1. It filters the `tasks` table where the `DATE` of `assigned_to` is the same as the `DATE` of `completion_date`.

---
> **Question 11**
> How would you optimize a query in Redshift that joins a large fact table with a small dimension table on a specific key?

> **Answer:**
> To optimize a join between a large fact table and a small dimension table in Redshift:
> 1. **Distribution Style:** Ensure the fact table has a `KEY` distribution style based on the join key, and the dimension table has an `ALL` distribution style. This minimizes data shuffling across nodes.
> 2. **Sort Keys:** Define the join key as the sort key for both tables. This speeds up the join operation as data is pre-sorted.
> 3. **Analyze Query Plan:** Use the `EXPLAIN` command to analyze the query plan and identify bottlenecks. Look for areas where data is being redistributed unnecessarily or where sequential scans are being performed instead of index scans.

---
> **Question 12**
> What is the purpose of the `VACUUM` command in Redshift, and when should you run it?

> **Answer:**
> The `VACUUM` command in Redshift reclaims space occupied by deleted or updated rows and resorts data to maintain sort key order.
>
> *   **Purpose:**
>     *   Reclaims storage space.
>     *   Restores sort order, improving query performance.
> *   **When to Run:**
>     *   After large deletes or updates.
>     *   Regularly, as part of maintenance, especially if tables have frequent updates or deletes.
>     *   When performance degrades due to unsorted regions.
>     *   Ideally, run during off-peak hours as it can be resource-intensive. There are different types of `VACUUM` such as `VACUUM FULL`, `VACUUM DELETE ONLY`, `VACUUM SORT ONLY`, and `VACUUM REINDEX` you should understand each of these.

---
> **Question 13**
> Explain the difference between `KEY` and `EVEN` distribution styles in Redshift and when you would choose one over the other.

> **Answer:**
> *   **EVEN Distribution:** Distributes data evenly across all slices, regardless of the data values. Use this when a table is not frequently joined or when there's no obvious key to distribute on.
> *   **KEY Distribution:** Distributes data based on the values in a specified column. Rows with the same value in the distribution key column will be stored on the same slice. Choose this when a table is frequently joined on a specific key, especially large fact tables joined with dimension tables. This minimizes data shuffling during joins, improving performance.
> *   **Choosing Between Them:**
>     *   Use `KEY` distribution for large tables that are frequently joined on a specific column.
>     *   Use `EVEN` distribution for tables that are not often joined or when there is no clear distribution key, or when data skew might be a major issue with `KEY` distribution.

---
> **Question 14**
> How does the `ANALYZE` command improve query performance in Redshift?

> **Answer:**
> The `ANALYZE` command updates the statistics metadata that the query planner uses to create optimized query plans.
>
> *   **How it Improves Performance:**
>     *   Provides the query planner with up-to-date information about table data distribution and size.
>     *   Helps the query planner choose the most efficient plan (e.g., choosing the right join order, selecting the best distribution style for intermediate results).
> *   **When to Run:**
>     *   After loading significant amounts of data.
>     *   Periodically, as part of regular maintenance.

---
> **Question 15**
> Describe how you would handle loading data into Redshift from S3, ensuring data integrity and efficiency.

> **Answer:**
> To load data from S3 to Redshift efficiently and with data integrity:
> 1. **Use the `COPY` Command:** The `COPY` command is the most efficient way to load data into Redshift from S3.
> 2. **Data Format:** Use a compressed, columnar format like Parquet or ORC for faster loading. If using CSV, ensure it's properly formatted and compressed (e.g., Gzip).
> 3. **Parallel Loading:** Split large files into multiple, roughly equal-sized files (between 1MB and 1GB after compression) to leverage parallel loading across multiple slices. The number of files should be a multiple of the number of slices in your Redshift cluster.
> 4. **Manifest File:** Use a manifest file to specify the list of files to load, ensuring all files are loaded and in the correct order.
> 5. **Validation:** Use the `NOLOAD` option to test the `COPY` command without actually loading data, checking for errors.
> 6. **Error Handling:** Use the `MAXERROR` option to allow a certain number of errors before failing the load. Review error logs in `STL_LOAD_ERRORS` to identify and fix data issues.
> 7. **Data Validation:** After loading, run validation checks to ensure data integrity (e.g., row counts, checksums, comparison with source data).

---
> **Question 16**
> What are the different sort key types in Redshift, and how do you choose the appropriate type for a table?

> **Answer:**
> Redshift offers two types of sort keys:
>
> *   **Compound Sort Key:**  Made up of multiple columns, listed in the order you define them in the `CREATE TABLE` statement. It's like sorting by the first column, then by the second column within the first, and so on. This is the default and generally recommended sort key type.
>     *   **Use Case:** Most efficient when queries often filter on columns in the order they are defined in the sort key.
> *   **Interleaved Sort Key:** Gives equal weight to each column in the sort key. This can be more efficient when queries filter on multiple columns, but it's less efficient than compound for queries filtering on just the first column of the sort key.
>     *   **Use Case:** Useful when you don't have a common pattern in filtering columns, or when you need to filter on different combinations of columns frequently. Can be significantly slower than compound sort keys for large tables and complex queries.
>
> **Choosing the Right Type:**
>
> *   **Analyze Query Patterns:** Understand how your data will be queried. If most queries filter on a specific set of columns in a specific order, a compound sort key is likely best.
> *   **Consider Data Skew:** If some columns have significantly more distinct values than others, an interleaved sort key might distribute the data more evenly.
> *   **Benchmark:** Test different sort key types and configurations with your actual workload to determine the best performance.
> *   **Generally, start with a compound sort key unless you have a specific reason to use an interleaved sort key.** Interleaved sort keys also take longer to `VACUUM`.

---
> **Question 17**
> Explain how distribution styles impact query performance when joining tables in Redshift.

> **Answer:**
> Distribution styles significantly affect query performance during joins by determining where data resides across the cluster's slices.
>
> *   **KEY Distribution:** When joining two tables with `KEY` distribution on the join key, matching keys reside on the same slice. This allows for local joins, which are very fast as they don't require data transfer across the network.
> *   **EVEN Distribution:** If tables have `EVEN` distribution or are joined on columns other than their distribution keys, Redshift will likely need to redistribute data across the network to perform the join. This can be slow, especially for large tables.
> *   **ALL Distribution:**  If a small dimension table has `ALL` distribution, a copy of the entire table exists on each slice. This can be efficient when joining with a large fact table, as the dimension table is locally available on each slice. However, it increases storage usage.
>
> **Impact:**
>
> *   Joins between tables with `KEY` distribution on the join key are the fastest.
> *   Joins involving `EVEN` distribution or mismatched `KEY` distribution often require data redistribution, leading to slower performance.
> *   Using `ALL` distribution for small dimension tables can speed up joins with large fact tables.

---
> **Question 18**
> What is data skew, and how can it impact Redshift performance? How can you mitigate it?

> **Answer:**
> Data skew occurs when data is unevenly distributed across slices, often due to a poor choice of distribution key or non-uniform data values.
>
> **Impact on Performance:**
>
> *   **Uneven Workload:** Some slices may have significantly more data to process than others, leading to longer processing times for those slices and overall slower query performance.
> *   **Resource Bottlenecks:** Slices with more data may become resource bottlenecks, impacting other queries running on the cluster.
>
> **Mitigation Strategies:**
>
> *   **Choose Appropriate Distribution Key:** Select a distribution key that distributes data evenly, avoiding columns with many repeated values or NULLs.
> *   **Use EVEN Distribution:** If no suitable distribution key can be found, use `EVEN` distribution.
> *   **Data Preprocessing:** In some cases, you can preprocess the data to reduce skew (e.g., splitting a large group into smaller groups).
> *   **Monitor for Skew:** Regularly monitor for skew using system tables like `SVV_DISKUSAGE` and adjust distribution keys if necessary.

---
> **Question 19**
> How would you troubleshoot a slow-running query in Redshift?

> **Answer:**
> To troubleshoot a slow-running query in Redshift:
> 1. **`EXPLAIN` Plan:** Use the `EXPLAIN` command to analyze the query plan. Look for:
    *   **Costly operations:** Identify the most expensive steps in the plan.
    *   **Data redistribution:** Check for excessive data shuffling between nodes (e.g., `DS_DIST_INNER`, `DS_DIST_OUTER`).
    *   **Sequential scans:** Look for table scans instead of index scans.
    *   **Nested loop joins:** These can be slow, especially if large tables are involved.
2. **System Tables:** Query system tables for performance insights:
    *   **`STL_QUERY`:**  Provides information about completed queries.
    *   **`STL_LOAD_ERRORS`:** Contains details about errors encountered during data loads.
    *   **`SVV_TABLE_INFO`:** Shows table metadata, including statistics about data skew and unsorted regions.
    *   **`SVV_DISKUSAGE`:**  Provides information about disk space usage per slice, helping identify data skew.
    *   **`STV_EXEC_STATE`:** Contains execution state information
3. **Query Monitoring Rules (WLM):** Use WLM query monitoring rules to identify and terminate long-running queries automatically.
4. **Consider Query Queues:** Check if the query is waiting in a queue for a long time. If so, you might need to adjust WLM queue configurations.
5. **Optimize:**
    *   **Review Distribution and Sort Keys:** Ensure they are appropriate for the query patterns.
    *   **Rewrite the Query:** Try to simplify the query, use more efficient joins, or break it down into smaller steps.
    *   **`VACUUM` and `ANALYZE`:** Make sure the tables involved are properly vacuumed and analyzed.

---
> **Question 20**
> How can you secure sensitive data stored in Redshift?

> **Answer:**
> To secure sensitive data in Redshift:
> 1. **Identity and Access Management (IAM):**
    *   Use IAM roles and policies to control access to Redshift clusters and databases.
    *   Grant least privilege access to users and applications.
    *   Regularly audit IAM policies and usage.
2. **Database Security:**
    *   Use database users and groups to manage access to specific databases and tables.
    *   Grant specific permissions (e.g., `SELECT`, `INSERT`, `UPDATE`, `DELETE`) on database objects.
3. **Encryption:**
    *   **Encryption at Rest:** Encrypt data at rest using AWS Key Management Service (KMS) or a hardware security module (HSM).
    *   **Encryption in Transit:**  Use SSL to encrypt data in transit between clients and the Redshift cluster.
4. **Network Security:**
    *   Use a Virtual Private Cloud (VPC) to isolate your Redshift cluster from the public internet.
    *   Configure security groups to control inbound and outbound traffic to the cluster.
5. **Auditing:**
    *   Enable audit logging to track user activity and database changes.
    *   Regularly review audit logs to detect suspicious activity.
6. **Data Masking and Tokenization:**
    *   Consider using data masking or tokenization techniques to protect sensitive data at rest or in transit. This involves replacing sensitive data with non-sensitive substitutes.
7. **Column-Level Encryption:** Redshift also allows encryption of individual columns using SQL functions along with a master symmetric 
