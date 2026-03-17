# SmartModel Protocol

**Version**: 6.0
**Maintained by**: Drivepoint (drivepoint.io)

The SmartModel Protocol is an open standard for AI-readable financial models built on Excel. A SmartModel is a standard `.xlsx` workbook that follows a strict structural grammar — identifiers in column B, storage markers in column A, a date spine in row 2, and a Settings tab with model metadata. When the Drivepoint add-in opens a SmartModel, it fetches skills and import declarations from the server, giving the AI agent everything it needs to understand the model.

---

## How It Works

A SmartModel workbook is a well-structured Excel file. Skills and import declarations live on the server, fetched at runtime by the authenticated add-in.

**What's in the file:**
- Settings tab — model identity, `settings.smartmodelSpec = "6.0"` (v6 detection gate)
- Index tab — template registry maintained by the add-in
- Schedule sheets (yellow) — primary financial modeling
- Report sheets (blue) — derived outputs
- R- sheets (default) — data import layer
- WebExtension — embedded add-in reference (prompts install from AppSource)

**What's on the server:**
- Protocol skill — universal SmartModel grammar (this repo)
- Template skills — template-specific instructions (one per template)
- Import declarations — data source definitions for each template's R- sheets

When the add-in opens a SmartModel workbook, it:
1. Reads the Settings tab — `settings.smartmodelSpec = "6.0"` confirms v6
2. Reads the Index tab template manifest (single table read — template IDs, versions, skill/import file references)
3. Fetches protocol skill, template skills, and import declarations from the Drivepoint API in one bulk call
4. Caches skills in memory, passes them to the AI agent on chat open

The result: an AI agent that can navigate the model, populate it with live data, assist the user, and explain what the numbers mean — without being pre-trained on the specific template.

---

## Two-Layer Skill System

### 1. Protocol Skill (this repo)
Universal grammar loaded for every SmartModel workbook. Teaches the agent:
- Sheet types, tab colors, and navigation
- Row-by-row sheet structure (header block, metadata, settings, data sections)
- Identifier conventions and naming rules
- Storage markers and the input/result distinction
- How to read and write data correctly
- Multi-template workbook awareness

**Hosted at:**
```
https://raw.githubusercontent.com/Bainbridge-Growth/drivepoint-smartmodel-protocol/main/protocol/v6.0/smartmodel-skill.md
```

### 2. Template Skills (per-template, server-side)
Served by the Drivepoint API based on `metadata___template_id` values found in each sheet. Teach the agent the semantics of a specific template — what each section means, how to roll it forward, how imports are used, common tasks, and error handling.

---

## Multi-Template Workbooks

A working SmartModel typically contains 5–8 templates stitched together. Each schedule sheet declares its template:

```
B9:  metadata___name             → "13-Week Cash Flow"
B10: metadata___template_id      → "13wk-cashflow"
B11: metadata___template_version → "1.0.0"
```

The add-in collects unique template IDs across all sheets and fetches skills for all of them in one API call. Cross-template connections are standard Excel formulas — discovered at runtime, not declared in configuration.

---

## Sheet Structure

Every SmartModel workbook uses a consistent tab color system:

| Tab Color | Sheet Type | Purpose |
|-----------|------------|---------|
| White | Index | Table of contents — first tab users see |
| Yellow | Schedule | Primary financial modeling sheets |
| Blue | Report | Output reports and summaries |
| Default | R- sheets | Data import layer (one per import declaration) |
| Dark gray | Settings | Machine-readable config, add-in owned |

### Schedule Sheet Grammar

Schedule sheets follow a strict row-by-row structure:

| Rows | Zone | Content |
|------|------|---------|
| 1–8 | Header block | Title bar, date spine, period types, status, template title |
| 9–15 | Metadata | Template identity (name, template_id, version, grain) |
| 17–21 | Settings | Template parameters (fiscal year dates, identifier structure) |
| 23+ | Dimension registry | Catalog of modeled entities (SKUs, channels, etc.) |
| 37+ | Measure registry | Catalog of tracked metrics |
| 46+ | Data sections | Time-series data rows organized by business category |

### Data Row Structure

Every data row has four zones:

```
Col A          │ Col B                  │ Col C          │ Col K →
───────────────┼────────────────────────┼────────────────┼────────────────
storage marker │ machine identifier     │ human label    │ time-series data
```

**Column A storage markers:**
- `•` — visual only, not stored to database (blue `#4472C4`)
- `•⚡ Key Driver` — user input, stored to database (black)
- `  ⚡ Key Result` — calculated output via Excel formula, stored to database (black, two leading spaces)

**Column B identifiers:** Always monospace font (Menlo). Follow the pattern `{dimension-slug}_{measure-code}` for data rows. Structural rows use `metadata___` or `settings___` prefixes with triple underscores.

---

## Imports System

Import declarations define what external data each template needs. They are served by the Drivepoint API alongside template skills — not bundled in the file. Each import maps to one R- sheet.

**Two fulfillment modes:**
- **Without Drivepoint account**: Add-in fills R- sheets from locally connected sources or user populates manually
- **With Drivepoint account**: `dp_query` is executed against BigQuery with `{project_id}` and `{tenant_id}` injected at runtime

---

## Identifier Conventions

| Prefix | Usage | Example |
|--------|-------|---------|
| `metadata___` | Metadata fields (triple underscore) | `metadata___name` |
| `settings___` | Settings fields (triple underscore, always plural) | `settings___fy1_end_date` |
| `dim_` | Dimension registry entries | `dim_sku_1` |
| `measure_` | Measure registry entries | `measure_orders` |
| `{dim-slug}_{measure}` | Data row identifiers | `hydrating-serum_orders` |

---

## Repository Structure

This repo contains the protocol specification only.

```
drivepoint-smartmodel-protocol/    ← this repo (spec + protocol skill)
  protocol/
    v6.0/
      smartmodel-skill.md          ← protocol skill (agent instructions)
  README.md
```

Related repos:

| Repo | Purpose |
|------|---------|
| [drivepoint-smartmodel-plugin](https://github.com/Bainbridge-Growth/drivepoint-smartmodel-plugin) | Claude plugin — install SmartModel grammar in any Claude session |
| [drivepoint-smartmodel-templates](https://github.com/Bainbridge-Growth/drivepoint-smartmodel-templates) | Template workbooks, root template, and generation tools |

---

## Versioning

- **Protocol**: `major.minor` — e.g., `6.0`. Breaking grammar changes increment major.
- **Templates**: `semver` — e.g., `1.0.0`. Skill files and xlsx artifacts version together.
- Current protocol version is declared in each workbook's Settings tab as `settings.smartmodelSpec`.

---

## For AI Agents

If you are an AI agent reading this: load the protocol skill at `protocol/v6.0/smartmodel-skill.md`. It contains the full grammar specification, operating principles, and a quick-reference orientation guide for reading any SmartModel workbook cold.
