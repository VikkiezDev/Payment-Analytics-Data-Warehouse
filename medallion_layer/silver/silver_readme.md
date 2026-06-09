# Silver Layer — Documentation

## Overview

The Silver layer is the second stage of the medallion architecture. Its sole purpose is to take the raw, untrusted data from Bronze and make it clean, typed, and trustworthy enough to build business logic on top of.

```
Bronze  →  what we received from the source
Silver  →  what we can actually trust and use
Gold    →  what we serve to analysts and BI tools
```

The rule governing Silver is simple:

> **Never modify Bronze. All cleaning, fixing, and enrichment happens here and only here.**

---

## What Silver Does to Every Table

### 1. Deduplication

Every table in Bronze can potentially have duplicate rows — the same order loaded twice, the same customer appearing multiple times. Silver eliminates this using a window function pattern applied consistently across all 11 tables:

```sql
WITH deduped AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY primary_key
            ORDER BY _ingestion_timestamp DESC
        ) AS rn
    FROM paymentdw.bronze.some_table
)
SELECT ... FROM deduped WHERE rn = 1
```

**How it works:** `ROW_NUMBER()` assigns a rank to each group of duplicates sharing the same primary key. The most recently ingested row gets rank 1. `WHERE rn = 1` keeps only that row, discarding all duplicates.

**Why it matters:** Every join and aggregation downstream depends on primary key uniqueness. A single duplicate order_id in the orders table can corrupt revenue calculations across every Gold query that touches it.

**Result across all tables:**

```
Table                Bronze Rows    Silver Rows    Duplicates Removed
customers            99,442         99,441         1
sellers               3,096          3,095         1
products             32,952         32,951         1
orders               99,442         99,441         1
order_items         112,651        112,650         1
order_payments      103,887        103,886         1
reviews             104,163        100,781         3,382
geolocation       1,000,164         19,015         981,149
users                 2,001          2,000         1
cards                 6,147          6,146         1
transactions     13,305,916     13,305,915         1
```

The geolocation table is the most significant case — 1,000,164 raw rows reduced to 19,015 unique zip codes. The original table stored hundreds of slightly different lat/lng readings per zip code. Silver keeps one canonical coordinate per zip code, which is all any downstream join ever needs.

---

### 2. Type Casting

Bronze ingests everything as STRING by default — Databricks plays it safe when reading raw CSVs and doesn't assume what type any column is. Silver is where you declare what each column actually is:

```sql
CAST(order_item_id AS INT)
CAST(price AS DECIMAL(12,2))
TO_TIMESTAMP(order_purchase_timestamp)
CAST(customer_zip_code_prefix AS STRING)
CAST(credit_score AS INT)
```

**Why DECIMAL(12,2) for money:** Floating point types like FLOAT and DOUBLE cannot represent currency values exactly — `0.1 + 0.2` in floating point is `0.30000000000000004`. DECIMAL stores exact values. For financial data this is non-negotiable.

**Why zip codes stay STRING:** Zip codes look like numbers but are identifiers, not measures. Casting to INT silently destroys leading zeros — Boston's `02101` becomes `2101`, which is wrong and undetectable. The rule is: if you will never do arithmetic on it, it is a STRING regardless of what characters it contains.

---

### 3. Dollar Sign Stripping

Three tables in the financial transactions dataset stored currency as formatted strings rather than numeric values:

```
transactions_data   amount        '$97.15'
cards_data          credit_limit  '$5,000.00'
users_data          yearly_income '$48,000.00'
                    total_debt    '$12,500.00'
```

Silver strips the formatting and casts to DECIMAL:

```sql
try_cast(
    regexp_replace(amount, '[$,]', '')
    AS DECIMAL(12,2)
) AS amount
```

**How it works:** `regexp_replace` removes all `$` and `,` characters from the string, turning `'$1,234.56'` into `'1234.56'`. `try_cast` then converts the clean string to DECIMAL. If any value still can't convert for any reason, `try_cast` returns NULL instead of crashing the pipeline.

**Why this matters:** Without this step, every financial aggregation — total spend, average transaction value, credit utilization — is impossible. The column is stored as text, not a number.

---

### 4. Null Handling by Business Context

Not all nulls mean the same thing. Silver treats each null based on what it represents in the business, not by blindly applying a single rule:

**Null → meaningful label (errors column in transactions):**
```sql
COALESCE(NULLIF(TRIM(errors), ''), 'No Error') AS error_type
```
A null error means the transaction succeeded — there was no error to report. Replacing null with `'No Error'` makes this queryable and groupable. `GROUP BY error_type` now has a meaningful bucket for successful transactions instead of a null bucket that most BI tools handle inconsistently.

