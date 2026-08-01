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

## Day 2: Transformation Layer — Debugging a Real ETL Pipeline

Today's goal was the transformation stage of the pipeline: a Glue ETL job that reads the cataloged raw taxi data, cleans it, and writes partitioned, analysis-ready Parquet to `processed/`. It ended up being less about writing the job and more about debugging it — which, it turns out, is most of what real data engineering work actually looks like.

**The nested-timestamp trap.** Glue Studio's visual schema preview showed my pickup/dropoff timestamp columns as a nested "object" type, blocking the auto-create-catalog-table feature entirely — even though the underlying Parquet file stores them as real timestamps. Root cause: a known interop quirk in how certain Spark/Glue versions surface Parquet's microsecond timestamp encoding. Fixed with an explicit `CAST(... AS timestamp)`. Along the way I also learned the hard way that `SELECT * REPLACE (...)` is BigQuery/Snowflake syntax, not valid Spark SQL — an easy trap when you're used to a different SQL dialect.

**Real, corrupted data.** My first successful run wrote a `pickup_date=2008-12-31` partition into a file that's supposed to be entirely May 2026 trips. NYC's public taxi dataset is known to contain a small number of rows with genuinely corrupted timestamps — a good reminder that "the pipeline ran successfully" and "the output is correct" are two different claims, and only one of them was true here.

**Chasing a UI bug to its root.** Adding a date filter should have been simple, but the Filter transform's Key dropdown refused to recognize a column I could prove existed one node upstream — confirmed by checking each node's own output schema in isolation rather than trusting the graph at face value. After ruling out a broken node connection, a leftover catalog table with a conflicting schema, and a genuinely stale cache (none of which fully explained it), the real fix was pragmatic: move the filtering logic into the SQL Query node's `WHERE` clause instead of fighting the visual Filter node. Spark has zero problem filtering on an aliased or computed column — this was specifically a Glue Studio UI staleness issue, not a limitation of the underlying engine. Knowing when to drop from a visual/low-code tool down to explicit code is a real, professional skill, not a workaround to be embarrassed about.

**Result:** `processed/` now contains exactly 31 clean partitions — one per day of May 2026, corrupted dates filtered out, catalog schema verified clean.

**Key takeaways for next time:** the single best-leverage habit here would have been data profiling *before* building any transform logic — pulling a small local sample and checking schema against the source's published data dictionary and scanning min/max ranges would have caught the corrupted dates in seconds, for free, instead of discovering them after a full pipeline run. I also learned the real distinction between when to let a Glue crawler infer a schema versus defining it explicitly: it's not about dataset size, it's about whether the schema is already known and how wide/complex it is. Going forward, I'd reach for AWS Glue's Data Quality rules (declarative, auditable expectations) over ad hoc filter conditions for anything catching real data issues.

---

**Stack:** AWS Glue Studio · Spark SQL · Parquet · AWS Glue Data Catalog
**Skills demonstrated:** root-cause debugging across a multi-stage pipeline, Spark SQL semantics, distinguishing engine limitations from tooling limitations, data quality awareness, schema design decisions (crawler vs. explicit definition)
