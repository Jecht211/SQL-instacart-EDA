# Instacart Market Basket, Customer Retention & Operations Analysis

## Project Overview
An end-to-end SQL analysis of the Instacart Online Grocery Shopping dataset designed to optimize shopper labor scheduling, uncover weekly stock-up purchasing patterns, and identify high-retention staple products. The project involved building a normalized PostgreSQL database, engineering a staging-based ETL pipeline to consolidate split transaction files, and writing analytical queries across millions of order records.

* **Database:** PostgreSQL
* **Tools:** pgAdmin 4, SQL
* **Key Concepts:** Relational Schema Design, ETL Staging & Consolidation (`UNION ALL`), Primary/Foreign Key Constraints, Common Table Expressions (CTEs), Window Functions (`ROW_NUMBER`), Conditional Logic (`CASE WHEN`), Multi-Table Joins, Aggregations (`HAVING`, `COUNT DISTINCT`)

---

## Database Schema & Architecture
The raw dataset is sourced from the [Instacart Market Basket Analysis dataset on Kaggle](https://kaggle.com) *(excluded from this repository due to file size limits)*.

The dataset is modeled as a 5-table relational database enforcing referential integrity across departments, aisles, products, customer orders, and line-item transactions.

| Table Name | Description | Primary Key | Foreign Keys |
| :--- | :--- | :--- | :--- |
| `departments` | High-level grocery store departments (e.g., produce, dairy, deli) | `department_id` | — |
| `aisles` | Specific product aisles within departments | `aisle_id` | — |
| `products` | Product catalog mapped to aisles and departments | `product_id` | `aisle_id`, `department_id` |
| `orders` | Customer order metadata, timing (`dow`, `hour`), and order sequence | `order_id` | — |
| `order_products` | Consolidated line-item details, cart position, and reorder flags | — | `order_id`, `product_id` |

---

## Data Engineering & ETL Process
To prepare the raw CSV files for analysis in PostgreSQL, the setup script performs the following steps:
1. **Idempotent Schema Reset:** Drops existing tables with `CASCADE` before recreating tables with strict data types and foreign key constraints.
2. **Bulk Ingestion:** Uses PostgreSQL's `COPY` command for high-speed loading of lookup and order tables.
3. **Staging & Consolidation:** Loads the split `order_products__prior.csv` and `order_products__train.csv` files into temporary staging tables (`TEMP TABLE`), merges them into a unified `order_products` master table via `UNION ALL`, and drops the temporary tables to keep the schema clean.

---

## Key Business Questions & Queries

### 1. How should shopper labor be scheduled across days and hours?
**Approach:** Grouped all orders by day of week (`order_dow`, where `0` = Sunday) and `order_hour_of_day`, applying a `CASE WHEN` expression to classify hourly demand into actionable staffing tiers (`High`, `Moderate`, and `Low`).

```sql
SELECT 
    order_dow, -- 0 = Sunday, 6 = Saturday
    order_hour_of_day,
    COUNT(order_id) AS total_orders,
    CASE 
        WHEN COUNT(order_id) > 20000 THEN 'High Staffing (Peak)'
        WHEN COUNT(order_id) BETWEEN 8000 AND 20000 THEN 'Moderate Staffing'
        ELSE 'Low Staffing (Off-Peak)'
    END AS recommended_staffing_tier
FROM orders
GROUP BY order_dow, order_hour_of_day
ORDER BY order_dow, order_hour_of_day;
```

**Sample Output (Sunday, Hours 0–9):**

| `order_dow` | `order_hour_of_day` | `total_orders` | `recommended_staffing_tier` |
| :---: | :---: | :---: | :--- |
| 0 | 0 | 3,936 | Low Staffing (Off-Peak) |
| 0 | 1 | 2,398 | Low Staffing (Off-Peak) |
| 0 | 2 | 1,409 | Low Staffing (Off-Peak) |
| 0 | 3 | 963 | Low Staffing (Off-Peak) |
| 0 | 4 | 813 | Low Staffing (Off-Peak) |
| 0 | 5 | 1,168 | Low Staffing (Off-Peak) |
| 0 | 6 | 3,329 | Low Staffing (Off-Peak) |
| 0 | 7 | 12,410 | Moderate Staffing |
| 0 | 8 | 28,108 | High Staffing (Peak) |
| 0 | 9 | 40,798 | High Staffing (Peak) |

**Finding:** High staffing demand runs largely from **8:00 AM to 7:00 PM**, with moderate staffing needed at **7:00 AM** and **8:00 PM – 10:00 PM**, and low staffing overnight. Rather than spiking for a single hour, order volume climbs rapidly, holds a sustained daytime plateau, and tapers off gradually over a three-hour wind-down window.

---

### 2. When do customers engage in "stock-up" shopping vs. quick fill-in trips?
**Approach:** Combined a daily summary query (`COUNT DISTINCT` orders vs. total items sold) with an order-level basket query grouped by day of the week to analyze both overall daily averages and individual order-size distributions.

```sql
-- Daily Summary: Stock-Up Shopping Behavior by Day of Week
SELECT 
    o.order_dow,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(op.product_id) AS total_items_sold,
    ROUND(CAST(COUNT(op.product_id) AS NUMERIC) / COUNT(DISTINCT o.order_id), 2) AS avg_basket_size
FROM order_products op
JOIN orders o ON op.order_id = o.order_id
GROUP BY o.order_dow
ORDER BY o.order_dow;
```

**Query Output:**

| `order_dow` | `total_orders` | `total_items_sold` | `avg_basket_size` |
| :---: | :---: | :---: | :---: |
| 0 (Sun) | 585,237 | 6,533,692 | 11.16 |
| 1 (Mon) | 576,377 | 5,871,834 | 10.19 |
| 2 (Tue) | 458,074 | 4,378,360 | 9.56 |
| 3 (Wed) | 428,087 | 3,998,498 | 9.34 |
| 4 (Thu) | 417,171 | 3,942,696 | 9.45 |
| 5 (Fri) | 443,388 | 4,386,443 | 9.89 |
| 6 (Sat) | 437,749 | 4,707,583 | 10.75 |

```sql
-- Granular Distribution: Item Count per Order Grouped by Day of Week
SELECT 
    o.order_id,
    o.order_dow,
    COUNT(op.product_id) AS items_per_order
FROM order_products op
JOIN orders o ON op.order_id = o.order_id
GROUP BY o.order_id, o.order_dow;
```

**Finding:** Average basket size remains remarkably consistent across all seven days at **~10 items per order** (ranging only from **9.34** on Wednesday to **11.16** on Sunday), showing that early-week spikes are driven by higher active order counts rather than dramatically larger stock-up baskets.

---

### 3. Which high-volume products have the highest customer retention rates?
**Approach:** Calculated product-level reorder percentages and used a `HAVING COUNT(op.order_id) > 1000` filter to eliminate low-volume statistical outliers and isolate true high-retention staple SKUs.

```sql
SELECT 
    p.product_name,
    COUNT(op.order_id) AS total_purchases,
    SUM(op.reordered) AS total_reorders,
    ROUND(CAST(SUM(op.reordered) AS NUMERIC) / COUNT(op.order_id) * 100, 2) AS retention_rate_pct
FROM order_products op
JOIN products p ON op.product_id = p.product_id
GROUP BY p.product_name
HAVING COUNT(op.order_id) > 1000
ORDER BY retention_rate_pct DESC
LIMIT 15;
```

**Top 15 High-Retention Products:**

| `product_name` | `total_purchases` | `total_reorders` | `retention_rate_pct` |
| :--- | :---: | :---: | :---: |
| Half And Half Ultra Pasteurized | 2,995 | 2,580 | 86.14% |
| Whole Organic Omega 3 Milk | 9,410 | 8,091 | 85.98% |
| Organic Lactose Free Whole Milk | 8,742 | 7,511 | 85.92% |
| Organic Homogenized Whole Milk | 4,095 | 3,514 | 85.81% |
| Ultra-Purified Water | 1,524 | 1,306 | 85.70% |
| Milk, Organic, Vitamin D | 20,770 | 17,753 | 85.47% |
| Organic Reduced Fat Milk | 36,869 | 31,394 | 85.15% |
| Goat Milk | 5,353 | 4,551 | 85.02% |
| Banana | 491,291 | 415,166 | 84.51% |
| Organic Lowfat 1% Milk | 15,352 | 12,914 | 84.12% |
| Organic Whole Milk | 10,102 | 8,494 | 84.08% |
| Organic Reduced Fat Omega-3 Milk | 5,299 | 4,445 | 83.88% |
| Lowfat Goat Milk | 1,198 | 1,004 | 83.81% |
| Organic Low Fat Milk | 8,806 | 7,376 | 83.76% |
| Organic Milk Reduced Fat, 2% Milkfat | 13,119 | 10,984 | 83.73% |

**Finding:** Products with short shelf lives and high daily consumption rates dominate top reorder percentages—specifically **milk/dairy**, **beverages**, and **fresh produce** (with **Bananas** standing out at **491,291 purchases** and an **84.51%** reorder rate).

---

### 4. Which departments and flagship SKUs drive repeat purchases?
**Approach:** Used CTEs to compare reorder rates across departments (`deli`, `babies`, `snacks`, `household`, `international`, `pantry`, `personal care`) and applied the `ROW_NUMBER()` window function partitioned by `department` to isolate the top 3 best-selling items in every store category. *(Full department-by-department breakdown available in `sql/02_analysis_queries.sql`.)*

```sql
WITH RankedProducts AS (
    SELECT 
        d.department,
        p.product_name,
        COUNT(op.product_id) AS total_ordered,
        ROUND(CAST(SUM(op.reordered) AS NUMERIC) / COUNT(op.product_id) * 100, 2) AS reorder_rate_pct,
        ROW_NUMBER() OVER (PARTITION BY d.department ORDER BY COUNT(op.product_id) DESC) AS rank
    FROM order_products op
    JOIN products p ON op.product_id = p.product_id
    JOIN departments d ON p.department_id = d.department_id
    GROUP BY d.department, p.product_id, p.product_name
)
SELECT department, product_name, total_ordered, reorder_rate_pct
FROM RankedProducts
WHERE rank <= 3;
```

**Finding:** Essential, high-use staples dominate the top 3 rankings within every store category (such as **toilet paper** leading the `household` department).

---

## Summary of Insights & Recommendations
* **Workforce Optimization:** Align picker and driver shift schedules with the hourly staffing tier matrix—ramping up capacity during the **8:00 AM – 7:00 PM** high-demand plateau on **Sundays and Mondays** while scaling down during overnight off-peak hours.
* **Traffic-Driven Promotions:** Because average basket size holds steady around **~10 items** regardless of day, focus promotions on driving **order frequency** (mid-week re-engagement prompts) rather than expecting larger weekend stock-up carts.
