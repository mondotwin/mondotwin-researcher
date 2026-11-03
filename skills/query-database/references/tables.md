# The tables, one line each: what to reach for

The map an agent routes by before it calls `twin_catalog(table=<name>)` for the table's full comment and columns.
Row counts are the served release of 3 Nov 2026 (`twin@2026-11-02T23:56:16+00:00`); they move with every release,
so read them as scale. The authoritative text is in the database: `catalog.about` and each column's comment.

Every table but `catalog` and `release_info` ends with the four provenance columns: `sources` (text[] of record keys
`<source>:<id>`, open one with `twin_record`), `known_from` (the earliest day the sources were public), `known_from_basis`,
`known_from_precision`.

## Navigation

| Table | One row per | Reach for it when | Trap |
|---|---|---|---|
| `entity_name` (3.1M) | a name of a trial, drug, product, org or disease | you have a name; prefer `twin_find`, which reads it | a whole-table GROUP BY times out; `kind` tells display from synonym from 'as written' |
| `external_id` (2.6M) | an outside id (nct, eudract, pmid, doi, unii, cik, lei, icd11, orpha, nda, anda, bla, eu_ma, tfda, hc, mfds, nca:<cc>, ...) | you have an id; `WHERE scheme = ... AND value = ...` is indexed | an id maps to several entities only where the twin has not merged them |
| `source_record` (2.6M) | a raw record a published row cites | which capture a key came from, its licence `markings` | content is not here: `twin_record(key)` |
| `coverage` (58) | (source, entity type, scope) read, through which date | before any "none", "not approved", "no deal" | `through_basis = 'not read'` with source NULL: no source exists for that entity type |
| `trial_provenance` (491,690) | a trial's recorded steps (stages, keys, ledger keys) | how a trial's rows were made | refs and summaries, never values |
| `claim` (0) | a web fact, cited and checked | nothing yet | empty: unknown, not "no claim" |

## Reference

| Table | One row per | Reach for it when | Trap |
|---|---|---|---|
| `disease` (27,115) | an ICD-11 code (chapters 01-22, 25) or an Orphanet entity | a disease's title, parents, subtree (`ancestor_ids @> ARRAY['icd11:2C25']`), `rare`, `status` | `name` is the code's title, not the trial's words; Orphanet groupings have `rare` NULL; filter `status = 'active'` |
| `disease_map` (19,885) | a mapping Orphanet or Mondo states | carrying a disease between orpha:, icd11: and mondo: | only `relation = 'exact'` keeps the meaning |
| `prevalence` (17,108) | Orphanet's figure per (disorder, measure, area) | how frequent a rare disease is | per 100,000; `year` NULL; areas are codes or Orphanet's names |
| `territory` (145) | a country or a named set (EU-central, EEA, EU, europe, ex-US, rest-of-world, north-america) | a country against a set, as of a date | `members` is dated: GB left EU-central on 2021-01-01 |

## Trials

