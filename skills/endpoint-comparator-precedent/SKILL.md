---
name: endpoint-comparator-precedent
description: Report which primary endpoints the registered trials of an indication used and how often each succeeded or failed, by phase, and which control arms and active comparators those trials used and how they fared, from mondoTWIN's twin. Use when the user asks which endpoints work or have succeeded for a disease, for endpoint precedent or success rates by endpoint, what primary endpoint a phase 3 should use, which comparator or control arm trials in the indication used, the standard-of-care comparator, placebo versus active comparator, or what trials in an indication have measured. Descriptive precedent, not a prediction.
---

# Endpoint and comparator precedent

Two design-time questions have an answer in the registered trials of an indication: which primary endpoints have held up, and which control arms analogous trials used. Answer both from the twin's tables, by phase, as counts a researcher can check row by row. The endpoint half needs a clustering step, because registries hold endpoint labels as free text; the comparator half is structured (`trial_drug.role`).

## Ground rules

1. Every trial, id, count and endpoint label in the answer comes from a mondoTWIN tool result. Never invent a trial or a number. The clustering of endpoint labels (step 2b) is your reading of the labels the tools returned: say so, and give the raw count behind each cluster.
2. Cite the release id (`SELECT release FROM release_info`) and, for any list of trials (the comparators, the trials of a small cell), their registry ids: `trial_record.record_key` where `is_primary`. Link an NCT id as `https://clinicaltrials.gov/study/<NCT id>`, an ISRCTN id as `https://www.isrctn.com/<ISRCTN id>`, a EudraCT number as `https://www.clinicaltrialsregister.eu/ctr-search/search?query=<EudraCT number>`; other registries' ids unlinked, with the registry's name.
3. Say plainly when a query returned nothing, with the query used.
4. State the data as-of date from the tools, never from memory: each registry's `through_on` in `coverage` (`SELECT source, through_on FROM coverage WHERE entity_type = 'trial record'`).
5. No patient data. If the user pastes identifiable patient information, do not repeat it and ask them to remove it.
6. The output is descriptive precedent for qualified professionals (clinicians, statisticians, regulatory) to weigh; it is not a prediction and no success-probability model is applied. Say so in one line at the end.
7. Answer in the user's language. Quote endpoint labels as the registries wrote them.
8. Use only `twin_find`, `twin_catalog`, `twin_sql`, `twin_text` and, to read one trial, `get_clinical_study`. `twin_sql` runs one read-only `SELECT` or `WITH` over the twin's tables (15 s, at most 500 rows; see the query-database skill); page with `OFFSET` when a reply says `more`. `twin_catalog(table="trial_endpoint")` shows a table's columns. If the mondoTWIN tools are not available, tell the user to connect the mondoTWIN connector, and stop.

## Step 1: resolve the indication to ICD-11 codes

The twin keys conditions on ICD-11 (`icd11:<code>`) and Orphanet (`orpha:<code>`). Resolve the name first, and show the candidates: a name often spans several codes, and not all are the same indication. The trials' own wording shows which codes they were coded to:

```
twin_sql(query="
  SELECT c.icd11_id, max(d.name) AS code_title, max(c.raw_text) AS example_text,
         count(DISTINCT c.trial_id) AS trials
  FROM trial_condition c LEFT JOIN disease d ON d.disease_id = c.icd11_id
  WHERE c.raw_text ILIKE '%amyotrophic lateral%'
  GROUP BY c.icd11_id ORDER BY trials DESC LIMIT 10")
```

Judgement, not blind aggregation:

- The dominant code and its sub-codes under the same parent are usually one indication (`8B60`, `8B60.0` and `8B60.5` are all ALS): take them together with a prefix match, `c.icd11_id LIKE 'icd11:8B60%'`.
- A different clinical entity that only mentions the name is not the same indication (`CB41`, respiratory failure secondary to ALS): leave it out.
- Some condition codes are wrong at the code level: trust the parent code over a leaf, and read `raw_text` when a count looks off.
- If it stays ambiguous, show the user the candidates and ask which to include before running the precedent queries.

State in the answer which codes you took.

## Step 2: endpoint precedent

### 2a. Pull the primary-endpoint labels

`trial_endpoint` holds one row per registered endpoint: `kind` is `primary` or `secondary`, `measure` is the registry's free text, `timepoint` its time frame. Trials whose primary record is ISRCTN have no endpoint rows. The outcome of a trial is `trial_outcome.outcome`. One row per (trial, primary endpoint), with phase and outcome:

```
twin_sql(query="
  WITH dis AS (
    SELECT DISTINCT t.trial_id, t.phase, o.outcome
    FROM trial t
    JOIN trial_condition c ON c.trial_id = t.trial_id
    JOIN trial_outcome o ON o.trial_id = t.trial_id
    WHERE c.icd11_id LIKE 'icd11:8B60%'
      AND o.outcome IN ('positive_significant','positive_trend','failed','harmful_or_inferior')
  )
  SELECT dis.phase, dis.outcome, e.measure AS endpoint_label, e.timepoint
  FROM dis JOIN trial_endpoint e ON e.trial_id = dis.trial_id
  WHERE e.kind = 'primary' AND coalesce(e.measure,'') <> ''",
  max_rows=500)
```

Count first how many trials of the indication have an outcome row at all (the same query without the `o.outcome IN` filter, and the trials with no `trial_outcome` row), so that the caveats can say how many are invisible here.

### 2b. Cluster the labels: this is what the skill adds

