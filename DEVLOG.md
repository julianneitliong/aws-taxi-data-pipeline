# Day 1: Building a Serverless Data Pipeline on AWS — Ingestion & Cataloging

Kicking off a week-long project to build a serverless batch data pipeline on AWS (S3 → Glue → Step Functions → Athena), aligned with what I'm studying for the AWS Certified Data Engineer – Associate exam. Today covered the first stage of any data pipeline: **ingestion and cataloging.**

## Account setup & security

Before touching any data tooling, I locked down the AWS root account with MFA and created a dedicated IAM user for daily work, scoped with specific managed policies rather than operating as root. I also set a budget alarm so I get alerted before any unexpected spend — good hygiene for a learning sandbox, and a direct reflection of the exam's Security & Governance domain (least-privilege access, account safety).

## Choosing a data source

I sourced [NYC TLC Yellow Taxi Trip Records](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) for May 2026 — a real-world, public dataset (~4.09M rows) that's rich enough to support meaningful transformations, filtering, and partitioning later in the pipeline.

**A quick note on format:** the file downloads as Parquet, not CSV. Parquet is a columnar, binary format that stores data compressed and stores its schema (column names + real types) directly inside the file, unlike CSV's row-based plain text where everything is an untyped string until something parses it. That distinction mattered immediately in the cataloging step below.

## Landing the raw data in S3

Created an S3 bucket as the foundation of the data lake, then structured it into purpose-built zones:

- `raw/` — the original data, untouched. Nothing downstream ever modifies this, so any transformation logic can always be re-run from scratch if it needs fixing.
- `processed/` — reserved for cleaned, analysis-ready output from tomorrow's ETL job.
- `athena-results/` — isolates query scratch output from the actual data zones.

This zoning is standard data lake architecture (sometimes called bronze/silver), and it's a deliberate separation of concerns rather than just folder tidiness — it keeps raw source data recoverable and keeps unvalidated data out of anything a downstream consumer might query.

Uploaded the raw Parquet file into `raw/`.

## Cataloging with AWS Glue

Used AWS Glue to catalog the dataset: created a Glue database (`raw_db`) and ran a crawler against the `raw/` prefix, scoped with its own least-privilege IAM role. The crawler scanned the file and registered a table with the full schema — 20 columns, correctly typed.

**Because the source file was Parquet rather than CSV, this step was faster and required no manual type correction** — the crawler read the schema straight out of the file's embedded metadata instead of inferring it heuristically, so timestamps, doubles, and strings were all correctly typed on the first pass.

## What's next

Tomorrow: building the transformation layer. That means writing a Glue ETL job (Spark under the hood, built visually in Glue Studio) that reads from the cataloged raw table, applies real transformation logic (filtering bad records, casting/deriving fields), and writes partitioned output to `processed/` — setting up for querying in Athena and orchestration in Step Functions later this week.

---

**Stack:** AWS IAM · Amazon S3 · AWS Glue (Data Catalog, Crawlers) · Parquet
**Skills demonstrated:** cloud account security & least-privilege IAM, data lake architecture, schema cataloging, columnar file formats
