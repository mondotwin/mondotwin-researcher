# The 58 tables of schema v1, one line each

Row counts are this release's (`SELECT row_count FROM catalog`), rounded. Read a table's `about` before the first
query against it: `SELECT about FROM catalog WHERE table_name = '<name>'`, or `twin_catalog(table="<name>")` for the
column comments too. Every table carries `sources` (record keys) and `known_from` (+ `_basis`, `_precision`); they are
not repeated below. **EMPTY** marks a table published with no rows because no source is read yet: an empty result from
it means unknown, never "none".

## Meta and navigation

| Table | One row per | Rows | Use it for | Columns that matter |
|---|---|---|---|---|
| `release_info` | the release served | 1 | the release id to cite; `inputs.captured` gives each source's capture date | `release`, `compiled_at`, `inputs` (jsonb: captured, snapshots, releases, provenance) |
| `catalog` | a table of this schema | 58 | what a table is and how to read it | `table_name`, `grp`, `one_row_per`, `about`, `row_count`, `columns` (jsonb list of {name, type, comment}) |
| `coverage` | (source, entity type, scope) | 58 | whether an absence is evidence | `source`, `entity_type`, `scope`, `complete`, `through_on`, `through_basis` (stated, capture, 'not read') |
| `entity_name` | a name of an entity | 3.1 M | names → ids; indexed on `lower(name)` and by word prefix | `entity_type` (trial, drug, product, org, disease), `entity_id`, `name`, `kind` (title, display, synonym, brand, code, 'as written'), `source` |
| `external_id` | an outside id of an entity | 2.6 M | ids → entities, indexed on (`scheme`, `value`) | `scheme` (nct, eudract, ctis, isrctn, anzctr, umin, chictr, jrct, cris, itmctr; unii, chembl, cas, pubchem, inchikey, rxcui, drugbank; nda, anda, bla, eu_ma, nca:de…; cik, lei; pmid, doi; icd11, orpha, mondo), `value`, `entity_type` (trial, drug, product, org, paper, disease), `entity_id` |
| `source_record` | a raw record a row cites | 2.6 M | which capture a record came from, its licence markings | `record_key`, `source`, `snapshot`, `published_at`, `markings` |
| `trial_provenance` | a trial of the release | 491,690 | every engine step behind a trial (the Trace page) | `trial_id`, `steps` (jsonb), `n_steps`, `other_pairs` |
| `claim` | a fact found on the web | **EMPTY** | — | — |

## Trials

