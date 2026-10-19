# mondoTWIN Researcher

Six skills and the mondoTWIN connector for clinical researchers who design trials, for Claude and ChatGPT. The assistant searches mondoTWIN's database of registered clinical trials and regulatory guidance, and every trial or passage in an answer is cited with its registry id or guideline section, so you can check it at the source.

## What it does

| Skill | Ask for | You get |
|---|---|---|
| `similar-trials-benchmark` | "Trials similar to a phase 3 of colchicine after MI" | A table of comparable trials: registry id with link, phase, status, enrolment (actual / target), primary endpoint and time frame, key criteria, stop reason; how many were searched and shown |
| `query-clinical-studies` | "How many phase 3 trials in IPF stopped early, and why?" | Counts and comparisons across registered trials (phases, status, enrolment by phase, primary endpoints and time frames, stop reasons, sponsor class and country), with the read-only SQL query used and the registry ids behind the numbers |
| `protocol-section-review` | "Review this endpoints section" (paste it) | Numbered, reasoned changes, each with its evidence (trial ids or a quoted ICH E6(R3), E8(R1), E9(R1) or M11 passage), marked "to validate" |
| `eligibility-criteria` | "Draft eligibility for a phase 2 in IPF" | Inclusion and exclusion criteria drawn from comparable trials, plus medication exclusions backed by a drug-interaction screen |
| `endpoint-and-estimand` | "What did similar trials measure, and what is my estimand?" | The endpoints and time points of comparable trials, and an estimand following ICH E9(R1): treatment, population, variable, intercurrent events and strategies, summary measure |
| `guidance-lookup` | "What does ICH E9(R1) say about the while-on-treatment strategy?" | The answer with the verbatim passage, guideline id and section |

The skills load when your request matches them, or pick one by typing `/` in the message box in Claude. The assistant answers in your language.

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

On Claude Free, or without plugins, download [mondotwin-researcher.zip](https://github.com/mondotwin/mondotwin-researcher/releases/latest/download/mondotwin-researcher.zip), upload it in **Customize > Skills**, and add the connector by hand in **Customize > Connectors > Add custom connector** (name `mondoTWIN`, URL `https://mcp.mondotwin.com/mcp`).

## Use in ChatGPT

ChatGPT has a **Plugins** page (left sidebar, or https://chatgpt.com/plugins). Its **Add** button offers "Create plugin", "Upload plugin archive" and "Add custom MCP server". Plugin upload and custom MCP servers depend on your ChatGPT plan; ChatGPT Business, Enterprise and Edu have them.

**With skills (ChatGPT desktop app)**

1. Download the plugin: https://github.com/mondotwin/mondotwin-researcher/releases/latest/download/mondotwin-researcher.zip
2. In ChatGPT open **Plugins > Add > Upload plugin archive**, choose the file and select **Add plugin**.
3. Open **mondoTWIN Researcher**, connect its mondoTWIN server and sign in with your mondoTWIN account.
4. Use it from the ChatGPT desktop app: plugin skills open there (ChatGPT on the web hands them to the desktop app).

**Connector only (ChatGPT on the web)**

1. Open **Plugins > Add > Add custom MCP server**.
2. **Name** `mondoTWIN`, **Server URL** `https://mcp.mondotwin.com/mcp`, **Authentication** OAuth (the description is optional).
3. Tick **I understand and want to continue** and select **Create as a plugin**.
4. Connect and sign in with your mondoTWIN account.
5. In a chat, open **+** and pick **mondoTWIN**.

The connector alone gives ChatGPT the tools without the skills' instructions.

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
.claude-plugin/plugin.json        Claude plugin manifest
.claude-plugin/marketplace.json   Claude marketplace catalog (this repository is its own marketplace)
.mcp.json                         the mondoTWIN connector, Claude format
plugin.json                       portable plugin manifest with the ChatGPT interface (OpenAI format)
mcp.json                          the mondoTWIN connector with OAuth, portable format (ChatGPT)
assets/icon.png                   plugin icon (ChatGPT)
skills/<name>/SKILL.md            the six skills (Agent Skills format), shared by both
skills/query-clinical-studies/references/   the data model and tested SQL recipes
```

## Licence

Copyright (c) 2026 SHRARE Technology AG. All rights reserved; see `LICENSE`.

Published by SHRARE Technology AG / mondoTWIN. https://www.mondotwin.com
