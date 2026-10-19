---
name: eligibility-criteria
description: Draft inclusion and exclusion criteria for a clinical trial from the criteria of comparable registered trials, and propose concomitant-medication exclusions by screening the investigational drug for drug-drug interactions against background and commonly co-prescribed drugs. Use when the user asks to write, draft, compare, tighten or broaden eligibility criteria, inclusion/exclusion criteria, or prohibited and restricted medications for a study.
---

# Eligibility criteria

Draft inclusion and exclusion criteria grounded in what comparable trials used, and back the medication exclusions with an interaction screen.

## Ground rules

1. Every trial, id, threshold and interaction finding in the answer comes from a mondoTWIN tool result. Never invent a trial, a threshold or an interaction. Background pharmacology may guide which drug pairs to screen; a finding is reported only as the tool returned it.
2. Cite each trial by its registry id linked to its registry page: `linkouts.registry_url` from `get_clinical_study`, or `https://clinicaltrials.gov/study/<NCT id>` for an NCT id.
3. Say plainly when a search or a screen returned nothing, with the query or pair used.
4. State the data as-of date: ClinicalTrials.gov as of 13 Jul 2026; EU CTR/CTIS, ISRCTN, ANZCTR and UMIN as of 14 May 2026; interaction evidence as held in the mondoTWIN database (DrugBank, label-derived organ toxicity).
5. No patient data. Criteria are written for a population, never assessed for a named or identifiable patient. If the user pastes patient information, do not repeat it and ask them to remove it.
6. The output is a draft for qualified professionals (investigators, clinical pharmacologists, regulatory) to validate. Say so in one line at the end.
7. Answer in the user's language. Quote registry criteria verbatim in their original language.
8. Use only `search_clinical_studies`, `get_clinical_study`, `screen_ddi`, `search_ich_guidelines` and, for enrichment questions only, `get_disease_targets`. `query_clinical_studies` is also available for counts and comparisons across many trials (read-only SQL over the trial tables; see the query-clinical-studies skill). If the mondoTWIN tools are not available, tell the user to connect the mondoTWIN connector, and stop.

## How the tools behave

**Trials.** `search_text` keeps the records in which every word (or "quoted phrase") appears, each in any of title, hypothesis, diseases, drugs, sponsor or main centre: use the disease ("pulmonary fibrosis" in quotes keeps the phrase whole) or one to three distinctive words, plus `phases`, `statuses` and `icd11_chapters` filters (the chapter is the first character of a relevant hit's `icd11_code`: `B…` is "11", `C…` is "12"; it removes passing mentions). If a query returns 0, drop a word and retry before concluding anything. A phase filter drops records without a phase (common for ANZCTR, EU CTR, UMIN). `get_clinical_study` is 10 to 40 KB per trial: read at most 6. `population.criteria_markdown` holds the registry criteria. It can be abridged: EU CTR text is often cut off, and CT.gov text may end with "More exclusion criteria may apply". Say so, and do not read a missing criterion as "the trial did not use it". `arms[].components[].drugbank_info.drugbank_id` gives DrugBank ids for `screen_ddi`. Fields such as `hypothesis`, `clinical_outcome` and `control_type` are machine-generated: do not cite them.

**Interactions.** `screen_ddi` screens one pair per call and returns a `verdict`, a `confidence` and three layers: `known_pair` (curated interaction records), `mechanistic_cyp` (enzyme perpetrator and victim overlap) and `organ_toxicity` (shared organ burden).
- Pass DrugBank ids where you have them; names are matched approximately.
- Set `exposure` to `chronic` (default) for long-term dosing, `subacute` for courses of weeks to months, `acute` for short courses.
- `none_found` with `confidence: low` means no data, not "safe". For an investigational drug without DrugBank interaction data every pair comes back `none_found`, even a pair that comparable trials exclude because of interaction risk. In that case base the exclusion on comparable trials and the investigator's brochure, and say the screen could not assess it.
- An `organ_toxicity` layer that is `not_evaluated` is unassessed, not clear.

**Guidance.** Use `search_ich_guidelines` for two anchors: E8(R1) on how narrowly or broadly to define the study population, and M11 (Step 2 draft) on writing criteria (each one assessable as yes or no, with the tests and time windows that document it). Quote what you use.

## Steps

1. **Profile.** Indication, phase, intervention (DrugBank id if known), background standard of care, and intent: a broad, representative population or an enriched one. Ask one question if the indication or intervention is missing.
2. **Find 4 to 6 comparable trials** and read their `criteria_markdown`.
3. **Build a criteria matrix** for yourself: for each criterion type (age, diagnosis and how it is confirmed, severity thresholds, prior and background therapy, washout periods, organ function, comorbidities, contraception, consent), which trials used it and with which threshold.
4. **Draft the criteria.** Keep thresholds as precedent sets them. Where trials differ, give the range and pick one with a reason. Mark a criterion with no precedent "proposed, no precedent found". Write each criterion so that it can be answered yes or no.
5. **Screen medications.** Collect candidate co-medications from (a) the background standard of care for the indication, (b) drugs the comparable trials prohibit, and (c) a few representative perpetrators for the intervention's likely pathways (for example a strong CYP3A4 inhibitor and inducer, a strong CYP1A2 inhibitor). Screen 6 to 12 pairs with `screen_ddi`. Turn each `hazardous` or otherwise flagged pair into an exclusion or restriction, and state the evidence layer behind it.
6. **Enrichment (only if asked).** `get_disease_targets` gives genetics-anchored targets for the disease (Open Targets). A genetic association suggests a stratification or enrichment idea; it is not a validated selection biomarker. Say so.
7. **Write the answer** in the format below.

## Output format

**Draft eligibility criteria:** *[study in one line]* · Data as of: ClinicalTrials.gov 13 Jul 2026; other registries 14 May 2026.
Based on: *[k]* comparable trials (*linked ids*).

**Inclusion criteria**
1. *Criterion.* (sources: [NCT…](url), [NCT…](url))

**Exclusion criteria**
1. *Criterion.* (sources: … / proposed, no precedent found)

**Prohibited and restricted medications**

| Co-medication | Verdict (confidence) | Evidence | Proposed rule |
|---|---|---|---|
| *drug* | *verdict* (*confidence*) | *layer summary, e.g. curated record; strong CYP1A2 inhibition* | *exclude / allow with monitoring / not assessable: follow precedent* |

**Where comparable trials differ:** the criteria with different thresholds, the range, and the choice made.

**Open questions:** decisions that need the team (for example, whether to allow a background therapy).

One closing line: this is a draft for qualified professionals to validate; the interaction screen does not replace the investigator's brochure or the product labels.
