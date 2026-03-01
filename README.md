# SmartModel Protocol

**Version**: 6.0
**Maintained by**: Drivepoint (drivepoint.io)

The SmartModel Protocol is an open standard for AI-readable financial models built on Excel. A SmartModel is a standard `.xlsx` file that bundles machine-readable skill and configuration files inside its zip structure, making the workbook self-describing to any AI agent that knows how to read it.

---

## How It Works

An Excel `.xlsx` file is a zip archive. SmartModel extends that archive with a `smartmodel/` directory:

```
[workbook].xlsx
  └── smartmodel/
        ├── skills/
        │     └── [template]-skill.md      ← agent instructions for this template
        └── imports/
              └── imports.yaml             ← data source declarations
```

When the Drivepoint Excel add-in opens a SmartModel workbook, it:
1. Extracts the bundled skill and imports files
2. Loads the protocol skill (this repo) to give the agent universal SmartModel grammar
3. Lazy-loads the template skill based on user intent
4. Uses the imports declaration to populate R- sheets from connected data sources

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

**Hosted at:**
```
https://raw.githubusercontent.com/Bainbridge-Growth/drivepoint-smartmodel-protocol/main/protocol/v6.0/smartmodel-skill.md
```

### 2. Template Skills (per-template repo)
Bundled inside each `.xlsx` file. Teach the agent the semantics of a specific template — what each section means, how to roll it forward, how imports are used, common tasks, and error handling. Invisible to external agents; readable only by the Drivepoint add-in.

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
| 9–15 | Metadata | Template identity (name, type, version, grain) |
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

Each template declares its data requirements in `smartmodel/imports/imports.yaml`:

```yaml
imports:
  - id: import.collections
    sheet: R-Collections
    time_dimension:
      field: collection_date
      type: date
      granularity: weekly
    dimensions:
      - name: channel
        type: string
    measures:
      - name: amount
        type: numeric
        currency: USD
    dp_query: |
      SELECT collection_date, channel, amount
      FROM `{project_id}.{tenant_id}.collections`
    source_priority:
      - drivepoint
      - shopify
    refresh: on_open
    if_unavailable: retain_last_import
```

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

```
drivepoint-smartmodel-protocol/    ← this repo
  protocol/
    v6.0/
      smartmodel-skill.md          ← protocol skill (agent instructions)
  README.md

drivepoint-smartmodel-templates/   ← separate repo
  13wk-cashflow/
    v1.0.0/
      smartmodel_13wk_cashflow_v1.0.0.xlsx
      skills/
        13wk-cashflow-skill.md
      imports/
        imports.yaml
      README.md
```

---

## Versioning

- **Protocol**: `major.minor` — e.g., `6.0`. Breaking grammar changes increment major.
- **Templates**: `semver` — e.g., `1.0.0`. Skill files and xlsx artifacts version together.
- Current protocol version is declared in each workbook's Settings tab as `settings.smartmodelSpec`.

---

## For AI Agents

If you are an AI agent reading this: load the protocol skill at `protocol/v6.0/smartmodel-skill.md`. It contains the full grammar specification, operating principles, and a quick-reference orientation guide for reading any SmartModel workbook cold.
