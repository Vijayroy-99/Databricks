# Marketplace Delivery Performance Pipeline

A serverless data engineering pipeline on Databricks that ingests raw marketplace order data, validates it, and produces business-ready tables answering a single question: **what does late delivery actually cost?**

---

## The finding

**Late delivery costs 2.6 review points.**

| Delivery outcome | Avg review score |
|---|---|
| Early (5+ days) | 4.3 |
| Early (1–4 days) | 4.2 |
| On time | 4.0 |
| Late (1–3 days) | 3.3 |
| Late (4–10 days) | 2.0 |
| Late (10+ days) | 1.7 |

Across 96,470 delivered orders, 6.8% missed their promised date. The relationship between lateness and satisfaction is steep and monotonic — a marketplace with a logistics problem also has a reputation problem, and this quantifies the exchange rate between them.

Two secondary findings:

- **Lateness is seasonal, not chronic.** The monthly baseline sits near 3–4%, but demand peaks push it above 19%. This points to capacity rather than process.
- **Geography dominates delivery time.** Remote states run materially longer delivery windows than the southeast corridor, with freight costs following the same gradient.

---

## Data

[Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — roughly 100,000 anonymized orders placed between 2016 and 2018 across Brazilian marketplaces, spanning eight related tables covering orders, items, payments, reviews, customers, sellers, and products.

---

## Architecture

```
CSV files  →  Unity Catalog volume  →  bronze  →  silver  →  gold  →  dashboard
                                    Auto Loader   typed &   business
                                                  deduped   aggregates
                                                       ↓
                                                  validation
                                                  (blocks promotion on failure)
```

**Bronze** — raw ingest via Auto Loader with checkpointing and schema evolution. Every column stays a string; provenance columns (`_source_file`, `_ingested_at`) are appended. Bronze interprets nothing, so it remains the ground truth when a downstream number looks wrong.

**Silver** — typed, deduplicated, and joined. Timestamps cast, primary keys deduped via window functions, English category names resolved, and the core delivery metrics derived (`delivery_days`, `days_late`, `is_late`).

**Gold** — `fact_order_delivery` at one row per delivered order, partitioned by month, plus three aggregate marts covering seller performance, state-level delivery, and the monthly trend.

---

## Data quality

Validation runs between silver and gold and writes every result to a persistent `dq_audit` table. Warnings log; critical failures raise and halt the pipeline before gold is built on bad data.

**Checks implemented:**

| Check | Layer | Severity |
|---|---|---|
| Primary key uniqueness | silver | critical |
| Primary key not null | silver | critical |
| Referential integrity (items → orders) | silver | critical |
| Temporal sanity (delivered ≥ purchased) | silver | critical |
| Order status within known domain | silver | critical |
| Delivered orders carry a timestamp | silver | warn |
| Row count drift vs. previous run | silver | warn |
| Fact table grain (no join fan-out) | gold | critical |
| Fact row count matches silver filter | gold | critical |
| Review scores within 1–5 | gold | critical |
| Late percentage within plausible band | gold | warn |

### A bug this caught

The first ingest produced 104,162 review rows against 99,441 orders — impossible, since each order carries at most one review.

Cause: review comment text contains embedded newlines, and the CSV parser was treating each one as a record boundary. 4,879 rows were fragments of real reviews rather than reviews.

Fixed by enabling multiline parsing for that source. Post-fix: 99,224 rows, zero malformed. A referential integrity check catches this class of failure automatically, which is the argument for writing the checks before you need them.

---

## Repository

```
00_setup              catalog, schemas, and volume creation
01_bronze_ingest      Auto Loader ingestion with schema evolution
02_silver_transform   typing, deduplication, entity resolution
03_validate           data quality checks and audit logging
04_gold_marts         fact table and aggregate marts
```

---

## Running it

Requires a Databricks workspace with Unity Catalog. Built and tested on Free Edition.

1. Download the dataset from Kaggle and organize into one folder per table
2. Run `00_setup` to create the catalog structure and landing volume
3. Upload the CSV folders to `/Volumes/olist/bronze/landing`
4. Run notebooks `01` through `04` in order

Ingestion is idempotent — re-running skips already-processed files via Auto Loader checkpoints. In production these run as a scheduled job with sequential task dependencies, so a validation failure prevents gold from rebuilding.

---

## Notes on the platform

Databricks Free Edition is serverless-only; classic compute isn't available and clusters can't be configured. This constrains the project in ways worth stating plainly rather than dressing up:

- Spark RDD APIs are unavailable — serverless runs on Spark Connect, so DataFrame APIs only
- Scala and R aren't supported
- DBFS access is limited; Unity Catalog volumes are used instead
- Compute is subject to daily quotas

None of these materially affected the work. Managed serverless compute is also where the platform is heading, so the constraint tracks current practice more closely than manual cluster tuning would have.

Delta time travel provides reproducibility — any prior state of any table can be queried by version or timestamp, which means a historical run can be reconstructed and a bad transform rolled back.

---

## Built with

Databricks Free Edition · Apache Spark (PySpark) · Delta Lake · Auto Loader · Unity Catalog · Databricks SQL · AI/BI Dashboards
