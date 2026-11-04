# Recipes: the query shapes, run against the twin

Every recipe below was run on release `twin@2026-11-02T23:56:16+00:00` on 3 Nov 2026; the line after each query says
what came back, so the shape can be checked against a later release. Ids are the twin's (`drug_id`, `trial_id`,
`org_id` are uuids; `disease_id` is `icd11:<code>` or `orpha:<code>`). Resolve them first (A), then query by id.
Bare table names; `count(DISTINCT trial_id)`; `LIMIT`.

## A. Resolve names to ids

**A1. A drug, a brand, a code, a registry id.** `twin_find("tulisokibart")` → one match, `entity_type: drug`,
`entity_id: f462fdd4-663d-59e0-951a-0597550ca382`, `via: name`, `kind: display`. `twin_find("Skyrizi")` → the drug
(`e9d253f9-80be-577d-b256-8bf32c2aca6d`, `kind: brand`) and one `product` per territory (`FDA/BLA761105/001`,
`EC/EU/1/19/1361/-`, `HC/02487454/-`, ...): take the entity type you want. `twin_find("nct:NCT04996797")` → the trial
(`via: external_id`). In SQL:

```sql
SELECT entity_type, entity_id, name, kind FROM entity_name WHERE lower(name) = lower('Skyrizi');
SELECT entity_type, entity_id FROM external_id WHERE scheme = 'nct' AND value = 'NCT04996797';
```

Then read the drug's own row: `SELECT drug_id, name, kind, unii, moiety_id, code_names, max_phase FROM drug WHERE
drug_id = '...'`. Query trials and products by `moiety_id`.

**A2. A disease: ICD-11 stem, residual codes, Orphanet entity.**

```sql
SELECT d.disease_id, d.system, d.code, d.name, d.rare, d.status, d.parent_ids
FROM disease d
WHERE d.disease_id IN (SELECT entity_id FROM entity_name WHERE entity_type = 'disease' AND lower(name) = 'ulcerative colitis')
   OR d.disease_id IN ('icd11:DD71.0', 'icd11:DD71.Z');
```

→ `icd11:DD71` Ulcerative colitis (parents `[]`), `icd11:DD71.0` Ulcerative pancolitis and `icd11:DD71.Z` Ulcerative
colitis, unspecified (parents `[icd11:DD71]`, ancestors `icd11:DD71`), `orpha:771` Ulcerative colitis (rare false).
Keep the stem and select its subtypes with `icd11_id LIKE 'icd11:DD71%'` or `ancestor_ids @> ARRAY['icd11:DD71']`.
The Orphanet id adds trials only for a rare disease (where `trial_condition.orpha_id` is set); say whether you kept it.

**A3. A company and its group.** `twin_find("Prometheus Biosciences")` → `org d3a43215-b499-5836-bafd-9f3a7ef3baca`
("Prometheus Biosciences Inc.", `via: name prefix`). Then the chain:

```sql
SELECT h.name AS holder, oe.controls, oe.ultimate, oe.valid_from, oe.valid_from_basis, oe.not_after
FROM ownership_effective oe JOIN org h ON h.org_id = oe.holder_org_id
WHERE oe.org_id = 'd3a43215-b499-5836-bafd-9f3a7ef3baca';
```

→ holder Merck & Co., Inc. (`a3dd7332-78cd-56e8-af14-22bb3ab96d47`), controls true, ultimate true, valid_from
2023-06-16, basis `8-K 5.01 sec:0001193125-23-168524`. A sponsor string may sit under several `org` rows
(`trial_org.resolution_status`): gather them all before counting (E1).

## B. Trials of a drug

**B1. By phase and status, counting trials.**

```sql
SELECT t.phase, t.status, count(DISTINCT t.trial_id) AS trials
FROM trial_drug td JOIN trial t ON t.trial_id = td.trial_id
WHERE td.drug_id = 'f462fdd4-663d-59e0-951a-0597550ca382' AND td.role = 'investigational'
GROUP BY 1, 2 ORDER BY 1, 2;
```

→ 6 rows, 13 trials: phase_2 active_not_recruiting 4, completed 2, not_yet_recruiting 2, recruiting 1; phase_3
active_not_recruiting 1, recruiting 3. One of the four phase 3 rows is CTIS-only (`ctis:2024-518296-56-00`, same title
as NCT06430801; its raw record's `protocol_code` reads "MK-7240-008 study 2"): the second study of a programme the
ClinicalTrials.gov record covers in one entry. Thirteen registrations, twelve if programmes are counted: say which.
Drop `td.role` to include trials where the drug is a comparator or background.

**B2. The registry ids and member records of those trials.**

```sql
SELECT t.trial_id, t.phase, t.status,
       string_agg(e.scheme || ':' || e.value, ', ' ORDER BY e.scheme) AS registry_ids,
       (SELECT count(*) FROM trial_record r WHERE r.trial_id = t.trial_id) AS records
