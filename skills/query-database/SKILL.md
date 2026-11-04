---
name: query-database
description: Use when a question is to be answered from MondoTwin's twin of the pharma world through the twin MCP tools (twin_catalog, twin_find, twin_sql, twin_text, twin_record): a count, list, landscape or timeline of trials, drugs, products, approvals, indications, exclusivity, generics, companies, ownership, filings, papers, diseases or prevalence; reading a raw registry record, a label's text or an abstract; and before any analysis that cites the twin.
when_to_use: Trigger phrases include "find trials for", "how many trials", "drugs tested for", "trials of <drug>", "approved where", "authorised in", "indications of", "what the label says", "loss of exclusivity", "generics", "who holds", "who owns", "the sponsor's trials", "the group's pipeline", "what did the trial find", "why was it stopped", "papers reporting", "prevalence", "ICD-11 code", "Orphanet", "as of <date>", "twin_sql", "twin_find", "query the twin", "query the database". The frontmatter fields stay on one line each.
---

# Query the database

## What you are querying

MondoTwin's twin of the pharma world: 58 tables (schema v1), served read only as PostgreSQL views, one active release at
a time. `release_info` names the release and the capture date of every source (`inputs.captured`); `catalog` describes
every table (`about`) and every column; `coverage` says which source was read, for which scope, through which date.
Table names are bare on the search path: write `trial`, never a schema prefix.

Three facts shape every answer.

- **Compiled.** A value is a decision (a code chosen, a drug linked, an outcome classed) with a measured accuracy per
  method. A `confidence` column is that measurement, never a per-row probability.
- **Dated on two axes.** What the source states (`event_on`, `start_date`, `approved_on`) and when the twin knew it
  (`known_from`). Only the tables whose `about` says so support `known_from <= <date>` as an as-of filter
  (`trial_outcome`, `regulatory_event`); `trial` holds today's state and its `known_from` is the first posting.
- **Partial on purpose.** Thirteen tables are published empty until a source is read: `deal`, `deal_party`, `payment`,
  `right_grant`, `royalty`, `royalty_party`, `royalty_paid`, `financial`, `org_runway`, `price`, `spend`, `claim`,
  `opportunity`. An empty result from them is "unknown", never "none".

Every row carries `sources` (record keys `<source>:<id>`, readable with `twin_record`) and `known_from`. Cite both,
with the release id, for every fact you take.

## The five tools

| Tool | What it does | Limits |
|---|---|---|
| `twin_catalog(table)` | the release ids and the tables whose name contains the word, with `about`, columns and comments | `table=""` returns about 80 KB and omits the columns of 36 tables: ask per word (`"trial"` gives the nine trial tables). The same text is in `catalog`: `SELECT about FROM catalog WHERE table_name = '...'`; for the columns use `SELECT c->>'name', c->>'comment' FROM catalog, jsonb_array_elements(columns) c WHERE table_name = '...'` (the `columns` cell itself is cut at 2,000 characters) |
| `twin_find(text, limit=20)` | a name or an outside id to the twin's ids: exact case-insensitive matches first (`via: name` or `external_id`), then word-prefix matches | up to 100. Outside ids as `scheme:value`: `nct:NCT…`, `pmid:…`, `doi:…`, `unii:…`, `chembl:…`, `cas:…`, `rxcui:…`, `drugbank:DB…` (11,168 drugs carry one), `cik:0000310158`, `lei:…`, `icd11:DD71`, `orpha:771`. A brand returns the drug and one `product` per territory; a common word returns trials whose title starts with it: take the hit of the entity type you want |
| `twin_sql(query, max_rows=100)` | one read-only `SELECT`, `WITH`, `VALUES` or `TABLE` over the views | 15 s statement timeout; `max_rows` 1 to 500; the reply stops near 60 KB (`"more": true`, a `note`) and cuts a cell over 2,000 characters; one query at a time. `{"error": ...}` instead of an exception: read it, fix, retry once |
| `twin_text(query, limit=10)` | full-text search over `passage` (registry records as the models read them, labels, EMA indication texts, abstracts): words ANDed, `"phrase"`, `OR`, `-word` | up to 25 passages, each with `passage_id`, `document_id`, `kind`, `section`, `snippet`, the first 2,000 characters, `links` (the trials, drugs, products it concerns). Ranks are flat; a query of many `OR` terms can time out: search one concept at a time |
| `twin_record(key, offset=0)` | the raw record behind a `sources` key, as captured: ClinicalTrials.gov JSON (API v2 shape), a registry export row, a label, a filing index, a paper | 20,000 characters a page (`next_offset`); served only for captures the principal is cleared for |

Tool names are bare here. In Claude Code they carry the server's prefix (`mcp__mondotwin__twin_sql`); the ChatGPT and
claude.ai connectors list them bare. If the tools are not available, say so and stop: never answer from memory as
though from the database.

