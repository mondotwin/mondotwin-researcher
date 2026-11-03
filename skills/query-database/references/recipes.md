# Recipes

Every query below ran against the served release on 3 Nov 2026 (`twin@2026-11-02T23:56:16+00:00`, twin
`h1:5425f72a…`) through `twin_sql`; the line after each gives what came back, so the shape can be checked against a
later release. The worked examples are pembrolizumab (`drug_id b4348b10-04cd-59ab-81ea-336b87772b9c`), idiopathic
pulmonary fibrosis (`icd11:CB03.4`, `orpha:2032`), Merck & Co. (`org_id a3dd7332-78cd-56e8-af14-22bb3ab96d47`,
`cik:0000310158`) and the trial NCT02484677 (`trial_id 62af9237-ac09-4c5e-909f-1d18e78cf2fd`). Replace them.

Contents: A. resolve · B. one trial · C. one drug · D. one disease · E. one company · F. outcomes and stops ·
G. strategy tags · H. text · I. release and coverage · J. territories · K. screens for scouting.

## A. Resolve names and ids

### A1. A drug

`twin_find("pembrolizumab")` → the drug `b4348b10-…` (`via: name`, kind synonym), then prefix hits: another drug
`89ZR-PEMBROLIZUMAB`, trials titled "Pembrolizumab in HCC", "Neoadjuvant Pembrolizumab"… Keep the `via: name` hit of
`entity_type: drug`, then:

```sql
SELECT drug_id, name, kind, unii, moiety_id, code_names, max_phase
FROM drug WHERE drug_id = 'b4348b10-04cd-59ab-81ea-336b87772b9c';
```

If `moiety_id <> drug_id`, the hit is a salt or a form: use `moiety_id` in every other table.

### A2. A disease

`twin_find("idiopathic pulmonary fibrosis")` → `icd11:CB03.4` (kind `display`), `orpha:2032` (`display`), and as
synonyms `icd11:CB03` (the parent, "Idiopathic interstitial pneumonitis"), `icd11:CB03.Y`, `icd11:CB03.Z`. Read them:

```sql
SELECT disease_id, system, code, name, rare, status, parent_ids
FROM disease WHERE disease_id IN ('icd11:CB03.4', 'icd11:CB03', 'orpha:2032');
-- CB03.4 rare true; CB03 "Idiopathic interstitial pneumonitis" rare false; orpha:2032 rare true, active
```

The subtree of a code: `SELECT disease_id, name FROM disease WHERE ancestor_ids @> ARRAY['icd11:CB03']` (9 rows,
CB03.0 to CB03.Z). Keep the display code and its Orphanet twin; widen to the parent only on purpose.

### A3. A company

`twin_find("Merck Sharp & Dohme")` → ten `org` rows (`Merck Sharp & Dohme Corp.`, `MERCK SHARP & DOHME LLC`,
`MERCK SHARP & DOHME B.V.`, spellings and affiliates). Roll them up:

```sql
SELECT o.org_id, o.name, o.kind, o.country,
       (SELECT count(*) FROM trial_org tr WHERE tr.org_id = o.org_id AND tr.role = 'lead sponsor' AND tr."primary") AS lead_trials,
       (SELECT string_agg(h.name || CASE WHEN e.ultimate THEN ' (ultimate)' ELSE '' END, '; ')
          FROM ownership_effective e JOIN org h ON h.org_id = e.holder_org_id WHERE e.org_id = o.org_id) AS holders
FROM org o WHERE o.org_id IN ('698c4092-fadd-5a75-8f34-5cfefc6ccbe3', 'bf0b4509-2720-5cae-93a3-dc80b5740a51',
                              '0143eae0-9ebd-5f35-b8db-c1f8ef19c56c', '5ce9839d-b3cf-50f0-bea3-5866412c6adc');
-- LLC 1,485 lead trials and Corp. 98, both under Merck & Co., Inc. (ultimate); B.V. 0 and no holder (1.8 s)
```

An outside id goes straight to the entity: `twin_find("cik:0000310158")` → org `a3dd7332-…` (Merck & Co., Inc.,
`kind industry`, `us_listed` true, `size_tier big_pharma`), or in SQL:

```sql
SELECT e.entity_type, e.entity_id, o.name, o.size_tier
FROM external_id e JOIN org o ON o.org_id = e.entity_id
WHERE e.scheme = 'cik' AND e.value = '0000310158';
```

