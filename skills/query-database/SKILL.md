---
name: query-database
description: Answers questions from MondoTwin's published twin of the pharma world (schema v1, trials, drugs and products, approvals and indications, exclusivity, companies and filings, papers, diseases) through the five twin MCP tools twin_catalog, twin_find, twin_sql, twin_text and twin_record. Use for any count, list, landscape or timeline question, "where is it authorised and by whom", "who owns", "what did the trial find", "why did it stop", "when does exclusivity end", "which papers report it", for reading a raw registry record or a label's text, and before any analysis that cites the twin. Read-only; every answer carries the release id and the record keys of the rows it used.
when_to_use: Trigger phrases include "find trials for", "how many trials", "drugs tested for", "trials of <drug>", "approved where", "authorised in", "indications of", "what the label says", "loss of exclusivity", "generics", "who holds", "who owns", "the sponsor's trials", "the group's pipeline", "what did the trial find", "why was it stopped", "papers reporting", "prevalence", "ICD-11 code", "Orphanet", "as of <date>", "twin_sql", "twin_find", "query the twin", "query the database". The platform's skill engine reads the frontmatter line by line, so these two fields stay on one line each.
---

# Query the database

## What you are querying

MondoTwin's twin of the pharma world, compiled by the twin engine into versioned releases and served read only
through PostgreSQL views. The agents' schema is **v1**: 58 tables named bare (`trial`, `drug`, `product`, `org`,
`paper`, `disease`, ...), one active release at a time; `release_info` says which and what it was compiled from.
Beside it the same principal is served the two domain releases v1 is projected from (`drugs_shadow.*`, the drug
registry's working shape; `pharma_shadow.*`): prefer the v1 tables.

The schema describes itself. Each table's `about` and each column's comment are in the database (`catalog`, and
`twin_catalog` serves them), so this skill holds the **method, the router and the rules**, not a copy of the schema:
`references/tables.md` is the one-line map you route by; `twin_catalog(table=<name>)` is the text you read before
you query a table for the first time.

Three facts shape every answer. The twin is **compiled**: a value is a decision (a code chosen, a drug linked, an
outcome classed) with a measured accuracy, not a scrape. It is **dated on two axes**: what the source states and
when the twin knew it. And it is **partial on purpose**: `coverage` says which sources were read for which scope,
and whole families (deals, payments, royalties, rights, prices, spend, financials, claims, opportunities) are
published empty until a source is read.

## The five tools

| Tool | What it does | Limits (measured 3 Nov 2026) |
|---|---|---|
| `twin_catalog(table)` | the domains and active releases, the tables matching the word with `about`, columns and comments, row counts | `table=""` returns 80 KB and leaves the columns of 36 tables out: call it **per word** (`"trial"` gives the ten trial tables, `"loe"` one). The reply is the text of `catalog.about` and the column comments |
| `twin_find(text, limit=20)` | a name or an outside id as `scheme:value` (`nct:NCT…`, `pmid:…`, `unii:DPT0O3T46P`, `cik:0000310158`, `lei:…`, `icd11:CB03.4`, `orpha:2032`) to the twin's ids: exact case-insensitive matches first (`via: name` or `via: external_id`), then word-prefix matches (`via: name prefix`) | up to 100; a common name returns trials whose title starts with it and salts and labelled forms beside the drug; a brand returns the drug and one `product` per territory: take the hit of the entity type you want. `drugbank:DB…` answers only for the 11,168 drugs that carry one (pembrolizumab has none: use `unii:`, `chembl:`, `cas:`, `rxcui:` or the name) |
| `twin_sql(query, max_rows=100)` | one read-only `SELECT`, `WITH`, `VALUES` or `TABLE` over the views, bare table names | 15 s statement timeout; `max_rows` 1 to 500; the reply stops at about 60 KB and cuts a cell over 2,000 characters; 4 connections per server process, so one query at a time. `{"error": ...}` instead of an exception: read it, fix the query, retry once |
| `twin_text(query, limit=10)` | full-text search over `passage` (registry records as the models read them, labels, EMA indication texts, abstracts) with web syntax: words ANDed, `"phrase"`, `OR`, `-word` | up to 25 passages, each with its `passage_id`, document, `links` (the trials, drugs, products it concerns), `markings` and the first 2,000 characters |
| `twin_record(key, offset=0)` | the raw record behind a `sources` key, as captured: ClinicalTrials.gov's JSON (API v2 shape), a label, a filing, a paper | 20,000 characters a page (`next_offset`); served only from captures the principal is cleared for |

Tool names are written bare here. In Claude Code they carry the server's prefix (`mcp__mondotwin__twin_sql`); in
ChatGPT and claude.ai the `mondoTWIN` connector lists them bare. If the tools are not available, say so and stop:
never answer from memory as though from the database.

## The method

1. **Frame** the question as a set of entities (which drug, disease, company, territory, period), a measure (count,
   list, median, share, date) and a grouping. State the assumptions you make for anything the user left open.
2. **Resolve names to ids** with `twin_find`. Then read the entity's own row (`drug.name`, `drug.moiety_id`;
   `disease.name`, `disease.parent_ids`; `org.name` with `ownership_effective`) and say which ids you kept. A disease
   name can match a parent, its residual codes (`.Y`, `.Z`) and an Orphanet entity at once (recipe A2); a company is
   several `org` rows (recipe A3).
