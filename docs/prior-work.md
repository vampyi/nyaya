# Prior work

What already exists in this space, and what it leaves open.

## Existing work

**[DAKSH Rule of Law Project](https://www.dakshindia.org/Daksh_Justice_in_India/court-case-pendency-india)**
The closest prior art. Case and hearing data across the Supreme Court, High
Courts and subordinate courts on a single platform. Largely covers 2016 to 2019
and is not a live, openly queryable warehouse.

**[National Judicial Data Grid](https://njdg.ecourts.gov.in/)**
The government's own pendency repository, updated daily. Publishes aggregate
counts by court, state and age bracket. Does not expose case level records, so
duration cannot be derived from it.

**[Verma, 2023](https://arxiv.org/abs/2307.10615)**
"Analyzing HC-NJDG Data to Understand the Pendency in High Courts in India".
Built from snapshots of the NJDG portal taken over 73 days between August 2017
and December 2018, covering 24 High Courts. Aggregate scrapes rather than case
records. Notable for reporting errors in NJDG's own published statistics,
including the number of judges per High Court.

**[Cornell Journal of Law and Public Policy](https://ojs.law.cornell.edu/index.php/joal/article/download/124/132/486)**
"Estimating Time to Clear Pendency of Cases in High Courts". A modelling paper
with no accompanying open pipeline.

**[Indian Disposal Time Index 2026](https://blogs.ecourtsindia.com/2026/04/27/the-indian-disposal-time-index-2026/)**
Published April 2026, covering 14 High Courts and the district tier in 12 states.
Built from a sample of 3,308 cases.

**[Ash et al., 2022](https://paulnovosad.com/pdf/india-judicial-bias.pdf)**
"In-group bias in the Indian judiciary". The paper behind the Development Data
Lab district court release. Scraped 77 million case records from eCourts covering
2010 to 2018, then filtered to 5.3 million for analysis, discarding roughly 93
percent to get clean identification. Two details are directly relevant here.
Only 27 percent of case dispositions could be coded clearly as a good or bad
outcome, with the rest recorded as "disposed" and nothing more. And the case to
judge join had to be made on the judge's designation plus filing date because no
judge identifier exists, costing a further 17 percent of records, with 81,232
judges sharing only 22,413 distinct names.

**[vanga/indian-high-court-judgments](https://github.com/vanga/indian-high-court-judgments)**
The scraper that produces the AWS dataset this project reads. Ingestion only,
with no modelling or analytics layer.

## What is left open

The scraper exists and the records are public. What does not exist is an open,
continuously updating, case level warehouse over the High Court corpus. Every
piece of work above either stops at aggregate counts, covers a fixed historical
window, or rebuilds the data privately for a single paper and discards the
cleaning afterwards.

Two specifics sharpen this. Verma found errors in NJDG's own statistics, so
independent verification of official judicial numbers is a real need rather than
a manufactured one. And the most recent published disposal time index used 3,308
sampled cases, which is roughly the size of a single bench year partition in the
dataset this project reads, because assembling more was too expensive.

The contribution here is engineering, not the discovery of the topic: a
reproducible, tested, incremental pipeline over the full corpus.

## Methodology worth borrowing

Report medians and percentiles rather than means. Duration is heavily right
skewed and the mean misleads.

Segment by case type. A writ petition and a criminal appeal are not comparable,
and case type has to be parsed out of the `title` field.

State exclusion rules explicitly. Every paper above dropped records. Ash et al.
documented theirs, which makes the result auditable.
