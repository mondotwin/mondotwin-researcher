# Data model - what `query_clinical_studies` can read

Tables, columns, JSONB shapes, enum values and join keys, with what each registry fills in. Counts describe the current data set and are given for scale.

## Contents

- Coverage by registry
- Tables at a glance and join keys
- `clinical_studies`: identity, scalar columns, JSONB columns
- `clinical_study_disease`, `clinical_study_drug`, `clinical_study_sponsor`
- Machine-generated fields

## Coverage by registry

373,779 trials. One row per trial: a trial found in several registries is kept once, under the id of one registry (`primary_registry`, `primary_registry_id`).

| `primary_registry` | Trials | Data as of | Phase | Status | Start and end dates | Endpoints | Enrolment |
|---|---:|---|---|---|---|---|---|
| `clinicaltrials.gov` | 251,850 | 13 Jul 2026 | stated; `NA` on 100,976 (devices, procedures, behavioural, observational) | registry status | yes | yes | one number: actual when completed, terminated or withdrawn, target otherwise |
| `umin` | 59,497 | 14 May 2026 | `NA` on 53,279 | always `completed` (not informative) | none | yes | target; actual on a minority |
| `isrctn` | 24,552 | 14 May 2026 | `NA` on 22,573 | mostly `active_not_recruiting`, even when finished | yes | none | target; actual on a minority |
| `anzctr` | 18,474 | 14 May 2026 | always `NA` | registry status | none | yes | target only |
| `eudract` (EU CTR) | 16,993 | 14 May 2026 | stated | registry status | none | yes (text often cut off) | target, often for one member state |
| `ctis` | 2,413 | 14 May 2026 | in `classification->>'phase'` only | in `classification->>'status'` only | none | yes | none |

Every trial has `population->>'criteria_markdown'` and `linkouts->>'registry_url'`. Trials registered or updated after the as-of dates are missing.

## Tables at a glance and join keys

| Table | Rows | Grain | Key |
|---|---:|---|---|
| `clinical_studies` | 373,779 | one trial | `id` (uuid); unique `(primary_registry, primary_registry_id)` |
| `clinical_study_disease` | ~444,000 | one (trial, coded condition) | `clinical_study_id` → `clinical_studies.id` |
| `clinical_study_drug` | ~593,000 | one (trial, arm, drug or biologic component) | `clinical_study_id` → `clinical_studies.id` |
| `clinical_study_sponsor` | ~361,000 | one (trial, lead sponsor) | `clinical_study_id` → `clinical_studies.id` |

`clinical_studies.id` is an internal random uuid: look it up from a registry id, never show it as a trial identifier. About 50,000 trials have no condition row, 167,000 no drug row (non-drug trials) and 14,000 no sponsor row; an inner join drops them.

## `clinical_studies`

### Identity

| Column | Meaning |
|---|---|
| `id` | internal uuid, the join key |
| `primary_registry` | `clinicaltrials.gov`, `umin`, `isrctn`, `anzctr`, `eudract`, `ctis` |
| `primary_registry_id` | the registry's id: `NCT…`, UMIN receipt number `R000…`, `ISRCTN…`, `ACTRN…`, EudraCT `2014-002739-32`, CTIS `2022-500007-52-00` (indexed) |
| `title` / `title_short` | official title / short or brief title |

### Scalar columns

| Column | Values |
|---|---|
| `phase` | `NA` (195,361), `phase_2` (75,722), `phase_3` (40,815), `phase_4` (30,076), `phase_1` (29,392); NULL on CTIS records |
| `status` | `completed`, `recruiting`, `unknown`, `active_not_recruiting`, `terminated`, `not_yet_recruiting`, `withdrawn`, `suspended`; NULL on CTIS records |
| `start_date`, `end_date` | dates; ClinicalTrials.gov and ISRCTN only; placeholders such as `1900-01-01` exist |
| `enrollment_actual`, `enrollment_target` | integers; see the coverage table |
| `sponsor_class` | `academic`, `unknown`, `industry`, `mixed`, `government` |
| `is_randomized` | boolean; NULL on CTIS records |
| `trial_purpose` | `treatment`, `diagnostic_validation`, `basic_science`, `device_performance` |
| `results_available` | boolean; true when the registry or a publication posted something (a raw "maybe" counts as true). It does not mean the trial succeeded |
| `clinical_outcome`, `positive_results` | machine-generated readout classification: do not report as results |
| `age_min_years`, `age_max_years`, `sex` | empty: use the `population` keys |

