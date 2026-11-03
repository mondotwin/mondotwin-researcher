---
name: eligibility-criteria
description: Draft inclusion and exclusion criteria for a clinical trial from the criteria of comparable registered trials, and propose concomitant-medication exclusions from what those trials prohibit and from the investigational drug's likely interactions with background and commonly co-prescribed drugs. Use when the user asks to write, draft, compare, tighten or broaden eligibility criteria, inclusion/exclusion criteria, or prohibited and restricted medications for a study.
---

# Eligibility criteria

Draft inclusion and exclusion criteria grounded in what comparable trials used, and ground the medication exclusions in the prohibited medications of those trials.

## Ground rules

1. Every trial, id and threshold in the answer comes from a mondoTWIN tool result. Never invent a trial or a threshold. mondoTWIN serves no drug-interaction screen: an interaction you propose from pharmacology is your reasoning, labelled "pharmacology reasoning, not tool-screened", never presented as a database finding.
2. Cite each trial by its registry id linked to its registry page: `https://clinicaltrials.gov/study/<NCT id>` for an NCT id, `https://www.isrctn.com/<ISRCTN id>` for ISRCTN, `https://www.clinicaltrialsregister.eu/ctr-search/search?query=<EudraCT number>` for EU CTR; other registries' ids unlinked, with the registry's name.
3. Say plainly when a search returned nothing, with the query used.
4. State the data as-of date from the tools, never from memory: each registry's `through_on` in `coverage` (`SELECT source, through_on FROM coverage WHERE entity_type = 'trial record'`) and the release id.
5. No patient data. Criteria are written for a population, never assessed for a named or identifiable patient. If the user pastes patient information, do not repeat it and ask them to remove it.
6. The output is a draft for qualified professionals (investigators, clinical pharmacologists, regulatory) to validate. Say so in one line at the end.
7. Answer in the user's language. Quote registry criteria verbatim in their original language.
8. Use only `search_clinical_studies`, `get_clinical_study`, `twin_sql` and `twin_record` (for the criteria text), `twin_text`, `search_ich_guidelines` and, for enrichment questions only, `get_disease_targets`. `twin_sql` also counts and compares across many trials (read-only SQL over the twin's tables; see the query-database skill). If the mondoTWIN tools are not available, tell the user to connect the mondoTWIN connector, and stop.

## How the tools behave

**Trials.** `search_text` keeps the trials in which every word (or "quoted phrase") appears, each in any of the title, a condition, a drug or a sponsor as the registry wrote them: use the disease ("pulmonary fibrosis" in quotes keeps the phrase whole) or one to three distinctive words, plus `phases`, `statuses` and `icd11_chapters` filters (the chapter is the first character of a relevant hit's `icd11_code`: `B…` is "11", `C…` is "12"; it removes passing mentions). If a query returns 0, drop a word and retry before concluding anything. A phase filter drops trials that state no phase (`NA`). Read at most 6 trials with `get_clinical_study`. It returns no eligibility criteria: v1 has no criteria column.

**Criteria text.** The criteria are in the trial's `population` passage (the title, description and eligibility the twin's population model read):
`twin_sql(query="SELECT p.passage_id, left(p.text, 2000) AS text FROM document_link l JOIN passage p ON p.document_id = l.document_id WHERE l.entity_type = 'trial' AND l.entity_id = '<id>' AND p.section IN ('population', 'rendered/population')")`; a passage longer than 2,000 characters is read whole with `SELECT text FROM passage WHERE passage_id = '<passage_id>'`. When the trial has no such passage, open the raw record: `twin_record(key=<a record_key from get_clinical_study's records>)` (ClinicalTrials.gov: `protocolSection.eligibilityModule.eligibilityCriteria`). Across trials, `twin_text(query="\"forced vital capacity\" \"pulmonary fibrosis\"")` finds the passages that state a criterion. The text can be abridged: EU CTR text is often cut off, and CT.gov text may end with "More exclusion criteria may apply". Say so, and do not read a missing criterion as "the trial did not use it". Age limits, sex and line of therapy are not columns either: read them in the text. `outcome`, `clinical_outcome` and `stop_class` are machine-made: do not cite them as registry facts.

**Interactions.** No interaction screen is served (the DrugBank interaction tables are not part of the twin). Medication exclusions rest on two kinds of evidence, kept apart in the answer: **precedent** (the prohibited and restricted medications comparable trials state in their criteria text, cited by registry id) and **pharmacology reasoning** (the enzyme and transporter pathways, narrow-therapeutic-window partners and shared organ toxicity you know for the drug class), labelled "not tool-screened" and referred to the investigator's brochure and the product labels for confirmation. For an investigational drug with no published interaction data, precedent is the only evidence: say so.

**Guidance.** Use `search_ich_guidelines` for two anchors: E8(R1) on how narrowly or broadly to define the study population, and M11 (Step 2 draft) on writing criteria (each one assessable as yes or no, with the tests and time windows that document it). Quote what you use.

## Steps

1. **Profile.** Indication, phase, intervention, background standard of care, and intent: a broad, representative population or an enriched one. Ask one question if the indication or intervention is missing.
2. **Find 4 to 6 comparable trials** and read their criteria text (above).
3. **Build a criteria matrix** for yourself: for each criterion type (age, diagnosis and how it is confirmed, severity thresholds, prior and background therapy, washout periods, organ function, comorbidities, contraception, consent), which trials used it and with which threshold.
4. **Draft the criteria.** Keep thresholds as precedent sets them. Where trials differ, give the range and pick one with a reason. Mark a criterion with no precedent "proposed, no precedent found". Write each criterion so that it can be answered yes or no.
5. **Medications.** Collect candidate co-medications from (a) the drugs the comparable trials prohibit or restrict (their criteria text), (b) the background standard of care for the indication, and (c) a few representative perpetrators for the intervention's likely pathways (for example a strong CYP3A4 inhibitor and inducer, a strong CYP1A2 inhibitor). Turn each into an exclusion, a restriction or an allowance, and state the evidence behind it: precedent with the trial ids, or pharmacology reasoning, not tool-screened.
6. **Enrichment (only if asked).** `get_disease_targets` gives genetics-anchored targets for the disease (Open Targets). A genetic association suggests a stratification or enrichment idea; it is not a validated selection biomarker. Say so.
7. **Write the answer** in the format below.

## Output format

**Draft eligibility criteria:** *[study in one line]* · Data as of: *[each registry's through_on, from coverage]*.
Based on: *[k]* comparable trials (*linked ids*) · Release *[id]*.

**Inclusion criteria**
1. *Criterion.* (sources: [NCT…](url), [NCT…](url))

**Exclusion criteria**
1. *Criterion.* (sources: … / proposed, no precedent found)

**Prohibited and restricted medications**

| Co-medication | Evidence | Basis | Proposed rule |
|---|---|---|---|
| *drug* | *e.g. prohibited in [NCT…](url), [NCT…](url); or strong CYP1A2 inhibition of a CYP1A2 substrate* | *precedent / pharmacology reasoning, not tool-screened* | *exclude / restrict / allow with monitoring* |

**Where comparable trials differ:** the criteria with different thresholds, the range, and the choice made.

**Open questions:** decisions that need the team (for example, whether to allow a background therapy).

One closing line: this is a draft for qualified professionals to validate; the medication rules do not replace the investigator's brochure or the product labels.
