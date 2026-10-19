# SQL recipes

Queries for `query_clinical_studies`, by question. Each is one `SELECT` or `WITH` statement and runs well inside the 10-second limit. Replace the example codes (`CB03.4` idiopathic pulmonary fibrosis, `BA41` acute myocardial infarction, `DB01394` colchicine) with the ones you resolved in section A. Columns and shapes: `data-model.md`.

## Contents

- Rules that prevent zero-row and over-count traps
- A. Find the codes (condition, drug)
- B. Trials by drug and condition
- C. What gets tested for a condition
- D. Counts and distributions (phase and status, start year, design)
- E. Why terminated trials stopped
- F. Enrolment by phase
- G. Primary endpoints and time frames of comparable trials
- H. Trials by sponsor class and country; most active sponsors
- I. Eligibility criteria across trials
- J. What is in the database
- K. One trial's structured record
- Performance
- Common mistakes

## Rules that prevent zero-row and over-count traps

| Rule | Why |
|---|---|
| `count(DISTINCT cs.id)` whenever a junction table is joined | a trial with 3 drugs and 2 conditions gives 6 joined rows |
| ICD-11 by prefix: `icd11_code LIKE 'BA41%'` | trials are coded at parent or leaf level (`BA41`, `BA41.0`, `BA41.1`) |
| Keep `role = 'primary'` on `clinical_study_disease` | otherwise trials where the condition is only secondary are counted |
| Phase values are `phase_1` … `phase_4`, `NA` | `'Phase 3'` or `'PHASE3'` returns 0 rows, silently |
| CTIS phase and status: `COALESCE(cs.phase, cs.classification->>'phase')` | the scalar columns are NULL on CTIS records |
| Status counts: say which registries are in scope | UMIN is always `completed`, ISRCTN mostly `active_not_recruiting` |
| Enrolment as actual size: `status IN ('completed','terminated')` and `primary_registry = 'clinicaltrials.gov'` | the same column holds the target for open trials and national targets for EU CTR |
| Time trends: bound `start_date` (for example `BETWEEN '2000-01-01' AND '2026-07-13'`) | dates exist for ClinicalTrials.gov and ISRCTN only, and placeholders such as `1900-01-01` exist |
| Age from `population`, ignoring `0` and `99` | the top-level age columns are empty; `0` and `99` mean not captured |
| Group drugs by `canonical_drug_id` | casing and spelling variants split one drug across display names |

## A. Find the codes

### A1. ICD-11 codes for a condition

```sql
SELECT icd11_code, count(DISTINCT clinical_study_id) AS n_trials,
       (array_agg(DISTINCT display_name))[1:3] AS example_labels
FROM clinical_study_disease
WHERE role = 'primary' AND display_name ILIKE '%pulmonary fibrosis%'
GROUP BY icd11_code
ORDER BY n_trials DESC LIMIT 20;
```

`display_name` is each trial's own wording, so this finds the codes, not all the trials: filter the trials by the codes you keep (`LIKE 'CB03.4%'`). Check a code's labels before keeping it; neighbouring codes can hold a different disease.

### A2. DrugBank ids for a drug

```sql
SELECT canonical_drug_id, count(DISTINCT clinical_study_id) AS n_trials,
       (array_agg(DISTINCT drug_display_name))[1:3] AS example_names
FROM clinical_study_drug
WHERE drug_display_name ILIKE '%colchicine%'
GROUP BY canonical_drug_id
ORDER BY n_trials DESC LIMIT 10;
```

A `name::…` id is a drug without a DrugBank match (often a sponsor code name); include it when it is the same product.

## B. Trials by drug and condition

```sql
SELECT cs.primary_registry_id, cs.primary_registry, cs.title_short,
       COALESCE(cs.phase, cs.classification->>'phase') AS phase,
       COALESCE(cs.status, cs.classification->>'status') AS status,
       cs.start_date, cs.enrollment_actual, cs.linkouts->>'registry_url' AS registry_url
FROM clinical_studies cs
WHERE cs.id IN (
  SELECT dis.clinical_study_id
  FROM clinical_study_disease dis
  JOIN clinical_study_drug d ON d.clinical_study_id = dis.clinical_study_id
  WHERE dis.icd11_code LIKE 'BA41%' AND dis.role = 'primary'
    AND d.canonical_drug_id = 'DB01394')
ORDER BY cs.start_date DESC NULLS LAST;
```