FROM trial t JOIN external_id e ON e.entity_type = 'trial' AND e.entity_id = t.trial_id
WHERE t.trial_id IN (SELECT trial_id FROM trial_drug WHERE drug_id = 'f462fdd4-663d-59e0-951a-0597550ca382')
GROUP BY t.trial_id, t.phase, t.status ORDER BY records DESC LIMIT 6;
```

→ e.g. `b2ea9860-…` phase_2 completed: `ctis:2023-509741-12-00, eudract:2021-000091-11, nct:NCT04996797`, 3 records.
One trial, three registrations: count the trial. To print titles use `coalesce(t.brief_title, t.title)`: a CTIS-only
trial has no lay title.

**B3. The indication landscape of a drug.**

```sql
SELECT pr.disease_id, ds.name AS disease, pr.highest_phase, pr.status, pr.best_outcome, pr.n_trials, pr.n_papers,
       pr.last_activity, o.name AS lead
FROM program pr LEFT JOIN disease ds ON ds.disease_id = pr.disease_id LEFT JOIN org o ON o.org_id = pr.lead_org_id
WHERE pr.drug_id = 'f462fdd4-663d-59e0-951a-0597550ca382'
ORDER BY pr.n_trials DESC, pr.disease_id;
```

→ 13 rows: Crohn disease phase_3 active positive_trend (4 trials, 1 paper), Ulcerative colitis phase_3 active
positive_significant (3, 3), then eleven phase_2 programs (sarcoidosis of lung, uveitis, CTD-ILD, plaque psoriasis,
hidradenitis suppurativa, RA, PsA, axial SpA, systemic sclerosis…), leads MERCK SHARP & DOHME LLC, MSD K.K. and
Prometheus Biosciences, Inc. A disease appears under both its ICD-11 and its Orphanet id when the condition linked to
a rare disease (`orpha:90291` Systemic sclerosis beside `icd11:CB05.1`).

## C. Products, labels, approvals, exclusivity

**C1. Where a drug is authorised, the summary.**

```sql
SELECT ts.territory, ts.status, o.name AS holder, ts.basis
FROM territory_status ts LEFT JOIN org o ON o.org_id = ts.holder_org_id
WHERE ts.drug_id = 'e9d253f9-80be-577d-b256-8bf32c2aca6d' AND ts.disease_id IS NULL
ORDER BY ts.status, ts.territory;
```

→ risankizumab: 16 rows over 14 territories, all `authorised` (CA twice: basis `product` and `product, not marketed`;
JP twice, one per source): BR, CA, CH, CN, EU-central (AbbVie Deutschland GmbH And Co. KG), IS, JP (AbbVie GK), KR, LI,
NO, SG, TW, US (AbbVie Inc.), XI. `holder` resolves in five territories; the register names an AbbVie affiliate in every
one (`product_holder.holder_as_written`: AbbVie AG for CH, AbbVie Corporation for CA, AbbVie Korea, AbbVie Pte for SG,
AbbVie Farmacêutica for BR, Abbvie Ltd for XI): join `product_drug` → `product_holder` for the string when `holder` is
NULL. The CN row rests on web evidence (`complete = false`, brand NULL). `disease_id IS NULL` is the drug in any
indication; per-indication rows exist where a label was read.

**C2. The products themselves.**

```sql
SELECT p.territory, p.regulator, p.brand, p.procedure, p.approved_on, p.marketed, o.name AS holder
FROM product_drug pd JOIN product p ON p.product_id = pd.product_id
LEFT JOIN product_holder ph ON ph.product_id = p.product_id LEFT JOIN org o ON o.org_id = ph.org_id
WHERE pd.drug_id = 'e9d253f9-80be-577d-b256-8bf32c2aca6d'
ORDER BY p.territory, p.approved_on NULLS LAST LIMIT 40;
```

→ 40+ rows (the LIMIT cuts before US): BR ANVISA SKYRIZI 2019-05-27 marketed; CA HC eight strengths 2019 to 2024; CH
Swissmedic six presentations (marketed NULL: the register does not say); EU-central EC Skyrizi `centralised full`
2019-04-26 marketed, holder AbbVie Deutschland; JP PMDA/MHLW six `new drug` rows with approval dates and six price-list
rows in Japanese (`originator`, marketed true); KR MFDS five. One product per regulator per presentation: aggregate
before counting, and prefer C1 for a verdict.

**C3. What the label says, with the quote.** Check coverage first (I1): `approved_indication` reads a minority of labels.

```sql
SELECT p.territory, p.brand, ai.icd11_id, ds.name AS disease, ai.population, ai.approved_on,
       ai.evidence->>'quote' AS quote, ai.evidence->>'passage_id' AS passage_id
