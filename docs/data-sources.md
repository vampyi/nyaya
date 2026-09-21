# Data sources

Every command here was run against the live buckets and returned what is
described. Measurements are dated so they can be rechecked.

## Indian High Court Judgments (primary)

| | |
|---|---|
| Registry | https://registry.opendata.aws/indian-high-court-judgments/ |
| Scraper | https://github.com/vanga/indian-high-court-judgments |
| Licence | CC-BY-4.0 |
| Bucket | `indian-high-court-judgments`, region `ap-south-1` |
| Volume | 17.8M judgments, 25 High Courts, 45 benches, year partitions 1950 to 2026 |
| Metadata size | 8,018,089,449 bytes across 1,494 Parquet files (measured 2026-09-16) |
| Credentials | None required, `--no-sign-request` works |

A further 1.25 TiB of judgment PDFs sits in the same bucket. Not used here, the
structured metadata is the product.

```bash
aws s3 ls s3://indian-high-court-judgments/metadata/parquet/ --no-sign-request

aws s3 cp s3://indian-high-court-judgments/metadata/parquet/year=2026/court=1_12/bench=kashmirhc/metadata.parquet . --no-sign-request
```

Path layout is `metadata/parquet/year=<YYYY>/court=<code>/bench=<name>/metadata.parquet`,
partitioned on decision year. Court codes contain `~` in the data but `_` in the
S3 path, so `1~12` is found at `court=1_12`.

Court codes are opaque and cannot be guessed. `court=16_20/bench=thcnc` is the
High Court of Tripura, not Telangana. A lookup seed is needed.

### Update frequency

The registry page says the dataset updates **quarterly**. The scraper repository
says it is **synced daily via GitHub Actions**. Observation supports daily:
Parquet files were rewritten on 2026-09-16 between 13:54 and 14:19, the same day
they were inspected, and decision dates in the 2026 partition ran to 2026-09-15.

Treat the registry metadata as stale. Confirm cadence by re-measuring over a
period before settling the pipeline schedule.

### Schema

Twelve columns. Verified identical across four partitions spanning three courts
and two years (J&K 2024 and 2026, Patna 2024, Tripura 2024).

| Column | Type | Notes |
|---|---|---|
| `cnr` | VARCHAR | National case identifier, for example `JKHC010010602025` |
| `title` | VARCHAR | Case type embedded as a prefix, for example `WP(C)/519/2025 of <party> Vs <party>` |
| `description` | VARCHAR | |
| `judge` | VARCHAR | Comma separated multi judge string, needs a bridge table |
| `date_of_registration` | VARCHAR | String in `DD-MM-YYYY` |
| `decision_date` | TIMESTAMP_NS | Type mismatch with the field above |
| `disposal_nature` | VARCHAR | Free text, inconsistent |
| `court` | VARCHAR | Full name, and it changes, see below |
| `court_code` | VARCHAR | For example `1~12` |
| `pdf_link` | VARCHAR | |
| `pdf_exists` | BOOLEAN | |
| `raw_html` | VARCHAR | Large, exclude from the warehouse |

The scraper README documents an `order_number` field. It does not exist in any
published partition checked.

## Measured findings

Unless stated otherwise the sample is J&K High Court, Srinagar bench, 2026
partition, 3,300 rows, measured 2026-09-16.

### Duration

Computed as `decision_date` minus `date_of_registration`.

Median 264 days, mean 605 days, maximum 8,229 days. All 3,300 registration dates
parsed cleanly and there were zero negative durations, so the base metric is
sound.

The median to mean gap is the headline. Most cases resolve inside a year and a
long tail drags the mean to more than double. Report medians and percentiles.

### Disposal outcomes

```
1547  Disposed Off
 539  Dismised                        misspelled at source
 538  DISMISSED FOR NON PROSECUTION   different casing
 488  Disposed Off as Withdrawn
 173  Dismissed as Infractuous        misspelling of Infructuous
  15  Transferred CAT SRINAGAR
```

Six variants of roughly three concepts in one bench. Across 25 courts expect
hundreds collapsing to fewer than ten canonical outcomes. A seed file mapping raw
to canonical is the right shape.

Note that only "Disposed Off" implies a decision on the merits. Dismissal for non
prosecution means nobody appeared, and withdrawal means the petitioner gave up.
Around a third of this bench's cases received no judicial determination.
Averaging duration across all of them measures something other than how long
justice takes.

### Judge multiplicity

2,757 records with one judge, 542 with two, 1 with three. Single judges hear
routine matters, division benches hear appeals and constitutional questions, so
bench strength correlates with case importance and is a variable to segment on
rather than average over.

The string carries honorifics and inconsistent whitespace, for example
`HON'BLE MR. JUSTICE  WASIM SADIQ  NARGAL` with a double space.

### The current year partition differs from closed partitions

This is the finding that shapes the pipeline.

