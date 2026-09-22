# E-Commerce Sales & Customer Analytics Dashboard

![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?logo=powerbi\&logoColor=black)
![SQL Server](https://img.shields.io/badge/SQL%20Server-Analysis-CC2927?logo=microsoftsqlserver\&logoColor=white)
![DAX](https://img.shields.io/badge/DAX-Measures-1F4E79)
![Data Analysis](https://img.shields.io/badge/Data%20Analysis-End--to--End-0F766E)


[ Access Google Drive Repository Assets](https://drive.google.com/drive/folders/119IWA15YBdNa5Frkm7gGyGopJv1cUc7h?usp=drive_link)



## Project Overview

This project analyzes an e-commerce business using **SQL Server, Power BI, and DAX** to understand sales performance, customer behavior, product performance, and operational distribution.

The project was developed as an **end-to-end business intelligence case study**, covering:

* Data validation and quality checks
* Relational data modeling
* SQL-based business analysis
* DAX measure development
* Interactive Power BI dashboarding
* Business insights and recommendations

The final Power BI solution contains **3 analytical pages** designed for both executive-level monitoring and detailed business analysis.

---

## Business Problem

The e-commerce business has a large volume of transactional, customer, product, inventory, and operational data, but lacks a consolidated analytical view of its performance. Management needs to understand **where revenue and profit are being generated, which customer segments are driving sales, which products and brands perform best, and how order outcomes vary across the business**.

The key business questions are:

* How are **revenue, cost, profit, and profit margin** performing over time?
* Which **product categories, individual products, and brands** generate the most revenue?
* Who are the major **customer segments** based on gender, age, geography, and traffic source?
* What proportion of orders are **shipped, completed, processing, cancelled, or returned**?
* Which **distribution centers** contribute most to revenue?
* Are there products in the catalog that generate little or no completed sales?

The objective of this project is to transform the raw e-commerce data into a **reliable SQL-based analytical layer and interactive Power BI dashboard** that enables management to monitor performance, identify important patterns, and investigate areas requiring further business analysis.

---

# Dataset

The project uses 7 relational tables:

| Table                  | Approx. Rows | Purpose                                                      |
| ---------------------- | -----------: | ------------------------------------------------------------ |
| `users`                |      100,000 | Customer demographics and profile information                |
| `orders`               |      125,226 | Order-level information and status                           |
| `order_items`          |      181,759 | Individual products associated with orders                   |
| `products`             |       29,120 | Product catalog, pricing, cost, category, and brand          |
| `inventory_items`      |      490,705 | Inventory-level records and product distribution information |
| `distribution_centers` |           10 | Distribution center information                              |
| `events`               |    2,431,963 | Website/event activity and traffic information               |

### Data Grain

Understanding table grain was important to avoid double-counting.

* `users` → one row per customer
* `orders` → one row per order
* `order_items` → one row per item associated with an order
* `products` → one row per product
* `inventory_items` → one row per inventory item
* `distribution_centers` → one row per distribution center
* `events` → one row per recorded event

The `events` table was retained as part of the analytical dataset but was not required for the core executive sales dashboard.

---

# Data Quality & Validation

Before dashboard development, the data was validated in SQL Server.

Key checks included:

### Row Count Validation

Final table counts were checked against expected dataset sizes.

### Order Item Validation

The number of item records associated with each order was compared with the `orders.num_of_item` field.

No mismatches were identified in the final validation.

### Relationship Validation

An additional Power BI check was used to confirm that `order_items` records correctly matched inventory records.

```DAX
Unmatched Inventory Items =
COALESCE(
    COUNTROWS(
        FILTER(
            order_items,
            ISBLANK(RELATED(inventory_items[id]))
        )
    ),
    0
)
```

Result:

**0 unmatched inventory items**

### Data Cleaning

During ingestion, formatting/type issues were encountered in the source files, including numeric-looking identifiers such as:

```text
35092.0
```

These were cleaned during the import/staging process so that relationships could be established reliably.

---

# SQL Analysis

SQL Server was used for:

* Data validation
* Aggregations
* Revenue analysis
* Cost and profit analysis
* Order status analysis
* Customer analysis
* Product analysis
* Distribution center analysis
* Benchmark validation for Power BI measures

## Sales Definition

A key business rule was established before creating the Power BI measures:

> **Gross sales = product retail price for `Shipped` and `Complete` order items.**

The benchmark SQL query was:

```sql
SELECT
    COUNT(*) AS gross_sales_items,
    SUM(p.retail_price) AS gross_sales_value,
    SUM(p.cost) AS gross_cost,
    SUM(p.retail_price - p.cost) AS gross_profit,
    AVG(p.retail_price) AS average_item_value
FROM dbo.order_items oi
JOIN dbo.products p
    ON oi.product_id = p.id
WHERE oi.status IN ('Shipped', 'Complete');
```

### Validated Results

| Metric        |             Value |
| ------------- | ----------------: |
| Gross Revenue | **$5,972,005.20** |
| Gross Cost    | **$2,872,168.49** |
| Gross Profit  | **$3,099,836.71** |
| Profit Margin |        **51.91%** |

These values were used as benchmarks for Power BI validation.

---

# Power BI Data Model

The final model uses a relational structure centered around `order_items`.

### Relationships

```text
users
  1
  |
  *
orders
  1
  |
  *
order_items
 /       \
*         *
|         |
products  inventory_items
  1
  |
distribution_centers
```

More specifically:

```text
users[id]                    1 ──── * orders[user_id]

orders[order_id]             1 ──── * order_items[order_id]

products[id]                 1 ──── * order_items[product_id]

inventory_items[id]          1 ──── * order_items[inventory_item_id]

distribution_centers[id]     1 ──── * products[distribution_center_id]
```

### Modeling Decision

A direct relationship between:

```text
distribution_centers ↔ inventory_items
```

was intentionally avoided because the model already provides a product-based path between these entities.

This reduced the risk of redundant filtering paths and ambiguity.

---

# DAX Measures

Core business measures were created in DAX so that KPIs remained dynamic under Power BI filters.

## Total Revenue

```DAX
Total Revenue =
CALCULATE(
    SUMX(
        order_items,
        RELATED(products[retail_price])
    ),
    order_items[status] IN {"Shipped", "Complete"}
)
```

## Total Cost

```DAX
Total Cost =
CALCULATE(
    SUMX(
        order_items,
        RELATED(products[cost])
    ),
    order_items[status] IN {"Shipped", "Complete"}
)
```

## Total Profit

```DAX
Total Profit =
[Total Revenue] - [Total Cost]
```

## Profit Margin

```DAX
Profit Margin % =
DIVIDE(
    [Total Profit],
    [Total Revenue],
    0
)
```

## Sales Orders

```DAX
Sales Orders =
CALCULATE(
    DISTINCTCOUNT(order_items[order_id]),
    order_items[status] IN {"Shipped", "Complete"}
)
```

## Average Order Value

```DAX
Average Order Value =
DIVIDE(
    [Total Revenue],
    [Sales Orders],
    0
)
```

## Total Customers

```DAX
Total Customers =
DISTINCTCOUNT(users[id])
```

## Purchasing Customers

```DAX
Purchasing Customers =
CALCULATE(
    DISTINCTCOUNT(order_items[user_id]),
    order_items[status] IN {"Shipped", "Complete"}
)
```

## Customer Purchase Rate

```DAX
Customer Purchase Rate % =
DIVIDE(
    [Purchasing Customers],
    [Total Customers],
    0
)
```

## Returned Orders

```DAX
Returned Orders =
CALCULATE(
    DISTINCTCOUNT(orders[order_id]),
    orders[status] = "Returned"
)
```

## Return Rate

```DAX
Return Rate % =
DIVIDE(
    [Returned Orders],
    DISTINCTCOUNT(orders[order_id]),
    0
)
```

## Cancelled Orders

```DAX
Cancelled Orders =
CALCULATE(
    DISTINCTCOUNT(orders[order_id]),
    orders[status] = "Cancelled"
)
```

## Cancellation Rate

```DAX
Cancellation Rate % =
DIVIDE(
    [Cancelled Orders],
    DISTINCTCOUNT(orders[order_id]),
    0
)
```

## Total Products

```DAX
Total Products =
DISTINCTCOUNT(products[id])
```

## Selling Products

```DAX
Selling Products =
CALCULATE(
    DISTINCTCOUNT(order_items[product_id]),
    order_items[status] IN {"Shipped", "Complete"}
)
```

## Unsold Products

```DAX
Unsold Products =
COUNTROWS(
    FILTER(
        products,
        CALCULATE(
            COUNTROWS(order_items),
            order_items[status] IN {"Shipped", "Complete"}
        ) = 0
    )
)
```

## Average Product Revenue

```DAX
Average Product Revenue =
DIVIDE(
    [Total Revenue],
    [Selling Products],
    0
)
```

## Average Product Margin

```DAX
Average Product Margin % =
AVERAGEX(
    FILTER(
        products,
        CALCULATE(
            COUNTROWS(order_items),
            order_items[status] IN {"Shipped", "Complete"}
        ) > 0
    ),
    DIVIDE(
        products[retail_price] - products[cost],
        products[retail_price],
        0
    )
)
```

## Total Brands

```DAX
Total Brands =
DISTINCTCOUNT(products[brand])
```

---

# Dashboard

The Power BI report contains 3 pages.

## Page 1 — Executive Sales Overview

![Executive Sales Overview](https://github.com/saud123/Ecommerce-Case-Study-SQP-POWER-BI-/blob/main/Executive.JPG?raw=true)

### Executive Dashboard: 
Overall Sales & Profit PerformanceStrong Profitability with Recent Contraction: The e-commerce division generated a highly efficient 51.91% profit margin ($3.10M profit out of $5.97M total revenue). However, the Revenue & Profit Trend visualization highlights that after reaching a peak in 2023, both revenue and profitability experienced a noticeable decline moving into 2024.

### Category and Product Revenue Drivers: 
Revenue is highly concentrated in premium apparel. Outerwear & Coats ($0.73M) and Jeans ($0.69M) are the top two revenue-generating product categories, heavily supported by high-performing individual items from The North Face and Nike Women's lines.

### Operational Leakage Risks: 
Despite booking 69K total orders, the business faces operational inefficiencies with a 10.01% return rate. Combined with a high volume of cancelled and returned items visible in the Order Status Distribution chart, these factors pose significant risks to maintaining long-term profit margins.

> Note: the final report layout may contain four primary analytical charts plus supporting visuals depending on the final dashboard arrangement.

---

## Page 2 — Customer & Order Analysis
![Order Analysis Overview](https://github.com/saud123/Ecommerce-Case-Study-SQP-POWER-BI-/blob/main/Order%20Analysis.JPG?raw=true)


### High Customer Churn & Low Retention: 
While the platform attracted 100.00K total customers, the unique net customer count drops to 52.91K, with a significant chunk (53K shoppers) buying only once. This indicates strong initial acquisition but low customer lifetime value, as top buyers frequently churn after a year.

### Demographic and Regional Concentrations: 
E-commerce sales lean male, generating $3.16M (52.98%) of revenue compared to female shoppers at $2.81M (47.02%). Geographically, the customer base is heavily concentrated in China and the United States, which dominate the top 5 revenue-generating countries.

### Search Traffic Dominance: 
The Customers by Traffic Source metric shows that Search is overwhelmingly the dominant driver for both total orders and customer acquisition. Traditional marketing channels like Organic, Facebook, and Email lag far behind, showing unexploited potential for targeted remarketing campaigns.---

## Page 3 — Product & Operations Analysis

![Product Analysis Overview](https://github.com/saud123/Ecommerce-Case-Study-SQP-POWER-BI-/blob/main/Product.JPG?raw=true)


### High Inventory Overreliance & Unsold SKUs: 
Out of a catalog of 29K total products, only 51.08% (28K) are actively selling, leaving an absolute volume of 1,014 items completely unsold. This underutilization points to potential deadstock or inefficiencies in product-market matching.

### Extreme Brand Concentration:
Revenue is dangerously dependent on just two major brands. The North Face accounts for 45.62% and Canada Goose accounts for 25.93% of total revenue. Combined, these two brands bring in over 71% of the company's entire money flow, creating a major structural risk if supply lines or consumer affinity for them changes.

### Fulfillment Centralization: 
Supply chain logistics rely heavily on a single node. The Houston, TX warehouse handles the massive lion's share of distributions, dwarfing secondary centers like Memphis, TN and Philadelphia, PA. This central bottleneck increases vulnerability to regional disruptions or localized shipping overloads.

### Product & Operations Analysis

## 1. Revenue grew strongly through 2023

Validated annual gross-sales figures were:

| Year |    Revenue |
| ---- | ---------: |
| 2019 |   $121,553 |
| 2020 |   $411,391 |
| 2021 |   $798,845 |
| 2022 | $1,374,087 |
| 2023 | $2,808,339 |
| 2024 |  $457,790* |

The business experienced substantial growth from 2019 through 2023.

* **2024 is a partial-year period**, with data extending only into January, so the 2024 value should not be interpreted as a full-year decline.

---

## 2. Gross profitability is strong

The validated sales population generated:

* **$5.97M revenue**
* **$2.87M cost**
* **$3.10M gross profit**
* **51.91% gross profit margin**

This indicates that the analyzed completed/shipped sales carry a substantial gross margin.

---

## 3. Men's products generate more revenue than women's products

Validated revenue by gender:

| Gender |    Revenue |
| ------ | ---------: |
| Men    | **$3.16M** |
| Women  | **$2.81M** |

The difference is descriptive and should be interpreted alongside customer count, product assortment, order volume, and other segmentation variables.

---

## 4. Product categories contribute differently to revenue and margin

Examples from the category analysis include:

* **Outerwear & Coats:** ~$732K revenue
* **Jeans:** ~$695K revenue
* **Sweaters:** ~$468K revenue
* **Suits & Sport Coats:** ~$364K revenue

Margin also varies across categories, meaning a high-revenue category should not automatically be treated as the most profitable category.

---

## 5. Order status requires separate interpretation

Order-level status counts include:

| Status     | Orders |
| ---------- | -----: |
| Shipped    | 54,440 |
| Complete   | 45,609 |
| Processing | 36,388 |
| Cancelled  | 27,090 |
| Returned   | 18,232 |

The dashboard therefore separates:

* **Sales performance** → based on `Shipped + Complete` order items
* **Return/cancellation analysis** → based on the full order population

This distinction prevents misleading denominators.

---

## 6. Returns and cancellations represent important operational signals

Using the full order population:

* **Returned orders:** 12,530
* **Return rate:** ~10.01%
* **Cancelled orders:** 27,090
* **Cancellation rate:** ~21.63%

These metrics should be investigated further by customer segment, product, category, geography, and operational factors before drawing causal conclusions.

---

# Business Recommendations

The recommendations below are designed as **data-driven next steps**, not conclusions that a single dashboard can prove by itself.

## 1. Investigate high cancellation rates

Break cancellation rates down by:

* Product category
* Product
* Customer state
* Traffic source
* Year/month
* Distribution center

This can help identify whether cancellations are concentrated in particular parts of the business.

---

## 2. Investigate return drivers

Analyze returned orders against:

* Product category
* Product
* Brand
* Customer demographics
* Order value
* Delivery timing
* Distribution center

The objective is to distinguish product-related return patterns from customer- or operational-related patterns.

---

## 3. Optimize product assortment

Use the product analysis to classify products into groups such as:

```text
High Revenue + High Margin
High Revenue + Low Margin
Low Revenue + High Margin
Low Revenue + Low Margin
Unsold Products
```

This can support more targeted assortment and inventory decisions.

---

## 4. Monitor revenue concentration

Top-product and top-brand analysis can be extended to measure:

* Revenue contribution of the Top 10 products
* Revenue contribution of the Top 10 brands
* Category concentration
* Distribution-center concentration

This can reveal whether overall revenue depends heavily on a small number of products or brands.

---

## 5. Build customer retention analysis

The current dashboard describes customer and purchasing behavior but does not by itself prove customer churn.

A future analysis should calculate:

* New vs returning customers
* Repeat purchase rate
* Customer retention by cohort
* Customer lifetime value
* First-to-second purchase conversion
* Annual customer retention

This would provide a stronger basis for customer-lifecycle decisions.

---

# Limitations

Several analytical limitations should be considered.

### 2024 is partial-year data

The 2024 revenue figure only represents the available portion of the year and is therefore not directly comparable with complete prior years.

### Revenue uses a defined sales population

The dashboard's revenue measures use:

```text
Shipped + Complete order items
```

This definition should be kept consistent when comparing revenue KPIs.

### Return and cancellation metrics use the full order population

Return and cancellation rates should not use the `Shipped + Complete` sales-order population as their denominator.

### The dashboard does not establish causality

For example, a high return rate does not automatically prove that returns cause customers to stop purchasing.

Further customer-level and operational analysis would be required.

### Website conversion was not included in the core dashboard

The `events` table contains website activity, but conversion-rate analysis was intentionally excluded from the main dashboard because true conversion requires a clear visitor/session/funnel definition.

---

# Tools & Technologies

### SQL Server / SSMS

Used for:

* Data loading
* Validation
* Joins
* Aggregations
* Business metrics
* Benchmark calculations

### Power BI

Used for:

* Data modeling
* Interactive dashboards
* Slicers
* Visual analytics
* KPI reporting

### DAX

Used for:

* Dynamic measures
* Profitability calculations
* Customer metrics
* Product metrics
* Time intelligence

### Power Query

Used for:

* Data transformation
* Date handling
* Column preparation

---

# Project Workflow

```text
Raw CSV Data
      ↓
Data Import & Cleaning
      ↓
SQL Server Validation
      ↓
Business Metric Definition
      ↓
Power BI Data Model
      ↓
DAX Measures
      ↓
Interactive Dashboard
      ↓
Business Insights
      ↓
Recommendations
```

---


---

# Dashboard Preview

## Dashboard Preview

### Executive Sales Overview
![Executive Sales Overview](https://github.com/saud123/Ecommerce-Case-Study-SQP-POWER-BI-/blob/main/Executive.JPG?raw=true)


### Customer & Order Analysis
![Order Analysis Overview](https://github.com/saud123/Ecommerce-Case-Study-SQP-POWER-BI-/blob/main/Order%20Analysis.JPG?raw=true)

### Product & Operations Analysis
![Product Analysis Overview](https://github.com/saud123/Ecommerce-Case-Study-SQP-POWER-BI-/blob/main/Product.JPG?raw=true)

```




---

# Key Takeaways

This project demonstrates an end-to-end analytics workflow:

**SQL → Data Validation → Data Modeling → DAX → Power BI → Business Insights**

Rather than focusing only on visualization, the project emphasizes:

* Clear business metric definitions
* Proper table relationships
* Grain awareness
* Data-quality validation
* Dynamic DAX measures
* Interactive reporting
* Evidence-based business interpretation

---

# Author

**Saud Ijaz**

Data Analyst | SQL | Power BI | DAX | Excel | Data Visualization

---
