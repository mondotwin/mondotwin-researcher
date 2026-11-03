# From the MPC Finder warehouse to the twin

For readers of the retired `query-mpc-finder` skill and for porting the analyst skills (`assess-mpc-candidates`,
`design-combination-therapy`, `endpoint-comparator-precedent`, `value-capture-archetypes`): what each warehouse table,
JSONB path and Mongo collection asked, and what answers it in schema v1. "Not in the twin" means exactly that: say
so in an answer, do not approximate.

## Tables and tools

| Warehouse (`execute_sql`, `query_clinical_studies`, `execute_mongo`) | Twin (`twin_sql`, `twin_find`, `twin_text`, `twin_record`) |
|---|---|
| `clinical_studies` (one row per trial under one registry id) | `trial` (one row per deduplicated study, `trial_id` a uuid); its registry records in `trial_record`; registry ids in `external_id` (scheme nct, eudract, ctis, isrctn, anzctr, umin, chictr, itmctr, jrct, cris) |
| `clinical_studies.primary_registry_id` = `NCT…` | `trial_record.record_key` of the `is_primary` row (`clinicaltrials.gov:NCT…`); `twin_find("nct:NCT…")` |
| `clinical_study_disease` (`icd11_code`, `role`, `display_name`, `orphanet_code`, `disease_key`) | `trial_condition` (`icd11_id` = `icd11:<code>`, `role`, `raw_text`, `orpha_id` = `orpha:<n>`, `method`, `confidence`) |
| `clinical_study_drug` (`canonical_drug_id` = DrugBank id or `name::…`, `arm_role`, `component_type`, `is_novel_for_disease`, `route`) | `trial_drug` (`drug_id` = the registry's moiety uuid, `raw_name`, `arm`, `role` investigational / comparator / background, `dose`, `route`); the drug's kind in `drug.kind`; a DrugBank id in `external_id` scheme drugbank where one is known (11,168 drugs) |
| `clinical_study_sponsor` (`normalized_sponsor_name`, `sponsor_class`, `is_lead_pharma`) | `trial_org` (`org_id`, `raw_name`, `role` lead sponsor / collaborator / funder, `"primary"`, `resolution_status`); the class in `org.kind`, the tier in `org.size_tier`, the group in `ownership_effective` |
| `clinical_studies.endpoints[]` | `trial_endpoint` (`kind`, `position`, `measure`, `description`, `timepoint`) |
| `clinical_studies.arms[].components[]` | `trial_drug`, plus the registry's arms text in `passage` (section `arms`) |
| `clinical_studies.population` (criteria, age, sex, line of therapy) | not published as columns (decision P4, 28 Oct 2026); the text in `passage` (section `population`) and in `twin_record` |
| `clinical_studies.classification` (masking, control type) | not published; `trial.randomised`; the rest in `twin_record` |
| `clinical_studies.organization.main_center` (sites, coordinates) | not published; `trial.countries` (ISO codes), `countries_other`; sites in `twin_record` |
| `clinical_studies.results` (`clinical_outcome`, `positive_results`, `why_stopped`, `results_available`) | `trial_outcome` (`outcome` in eight classes, `basis`, `known_from`, `confidence`, `evidence`; `legacy_label` maps to the old five); `trial.why_stopped`, `stop_class`; `trial.has_results` |
| `clinical_studies.strategy_tags` | `trial_strategy` (same fields: `regulatory_pathway`, `novelty_angle`, `endpoint_type`, `endpoint_duration_bucket`, `trial_size_bucket`, `has_positive_readout`; `lead_sponsor_tier` is `org.size_tier` through `trial_org`) |
| `clinical_studies.prior_art_flags` (`all_drugs_novel`, `any_drug_approved`, `regimen_fully_approved`) | `program` (`highest_phase`, `approval_known`) and `territory_status` (`status = 'authorised'`) per moiety; no regimen-level flag |
| `clinical_studies.patents`, `ip_flag`, `covers_therapy`, `mpc_validity.patent_gate`, `therapeutic_opportunities.patents[]` | **not in the twin.** `protection` (Orange Book and Purple Book listings, EU protection periods) and `loe` are per product and moiety; no method-of-use search per trial and no freedom-to-operate verdict. The platform's agents keep their patent answers in its own `trial_annotation` table |
| `clinical_studies.ownership_assessment` | `ownership`, `ownership_effective` (from 10-K exhibits, GLEIF, stated parents, cited web answers) |
| `clinical_studies.linkouts` (`pubmed_ids`, `openalex_publications`, `registry_url`) | `paper`, `paper_trial` (`relation`), `document` (kind abstract) with the abstract in `passage`; the registry URL in `document.url` of the record's document, or built from the record key |
| `therapeutic_opportunities` (regimen × indication, `highest_phase` UPPER, `evidence_count`, `has_positive_readout`, `industry_pickup`, `sponsor_class_transition`, `lead_sponsor_tier`) | `program` is (moiety × disease): `highest_phase` lower-case, `best_outcome`, `n_trials`, `n_papers`, `status`, `lead_org_id`. A **regimen** view is not materialised: group `trial_drug` per trial yourself. Pickup and transition signals are not computed |
| `mpc_candidates` (`buckets`, `priority_score`, `time_to_approval`, `rationale`) | **not in the twin**; the candidate machinery was deleted on 28 Oct 2026. `opportunity` is the table a detector will fill; empty today |
| `clinical_trial_predictions` (`signal0_prob_cal`, `signal1_prob_cal`) | **not in the twin**: a platform product table |
| `drug_intelligence` (`n_indications_tested`, `n_indications_approved`, `patent_status`, `is_generic_available`, routes) | `program` per moiety (tested), `territory_status` and `approved_indication` (approved, a slice), `loe` and `protection` (patent status, per product), `cheap_version` (generics), `product.route` |
| `disease_intelligence` (`n_approved_drugs`, pipelines, `disease_rarity_tier`, `has_orphan_potential`) | `program` grouped by `disease_id`; `disease.rare`; `prevalence`; `designation` (EMA orphan) |
| `sponsor_intelligence` (`tier`) | `org.size_tier`, `org.kind`, `org.us_listed`, `listing` |
| Mongo `clinicaltrialgov` (API v2 document) | `twin_record("clinicaltrials.gov:NCT…")`: the same JSON shape (`protocolSection`, `derivedSection`, `hasResults`) |
| Mongo `unique_trials` (dedup clusters) | `trial_record` (members of a `trial_id`), `trial_provenance` (the dedup decisions as steps) |
| Mongo `eudract`, `ctis`, `umin`, `anzctr`, `isrctn` | `twin_record` by the record key of that registry (`trial_record.record_key`) |
| Mongo `drugbank` | **not in the twin** (licence); `drug`, `drug_target` (Open Targets), `external_id` scheme drugbank |
| Mongo `orphanet_codes`, `orphanet_stats` | `disease` (system orphanet, `status`, `rare`), `disease_map`, `prevalence` |
| Mongo `icd11_entities`, `icd_map` | `disease` (system icd11, `parent_ids`, `ancestor_ids`), `disease_map` (Orphanet and Mondo mappings; no ICD-10) |
| Mongo `fda_guides`, `ich_guides`, `ectd_*`, `odd_section_templates` | not in the twin; the guideline MCP tools (`search_fda_guidelines`, `search_ich_guidelines`, `get_*_guideline_full`) stay |
| analyst tools (`create_shortlist`, `set_study_state`, `write_study_note`, …) | unchanged; outside this skill. A shortlist item is a twin `trial_id` |

## Values

| Warehouse | Twin |
|---|---|
| `phase` `phase_1`…`phase_4`, `NA`; TO `PHASE3` | `trial.phase` `phase_1`, `phase_1_2`, `phase_2`, `phase_2_3`, `phase_3`, `phase_4`, `NA`, NULL (a stated phase no rule reads); `program.highest_phase` the same words plus `approved` |
| `status` eight values | the same eight plus `not_recruiting` (ICTRP's fold) and NULL; the registry's words in `status_stated` |
| `clinical_outcome` `positive_result`, `positive_signal`, `negative_signal`, `negative_result`, `'none'` | `trial_outcome.outcome` `positive_significant`, `positive_trend`, `descriptive_met`, `failed`, `harmful_or_inferior`, `stopped_operational`, `inconclusive`, `not_reported`; `legacy_label` holds the old word (`failed` → `negative_signal`: do not read it as weak) |
| `arm_role` `experimental`, `active_comparator`, `placebo`, `soc`, … | `trial_drug.role` `investigational`, `comparator`, `background` (`drug_role@1`) |
| `sponsor_class` `industry`, `academic`, `government`, `mixed`, `unknown` | `org.kind` `industry`, `hospital`, `academic`, `government`, `fund`, `vehicle`, NULL, through `trial_org` |
| `lead_sponsor_tier` `big_pharma`, `mid_pharma`, … | `org.size_tier` (same words where assessed; NULL otherwise) |
| `ip_flag` `iplock_originator`, `covered_novel`, `fto_off_patent` | none. `loe.basis` and `protection.kind` are the nearest facts, per product |
| `is_novel_for_disease` | `trial_drug.role = 'investigational'` for the test article; novelty for the indication is `program.highest_phase` and `territory_status` of that drug in that disease |
| ICD-11 `LIKE '8B88%'`, Orphanet `disease_key = 'ORPHA:2073'` | `icd11_id LIKE 'icd11:8B88%'` or `disease.ancestor_ids @> ARRAY['icd11:8B88']`; `orpha_id = 'orpha:2073'` |

## Rules that carry over, and the ones that do not

Carry over: count `DISTINCT` trials across joined tables; filter diseases by code prefix; keep `role = 'primary'`
only when the question is about the listed-first condition; group drugs by id, not display name; filter on an
indexed column first; never `SELECT *` from a wide table; a registry status is a snapshot, confirm in the record.

Gone with the warehouse: `covers_therapy` and its dirty values, `canonical_drug_id LIKE 'DB%'`, the `name::` ids,
`age_min_years` in JSONB, the dead `has_fda_approval` and `phase_progression`, `ownership_assessment` as evidence,
bucket rules, `*_prob_cal`, Mongo index rules, the prefix stripping between warehouse and Mongo ids.

New in the twin and without a warehouse equivalent: `coverage`, `known_from` and the as-of rules, `confidence` as
a method's measured accuracy, `sources` record keys on every row, `trial_provenance`, `territory_status`, `loe`,
`cheap_version`, `regulatory_event`, `designation`, `ownership_effective`, `filing`, `trial_event`, `passage` with
`twin_text`.
