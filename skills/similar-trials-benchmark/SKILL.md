---
name: similar-trials-benchmark
description: Benchmark a planned or existing clinical trial against comparable registered trials (ClinicalTrials.gov, EU CTR/CTIS, ISRCTN, ANZCTR, UMIN) and return a table with registry id and link, phase, status, enrolment, primary endpoint and time frame, key eligibility criteria and stop reason. Use when the user asks for similar, comparable, competing or precedent trials, a trial landscape for a drug, indication or design, or "what has been done before" for a study they are planning.
---

# Similar-trials benchmark

Find the registered trials most comparable to the user's study and present them as one evidence table that a clinical researcher can check row by row.

## Ground rules

1. Every trial, id, number and date in the answer comes from a mondoTWIN tool result. Never invent a trial or a value. Background knowledge may suggest search terms (a drug class, a trial acronym), but a trial appears in the answer only if a tool returned it. When a field is empty, write "not stated in the record".
2. Cite each trial by its registry id linked to its registry page: use `linkouts.registry_url` from `get_clinical_study`; for an NCT id without a detail call, `https://clinicaltrials.gov/study/<NCT id>`. For other registries without a detail call, give the id and registry name unlinked.
3. Say plainly when a search returned nothing, with the query and filters used. "Not in the database" is not "does not exist".
4. State the data as-of date: ClinicalTrials.gov as of 13 Jul 2026; EU CTR/CTIS, ISRCTN, ANZCTR and UMIN as of 14 May 2026. Later registrations and updates are missing.
5. No patient data. Work at protocol and registry level only. If the user pastes identifiable patient information, do not repeat it and ask them to remove it.
6. The output is a draft for qualified professionals to validate. Say so in one line at the end.
7. Answer in the user's language. Keep registry ids, trial titles and quoted registry text in their original language.
8. Use only `search_clinical_studies` and `get_clinical_study` here. If the mondoTWIN tools are not available, tell the user to connect the mondoTWIN connector, and stop. Do not answer from memory as though from the database.

## How the trial tools behave

- `search_text` keeps the records in which every word (or "quoted phrase") appears, each in any of title, hypothesis, diseases, drugs, sponsor or main centre. "colchicine stroke" keeps trials that mention both words; `"pulmonary fibrosis"` in quotes keeps the phrase whole. Start with one to three distinctive words (the drug, the disease) plus filters. A word the registry spells differently (an abbreviation, "MI" for myocardial infarction) removes good matches: if a query returns 0 or too few, drop a word and retry before concluding anything.
- Results come newest first; `total` is the match count; use `page_size` 10 to 20. Many matches mention the word in passing (background therapy, intolerance, a co-intervention): screen by title and diseases.
- Filters: `phases`, `statuses`, `sponsor_classes`, `icd11_chapters`, `rare_disease_only`. Records from ANZCTR, EU CTR and UMIN often have phase "NA" and no dates, so a phase filter silently drops them. Run at least one search without the phase filter.
- `icd11_chapters` keeps trials whose indication is in an ICD-11 chapter, which removes most passing mentions of a drug. Take the chapter from the first character of `icd11_code` on a relevant hit: `B…` is "11" (circulatory), `C…` is "12" (respiratory), `2…` is "02" (neoplasms), and so on. Do not guess it.
- Use `statuses: ["completed", "terminated"]` to surface finished trials, which are otherwise buried under recent registrations. A trial acronym finds a record only if the acronym is in that record's title, so a miss proves nothing; follow it with the drug word and the chapter filter.
- One trial can sit in the database under one registry id only (for example an EU CTR national record instead of the NCT record). Check `linkouts.other_registry_ids`. Never count a trial twice.
- Enrolment: `enrollment_actual` is an actual count only when the status is completed, terminated or withdrawn. For recruiting, active or not-yet-recruiting trials it repeats the target: show it as the target. An EU CTR record gives the target for one member state; label it "national".
- Machine-generated fields are not registry content: `hypothesis`, `clinical_outcome`, `positive_results`, `results.outcome_explanation`, `control_type`, `novelty_angle`, `regulatory_pathway`, `strategy_tags`, patent fields and `clinical_trial_prediction`. Do not present them as registry facts and never show prediction scores. Take the design from `arms` and `classification.masking`.
- Drug names are normalised and can be wrong for sponsor code names. When the normalised name looks inconsistent with the record (route, sponsor, arm label), show the registry's own intervention label.
- `get_clinical_study` returns 10 to 40 KB per trial (site lists, patents). Read at most 6 per answer. It gives `endpoints` (type, label, description; the time frame sits inside the description, there is no separate field), `population.criteria_markdown`, `arms`, `results.why_stopped` and `linkouts.registry_url`.
- `criteria_markdown` can be abridged: EU CTR text is often cut off, and CT.gov text may end with "More exclusion criteria may apply". Say so when you rely on it.

## Steps

1. **Profile the study.** Intervention (drug or class), indication, phase, population, comparator, primary endpoint. If the intervention or the indication is missing, ask one question; otherwise state your assumptions and go on.
2. **Search, within about 6 calls.** Keep a log of each query, its filters and `total`:
   - the drug word alone (page size 10), to read the `icd11_code` of relevant hits;
   - the drug word with the ICD-11 chapter and `statuses: ["completed", "terminated"]`, without a phase filter;
   - the drug word or indication phrase with the chapter and phase filters, for ongoing trials;
   - one or two known trial acronyms, if the field has landmark trials.
3. **Shortlist** from the search fields (title, diseases, drugs, phase, status, enrolment, dates). Drop trials where the word is only background therapy or a co-condition. Rank by closeness of population and intervention, then by status (completed and terminated first), then by size.
4. **Read details** for the 4 to 6 best candidates with `get_clinical_study`.
5. **Write the answer** in the format below.

## Output format

**Comparable trials for** *[one-line profile]*
Data as of: ClinicalTrials.gov 13 Jul 2026; EU CTR/CTIS, ISRCTN, ANZCTR, UMIN 14 May 2026.
Searched: *[n]* queries, *[X]* records matched in total, *[Y]* screened, *[Z]* shown, details read for *[K]*. Queries: *[list query + filters → total]*.

| Registry id | Trial | Phase | Status | Enrolment (actual / target) | Primary endpoint (time frame) | Key criteria | Stop reason |
|---|---|---|---|---|---|---|---|
| [*id*](*registry_url*) | *acronym or short title* | *phase* | *status* | *actual* / *target* | *endpoint* (*time point*) | *2 to 4 criteria* | *why_stopped or —* |

- Enrolment: "— / 300" when there is no actual count; add "(national, EU CTR)" where it applies.
- Primary endpoint: the registry label, shortened, with the time point from the description, or "time frame not stated".
- Key criteria: 2 to 4 criteria that define the population (age, disease definition, severity thresholds, key exclusions), quoted or closely paraphrased from `criteria_markdown`.
- Stop reason: `results.why_stopped` verbatim for terminated, withdrawn or suspended trials; "—" otherwise.

Then:

- **Also matched, not read in detail:** registry id, phase, status and target for the next most relevant trials (at most 10).
- **What the comparison shows:** 3 to 5 short observations (endpoint choices, follow-up lengths, sizes, why trials stopped), each citing registry ids.
- **Limits of this search:** records without phase, national EU records, abridged criteria, anything you looked for and did not find.
- One closing line: this is a draft from registry data for qualified professionals to validate.