FROM approved_indication ai JOIN product p ON p.product_id = ai.product_id
JOIN product_drug pd ON pd.product_id = p.product_id LEFT JOIN disease ds ON ds.disease_id = ai.icd11_id
WHERE pd.drug_id = '<moiety>' ORDER BY p.territory, ai.icd11_id;
```

→ For risankizumab and upadacitinib: **no row** (their labels are not among those read). Rows exist for 989 US
products (5,160 statements) and 51 EU-central products (152): e.g. Abevmy (bevacizumab, EU-central): `icd11:2C61`
Invasive carcinoma of breast, population "age: adult patients; line: first-line; stage: metastatic…", quote "Abevmy in
combination with paclitaxel is indicated for first-line treatment of adult patients with metastatic breast cancer.",
passage `9524d1f8-…/therapeutic_indication`; a statement the coder could not place has `icd11_id` NULL. An empty result
means the label was not read (coverage `approved_indication` is complete = false for US and EU-central), not that
the drug has no indication: say "label not read", then fall back to `regulatory_event` (C4) and `territory_status` (C1).

**C4. Approval history: approvals, supplements, opinions.**

```sql
SELECT re.regulator, re.procedure, re.kind, re.event_on, re.planned, o.name AS applicant
FROM regulatory_event re LEFT JOIN org o ON o.org_id = re.applicant_org_id
WHERE re.drug_id = 'e9d253f9-80be-577d-b256-8bf32c2aca6d' ORDER BY re.event_on;
```

→ 16 rows: EC EMEA/H/C/004759 opinion 2019-02-28 and approved 2019-04-26 (AbbVie Deutschland); FDA BLA761105 approved
2019-04-23, then efficacy supplements S-009/S-010 2021-04-26, S-014 2022-01-21, S-016 2022-06-16, …, S-045 2026-06-26;
FDA BLA761262 approved 2022-06-16 with its own supplements. `planned = true` rows are PDUFA dates. The indication a
supplement added is not modelled (`disease_id` NULL): read the label (C3) or the record (`sources`).

**C5. What protects a product and when exclusivity ends.**

```sql
WITH d AS (SELECT drug_id FROM drug WHERE lower(name) = 'upadacitinib' AND moiety_id = drug_id)
SELECT 'loe' AS what, l.territory, l.basis AS kind, l.set_by AS number, l.expires, l.completeness AS note
FROM loe l WHERE l.drug_id IN (SELECT drug_id FROM d)
UNION ALL
(SELECT 'protection', pr.territory, pr.kind, pr.number, pr.expires, string_agg(DISTINCT coalesce(pr.use_code, ''), ' ')
 FROM product_drug pd JOIN product p ON p.product_id = pd.product_id JOIN protection pr ON pr.product_id = p.product_id
 WHERE pd.drug_id IN (SELECT drug_id FROM d) AND pr.territory = 'US' AND NOT pr.delisted
 GROUP BY pr.territory, pr.kind, pr.number, pr.expires ORDER BY pr.expires DESC LIMIT 12)
