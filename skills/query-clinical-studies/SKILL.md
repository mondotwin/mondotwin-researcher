---
name: query-clinical-studies
description: Count, compare and summarise registered clinical trials with one read-only SQL query over mondoTWIN's trial tables. Use for questions a trial search cannot answer by listing records - how many trials match, distributions (by phase, status, year, sponsor class, country, registry), rankings (most tested drugs, most active sponsors), enrolment statistics by phase, why terminated trials stopped, which primary endpoints and time frames comparable trials used, how often a criterion appears - and when the user asks what is in the database, which registries it covers or how current it is.
---

# Query clinical studies

Answer counting and comparison questions about registered trials with SQL, and show the numbers together with the query that produced them and the registry ids behind them.

## Ground rules

1. Every number, trial, id and date in the answer comes from a mondoTWIN tool result. Never invent a count, a trial or a value, and never extrapolate a number the query did not return. Background knowledge may suggest codes and search terms; it is not evidence.
2. Cite each trial by its registry id linked to its registry page: select `linkouts->>'registry_url'`, or use `https://clinicaltrials.gov/study/<NCT id>` for an NCT id. Every count cites its query: give the SQL and state its filters in words.
3. Say plainly when a query returned 0 rows, failed or was truncated, with the filter used. A count is a count of records in this database, not of every trial ever run: "not in the database" is not "does not exist".
4. State the data as-of date: ClinicalTrials.gov as of 13 Jul 2026; EU CTR/CTIS, ISRCTN, ANZCTR and UMIN as of 14 May 2026. Later registrations and updates are missing.
5. No patient data. The tables hold registry records about trials, not participants. If the user pastes identifiable patient information, do not repeat it, never put it in a query, and ask them to remove it.
6. The output is a draft for qualified professionals to validate. Say so in one line at the end.
7. Answer in the user's language. Keep registry ids, titles and quoted registry text in their original language.
8. Use `query_clinical_studies` for counts and comparisons, and `search_clinical_studies` or `get_clinical_study` to read a single trial in full. If the mondoTWIN tools are not available, tell the user to connect the mondoTWIN connector, and stop. Do not answer from memory as though from the database.

## The tool

`query_clinical_studies(query, max_rows=200)` runs one read-only `SELECT` or `WITH` statement.

- Readable tables: `clinical_studies`, `clinical_study_disease`, `clinical_study_drug`, and `clinical_study_sponsor`. Write the names without a schema prefix. Nothing else is readable; there are no patent, prediction or source-registry tables behind it.
- 10-second statement timeout. `max_rows` defaults to 200 (hard cap 1000). `truncated: true` means more rows exist: aggregate in SQL instead of paging through rows.
- Returns `{columns, rows, row_count, truncated, elapsed_ms}`, or `{error, sqlstate}`. On an error, read the message, fix the query (a column name, a cast, a missing `GROUP BY`) and retry. `57014` is the timeout: filter on an indexed column first (rule 12).
- One statement per call: no semicolon-separated batches, no writes, no temporary tables.

Columns, JSONB shapes, enum values and coverage per registry: `references/data-model.md`. Tested queries by question: `references/sql-recipes.md`.

## Rules that prevent wrong numbers

