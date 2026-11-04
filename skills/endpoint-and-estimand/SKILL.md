---
name: endpoint-and-estimand
description: Summarise which primary and key secondary endpoints comparable registered trials used and at which time points, and draft the estimand for a trial objective following ICH E9(R1) (treatment, population, variable, intercurrent events with their strategies, population-level summary). Use when the user is choosing or justifying a primary or secondary endpoint, asks what similar trials measured and when, or asks to define, check or write an estimand or an intercurrent-event strategy.
---

# Endpoints and estimand

Show what comparable trials measured and when, then draft the estimand that the chosen endpoint serves, in the structure of ICH E9(R1).

## Ground rules

1. Every trial, id, endpoint, time point and guideline passage in the answer comes from a mondoTWIN tool result. Never invent a trial, a number or a quotation. Statistical reasoning is yours; label it as reasoning, not as evidence.
2. Cite each trial by its registry id linked to its registry page: `https://clinicaltrials.gov/study/<NCT id>` for an NCT id, `https://www.isrctn.com/<ISRCTN id>` for ISRCTN, `https://www.clinicaltrialsregister.eu/ctr-search/search?query=<EudraCT number>` for EU CTR; other registries' ids unlinked, with the registry's name. Cite each guideline passage by guideline, section (when visible) and a short verbatim quote.
3. Say plainly when a search returned nothing, with the query used.
4. State the data as-of date from the tools, never from memory: each registry's `through_on` in `coverage` (`SELECT source, through_on FROM coverage WHERE entity_type = 'trial record'`, through `twin_sql`) and the release id; ICH guidance as held in the mondoTWIN database.
5. No patient data. If the user pastes identifiable patient information, do not repeat it and ask them to remove it.
6. The output is a draft for qualified professionals (clinicians and statisticians) to validate. Say so in one line at the end.
7. Answer in the user's language. Quote registry text and guidelines verbatim in their original language.
8. Use only `search_clinical_studies`, `get_clinical_study`, `search_ich_guidelines`, `get_ich_guideline_full` and, for biomarker endpoints only, `get_disease_targets`. `twin_sql` is also available for counts and comparisons across many trials (read-only SQL over the twin's tables, for example how often an endpoint is primary; see the query-database skill), and for the criteria and arms text of a trial. If the mondoTWIN tools are not available, or a call answers that authorization is required, the connector is not connected yet: ask the user to connect **mondoTWIN** (the assistant opens the sign-in window; they allow access, then sign in or register with their mondoTWIN account, and a new account waits for approval), and continue once the tools answer. Never answer from memory in the meantime.

## How the tools behave

**Trials.** `search_text` keeps the trials in which every word (or "quoted phrase") appears, each in any of the title, a condition, a drug or a sponsor as the registry wrote them: use one to three distinctive words (the drug, the disease), plus `phases`, `statuses` and `icd11_chapters` filters (the chapter is the first character of a relevant hit's `icd11_code`: `B…` is "11", `C…` is "12"; it removes passing mentions). If a query returns 0, drop a word and retry before concluding anything. A phase filter drops trials that state no phase (`NA`): run one search without it. Read at most 6 trials with `get_clinical_study`. Its `endpoints[]` have `kind` (primary or secondary), `position`, `measure`, `description` and `timepoint` (the time frame as the registry states it; NULL when it states none, then look in the measure or description). When no time point is given, write "time frame not stated". EU CTR endpoint text is often cut off; trials whose primary record is ISRCTN register no endpoints. `why_stopped` explains terminated trials. `outcome` and `clinical_outcome` are a model's reading of the results, not registry results: do not report them as outcomes. For intercurrent events, the criteria and arms text of a trial (rescue medication, prohibited therapy, discontinuation rules) is in its `population` and `arms` passages: `twin_sql(query="SELECT p.section, left(p.text, 2000) AS text FROM document_link l JOIN passage p ON p.document_id = l.document_id WHERE l.entity_type = 'trial' AND l.entity_id = '<id>' AND p.section IN ('population', 'arms', 'rendered/population', 'rendered/arms')")`, or the raw record with `twin_record`.

**ICH.** `search_ich_guidelines` returns passages with a bare `guide_type`. `E9` holds E9(R1), with the estimands addendum in the A sections: A.3.1 intercurrent events, A.3.2 strategies, A.3.3 estimand attributes, A.3.4 constructing an estimand, A.5.2 sensitivity analysis, A.5.3 supplementary analysis. `M11` holds the Step 2 draft protocol template, which has an estimand table per objective; say "M11 (Step 2 draft)" when citing it. Read section numbers from headings inside the passage; if none is visible, search again with the heading words or write "section not identified". Good queries use E9(R1) vocabulary: "intercurrent event", "treatment policy strategy", "hypothetical strategy", "composite variable strategy", "while on treatment strategy", "principal stratum", "population-level summary", "sensitivity analysis". `get_ich_guideline_full` returns the whole guideline and can exceed the app's tool-result limit: use it at most once, only if search cannot find a section you need.

## Steps

1. **Profile.** Indication, phase, intervention, comparator, and the objective the endpoint must answer. If the objective is unclear, state the one you assume.
2. **Endpoint landscape.** Search for comparable trials (2 or 3 searches), read 4 to 6 records, and tabulate primary and key secondary endpoints with their time points. Count the patterns ("4 of 6 used … at week 52"). For how often each endpoint succeeded across the whole indication, by phase, and for the comparators its trials used, run the endpoint-comparator-precedent skill.
3. **Choose or check the variable.** Compare the user's endpoint with the landscape: measure, time point, responder definition or continuous change, composite components. Note where the user departs from precedent and what that implies.
4. **Retrieve E9(R1).** Search for the attributes (A.3.3) and for the strategies that fit the likely intercurrent events. Keep the passages you will quote.
5. **Draft the estimand** with all five E9(R1) attributes:
   - **Treatment:** the treatment condition of interest and the comparator, including background therapy and permitted rescue.
   - **Population:** the patients targeted by the clinical question.
   - **Variable:** the endpoint and time point, from step 3.
   - **Intercurrent events and strategies:** list the events likely in this setting (for example treatment discontinuation, rescue or prohibited medication, change of background therapy, death, transplant or surgery). Take the candidates from the indication and from the criteria and arms of comparable trials. Give each a strategy (treatment policy, hypothetical, composite variable, while on treatment, principal stratum) with a reason.
   - **Population-level summary:** for example a difference in means, a hazard ratio or a difference in proportions, consistent with the variable.
6. **Consequences.** What each strategy requires in data collection (for example, collecting outcomes after discontinuation under treatment policy) and which sensitivity analyses follow. Cite E9(R1) where retrieved.
7. **Write the answer** in the format below.

## Output format

**Endpoints and estimand:** *[study and objective in one line]* · Data as of: *[each registry's through_on, from coverage]* · Release *[id]*.

**What comparable trials measured**

| Registry id | Status | Primary endpoint | Time point | Key secondary endpoints | Stop reason |
|---|---|---|---|---|---|
| [*id*](*registry_url*) | *status* | *endpoint* | *time point or "not stated"* | *…* | *why_stopped or —* |

Patterns: *[two or three sentences with counts and ids]*.

**Proposed estimand** (ICH E9(R1) A.3.3)

| Attribute | Proposal | Rationale and evidence |
|---|---|---|
| Treatment | *…* | *…* |
| Population | *…* | *…* |
| Variable | *…* | *ids of trials using it* |
| Intercurrent events | *event → strategy*, one per line | *reason; E9(R1) A.3.2: "quote"* |
| Population-level summary | *…* | *…* |

**What this implies:** data to collect after intercurrent events, sensitivity analyses, alignment with the sample size and the analysis.

**Open questions:** choices that need the clinical team or the statistician.

One closing line: this is a draft for qualified professionals to validate.