ORDER BY what, expires DESC NULLS LAST;
```

→ loe US `protection` 2031-01-15, set_by `us/NDA211675/003/patent_substance/8962629`, completeness "compound patent (the
earliest-expiring drug-substance patent, its extensions included)…"; loe EU-central `protection` 2029-12-16, set_by
`eu/EU/1/19/1404/EMEA/H/C/004760/eu_market/2004` ("regulatory protection from the first central authorisation…");
protection US `patent_use` 11564922 and 11607411 expiring 2038-03-09 with use codes U-4303, U-4302, U-3341, U-3624,
and lower down 8962629 as both `patent_substance` and `patent_use` (2031-01-15), the row `loe` counted. Group by
(`kind`, `number`, `expires`): without it each patent repeats once per product strength and use code, and a `DISTINCT`
on (`number`, `expires`) alone merges the two kinds. Read `loe.basis` before the date.

**C6. Generics marketed and authorised.**

```sql
WITH d AS (SELECT drug_id FROM drug WHERE lower(name) = 'apixaban' AND moiety_id = drug_id)
SELECT cv.territory, cv.form, cv.route, cv.strength, cv.generic_products, cv.generic_products_authorised, cv.covered
FROM cheap_version cv WHERE cv.drug_id IN (SELECT drug_id FROM d) AND cv.territory IN ('US', 'EU-central', 'DE', 'JP')
ORDER BY cv.territory, cv.strength;
```

→ US TABLET ORAL 2.5MG and 5MG: 1 marketed, 13 authorised, covered true (approved generics held back); EU-central 1/1,
covered true; DE and JP rows with 0/0 and `covered = false`: no complete register, so zero is not evidence.

**C7. Designations.** `SELECT kind, status, number, granted_on, condition_as_written, substance_as_written FROM
designation WHERE drug_id = '<moiety>' OR substance_as_written ILIKE '%<name>%'` → EMA orphan and PRIME only today
(kinds `orphan` 3,305 rows, `prime` 46); FDA orphan and breakthrough are not read (coverage).

## D. Trials of a disease

**D1 + D3. Trials in a disease started in a year.**

```sql
SELECT count(DISTINCT t.trial_id) AS trials,
       count(DISTINCT t.trial_id) FILTER (WHERE t.phase IN ('phase_2', 'phase_1_2', 'phase_2_3')) AS phase_2,
       count(DISTINCT t.trial_id) FILTER (WHERE t.phase = 'phase_3') AS phase_3
FROM trial t
JOIN trial_condition tc ON tc.trial_id = t.trial_id AND tc.icd11_id LIKE 'icd11:DD71%'
JOIN trial_event ev ON ev.trial_id = t.trial_id AND ev.kind = 'first patient'
WHERE ev.event_on >= '2024-01-01' AND ev.event_on < '2025-01-01'
  AND EXISTS (SELECT 1 FROM trial_drug td WHERE td.trial_id = t.trial_id);
