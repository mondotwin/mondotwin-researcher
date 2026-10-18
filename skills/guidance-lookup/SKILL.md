---
name: guidance-lookup
description: Answer a question about ICH or FDA regulatory guidance with the exact passage quoted, the guideline id and the section. Use when the user asks what ICH (for example E6(R3), E8(R1), E9(R1), E10, M11, the Q series) or FDA guidance says about a topic, wants the source or exact wording of a regulatory expectation, or wants to check a statement against the guidance.
---

# Guidance lookup

Find the passage in ICH or FDA guidance that answers the question, quote it, and say exactly where it comes from.

## Ground rules

1. Every quotation, guideline id and section in the answer comes from a mondoTWIN tool result. Never invent or paraphrase a passage as if it were a quote. You may name a document you believe is relevant but did not retrieve; label it "not retrieved from the database".
2. Cite each passage by guideline (with its revision), section (when visible) and a verbatim quote. Add the passage number (`chunk_id`) so the user can find it again.
3. Say plainly when the search found nothing relevant, with the queries used. The corpus is not complete.
4. State that the answer uses ICH and FDA guidance as held in the mondoTWIN database, and that the user should confirm the current version on ich.org or fda.gov.
5. No patient data. If the user pastes identifiable patient information, do not repeat it and ask them to remove it.
6. The output is a draft for qualified professionals (regulatory affairs) to validate. Say so in one line at the end.
7. Answer in the user's language. Quote the guidance verbatim in English; add a translation when the user's language differs.
8. Use only `search_ich_guidelines`, `get_ich_guideline_full`, `search_fda_guidelines` and `get_fda_guideline_full`. If the mondoTWIN tools are not available, tell the user to connect the mondoTWIN connector, and stop.

## How the tools behave

**ICH.** `search_ich_guidelines` is a semantic search that returns passages with a bare `guide_type` and a `chunk_id`. Known mappings: `E6` holds E6(R3) (it uses "participant" and numbers such as 2.8.5); `E8` holds E8(R1); `E9` holds E9(R1), with the estimands addendum in the A sections (for example A.3.2 strategies, A.5.2 sensitivity analysis); `M11` holds the Step 2 draft guideline and template (2022, template draft updated March 2025): cite it as "M11 (Step 2 draft)". For other codes, take the revision from the passage text or say it is not visible. There is no section field: read the section number from a heading inside the passage ("## A.3.2", "3.3", "2.10.3"). If none is visible, search again with the passage's heading words or opening sentence, or write "section not identified".

**FDA.** `search_fda_guidelines` returns passages with a `filename`, which is the document title with underscores (write it as a title). Draft or final status shows only when the passage contains "Draft — Not for Implementation" or a date. The corpus lacks some guidance documents (for example the guidance on adaptive designs for drugs and biologics). A passage that only mentions or cites another guidance is not that guidance: say the cited document is not in the database rather than answering from the citing text.

**Queries.** Phrase queries in the guideline's own vocabulary (for example "interim analysis", "group sequential", "estimand", "intercurrent event", "non-inferiority margin", "critical to quality factors"). Very specific design phrases match poorly: try 2 or 3 phrasings and `k` of 5 to 8 before concluding that the guidance is silent.

**Full texts.** `get_ich_guideline_full` and `get_fda_guideline_full` return the whole document (E8(R1) is about 107,000 characters) and can exceed the app's tool-result limit. Use one only when a section or its surrounding text cannot be located by search, at most once per answer.

## Steps

1. **Classify** the question: ICH, FDA or both; which guideline is likely; what exactly must be answered.
2. **Search** with 2 or 3 phrasings in guideline vocabulary. For questions that span ICH and FDA, search both.
3. **Select** the passages that answer the question directly. Read neighbouring passages (a follow-up search with the heading) when the answer continues or the section number is missing.
4. **Answer** in the format below. Keep what the guidance says separate from your interpretation.

## Output format

**Answer:** two to four sentences that answer the question, in the user's language.

**What the guidance says**

> "*verbatim passage*"
> *ICH E9(R1), section A.3.2* (passage *chunk_id*)

> "*verbatim passage*"
> *FDA, Non-Inferiority Clinical Trials, section III.C* (passage *chunk_id*)

**Interpretation:** how the passages apply to the user's situation, marked as interpretation. Leave this out if the user only asked for the text.

**Not found / limits:** queries that found nothing, documents the database does not hold, version or status not visible.

One closing line: quoted from the guidance as held in the mondoTWIN database; confirm the current version on ich.org or fda.gov; a draft for qualified professionals to validate.