Add `AND d.arm_role = 'experimental'` to keep trials where the drug is in an experimental arm, not only the comparator.

## C. What gets tested for a condition

```sql
SELECT d.canonical_drug_id, min(d.drug_display_name) AS drug,
       count(DISTINCT cs.id) AS n_trials,
       count(DISTINCT cs.id) FILTER (WHERE cs.phase IN ('phase_2', 'phase_3')) AS n_phase_2_3,
       count(DISTINCT cs.id) FILTER (WHERE cs.status = 'completed') AS n_completed,
       min(cs.start_date) AS first_start
FROM clinical_study_disease dis
JOIN clinical_study_drug d ON d.clinical_study_id = dis.clinical_study_id
JOIN clinical_studies cs ON cs.id = dis.clinical_study_id
WHERE dis.icd11_code LIKE 'CB03.4%' AND dis.role = 'primary'
  AND d.arm_role = 'experimental'
GROUP BY d.canonical_drug_id
ORDER BY n_trials DESC LIMIT 25;
```

The experimental arm also holds background and supportive drugs. To keep only the drug under test, add `AND d.is_novel_for_disease` and say the list is machine-classified.

## D. Counts and distributions

### D1. Phase and status for a condition

```sql
WITH t AS (
  SELECT DISTINCT clinical_study_id AS id FROM clinical_study_disease
  WHERE icd11_code LIKE 'CB03.4%' AND role = 'primary')
SELECT COALESCE(cs.phase, cs.classification->>'phase') AS phase,
       count(*) AS n_trials,
       count(*) FILTER (WHERE COALESCE(cs.status, cs.classification->>'status') = 'completed') AS completed,
       count(*) FILTER (WHERE COALESCE(cs.status, cs.classification->>'status') = 'terminated') AS terminated,
       count(*) FILTER (WHERE COALESCE(cs.status, cs.classification->>'status')
                        IN ('recruiting', 'not_yet_recruiting', 'active_not_recruiting')) AS ongoing,
       count(*) FILTER (WHERE cs.primary_registry IN ('umin', 'isrctn')) AS umin_or_isrctn
FROM t JOIN clinical_studies cs ON cs.id = t.id
GROUP BY 1 ORDER BY 1;
```

The last column shows how many rows carry a status that says little (rule 4 of the skill).

### D2. Trials started per year

```sql
SELECT extract(year FROM cs.start_date)::int AS start_year,
       count(DISTINCT cs.id) AS n_trials,
       count(DISTINCT cs.id) FILTER (WHERE cs.sponsor_class = 'industry') AS industry
FROM clinical_study_disease dis
JOIN clinical_studies cs ON cs.id = dis.clinical_study_id
WHERE dis.icd11_code LIKE 'CB03.4%' AND dis.role = 'primary'
  AND cs.start_date BETWEEN '2000-01-01' AND '2026-07-13'
GROUP BY 1 ORDER BY 1;
```

Covers ClinicalTrials.gov and ISRCTN records only (the others have no dates). The last full year is 2025.

### D3. Design features by phase

```sql
SELECT cs.phase, count(*) AS n_trials,
       count(*) FILTER (WHERE cs.is_randomized) AS randomised,
       count(*) FILTER (WHERE cs.classification->>'masking' IN ('double', 'triple', 'quadruple')) AS double_blind_or_more,
       count(*) FILTER (WHERE cs.classification->>'control_type' = 'placebo') AS placebo_controlled,
       round(avg(jsonb_array_length(cs.arms)), 1) AS mean_arms
FROM clinical_studies cs
WHERE cs.id IN (SELECT clinical_study_id FROM clinical_study_disease
                WHERE icd11_code LIKE 'CB03.4%' AND role = 'primary')
  AND cs.phase IN ('phase_2', 'phase_3')
GROUP BY cs.phase ORDER BY cs.phase;
```

## E. Why terminated trials stopped