```

→ 67 drug trials in ulcerative colitis (code DD71 and subtypes) with a first-patient date in 2024, 24 phase 2, 11 phase
3; 58 by `trial.start_date` instead. The `EXISTS` keeps trials with a drug arm; drop it for every trial. Adding the
Orphanet id (`orpha:771`) changes nothing for a common disease; it matters for rare ones, where `orpha_id` is set.
`trial_event.actual` is true, false or NULL (the registry did not say) and a trial has one `first patient` row per
member record that states it (80 rows for these 67 trials): take `min(event_on)` per trial, or `bool_or(actual)`,
before grouping by the flag. For a drug's starts per year: group `trial_event` by `extract(year FROM event_on)` over
the trial ids of B1 (tulisokibart: 2021 2, 2022 1, 2023 1, 2024 2, 2025 5, 2026 3; `actual` true on 10 of 14).

**D2. What is tested in a disease, ranked.**

```sql
SELECT d.name AS drug, count(DISTINCT td.trial_id) AS trials
FROM trial_condition tc
JOIN trial_drug td ON td.trial_id = tc.trial_id AND td.role = 'investigational'
JOIN drug d ON d.drug_id = td.drug_id
WHERE tc.icd11_id LIKE 'icd11:DD71%'
GROUP BY d.name ORDER BY trials DESC LIMIT 10;
```

→ MESALAMINE 89, INFLIXIMAB 57, FECAL MICROBIOTA 55, VEDOLIZUMAB 51, BUDESONIDE 26, GOLIMUMAB 25, ADALIMUMAB 24,
TOFACITINIB 22, UPADACITINIB 18, ETRASIMOD 17. Components not linked to a drug (`drug_id` NULL) are absent from this
ranking: add `WHERE td.drug_id IS NULL` over `raw_name` to see them.

**D4. How rare, and the code on the other side.**

```sql
SELECT d.disease_id, d.name, d.rare,
       (SELECT string_agg(m.to_id || ' (' || m.relation || ', ' || m.source || ')', '; ') FROM disease_map m WHERE m.from_id = d.disease_id) AS maps_to,
       (SELECT string_agg(pv.measure || ' ' || coalesce(pv.value::text, pv.low || '-' || pv.high) || ' @' || pv.area, '; ') FROM prevalence pv WHERE pv.disease_id = d.disease_id) AS prevalence
FROM disease d WHERE d.disease_id = 'orpha:98896';
```

→ Duchenne muscular dystrophy, rare true, maps to `icd11:8C70.1` (exact, orphanet and mondo); prevalence per
100,000: birth 9.9 @world, 14 @US, 12 @NL…; point 2.8 @world, 3.56 @JP…; lifetime 7.66 @EG. `class only` rows carry
`low`/`high` with `value` NULL.

## E. Companies

**E1. A group's trials, rolled up.**

```sql
SELECT t.phase, count(DISTINCT t.trial_id) AS trials
FROM trial_org tor JOIN trial t ON t.trial_id = tor.trial_id
WHERE tor.role = 'lead sponsor' AND tor."primary"
  AND (tor.org_id = 'a3dd7332-78cd-56e8-af14-22bb3ab96d47'
       OR tor.org_id IN (SELECT org_id FROM ownership_effective WHERE holder_org_id = 'a3dd7332-78cd-56e8-af14-22bb3ab96d47'))