3. **Read coverage** (`coverage`, recipe I2) whenever the answer could be "none", "not approved", "no deal", or when
   a count will be read as a count of the world.
4. **Read the table** with `twin_catalog(table=<name>)` the first time you use it in a session: its `about` holds the
   as-of rule, the traps and what is empty. `references/tables.md` says which table to read.
5. **Query by id**, in a recipe's shape (`references/recipes.md`): filter on an indexed id first, join after, count
   `DISTINCT trial_id`, select the columns you need, `LIMIT`. One statement, one call.
6. **Confirm at the source** for anything you will quote or that surprises you: `twin_record(<record key>)` for a
   registry entry or a label, `SELECT text FROM passage WHERE passage_id = ...` for a quoted sentence, the
   `evidence` spans of an outcome or an indication row.
7. **Write the answer** in the format below: the release, the scope, the rows with their record keys, the limits,
   and the SQL.

## Which table answers which question

| Question | Tables | Recipe |
|---|---|---|
| Trials of a drug, by phase and role | `trial_drug` (moiety) → `trial` | C1 |
| A drug's indications landscape (which diseases, how far, with what result) | `program`, then `trial_condition` for detail | C2 |
| Where a drug is authorised, under which brand, by whom | `product_drug` → `product` → `product_holder` → `org`; the summary `territory_status` | C3, C4 |
| What a label says it is for | `approved_indication` (+ `passage` for the quote); `coverage` first | C5 |
| When it was approved, how many supplements, PDUFA dates | `regulatory_event` | C6 |
| What protects it and when exclusivity ends | `protection`, `loe` (read `basis`) | C7 |
| How many generics are marketed | `cheap_version` | C8 |
| Trials of a disease and its subtypes | `trial_condition` (`icd11_id` prefix or `disease.ancestor_ids`, `orpha_id`) → `trial` | D1 |
| What is tested for a disease, ranked | `trial_condition` ⋈ `trial_drug` (role investigational) ⋈ `drug` | D2 |
| Trials started per year | `trial_event` (`kind = 'first patient'`) | D3 |
| How rare, how frequent; Orphanet ↔ ICD-11 | `prevalence`, `disease_map`, `disease.rare` | D4 |
| A sponsor's or a group's trials | `trial_org` (`"primary"`) → `ownership_effective` → `org` | E1, E2 |
| A company's filings | `external_id` (cik) → `filing_party` → `filing` | E3 |
| What a trial found; what was known of it on a date | `trial_outcome` (`known_from <= date`) | F1, F3 |
| Why trials stopped | `trial.why_stopped`, `stop_class` (a keyword rule: read the text) | F2, F4 |
| Likely pathway, novelty angles, endpoint type | `trial_strategy` (inferences, measured per field) | G1 |
| Everything about one trial | the one-pager over `trial`, `trial_condition`, `trial_drug`, `trial_org`, `trial_outcome`, `trial_event`, `trial_record`, `paper_trial`; `trial_endpoint`; `trial_strategy`; `twin_record` | B1 to B4 |
| Papers reporting a trial or a drug | `paper_trial` (`relation = 'exact'`), `paper_drug`, `paper` | B1, H4 |
| A sentence in a record, a label, an abstract | `twin_text`, then `passage` by id | H1 to H3 |
| What the database holds and how current it is | `release_info`, `coverage`, `catalog` | I1 to I3 |
| A country against EU-central, as of a date | `territory.members` | J1 |