A plain count of labels is useless: one instrument appears under dozens of strings ("Change from baseline in ALSFRS-R", "ALSFRS-R total score", "revised ALS functional rating scale"). Cluster the labels yourself, by meaning, in two passes:

1. **Set aside non-efficacy endpoints.** Bucket each label as `safety` ("adverse event", "safety", "tolerability", "laboratory", "vital sign"), `pharmacokinetic` ("pharmacokinetic", "concentration", "Cmax", "AUC") or `efficacy`. Report safety and PK as lines of their own; they must not drown the efficacy signal (they often outnumber it).
2. **Cluster the efficacy labels by concept.** Name each cluster by the recognised instrument or outcome, not by a raw string, and expand acronyms (NSAA, North Star Ambulatory Assessment; 6MWD, 6-minute walk distance).

### 2c. Success rate per cluster, by phase

Aggregate the clusters against the outcome, split by phase: the phase mix confounds a pooled rate (safety endpoints sit early). `positive_significant` and `positive_trend` count as positive, `failed` and `harmful_or_inferior` as negative; `descriptive_met`, `stopped_operational`, `inconclusive` and `not_reported` are neither and stay out of the rate. Keep `NA` (no phase stated) as a row of its own. Present cluster | phase | trials (n) | positive (n, %).

## Step 3: comparator precedent

`trial_drug.role` is `investigational` (the test article), `comparator` (a drug of an active-comparator, placebo, sham, standard-of-care or no-intervention arm that no experimental arm also gives) or `background`. Placebo, sham and procedures give no row: a placebo arm is read from the arms text, not from `trial_drug`.

### 3a. Control-arm composition and how each fared

```
twin_sql(query="
  WITH dis AS (
    SELECT DISTINCT t.trial_id, o.outcome,
           EXISTS (SELECT 1 FROM trial_drug x WHERE x.trial_id = t.trial_id
                   AND x.role = 'comparator') AS active_comparator
    FROM trial t
    JOIN trial_condition c ON c.trial_id = t.trial_id
    JOIN trial_outcome o ON o.trial_id = t.trial_id
    WHERE c.icd11_id LIKE 'icd11:8B60%'
      AND o.outcome IN ('positive_significant','positive_trend','failed','harmful_or_inferior')
  )
  SELECT active_comparator, count(*) AS trials,
         count(*) FILTER (WHERE outcome IN ('positive_significant','positive_trend')) AS positive
  FROM dis GROUP BY 1 ORDER BY 2 DESC")
```

For placebo-controlled trials, read the arms text: `twin_text(query="placebo \"amyotrophic lateral sclerosis\"")`, or in SQL the trials' `arms` passages (`passage p JOIN document_link l ON l.document_id = p.document_id AND l.entity_type = 'trial' WHERE p.section IN ('arms', 'rendered/arms') AND p.text ILIKE '%placebo%'`), scoped to the trials of 2a and kept to a few hundred trials, or it runs past the 15 s limit.

### 3b. Which active comparators, and how often

```
twin_sql(query="
  WITH dis AS (
    SELECT DISTINCT c.trial_id FROM trial_condition c WHERE c.icd11_id LIKE 'icd11:8B60%'
  )
  SELECT coalesce(g.name, x.raw_name) AS comparator,
         count(DISTINCT dis.trial_id) AS used_as_comparator,
         string_agg(DISTINCT dis.trial_id, ', ') AS trial_ids
  FROM dis JOIN trial_drug x ON x.trial_id = dis.trial_id
  LEFT JOIN drug g ON g.drug_id = x.drug_id
  WHERE x.role = 'comparator'
  GROUP BY 1 ORDER BY 2 DESC LIMIT 10")
```

The top active comparator is usually the indication's standard of care (for ALS, riluzole). Name it as the de-facto benchmark. Collapse case and spelling variants of one drug when you read the list (`raw_name` is the registry's spelling where no `drug` row matched). For the registry ids of those trials: `SELECT trial_id, record_key FROM trial_record WHERE is_primary AND trial_id IN (...)`.

## Output format

**Endpoint and comparator precedent:** *[indication, phase of interest]* · ICD-11 codes taken: *[codes]* · Data as of: *[each registry's through_on, from coverage]* · Release *[id]*.

**Primary endpoints and how they fared**

| Endpoint (cluster) | Phase | Trials | Positive |
|---|---|---|---|
| *instrument* | *phase* | *n* | *n (%)* |
| Safety / tolerability (non-efficacy) | … | … | … |

**Control arms**

| Control arm | Trials | Positive |
|---|---|---|
| no active drug comparator (placebo, sham, no intervention or not stated) | *n* | *n* |
| active drug comparator | *n* | *n* |

Active comparators used: *[drug (n, registry ids)]*. De-facto standard-of-care benchmark: *[drug]*.

**Reading for a new protocol:** one or two sentences, with the n behind each claim.

**Caveats**, always, in full:
- Descriptive, not statistical: a cell can hold fewer than ten trials; the n is given beside every rate.
- `trial_outcome.outcome` is a model's reading of posted results, papers and the registry text (its `confidence` is in the row); `positive_trend` was counted with `positive_significant`.
- Reporting bias: only trials with a positive or negative outcome are counted; *[n]* trials of the indication have no outcome row or a `not_reported`, `inconclusive` or `descriptive_met` one and are invisible here.
- Endpoint clusters are not mutually exclusive: a trial with two primary endpoints counts in two.
- The clustering of labels is this assistant's reading; the raw labels are available on request.

One closing line: this is descriptive precedent for qualified professionals to weigh, not a prediction.
