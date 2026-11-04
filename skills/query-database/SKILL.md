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
| `twin_find(text, limit=20)` | a name or an outside id to the twin's ids: exact case-insensitive matches first (`via: name` or `external_id`), then word-prefix matches | up to 100. Outside ids as `scheme:value`: `nct:NCT…`, `pmid:…`, `doi:…`, `unii:…`, `chembl:…`, `cas:…`, `rxcui:…`, `drugbank:DB…` (11,168 drugs carry one), `cik:0000310158`, `lei:…`, `icd11:DD71`, `orpha:771`. A brand returns the drug and one `product` per territory; an INN returns the moiety and its salts and forms; a common word returns trials whose title starts with it: take the hit of the entity type you want |
| `twin_sql(query, max_rows=100)` | one read-only `SELECT`, `WITH`, `VALUES` or `TABLE` over the views | 15 s statement timeout; `max_rows` 1 to 500; the reply stops near 60 KB (`"more": true`, a `note`) and cuts a cell over 2,000 characters; one query at a time. `{"error": ...}` instead of an exception: read it, fix, retry once |
| `twin_text(query, limit=10)` | full-text search over `passage` (registry records as the models read them, labels, EMA indication texts, abstracts): words ANDed, `"phrase"`, `OR`, `-word` | up to 25 passages, each with `passage_id`, `document_id`, `kind`, `section`, `snippet`, the first 2,000 characters, `links` (the trials, drugs, products a registry record or label concerns; an abstract carries `paper_id` instead and reaches its trials through `paper_trial`). Ranks are flat; a query of many `OR` terms can time out: search one concept at a time. A registry id is not an indexed word |
| `twin_record(key, offset=0)` | the raw record behind a `sources` key, as captured: ClinicalTrials.gov JSON (API v2 shape), a registry export row, a label, a filing index, a paper | 20,000 characters a page (`next_offset`); a CTIS record can run to 24 pages (its `protocol_code` and titles are in the first); served only for captures the principal is cleared for |

Tool names are bare here. In Claude Code they carry the server's prefix (`mcp__mondotwin__twin_sql`); the ChatGPT and
claude.ai connectors list them bare. If the mondoTWIN tools are not available, or a call answers that authorization is
required, the mondoTWIN connector is not connected yet: never answer from memory as though from the database. Tell the
user in one short message how to connect it, and continue once the tools answer: in Claude, select Connect when Claude
offers it, or Customize > Connectors > mondotwin > Connect; in ChatGPT, Plugins > mondoTWIN Researcher (or the
mondoTWIN connector) > Connect, then start a new chat. They allow access, then sign in or register with a mondoTWIN
account; a new account waits for mondoTWIN's approval.

## The method

1. **Frame** the question as entities (drug, disease, company, territory, period), a measure (count, list, median,
   share, date) and a grouping. State the assumption you make for anything left open.
2. **Resolve names to ids** with `twin_find`, or in SQL through `entity_name` (indexed on `lower(name)`) and
   `external_id` (indexed on `scheme, value`), one lookup per call. Then read the entity's own row and say which ids
   you kept. A disease name matches an ICD-11 stem, its residual `.Y`/`.Z` codes and an Orphanet entity at once
   (recipe A2); a company is several `org` rows (A3); a drug's salts point at a moiety (`drug.moiety_id`).
3. **Read the table's `about`** the first time you use it in a session, several at once:
   `SELECT table_name, about FROM catalog WHERE table_name IN ('trial', 'trial_event', ...)`. It holds the as-of
   rule, the traps and what is empty. `references/tables.md` says which table to read.
4. **Check coverage** whenever the answer could be "none", "not approved", "no deal", or when a count will be read
   as a count of the world (recipe I1). `complete = true` makes an absence evidence; `complete = false` or
   `through_basis = 'not read'` makes it "unknown".