A brand finds the drug and its products at once: `twin_find("Keytruda")` → the drug (`kind brand`) and one
`product` row per territory (`ANVISA/100290196/-`, `EC/EU/1/15/1024/-`, `FDA/BLA125514/001`, `HC/02441152/-`, …).
A disease id works as `scheme:value` too: `twin_find("icd11:CB03.4")`, `twin_find("orpha:2032")`.

### A4. A trial by registry id

`twin_find("nct:NCT02484677")` → trial `62af9237-ac09-4c5e-909f-1d18e78cf2fd` (`via: external_id`). The same in SQL:
`SELECT entity_id FROM external_id WHERE scheme = 'nct' AND value = 'NCT02484677'`. Schemes a trial can carry:
nct, eudract, ctis, isrctn, anzctr, umin, chictr, itmctr, jrct, cris. The registry id to cite beside a `trial_id`,
the NCT first where the trial has one:

```sql
SELECT t.trial_id, x.value AS registry_id, t.title, t.phase, t.status
FROM trial t
LEFT JOIN LATERAL (SELECT e.value FROM external_id e WHERE e.entity_type = 'trial' AND e.entity_id = t.trial_id
                   ORDER BY e.scheme <> 'nct', e.scheme, e.value LIMIT 1) x ON true
WHERE t.trial_id IN (...);
```

## B. One trial

### B1. The one-pager

```sql
SELECT 'trial' AS what, title AS a, phase AS b, status AS c, registered_on::text AS d FROM trial WHERE trial_id = :t
UNION ALL SELECT 'record', record_key, registry, is_primary::text, NULL FROM trial_record WHERE trial_id = :t
UNION ALL SELECT 'condition', raw_text, icd11_id, role, method FROM trial_condition WHERE trial_id = :t
UNION ALL SELECT 'drug', raw_name, drug_id, role, arm FROM trial_drug WHERE trial_id = :t
UNION ALL SELECT 'org', raw_name, org_id, role, resolution_status FROM trial_org WHERE trial_id = :t
UNION ALL SELECT 'event', kind, event_on::text, actual::text, method FROM trial_event WHERE trial_id = :t
UNION ALL SELECT 'outcome', outcome, basis, known_from::text, model FROM trial_outcome WHERE trial_id = :t
UNION ALL SELECT 'paper', p.pmid, pt.relation, pt.found_by, p.title
           FROM paper_trial pt JOIN paper p USING (paper_id) WHERE pt.trial_id = :t;
```

NCT02484677 (0.06 s): a phase 4, completed, single record `clinicaltrials.gov:NCT02484677`; condition "squamous cell
carcinoma of the head and neck" → `icd11:2B6E.0` by `model_pick@1`; fluorouracil investigational, cisplatin and
docetaxel background; lead sponsor Assistance Publique Hopitaux De Marseille (`clustered`); events registered
2015-06-30, first patient 2015-07-15, primary completion 2017-01-25, completion 2023-05-25, all actual; outcome
`not_reported`, basis `none`; no paper.

Then the scalar row itself: `SELECT * FROM trial WHERE trial_id = :t` is safe (no wide columns): title,
brief_title, phase, status, status_stated, registered_on, start_date (+precision), completion_date (+precision),
enrollment_planned, enrollment_actual (12), randomised, countries (`{FR}`), has_results, why_stopped, stop_class,
purpose, sources, known_from.

### B2. Endpoints

```sql
SELECT kind, position, measure, description, timepoint
FROM trial_endpoint WHERE trial_id = :t ORDER BY kind, position;
-- two primary endpoints, timepoint '24 months'; ISRCTN-primary trials have no rows
```

### B3. Strategy tags

```sql
SELECT regulatory_pathway, novelty_angle, endpoint_type, endpoint_duration_bucket, trial_size_bucket,
       has_positive_readout, rules, model
FROM trial_strategy WHERE trial_id = :t;
-- combination_fdc, {new_regimen}, biomarker, >1yr, <50, NULL, {"endpoint_duration_bucket": "endpoint_duration_parsed@2"}, bedrock/us.moonshotai.kimi-k3
```

### B4. The raw record and the text the models read