**Null → 'unknown' (product categories):**
```sql
COALESCE(product_category_name, 'unknown') AS category_portuguese
```
610 products had no category assigned. Dropping them would lose real revenue data. Replacing with 'unknown' keeps them in aggregations while making their incomplete state visible.

**Null → stays NULL (review scores):**
```sql
CASE
    WHEN review_score_clean BETWEEN 1 AND 5 THEN review_score_clean
    ELSE NULL
END AS review_score
```
A fabricated review score is worse than no score at all. If the score is invalid or missing, it stays NULL. Downstream aggregations use `AVG()` which ignores NULLs naturally — no fake data pollutes the ratings.

**Null → stays NULL (zip codes in transactions):**
1,652,706 transaction rows had no zip code — common for online transactions where location isn't captured. These rows are kept as-is. Dropping 12% of your transaction dataset because of a missing zip code would corrupt every volume and revenue metric.

---

### 5. Derived Business Rule Columns

Silver adds columns that don't exist in the source data but answer real business questions. These are computed once here so every downstream consumer gets them for free:

**Delivery performance (orders table):**
```sql
CASE
    WHEN order_delivered_customer_date IS NOT NULL
     AND order_estimated_delivery_date IS NOT NULL
     AND TO_TIMESTAMP(order_delivered_customer_date)
         <= TO_TIMESTAMP(order_estimated_delivery_date)
    THEN true
    ELSE false
END AS is_delivered_on_time,

DATEDIFF(
    TO_TIMESTAMP(order_delivered_customer_date),
    TO_TIMESTAMP(order_purchase_timestamp)
) AS delivery_days
```

These two columns alone enable an entire category of logistics analysis — on-time delivery rates, average delivery time by seller, worst-performing corridors — without any analyst needing to re-implement the date logic.

**Refund flag (order_payments table):**
```sql
CASE
    WHEN payment_value < 0 THEN true
    ELSE false
END AS is_refund
```
In payment systems, negative values represent money flowing back to the customer. Flagging these explicitly means refund analysis is a simple `WHERE is_refund = true` instead of requiring everyone downstream to know the sign convention.

**Error flag (transactions table):**
```sql
CASE
    WHEN errors IS NULL OR TRIM(errors) = '' THEN false
    ELSE true
END AS has_error
```
Boolean flags are faster to filter and easier to aggregate than string comparisons. `SUM(has_error)` gives error count. `AVG(has_error)` gives error rate. No string parsing needed downstream.

**Zero limit flag (cards table):**
```sql
CASE
    WHEN credit_limit = 0 THEN true
    ELSE false
END AS is_limit_zero
```
31 cards had a credit limit of zero — possibly new unactivated cards, suspended accounts, or debit cards with no credit facility. Rather than dropping these rows or guessing, Silver flags them and keeps the data. Downstream analysis can include or exclude them with a simple filter.

---

### 6. Text Standardization

Free text fields from CSVs arrive inconsistently — different cases, trailing spaces, empty strings masquerading as values. Silver standardizes all of them:

```sql
UPPER(TRIM(customer_city))    -- ' são paulo ' → 'SÃO PAULO'
UPPER(TRIM(customer_state))   -- 'sp ' → 'SP'
NULLIF(TRIM(review_message), '') -- '   ' → NULL
```

**Why this matters:** Without standardization, `GROUP BY customer_city` treats `'são paulo'`, `'SAO PAULO'`, `'Sao Paulo'`, and `' sao paulo'` as four different cities. Your revenue by city report is wrong and the error is invisible — no query will ever throw an error, it will just silently return fragmented results.

`NULLIF(TRIM(value), '')` converts empty strings to NULL. Empty string and NULL are different things in SQL — `COUNT(review_message)` counts empty strings but ignores NULLs. Standardizing to NULL means all null-handling logic works consistently.

---

### 7. Cross-Table Enrichment — Product Category Translation

The Olist dataset stores product categories in Portuguese. A separate translation table maps Portuguese names to English. Rather than making every downstream query perform this join, Silver does it once:

```sql
LEFT JOIN paymentdw.bronze.product_category_translation t
    ON d.product_category_name = t.product_category_name
```

Every Silver and Gold table that touches product categories now has English names available without any additional join. This is called **pushing the join down** — compute it at the earliest layer so everyone above gets the benefit.

---

### 8. Tolerant Casting with try_cast

