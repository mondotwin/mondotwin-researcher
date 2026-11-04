---
name: similar-trials-benchmark
description: Benchmark a planned or existing clinical trial against comparable registered trials (ClinicalTrials.gov, EU CTR/CTIS, ISRCTN, ANZCTR, UMIN) and return a table with registry id and link, phase, status, enrolment, primary endpoint and time frame, key eligibility criteria and stop reason. Use when the user asks for similar, comparable, competing or precedent trials, a trial landscape for a drug, indication or design, or "what has been done before" for a study they are planning.
---

# Similar-trials benchmark

Find the registered trials most comparable to the user's study and present them as one evidence table that a clinical researcher can check row by row.

## Ground rules

1. Every trial, id, number and date in the answer comes from a mondoTWIN tool result. Never invent a trial or a value. Background knowledge may suggest search terms (a drug class, a trial acronym), but a trial appears in the answer only if a tool returned it. When a field is empty, write "not stated in the record".
2. Cite each trial by its registry id linked to its registry page: `https://clinicaltrials.gov/study/<NCT id>` for an NCT id, `https://www.isrctn.com/<ISRCTN id>` for ISRCTN, `https://www.clinicaltrialsregister.eu/ctr-search/search?query=<EudraCT number>` for EU CTR. For other registries give the id and registry name unlinked.
3. Say plainly when a search returned nothing, with the query and filters used. "Not in the database" is not "does not exist".
4. State the data as-of date from the tools, never from memory: each registry's `through_on` in `coverage` (one `twin_sql` call: `SELECT source, through_on FROM coverage WHERE entity_type = 'trial record'`) and the `release` that `get_clinical_study` returns. Later registrations and updates are missing.
5. No patient data. Work at protocol and registry level only. If the user pastes identifiable patient information, do not repeat it and ask them to remove it.
6. The output is a draft for qualified professionals to validate. Say so in one line at the end.
7. Answer in the user's language. Keep registry ids, trial titles and quoted registry text in their original language.
8. Use `search_clinical_studies` and `get_clinical_study` to find and read trials, and `twin_sql` or `twin_record` for the eligibility text they do not return. `twin_sql` also counts and compares across many trials (read-only SQL over the twin's tables; see the query-database skill). If the mondoTWIN tools are not available, or a call answers that authorization is required, the connector is not connected yet: ask the user to connect **mondoTWIN** (the assistant opens the sign-in window; they allow access, then sign in or register with their mondoTWIN account, and a new account waits for approval), and continue once the tools answer. Never answer from memory in the meantime. Do not answer from memory as though from the database.

## How the trial tools behave

- `search_text` keeps the trials in which every word (or "quoted phrase") appears, each in any of the title, the brief title, a condition, a drug or a sponsor as the registry wrote them, or matches a registry id. "colchicine stroke" keeps trials that mention both words; `"pulmonary fibrosis"` in quotes keeps the phrase whole. Start with one to three distinctive words (the drug, the disease) plus filters. A word the registry spells differently (an abbreviation, "MI" for myocardial infarction) removes good matches: if a query returns 0 or too few, drop a word and retry before concluding anything.
- Results come newest first (by start date); `total` is the match count; use `page_size` 10 to 20. Many matches name the word in passing (background therapy, a co-condition): screen by title, `diseases` and `drugs`.
- Filters: `phases` (a phase 2 filter also keeps phase 1/2 and phase 2/3 trials), `statuses`, `icd11_chapters`, `rare_disease_only`. There is no sponsor-class filter. Trials that state no phase have phase `NA`, so a phase filter silently drops them. Run at least one search without the phase filter.
- `icd11_chapters` keeps trials with a condition in an ICD-11 chapter, which removes most passing mentions of a drug. Take the chapter from the first character of `icd11_code` on a relevant hit: `B…` is "11" (circulatory), `C…` is "12" (respiratory), `2…` is "02" (neoplasms), and so on. Do not guess it.
- Use `statuses: ["completed", "terminated"]` to surface finished trials, which are otherwise buried under recent registrations. A trial acronym finds a record only if the acronym is in its title, so a miss proves nothing; follow it with the drug word and the chapter filter.
- A trial registered in several registries is one trial: `primary_registry_id` is its NCT number when it has one, and `get_clinical_study` lists every registry id in `registry_ids`. Never count a trial twice.
- Enrolment: `enrollment_actual` is the stated final count, `enrollment_planned` the target; a trial carries one or the other. Show the target as the target. An EU CTR target can be for one member state; label it "national" when the record says so.
- Machine-made fields are not registry content: the condition codes (`icd11_code`, `orphanet_codes`), the linked drug names and arm roles, `outcome` and `clinical_outcome` (a model's reading of the results), `stop_class` (a keyword rule) and, for staff accounts, `clinical_trial_prediction`. Do not present them as registry facts and never show prediction scores. `hypothesis`, the main centre and its coordinates are always empty: v1 does not hold them.
- Drug names are linked to the drug registry and can be wrong for sponsor code names. When the linked name looks inconsistent with the record (route, sponsor, arm label), show the registry's own name: `arms[].raw_name` in `get_clinical_study`.
- `get_clinical_study` takes the `id` of a hit, or a registry id. Read at most 6 per answer. It gives `endpoints` (`kind` primary or secondary, `measure`, `description`, `timepoint`: the time frame as the registry states it), `arms` (arm label, the component as written, role, dose, route), `conditions`, `orgs`, `registry_ids`, `records` (each registry record's `record_key`), `why_stopped`, `stop_class`, `events` and `papers`. It gives **no eligibility criteria**: v1 holds no criteria column.
- **Eligibility.** The criteria are in the trial's `population` passage, as the twin's population model read the record (title, description, eligibility):
  `twin_sql(query="SELECT p.passage_id, left(p.text, 2000) AS text FROM document_link l JOIN passage p ON p.document_id = l.document_id WHERE l.entity_type = 'trial' AND l.entity_id = '<id>' AND p.section IN ('population', 'rendered/population')")`.
  When that returns nothing, open the raw registry record with `twin_record(key=<a record_key from records>)` (ClinicalTrials.gov: `protocolSection.eligibilityModule.eligibilityCriteria`). The text can be abridged: EU CTR text is often cut off, and CT.gov text may end with "More exclusion criteria may apply". Say so when you rely on it.

## Steps

1. **Profile the study.** Intervention (drug or class), indication, phase, population, comparator, primary endpoint. If the intervention or the indication is missing, ask one question; otherwise state your assumptions and go on.
2. **Search, within about 6 calls.** Keep a log of each query, its filters and `total`:
   - the drug word alone (page size 10), to read the `icd11_code` of relevant hits;
   - the drug word with the ICD-11 chapter and `statuses: ["completed", "terminated"]`, without a phase filter;
   - the drug word or indication phrase with the chapter and phase filters, for ongoing trials;
   - one or two known trial acronyms, if the field has landmark trials.
3. **Shortlist** from the search fields (title, diseases, drugs, phase, status, enrolment, dates). Drop trials where the word is only background therapy or a co-condition. Rank by closeness of population and intervention, then by status (completed and terminated first), then by size.
4. **Read details** for the 4 to 6 best candidates with `get_clinical_study`, and their eligibility text (above).
5. **Write the answer** in the format below.

## Output format

**Comparable trials for** *[one-line profile]*
Data as of: *[each registry's through_on, from coverage]* · Release *[id]*.
Searched: *[n]* queries, *[X]* records matched in total, *[Y]* screened, *[Z]* shown, details read for *[K]*. Queries: *[list query + filters → total]*.

| Registry id | Trial | Phase | Status | Enrolment (actual / target) | Primary endpoint (time frame) | Key criteria | Stop reason |
|---|---|---|---|---|---|---|---|
| [*id*](*registry_url*) | *acronym or short title* | *phase* | *status* | *actual* / *target* | *endpoint* (*time point*) | *2 to 4 criteria* | *why_stopped or —* |

- Enrolment: "— / 300" when there is no actual count; add "(national, EU CTR)" where it applies.
- Primary endpoint: the registry's `measure`, shortened, with its `timepoint` (or the time point the description states), or "time frame not stated".
- Key criteria: 2 to 4 criteria that define the population (age, disease definition, severity thresholds, key exclusions), quoted or closely paraphrased from the eligibility text; "criteria not held" when neither the passage nor the raw record has them.
- Stop reason: `why_stopped` verbatim for terminated, withdrawn or suspended trials; "—" otherwise.

Then:

- **Also matched, not read in detail:** registry id, phase, status and target for the next most relevant trials (at most 10).
- **What the comparison shows:** 3 to 5 short observations (endpoint choices, follow-up lengths, sizes, why trials stopped), each citing registry ids.
- **Limits of this search:** records without phase, national EU records, abridged or missing criteria, anything you looked for and did not find.
- One closing line: this is a draft from registry data for qualified professionals to validate.