`twin_record("clinicaltrials.gov:NCT02484677")` → 9,773 characters of ClinicalTrials.gov's JSON (`protocolSection`,
`derivedSection`, `hasResults`), capture `ctgov@2026-10-23`. The rendered sections:

```sql
SELECT d.kind, p.section, length(p.text) AS chars, p.passage_id
FROM document_link dl JOIN document d USING (document_id) JOIN passage p USING (document_id)
WHERE dl.entity_type = 'trial' AND dl.entity_id = :t ORDER BY p.section;
-- registry record: arms (1,845 chars), population (4,187), strategy/outcomes (339), strategy/trial (2,810)
```

## C. One drug

### C1. Trials by phase and role

```sql
SELECT t.phase, td.role, count(DISTINCT t.trial_id) AS trials
FROM trial_drug td JOIN trial t USING (trial_id)
WHERE td.drug_id = :moiety GROUP BY 1, 2 ORDER BY 1, 2;
-- pembrolizumab (2.6 s): phase_2 investigational 919, background 398; phase_3 investigational 163, comparator 74,
-- background 139; phase_1 background 404 vs investigational 214; NULL phase 40 trials
```

Add `AND td.role = 'investigational'` for the trials that test it; add a condition filter (D1) for one disease.

### C2. The indications landscape

```sql
SELECT pg.disease_id, dd.name, pg.highest_phase, pg.approval_known, pg.best_outcome, pg.status,
       pg.n_trials, pg.n_papers, pg.last_activity, o.name AS lead
FROM program pg LEFT JOIN disease dd ON dd.disease_id = pg.disease_id LEFT JOIN org o ON o.org_id = pg.lead_org_id
WHERE pg.drug_id = :moiety ORDER BY pg.n_trials DESC LIMIT 25;
-- pembrolizumab (0.2 s): lung 223 trials (phase_4, approval_known false, positive_significant, active),
-- breast 149, melanoma 113, head and neck 104, 2D4Z unspecified site 68, ... lead MERCK SHARP & DOHME LLC
```

`approval_known` false on every row here: the drug's labels were not in the indication slice, so `highest_phase`
never reads `approved`. Approval is C4.

### C3. Products, by territory, with the holder

```sql
SELECT p.territory, p.regulator, p.brand, p.form, p.route, p.procedure, p.approved_on, p.marketed, p.withdrawn_on,
       ph.holder_as_written, o.name AS holder, p.sources
FROM product_drug pd JOIN product p USING (product_id)
LEFT JOIN product_holder ph USING (product_id) LEFT JOIN org o ON o.org_id = ph.org_id
WHERE pd.drug_id = :moiety ORDER BY p.territory, p.approved_on;
-- 28 products (0.1 s): US KEYTRUDA BLA 351(a) 2014-09-04 and KEYTRUDA QLEX 2025-09-19 (holder resolved);
-- EU-central Keytruda 2015-07-17; BR, CA, CH, JP, KR, SG, TW with holder_as_written only; IS, LI, NO undated;
-- CN a 'web evidence' row with brand NULL and holder Astellas Pharma Inc.: read basis and sources before citing
```

### C4. Territory status

```sql
SELECT s.territory, s.status, s.disease_id, d.name AS disease, o.name AS holder, s.basis, s.sources
FROM territory_status s LEFT JOIN disease d ON d.disease_id = s.disease_id LEFT JOIN org o ON o.org_id = s.holder_org_id
WHERE s.drug_id = :moiety ORDER BY s.territory, s.disease_id NULLS FIRST;
-- pembrolizumab: 15 rows, all authorised, all disease_id NULL (the drug in any indication); BR has one row
-- 'product, not marketed' beside 'product'; holder NULL on BR, CA, CH, KR, SG, TW (unresolved national names)
```

A territory with no row is not "not authorised": `coverage` (I2) says whether a complete product register was
read for it (US, EU-central, BR, CA, TW complete; JP, KR, CH, SG, CN, EEA-national incomplete).

### C5. What the labels say (and why nothing may come back)

