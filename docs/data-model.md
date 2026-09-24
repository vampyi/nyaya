# Data model

Dimensional model for the judgment warehouse, designed with the Kimball four
step process: select the business process, declare the grain, identify the
dimensions, identify the facts
([reference](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/four-4-step-design-process/)).

Nothing here is built yet. This is the design the build follows.

## Business process

A High Court disposes of a matter by issuing a judgment or order.

The model describes that process, not the questions asked of it. Disposal time is
a calculation over these events rather than a stored measure, which keeps the
model open to questions that have not been asked yet.

## Grain

> One row per judgment or order issued by an Indian High Court.

This is the atomic grain, the lowest level the source captures. Coarser grains
were rejected because detail can always be rolled up and never recovered once
discarded.

Specifically, one row per case was rejected for two reasons. It discards real
events: a case dismissed for non prosecution and later restored and decided
produces two genuine orders, and collapsing them loses the restoration. And it
cannot be built incrementally, because identifying the latest order for a case
requires visibility of partitions other than the one being loaded.

### Physical key

`(cnr, decision_date)`, verified on the J&K 2026 partition:

```
total rows                       3,300
distinct cnr                     3,288   not unique
distinct (cnr, decision_date)    3,300   unique
exact duplicate rows                 0
```

`cnr` alone is not a primary key. See `data-sources.md` for the measurements.
This composite is both the merge key for incremental loads and a uniqueness test
that must run on every build rather than an assumption baked into the model.

## Dimensions

| Dimension | Grain | SCD | Source |
|---|---|---|---|
| `dim_court` | One row per bench per version | Type 2 | Source data |
| `dim_case_type` | One row per (court, raw code) | Type 1 | Seed file |
| `dim_disposal_outcome` | One row per raw outcome string | Type 1 | Seed file |
| `dim_judge` | One row per resolved judge | Type 1 | Parsed from source |
| `dim_date` | One row per calendar day | Type 0 | Generated |

### Why the SCD types differ

`dim_court` is Type 2 because the source changes court names, and it does so
years after the legal change. The 2026 partition carries two names for the same
`court_code`, split by decision date. Overwriting in place would rewrite 2,711
rows of history to say something the source never said at the time.

`dim_case_type` and `dim_disposal_outcome` are Type 1 because the mappings are
this project's own interpretation rather than facts about the world. When a
mapping is corrected, the correction should apply retroactively to every row.

`dim_date` is Type 0. Calendar facts do not change.

### Notes on individual dimensions

**`dim_court`** holds court and bench in one table rather than two. A bench
belongs to exactly one court, so separating them would add a join without adding
information.

**`dim_disposal_outcome`** carries an `is_adjudicated` boolean. This matters more
than the cleaned label. "Disposed Off" is a decision on the merits, while
dismissal for non prosecution and withdrawal are not. Around a third of records
in the sample bench received no judicial determination, and mixing them into a
single duration figure measures something other than how long a decision takes.

**`dim_date`** is a role playing dimension. Registration date and decision date
both join to it with different meanings. One physical table, joined twice.

**`dim_judge`** exists to support bench strength, meaning whether one, two or
three judges heard a matter, which correlates with case importance. Judge name
resolution is non trivial: Ash et al. found 81,232 judges sharing only 22,413
distinct names in the district court data, and the strings here carry honorifics
and inconsistent whitespace.

## Facts

### `fact_judgment`, transaction fact at the declared grain

| Column | Type | Notes |
|---|---|---|
| `cnr` | degenerate dimension | Identifier with nothing to describe |
| `court_sk` | FK | |
| `case_type_sk` | FK | |
| `outcome_sk` | FK | |
| `registration_date_sk` | FK | Role playing |
| `decision_date_sk` | FK | Role playing |
| `days_to_this_order` | measure | Non additive, average but never sum |
| `judge_count` | measure | Non additive. 1, 2 or 3 |

Row counts are implicit and need no stored column.

### `fact_case`, accumulating snapshot derived from `fact_judgment`

One row per case, updated as the case passes milestones.

| Column | Notes |
|---|---|
| `cnr` | degenerate dimension |
| `court_sk`, `case_type_sk` | FK |
| `registration_date_sk` | FK |
| `first_disposal_date_sk` | FK |
| `final_disposal_date_sk` | FK |
| `final_outcome_sk` | FK |
| `days_to_first_disposal` | Stable once computed |
| `days_to_final_disposal` | Can change if the case is restored again |
| `order_count` | Number of orders issued |
| `was_restored` | True where `order_count` exceeds one |

Both durations are reported rather than one, because they answer different
questions and a restored case makes them diverge. First disposal is the point
the court cleared the matter from its list. Final disposal is the litigant's
actual wait. For the Green Land Cements case the two differ by four months
across a ten year span. `was_restored` flags where the distinction applies.

### `bridge_judgment_judge`

One row per judgment per judge, resolving the many to many between the two. A
judgment has one to three judges, a judge has many judgments, so neither side
can hold the key.

## Aggregation constraint

Medians do not compose. A median per court cannot be combined into a national
median, so the headline metric cannot be precomputed into a summary table and
must be calculated from atomic rows at query time.

This is the practical reason the atomic grain was not optional.

## Publishing policy on judges

The model resolves individual judges. Published outputs will not rank them or
report metrics at judge level.

Case duration is confounded by case complexity, adjournments requested by the
parties, appeals, and bench vacancies. A ranking built on it would attribute
systemic delay to individuals and would be wrong on its own terms. Judge data is
used only to derive bench strength.

## Schema

```
                              dim_date
                         (role playing, Type 0)
                          /                \
             registration/                  \decision
                        v                    v
   dim_court ------>  fact_judgment  <------ dim_case_type
   (SCD Type 2)       ---------------        (Type 1, seed)
                      cnr          DD
                      court_sk
                      case_type_sk
                      outcome_sk
                      registration_date_sk
                      decision_date_sk
                      days_to_this_order
                      judge_count
                        ^              ^
                        |              |
             dim_outcome              bridge_judgment_judge ---> dim_judge
             (Type 1, seed)           (many to many)
             is_adjudicated


   fact_case   (accumulating snapshot, derived from fact_judgment)
```

`dim_court`, `dim_case_type` and `dim_date` are conformed, shared by both fact
tables with identical meaning.

## Scope: hearing data

A second source carries hearing level records for Allahabad and Bombay High
Courts, with partial coverage elsewhere (`data-sources.md`). It sits outside the
conformed model, which stays on the 12 column data available for all 25 courts.

For Allahabad and Bombay the extra fields support a separate, more detailed
analysis: hearings per case, intervals between hearings, which judge sat on each
listing, filing date as distinct from registration date, and acts and sections
under which the case was brought.

Those outputs are reported as covering those two courts specifically, not as
national figures.

## Open questions

Whether duplicate `cnr` values occur across partitions as well as within the
live one, where a case is decided in one year and again in a later year. Only a
multi year load answers this, and it affects how `fact_case` is incrementally
rebuilt.

Whether `SWP` and `OWP` were formally replaced by `WP(C)` after the 2019
reorganisation, or whether the pattern in the data has another cause. The
mapping seed needs this settled for at least one other court.