| Table | One row per | Rows | Use it for | Columns that matter |
|---|---|---|---|---|
| `trial` | a study, deduplicated across registries | 491,690 | the trial's current state: phase, status, dates, enrolment, countries, why stopped | `phase` (phase_1, phase_1_2, phase_2, phase_2_3, phase_3, phase_4, NA, NULL), `status` (completed, recruiting, not_recruiting, unknown, terminated, not_yet_recruiting, active_not_recruiting, withdrawn, suspended, NULL), `status_stated`, `registered_on`, `start_date` + `start_date_precision`, `completion_date` + precision, `enrollment_planned` / `enrollment_actual` (never both), `randomised`, `countries` (ISO-2[]), `has_results`, `why_stopped`, `stop_class` (safety, efficacy, regulatory, enrolment, funding, business, other, not_stated), `purpose` (treatment, prevention, diagnostic_validation, supportive_care, basic_science, health_services_implementation, device_performance), `title`, `brief_title` |
| `trial_record` | a registry record of a trial | 539,356 | the members a trial was merged from; which is primary | `record_key` (`clinicaltrials.gov:NCT…`, `ictrp:ChiCTR…`), `registry` (clinicaltrials.gov, eudract, ctis, isrctn, anzctr, umin, chictr, itmctr, jrct, cris), `is_primary` |
| `trial_condition` | an index condition of a trial | 631,814 | trials of a disease; the trial's own wording | `icd11_id` (`icd11:<code>`), `orpha_id`, `raw_text`, `role` (primary = listed first), `method`, `confidence` |
| `trial_drug` | a drug in an arm | 765,483 | trials of a drug; what each arm gives | `drug_id` (moiety; NULL = component not linked), `raw_name`, `arm`, `role` (investigational, comparator, background), `dose`, `route` |
| `trial_org` | an organisation in a role on a trial | 916,656 | sponsors, collaborators, funders | `org_id` (NULL for a person or a placeholder), `raw_name`, `role` (lead sponsor, collaborator, funder), `primary` (one lead per trial), `resolution_status` (resolved, clustered, minted, person, unresolved, 'not a name') |
| `trial_endpoint` | a registered endpoint | 2.55 M | protocol endpoints and timepoints (not results) | `kind` (primary, secondary), `position`, `measure`, `description`, `timepoint` |
| `trial_event` | a dated milestone | 1.82 M | when a trial registered, started, completed, posted results, stopped | `kind` (registered, 'first patient', 'last patient', 'primary completion', completion, 'results posted', stopped), `event_on`, `precision`, `actual` (true actual, false planned), `seen_at` |
| `trial_outcome` | a trial's result class | 491,223 | what a trial found and when it was knowable | `outcome` (positive_significant, positive_trend, descriptive_met, failed, harmful_or_inferior, stopped_operational, inconclusive, not_reported), `primary_met_significant`, `basis` ('posted results', paper, 'registry results', 'registry status', none), `n_randomised`, `n_analysed`, `evidence` (jsonb spans into `passage`), `model`, `confidence`, `known_from` (**the as-of column**) |
| `trial_strategy` | strategy tags of a trial | 491,490 | regulatory pathway, novelty angles, endpoint type and read-out horizon | `regulatory_pathway` (de_novo_nde, 505b2_repurposing, 505b2_reformulation, combination_fdc, prodrug_enantiomer, biologic_biosimilar, none), `novelty_angle` (novel_moa, new_route, new_regimen, new_population; filter `@> ARRAY[...]`), `endpoint_type` (clinical, surrogate_accepted, surrogate_experimental, biomarker, pro, none), `endpoint_duration_bucket` (<=1wk, <=3mo, <=1yr, >1yr), `trial_size_bucket` (<50, <200, <500, >=500), `has_positive_readout` |

## Drugs and products

| Table | One row per | Rows | Use it for | Columns that matter |
|---|---|---|---|---|
| `drug` | a substance of the drug registry | 174,945 | the drug and its moiety; development codes; highest phase | `drug_id`, `name`, `kind` (small_molecule, biologic, combination, other), `unii`, `moiety_id` (join here to roll salts up), `code_names`, `max_phase` (approved, phase_3, phase_2, phase_1, phase_1_2, phase_2_3, early_phase_1, ind, preclinical, preapproval, NULL) |
| `drug_target` | a mechanism Open Targets states | 7,708 (5,930 drugs) | target and action where Open Targets has one | `target`, `genes` (ENSG ids), `action` |
| `product` | an authorised product per regulator | 403,501 | where a drug is authorised, under which brand, since when, marketed or not | `brand`, `form`, `route`, `territory` (ISO-2 or EU-central), `regulator` (FDA, EC, PMDA/MHLW, HC, TFDA, MFDS, ANVISA, Swissmedic, HSA, NMPA, NCA:<cc>), `procedure` (NDA, ANDA, BLA…, 'centralised full|conditional|exceptional', generic, biosimilar, national…), `approved_on`, `withdrawn_on`, `marketed` (true, false, NULL) |
| `product_drug` | an ingredient of a product | 662,646 | a drug's products (join on the moiety) | `drug_id` (NULL = unresolved ingredient), `ingredient_as_written`, `strength`, `strength_basis` |
| `product_holder` | the holder of a product | 401,566 | who holds a product now | `org_id`, `holder_as_written`, `valid_from` (= capture day, **not** the transfer date), `valid_to` (NULL) |
| `approved_indication` | an indication statement on a label | 5,312 | what a US or EU label says it is for, with the quote | `product_id`, `icd11_id`, `population`, `line`, `approved_on`, `evidence` (jsonb: passage_id, quote…) |