```sql
SELECT p.territory, p.brand, ai.icd11_id, dis.name AS disease, ai.population, ai.line, ai.approved_on,
       ai.evidence->>'dated_by' AS dated_by, ai.evidence->>'quote' AS quote, ai.evidence->>'passage_id' AS passage_id
FROM approved_indication ai JOIN product p USING (product_id) LEFT JOIN disease dis ON dis.disease_id = ai.icd11_id
WHERE ai.product_id IN (SELECT product_id FROM product_drug WHERE drug_id = :moiety)
ORDER BY p.territory, ai.approved_on;
-- pembrolizumab: 0 rows. coverage: approved_indication US and EU-central complete = false (a slice of the labels).
-- Answer: "indications not read for this drug", never "no indication".
```

The rows that exist look like this (EU-central, newest first): Kayshild (semaglutide) `icd11:DB92.1`
non-alcoholic steatohepatitis, population "adults; stage: moderate to advanced liver fibrosis…", approved
2026-03-26, `dated_by authorisation`, quote and `passage_id …/therapeutic_indication`.

### C6. The regulatory timeline

```sql
SELECT re.regulator, re.procedure, re.kind AS step, re.event_on, re.planned, re.product_id IS NOT NULL AS has_product,
       o.name AS applicant, re.sources
FROM regulatory_event re LEFT JOIN org o ON o.org_id = re.applicant_org_id
WHERE re.drug_id = :moiety ORDER BY re.event_on;
-- pembrolizumab (0.1 s): FDA BLA125514 approved 2014-09-04; EC EMEA/H/C/003820 opinion 2015-05-20, approved
-- 2015-07-17; then 60+ 'efficacy supplement approved' rows (S-004 … S-096), product_id NULL, disease_id NULL
```

`kind` is a column of `org` too: alias it (`re.kind AS step`). As-of works here (`known_from = event_on`).

### C7. Protection and loss of exclusivity

```sql
SELECT pr.territory, pr.kind, pr.number, pr.use_code, pr.pediatric, pr.expires, pr.how, pr.extension, p.brand
FROM protection pr JOIN product p USING (product_id)
WHERE pr.product_id IN (SELECT product_id FROM product_drug WHERE drug_id = :moiety) ORDER BY pr.expires;
-- Keytruda: eu_data 2023-07-17, eu_market 2025-07-17 (eu_protection@1:regime=2004), bpcia_reference 2026-09-04,
-- orphan 2031-01-25 (stated), QLEX bpcia_reference 2037-09-19

SELECT territory, expires, basis, set_by, completeness FROM loe WHERE drug_id = :moiety;
-- US 2026-09-04 basis protection (BPCIA twelve years; biologic patents not on file; no generic approved);
-- EU-central 2025-07-17 basis protection (8+2; SPCs and patents not on file; 6 national products not read)
```

### C8. Cheap versions

```sql
SELECT territory, form, route, strength, generic_products, generic_products_authorised, covered
FROM cheap_version WHERE drug_id = :moiety ORDER BY territory, generic_products DESC;
-- the US leaders, for scale: hydrochlorothiazide 12.5MG tablet 138 marketed of 289 authorised (3.7 s for the top-5 scan)
```

## D. One disease

### D1. Trials of a disease and its subtypes, by phase

```sql
SELECT t.phase, count(DISTINCT t.trial_id) AS trials
FROM trial_condition c JOIN trial t USING (trial_id)
WHERE (c.icd11_id LIKE 'icd11:CB03%' OR c.orpha_id = 'orpha:2032')
GROUP BY 1 ORDER BY 2 DESC;
-- IPF and its parent's subtree (1.0 s): phase_2 187, NA 183, NULL 126, phase_3 72, phase_1 60, phase_4 25, ...
```

The subtree through the hierarchy (preferred when the code has children with their own codes):

```sql
SELECT count(DISTINCT c.trial_id) AS trials
FROM trial_condition c JOIN disease d ON d.disease_id = c.icd11_id
WHERE d.disease_id = 'icd11:2C25' OR d.ancestor_ids @> ARRAY['icd11:2C25'];
-- lung cancer and subtypes: 14,350 trials (4.9 s)
```

### D2. What is tested, ranked

```sql
SELECT d.name, td.drug_id, count(DISTINCT t.trial_id) AS trials, array_agg(DISTINCT t.phase) AS phases
FROM trial_condition c JOIN trial t USING (trial_id) JOIN trial_drug td USING (trial_id) JOIN drug d ON d.drug_id = td.drug_id
WHERE (c.icd11_id LIKE 'icd11:CB03%' OR c.orpha_id = 'orpha:2032') AND td.role = 'investigational'
GROUP BY 1, 2 ORDER BY 3 DESC LIMIT 25;
-- nintedanib 29, pirfenidone 24, acetylcysteine 10, bosentan 8, sildenafil 8, rituximab 7, ... (1.0 s)
```