GROUP BY 1 ORDER BY 2 DESC;
```

→ Merck & Co. and the companies it controls as primary lead sponsor: phase_3 731, phase_2 535, phase_1 449, phase_4 96,
phase_1_2 79, phase_2_3 31, NA 24, NULL 7. `"primary"` is a reserved word: quote it. Sponsor strings the companies
release did not fold (`resolution_status` minted or clustered) are separate `org` rows: find them with `twin_find` on
the string and add their ids.

**E2. Who owns whom, dated.** `SELECT h.name AS holder, o.name AS company, ow.kind, ow.method, ow.valid_from,
ow.valid_from_basis, ow.valid_to, ow.not_after FROM ownership ow JOIN org o ON o.org_id = ow.org_id JOIN org h ON
h.org_id = ow.holder_org_id WHERE o.name ILIKE 'Prometheus Bio%'` → Merck & Co., Inc. over Prometheus Biosciences,
Inc., kind ownership, method sec_exhibit, valid_from 2023-06-16, not_after 2025-02-25. `method` ranks the evidence:
sec_exhibit > gleif_consolidation > gleif_branch > stated_parent > web_parent (a web answer with cited pages).

**E3. A company's filings.**

```sql
SELECT f.form, f.filed_at, f.items, f.filing_id
FROM filing f JOIN filing_party fp ON fp.filing_id = f.filing_id
JOIN external_id e ON e.scheme = 'cik' AND e.entity_type = 'org' AND 'cik:' || e.value = fp.filer_id
WHERE e.entity_id = 'a3dd7332-78cd-56e8-af14-22bb3ab96d47' AND f.form = '8-K' AND f.items @> ARRAY['2.01']
ORDER BY f.filed_at DESC LIMIT 6;
```

→ Merck 8-Ks with item 2.01 (completion of an acquisition): 2021-06-03 `sec:0001193125-21-180657`, 2014-10-02,
2009-11-04 (items 2.01, 2.03, 3.03, 4.01, 5.02, 5.03, 5.05, 8.01), 2007-11-26. The filing's text is not held: the deal
terms need the filing itself.

**E4. Partners on a drug while `deal` is empty.** The twin holds no agreements; the registries show who runs the
trials over time.

```sql
SELECT t.registered_on, coalesce(t.brief_title, t.title) AS title, tor.role, tor."primary", tor.raw_name
FROM trial_drug td JOIN trial t ON t.trial_id = td.trial_id JOIN trial_org tor ON tor.trial_id = t.trial_id
WHERE td.drug_id = '6d82d11b-9a3b-5d4c-aa0f-e16a758602ca' AND tor.role IN ('lead sponsor', 'collaborator')
ORDER BY t.registered_on, tor.role;
```

→ duvakitug: Teva Branded Pharmaceutical Products R&D as lead sponsor with Sanofi as collaborator on the 2020 to 2023
trials; Sanofi (and Sanofi K.K.) as lead with Teva as collaborator from 2025. Present this as the sponsorship pattern
it is, with `coverage` saying deals are not read; never as a licence.

## F. Results, stops, papers

**F1. What a trial found, and from when.**

```sql
SELECT o.outcome, o.primary_met_significant, o.basis, o.known_from, o.n_randomised, o.n_analysed, o.model, o.confidence,
       o.evidence