| Table | One row per | Reach for it when | Trap |
|---|---|---|---|
| `trial` (491,690) | a deduplicated study | phase, status, dates, enrolment, countries, has_results, why_stopped, stop_class | latest state at capture: never `known_from` as an as-of filter; `phase` NULL (73,874) is a stated phase no rule reads, 'NA' (190,726) devices, behavioural, observational, none stated; `status` 'not_recruiting' (59,284) is ICTRP's fold of completed, terminated and active |
| `trial_record` (539,356) | a registry record of a trial | which registries hold it, which is primary, the record key for `twin_record` | one trial, several records: count trials, not records |
| `trial_condition` (631,814) | a coded condition of a trial | the diseases a trial is about; filter by `icd11_id LIKE 'icd11:2C25%'` or join `disease.ancestor_ids`; `orpha_id` for rare | `raw_text` is the trial's wording; `role = 'primary'` means listed first; `icd11_id` NULL means not coded; no row means no condition stated; `confidence` is the stage's measured accuracy (0.800), not per row |
| `trial_drug` (765,483) | a drug component of an arm | which drugs a trial gives and in which role (investigational, comparator, background) | `drug_id` is the moiety; NULL when not linked; placebo and devices give no row |
| `trial_org` (916,656) | an organisation in a role on a trial | sponsors and collaborators; `"primary"` marks one lead sponsor per trial | `resolution_status`: only `resolved` rows name a registered company; `clustered` and `minted` org_ids come from the string, so one sponsor sits under several (Merck Sharp & Dohme: 10 rows) |
| `trial_endpoint` (2.6M) | a registered endpoint | primary and secondary measures and their timepoints | the protocol's endpoints, not results; none for ISRCTN-primary trials |
| `trial_outcome` (491,223) | a trial's result class, dated by its evidence | what the trial found; **the one trial table that takes `known_from <= date`** | 339,834 are `not_reported`; `confidence` is the model's measured accuracy (Opus 5.5 0.882, Sonnet 5.5 0.856); `evidence` spans point into `passage` |
| `trial_strategy` (491,490) | the strategy stage's tags | regulatory_pathway, novelty_angle (text[]), endpoint_type, endpoint_duration_bucket, trial_size_bucket, has_positive_readout | inferences by Kimi K3 (pathway 0.903, angles exact set 0.548): a lead, not a filter of record; NULL is unknown |
| `trial_event` (1.8M) | a dated milestone (registered, first patient, last patient, primary completion, completion, stopped, results posted) | "when did trials start" by year; `actual` true or planned | no status history: `stopped` is derived from a completion date |

## Literature and text

| Table | One row per | Reach for it when | Trap |
|---|---|---|---|
| `paper` (491,466) | a publication a trial of the release lists | pmid, doi, journal, date, `retracted` | by default only papers the trials list (coverage says); check `retracted` |
| `paper_trial` (624,469) | a (paper, trial) link | the papers reporting a trial: `relation = 'exact'` (237,349); protocol (21,603); related (365,517) | `confidence` only on judged links; a declared link has NULL |
| `paper_drug` (342,016) | (paper, moiety) through the trials it reports | a drug's literature, via its trials | comparators and background count too: join `trial_drug.role` |
| `document` (1.0M) | a text: registry record (491,690), abstract (477,812), US label (33,207), EMA medicine record (2,255) | the provenance of a passage; its `url`, `markings` | a new version is a new document |
| `passage` (3.7M) | a section of a document | the text itself; search with `twin_text`; read by `passage_id` | registry record sections: population, arms, strategy/trial, strategy/outcomes, results/<part>; label: indications_and_usage; EMA: therapeutic_indication; abstract: abstract. A whole-table GROUP BY times out |
| `document_link` (632,482) | an entity a document concerns (trial, product, drug) | from a trial or product to its documents | abstracts link through `document.paper_id`, not here |

## Drugs and products

| Table | One row per | Reach for it when | Trap |
|---|---|---|---|
| `drug` (174,945) | a substance of the registry | name, `kind` (small_molecule 131,575, other 32,537, biologic 7,803, combination 3,030), unii, `moiety_id`, code_names, `max_phase` | 17,207 rows are salts or forms (`moiety_id <> drug_id`): roll up to the moiety; `max_phase` NULL on 159,014 |
| `drug_target` (7,708) | an Open Targets mechanism | target, action, Ensembl genes | few drugs covered |
| `product` (403,501) | an authorised product in one territory | brand, form, route, regulator, authorisation_no, approved_on, `marketed`, withdrawn_on | `marketed` NULL where the register does not say; `withdrawn_on` rarely set in the US |
| `product_drug` (662,646) | an ingredient of a product | from a moiety to its products: `WHERE drug_id = <moiety>` | `drug_id` NULL: the ingredient did not resolve |
| `product_holder` (401,566) | the holder of a product | who holds it now; `holder_as_written` when `org_id` is NULL | `valid_from` is a capture day, no history; national holders resolve rarely (FDA 100 %, EC 99 %, others 0 to 50 %) |
| `approved_indication` (5,312) | an indication statement read from a label | the diseases a product is authorised for, with the quote and `passage_id` | **a slice**: 989 US and 51 EU-central products of 403,501 (coverage `complete` false); an absence is never "no indication" |