## The method

1. **Count trials, not records.** `trial` is deduplicated across ten registries; a trial's member records are in
   `trial_record` (`is_primary` marks the one its values come from) and its registry ids in `external_id` (schemes
   `nct`, `eudract`, `ctis`, `isrctn`, `anzctr`, `umin`, `chictr`, `jrct`, `cris`, `itmctr`). Always
   `count(DISTINCT trial_id)`. The merge is a judge's decision (measured agreement 0.939): a CTIS- or EudraCT-only
   trial whose title equals an NCT trial's may be an unmerged duplicate; say so rather than counting it twice.
2. **Resolve names to ids** with `twin_find`, or in SQL through `entity_name` (indexed on `lower(name)`) and
   `external_id` (indexed on `scheme, value`). Then read the entity's own row and say which ids you kept. A disease
   name matches an ICD-11 stem, its residual `.Y`/`.Z` codes and an Orphanet entity at once (recipe A2); a company is
   several `org` rows (A3); a drug's salts point at a moiety (`drug.moiety_id`).
3. **Read the table's `about`** the first time you use it in a session (`twin_catalog(table=...)` or `catalog`): it
   holds the as-of rule, the traps and what is empty. `references/tables.md` says which table to read.
4. **Dates and starts.** `start_date` and `completion_date` carry a `_precision` (day, month, year; a month is
   stored as its first day). Milestones are `trial_event` rows (`registered`, `first patient`, `last patient`,
   `primary completion`, `completion`, `results posted`, `stopped`) with `actual` true for an actual date and false
   for a planned one. `registered_on` is the earliest public posting over the member records. "Started in 2024" is
   `trial_event.kind = 'first patient'` in 2024 (say whether planned dates count) or `start_date` with its
   precision; the two can differ. **There is no study-type column**: the twin holds no observational studies and no
   healthy-volunteer ClinicalTrials.gov studies; "drug trials" are the trials with a `trial_drug` row,
   "interventional" those with a `purpose` or a phase other than `NA`. State which proxy you used.
5. **Drugs are substances.** `drug` has 174,945 rows; join on `moiety_id` to roll salts up. Names live in
   `entity_name` (kinds `display`, `synonym`, `brand`, `code`, `as written`), ids in `external_id` (`unii`, `chembl`,
   `cas`, `pubchem`, `inchikey`, `rxcui`, `drugbank`). `code_names` holds some development codes, not all: search a
   code through `entity_name` and `trial_drug.raw_name` too. `twin_find` on an INN returns the moiety and its salts,
   hydrates, impurities and metabolites as separate drug rows: keep the row with `moiety_id = drug_id` (or
   `kind = display`) and query by `moiety_id`. `drug_target` holds Open Targets' stated mechanisms only (7,708 rows
   over 5,930 drugs): a drug with no row has no stated mechanism in the twin, not no target.
6. **Confirm at the source** anything you quote or that surprises you: `twin_record(<record key>)` for a registry
   entry or a label, `SELECT text FROM passage WHERE passage_id = ...` for a sentence, the `evidence` spans of
   `trial_outcome` and `approved_indication`.
7. **Conditions are ICD-11 codes** (`icd11:<code>`), plus `orpha_id` on the same row for a rare disease. A disease
   and its subtypes: `icd11_id LIKE 'icd11:DD71%'`, or `disease.ancestor_ids @> ARRAY['icd11:DD71']` (populated for
   ICD-11 and Orphanet). Query by `icd11_id`; add `orpha_id` for a rare disease. `raw_text` is the trial's own
   wording; `role = 'primary'` means listed first, not the treated disease. Traps: no NSCLC concept (2C25 is lung
   cancer, 2C25.1 small cell: read `raw_text`); colorectal cancer sits at 2B91.Z, so 2B90 alone misses it; residual
   codes end in Y or Z. A trial with no row stated no condition; a row with `icd11_id` NULL named a condition that
   got no code.
1. **Count trials, not records.** `trial` is deduplicated across ten registries; a trial's member records are in
   `trial_record` (`is_primary` marks the one its values come from) and its registry ids in `external_id` (schemes
   `nct`, `eudract`, `ctis`, `isrctn`, `anzctr`, `umin`, `chictr`, `jrct`, `cris`, `itmctr`). Always
   `count(DISTINCT trial_id)`.
2. **Phase vocabulary:** `phase_1`, `phase_1_2`, `phase_2`, `phase_2_3`, `phase_3`, `phase_4`, `NA` (devices,
   behavioural, observational, no phase stated), `NULL` (a phase no rule reads). "Phase 2" is
   `phase IN ('phase_1_2', 'phase_2', 'phase_2_3')`.