## Regulatory

| Table | One row per | Rows | Use it for | Columns that matter |
|---|---|---|---|---|
| `regulatory_event` | a dated step of an application | 41,361 | approval dates, supplements, CHMP opinions, refusals, PDUFA dates | `drug_id`, `product_id`, `regulator` (FDA, EMA, EC), `procedure`, `kind` (approved, 'efficacy supplement approved', opinion, 'tentative approval', withdrawn, 'withdrawn by applicant', refused, validated), `event_on`, `planned`, `applicant_org_id`, `reasons` |
| `protection` | a listed patent, exclusivity or extension | 34,767 | what protects a product and until when | `product_id`, `territory` (US, EU-central), `kind` (patent_substance, patent_product, patent_use, nce, orphan, pediatric, new_indication, new_product, gain, bpcia_reference, bpcia_interchangeable, eu_data, eu_market, eu_orphan, reference_exclusivity, other), `number`, `use_code`, `expires`, `extension`, `delisted` |
| `loe` | loss of exclusivity, derived per (moiety, territory) | 3,512 | when the market sees exclusivity end | `drug_id`, `territory` (US, EU-central), `expires`, `basis` ('generic launch', settlement, 'generic approval', protection, blocked, 'national authorisation first', 'nothing on file'), `set_by`, `completeness` |
| `designation` | an orphan or PRIME designation | 3,351 | EMA orphan and PRIME designations (FDA not read yet) | `drug_id`, `substance_as_written`, `condition_as_written`, `regulator` (EMA), `kind` (orphan, prime), `number`, `status` (granted, withdrawn, expired, refused), `granted_on` |

## Derived views

| Table | One row per | Rows | Use it for | Columns that matter |
|---|---|---|---|---|
| `territory_status` | (moiety, territory, indication, holder) | 149,273 | authorised / not authorised / unknown per territory, with the reason | `drug_id`, `disease_id` (NULL = any indication), `territory`, `status` (authorised, 'licensed out', 'not authorised', 'no disclosed grant', unknown), `holder_org_id`, `basis` |
| `cheap_version` | (moiety, territory, form, route, strength) | 107,970 | how many generics or biosimilars are marketed or authorised | `generic_products` (marketed), `generic_products_authorised`, `covered` (a complete register covers the territory), `lowest_price` (NULL today) |
| `ownership_effective` | (holder, company) up the control chain | 8,246 | roll companies up to their group; who ultimately holds whom | `holder_org_id`, `org_id`, `controls`, `ultimate`, `conflict`, `valid_from`, `not_after` |
| `program` | (moiety, disease) over the release's trials and indications | 106,063 | a drug's indication landscape at a glance | `highest_phase` (approved, phase_4 … phase_1, NA), `approval_known`, `best_outcome`, `primary_met_significant`, `status` (approved, active, completed, stopped, unknown), `lead_org_id`, `n_trials`, `n_papers`, `last_activity` |
| `org_runway` | a group's cash and burn per quarter | **EMPTY** | — | — |
| `royalty_paid` | royalty receipts per period | **EMPTY** | — | — |
| `opportunity` | a detector's finding | **EMPTY** | — | — |

## Organisations