5. **Query by id**, in a recipe's shape (`references/recipes.md`): filter on an indexed id first, join after, count
   `DISTINCT trial_id`, select the columns you need, `LIMIT`. One statement, one call.
6. **Confirm at the source** anything you quote or that surprises you: `twin_record(<record key>)` for a registry
   entry or a label, `SELECT text FROM passage WHERE passage_id = ...` for a sentence, the `evidence` spans of
   `trial_outcome` and `approved_indication`.
7. **Write the answer** in the format below.

## Which table answers which question

| Question | Tables | Recipe |
|---|---|---|
| Resolve a drug, a disease, a company, a registry id | `twin_find`; `entity_name`, `external_id`, `disease`, `org` | A1, A2, A3 |
| Trials of a drug, by phase, status, role | `trial_drug` (moiety, `role`) → `trial`; registry ids in `external_id` | B1, B2 |
| A drug's indication landscape | `program`; detail from `trial_condition` | B3 |
| Where a drug is authorised, under which brand, by whom | `territory_status` (summary); `product_drug` → `product` → `product_holder` → `org` (detail) | C1, C2 |
| What a label says it is for, with the quote | `approved_indication` (+ `passage`); `coverage` first | C3 |
| When it was approved, supplements, CHMP opinions, PDUFA dates | `regulatory_event` | C4 |
| What protects it and when exclusivity ends | `protection`, `loe` (read `basis`) | C5 |
| How many generics or biosimilars are marketed | `cheap_version` | C6 |
| Orphan or PRIME designations | `designation` (EMA only today) | C7 |
| Trials of a disease and its subtypes | `trial_condition` (`icd11_id` prefix or `disease.ancestor_ids`, `orpha_id`) → `trial` | D1 |
| What is tested for a disease, ranked | `trial_condition` ⋈ `trial_drug` (role investigational) ⋈ `drug` | D2 |
| Trials started per year | `trial_event` (`kind = 'first patient'`) | D3 |
| How rare, how frequent; Orphanet ↔ ICD-11 | `prevalence`, `disease_map`, `disease.rare` | D4 |
| A sponsor's or a group's trials | `trial_org` (`"primary"`, role) → `ownership_effective` → `org` | E1 |
| Who owns whom, since when | `ownership`, `ownership_effective` | E2 |
| A company's filings | `external_id` (cik) → `filing_party` → `filing` | E3 |
| Who partners on a drug, while `deal` is empty | `trial_org` roles over a drug's trials; `program.lead_org_id` | E4 |
| What a trial found; what was known of it on a date | `trial_outcome` (`known_from <= date`), `evidence` → `passage`; `trial_event` (`results posted`) | F1, F2 |
| Why trials stopped | `trial.why_stopped`, `stop_class` (a keyword rule: read the text) | F3 |
| Papers reporting a trial or a drug | `paper_trial` (`relation = 'exact'`), `paper_drug`, `paper` | F4 |
| A quote, a sentence, a passage; the raw record | `twin_text`; `passage`; `twin_record` | G1, G2 |
| The registered endpoints of a trial | `trial_endpoint` | G3 |
| Is an absence evidence? What is empty? | `coverage`, `catalog.row_count` | I1, I2 |

## Rules that bite (R1 to R18)

R1. **Count trials, not records.** `trial` is deduplicated across seven registry sources (ClinicalTrials.gov,
    EudraCT, CTIS, ISRCTN, ANZCTR, UMIN and WHO's ICTRP export, which carries ChiCTR, jRCT, CRIS and ITMCTR: ten
    `registry` values). A trial's member records are in `trial_record` (`is_primary` marks the one its values come
    from) and its registry ids in `external_id`. Always `count(DISTINCT trial_id)`. The merge is a judge's decision
    (measured agreement 0.939). Two patterns look like duplicates and need a word: a CTIS- or EudraCT-only trial whose
    title equals an NCT trial's (an unmerged duplicate, or a second EU application of the same programme), and a
    ClinicalTrials.gov "programme" record that covers two studies registered separately in CTIS (the CTIS raw record's
    `protocol_code`, e.g. "… study 2", tells). Count trials as the registries do and say which rows are one programme.
    `brief_title` is NULL when the registry gives no lay title (CTIS-only trials): select `coalesce(brief_title, title)`.