3. **Status is today's state:** `completed`, `recruiting`, `active_not_recruiting`, `not_yet_recruiting`,
   `terminated`, `withdrawn`, `suspended`, `unknown` (the registry's own word), `not_recruiting` (the ICTRP
   registries fold completed, terminated and active into it), `NULL`. `status_stated` keeps the registry's words.
   Never filter `trial.known_from` for an as-of state.
4. **Dates:** `start_date` and `completion_date` carry a `_precision` (day, month, year; a month is stored as its
   first day). Milestones are `trial_event` rows (`registered`, `first patient`, `last patient`,
   `primary completion`, `completion`, `results posted`, `stopped`) with `actual` true for an actual date and false
   for a planned one. `registered_on` is the earliest public posting over the member records.
5. **Drugs are substances.** `drug` has 174,945 rows; join on `moiety_id` to roll salts up. Names live in
   `entity_name` (kinds `display`, `synonym`, `brand`, `code`, `as written`), ids in `external_id` (`unii`, `chembl`,
   `cas`, `pubchem`, `inchikey`, `rxcui`, `drugbank`). `code_names` holds some development codes, not all: search a
   code through `entity_name` and `trial_drug.raw_name` too. `drug_target` holds Open Targets' stated mechanisms only
   (7,708 rows over 5,930 drugs): a drug with no row has no stated mechanism in the twin, not no target.
6. **`trial_drug.role`:** `investigational` (the test article), `comparator`, `background`. `drug_id` NULL means the
   component's name was not linked to a substance; `raw_name` still holds it. Placebo, procedures and devices have no
   row.
7. **Conditions are ICD-11 codes** (`icd11:<code>`), plus `orpha_id` for a rare disease. A disease and its subtypes:
   `icd11_id LIKE 'icd11:DD71%'`, or `disease.ancestor_ids @> ARRAY['icd11:DD71']` (populated for ICD-11 and
   Orphanet). `raw_text` is the trial's own wording; `role = 'primary'` means listed first, not the treated disease.
   Traps: no NSCLC concept (2C25 is lung cancer, 2C25.1 small cell: read `raw_text`); colorectal cancer sits at
   2B91.Z, so 2B90 alone misses it; residual codes end in Y or Z. A trial with no row stated no condition (healthy
   volunteers); a row with `icd11_id` NULL named a condition that got no code.
8. **Products are per regulator and territory** (41 territories; `EU-central` for a Commission authorisation,
   `NCA:<cc>` rows for national registers, `XI` Northern Ireland). `product_drug` links the moiety; `product_holder`
   the current holder (`valid_from` is the capture day, not a transfer date: no as-of). `marketed` is true, false or
   NULL as the source says. For a per-territory verdict read `territory_status` (`authorised`, `not authorised`,
   `unknown`, with `basis`) rather than inferring from `product`. The holder resolves to a company for nearly every
   US and EU-central product and for a minority elsewhere (Health Canada 9 %, PMDA 21 %, MFDS 15 %, Swissmedic 12 %,
   ANVISA, TFDA and HSA 0 %): `holder_org_id` is then NULL, so carry `product_holder.holder_as_written` beside
   `org.name`. A territory can have two product sources (Japan: the PMDA new-drug list and the MHLW price list),
   giving two rows per presentation with different brand strings: aggregate by brand, not by row.
9. **Labels cover few products.** `approved_indication` holds 5,312 statements for 1,040 of 403,501 products (989 US,
   51 EU-central: the indication stages read a slice of the US labels and EMA medicines, and no national register
   states indications), so `territory_status` carries a `disease_id` on about 1,000 of its 149,273 rows and the rest
   read "any indication". No row is not "no indication": say "label not read" and fall back to `regulatory_event`
   (approvals, efficacy supplements, CHMP opinions, tentative approvals, refusals, planned PDUFA dates; its
   `known_from` equals `event_on`) and to the raw record.
10. **Exclusivity:** `protection` lists Orange Book patents and exclusivities (US, complete), Purple Book
    exclusivities and EU regulatory protection, per product: one patent appears once per product strength and use
    code, so `DISTINCT` on (`number`, `expires`); exclusivity codes (orphan, nce, new_indication, pediatric, …)
    surface with `kind NOT LIKE 'patent%'`. `loe` is one derived row per (moiety, US | EU-central): read `basis`
    before the date (`generic launch`, `settlement`, `generic approval`, `protection`, `blocked`,
    `national authorisation first`, `nothing on file`); it is a floor built from what is on file (biologic patents,
    SPCs and most settlements are not sources). `cheap_version` counts generics marketed (`generic_products`) and
    authorised (`generic_products_authorised`) per presentation; trust a zero only where `covered` is true.