### D3. Trials started per year

```sql
SELECT extract(year FROM e.event_on)::int AS year, count(DISTINCT e.trial_id) AS trials_started
FROM trial_event e JOIN trial_condition c USING (trial_id)
WHERE e.kind = 'first patient' AND c.icd11_id LIKE 'icd11:CB03.4%'
  AND e.event_on BETWEEN '2010-01-01' AND '2026-12-31'
GROUP BY 1 ORDER BY 1;
-- IPF (1.1 s): 13 in 2010 … 41 in 2020, 43 in 2023, 50 in 2026 (planned starts included: add e.actual for actual)
```

### D4. Prevalence and the Orphanet ↔ ICD-11 map

```sql
SELECT measure, value, low, high, area, year, source FROM prevalence WHERE disease_id = 'orpha:2032';
-- point 16.125 per 100,000 world; CA 29.8, KR 37, US 6.7, GB 11.6, FI 8.6 …; source '34233665[PMID]'

SELECT from_id, to_id, relation, source FROM disease_map WHERE from_id = 'orpha:2032' OR to_id = 'orpha:2032';
-- orpha:2032 → icd11:CB03.4 exact (orphanet and mondo); mondo:0800504 → orpha:2032 exact
```

## E. One company or group

### E1. Trials a company leads

```sql
SELECT count(DISTINCT tr.trial_id) AS trials
FROM trial_org tr WHERE tr.org_id = :org AND tr.role = 'lead sponsor' AND tr."primary";
```

`"primary"` needs its quotes (a keyword). Without the flag a trial with several records counts once per lead
named.

### E2. A group's trials through the ownership chain

```sql
WITH grp AS (SELECT org_id FROM ownership_effective WHERE holder_org_id = :parent UNION SELECT :parent)
SELECT count(DISTINCT tr.trial_id) AS trials, count(DISTINCT tr.org_id) AS companies
FROM trial_org tr JOIN grp USING (org_id) WHERE tr.role = 'lead sponsor';
-- Merck & Co., Inc.: 2,030 trials over 45 companies (0.1 s). Current chain only: for a past year read ownership
```

### E3. Filings

```sql
SELECT f.form, f.filed_at, f.items, f.filing_id
FROM filing f JOIN filing_party fp USING (filing_id)
WHERE fp.filer_id = 'cik:0000310158' AND f.form = '8-K' ORDER BY f.filed_at DESC LIMIT 20;
-- 8-K 2026-10-13 item 5.02; 2026-08-04 items 2.02, 9.01; … (4.2 s). filer_id is the cik key, not an org_id
```

## F. Outcomes and stops

### F1. The outcome distribution of a set

```sql
SELECT o.outcome, o.basis, count(*) FROM trial_outcome o GROUP BY 1, 2 ORDER BY 3 DESC;
-- release-wide (0.8 s): not_reported/none 339,834; positive_significant/paper 28,366; failed/paper 21,341;
-- stopped_operational/registry status 21,293; descriptive_met/paper 18,402; positive_trend/paper 16,160;
-- positive_trend/posted results 6,629; failed/posted results 5,965; positive_significant/posted results 5,412 …
```

Add `JOIN trial_condition c USING (trial_id) WHERE c.icd11_id LIKE …` for a disease. Cite `basis` with the class.

### F2. Phase 3 trials stopped for efficacy in a disease, with the registry's words

```sql
SELECT t.trial_id, t.title, t.why_stopped, t.stop_class, t.stop_method, o.outcome, o.basis
FROM trial t LEFT JOIN trial_outcome o USING (trial_id)
WHERE t.phase = 'phase_3' AND t.stop_class = 'efficacy'
  AND EXISTS (SELECT 1 FROM trial_condition c WHERE c.trial_id = t.trial_id AND c.icd11_id LIKE 'icd11:2C25%')
LIMIT 25;
-- lung cancer (0.2 s): "terminated at pre-planned futility analysis", "lack of efficacy and not driven by safety
-- concerns" …; stop_method rule@3 reads negation. Quote why_stopped: the class is a keyword rule
```