## Rules that prevent wrong answers

1. **The release first.** `SELECT release, compiled_at, inputs->'captured' FROM release_info` gives the release id and
   each source's capture day; `coverage.through_on` gives a source's own as-of date. Put them in the answer. Never
   write an as-of date from memory or from an older answer: it changes with every release.
2. **Coverage before "none".** `coverage` has one row per (source, entity type, scope). `complete` true means an
   absence is evidence; false means a slice or an incomplete source; `through_basis = 'not read'` (source NULL) means
   no source exists for that entity type: deal, payment, royalty, right_grant, financial, price, spend, claim. For
   those the answer is "unknown, not read", never "none". `approved_indication` is a slice (989 US and 51 EU-central
   products of 403,501); the trial tables are a release's slice (`pharma`, `trial`, complete false).
3. **Count trials, not rows.** `trial_record`, `trial_condition`, `trial_drug`, `trial_org`, `paper_trial` fan out:
   `count(DISTINCT trial_id)`, or filter with `trial_id IN (SELECT ...)`. A count by lead sponsor keeps
   `trial_org."primary"` so a trial counts once.
4. **Phase sets.** `phase_1_2` and `phase_2_3` are combined phases: phase 2 is `phase IN ('phase_1_2', 'phase_2',
   'phase_2_3')`. `'NA'` (190,726) is not applicable or none stated (devices, behavioural, observational, most
   ANZCTR, UMIN and ISRCTN records); NULL (73,874) is a stated phase no rule reads (27,399 of ChiCTR's 73,823 primary
   trials, whose form writes '0'). Report the `NA` and NULL counts beside any phase breakdown.
5. **Status is today's.** `trial` holds the latest state at capture; `not_recruiting` (59,284) is ICTRP's fold of
   completed, terminated and active, so say which registries are in a status count. There is no status history. A
   trial existed on a date when `registered_on <= date`; its milestones are `trial_event`; what was known of its
   result is `trial_outcome` with `known_from <= date`. **Never filter `trial.known_from` for an as-of state.** Only
   `trial_outcome` and `regulatory_event` take `known_from` as an as-of filter.
6. **Diseases by code, read as text.** Find the code with `twin_find`, keep the `display` hit (`icd11:CB03.4` for
   idiopathic pulmonary fibrosis; the parent `icd11:CB03`, idiopathic interstitial pneumonitis, carries the same
   name as a synonym and would widen the set), then filter `trial_condition.icd11_id LIKE 'icd11:CB03.4%'` or join
   `disease.ancestor_ids @> ARRAY['icd11:CB03.4']` for the subtree; add `orpha_id` for a rare disease. `raw_text` is
   the trial's wording and `disease.name` the code's title: never group by the code and label by the mention.
   ICD-11 traps: no NSCLC concept (2C25 is lung cancer, 2C25.1 small cell: use `raw_text`); colorectal cancer NOS
   sits at 2B91 (rectosigmoid junction), so 2B90 alone misses it; residual codes end in Y or Z. `role = 'primary'`
   means listed first, not treated. A row with `icd11_id` NULL is an uncoded condition; no row is no condition stated.