## Regulatory clocks

| Table | One row per | Reach for it when | Trap |
|---|---|---|---|
| `designation` (3,351) | an EMA orphan (3,305) or PRIME (46) designation | orphan status in the EU | `disease_id` and `sponsor_org_id` NULL on every row; 1,338 rows have no `drug_id`: read `substance_as_written` |
| `regulatory_event` (41,361) | a dated step of an application (FDA approvals and efficacy supplements, EMA opinions and decisions) | when a drug was approved, how many supplements, PDUFA dates (`planned`) | `disease_id` NULL: what a supplement added is not read; `known_from = event_on`, so as-of works |
| `protection` (34,767) | a listed patent, exclusivity or EU protection period | what protects a product and until when | which patent is the compound patent is not stated; biologic patents are not on file |
| `loe` (3,512) | the derived loss of exclusivity per (moiety, US or EU-central) | when generics may come | read `basis` before `expires`; `completeness` says what was not read |

## Organisations

| Table | One row per | Reach for it when | Trap |
|---|---|---|---|
| `org` (309,844) | a company or institution | name, `kind` (industry 99,236, hospital, academic, government, fund, vehicle; NULL 123,945), country, `us_listed`, `size_tier` | one real company is often several rows: `twin_find` all of them, then `ownership_effective` |
| `ownership` (7,956) | a direct stake (holder, company), dated where a source dates it | who owned X in a past year (`valid_from`, `valid_to`, `not_after`) | `pct` NULL everywhere |
| `ownership_effective` (8,246) | a holder up the chain, current state | a group's trials: join `trial_org.org_id` to `org_id`, filter `holder_org_id` | current only; `conflict` true where sources disagree |
| `listing` (1,944) | an EDGAR ticker | exchange, ticker, isin | |
| `filing` (3.3M), `filing_party` (3.4M) | an SEC filing and its filer | 8-K items, 10-K dates | `filer_id` is `cik:<10 digits>`, not an org_id: `external_id` scheme cik maps it; filing text is not held |
| `financial` (0) | a reported value | nothing yet | empty |

## Scouting, views and the empty families

| Table | One row per | Reach for it when | Trap |
|---|---|---|---|
| `program` (106,063) | a (moiety, disease) over the release's trials and indications | a drug's indications landscape: highest_phase, best_outcome, status, n_trials, n_papers, lead_org_id | `approval_known` false means `highest_phase` 'approved' could not be decided (pembrolizumab reads phase_4 in lung cancer): take approval from `territory_status` |
| `territory_status` (149,273) | (moiety, territory, indication, holder) | where a drug is authorised, by whom; absences only where coverage is complete | `disease_id` NULL on 148,262 rows (the drug in any indication); read `basis` |
| `cheap_version` (107,970) | (moiety, territory, form, route, strength) | how many generics are marketed or authorised | `covered` false: zero means nothing; `lowest_price` NULL |
| `opportunity` (0) | a detector's finding | nothing yet | empty |
| `deal`, `deal_party`, `right_grant`, `payment`, `royalty`, `royalty_party`, `price`, `spend`, `royalty_paid`, `org_runway` (0) | | nothing yet | all empty with a 'not read' coverage row: answer "unknown" |

## Meta

| Table | One row per | Reach for it when |
|---|---|---|
| `release_info` (1) | the release served | the release id and `inputs.captured` (each source's capture day) for the "data as of" line |
| `catalog` (58) | a table of this schema | the same text `twin_catalog` serves, queryable: `SELECT table_name, about FROM catalog WHERE table_name = 'loe'` |