### F3. What was known of a result on a date

```sql
SELECT outcome, basis, known_from, confidence, model FROM trial_outcome
WHERE trial_id = :t AND known_from <= '2021-06-01' ORDER BY known_from DESC LIMIT 1;
```

The `evidence` column points into `passage`: `{"passage_id": "6f5a46a3-…/results/primary_results", "start": 1397,
"end": 1412, "quote": "P-value: 0.0165"}`; `substr(text, start + 1, end - start)` on that passage returns the quote.

### F4. Why stopped trials stopped

```sql
SELECT stop_class, count(*) FROM trial
WHERE status IN ('terminated', 'withdrawn', 'suspended') GROUP BY 1 ORDER BY 2 DESC;
-- not_stated 12,904; enrolment 11,302; other 8,367; business 4,195; funding 2,801; efficacy 1,788; safety 804;
-- regulatory 174 (0.1 s)
```

## G. Strategy tags

```sql
SELECT regulatory_pathway, count(*) FROM trial_strategy GROUP BY 1 ORDER BY 2 DESC;
-- none 239,878; 505b2_repurposing 96,360; de_novo_nde 95,290; combination_fdc 44,145; 505b2_reformulation 11,380;
-- NULL 3,697; biologic_biosimilar 660; prodrug_enantiomer 80

SELECT t.trial_id, t.title, t.phase, t.status, s.novelty_angle
FROM trial t JOIN trial_strategy s USING (trial_id)
WHERE t.phase = 'phase_2' AND s.regulatory_pathway = '505b2_repurposing' AND s.novelty_angle @> ARRAY['new_population']
LIMIT 25;
```

Label the result "inferred by the strategy model (Kimi K3; pathway 0.903, angles exact set 0.548 on the test gold)".

## H. Text

### H1. Search

`twin_text('pirfenidone "idiopathic pulmonary fibrosis" combination', limit=10)` → passages of registry records
(`section strategy/trial`) from ChiCTR, ClinicalTrials.gov and UMIN, each with `links` to its trial, its `url`,
its `markings` (`licence:ictrp`, `licence:ctgov-terms`, `licence:umin`, `pii`). Follow `links[].entity_id` into
the trial recipes.

### H2. The sections a document kind has

Never `GROUP BY section` over `passage` (3.7M rows, times out). Read one document of the kind:

```sql
SELECT d.kind, d.title, p.section, length(p.text) AS chars, p.markings
FROM passage p JOIN document d USING (document_id)
WHERE p.document_id = (SELECT document_id FROM document WHERE kind = 'US label' LIMIT 1);
-- US label: indications_and_usage (licence:CC0-1.0); abstract: abstract; EMA medicine record:
-- therapeutic_indication; registry record: population, arms, strategy/trial, strategy/outcomes, results/<part>
```

### H3. Read a passage

```sql
SELECT passage_id, section, text, markings FROM passage WHERE passage_id = :passage_id;
```

### H4. Papers reporting a trial, and a drug's papers

```sql
SELECT p.pmid, p.doi, p.title, p.journal, p.published_on, p.retracted, pt.relation, pt.found_by, pt.confidence
FROM paper_trial pt JOIN paper p USING (paper_id)
WHERE pt.trial_id = :t AND pt.relation IN ('exact', 'protocol') ORDER BY p.published_on;

SELECT pd.relation, count(DISTINCT pd.paper_id) AS papers FROM paper_drug pd WHERE pd.drug_id = :moiety GROUP BY 1;
```

Release-wide: `exact` 237,349, `protocol` 21,603, `related` 365,517; `confidence` set on 140 judged links only.

## I. Release and coverage

### I1. The release

```sql
SELECT release, compiled_at, inputs->'releases' AS domain_releases, inputs->'captured' AS captured FROM release_info;
-- twin@2026-11-02T23:56:16+00:00; captured: clinicaltrials.gov 2026-10-24, ctis 2026-10-25, eudract 2026-10-13,
-- isrctn 2026-10-26, umin 2026-10-24, ictrp 2026-10-20, anzctr 2026-10-04, pubmed 2026-10-25, openalex 2026-10-16,
-- drugsfda 2026-10-23, ema_medicines 2026-10-24, orange_book 2026-10-20, sec_submissions 2026-10-24, …
```