Indexes: `primary_registry_id`, `phase`, `status`, `sponsor_class`, `is_randomized`, `results_available`; GIN on `conditions`.

### JSONB columns

Read with `->` (JSON) and `->>` (text); cast numbers explicitly (`(population->>'age_min_years')::numeric`).

**`description`** - `title, title_short, phase, status, start_date, end_date, enrollment_actual, enrollment_target, description` (the registry summary) and the machine-generated `hypothesis` and `novelty_tags[]`. CTIS records have `title_long, brief_summary, detailed_description` instead.

**`classification`** - `{purpose, is_randomized, masking, control_type}`. `masking`: `open, NA, double, single, quadruple, triple`. `control_type`: `none, active, placebo, sham`. CTIS records add `phase` and `status` here.

**`endpoints`** - array of `{type, label, description}`; `type` is `primary` or `secondary`. The time frame is inside `label` or `description`; there is no separate field. ISRCTN records have an empty array.

```sql
SELECT e->>'label', e->>'description'
FROM clinical_studies cs CROSS JOIN LATERAL jsonb_array_elements(cs.endpoints) e
WHERE cs.primary_registry_id = 'NCT03156816' AND e->>'type' = 'primary'
```

**`results`** - `why_stopped` (the registry's stop reason; on about 22,000 terminated, 7,500 withdrawn and 800 suspended trials) and `results_available` (raw: `none, registry, journal, both, maybe, true`) come from the source. `clinical_outcome`, `outcome_explanation`, `confidence_score` and `positive_results` are machine-generated.

**`population`** - `criteria_markdown` (the registry's inclusion and exclusion text, sometimes abridged: EU CTR text is often cut off, ClinicalTrials.gov text may end with "More exclusion criteria may apply"), `age_min_years`, `age_max_years` (numbers; `0` and `99` mean not captured), `sex` (`all, female, male`), `index_conditions[]` (same objects as `conditions`), `line_of_therapy` (machine-generated: `n/a, 1L, 2L, 3L+, mixed, refractory, maintenance`), `line_of_therapy_rationale`, `notes`.

**`conditions`** (GIN-indexed) - array of `{label, icd11: {code, title, extensions[]}, orphanet: {code, label, rationale}, rationale, description}`; element 0 is the primary condition. `label` is the trial's own wording; the codes, rationales and descriptions are machine-assigned.

**`arms`** - array of `{arm_id, role, label, components[]}`. `role` is the raw arm role: `experimental, active_comparator, placebo, no_intervention, sham, SoC, other, …` (casing varies). Each component is `{name, type, route, is_novelty, regimen_dosage_description, drugbank_info: {name, drugbank_id}}`; `type` is `drug, procedure, device, biologic, behavioral, placebo, other, diagnostic, dietary, radiation, …`. `is_novelty` is machine-generated.

```sql
SELECT a->>'arm_id', a->>'role', a->>'label', c->>'name', c->>'type', c->>'route',
       c->>'regimen_dosage_description', c->'drugbank_info'->>'drugbank_id'
FROM clinical_studies cs
CROSS JOIN LATERAL jsonb_array_elements(cs.arms) a
CROSS JOIN LATERAL jsonb_array_elements(a->'components') c
WHERE cs.primary_registry_id = 'NCT03156816'
```

**`organization`** - `lead_sponsors[]` (names as text), `sponsor_class`, `main_center: {name, city, country, coordinates: {lat, lng}, contact[]}`, `other_centers[]` (same shape as `main_center`), `investigators[]` (`{name, affiliation, …}`), `funding_sources[]`, `funding_description`, `execution_description`, `study_contact`. There are no collaborator keys. `main_center.country` is not normalised and is empty on many records.

**`linkouts`** - `registry_url` (on every trial), `pubmed_ids[]`, `pubmed_results_ids[]` (PMIDs the registry marks as results papers), `other_links[]`, `other_registry_ids` (unreliable: always a junk `"id": "type"` entry), and on about 241,000 trials `openalex_publications[]` (`{title, journal, publication_year, doi, pmid, abstract, cited_by_count, …}`) and `epmc_publications[]` (`{title, journal, publication_year, doi, pmid, pmcid, abstract, …}`).

## `clinical_study_disease`

One row per coded condition of a trial (from `population.index_conditions`).

| Column | Meaning |
|---|---|
| `clinical_study_id` | → `clinical_studies.id` |
| `icd11_code` | ICD-11 code, at parent or leaf level: use `LIKE 'BA41%'` for a code and its children (indexed) |
| `role`, `position` | `primary` (position 0, the trial's main condition) or `secondary` |
| `display_name` | the trial's own wording of the condition; varies between trials with the same code |
| `orphanet_code`, `orphanet_label` | Orphanet code without prefix, empty string when none |
| `disease_key` | generated: `ORPHA:<code>` when an Orphanet code exists, else `ICD11:<code>` |

## `clinical_study_drug`

One row per drug or biologic component of an arm. Placebo, devices, procedures and behavioural components are not here (read `arms`).

| Column | Meaning |
|---|---|
| `clinical_study_id` | → `clinical_studies.id` |
| `canonical_drug_id` | DrugBank id `DB…` (about 496,000 rows) or `name::<lower-cased name>` (about 97,000 rows, no DrugBank match) (indexed) |
| `drug_display_name` | the normalised name; casing varies |
| `arm_id`, `component_position` | `arm_1`, … and the order in the arm |
| `arm_role` | lower-cased arm role: `experimental` (most), `active_comparator`, `placebo`, `no_intervention`, `soc`, `sham`, `other` |
| `component_type` | `drug` or `biologic` |
| `route` | `po` and `oral` (both occur), `iv`, `sc`, `im`, `topical`, `inhalation`, `intravitreal`, `intranasal`, `intrathecal`, `implant`, `ophthalmic`, `intra-articular`, `other`, or empty |
| `is_novel_for_disease` | machine-generated: the component is the one under test (not background therapy or comparator) |

`arm_role = 'experimental'` also holds supportive drugs given in the experimental arm (premedication, background therapy).

## `clinical_study_sponsor`

One row per lead sponsor of a trial; there are no collaborator or funder rows.

| Column | Meaning |
|---|---|
| `clinical_study_id` | → `clinical_studies.id` |
| `normalized_sponsor_name` | lower-cased, corporate suffixes removed: group by it (indexed) |
| `display_sponsor_name` | the name as registered |
| `role` | always `lead` |
| `sponsor_class` | the trial's sponsor class |
| `is_lead_pharma` | first lead sponsor of an industry or mixed trial |
| `position` | order among the lead sponsors |

## Machine-generated fields

Generated by models from registry text and kept for search and grouping, not registry facts: `description.hypothesis`, `description.novelty_tags`, `clinical_outcome`, `positive_results`, `results.clinical_outcome`, `results.outcome_explanation`, `results.confidence_score`, `results.positive_results`, `population.line_of_therapy`, the ICD-11 and Orphanet coding (`conditions`, `clinical_study_disease`), the drug normalisation (`drugbank_info`, `canonical_drug_id`), `arms[].components[].is_novelty` and `clinical_study_drug.is_novel_for_disease`. When an answer groups or filters by one of them, label it "machine-classified".