```
partition          rows      distinct cnr   duplicate cnr   distinct court names
Patna 2024      123,106         123,106               0            1
Tripura 2024      2,149           2,149               0            1
J&K 2024          5,129           5,129               0            1
J&K 2026          3,300           3,288              12            2
```

Only the live partition contains duplicates, and only the live partition contains
more than one name for the same court. Closed years are clean on both counts.

The duplicates are genuine repeat disposals rather than corruption:

```
JKHC010012102016  OWP/370/2016  GREEN LAND CEMENTS  registered 05-01-2016
  2026-04-18  DISMISSED FOR NON PROSECUTION  Justice Nargal
  2026-08-05  Disposed Off                   Justice Bharti
```

Same case, same registration date, two decision dates, two judges, two outcomes.
A case dismissed for non prosecution can be restored on application and decided
later.

Consequences:

- `cnr` is unique in every closed partition checked but not in the live one, so
  uniqueness is a test that must run rather than an assumption to encode. The
  merge key should be `cnr` plus `decision_date`.
- The current year has to be handled differently from closed years. Validating
  the pipeline only against history would miss every problem in this table.
- Whether duplicates also appear across partitions, where a case is decided in
  2024 and again in 2026, is untested. It needs a multi year load to answer.

### Court names lag the legal record

Within the 2026 partition, split by decision date rather than registration date:

```
High Court of Jammu and Kashmir              2,711 rows   decided 2026-01-14 to 2026-08-11
High Court of Jammu and Kashmir and Ladakh     589 rows   decided 2026-08-04 to 2026-09-15
```

The overlap between 4 and 11 August suggests a gradual rollout across servers.

Legal timeline for comparison. The court was reconstituted when the
[Jammu and Kashmir Reorganisation Act, 2019](https://en.wikipedia.org/wiki/Jammu_and_Kashmir_Reorganisation_Act,_2019)
took effect on 31 October 2019, becoming the Common High Court of the UT of
Jammu and Kashmir and the UT of Ladakh. It took its present name by Presidential
Order in [July 2021](https://www.barandbench.com/news/renamed-high-court-of-jammu-and-kashmir-and-ladakh)
([LiveLaw](https://www.livelaw.in/news-updates/jammu-kashmir-high-court-renamed-as-high-court-of-jammu-and-kashmir-and-ladakh-177623)).

So the portal displayed a superseded name for roughly seven years, skipped the
interim name entirely, and adopted the current one five years late. The attribute
changes when the source system catches up, not when the fact changes. That is the
ordinary case for a slowly changing dimension, and it is measured here rather
than assumed. Overwriting in place would silently rewrite 2,711 rows of history.

### Case type is court specific and era specific

Case type is the prefix in `title`. Registration year range per type in the
sample:

```
WP(C)   2019 to 2026   1340        SWP   2009 to 2019    56
CRM(M)  2019 to 2026    431        OWP   2008 to 2019    54
HCP     2024 to 2026    278
```

`SWP` and `OWP` stop at 2019 and `WP(C)` begins where they end. The timing is
consistent with the 2019 reorganisation, but no primary source confirming a
formal renaming of these case types was found. The pattern is measured, the cause
is inference, and it should be checked against another court before being stated
as fact.

Either way, grouping on the raw code splits J&K writ petitions across three
labels and makes any trend line crossing 2019 meaningless. A
`(court, raw_code)` to canonical type mapping is required.

### There are no pending cases in this data

Every record carries a decision date. The dataset covers concluded cases only,
so national pendency figures cannot be derived from it.

Registration years for cases decided in 2026 in the sample:

```
2026  1326    2024  391    2022  156    2020   46
2025   815    2023  287    2021   96    2019   35
```

Fast cases dominate any single year's snapshot because slow ones have not
finished yet. The 264 day median is biased downward. The same applies to filing
rates, which look artificially thin in recent years because only filings that
have already been disposed are visible. There is no correction, only disclosure.

## Indian Supreme Court Judgments (secondary)

Bucket `indian-supreme-court-judgments`, same access pattern. Partitions
`year=2023` through `year=2026` confirmed present.
Registry: https://registry.opendata.aws/indian-supreme-court-judgments/

Adds a third tier. Not needed for the first phase.

## Development Data Lab, district courts (historical)

https://www.devdatalab.org/judicial-data and
https://justicehub.in/dataset/e-courts-dataset-2010-2018

81M district court cases covering 2010 to 2018. Per case: state, district, court,
case type, filing date, decision date, legal codes, petitioner and defendant
gender, outcome. Fully anonymised, so individual judges and litigants cannot be
identified.

Licensed CC BY-NC-SA 4.0, which is non commercial only. Static, ends 2018. Useful
for historical depth and district tier coverage, not for anything current.

## Privacy

The district court data above is anonymised. The High Court judgments are not,
and published judgments carry real party names.

A name searchable index over those records would republish personal data and
engage India's DPDP Act 2023, alongside the right to be forgotten rulings courts
have issued about their own judgments. Aggregate and time series views are fine.
A case or person search interface is not, and this project will not build one.