FROM trial_outcome o JOIN external_id e ON e.entity_type = 'trial' AND e.entity_id = o.trial_id
WHERE e.scheme = 'nct' AND e.value = 'NCT04996797';
```

→ `positive_significant`, primary met, basis `paper`, known_from 2024-09-25 (published), n_randomised 178, n_analysed
135, model claude-sonnet-5-5, confidence 0.856; evidence quote "A significantly higher percentage of patients who
received tulisokibart had clinical remission than those who received placebo (26% vs. 1%; …P<0.001)." with its
`passage_id`. **As of a date:** add `AND o.known_from <= '2024-06-30'` → no row: on that day the twin had no
established outcome (the posted results of 2024-06-27 arrive in a later `known_from` only if they were the latest
evidence; here the paper superseded them). Say "not knowable then", not "negative".

**F2. The quoted sentence, in full.** `SELECT section, text FROM passage WHERE passage_id = '<evidence.passage_id>'`.
Cite `passage_id`.

**F3. Why trials stopped.**

```sql
SELECT t.stop_class, count(*) AS trials, min(t.why_stopped) AS an_example
FROM trial t
WHERE t.status IN ('terminated', 'withdrawn', 'suspended') AND t.phase = 'phase_3' AND t.registered_on >= '2022-01-01'
GROUP BY 1 ORDER BY 2 DESC;
```

→ business 115 ("Business decision"), not_stated 64, other 59, enrolment 36, efficacy 30 ("…no clinical benefit for
the patients."), funding 22, regulatory 3, safety 3. `stop_class` is a keyword rule (`stop_method` names its
version): read `why_stopped` before citing a class.

**F4. Papers reporting a trial.**

```sql
SELECT pt.relation, p.pmid, p.doi, p.title, p.journal, p.published_on, p.type, p.retracted, pt.found_by
FROM paper_trial pt JOIN paper p ON p.paper_id = pt.paper_id
WHERE pt.trial_id = (SELECT entity_id FROM external_id WHERE scheme = 'nct' AND value = 'NCT04996797' AND entity_type = 'trial')
ORDER BY pt.relation, p.published_on;
```

→ 3 exact: PMID 39321363 (NEJM 2024-09-25, article, declared by the paper and by the registry), and two
conference abstracts (ECCO 2026 P0837, BSG 2026 P170). `retracted` false on all. A paper's drugs: `paper_drug` on the
moiety, `relation` the strongest over its trials.

## G. Text and raw records

**G1. Find a sentence.** `twin_text('tulisokibart "clinical remission" week 12', limit=3)` → three passages: two
`registry record` sections `strategy/trial` of the phase 3 programmes (NCT06052059, NCT06430801) with the trial in
`links`, and the NEJM `abstract` (`kind: abstract`, `paper_id`, licence CC0). Each has `passage_id`, `snippet`, the
first 2,000 characters and `url`. Quote from `text`, cite `passage_id` and the document title.

**G2. Read the record.** `twin_record("clinicaltrials.gov:NCT04676178")` → `version: ctgov@2026-10-23`, `chars: 7103`,
`text`: the API v2 JSON (`protocolSection.{identificationModule, statusModule, designModule, armsInterventionsModule,
eligibilityModule, …}`, `hasResults`). Longer records page with `offset` / `next_offset`. Keys come from any row's
`sources` or from `trial_record.record_key`.

**G3. The registered endpoints.**

```sql
SELECT te.kind, te.position, te.measure, te.timepoint
FROM trial_endpoint te
WHERE te.trial_id = (SELECT entity_id FROM external_id WHERE scheme = 'nct' AND value = 'NCT04996797' AND entity_type = 'trial')
ORDER BY te.kind, te.position;
```

→ primary 0 "Percentage of Participants in Cohort 1 Achieving Clinical Remission" (Baseline and Week 12), primary 1–3
safety measures (Up to ~14 weeks), secondary 0 Endoscopic Improvement, 1 Clinical Response, … These are the protocol's
endpoints; what was found is F1.

## I. Coverage and empties

**I1. Is an absence evidence?**

```sql
SELECT entity_type, coalesce(source, '-') AS source, scope, complete, through_on, through_basis
FROM coverage WHERE entity_type IN ('product', 'approved_indication', 'protection', 'paper', 'trial', 'deal')
ORDER BY entity_type, scope, source;
```

→ product: complete for US (drugsfda, openfda/drugsfda, purple_book), EU-central (ema_medicines, eu_union_register), CA
(hc_dpd), BR (anvisa), TW (tfda); **not** complete for CH, JP, KR, SG, CN, EEA-national. approved_indication US and
EU-central complete false. protection orange_book US complete true. paper: "papers a trial of the release lists", not a
corpus. trial (source pharma): complete **false**. deal: `through_basis = 'not read'`. So "no US generic" (US product
registers complete) is evidence; "not authorised in Japan" is not; "no deal" is unknown.

**I2. What is empty in this release.** `SELECT table_name, about FROM catalog WHERE row_count = 0` → deal, deal_party,
payment, right_grant, royalty, royalty_party, royalty_paid, financial, org_runway, price, spend, claim, opportunity. Each
`about` begins "Published EMPTY". Answer "unknown" and say why.
