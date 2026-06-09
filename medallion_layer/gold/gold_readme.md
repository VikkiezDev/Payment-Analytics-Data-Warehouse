# Gold Layer — Documentation

## Overview

The Gold layer is the third and final stage of the medallion architecture. It is the only layer that analysts, BI tools, and business stakeholders ever touch directly.

```
Bronze  →  what we received
Silver  →  what we can trust
Gold    →  what we serve
```

The Gold layer has one job:

> **Organize clean Silver data into a structure that answers business questions as fast and simply as possible.**

That structure is called a **Star Schema**.

---

## What is a Star Schema

A star schema organizes data into two types of tables:

**Fact tables** store measurable business events — things that happen and can be counted or summed:
```
an order was placed       → fact_orders
a payment was made        → fact_transactions
```

**Dimension tables** store the context around those events — the who, what, where, when:
```
who ordered    → dim_customer
what product   → dim_product
where from     → dim_location
when           → dim_date
```

When you draw lines between them it looks like a star — one fact table in the center, dimension tables radiating outward. That's where the name comes from.

```
              dim_date
                 │
dim_customer ────┼──── dim_product
                 │
           fact_orders
                 │
dim_seller  ────┼──── dim_payment_method
                 │
            dim_location
```

**Why star schema instead of just querying Silver directly:**

```
Silver                          Gold Star Schema
──────────────────────          ──────────────────────────────
Normalized, many tables   →     Denormalized, fewer joins needed
Optimized for storage     →     Optimized for query speed
Requires knowing structure →    Self-explanatory table names
No surrogate keys         →     Integer surrogate keys = fast joins
Not BI-tool friendly      →     Plug directly into Power BI
```

---

## The Two Fact Tables

### fact_orders
```
Grain:   One row per order line item
Source:  silver.order_items + silver.orders + silver.order_payments + silver.reviews
Rows:    112,650
Partitioned by: order_year, order_month
```

Each row represents one product within one order. If an order has 3 products, it has 3 rows in fact_orders. This is called the **grain** — the most atomic level of detail the fact table stores.

**Why this grain:** It lets you answer questions at any level — per item, per order, per customer, per month — by simply changing what you GROUP BY. A coarser grain (one row per order) would lose the ability to analyze individual product performance.

**Measures stored:**
```
item_price              what the customer paid for this item
freight_value           shipping cost for this item
total_item_value        item_price + freight_value
total_payment_value     total payment for the entire order
max_installments        how many installments the order was split into
num_payment_methods     how many payment methods used in one order
has_refund              was any part of this order refunded
review_score            customer satisfaction score (1-5)
delivery_days           how many days from purchase to delivery
is_delivered_on_time    was it delivered by the estimated date
```

---

### fact_transactions
```
Grain:   One row per financial card transaction
Source:  silver.transactions + silver.cards + silver.users
Rows:    13,305,915
Partitioned by: txn_year, txn_month
```

Each row represents one card transaction. This is the financial behavior layer of the project — while fact_orders covers e-commerce activity, fact_transactions covers raw payment instrument behavior including fraud signals.

**Measures stored:**
```
amount                  transaction value in dollars
is_debit                whether money left the account
has_error               whether the transaction encountered an error
error_type              specific error (or 'No Error')
card_on_dark_web        fraud risk signal from cards dim
credit_limit            cardholder's credit limit
payment_channel         how payment was made (chip/swipe/online)
```

---

## The Seven Dimension Tables

### dim_date
```
Source:   Generated — no source table
Rows:     6,209 (2010-01-01 to 2026-12-31)
```

The date dimension is the only table in the entire warehouse not sourced from any input data. It is generated mathematically using Databricks' `SEQUENCE` function, producing one row for every calendar day across a 17-year range covering all transaction history plus future dates for forecasting.

```sql
SELECT EXPLODE(
    SEQUENCE(DATE('2010-01-01'), DATE('2026-12-31'), INTERVAL 1 DAY)
) AS d
```

**Why a dedicated date dimension instead of just using date functions:**

Every analytical query filters or groups by time. A date dimension pre-computes all time attributes once:

```
date_key          20240115        integer, used for fast joining
full_date         2024-01-15      actual date
year              2024
quarter           1
month_num         1
month_name        January
day_of_month      15
day_of_week_num   2
day_name          Monday
is_weekend        false
quarter_label     Q1-2024
month_year_label  Jan-2024
```

Without this, every query that needs month name, quarter label, or weekend flag has to compute it on the fly for millions of rows. With dim_date, it is a simple join on an integer key.