```sql
SELECT cs.primary_registry_id, cs.phase, cs.status, cs.title_short,
       cs.enrollment_actual, cs.start_date, cs.end_date,
       cs.results->>'why_stopped' AS why_stopped,
       cs.linkouts->>'registry_url' AS registry_url
FROM clinical_studies cs
WHERE cs.id IN (SELECT clinical_study_id FROM clinical_study_disease
                WHERE icd11_code LIKE 'CB03.4%' AND role = 'primary')
  AND cs.status IN ('terminated', 'withdrawn', 'suspended')
ORDER BY cs.phase DESC, cs.end_date DESC NULLS LAST;
```

`why_stopped` is the registry's free text, sometimes empty. Group the reasons yourself (efficacy or futility, safety, recruitment, business or sponsor decision, COVID-19, other), give the count per group with the ids, and quote the text; do not force an unclear reason into a group. `enrollment_actual` here is the number enrolled before stopping (0 for most withdrawn trials).

## F. Enrolment by phase

```sql
SELECT cs.phase, count(*) AS n_trials,
       percentile_disc(0.25) WITHIN GROUP (ORDER BY cs.enrollment_actual) AS p25,
       percentile_disc(0.5)  WITHIN GROUP (ORDER BY cs.enrollment_actual) AS median,
       percentile_disc(0.75) WITHIN GROUP (ORDER BY cs.enrollment_actual) AS p75,
       max(cs.enrollment_actual) AS max
FROM clinical_studies cs
WHERE cs.id IN (SELECT clinical_study_id FROM clinical_study_disease
                WHERE icd11_code LIKE 'CB03.4%' AND role = 'primary')
  AND cs.primary_registry = 'clinicaltrials.gov'
  AND cs.status IN ('completed', 'terminated')
  AND cs.enrollment_actual > 0
GROUP BY cs.phase ORDER BY cs.phase;
```

Actual sizes of finished ClinicalTrials.gov trials. For planned sizes of ongoing trials, use `status IN ('recruiting', 'not_yet_recruiting', 'active_not_recruiting')` and call the numbers targets. Keep terminated trials apart (`status = 'completed'` only) when early stops would pull the median down.

## G. Primary endpoints and time frames of comparable trials

### G1. The endpoints, trial by trial

```sql
SELECT cs.primary_registry_id, cs.phase, cs.status, cs.enrollment_actual,
       e->>'label' AS primary_endpoint,
       left(e->>'description', 300) AS description,
       cs.linkouts->>'registry_url' AS registry_url
FROM clinical_studies cs
CROSS JOIN LATERAL jsonb_array_elements(cs.endpoints) e
WHERE cs.id IN (SELECT clinical_study_id FROM clinical_study_disease
                WHERE icd11_code LIKE 'CB03.4%' AND role = 'primary')
  AND cs.phase IN ('phase_2', 'phase_3')
  AND e->>'type' = 'primary'
ORDER BY cs.phase DESC, cs.start_date DESC NULLS LAST
LIMIT 60;
```

The time frame sits inside the label or the description ("at Week 52", "over 52 weeks"); write "time frame not stated" when neither gives one. One trial can have several primary endpoints (several rows). ISRCTN records have no endpoints.

### G2. How often an endpoint is primary

```sql
SELECT cs.phase,
       count(DISTINCT cs.id) AS n_trials_with_primary_endpoint,
       count(DISTINCT cs.id) FILTER (WHERE e->>'label' ~* '(FVC|forced vital capacity)') AS fvc_primary,
       count(DISTINCT cs.id) FILTER (WHERE (e->>'label') || ' ' || coalesce(e->>'description', '')
                                     ~* '(52 weeks|week 52|12 months|1 year)') AS at_one_year
FROM clinical_studies cs
CROSS JOIN LATERAL jsonb_array_elements(cs.endpoints) e
WHERE cs.id IN (SELECT clinical_study_id FROM clinical_study_disease
                WHERE icd11_code LIKE 'CB03.4%' AND role = 'primary')
  AND cs.phase IN ('phase_2', 'phase_3')
  AND e->>'type' = 'primary'
GROUP BY cs.phase ORDER BY cs.phase;
```