R2. **Phase vocabulary:** `phase_1`, `phase_1_2`, `phase_2`, `phase_2_3`, `phase_3`, `phase_4`, `NA` (devices,
    behavioural, no phase stated), `NULL` (a phase no rule reads). "Phase 2" is
    `phase IN ('phase_1_2', 'phase_2', 'phase_2_3')`.
R3. **Status is today's state:** `completed`, `recruiting`, `active_not_recruiting`, `not_yet_recruiting`,
    `terminated`, `withdrawn`, `suspended`, `unknown` (the registry's own word), `not_recruiting` (the ICTRP
    registries fold completed, terminated and active into it), `NULL`. `status_stated` keeps the registry's words.
    Never filter `trial.known_from` for an as-of state.
R4. **Dates and starts.** `start_date` and `completion_date` carry a `_precision` (day, month, year; a month is
    stored as its first day). Milestones are `trial_event` rows (`registered`, `first patient`, `last patient`,
    `primary completion`, `completion`, `results posted`, `stopped`), one per member record that states the date, with
    `actual` **true** (stated actual), **false** (planned, estimated, anticipated) or **NULL** (the registry does not
    say; a posting is actual). Aggregate per trial (`min(event_on)`) before grouping by `actual`, or a trial with three
    records counts three times. `registered_on` is the earliest public posting over the member records. "Started in
    2024" is a `first patient` event in 2024 (say whether planned and unflagged dates count) or `start_date` with its
    precision; the two differ (ulcerative colitis drug trials in 2024: 67 by event, 58 by `start_date`). **There is no
    study-type column**: the twin holds no observational studies and no healthy-volunteer ClinicalTrials.gov studies;
    "drug trials" are the trials with a `trial_drug` row, "interventional" those with a `purpose` or a phase other
    than `NA`. State the proxy you used.
R5. **Drugs are substances.** `drug` has 174,945 rows; join on `moiety_id` to roll salts up. Names live in
    `entity_name` (kinds `display`, `synonym`, `brand`, `code`, `as written`), ids in `external_id` (`unii`, `chembl`,
    `cas`, `pubchem`, `inchikey`, `rxcui`, `drugbank`). `code_names` holds some development codes, not all: search a
    code through `entity_name` and, inside a filtered set of trials, `trial_drug.raw_name`. `twin_find` on an INN
    returns the moiety and its salts, hydrates, impurities and metabolites as separate drug rows: keep the row with
    `moiety_id = drug_id` (or `kind = display`) and query by `moiety_id`. `drug_target` holds Open Targets' stated
    mechanisms only (7,708 rows over 5,930 drugs): a drug with no row has no stated mechanism in the twin, not no
    target.
R6. **`trial_drug.role`:** `investigational` (the test article), `comparator`, `background`. `drug_id` NULL means the
    component's name was not linked to a substance; `raw_name` still holds it. Placebo, procedures and devices have no
    row.
R7. **Conditions are ICD-11 codes** (`icd11:<code>`), plus `orpha_id` on the same row for a rare disease. A disease
    and its subtypes: `icd11_id LIKE 'icd11:DD71%'`, or `disease.ancestor_ids @> ARRAY['icd11:DD71']` (populated for
    ICD-11 and Orphanet). Query by `icd11_id`; add `orpha_id` for a rare disease (for a common one it adds nothing).
    `raw_text` is the trial's own wording; `role = 'primary'` means listed first, not the treated disease. Traps: no
    NSCLC concept (2C25 is lung cancer, 2C25.1 small cell: read `raw_text`); colorectal cancer sits at 2B91.Z, so 2B90
    alone misses it; residual codes end in Y or Z. A trial with no row stated no condition; a row with `icd11_id` NULL
    named a condition that got no code.