| Table | One row per | Rows | Use it for | Columns that matter |
|---|---|---|---|---|
| `org` | a company or institution | 309,844 | the company behind a sponsor or holder string; one real company may be several rows | `name`, `kind` (industry, hospital, academic, government, fund, vehicle, NULL), `country`, `status` (GLEIF), `us_listed`, `size_tier` (big_pharma, mid_pharma, small_pharma, startup_biotech, NULL) |
| `ownership` | a direct stake (holder, company) | 7,956 | who owns whom, dated where a source dates it | `holder_org_id`, `org_id`, `controls`, `kind` (ownership, division, identity), `method` (sec_exhibit, gleif_consolidation, gleif_branch, stated_parent, web_parent), `valid_from` + basis, `valid_to`, `not_after`, `evidence` |
| `listing` | a security on an exchange | 1,944 | tickers of EDGAR filers | `org_id`, `exchange` (MIC), `ticker`, `valid_from`, `valid_to` |
| `filing` | an SEC filing | 3.29 M | which forms a company filed and when; 8-K items (1.01 agreement, 2.01 acquisition completed, 5.01 change in control); **no text** | `filing_id` (sec:<accession>), `form`, `filed_at`, `period_end`, `items`, `document_id` (NULL) |
| `filing_party` | a company a filing concerns | 3.42 M | join a filing to its filer | `filer_id` (`cik:0000000000`: join `external_id` scheme cik to reach `org`), `role` |
| `financial` | a reported value | **EMPTY** | — | — |

## Deals and market (all EMPTY in this release)

| Table | One row per | Rows | Note |
|---|---|---|---|
| `deal`, `deal_party`, `payment`, `right_grant`, `royalty`, `royalty_party` | agreements, parties, payments, rights, royalties | **EMPTY** | `coverage` holds 'not read' rows; `territory_status` therefore never reads 'licensed out'. Answer "unknown". |
| `price`, `spend` | a price while it holds; public spend per year | **EMPTY** | `cheap_version.lowest_price` is NULL for the same reason. |

## Literature and text

| Table | One row per | Rows | Use it for | Columns that matter |
|---|---|---|---|---|
| `paper` | a publication the release holds | 491,466 | papers linked to the release's trials (not a corpus) | `pmid`, `doi`, `title`, `journal`, `published_on`, `type` (article, conference-abstract, review, preprint, letter…), `retracted`, `why_in_corpus` |
| `paper_trial` | (paper, trial) link | 624,469 | which papers report a trial | `relation` (exact, protocol, related; derived), `found_by` (declared vs judged, and any demotion), `confidence` (judged links only) |
| `paper_drug` | (paper, moiety) via a reported trial | 342,016 | a drug's papers, through its trials | `relation`, `found_by` |
| `document` | a text the twin holds, in versions | 1.0 M | what kind of text exists and where it came from | `kind` ('registry record', abstract, 'US label', 'EMA medicine record', filing), `title`, `published_on`, `url`, `provider`, `paper_id`, `markings` |
| `document_link` | an entity a document concerns | 632,482 | the trial, product or drug a record or label is about (abstracts link via `paper_trial`) | `entity_type` (trial, product, drug), `entity_id`, `method` |
| `passage` | a section of a document | 3.68 M | the text itself: quotes, evidence spans, full-text search through `twin_text` | `passage_id` (`<document_id>/<section>`), `section` (population, arms, results/<part>, strategy/<part>, abstract, indications_and_usage, therapeutic_indication), `text`, `digest`, `markings` |

## Reference

| Table | One row per | Rows | Use it for | Columns that matter |
|---|---|---|---|---|
| `disease` | an ICD-11 category or an Orphanet entity | 27,115 | names and codes of conditions; rarity; hierarchy | `disease_id` (`icd11:<code>`, `orpha:<code>`), `system`, `code`, `name`, `parent_ids`, `ancestor_ids`, `rare`, `status` (active, obsolete, deprecated) |
| `disease_map` | a mapping between vocabularies | 19,885 | carry Orphanet ↔ ICD-11 ↔ Mondo | `from_id`, `to_id`, `relation` (exact, broader, narrower, related), `source` (orphanet, mondo), `mondo_id` |
| `prevalence` | Orphanet epidemiology per (disorder, measure, area) | 17,108 | how rare, how frequent | `measure` (point, birth, 'annual incidence', lifetime, cases, 'class only'), `value` (per 100,000), `low`, `high`, `area` |
| `territory` | a country or a named set | 145 | which countries a set (EU-central, EEA, Europe, world, ex-US) holds on a date | `code`, `name`, `members` (jsonb [{code, from, to}]), `excluded` |