7. **Drugs by moiety.** `trial_drug.drug_id`, `product_drug.drug_id`, `program.drug_id`, `loe.drug_id` are moieties.
   17,207 `drug` rows are salts or forms (`moiety_id <> drug_id`): if `twin_find` returned one, query with its
   `moiety_id`. `trial_drug.role` separates the test article (`investigational`) from `comparator` and `background`;
   pembrolizumab is background in 404 phase 1 trials and investigational in 214.
8. **Companies are several rows.** `trial_org.resolution_status`: only `resolved` (95,934 rows, 5,623 companies) names
   a registered company; `clustered` and `minted` org_ids are made from the sponsor string, so one sponsor sits under
   several ids (Merck Sharp & Dohme: ten `org` rows, three of which roll up to Merck & Co., Inc.). Find them all with
   `twin_find`, roll up through `ownership_effective` (current state) and say which rows you summed. `org.kind` is
   NULL on 123,945 of 309,844 rows.
9. **Approval is `territory_status`, not `program`.** `program.highest_phase = 'approved'` and `approval_known` come
   from the indication slice: pembrolizumab reads `phase_4` with `approval_known` false in lung cancer. Read
   `territory_status` (`status`, `basis`) for where a drug is authorised, `product` and `product_holder` for the
   brand and the holder. A NULL `holder_org_id` is an unresolved name (most national registers), not no holder:
   `product_holder.holder_as_written` keeps it. A per-indication row exists only where a label was read; a
   national product has none even then.
10. **Exclusivity: `basis` before `expires`.** `loe.basis` says what set the date (generic launch, settlement,
    generic approval, protection, blocked, national authorisation first, nothing on file) and `completeness` what was
    not read (biologic patents, SPCs, most settlements). `protection` lists what the registries list: which patent is
    the compound patent is not stated.
11. **`confidence` is a method's measured accuracy**, one number per method from the eval journal, never a per-row
    probability and never a model's own score: ICD-11 coding 0.800, arm drugs 0.817, sponsor to its own filer 0.883,
    outcome class per model (Opus 5.5 0.882, Sonnet 5.5 0.856), the paper judge 0.982. `trial_strategy` has no
    confidence column: its tags are one model's inferences, measured per field (pathway 0.903, endpoint type 0.785,
    duration bucket 0.914, novelty angles exact set 0.548): a lead, not a filter of record. `trial_outcome`:
    339,834 of 491,223 are `not_reported`; `failed` means a tested primary endpoint missed, not a weak result.
12. **Say what is the registry's and what is the twin's.** Registry statements: `title`, `phase`, `status`,
    `status_stated`, dates, enrolment, `countries`, `has_results`, `why_stopped`, `trial_endpoint`, `raw_text`,
    `raw_name`, the products and their dates. Decisions of the twin, to label as such: the ICD-11 and Orphanet codes,
    the drug links and roles, `org_id`, `stop_class` (a keyword rule: quote `why_stopped`), `trial_outcome`,
    `trial_strategy`, `approved_indication.icd11_id`, `loe`, `program`, `territory_status`.
13. **Text has a licence.** `markings` on a document or passage (`licence:ictrp`, `licence:ctgov-terms`, `pii`)
    say what it is: quote only what the answer needs, cite the `passage_id` and the document's title. Registry
    passages (sections `population`, `arms`, `strategy/trial`, `strategy/outcomes`, `results/<part>`) are the text
    the models read, rendered from the record; the record itself is `twin_record`.
14. **Papers.** `paper_trial.relation = 'exact'` (237,349) is a paper reporting the trial; `related` (365,517) is a
    review, a meta-analysis, a dataset, an older paper or a digest. Check `paper.retracted`. `paper_drug` counts
    comparators and background drugs: join `trial_drug.role`.
