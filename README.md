# Nyaya

An open warehouse of Indian High Court judgment records, updated daily, built to
measure how long cases actually take.

17.8 million judgments, 25 High Courts, 1950 to 2026.

## Objective

India has roughly 58.6 million pending court cases, about 89 percent of them in
district courts, and more than 180,000 that have been pending for over 30 years
([NJDG](https://njdg.ecourts.gov.in/), [summary](https://en.wikipedia.org/wiki/Pendency_of_court_cases_in_India)).

The government's own National Judicial Data Grid publishes how many cases are
pending. It does not publish how long they take. That number has to be derived
from case level records, and while those records are public, nobody has assembled
them into something queryable and kept it current. Researchers who need the data
rebuild it privately, publish once, and the cleaning is discarded. The most recent
published disposal time index for India worked from a sample of 3,308 cases.

Nyaya assembles the judgment records published by all 25 High Courts into a
warehouse that refreshes daily, so disposal time can be measured by court, bench,
case type and year, and so the data quality problems in the official record are
visible rather than silently cleaned away.

Intended users are journalists, researchers, and policy and civil society
organisations. It is not a case lookup tool and will not help an individual
litigant with their own matter.

## Status

Early. Sources verified, schema profiled across four partitions, findings below
are measured. No pipeline built yet.

## Data source

Indian High Court Judgments, published on the AWS Open Data Registry under
CC-BY-4.0.

| | |
|---|---|
| Registry | https://registry.opendata.aws/indian-high-court-judgments/ |
| Bucket | `indian-high-court-judgments`, region `ap-south-1` |
| Scraper | https://github.com/vanga/indian-high-court-judgments |
| Credentials | None. `--no-sign-request` works. |
| Volume | 8.0 GB of Parquet metadata across 1,494 files |

Path layout is `metadata/parquet/year=<YYYY>/court=<code>/bench=<name>/metadata.parquet`,
partitioned by decision year. Court codes use `~` inside the data and `_` in the
S3 path, so `1~12` lives at `court=1_12`.

A separate 1.25 TiB of judgment PDFs exists in the same bucket. This project uses
the structured metadata only.

```bash
# list year partitions
aws s3 ls s3://indian-high-court-judgments/metadata/parquet/ --no-sign-request

# one bench
aws s3 cp s3://indian-high-court-judgments/metadata/parquet/year=2026/court=1_12/bench=kashmirhc/metadata.parquet . --no-sign-request
```

### Source documentation is inaccurate in two places

Worth recording, since this project is partly about the reliability of official
judicial data.

The AWS registry entry states the dataset updates **quarterly**. The scraper
repository states it is **synced daily via GitHub Actions**, and files were
observed rewritten the same day they were inspected, with decision dates current
to the previous day. Daily is correct.

The scraper README describes an `order_number` field. It is not present in the
published Parquet. Four partitions were checked across three courts and two years
and all carry the same 12 columns.

## Schema

Twelve columns, identical across every partition checked.

| Column | Type | Notes |
|---|---|---|
| `cnr` | VARCHAR | National case identifier, for example `JKHC010010602025` |
| `title` | VARCHAR | Case type is embedded here and has to be parsed out |
| `description` | VARCHAR | |
| `judge` | VARCHAR | Comma separated, one to three judges per record |
| `date_of_registration` | VARCHAR | String, `DD-MM-YYYY` |
| `decision_date` | TIMESTAMP_NS | Timestamp, so the two date fields disagree on type |
| `disposal_nature` | VARCHAR | Free text, inconsistent across and within courts |
| `court` | VARCHAR | Full court name |
| `court_code` | VARCHAR | For example `1~12` |
| `pdf_link` | VARCHAR | |
| `pdf_exists` | BOOLEAN | |
| `raw_html` | VARCHAR | Large, excluded from the warehouse |

## Early findings

Measured, not estimated. Sample is the Jammu and Kashmir High Court, Srinagar
bench, 2026 partition, 3,300 records, unless stated otherwise.

**Disposal time is heavily right skewed.** Median 264 days, mean 605 days, maximum
8,229 days. Reporting the mean would overstate typical duration by more than
double. Medians and percentiles only.

**Outcome text is inconsistent within a single bench.**

```
1547  Disposed Off
 539  Dismised                        misspelled at source
 538  DISMISSED FOR NON PROSECUTION   different casing
 488  Disposed Off as Withdrawn
 173  Dismissed as Infractuous        misspelling of Infructuous
  15  Transferred CAT SRINAGAR
```

Six spellings of roughly three concepts in one bench. Note also that only the
first category represents a decision on the merits. Treating all six as
equivalent when measuring duration would be wrong.

**The current year partition behaves differently from closed ones.** The 2026
partition contains 12 duplicate `cnr` values in 3,300 rows, where the same case
was disposed twice under different judges with different outcomes. It also
contains two different names for the same court. Neither appears in closed years.

```
Patna 2024      123,106 rows   0 duplicate cnr   1 court name
Tripura 2024      2,149 rows   0 duplicate cnr   1 court name
J&K 2024          5,129 rows   0 duplicate cnr   1 court name
J&K 2026          3,300 rows  12 duplicate cnr   2 court names
```

**Court names in the source lag the legal record by years.** The 2026 partition
carries both "High Court of Jammu and Kashmir" and "High Court of Jammu and
Kashmir and Ladakh", split by decision date, with the change appearing in early
August 2026. The court was reconstituted on
[31 October 2019](https://en.wikipedia.org/wiki/Jammu_and_Kashmir_Reorganisation_Act,_2019)
and formally took its current name by Presidential Order in
[July 2021](https://www.barandbench.com/news/renamed-high-court-of-jammu-and-kashmir-and-ladakh).
The portal adopted it five years later, and skipped the interim name entirely.

**Volume varies by more than an order of magnitude between courts.** Patna
produced 123,106 records in 2024 against 5,129 for Jammu and Kashmir.

## Known limitations

**There are no pending cases in this data.** Every record has a decision date, so
the dataset contains only cases that have already concluded. The national pendency
figures quoted above come from a different source and cannot be derived from here.

**This makes every duration figure survivorship biased.** Slow cases are invisible
until they finish. Among cases decided in 2026 in the sample bench, 1,326 of 3,300
were also registered in 2026, while only 35 dated from 2019. The reported median
is therefore biased downward, and the same flaw applies to filing rates, which
look artificially thin in recent years. There is no correction for this, only
disclosure. The metric is disposal time among disposed cases.

**High Courts are not where the backlog is.** About 89 percent of pending cases
sit in district courts, which this dataset does not cover.

**Case type coding is court specific and changes over time.** In the sample, the
Jammu and Kashmir codes `SWP` and `OWP` stop appearing after 2019 and `WP(C)`
begins, which is consistent with the 2019 reorganisation but is not confirmed by
a primary source. Any trend line crossing that boundary compares different
labels for the same thing unless the codes are mapped first.

**No judge level analysis.** Duration is confounded by case complexity,
adjournments requested by the parties, appeals and bench vacancies. Analysis stays
at court, bench and case type level.

**No case or party search.** High Court judgments carry real party names and are
not anonymised. A searchable index over them would republish personal data and
engage India's DPDP Act 2023. Aggregate and time series views only.

## Planned architecture

Batch ELT. Nothing here is built yet.

```
S3 (public, daily)
      |
      v
  DuckDB  ................  local development and the deployed warehouse
      |
      v
   dbt  ..................  staging, dimensional models, tests
      |                     second target: Snowflake
      v
 Dagster  ................  scheduling and asset lineage
      |
      v
 Evidence  ...............  static site
```

DuckDB rather than Spark because 8 GB does not need a cluster, and a single node
reads the corpus faster than a JVM cluster can start. Snowflake is configured as a
second dbt target to demonstrate portability, not as the deployed warehouse.

## Repository layout

```
nyaya/
  docs/
    data-sources.md    verified access, measured profiling, findings
    prior-work.md      existing research and what it leaves open
  README.md
  LICENSE
```

## Licence and attribution

Code in this repository is MIT licensed. See `LICENSE`.

The underlying judgment data is published under
[CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/) and is credited to the
[Indian High Court Judgments dataset](https://registry.opendata.aws/indian-high-court-judgments/)
on the AWS Open Data Registry, scraped from the eCourts portal by
[vanga/indian-high-court-judgments](https://github.com/vanga/indian-high-court-judgments).
Any published derivative of the data carries the same attribution requirement.