### I2. Coverage

```sql
SELECT source, entity_type, scope, complete, through_on, through_basis FROM coverage ORDER BY entity_type, source;
-- 58 rows. product: US (drugsfda, purple_book), EU-central (ema_medicines), BR (anvisa), CA (hc_dpd), TW (tfda)
-- complete; JP, KR, CH, SG, CN, EEA-national incomplete. approved_indication US and EU-central: complete false.
-- trial: pharma 'all' complete false (the release's slice); each registry's 'trial record' row complete true.
-- designation EU-central complete; protection US (orange_book) complete; deal, payment, royalty, right_grant,
-- financial, price, spend, claim: source NULL, through_basis 'not read'
```

### I3. The catalog as a table

```sql
SELECT table_name, grp, row_count, one_row_per FROM catalog ORDER BY grp, table_name;
SELECT about, columns FROM catalog WHERE table_name = 'territory_status';
```

## J. Territories

```sql
SELECT t.code, t.name, jsonb_array_length(t.members) AS n_members,
       (SELECT m FROM jsonb_array_elements(t.members) m WHERE m->>'code' = 'GB') AS gb
FROM territory t WHERE t.members IS NOT NULL;
-- EU-central 32 members, GB {from 1994-01-01, to 2021-01-01}; EEA 31; EU 28; europe 47; ex-US, rest-of-world,
-- north-america
```

A country inside a set on a day:

```sql
... WHERE EXISTS (SELECT 1 FROM territory t, jsonb_array_elements(t.members) m
                  WHERE t.code = s.territory AND m->>'code' = 'DE'
                    AND (m->>'from' IS NULL OR m->>'from' <= '2024-06-01')
                    AND (m->>'to' IS NULL OR m->>'to' > '2024-06-01'))
```

## K. Screens for scouting

What the retired warehouse's opportunity and intelligence tables answered, as queries over `program` and the
trial tables (`opportunity` is empty). Each label on the result is the twin's, not the registry's: say so.

### K1. A combination tested as such (both drugs investigational in one trial)

```sql
SELECT t.trial_id, t.title, t.phase, t.status
FROM trial t
WHERE EXISTS (SELECT 1 FROM trial_drug a WHERE a.trial_id = t.trial_id AND a.drug_id = :drug_a AND a.role = 'investigational')
  AND EXISTS (SELECT 1 FROM trial_drug b WHERE b.trial_id = t.trial_id AND b.drug_id = :drug_b AND b.role = 'investigational');
-- pirfenidone + nintedanib: 7 trials (NCT03939520, NCT02606877, NCT07141810, two ChiCTR, two UMIN) (0.2 s)
```

`program` is one moiety × one disease: a regimen has no row there.

### K2. Off-patent drugs tested in a disease (repurposing candidates)

```sql
SELECT g.name, p.disease_id, p.highest_phase, p.best_outcome, p.n_trials,
       (SELECT l.basis FROM loe l WHERE l.drug_id = p.drug_id AND l.territory = 'US') AS us_loe_basis,
       (SELECT max(v.generic_products) FROM cheap_version v WHERE v.drug_id = p.drug_id AND v.territory = 'US') AS us_generics_marketed
FROM program p JOIN drug g ON g.drug_id = p.drug_id
WHERE p.disease_id IN (SELECT coalesce(c.orpha_id, c.icd11_id) FROM trial_condition c WHERE c.icd11_id LIKE 'icd11:CB03.4%')
ORDER BY p.n_trials DESC LIMIT 50;
-- IPF (1.3 s): nintedanib 26 trials, 'generic approval', 8 US generics marketed; pirfenidone 22, 12 generics;
-- acetylcysteine 10, 7; sildenafil 8, 15; pamrevlumab 5, no loe row (never approved); nerandomilast 5, 'protection'
```

The program of a rare disease is keyed by its Orphanet id (`orpha:2032` here), so match the trials' condition ids
through `coalesce(orpha_id, icd11_id)` rather than the ICD-11 prefix alone.

### K3. Positive programs led by non-industry sponsors