The reviews table had review comment text leaked into the date columns in some rows — a column shift issue in the original CSV. Strict casting crashed:

```sql
-- Crashes on rows where date column contains text
TO_TIMESTAMP(review_creation_date)
```

The fix uses `try_cast` which returns NULL for unconvertible values instead of failing:

```sql
-- Bad rows become NULL, pipeline continues
try_cast(review_creation_date AS TIMESTAMP) AS review_created_at
```

**When to use each:**

```
TO_TIMESTAMP() / CAST()
→ Use when source is controlled and known clean
→ Failing loudly is intentional — bad data should block the pipeline

try_cast()
→ Use when source is external, untrusted, or historically dirty
→ A few bad rows should not block 100k good rows
→ Always follow with a DQ check that counts how many NULLs were produced
```

---

### 9. Metadata Columns

Every Silver table gets two additional columns:

```sql
current_timestamp() AS _processed_timestamp,
'source_table_name'  AS _source_table
```

**`_processed_timestamp`:** Records exactly when this row was cleaned and written to Silver. If a data quality issue is discovered in Gold three weeks from now, this timestamp tells you which Silver run produced the bad data.

**`_source_table`:** Records which Bronze table this row came from. In a complex pipeline with multiple source tables feeding one Silver table, this is essential for tracing data lineage.

---

## Silver Data Quality Issues Found and Handled

| Table | Issue Found | How Handled |
|---|---|---|
| `transactions_data` | `amount` stored as `'$97.15'` STRING | `regexp_replace` + `try_cast` to DECIMAL |
| `cards_data` | `credit_limit` stored as `'$5,000'` STRING | `regexp_replace` + `try_cast` to DECIMAL |
| `users_data` | `yearly_income`, `total_debt` stored as `'$48,000'` STRING | `regexp_replace` + `try_cast` to DECIMAL |
| `olist_reviews` | Text leaked into date columns | `try_cast` instead of `TO_TIMESTAMP` |
| `olist_reviews` | Non-numeric characters in `review_score` | `regexp_replace('[^0-9]')` + range check |
| `olist_products` | 610 null product categories | `COALESCE(value, 'unknown')` |
| `olist_order_payments` | 9 zero/negative payment values | Flagged as `is_refund`, zeroes set to NULL |
| `transactions_data` | 13M null `errors` | `COALESCE(value, 'No Error')` |
| `transactions_data` | 1.6M null zip codes | Kept as NULL — expected for online transactions |
| `cards_data` | 31 cards with `credit_limit = 0` | Flagged as `is_limit_zero = true`, kept |
| `olist_geolocation` | 1M rows, only ~19k unique zips | Deduplicated to one row per zip code |
| All tables | Potential duplicates on primary key | `ROW_NUMBER()` deduplication pattern |
| All text columns | Mixed case, trailing spaces | `UPPER(TRIM())` applied consistently |
| All text columns | Empty strings | `NULLIF(TRIM(value), '')` → NULL |

---

## What Silver Does Not Do

Understanding what Silver intentionally avoids is as important as what it does:

- **Does not aggregate** — no `GROUP BY`, no `SUM()`, no metrics. That is Gold's job.
- **Does not create surrogate keys** — foreign key relationships stay as natural business keys. Surrogate keys are a Gold layer concern.
- **Does not drop columns** — even columns not used downstream are kept. Silver is a complete, clean representation of the source. Dropping happens in Gold through intentional selection.
- **Does not apply business labels** — no `CASE WHEN payment_type = 'credit_card' THEN 'Credit Card'` display formatting. That belongs in dimension tables in Gold.
- **Does not join across datasets** — Silver tables map one-to-one with Bronze tables (with the exception of the product category translation which is a pure lookup enrichment). Cross-dataset joining happens in Gold when building facts.

---

## Silver Tables Produced

```
paymentdw.silver
├── customers       99,441 rows   cleaned from olist_customers
├── sellers          3,095 rows   cleaned from olist_sellers
├── products        32,951 rows   cleaned + translated from olist_products
├── orders          99,441 rows   cleaned + delivery metrics derived
├── order_items    112,650 rows   cleaned from olist_order_items
├── order_payments 103,886 rows   cleaned + refund flag added
├── reviews        100,781 rows   cleaned + score validated
├── geolocation     19,015 rows   deduplicated from 1,000,164 raw rows
├── users            2,000 rows   cleaned + $ columns converted
├── cards            6,146 rows   cleaned + $ columns + flags
└── transactions 13,305,915 rows  cleaned + $ converted + error flags
```