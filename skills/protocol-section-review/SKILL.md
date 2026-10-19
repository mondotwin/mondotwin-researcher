---
name: protocol-section-review
description: Review a draft clinical trial protocol or synopsis section (objectives, endpoints, eligibility criteria, sample size rationale, schedule of activities) against comparable registered trials and ICH E6(R3), E8(R1), E9(R1) and M11, and return numbered, reasoned changes, each backed by trial registry ids or a quoted guideline passage and marked "to validate". Use when the user pastes or describes part of a protocol and asks for a review, critique, gap check, improvements, or consistency with ICH or with what similar trials did.
---

# Protocol section review

Check one protocol section against what comparable trials did and what ICH guidance says, and propose concrete, evidenced changes.

## Ground rules

1. Every trial, id, number, date and guideline passage in the answer comes from a mondoTWIN tool result. Never invent a trial, a value or a quotation. Background knowledge may shape the review and the search terms, but a statement presented as evidence must come from a tool result.
2. Cite each trial by its registry id linked to its registry page: `linkouts.registry_url` from `get_clinical_study`, or `https://clinicaltrials.gov/study/<NCT id>` for an NCT id. Cite each guideline passage by guideline, section (when visible) and a short verbatim quote.
3. Say plainly when a search returned nothing, with the query used.
4. State the data as-of date: ClinicalTrials.gov as of 13 Jul 2026; EU CTR/CTIS, ISRCTN, ANZCTR and UMIN as of 14 May 2026; ICH and FDA guidance as held in the mondoTWIN database.
5. No patient data. A protocol is fine; identifiable patient information is not. If the user pastes it, do not repeat it and ask them to remove it.
6. The output is a draft for qualified professionals to validate. Every proposed change is marked "to validate".
7. Answer in the user's language. Quote registry text and guidelines verbatim in their original language; add a translation when the user's language differs.
8. Use only `search_clinical_studies`, `get_clinical_study`, `search_ich_guidelines`, `get_ich_guideline_full`, `search_fda_guidelines`, `get_fda_guideline_full` and, for a mechanistic rationale only, `get_disease_targets`. `query_clinical_studies` is also available for counts and comparisons across many trials (read-only SQL over the trial tables; see the query-clinical-studies skill). If the mondoTWIN tools are not available, tell the user to connect the mondoTWIN connector, and stop.

## How the tools behave

**Trials.** `search_text` keeps the records in which every word (or "quoted phrase") appears, each in any of title, hypothesis, diseases, drugs, sponsor or main centre, so use one to three distinctive words (drug, disease), plus `phases`, `statuses` and `icd11_chapters` filters (the chapter is the first character of a relevant hit's `icd11_code`: `B…` is "11", `C…` is "12"; it removes passing mentions). If a query returns 0, drop a word and retry before concluding anything. Records without a phase (common for ANZCTR, EU CTR, UMIN) are dropped by a phase filter: run one search without it. `enrollment_actual` is an actual count only for completed, terminated or withdrawn trials. Fields such as `hypothesis`, `clinical_outcome`, `control_type`, `strategy_tags` and `clinical_trial_prediction` are machine-generated: do not cite them as registry facts. `get_clinical_study` is 10 to 40 KB per trial: read at most 5. The endpoint time frame is inside `endpoints[].description`. `criteria_markdown` can be abridged (EU CTR text cut off; "More exclusion criteria may apply").

**ICH.** `search_ich_guidelines` is a semantic search returning passages with a bare `guide_type`: `E6` holds E6(R3), `E8` holds E8(R1), `E9` holds E9(R1) including the estimands addendum (the A sections, e.g. A.3.3 estimand attributes), `M11` holds the Step 2 draft guideline and template (2022, template draft updated March 2025). Say "M11 (Step 2 draft)" when citing it. There is no section field: read the section number from headings inside the passage ("## A.3.2", "2.8.5"); if there is none, search again with the passage's heading words, or write "section not identified". Use the guideline's vocabulary (interim analysis, group sequential, estimand, intercurrent event, critical to quality factors); very specific design phrases match poorly, so try 2 or 3 phrasings.

**FDA.** `search_fda_guidelines` returns passages with a `filename` (the title with underscores). The corpus lacks some drug guidance, for example the guidance on adaptive designs for drugs and biologics. A passage that only cites another guidance is not that guidance: say the cited document is not in the database. Draft or final status and dates are visible only when the passage shows them.

**Full texts.** `get_ich_guideline_full` and `get_fda_guideline_full` return a whole document (E8(R1) is about 107,000 characters) and can exceed the app's tool-result limit. Use them only when a needed section cannot be found by search, at most once per answer.

## Which guidance to check, by section type

| Section | Search the guidance for |
|---|---|
| Objectives and endpoints | E9(R1) estimand attributes and intercurrent-event strategies; M11 objectives, endpoints and estimand table; E8(R1) choice of endpoints |
| Eligibility | E8(R1) study population (narrow vs representative); M11 inclusion and exclusion criteria (each criterion assessable as yes or no) |
| Sample size rationale | E9 sample size, assumptions and the alignment of the analysis with the estimand; interim analysis and group sequential designs |
| Schedule of activities | E8(R1) critical to quality factors and participant burden; E6(R3) proportionate, risk-based approach; M11 schedule of activities |

## Steps

1. **Identify** the section type, phase, indication, intervention and comparator from the text. Do not ask for the full protocol.
2. **Find 3 to 5 comparable trials** (one or two searches, then detail calls on the closest) and note what they did for this section: endpoints and time points, criteria and thresholds, sizes, why trials stopped.
3. **Retrieve the guidance** for the section type (2 to 4 searches) and keep only passages that bear on the draft.
4. **Compare** the draft with both. Look for: missing elements (an endpoint without a time point; an objective without an estimand; a criterion that cannot be answered yes or no), inconsistencies (an endpoint that does not match the objective; a sample size that does not match the primary endpoint), departures from what comparable trials did (justify them or change them), and avoidable burden.
5. **Write the answer** in the format below. Do not rewrite the whole section unless asked; offer to.

## Output format

**Review of:** *[section type, study in one line]* · Data as of: ClinicalTrials.gov 13 Jul 2026; other registries 14 May 2026.
Evidence used: *[k]* comparable trials (*ids*), guidance passages from *[guidelines]*.

Numbered changes, most important first:

**1. *[The change in a few words]*** · *to validate (clinician / statistician / regulatory)*
- Current: "*quote from the draft*"
- Proposed: "*replacement text, or what to add*"
- Why: *one to three sentences of reasoning*
- Evidence: [NCT…](url) used *…*; ICH E9(R1) A.3.2: "*verbatim quote*"
- Type: required by guidance / common practice in comparable trials / reviewer suggestion

After the list:

- **Questions for the team:** points where the right answer depends on information not in the draft.
- **What was not checked:** guidance not found in the database, trials without readable criteria.
- One closing line: these are draft suggestions for qualified professionals to validate.