15. **Nothing here writes.** Shortlists, triage states and notes are the analyst tools' job, never SQL.

## Performance (measured 3 Nov 2026)

| Shape | Time |
|---|---|
| a GROUP BY on an indexed `trial` column (phase, status) over 491,690 rows | about 0.1 s |
| joins filtered by one id (a drug's trials, a trial's rows, a drug's products) | 0.01 to 3 s |
| a disease subtree through `ancestor_ids` over `trial_condition` (lung cancer: 14,350 trials) | 5 s |
| `trial_org` GROUP BY over 916,656 rows; `external_id` GROUP BY scheme over 2.6M | 5 to 7 s |
| a whole-table GROUP BY on `entity_name` (3.1M) or `passage` (3.7M) | **times out** |

Filter on an id or an indexed column first (`trial_id`, `drug_id`, `org_id`, `icd11_id`, `phase`, `status`,
`(scheme, value)`, `record_key`, `passage_id`, `document_id`, `(kind, event_on)`); never `SELECT *` from a table
with `sources` and `evidence` columns; `left(text, n)` for wide text; page with `OFFSET` only when aggregation
cannot replace it. Several queries at once share four connections: a burst makes a trivial query time out.

## How to write the answer

**The question in one line.** Release `twin@<compiled_at>`; data as of: the capture days of the sources used
(`release_info.inputs.captured`) or `coverage.through_on`.
Scope: the ids kept (drug_id with its name, disease ids with their titles, org ids), the filters, the registries.

One results table, columns named in plain words.

- **What the rows show:** two to four sentences with the counts.
- **Rows behind it:** the trials (registry id from `trial_record.record_key`, title), products or papers, at most
  ten, each with its record keys (`sources`) and, for a quote, its `passage_id`.
- **Limits:** coverage (slice or complete), twin-made columns used and their measured accuracy, `NA` and NULL
  shares, truncation, codes possibly missed, tables empty by design.
- **Query:** the SQL of each query, in a code block.
- One closing line: compiled from registered and regulatory records for professionals to verify.

## What the twin does not hold, and what to do

- **Deals, rights, payments, royalties, prices, public spend, financial statements, web claims, opportunity
  findings**: tables exist, published empty, `coverage` says 'not read'. Say "unknown, not read".
- **Patents on a trial's regimen or a method of use**: `protection` lists Orange Book and Purple Book entries and EU
  protection periods per product; no method-of-use search per trial and no freedom-to-operate verdict exists. Say
  so; do not infer one from `loe`.
- **Indications for national products** (Canada, Brazil, Switzerland, Korea, Singapore, China, Japan's price list,
  the EEA national registers): no text source yet; the moiety's US or EU-central indications are the nearest
  evidence and must be labelled as such.
- **Population, eligibility, design (masking, control), sites**: not published as columns. The registry's text is in
  `passage` (sections `population`, `arms`) and in `twin_record`; read it, do not count over it.
- **Success predictions, candidate buckets, priority scores, MPC lists**: platform product data, not the twin.
- **Full-text filings and papers**: `filing` holds dates, forms and 8-K items, not text; `paper` holds abstracts in
  `passage` where the corpus scope allows.
- **DrugBank and MedDRA content**: licensed; `drug` carries the registry's names and `external_id` a DrugBank id for
  11,168 drugs, nothing more.

## References

- `references/tables.md`: the 58 tables, one line each: one row per, when to reach for it, the trap.
- `references/recipes.md`: the queries, each run against the served release on 3 Nov 2026 with its result.
- `references/from-the-warehouse.md`: the legacy MPC Finder warehouse's tables and JSONB paths and what answers them
  now, for porting the analyst skills and for readers who know the old names.
- For maintainers: `docs/mondotwin/pharma-schema-v1.md` (generated from `twin_pharma/schema/columns.py`; it may
  describe columns the served release does not carry yet, so the served truth is `twin_catalog(table)`).