A text pattern counts what the registry wrote, not what was measured: check a sample of matches (G1) before quoting the share, and give the pattern used.

## H. Trials by sponsor class and country

### H1. Sponsor class by main-site country

```sql
SELECT COALESCE(NULLIF(cs.organization->'main_center'->>'country', ''), 'not stated') AS country,
       count(*) AS n_trials,
       count(*) FILTER (WHERE cs.sponsor_class = 'industry') AS industry,
       count(*) FILTER (WHERE cs.sponsor_class = 'academic') AS academic,
       count(*) FILTER (WHERE cs.sponsor_class = 'government') AS government,
       count(*) FILTER (WHERE cs.sponsor_class = 'mixed') AS mixed,
       count(*) FILTER (WHERE cs.sponsor_class = 'unknown' OR cs.sponsor_class IS NULL) AS unknown
FROM clinical_studies cs
WHERE cs.id IN (SELECT clinical_study_id FROM clinical_study_disease
                WHERE icd11_code LIKE 'CB03.4%' AND role = 'primary')
GROUP BY 1 ORDER BY n_trials DESC LIMIT 25;
```

The country is the main or first site's, not every country that recruited; names are not normalised ("UK" and "United Kingdom"; "EEA only" on CTIS records): merge them in the answer and say so. Always filter to a condition or drug first: grouping all trials by country exceeds the time limit.

### H2. Most active lead sponsors

```sql
SELECT s.normalized_sponsor_name, min(s.display_sponsor_name) AS sponsor,
       min(s.sponsor_class) AS sponsor_class,
       count(DISTINCT s.clinical_study_id) AS n_trials,
       count(DISTINCT cs.id) FILTER (WHERE cs.phase IN ('phase_2', 'phase_3')) AS n_phase_2_3,
       string_agg(DISTINCT cs.primary_registry_id, ', ') AS registry_ids
FROM clinical_study_sponsor s
JOIN clinical_studies cs ON cs.id = s.clinical_study_id
WHERE s.clinical_study_id IN (SELECT clinical_study_id FROM clinical_study_disease
                              WHERE icd11_code LIKE 'CB03.4%' AND role = 'primary')
GROUP BY s.normalized_sponsor_name
ORDER BY n_trials DESC LIMIT 15;
```

Lead sponsors only; a company and its subsidiary (for example Roche and Genentech) stay separate.

## I. Eligibility criteria across trials

### I1. How many comparable trials mention a criterion

```sql
SELECT cs.phase, count(*) AS n_trials,
       count(*) FILTER (WHERE cs.population->>'criteria_markdown' ~* '(FVC|forced vital capacity)') AS mention_fvc,
       count(*) FILTER (WHERE cs.population->>'criteria_markdown' ~* '(DLCO|diffusing capacity)') AS mention_dlco,
       count(*) FILTER (WHERE cs.population->>'criteria_markdown' ~* '(nintedanib|pirfenidone)') AS mention_antifibrotic
FROM clinical_studies cs
WHERE cs.id IN (SELECT clinical_study_id FROM clinical_study_disease
                WHERE icd11_code LIKE 'CB03.4%' AND role = 'primary')
  AND cs.phase IN ('phase_2', 'phase_3')
GROUP BY cs.phase ORDER BY cs.phase;
```

A mention is not a criterion: the word can appear in an inclusion rule, an exclusion rule or a definition. Read the lines (I2) before saying "N trials required …".

### I2. The criterion lines themselves

```sql
SELECT cs.primary_registry_id, cs.phase,
       left(substring(cs.population->>'criteria_markdown'
                      from '(?i)[^\n]*(?:FVC|forced vital capacity)[^\n]*'), 250) AS first_matching_line,
       cs.linkouts->>'registry_url' AS registry_url
FROM clinical_studies cs
WHERE cs.id IN (SELECT clinical_study_id FROM clinical_study_disease
                WHERE icd11_code LIKE 'CB03.4%' AND role = 'primary')
  AND cs.phase = 'phase_3'
  AND cs.population->>'criteria_markdown' ~* '(FVC|forced vital capacity)'
LIMIT 20;
```