R8. **Products are per regulator and territory** (41 territories; `EU-central` for a Commission authorisation,
    `NCA:<cc>` rows for national registers, `XI` Northern Ireland). `product_drug` links the moiety; `product_holder`
    the current holder (`valid_from` is the capture day, not a transfer date: no as-of). `marketed` means: true, a US
    label on file in openFDA's export, EMA's status "Authorised", or a national register's own flag; false, Drugs@FDA
    "Discontinued" (an approved product never launched and a discontinued one are not told apart); NULL, the source is
    silent (Article 57 national rows, Swissmedic, HSA). For a per-territory verdict read `territory_status`
    (`authorised`, `not authorised`, `unknown`, with `basis`) rather than inferring from `product`; it has one row per
    (territory, holder, basis), so CA appears twice (`product` and `product, not marketed`) and Japan once per source:
    aggregate by territory. A `CN` row rests on web evidence (source `web_evidence/tavily`, record key
    `request:h1:…`, `complete = false`, brand NULL): say so. The holder resolves to a company for nearly every US and
    EU-central product and for a minority elsewhere (Health Canada 9 %, PMDA 21 %, MFDS 15 %, Swissmedic 12 %,
    ANVISA, TFDA and HSA 0 %): `holder_org_id` is then NULL, so carry `product_holder.holder_as_written` beside
    `org.name` (risankizumab: an AbbVie affiliate named in every territory, resolved in five). Japan has two product
    sources (the PMDA new-drug list and the MHLW price list) with different brand strings: aggregate by brand.
R9. **Labels cover few products.** `approved_indication` holds 5,312 statements for 1,040 of 403,501 products (989 US,
    51 EU-central: the indication stages read a slice of the US labels and EMA medicines, and no national register
    states indications), so `territory_status` carries a `disease_id` on about 1,000 of its 149,273 rows and the rest
    read "any indication". No row is not "no indication": say "label not read" and fall back to `regulatory_event`
    (approvals, efficacy supplements, CHMP opinions, tentative approvals, refusals, planned PDUFA dates; its
    `known_from` equals `event_on`) and to the raw record.
