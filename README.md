# mondoTWIN Researcher

Five skills and the mondoTWIN connector for clinical researchers who design trials. The assistant searches mondoTWIN's database of registered clinical trials and regulatory guidance, and every trial or passage in an answer is cited with its registry id or guideline section, so you can check it at the source.

## What it does

| Skill | Ask for | You get |
|---|---|---|
| `similar-trials-benchmark` | "Trials similar to a phase 3 of colchicine after MI" | A table of comparable trials: registry id with link, phase, status, enrolment (actual / target), primary endpoint and time frame, key criteria, stop reason; how many were searched and shown |
| `protocol-section-review` | "Review this endpoints section" (paste it) | Numbered, reasoned changes, each with its evidence (trial ids or a quoted ICH E6(R3), E8(R1), E9(R1) or M11 passage), marked "to validate" |
| `eligibility-criteria` | "Draft eligibility for a phase 2 in IPF" | Inclusion and exclusion criteria drawn from comparable trials, plus medication exclusions backed by a drug-interaction screen |
| `endpoint-and-estimand` | "What did similar trials measure, and what is my estimand?" | The endpoints and time points of comparable trials, and an estimand following ICH E9(R1): treatment, population, variable, intercurrent events and strategies, summary measure |
| `guidance-lookup` | "What does ICH E9(R1) say about the while-on-treatment strategy?" | The answer with the verbatim passage, guideline id and section |

The skills load when your request matches them, or pick one by typing `/` in the message box. The assistant answers in your language.

## Data

- ClinicalTrials.gov, as of 13 July 2026
- EU CTR / CTIS, ISRCTN, ANZCTR and UMIN, as of 14 May 2026
- ICH guidelines and FDA guidance documents, as held in the mondoTWIN database (not every FDA guidance is included)
- Drug-interaction evidence from DrugBank and label-derived organ-toxicity data; disease-target associations from Open Targets

Trials registered or updated after these dates are not in the database.

## Requirements

- A mondoTWIN account with researcher access. Request it at https://www.mondotwin.com/researchers.
- The bundled connector, `mondotwin`, at `https://mcp.mondotwin.com/mcp` (remote MCP over streamable HTTP). You sign in with your mondoTWIN account through OAuth; the plugin contains no keys or secrets.

## Install on Claude (claude.ai, the desktop app, Cowork)

1. Open **Customize > Plugins** (https://claude.ai/customize/plugins).
2. Select **Add > Add marketplace** and enter `mondotwin/mondotwin-researcher`.
3. Find **mondoTWIN Researcher** and select **Add**.
4. Open the plugin, go to its **Connectors** tab, and connect **mondotwin**. Sign in with your mondoTWIN account. (On Team and Enterprise plans an Owner may have to add the connector for the organisation first.)
5. Start a new chat and ask, for example, "Find trials similar to a phase 2 trial of an inhaled antifibrotic in IPF."

The plugin then also reaches Claude Code as a synced plugin at the next session start.

## Install in Claude Code

```
/plugin marketplace add mondotwin/mondotwin-researcher
/plugin install mondotwin-researcher@mondotwin
/reload-plugins
/mcp
```

In `/mcp`, select the `mondotwin` server and sign in. If you already added the mondoTWIN server yourself (for example as a user-scope server named `mondotwin`), remove one of the two so the tools are not listed twice.

## Using it well

- Name the drug and the indication. The trial search keeps the trials that contain every word you give it, so one to three distinctive words (a drug, a disease) work best.
- Paste only protocol text. Do not paste patient data.
- Treat every output as a draft. The skills cite their evidence so that clinicians, statisticians and regulatory specialists can validate it before use.

## What data leaves your session

Your search terms, drug names, and the protocol excerpts the assistant needs to search with are sent to `mcp.mondotwin.com`, which returns registry records and guidance passages. mondoTWIN processes these requests under your account; see https://www.mondotwin.com for its terms and privacy policy.

## Contents

```
.claude-plugin/plugin.json        plugin manifest
.claude-plugin/marketplace.json   marketplace catalog (this repository is its own marketplace)
.mcp.json                         the mondoTWIN connector
skills/<name>/SKILL.md            the five skills (Agent Skills format)
```

## Licence

Copyright (c) 2026 SHRARE Technology AG. All rights reserved; see `LICENSE`.

Published by SHRARE Technology AG / mondoTWIN. https://www.mondotwin.com
