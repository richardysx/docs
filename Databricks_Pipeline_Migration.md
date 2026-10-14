# Migrating Production Spark Pipelines off Databricks: What Broke, and How We Proved the Data Matched

> Originally published at **[pipeprompt.com](https://pipeprompt.com/blog/databricks-pipeline-migration/)**, where you can also find more posts on data engineering and AI.

Recently I spent a few weeks moving **a few dozen scheduled Spark pipelines** from Databricks to a managed Spark service on a different cloud. They feed a retail group's BI dashboards: inventory, sales, purchasing, supplier performance, stock cover days. Some of the tables are big. The largest fact and history tables run to **billions of rows**.

The code changes were the easy part. The hard part was the bugs that let a pipeline finish green while writing the wrong data, and then **proving** that the new platform produced the same data as the old one. This post covers what broke, how we fixed it, and where an AI coding assistant helped and where it didn't.

Company names, table names, platform details, dates and exact figures are generalized; the bugs and the methods are real.

## A migration changes three layers at once

It's tempting to think of this as "copy the notebooks, recreate the schedules". In practice three layers change together:

| Layer | Databricks | Target platform |
| --- | --- | --- |
| Scheduling | Jobs and tasks | Workflows and tasks, with their own versioning and publish step |
| Compute | Databricks Runtime clusters | A managed Spark engine with resource groups |
| Data access | Unity Catalog names, Delta tables | Different catalog paths, Iceberg tables |

Each layer has its own failure modes, and a bug in one often looks like a bug in another. A "table not found" error can mean a wrong catalog path, a missing permission, or a table the pipeline was supposed to create itself.

## Problem 1: the obvious incompatibilities

These are the ones everyone expects, and they fail loudly, which makes them the cheapest to fix:

- **`dbutils`** (widgets, file listing) has to become the target platform's utility module. Even the import style mattered: one form of the import ran fine interactively and failed in batch mode with `module has no attribute 'widgets'`.
- **`display()`** becomes `df.show(truncate=False)`.
- **Bare `%sql` cells** become `spark.sql("""...""")` in Python cells.
- **`OPTIMIZE ... ZORDER BY`** and **`VACUUM`** are Delta maintenance commands. We disabled them during migration and planned Iceberg compaction and snapshot expiry separately.
- **`CREATE TABLE ... LIKE`** wasn't supported. Explicit DDL replaced it.
- **Three-level names** like `ext.schema.table` were rejected outright. Every source table had to be mapped to its real catalog path on the new side.

Before touching anything, we scanned every notebook for a list of platform-specific markers: `dbutils`, `display(`, `%sql`, `OPTIMIZE`, `ZORDER`, `VACUUM`, `DESCRIBE DETAIL`, `DeltaTable`, `dbfs:/`, `abfss:/`. That list became the migration checklist.

Checking the mapping was its own job. A read-only scan of the target catalog found **thousands of tables across hundreds of namespaces**, and we confirmed every source table each pipeline read existed there before running anything.

## Problem 2: SQL that translated "successfully" and was wrong

This was the most dangerous class of bug, because nothing failed.

Databricks SQL supports a three-argument `DATEADD(unit, amount, date)`. The target engine wanted Spark's two-argument `date_add(date, days)`. In one pipeline the first-pass conversion produced this:

```sql
-- Original (Databricks), simplified
WHERE day_dt >= DATEADD(day, -7, current_date())

-- After conversion
WHERE day_dt >= DATE_ADD(current_date(), CAST(7 AS INT))
```

The minus sign was gone. "Last 7 days" quietly became "the next 7 days". There were **dozens of sign flips like this in a single notebook**, covering nearly every lookback window in it.

A couple more were worse:

```sql
-- Original: shift a UTC timestamp to local business time
DATEADD(hour, 7, create_datetime)

-- After conversion: adds 7 *days* and drops the time of day
DATE_ADD(create_datetime, CAST(7 AS INT))
```

All of these queries ran without errors. A "last 7 days" window that points into the future just returns zero rows. Feed that into a delete-then-insert step and history quietly disappears. In this pipeline, **one history table was found empty** and had to be backfilled.

How we fixed it:

1. **Fix it mechanically, compared against the original.** I wrote a script that walks the original and migrated notebooks cell by cell, pairs each original `DATEADD(day, n, ...)` with the corresponding migrated `DATE_ADD`, and restores the original sign. It refuses to change anything if the cell count, call count or absolute value doesn't line up. It only fixes what it can prove, and stops on anything ambiguous.
2. **Add a write guard.** Before the job deletes a history window, it now checks that the staging table is non-empty and that its latest date is at least yesterday. A wrong date window now fails loudly instead of wiping data.
3. **Pin time zones down.** The business day was in a different time zone from the cluster. We set the Spark session time zone to UTC and converted explicitly, instead of relying on `getdate()`, which the target engine didn't support anyway.

The same pipeline had a type issue too: a shelf-life column arrived as `STRING` and was written into a `DOUBLE` column. Databricks had cast it implicitly; the new engine rejected the write. A `TRY_CAST` at the source fixed it.

## Problem 3: hidden state the old system already had

Some pipelines assumed tables that had existed on Databricks for years:

- A history table whose `CREATE TABLE AS SELECT` had been **commented out** long ago, because it only needed to run once. Every later run just did `DELETE` and `INSERT`. On a fresh platform, the first run failed.
- A debug cell that ran `DESC HISTORY` on several tables had been converted into `SELECT ... VERSION AS OF 10`. That's a time-travel query to a snapshot number that doesn't exist on Iceberg. The right fix was to query the Iceberg `snapshots` metadata table, not to fake a snapshot.
- In one notebook, the conversion had replaced the input table and the output table with **the same name**, so the job read the table it was about to overwrite.
- One pipeline ran `COUNT` and `DELETE` against one history table and then `INSERT`ed into a similarly named table with a different year suffix. That bug was **in the original code**. Migration just exposed it, and only the business owner could say which table was the real one.

One rule we kept throughout: **never create an empty table just to make a job pass.** An empty table with a guessed schema turns a loud failure into a silent one. Initialization tables were rebuilt from the pipeline's own staging data, with the original schema and partitioning.

## Problem 4: "saved" is not "deployed"

The target scheduler keeps notebook source and the published workflow version separately. We fixed a bug, saved the notebook, reran the job, and the driver log still showed code we had deleted. The workflow was running the **previously published snapshot**.

A read-only audit of the tasks that had passed test runs found that **more than half had never been submitted to the scheduler**. Their test runs had passed, but the scheduled job would never have used that code.

After that, every change went through the same steps: publish, read the source back from the API, compare hashes with the local version, and confirm the published version points at it. Only then did we rerun anything.

## Problem 5: proving the data matched

Row counts aren't enough. Two tables can have the same number of rows and different contents. For each output table, we pinned a version on both sides (a Delta version and an Iceberg snapshot) and computed a fingerprint for every column:

```python
from pyspark.sql import functions as F
from pyspark.sql.types import DecimalType, DoubleType, FloatType, TimestampType

def normalize(df, exclude=("data_refresh_time",)):
    """Make values comparable across engines before hashing."""
    cols = []
    for f in df.schema.fields:
        if f.name in exclude:          # technical columns that always differ
            continue
        c = F.col(f.name)
        if isinstance(f.dataType, (DoubleType, FloatType, DecimalType)):
            c = c.cast("decimal(38,8)")   # 8 decimal places
        elif isinstance(f.dataType, TimestampType):
            c = F.date_format(c, "yyyy-MM-dd HH:mm:ss.SSSSSS")
        cols.append(c.alias(f.name))
    return df.select(cols)

def fingerprint(df):
    aggs = [F.count(F.lit(1)).alias("_rows")]
    for name in df.columns:
        aggs.append(F.count(name).alias(f"{name}__non_null"))
        # Order-independent: the sum of row hashes doesn't depend on row order.
        aggs.append(F.sum(F.xxhash64(name).cast("decimal(38,0)")).alias(f"{name}__hash"))
    return df.agg(*aggs).first().asDict()
```

Run it on both sides and compare the dictionaries. Matching fingerprints are strong evidence, not a mathematical proof, since hashes can collide. **Different fingerprints are proof** that the contents differ, and they tell you which columns to look at. For the billion-row fact tables we computed fingerprints per business date over recent weeks instead of over the full table.

In the formal acceptance round, results fell into a few groups:

- **Every column matches.** The largest group.
- **Keys match, some numeric columns differ.** Same rows, different inventory or sales values.
- **Rows differ.** Different row sets, not just different values.
- **Large fact tables mostly match**, with a couple of business dates that didn't.
- **No valid comparison**, because the old job itself failed that day.

Every difference got a root cause. Two of them are worth telling.

### The old pipelines were reading yesterday's data

Several Databricks jobs ran on a fixed cron schedule **with no check that their inputs were ready**. On most days the purchase-order table they depended on landed a few minutes *after* those jobs started, so they read **the previous day's batch** and nobody noticed, because the dashboard still looked plausible.

On the new platform we added a readiness gate. A job won't start processing until each input has a real business commit for today, meaning an append or overwrite that added rows, not just a compaction:

```python
def require_today(table):
    last = spark.sql(f"""
        SELECT max(committed_at) AS last_commit
        FROM {table}.snapshots
        WHERE operation IN ('append', 'overwrite')
          AND CAST(summary['added-records'] AS BIGINT) > 0
    """).first().last_commit
    if last is None or business_date(last) < business_today():
        raise RuntimeError(f"BATCH_NOT_READY: {table}, last commit {last}")
```

The scheduler retries on that error, so the job just waits. This made the new outputs **more correct** than the old ones, and also **different** from them, which is awkward when the acceptance test is "match the old system".

To prove the difference was about inputs and not logic, we **replayed** the old Databricks code with its inputs pinned (via time travel) to the previous day's version. The output matched the production Databricks table row for row. With inputs pinned to the current day, the replay matched the new platform's fingerprint. That turned "probably a timing issue" into evidence.

### The missing rows were our own fault

One fact table was a few hundred rows short on a couple of business dates. The obvious suspect was the purchase-order input, but its contents matched on both sides exactly.

The real cause: the new platform's scheduled runs had **failed several days in a row** because of a job-submission error in the platform. The pipeline only rewrites a short rolling window, so those dates were never refreshed, and their row counts matched an older input batch exactly. The lesson: when the new platform is "missing data", check its own run history before blaming the old system.

## Performance: faster, until it wasn't

| Pipeline | Target platform vs Databricks |
| --- | --- |
| Large fact tables | about 6× faster |
| Daily item-location review | about 3× faster |
| Shipment detail | about 4× faster |
| Stock and sales components | about 3× **slower** |

The slow one wasn't the engine. Databricks gave that job an autoscaling multi-node cluster. On the new platform it was configured with **a single small executor**. Resource configuration doesn't migrate itself; check every job's executor count and size explicitly, and compare runtimes from the first week.

## Where AI helped, and where it didn't

I did most of this work with an AI coding assistant (Claude Code) in the terminal. It changed how much ground one person could cover.

**Where it helped most**

- **Inventory.** Pulling the source of every notebook through the platform APIs, extracting every table each one reads and writes, and building a map of which pipeline depends on which. That's a few days of manual work; it took hours.
- **Diffing at scale.** Cell-by-cell comparison between original and migrated notebooks is exactly what found the flipped signs. Nobody would spot those by reading.
- **Narrow, guarded fix scripts.** The date-offset repair above, catalog path rewrites, publish-and-verify scripts. Each one changed only what it could prove and stopped on anything ambiguous.
- **Read-only verification.** Fingerprint queries, snapshot timelines, replay experiments, and the acceptance report itself, in English and Chinese.

**Where it needed a short leash**

- **It overclaims if you let it.** At one point it summarized that publishing hadn't changed any task references. Reading the configuration back showed otherwise, and the statement had to be corrected. After that, every report labeled each claim as *verified*, *inferred* or *quoted from an earlier document*.
- **Business decisions stay with people.** Which history table is the real one, whether a "more correct" output is acceptable, when to switch production over.
- **Production writes need a human yes.** Our rules: read-only by default; test runs write only to tables with a test suffix; never edit someone else's production notebook; no automatic retries after a publish times out, because you don't know whether the first call went through.
- **"Make it pass" is the wrong goal.** An assistant told to get a job green will find a way, for example by creating the missing table. The goal has to be "match the old output, and explain every difference".

## A checklist for your next Spark platform migration

1. Scan every notebook for platform-specific APIs and syntax before converting anything.
2. Check every source table on the new catalog with a real `SELECT ... LIMIT 1`, not just a name match.
3. Diff converted SQL against the original. Look hard at **date arithmetic, signs, units and implicit casts**.
4. Find tables the pipelines assume already exist, and initialize them from real data, never empty.
5. Point first runs at test tables, never at production outputs.
6. After every change, verify the **published** version, not the saved one.
7. Compare outputs with pinned versions and per-column fingerprints, not row counts.
8. Record when each input landed. Timing differences explain more mismatches than logic bugs do.
9. Set compute resources for every job explicitly and compare runtimes in week one.
10. Keep the old jobs running until the comparison is signed off. That's your rollback.

## Takeaways

- The bugs that hurt in a migration don't throw errors. Date arithmetic and implicit casts are the first places to look.
- "Same output as the old system" is a good acceptance test, as long as you're ready to find out the old system was wrong.
- Fingerprinting pinned versions turns "looks the same" into evidence, and replaying with pinned inputs turns "probably timing" into proof.
- An AI assistant is a huge multiplier for inventory, diffing and verification. Give it read-only access by default, make it say how it knows each claim, and keep decisions about production with people.

If you're planning a similar migration and want a second pair of eyes, [get in touch](https://pipeprompt.com/about/). For more on parity checks, see [Automated Data Quality Checks](https://pipeprompt.com/blog/automated-data-quality-framework/).