11. **Companies come in several rows.** `trial_org.resolution_status` says how a sponsor string became an org:
    `resolved` (a registered entity), `clustered` or `minted` (the string's own company), `person`, `not a name`.
    Roll a group up with `ownership_effective` (holder → companies, `controls`, `ultimate`), count a trial once
    through `trial_org."primary"`. `ownership` dates a stake only where a source does (`valid_from` + basis).
    `filing` indexes 3.29 M SEC filings with form, date and 8-K `items` but holds no filing text.
12. **Papers are the ones trials list.** `paper` holds the papers linked to the release's trials, not a corpus
    (`coverage`, entity `paper`). `paper_trial.relation`: `exact` reports the trial, `protocol`, `related` (reviews,
    comparisons, older papers, digests of 8+ trials, retracted). Declared links (`found_by` with `paper_declared`)
    carry no measured precision and can attach an earlier-phase abstract to a later-phase record: read the title
    before citing. Check `paper.retracted`. `paper_drug` reaches a drug only through its trials.
13. **Results are classes with evidence.** `trial_outcome.outcome`: `positive_significant`, `positive_trend`,
    `descriptive_met`, `failed`, `harmful_or_inferior`, `stopped_operational`, `inconclusive`, `not_reported` (by
    rule when nothing reports). `basis` says what it rests on; `evidence` holds the quoted spans with `passage_id`.
    `known_from` dates the evidence the class rests on (a paper can postdate the registry's posting): "what was
    knowable on a date" reads `trial_outcome.known_from <= date` **and** `trial_event` (`kind = 'results posted'`)
    together, and "when were results first public" is the `results posted` event. `program.best_outcome` summarises
    per (drug, disease).
14. **Text lives in `passage`.** Sections of a registry record: `population`, `arms`, `strategy/<part>` (title,
    phase, status, dates, design, descriptions), `results/primary_results`, `results/publication_results/<n>`; of a
    paper: `abstract`; of a label: `indications_and_usage`; of an EMA record: `therapeutic_indication`. SQL over
    3.7 M passages times out: reach text through `twin_text`, through a known `passage_id` (the `evidence` spans give
    one), or through `document_link` (trial → document) then `passage.document_id`. A registry id is not a word
    `twin_text` indexes: resolve it through `external_id`, then read the trial's passages.
15. **Searching names in SQL:** `entity_name` with `lower(name) = lower('...')` or `name ILIKE 'prefix%'` is indexed;
    a regex over `trial.title` scans 491,690 rows and runs 10 to 15 s against the 15 s limit. Prefer `twin_find`,
    `entity_name`, `trial_drug.raw_name`, and `twin_text` for words inside records.
16. **Territories and regulators:** ISO-2 codes (`XI` is Northern Ireland), `EU-central` (a Commission
    authorisation), `EEA-national` (the coverage scope of the national registers read through EMA's Article 57
    file); regulators are `FDA`, `EC`, `EMA`, `PMDA/MHLW`, `HC`, `TFDA`, `MFDS`, `ANVISA`, `Swissmedic`, `HSA`,
    `NMPA`, `NCA:<cc>`. `territory` holds countries and dated sets (`EU-central`, `EEA`, `Europe`, `world`, `ex-US`);
    a country against a set reads `members` as of a day (the `about` gives the SQL).
17. **Empty tables answer "unknown".** Before "no deal", "no royalty", "no price", read `coverage`: the row has
    `through_basis = 'not read'`. `territory_status` never reads `licensed out` while `right_grant` is empty.
18. **Confidence is a measurement.** `trial_condition.confidence`, `trial_drug.confidence`,
    `trial_outcome.confidence`, `paper_trial.confidence` and `trial_org.confidence` are the measured accuracy of the
    method that made the row (the `about` names the evaluation). Quote them as such.

## The answer

1. **Release and as-of.** `release@...` from `release_info`, the source's `through_on` from `coverage` for the tables
   used, and the as-of date applied, if any.
2. **Scope.** The ids kept (drug, disease codes, orgs), the filters, the assumptions.
3. **Result.** The rows or the number, each fact with its record keys (`sources`) or `passage_id`; a count says
   "trials of the release", not "trials in the world", unless `coverage` is complete for the scope.
4. **Limits.** What the twin does not hold for this question (empty tables, `complete = false`, unlinked
   components, NULL codes), stated as unknown.
5. **The SQL**, so the reader can rerun it.

## Files

- `references/tables.md`: the 58 tables, one line each, with row counts and the value sets of the columns that matter.
- `references/recipes.md`: the query shapes (A1 to I2), each run against the twin, with what came back.
- The database describes itself: `catalog` (every table's `about`, every column's comment and value set) and
  `coverage`. The platform repo's `docs/mondotwin/pharma-schema-v1.md` is the same schema written out at length, with
  the known gaps of the served release; read it when you have the repo, the catalog when you do not.
