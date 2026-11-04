---
name: protocol-section-review
description: Review a draft clinical trial protocol or synopsis section (objectives, endpoints, eligibility criteria, sample size rationale, schedule of activities) against comparable registered trials and ICH E6(R3), E8(R1), E9(R1) and M11, and return numbered, reasoned changes, each backed by trial registry ids or a quoted guideline passage and marked "to validate". Use when the user pastes or describes part of a protocol and asks for a review, critique, gap check, improvements, or consistency with ICH or with what similar trials did.
---

# Protocol section review

Check one protocol section against what comparable trials did and what ICH guidance says, and propose concrete, evidenced changes.

## Ground rules

1. Every trial, id, number, date and guideline passage in the answer comes from a mondoTWIN tool result. Never invent a trial, a value or a quotation. Background knowledge may shape the review and the search terms, but a statement presented as evidence must come from a tool result.
2. Cite each trial by its registry id linked to its registry page: `https://clinicaltrials.gov/study/<NCT id>` for an NCT id, `https://www.isrctn.com/<ISRCTN id>` for ISRCTN, `https://www.clinicaltrialsregister.eu/ctr-search/search?query=<EudraCT number>` for EU CTR; other registries' ids unlinked, with the registry's name. Cite each guideline passage by guideline, section (when visible) and a short verbatim quote.
3. Say plainly when a search returned nothing, with the query used.
4. State the data as-of date from the tools, never from memory: each registry's `through_on` in `coverage` (`SELECT source, through_on FROM coverage WHERE entity_type = 'trial record'`, through `twin_sql`) and the release id; ICH and FDA guidance as held in the mondoTWIN database.
5. No patient data. A protocol is fine; identifiable patient information is not. If the user pastes it, do not repeat it and ask them to remove it.
6. The output is a draft for qualified professionals to validate. Every proposed change is marked "to validate".
7. Answer in the user's language. Quote registry text and guidelines verbatim in their original language; add a translation when the user's language differs.
8. Use only `search_clinical_studies`, `get_clinical_study`, `search_ich_guidelines`, `get_ich_guideline_full`, `search_fda_guidelines`, `get_fda_guideline_full` and, for a mechanistic rationale only, `get_disease_targets`. `twin_sql` and `twin_record` read a trial's criteria and arms text, and `twin_sql` also counts and compares across many trials (read-only SQL over the twin's tables; see the query-database skill). If the mondoTWIN tools are not available, or a call answers that authorization is required, the mondoTWIN connector is not connected yet: never answer from memory as though from the database. Tell the user in one short message how to connect it, and continue once the tools answer: in Claude, select Connect when Claude offers it, or Customize > Connectors > mondotwin > Connect; in ChatGPT, Plugins > mondoTWIN Researcher (or the mondoTWIN connector) > Connect, then start a new chat. They allow access, then sign in or register with a mondoTWIN account; a new account waits for mondoTWIN's approval.

## How the tools behave

**Trials.** `search_text` keeps the trials in which every word (or "quoted phrase") appears, each in any of the title, a condition, a drug or a sponsor as the registry wrote them, so use one to three distinctive words (drug, disease), plus `phases`, `statuses` and `icd11_chapters` filters (the chapter is the first character of a relevant hit's `icd11_code`: `B…` is "11", `C…` is "12"; it removes passing mentions). If a query returns 0, drop a word and retry before concluding anything. Trials that state no phase (`NA`) are dropped by a phase filter: run one search without it. `enrollment_actual` is the stated final count and `enrollment_planned` the target; a trial carries one or the other. `outcome`, `clinical_outcome`, `stop_class` and, for staff accounts, `clinical_trial_prediction` are machine-made: do not cite them as registry facts. Read at most 5 trials with `get_clinical_study`; its endpoints carry `measure`, `description` and `timepoint` (the time frame as stated). It returns no eligibility criteria and no design fields (masking, control type): v1 holds no such columns. The criteria and arms text is in the trial's `population` and `arms` passages: `twin_sql(query="SELECT p.section, left(p.text, 2000) AS text FROM document_link l JOIN passage p ON p.document_id = l.document_id WHERE l.entity_type = 'trial' AND l.entity_id = '<id>' AND p.section IN ('population', 'arms', 'rendered/population', 'rendered/arms')")`; otherwise in the raw record (`twin_record` with a `record_key` from the trial's `records`). Criteria text can be abridged (EU CTR text cut off; "More exclusion criteria may apply").

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

**Review of:** *[section type, study in one line]* · Data as of: *[each registry's through_on, from coverage]* · Release *[id]*.
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