```sql
SELECT g.name AS drug, s.name AS disease, s.rare, p.highest_phase, p.best_outcome, o.name AS lead_sponsor, o.kind, p.n_trials
FROM program p JOIN drug g ON g.drug_id = p.drug_id JOIN org o ON o.org_id = p.lead_org_id
LEFT JOIN disease s ON s.disease_id = p.disease_id
WHERE p.best_outcome IN ('positive_significant', 'positive_trend')
  AND o.kind IN ('academic', 'hospital', 'government') AND p.highest_phase <> 'approved'
ORDER BY (p.best_outcome = 'positive_significant') DESC, p.n_trials DESC LIMIT 50;
-- release-wide (1.0 s) the top rows are generic oncology backbones in common cancers (irinotecan, bortezomib,
-- venetoclax ...), so scope it to a disease or a rarity before reading it as a list of stranded assets
```

`lead_org_id` is the sponsor most trials name, not the only one; `highest_phase <> 'approved'` keeps marketed drugs
whose labels were not read (C2), so check `territory_status` for each survivor.

### K4. Did industry pick up a drug after non-industry evidence (one program)

```sql
WITH tr AS (
  SELECT DISTINCT t.trial_id, t.start_date, t.phase, coalesce(g.kind, 'unknown') AS kind
  FROM trial_drug d JOIN trial t ON t.trial_id = d.trial_id
  LEFT JOIN trial_org o ON o.trial_id = t.trial_id AND o.role = 'lead sponsor' AND o."primary"
  LEFT JOIN org g ON g.org_id = o.org_id
  WHERE d.drug_id = :moiety AND d.role = 'investigational'
    AND EXISTS (SELECT 1 FROM trial_condition c WHERE c.trial_id = t.trial_id AND c.icd11_id LIKE 'icd11:CB03.4%'))
SELECT count(*) AS trials, count(*) FILTER (WHERE kind = 'industry') AS industry_trials,
       count(*) FILTER (WHERE kind = 'unknown') AS unknown_kind,
       min(start_date) FILTER (WHERE kind IN ('academic', 'hospital', 'government')) AS first_non_industry_start,
       min(start_date) FILTER (WHERE kind = 'industry' AND phase IN ('phase_2', 'phase_2_3', 'phase_3', 'phase_4')) AS first_industry_phase2plus_start
FROM tr;
-- pirfenidone in IPF: 22 trials, 12 industry, 4 of unknown kind; industry phase 2+ from 2006-04-01, non-industry
-- from 2009-11-01: no pickup. Report unknown_kind: org.kind is NULL on 40 % of companies
```

### K5. Rare diseases with a positive program and nothing active

```sql
SELECT s.disease_id, s.name,
       (SELECT max(v.value) FROM prevalence v WHERE v.disease_id = s.disease_id AND v.measure = 'point') AS point_per_100k,
       count(*) AS n_programs, count(*) FILTER (WHERE p.status = 'active') AS n_active,
       count(*) FILTER (WHERE p.highest_phase = 'approved') AS n_approved,
       count(*) FILTER (WHERE p.best_outcome IN ('positive_significant', 'positive_trend')) AS n_positive
FROM program p JOIN disease s ON s.disease_id = p.disease_id
WHERE s.rare IS TRUE AND s.status = 'active'
GROUP BY s.disease_id, s.name
HAVING count(*) FILTER (WHERE p.best_outcome IN ('positive_significant', 'positive_trend')) > 0
ORDER BY n_active, n_positive DESC LIMIT 30;
-- (0.1 s) myelodysplastic neoplasm with low blasts 11 programs, 0 active, 8 positive; intravascular large B-cell
-- lymphoma 4/0/4; … point_per_100k is NULL where Orphanet gives a class, not a point
```

### K6. Drug kinds and mechanisms in a disease

`drug.kind` has four values in the release: `small_molecule` (131,575), `other` (32,537), `biologic` (7,803),
`combination` (3,030). Mechanisms are Open Targets' (`drug_target`, 7,708 rows):

```sql
SELECT g.name, dt.target, dt.action, count(DISTINCT d.trial_id) AS trials
FROM trial_condition c JOIN trial_drug d ON d.trial_id = c.trial_id AND d.role = 'investigational'
JOIN drug g ON g.drug_id = d.drug_id JOIN drug_target dt ON dt.drug_id = d.drug_id
WHERE c.icd11_id LIKE 'icd11:CB03.4%'
GROUP BY 1, 2, 3 ORDER BY 4 DESC LIMIT 60;
```