**Why integer date_key (e.g. 20240115):**
Integer joins are significantly faster than string or date joins at scale. Every fact table stores `date_key` as INT and joins to dim_date on that integer — not on a date string or timestamp comparison.

---

### dim_customer
```
Source:   silver.customers + silver.geolocation (enrichment)
Rows:     99,441
```

One row per unique customer. Enriched with latitude and longitude from the geolocation table so every customer record already has coordinates — no additional join needed in Power BI for geographic visualizations.

**Key design decision — customer_id vs customer_unique_id:**

Olist has two customer ID fields:
```
customer_id         unique per order — same person ordering twice gets two IDs
customer_unique_id  unique per person — the true customer identifier
```

Both are kept in dim_customer. fact_orders joins on `customer_id` (order-level) while fact_transactions joins on `customer_unique_id` (person-level). This design correctly handles the Olist data model without losing information.

---

### dim_seller
```
Source:   silver.sellers
Rows:     3,095
```

One row per marketplace seller. Contains seller location (city, state, zip) which enables the logistics corridor analysis — which seller-state to customer-state routes have the worst delivery performance.

---

### dim_product
```
Source:   silver.products (already enriched with English categories in Silver)
Rows:     32,951
```

One row per product. Category is already in English because Silver performed the Portuguese-to-English translation join. Gold inherits this enrichment for free — no re-joining needed.

Physical dimensions (weight, length, height, width) are included because they correlate with freight costs and delivery times — relevant for logistics analysis.

---

### dim_payment_method
```
Source:   DISTINCT payment_type values from silver.order_payments
Rows:     5
```

The smallest dimension in the warehouse — just 5 rows. One per payment method: credit card, debit card, boleto, voucher, and other.

Two label columns are added in Gold that did not exist in Silver:

```sql
payment_type_label   'credit_card'  → 'Credit Card'
payment_category     'credit_card'  → 'Card'
                     'boleto'       → 'Bank Transfer'
```

**Why here and not Silver:** Display formatting and business labeling belong in Gold. Silver stores the raw value. Gold makes it presentable for BI tools and analysts.

---

### dim_card
```
Source:   silver.cards
Rows:     6,146
```

One row per payment card. Contains the fraud risk signals that make the financial analysis interesting:

```
card_on_dark_web     has this card number been found on dark web marketplaces
is_limit_zero        is this card's credit limit zero
credit_limit         the card's credit limit
card_brand           Visa, Mastercard, Amex etc
card_type            credit, debit, prepaid
```

These attributes are pulled into fact_transactions so every transaction row already carries its card's risk profile — no additional join needed in analytical queries.

---

### dim_location
```
Source:   silver.geolocation
Rows:     19,015
```

One row per unique zip code with canonical lat/lng coordinates. Used by both fact tables — fact_orders joins through dim_customer, fact_transactions joins directly on merchant zip code.

Having a single location dimension shared by both facts is what makes geographic analysis consistent across the e-commerce and payments datasets.

---

## Surrogate Keys — What They Are and Why Gold Has Them

Every dimension table in Gold has a surrogate key — a simple integer generated by the warehouse that has no meaning in the real world:

```sql
ROW_NUMBER() OVER (ORDER BY customer_id) AS customer_key
```

**Natural key vs Surrogate key:**
```
Natural key:   customer_id = 'abc123def456...'   (from source system)
Surrogate key: customer_key = 47291              (generated by warehouse)
```

**Why surrogate keys:**

Integer joins are faster than string joins — joining on `customer_key = 47291` is significantly faster than joining on `customer_id = 'abc123def456...'` across 13 million rows.

They also protect against source system changes. If the upstream system changes how it formats customer IDs, your surrogate key stays stable. The mapping between old and new natural keys is handled in one place without breaking every downstream query.

---

## Partitioning

Both fact tables are partitioned by year and month:

```sql
CREATE OR REPLACE TABLE paymentdw.gold.fact_orders
USING DELTA
PARTITIONED BY (order_year, order_month)
```

**What partitioning does:**

Databricks physically stores data in separate folders by partition:

```
fact_orders/
├── order_year=2016/order_month=1/  ← data files for Jan 2016
├── order_year=2016/order_month=2/  ← data files for Feb 2016
├── order_year=2017/order_month=1/
...
```

When a query filters by date range:
```sql
WHERE order_year = 2018 AND order_month BETWEEN 1 AND 6
```

Databricks skips every other partition entirely — it never reads 2016 or 2017 data at all. For a 13.3 million row fact table, this can reduce query time from minutes to seconds.

**Why year and month specifically:**

