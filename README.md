# Payment Analytics Data Warehouse

A production-grade data warehouse built on Databricks with AWS S3 storage,
following medallion architecture (Bronze → Silver → Gold) and dimensional
modeling (Star Schema).

## Architecture

```
Kaggle Datasets (Olist E-commerce + Financial Transactions)
        ↓
   AWS S3 Bucket (storage layer)
        ↓
   Databricks (compute + transformation)
   ├── Bronze  → raw data, append-only, timestamped
   ├── Silver  → cleaned, typed, deduplicated, business rules applied
   └── Gold    → star schema, optimized for analytics
        ↓
   Power BI (visualization layer)
```

## Tech Stack

| Tool | Purpose |
|---|---|
| Databricks | Data transformation, SQL Warehouse, Unity Catalog |
| AWS S3 | Data lake storage — Bronze / Silver / Gold layers |
| AWS IAM | Secure, credential-free auth between Databricks and S3 |
| Delta Lake | ACID transactions, time travel, schema evolution |
| Power BI | Business intelligence and dashboards |
| SQL | All transformations, modeling, and analytics |

## Dataset

**Olist Brazilian E-Commerce** (kaggle.com/datasets/olistbr/brazilian-ecommerce)
- 100k real orders from 2016–2018
- Orders, payments, sellers, products, reviews, geolocation

**Financial Transactions** (kaggle.com/datasets/computingvictor/transactions-fraud-datasets)
- 13.3M card transactions
- Transaction details, card metadata, fraud signals

## Data Model

### Fact Tables
| Table | Rows | Grain |
|---|---|---|
| fact_orders | 112,650 | One row per order line item |
| fact_transactions | 13,305,915 | One row per card transaction |

### Dimension Tables
| Table | Rows | Description |
|---|---|---|
| dim_date | 6,209 | Calendar dates 2010–2026 |
| dim_customer | 99,441 | Customer profiles + geo coordinates |
| dim_product | 32,951 | Product catalog + English categories |
| dim_seller | 3,095 | Seller profiles + location |
| dim_payment_method | 5 | Payment types + labels |
| dim_card | 6,146 | Card details + fraud signals |
| dim_location | 19,015 | Zip code → lat/lng mapping |

## Medallion Architecture

### Bronze
- Raw CSVs ingested as-is into Delta tables
- Two metadata columns added: `_ingestion_timestamp`, `_source_file`
- Append-only — data is never modified or deleted
- 12 tables, ~13.7M total rows

### Silver
- Deduplication on primary keys across all tables
- Type casting — STRING → INT, DECIMAL, TIMESTAMP
- Dollar sign stripping on currency columns ($97.15 → 97.15)
- Null handling by business context (not blanket replacement)
- Derived columns: `is_delivered_on_time`, `delivery_days`, `is_refund`, `has_error`
- Text standardization: UPPER(TRIM()) on all categorical columns
- Geolocation deduplicated from 1,000,164 → 19,015 unique zip codes
- try_cast() used for tolerant casting on dirty date columns

### Gold
- Star schema with 2 fact tables and 7 dimension tables
- Surrogate keys on all dimension tables (integer joins)
- Partitioned by year + month for query performance
- OPTIMIZE + ZORDER on fact tables
- dim_date generated independently (2010–2026)

## Business Questions Answered

1. What is our monthly GMV trend?
2. Which product categories drive the most revenue?
3. Who are our top and bottom performing sellers?
4. Which payment methods have the highest installment usage?
5. Which seller → customer state corridors have worst delivery performance?
6. What is the transaction error rate by card brand and error type?
7. How does client spending evolve in months 0–5 after first transaction?
8. What is our total exposure from cards flagged on the dark web?

## Data Quality

15 automated checks across 6 Silver tables:

| Check Type | Count | Passed | Failed |
|---|---|---|---|
| NULL_CHECK | 6 | 6 | 0 |
| DUPLICATE_CHECK | 2 | 2 | 0 |
| RANGE_CHECK | 6 | 5 | 1 |
| REFERENTIAL_INTEGRITY | 1 | 1 | 0 |

**One investigated failure:** 660,049 negative transaction amounts.
Investigation confirmed 98.5% are refunds/credits (No Error status)
and 1.5% are reversed failed transactions — both valid business scenarios.

## AWS Setup

- S3 bucket with versioning, encryption, public access blocked
- IAM Role with least-privilege S3 policy
- Databricks Unity Catalog External Location → S3
- Credential-free auth via IAM Role assumption

## Project Structure

```
payment-analytics-dw/
├── ingestion/          CSV upload scripts
├── bronze/             Bronze layer DDL + ingestion
├── silver/             Silver transforms + cleaning
├── gold/               Star schema DDL + population
├── analytics/          8 business analytical queries
├── dq/                 Data quality checks
├── logs/               Pipeline logging setup
├── docs/               Architecture + layer documentation
└── README.md
```

## Key Design Decisions

**Why Medallion Architecture:**
Separating raw (Bronze), clean (Silver), and modeled (Gold) data means
any layer can be rebuilt independently without affecting others.
Debugging a data issue is traceable to a specific layer.

**Why Star Schema over normalized model:**
Analytical queries on a star schema require fewer joins, run faster,
and are self-explanatory to business users and BI tools.

**Why Delta Lake:**
ACID transactions prevent partial writes. Time travel allows debugging
by querying previous versions. Schema evolution handles source changes
without breaking downstream tables.

**Why S3 as storage layer:**
Decoupling storage (S3) from compute (Databricks) means storage costs
are minimal even when compute is off. This is the standard lakehouse
architecture pattern used at scale.

**Why surrogate keys:**
Integer joins are faster than string joins at 13M+ row scale.
Surrogate keys also decouple the warehouse from upstream ID format changes.

**Why partitioning by year/month:**
90% of analytical queries are time-bounded. Partitioning aligns
physical storage layout with the most common query pattern,
enabling partition pruning that skips irrelevant data entirely.