R10. **Exclusivity:** `protection` lists Orange Book patents and exclusivities (US, complete), Purple Book
    exclusivities and EU regulatory protection, per product: a patent appears once per product strength, per use code
    and per `kind` (US 8,962,629 is both `patent_substance` and `patent_use` for RINVOQ), so group by (`kind`,
    `number`, `expires`); exclusivity codes (orphan, nce, new_indication, pediatric, …) surface with
    `kind NOT LIKE 'patent%'`; `extension` is 'unknown' or NULL where the source does not say. `loe` is one derived row
    per (moiety, US | EU-central): read `basis` before the date (`generic launch`, `settlement`, `generic approval`,
    `protection`, `blocked`, `national authorisation first`, `nothing on file`); it is a floor built from what is on
    file (biologic patents, SPCs and most settlements are not sources), and the patents it counted are listed only in
    the prose of `completeness` (a long cell: read that column alone). `cheap_version` counts generics marketed
    (`generic_products`) and authorised (`generic_products_authorised`) per presentation; trust a zero only where
    `covered` is true. A worked example inside a column comment can lag the data (the apixaban figures quoted in
    `cheap_version`'s comment are older than the rows): the rows win.
R11. **Companies come in several rows.** `trial_org.resolution_status` says how a sponsor string became an org:
    `resolved` (a registered entity), `clustered` or `minted` (the string's own company), `person`, `not a name`.
    Roll a group up with `ownership_effective` (holder → companies, `controls`, `ultimate`), count a trial once
    through `trial_org."primary"` (a reserved word: quote it). `ownership` dates a stake only where a source does
    (`valid_from` + basis). `org.size_tier` is the group's tier where the company belongs to one. `filing` indexes
    3.29 M SEC filings with form, date and 8-K `items` but holds no filing text; `filing_party.role` is often NULL.
R12. **Papers are the ones trials list.** `paper` holds the papers linked to the release's trials, not a corpus
    (`coverage`, entity `paper`). `paper_trial.relation`: `exact` reports the trial, `protocol`, `related` (reviews,
    comparisons, older papers, digests of 8+ trials, retracted). Declared links (`found_by` with `paper_declared`)
    carry no measured precision and can attach an earlier-phase abstract to a later-phase record: read the title
    before citing. Check `paper.retracted`. `paper_drug` reaches a drug only through its trials.
R13. **Results are classes with evidence.** `trial_outcome.outcome`: `positive_significant`, `positive_trend`,
    `descriptive_met`, `failed`, `harmful_or_inferior`, `stopped_operational`, `inconclusive`, `not_reported` (by
    rule when nothing reports). `basis` says what it rests on; `evidence` holds the quoted spans with `passage_id`.
    `known_from` dates the evidence the class rests on (a paper can postdate the registry's posting): "what was
    knowable on a date" reads `trial_outcome.known_from <= date` **and** `trial_event` (`kind = 'results posted'`)
    together, and "when were results first public" is the `results posted` event. `program.best_outcome` summarises
    per (drug, disease).
R14. **Text lives in `passage`.** Sections of a registry record: `population`, `arms`, `strategy/<part>` (title,
    phase, status, dates, design, descriptions), `results/primary_results`, `results/publication_results/<n>`; of a
    paper: `abstract`; of a label: `indications_and_usage`; of an EMA record: `therapeutic_indication`. SQL over
    3.7 M passages times out: reach text through `twin_text`, through a known `passage_id` (the `evidence` spans give
    one), or through `document_link` (trial → document) then `passage.document_id`. A registry id is not a word
    `twin_text` indexes: resolve it through `external_id`, then read the trial's passages.
R15. **Searching names in SQL:** `entity_name` is indexed on `lower(name)` and by word prefix, and `external_id` on
    (`scheme`, `value`), **one predicate at a time**: an `OR` across columns or tables, or a `UNION` of lookups in one
    statement, loses the index and runs past 15 s; make one call per lookup. A regex over `trial.title` scans 491,690
    rows (10 to 15 s); an `ILIKE` over `trial_drug.raw_name` scans 765,483 and times out: filter the trial ids first
    (by condition, sponsor or phase) and search `raw_name` inside that set, or find the code with `twin_find`. Prefer
    `twin_find`, `entity_name`, and `twin_text` for words inside records.
R16. **Territories and regulators:** ISO-2 codes (`XI` is Northern Ireland), `EU-central` (a Commission
    authorisation), `EEA-national` (the coverage scope of the national registers read through EMA's Article 57
    file); regulators are `FDA`, `EC`, `EMA`, `PMDA/MHLW`, `HC`, `TFDA`, `MFDS`, `ANVISA`, `Swissmedic`, `HSA`,
    `NMPA`, `NCA:<cc>`. `territory` holds countries and dated sets (`EU-central`, `EEA`, `Europe`, `world`, `ex-US`);
    a country against a set reads `members` as of a day (the `about` gives the SQL).
R17. **Empty tables answer "unknown".** Before "no deal", "no royalty", "no price", read `coverage`: the row has
    `through_basis = 'not read'` and `source` NULL (`coalesce(source, '-')` when you concatenate). `territory_status`
    never reads `licensed out` while `right_grant` is empty. The only partner evidence the twin holds is
    circumstantial: `trial_org` roles over a drug's trials in registration order (a lead sponsor becoming a
    collaborator, a new lead appearing: recipe E4) and `program.lead_org_id`. Offer it as such, never as a deal.
R18. **Confidence is a measurement.** `trial_condition.confidence`, `trial_drug.confidence`,
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