1. **One trial, one row.** A row of `clinical_studies` is one deduplicated trial under one registry id. Joining the junction tables multiplies rows: count `count(DISTINCT cs.id)`, or filter with `cs.id IN (SELECT clinical_study_id …)`.
2. **Conditions by code, not by words.** Find the ICD-11 codes first (recipe A1), then filter `clinical_study_disease.icd11_code LIKE 'CB03.4%'` (the code and its children) with `role = 'primary'` for the trial's main condition. Codes were assigned by a model from the registry's condition text, so read the titles of the trials you cite.
3. **Phase.** Values are `phase_1` to `phase_4` and `NA`. `NA` covers non-drug trials and almost all ANZCTR, UMIN and ISRCTN records, which state no phase: a phase filter drops them. CTIS records keep phase and status only in `classification` (`COALESCE(cs.phase, cs.classification->>'phase')`).
4. **Status.** Reliable for ClinicalTrials.gov and EU CTR records. UMIN records are all `completed`; ISRCTN records are mostly `active_not_recruiting`, even long-finished ones. Say so when they are in a status count.
5. **Enrolment.** ClinicalTrials.gov gives one number: the actual count for completed, terminated and withdrawn trials, the anticipated target for the others. Both `enrollment_actual` and `enrollment_target` hold it, so `enrollment_actual` repeats the target for open trials. Treat it as an actual size only for completed or terminated trials. EU CTR targets are often for one member state ("national").
6. **Dates.** `start_date` and `end_date` exist for ClinicalTrials.gov and ISRCTN records only; a time trend covers those two registries. Placeholder dates such as `1900-01-01` and far-future years exist: bound the range.
7. **Machine-generated fields are not registry content:** `description->>'hypothesis'`, `description->'novelty_tags'`, `clinical_outcome`, `positive_results`, everything in `results` except `why_stopped` and `results_available`, `population->>'line_of_therapy'`, `clinical_study_drug.is_novel_for_disease`, the ICD-11 and Orphanet coding and the drug normalisation. Never report `clinical_outcome` as a trial result. Group by such a field only if the user accepts that, and label it "machine-classified".
8. **Age.** Use `population->>'age_min_years'` and `'age_max_years'` (the top-level age columns are empty). `0` and `99` mean "not captured": read `criteria_markdown` for age limits.
9. **Country and sponsor class.** `organization->'main_center'->>'country'` is the main or first site's country: not normalised ("UK" and "United Kingdom"), "EEA only" on CTIS records, empty on many. `sponsor_class` is `unknown` on most UMIN records and unreliable on CTIS records.
10. **Drugs.** `canonical_drug_id` is a DrugBank id (`DB…`) or `name::<lower-cased name>` when no DrugBank match was found. Group by the id, not the display name. The normalisation can be wrong for sponsor code names. `clinical_study_drug` holds only drug and biologic components; placebo, devices and procedures are in `arms` only.
11. **Cross-registry ids.** A trial registered in several registries appears once, under one id. `linkouts->'other_registry_ids'` always holds a junk `"id": "type"` entry and is not a reliable cross-registry map.
12. **Speed.** Filter first on an indexed column (`clinical_study_disease.icd11_code`, `clinical_study_drug.canonical_drug_id`, `clinical_studies.primary_registry_id`, `phase`, `status`, `sponsor_class`), then read JSONB. Grouping all 374,000 trials by a JSONB field (country, endpoint text) exceeds the timeout. Never `SELECT *` from `clinical_studies`: name the columns.

## Steps

1. **Frame the question** as a set of trials (condition, drug, phase, status, registry, period), a measure (count, median, share) and a grouping. If the condition or drug is missing, ask one question; otherwise state your assumptions.
2. **Resolve codes**: ICD-11 codes for the condition (recipe A1), DrugBank ids for the drug (recipe A2). Tell the user which codes you kept.
3. **Run the aggregate** from `references/sql-recipes.md`, adapted. Keep to about 6 queries.
4. **Check it.** Totals should add up. When a number surprises you, list a few rows behind it (id, title) to confirm the filter caught what was meant.
5. **Pull the ids** behind the headline numbers, with `registry_url`, so the user can check them at the source.
6. **Write the answer** in the format below. For a close reading of single trials, hand over to `get_clinical_study`.

## Output format

**[The question in one line]** · Data as of: ClinicalTrials.gov 13 Jul 2026; other registries 14 May 2026.
Scope: *[ICD-11 codes or DrugBank ids, filters, registries included]*.

*[One results table, columns named in plain words.]*

- **What the numbers show:** two to four sentences, with the counts.
- **Trials behind it:** linked registry ids (at most 10, or the ones the user asked about).
- **Limits:** registries without phase, status or dates in scope; machine-classified fields used; truncated results; codes possibly missed.
- **Query:** the SQL of each query used, in a code block, so the user can rerun or adapt it.
- One closing line: a draft from registry data for qualified professionals to validate.
