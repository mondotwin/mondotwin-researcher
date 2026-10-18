---
name: endpoint-and-estimand
description: Summarise which primary and key secondary endpoints comparable registered trials used and at which time points, and draft the estimand for a trial objective following ICH E9(R1) (treatment, population, variable, intercurrent events with their strategies, population-level summary). Use when the user is choosing or justifying a primary or secondary endpoint, asks what similar trials measured and when, or asks to define, check or write an estimand or an intercurrent-event strategy.
---

# Endpoints and estimand

Show what comparable trials measured and when, then draft the estimand that the chosen endpoint serves, in the structure of ICH E9(R1).

## Ground rules

1. Every trial, id, endpoint, time point and guideline passage in the answer comes from a mondoTWIN tool result. Never invent a trial, a number or a quotation. Statistical reasoning is yours; label it as reasoning, not as evidence.
2. Cite each trial by its registry id linked to its registry page: `linkouts.registry_url` from `get_clinical_study`, or `https://clinicaltrials.gov/study/<NCT id>` for an NCT id. Cite each guideline passage by guideline, section (when visible) and a short verbatim quote.
3. Say plainly when a search returned nothing, with the query used.
4. State the data as-of date: ClinicalTrials.gov as of 13 Jul 2026; EU CTR/CTIS, ISRCTN, ANZCTR and UMIN as of 14 May 2026; ICH guidance as held in the mondoTWIN database.
5. No patient data. If the user pastes identifiable patient information, do not repeat it and ask them to remove it.
6. The output is a draft for qualified professionals (clinicians and statisticians) to validate. Say so in one line at the end.
7. Answer in the user's language. Quote registry text and guidelines verbatim in their original language.
8. Use only `search_clinical_studies`, `get_clinical_study`, `search_ich_guidelines`, `get_ich_guideline_full` and, for biomarker endpoints only, `get_disease_targets`. If the mondoTWIN tools are not available, tell the user to connect the mondoTWIN connector, and stop.

## How the tools behave

**Trials.** `search_text` keeps the records in which every word (or "quoted phrase") appears, each in any of title, hypothesis, diseases, drugs, sponsor or main centre: use one to three distinctive words (the drug, the disease), plus `phases`, `statuses` and `icd11_chapters` filters (the chapter is the first character of a relevant hit's `icd11_code`: `B…` is "11", `C…` is "12"; it removes passing mentions). If a query returns 0, drop a word and retry before concluding anything. A phase filter drops records without a phase (common for ANZCTR, EU CTR, UMIN): run one search without it. `get_clinical_study` is 10 to 40 KB per trial: read at most 6. `endpoints[]` has `type` (primary or secondary), `label` and `description`; the time frame is inside the label or description, there is no separate field. When no time point is given, write "time frame not stated". EU CTR endpoint text is often cut off. `results.why_stopped` explains terminated trials. `clinical_outcome`, `positive_results` and `results.outcome_explanation` are machine-generated summaries, not registry results: do not report them as outcomes.

**ICH.** `search_ich_guidelines` returns passages with a bare `guide_type`. `E9` holds E9(R1), with the estimands addendum in the A sections: A.3.1 intercurrent events, A.3.2 strategies, A.3.3 estimand attributes, A.3.4 constructing an estimand, A.5.2 sensitivity analysis, A.5.3 supplementary analysis. `M11` holds the Step 2 draft protocol template, which has an estimand table per objective; say "M11 (Step 2 draft)" when citing it. Read section numbers from headings inside the passage; if none is visible, search again with the heading words or write "section not identified". Good queries use E9(R1) vocabulary: "intercurrent event", "treatment policy strategy", "hypothetical strategy", "composite variable strategy", "while on treatment strategy", "principal stratum", "population-level summary", "sensitivity analysis". `get_ich_guideline_full` returns the whole guideline and can exceed the app's tool-result limit: use it at most once, only if search cannot find a section you need.

## Steps

1. **Profile.** Indication, phase, intervention, comparator, and the objective the endpoint must answer. If the objective is unclear, state the one you assume.
2. **Endpoint landscape.** Search for comparable trials (2 or 3 searches), read 4 to 6 records, and tabulate primary and key secondary endpoints with their time points. Count the patterns ("4 of 6 used … at week 52").
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

**Endpoints and estimand:** *[study and objective in one line]* · Data as of: ClinicalTrials.gov 13 Jul 2026; other registries 14 May 2026.

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