Almost every business question is time-bounded — "last quarter", "this year vs last year", "month over month trend". Partitioning on year and month aligns storage layout with the most common query patterns.

---

## OPTIMIZE and ZORDER

After loading, OPTIMIZE and ZORDER were run on both fact tables and the three largest dimension tables.

**OPTIMIZE:**
```sql
OPTIMIZE paymentdw.gold.fact_orders
ZORDER BY (date_key, customer_key, product_key);
```

Delta Lake writes data in many small files during ingestion. OPTIMIZE compacts these into fewer, larger files — reducing the number of file opens per query and improving scan speed.

**ZORDER BY:**

ZORDER physically co-locates rows with similar values for the specified columns on disk. When a query filters by `date_key`, Databricks can skip entire files that don't contain the relevant date range — a feature called data skipping.

```
Without ZORDER:  query scans all files to find date range
With ZORDER:     query skips files guaranteed not to contain the date range
```

Columns chosen for ZORDER:
```
date_key        → most queries filter by date
customer_key    → cohort and customer-level analysis
product_key     → category and product analysis
card_key        → transaction-level card analysis
```

**Why ZORDER was not applied to dimension tables:**

Dimension tables are small — the largest (dim_customer) has 99,441 rows. At this size, full table scans are fast enough that the overhead of maintaining ZORDER is not worth it. ZORDER pays off at millions of rows.

**Note on OPTIMIZE output in this project:**

Because all tables were created in a single `CREATE TABLE AS SELECT` statement, Databricks wrote them in one optimally-sized file per partition. OPTIMIZE reported nothing to compact. In production where data arrives in thousands of small incremental files over time, OPTIMIZE would compact hundreds of files per run — this is its primary use case.

---

## How the Two Fact Tables Connect

The power of having two fact tables sharing the same dimensions is that you can answer questions that span both datasets:

```sql
-- Customers who have both placed orders AND had fraudulent transactions
SELECT DISTINCT fo.customer_key
FROM paymentdw.gold.fact_orders fo
JOIN paymentdw.gold.fact_transactions ft
    ON fo.customer_key = ft.customer_key
WHERE ft.card_on_dark_web = 'Yes'
  AND fo.total_payment_value > 500;
```

This kind of cross-fact analysis is only possible because both facts share `dim_customer` and `dim_date` as conformed dimensions — dimensions with consistent keys and definitions across multiple fact tables. This is one of the core principles of dimensional modeling.

---

## Gold Tables Produced

```
paymentdw.gold
│
├── DIMENSION TABLES
│   ├── dim_date              6,209 rows    2010–2026 calendar
│   ├── dim_customer         99,441 rows    enriched with geo coordinates
│   ├── dim_seller            3,095 rows    seller location data
│   ├── dim_product          32,951 rows    English categories
│   ├── dim_payment_method        5 rows    payment types + labels
│   ├── dim_card              6,146 rows    card details + fraud signals
│   └── dim_location         19,015 rows    canonical zip → lat/lng
│
└── FACT TABLES
    ├── fact_orders          112,650 rows    e-commerce order line items
    │   └── partitioned by order_year, order_month
    └── fact_transactions 13,305,915 rows    financial card transactions
        └── partitioned by txn_year, txn_month
```

---

## What Gold Does Not Do

Just as important as what Gold builds is what it intentionally avoids:

- **Does not clean data** — all cleaning happened in Silver. If dirty data reaches Gold it means Silver has a gap, not that Gold should compensate.
- **Does not store raw source columns** — Gold selects only what is needed for analysis. Columns with no analytical value are left behind.
- **Does not aggregate into summary tables** — Gold stores the most granular fact data. Aggregations happen at query time or in Power BI measures. Pre-aggregating in Gold would limit the questions you can ask.
- **Does not connect to external systems** — Gold is the final internal layer. Power BI, S3 export, and any other consumer connects to Gold but Gold has no knowledge of them.

---

## Business Questions Gold Enables

Every table in Gold was designed to answer specific business questions:

```
Question                                        Tables Used
────────────────────────────────────────────────────────────────────
Monthly GMV trend                               fact_orders + dim_date
Top product categories by revenue               fact_orders + dim_product
Seller performance scorecard                    fact_orders + dim_seller
Payment method popularity + installments        fact_orders + dim_payment_method
Delivery performance by state corridor          fact_orders + dim_seller + dim_customer
Transaction error rate by card brand            fact_transactions + dim_card
Customer cohort spend retention                 fact_transactions + dim_date
Dark web card exposure analysis                 fact_transactions + dim_card
Cross-dataset: high-value orders + fraud risk   fact_orders + fact_transactions + dim_customer
```