# Instacart Market Basket, Customer Retention & Operations Analysis

## Project Overview
This repository contains an exploratory data analysis (EDA) and operational case study of Instacart consumer purchasing patterns, traffic flows, and reorder dynamics. Leveraging a local PostgreSQL database, Python (`pandas`, `sqlalchemy`, `psycopg`), and interactive Plotly visualizations. This project uncovers the behavioral drivers behind grocery delivery logistics, highlighting key metrics like the **fixed ~10-item basket invariant** and the **100K product volume reorder stability threshold**.

* **Database:** PostgreSQL (pgAdmin 4)
* **Python Stack:** `pandas`, `sqlalchemy`, `psycopg`, `python-dotenv`, `plotly`
* **Key Concepts:** Relational Schema Design, ETL Staging & Consolidation (`UNION ALL`), Common Table Expressions (CTEs), Window Functions (`ROW_NUMBER`), Conditional Logic (`CASE WHEN`), Behavioral & Operational Analytics

---

## Database Schema & Architecture
The raw dataset is sourced from the [Instacart Market Basket Analysis dataset on Kaggle](https://kaggle.com) *(excluded from this repository due to file size limits)*. 

The data is modeled as a 5-table relational database enforcing referential integrity across store hierarchies, orders, and consolidated line-item transactions (`prior` + `train` sets merged via staging tables).

| Table Name | Description | Primary Key | Foreign Keys |
| :--- | :--- | :--- | :--- |
| `departments` | High-level grocery store departments (e.g., produce, dairy, deli) | `department_id` | — |
| `aisles` | Specific product aisles within departments | `aisle_id` | — |
| `products` | Product catalog mapped to aisles and departments | `product_id` | `aisle_id`, `department_id` |
| `orders` | Customer order metadata, timing (`dow`, `hour`), and order sequence | `order_id` | — |
| `order_products` | Consolidated line-item details, cart position, and reorder flags | — | `order_id`, `product_id` |

*(For the complete DDL/ETL script and full SQL query walkthrough, see [`sql/SQL_ANALYSIS.md`](sql/SQL_ANALYSIS.md).)*

---

## Key Analytical Findings & Visualizations

### 1. Order Volume & Basket Size Distribution by Day of the Week
Querying daily order volume (`df_dow`) alongside order-level item distributions (`df_box`, zoomed to a 0–35 item y-axis range to highlight core distributions) tested whether weekend demand spikes reflect larger stock-up baskets or higher customer traffic.
* **Order Volume:** **Sunday and Monday** generate the highest volume at **~58K orders per day**, compared to **~44K orders per day** across the rest of the week.
* **Basket Size Invariant:** Average basket size remains steady at **~10 items per order** across all seven days. Box-and-whisker distributions show minimal day-to-day variance: **Sunday and Saturday** tie for the slightly highest median, while extreme bulk outliers vary randomly (e.g., **Tuesday** reaches a max of **144 items** vs. **Sunday** at **99 items**, despite Sunday having the highest overall order volume). Early week surges are driven by **higher active shopper traffic** rather than larger individual stock-up carts.

![Instacart Order Volume by Day of the Week](visualizations/dow_order_volume.png)
![Basket Size Distribution by Day of the Week](visualizations/basket_size_boxplot.png)

---

### 2. Hourly Traffic Curves & Multi-Day Staffing Patterns
To support shopper labor scheduling, total order volume was plotted across a 24-hour cycle (`df_hourly`) and segmented across all seven days of the week (`df_hourly_dow`) using chronologically ordered multiple line chart.
* **Finding:** While orders occur during every hour of the day, volume drops into a deep valley between **1:00 AM and 5:00 AM**, climbs steadily starting at **6:00 AM**, holds a sustained daytime plateau from **9:00 AM to 4:00 PM**, and declines steadily back toward the 1:00 AM trough. Segmenting by day of the week shows that **all seven days follow this identical curve**, simply scaling up to higher daytime peaks on **Sunday and Monday**.

![Instacart Order Volume by Hour of the Day](visualizations/hourly_order_volume.png)
![Hourly Order Volume by Day of the Week](visualizations/hourly_dow_order_volume.png)

---

### 3. Department Reorder Rates
Aggregating purchase volume against reorder flags across all 21 store departments (`df_dept`) separated habitual replenishment categories from occasional purchases.
* **Finding:** Department level reorder rates are largely driven by **shelf life and consumption speed**. Perishable, fast-depleting categories, led by **`dairy eggs`**, **`beverages`**, and **`produce`**, rank highest in repeat purchases because they spoil quickly or are consumed daily, whereas slower-turning categories (`personal care`, `household`, `pantry`) see far lower reorder rates.

![Department Reorder Rates](visualizations/department_reorders.png)

---

### 4. Product Volume vs. Reorder Rate (The 100K Stability Threshold)
Splitting departments down to the top 100 individual products (`df_products`, colored by department) evaluated how SKU order volume correlates with item-level retention.
* **Finding:** While higher total order volume generally trends toward higher reorder rates, items in the **35K to 100K order range** exhibit wide variance in repeat behavior. Shelf life, consumption rate, and order volume all contribute to retention, but **only above the 100K total order volume threshold does the noise disappear**, converging into consistently high reorder rates.

![Product Volume vs. Reorder Rate](visualizations/product_volume_vs_reorder.png)

---

## Strategic Recommendations
* **Dynamic Shift Scheduling:** Align picker and driver shift blocks with the 9:00 AM – 4:00 PM daytime plateau and Sunday/Monday traffic surges (~58K orders/day vs. ~44K mid-week) while scaling down overnight shifts during the 1:00 AM – 5:00 AM valley.
* **Traffic-Driven Promotions:** Because basket size and distribution remain invariant across the entire week, focus promotions on driving **order frequency** (mid-week re-engagement prompts) rather than attempting to inflate individual basket sizes.
* **High-Volume Anchor Merchandising:** Feature 100K+ volume staple SKUs (where reorder predictability stabilizes) and top-3 department leaders (identified via `ROW_NUMBER()` partitioning) in "Buy It Again" checkout prompts to streamline repeat conversions.

---

## Repository Structure
```text
├── Graph 6-.png                # Average Basket Size Box-and-Whisker plot for each day of the week
├── Graphs 1-3.png              # Daily order volume, Department Reorder Rates, and Reorder Rate Scatterplot
├── Graphs 4-5.png              # Instacart Order Volume by Hour of the Day and Hourly day of the week Order Volume curves
├── instacart_analytics.ipynb   # Python EDA notebook (SQLAlchemy + Plotly visualizations)
├── SQL_queries.md              # Detailed SQL schema, ETL setup, and query documentation
├── .env.example                # Template for PostgreSQL connection environment variables
├── .gitignore                  # Excludes .env credentials and raw Kaggle CSV files
└── README.md                   # Executive project summary (main landing page)
```