Only the first matching line is returned; for the whole text of a trial use `get_clinical_study`. Criteria text can be abridged (EU CTR, "More exclusion criteria may apply"): an absent criterion is not proof that the trial did not use it.

## J. What is in the database

### J1. Coverage by registry

```sql
SELECT primary_registry, count(*) AS n_trials,
       count(*) FILTER (WHERE phase = 'NA') AS phase_na,
       count(*) FILTER (WHERE start_date IS NULL) AS no_start_date,
       count(*) FILTER (WHERE enrollment_actual IS NULL) AS no_enrolment
FROM clinical_studies
GROUP BY primary_registry ORDER BY n_trials DESC;
```

### J2. All trials by phase and status

```sql
SELECT COALESCE(phase, classification->>'phase') AS phase, count(*) AS n_trials,
       count(*) FILTER (WHERE status = 'completed') AS completed,
       count(*) FILTER (WHERE status = 'terminated') AS terminated,
       count(*) FILTER (WHERE status IN ('recruiting', 'not_yet_recruiting', 'active_not_recruiting')) AS ongoing
FROM clinical_studies
GROUP BY 1 ORDER BY 1;
```

Pair these numbers with the as-of dates: ClinicalTrials.gov 13 Jul 2026, the other registries 14 May 2026. Which registries lack endpoints, dates or a meaningful status: the coverage table in `data-model.md` (counting over JSONB across all trials is too slow for the time limit).

## K. One trial's structured record

```sql
SELECT cs.primary_registry, cs.primary_registry_id, cs.title,
       COALESCE(cs.phase, cs.classification->>'phase') AS phase,
       COALESCE(cs.status, cs.classification->>'status') AS status,
       cs.start_date, cs.end_date, cs.enrollment_actual, cs.sponsor_class,
       cs.organization->'lead_sponsors' AS lead_sponsors, cs.classification,
       cs.endpoints, cs.results->>'why_stopped' AS why_stopped,
       cs.linkouts->>'registry_url' AS registry_url
FROM clinical_studies cs
WHERE cs.primary_registry_id = 'NCT03156816';
```

Arms and components of the same trial: the `arms` query in `data-model.md`. A trial filed under another registry's id returns 0 rows here: search for it with `search_clinical_studies`.

## Performance

- Start from an indexed filter (`clinical_study_disease.icd11_code`, `clinical_study_drug.canonical_drug_id`, `clinical_studies.primary_registry_id`, `phase`, `status`, `sponsor_class`) and only then read JSONB.
- Whole-table aggregates over scalar columns (J1, J2) take about a second. Whole-table aggregates over JSONB (country, endpoint text, criteria text) exceed the 10-second limit: narrow to a condition, drug, phase or registry first.
- `cs.id IN (SELECT clinical_study_id …)` avoids the row multiplication of joins and needs no `DISTINCT`.
- Never `SELECT *` from `clinical_studies`: the JSONB columns make each row tens of kilobytes. Name the columns, and cut long text with `left(…, 300)`.
- Return aggregates, not raw rows: the result is capped at 1,000 rows.

## Common mistakes

| Mistake | Fix |
|---|---|
| `icd11_code = 'BA41'` misses the leaf codes | `LIKE 'BA41%'` |
| Counting joined rows | `count(DISTINCT cs.id)` or `cs.id IN (…)` |
| Filtering the condition by `display_name ILIKE …` | find the codes with A1, then filter by code |
| `phase = 'Phase 3'` | `phase = 'phase_3'` |
| Reading `enrollment_actual` of a recruiting trial as an actual size | it is the target; restrict to completed or terminated trials |
| A time trend including UMIN, ANZCTR or EU CTR | they have no dates; say the trend covers ClinicalTrials.gov and ISRCTN |
| Reporting `clinical_outcome` or `outcome_explanation` as results | machine-generated; registry results are not in these tables, so read the record at `registry_url` |
| `arm_role IN ('comparator', 'control')` | values are `experimental`, `active_comparator`, `placebo`, `no_intervention`, `soc`, `sham`, `other` |
| Reading another table (patents, predictions, opportunities, source registries) | not readable through this tool |
