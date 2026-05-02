# Studio View
Lives a portfolio of game projects. The studio shell handles things that belong to the company as a whole: the list of projects, who the users are, what each role can see, which sales platforms are configured, and cross-project rollups (combined revenue, combined burn). You're operating above any single game.

# Project view
 A specific project from the studio's portfolio (e.g. "Covenant"). Now you're operating inside one game. Every budget cell, every ledger entry, every P&L scenario, every milestone — all of that lives at the project level. The 24 views I documented (Home, Budget Grid, P&L, Ledger, Reconciliation, etc.) are all project-scoped.

## Project Setup Wizard — Field Analysis

Let me trace through the wizard code to give you an accurate inventory.Now let me check the create functions:I have everything I need now. Here's the structured analysis:

## Project Creation — Three Paths, Same Destination

The "+ New Project" button opens a chooser modal (`step === 0`) with three options. Each path produces a project with the same data shape; they differ only in how much the user fills in upfront.

### Path A — "Blank Project"

Calls `createBlankProject()` directly, no questions asked. Uses `createDefaultProject()` (line 88) which produces a hard-coded seed: name = "Covenant", studio = "Fictions", 36-month timeline starting 2025-08, 9 default phases, **15 pre-seeded budget rows** (Lead Programmer, Senior Game Designer, Art Director, Producer, etc.), $9.75M of pre-filled contracted budget. The user opens the new project and edits everything from there.

### Path B — "Demo Project"

Calls `createTemplateProject()` — same as blank, but additionally pre-populates ledger entries, sales data, and notes so the user can explore every page with realistic data.

### Path C — "Setup Wizard" (4 steps)

This is the only path with editable fields at creation time. The other two paths give you zero fields — everything is post-creation editing.

---

## Setup Wizard — Field Inventory by Step

### Step 1 — Project Details

All fields are **static, user-entered**.

| Field | Type | Default | Notes |
|---|---|---|---|
| Project Name | Text | empty | **Required** — Next button disabled until set |
| Studio | Text | empty | Falls back to `studio.name` if left blank |
| Start Month | `<input type="month">` | Current month (`new Date().toISOString().slice(0,7)`) | Format: `YYYY-MM` |
| Duration (months) | Number | 36 | Clamped to `1..120` at finish (line 10443) |
| Multi-Currency toggle | Boolean | false | When on, reveals currency selector |
| Alt Currency | Dropdown | EUR | 20 options: EUR, GBP, JPY, CAD, AUD, CHF, CNY, KRW, BRL, MXN, SEK, NOK, DKK, PLN, CZK, INR, SGD, HKD, NZD, ZAR. Only relevant if Multi-Currency is on. |
| Point Person(s) | Multi-select chips | empty | One chip per user in `studio.users[]`. Multiple selectable. |
| Target Release Date | `<input type="month">` | empty | Optional |
| Platforms | Multi-select chips | empty | Sourced from `studio.platformOptions` (configured in studio admin); typical defaults include Steam, PS5, Xbox Series, Nintendo Switch, Epic, etc. |

**No calculated values on this step.** Everything is captured as-typed.

### Step 2 — Import Budget (optional)

| Field | Type | Default | Notes |
|---|---|---|---|
| Target Mode | Radio (3 options) | "both" | "Both (Live + Greenlit)" / "Live Only" / "Greenlit Only" — controls how imported numbers are written to the row |
| CSV Upload | File input | none | Optional; PapaParse-parsed |

**Heavy calculation happens here**, but it's parsing logic, not financial math. The CSV importer auto-detects:

- **Month columns** (`pML` parser) — reads headers like "Aug-25", "Aug 25", "2025-08" → resolves to month keys
- **Label column** — finds whichever column header matches "role/title", "item", "cost item", "description", etc.
- **Department column** — looks for "department", "dept", "category"
- **Name column** — looks for "name", "person"
- **Section assignment** — when it sees a row that looks like an ALL-CAPS section header (e.g. "DEVELOPER FEES", "PORTING", "EXTERNAL"), it switches the current section context. Mapping is hardcoded in `sM` (line 11571).
- **Subsection assignment** — same pattern with subsection keywords like "EMPLOYEE", "CONTRACTOR", "ONE-OFF" via the `sbM` map.

After parsing, the **`startDate`** and **`monthCount`** fields **get auto-overridden** by what was found in the CSV (line 11585):

```js
startDate: ak[0] || w.startDate,
monthCount: ak.length || w.monthCount
```

So if your Step 1 said "Start Aug 2025, 36 months" but your CSV has months Jan 2024 – Dec 2026, the project will quietly use Jan 2024 / 36 months. **This is one of the few places the wizard derives values rather than just capturing them.**

A preview table then lets the user **manually correct** the auto-detected section and subsection per row (two dropdowns per row).

### Step 3 — Contracted Budget (optional)

Six numeric inputs, all **static, user-entered**:

| Field | Type | Default | Feeds |
|---|---|---|---|
| Development Costs | Number | 0 | `masterBudget.developmentCosts` → P&L `devBudget`, Top Sheet, Master Budget page |
| Porting Costs | Number | 0 | `masterBudget.portingCosts` → P&L `portingBudget` |
| External Support Costs | Number | 0 | `masterBudget.externalCosts` → P&L `externalBudget` |
| Reserve | Number | 0 | `masterBudget.reserve` → P&L `reserveBudget` |
| Contingency | Number | 0 | `masterBudget.contingency` → P&L `contingencyBudget` |
| Marketing Costs | Number | 0 | `masterBudget.marketing` → Marketing Budget page contracted ceiling, P&L `marketingBudget` |

**One calculated field shown live on this step:**

| Field | Source |
|---|---|
| Total Project Costs | `Σ all six values` — recomputed on every keystroke |

### Step 4 — Review

**Read-only summary**, no editable fields. Re-displays everything entered, plus:

| Calculated field | Source |
|---|---|
| Total Project Costs (banner) | Same `Σ masterBudget` formula as Step 3 |
| Imported items count | `setupWizard.importedRows.length` (only shown if a CSV was imported) |

The **"Create Project"** button calls `finishWizardProject(setupWizard)`.

---

## What `finishWizardProject` Actually Does (line 10443)

This is where wizard inputs become a real project. It calls `createDefaultProject()` first (which gives you the 15-row default seed and 9 default phases) and then **overwrites specific fields** from the wizard:

**Captured directly from wizard:**
- `name`, `studio`, `startDate`, `multiCurrency`, `altCurrency`, `pointPerson`, `platforms`, `targetReleaseDate`, `masterBudget`

**Calculated/derived at creation time:**

| Field | Formula |
|---|---|
| `monthCount` | `Math.max(1, Math.min(120, Number(w.monthCount) \|\| 36))` — clamp to 1–120 |
| `phaseAssignment` | `buildPhaseAssignment(p.phases, ms.map(m => m.key))` — walks phases in order, consumes their `months` count, producing a `{monthKey: phaseId}` map |
| `glStartDate` | Copy of `startDate` |
| `glMonthCount` | Copy of `monthCount` |
| `glPhases` | Deep copy of `phases` |
| `glPhaseAssignment` | Copy of `phaseAssignment` |

**The greenlit shadow data is initialized to mirror live exactly.** This is critical: it means a brand-new project has zero variance until someone actually edits one side or the other.

**Conditional row handling for CSV imports:**

| `targetMode` | Effect on imported rows |
|---|---|
| `"both"` | `live = imported values`, `greenlit = copy of live` |
| `"greenlit"` | `greenlit = imported values`, `live = {}` (empty) |
| `"live"` | `live = imported values`, `greenlit = {}` (empty) |

**If no CSV was imported, the 15 default rows from `createDefaultProject()` survive — but with empty `live` and `greenlit` maps.** The user has rows to fill in but no numbers in them.

---

## Defaults the User Never Sees But Inherits

These come from `createDefaultProject()` (line 88) and are never exposed in the wizard:

- 15 pre-seeded budget rows (Lead Programmer, Senior Game Designer, Art Director, etc.) — these will be present even if the user only fills in Steps 1, 3, 4
- 9 default phases (Concept, Prototype, VS, Pre-Alpha, Alpha, Beta, RC, Post-Launch, Archive)
- `greenlitVersion: 1`, `greenlitDate: today`, `greenlitLocked: true`
- Empty `ledger`, `marketingBudget`, `salesData`, `notes`, `auditLog`, `changeHistory`, `closedMonths`, `fxHedges`
- `globalSpotRate: ""` (empty — uses 1.0 as fallback in `spotRate()`)

---

## Summary: Editable vs. Calculated

**Editable at creation (10 fields total):**
1. Project Name
2. Studio
3. Start Month
4. Duration in months
5. Multi-currency on/off
6. Alt currency code
7. Point Persons (multi-select)
8. Target Release Date
9. Platforms (multi-select)
10. Six contracted-budget line items

**Calculated/derived at creation:**
1. **Total Project Costs** — live sum of the six budget inputs (visible on Steps 3 and 4)
2. **`monthCount` clamp** — bounded to 1–120
3. **`startDate` and `monthCount` override** — overwritten by the CSV's actual month range if a CSV was imported
4. **`phaseAssignment`** — auto-built by walking the 9 default phases against the timeline
5. **Greenlit shadow data** — `glStartDate`, `glMonthCount`, `glPhases`, `glPhaseAssignment` all mirror live at creation
6. **Per-row section/subsection inference** — for CSV-imported rows, auto-detected from header keywords with manual correction allowed
7. **`live` and `greenlit` row maps** — populated from the CSV based on `targetMode`

---

## Gap Analysis

- **Validation is minimal.** Only "Project Name" is required for the wizard to advance. There's no validation that Start Date is before Target Release Date, that Duration accommodates the planned phases, or that the CSV's months match what the user typed in Step 1 (the CSV silently wins).
- **The wizard doesn't let you edit phases.** You always inherit the 9-phase default, even though phases are central to the timeline. To customize them, you must finish the wizard and then go to the Phases page. The brief's spec doesn't require phase editing in the wizard, so this isn't a gap against the brief — but it's a UX seam worth noting for the backend.
- **No way to invite collaborators during creation.** Point Persons are limited to existing studio users. The wizard doesn't tie into the user-creation flow.
- **No project template selection.** "Demo Project" is the only template; you can't, for example, "create a new project based on Project X's structure." For a studio with multiple similar games, this means re-doing the same setup repeatedly.
- **The `createDefaultProject()` seed still uses "Covenant" / "Fictions" as defaults.** If the user picks "Blank Project," they get a project literally named Covenant. This is harmless because they'll rename it, but the backend should not carry these hardcoded names into production seeds.

Want to look at how the **Setup Wizard's CSV importer** maps real-world spreadsheets into the section/subsection structure (it's surprisingly intricate), or move to a different area?


## First: What Are "Greenlit" and "Live"?

These two terms come from the film/games industry and are the spine of how Fictions Folio thinks about budgets. Understanding them is essential before designing any API.

### Greenlit Budget — "the promise"

When a publisher decides to fund a game, they go through a **greenlight process**: producers present a P&L, a one-sheet, and a detailed budget; finance and executives review it; if approved, the budget is **greenlit**. That approved number becomes the contractual commitment — what the studio promised to deliver the game for. Once greenlit, the budget is **locked**. You cannot change it without going through a formal **amendment** workflow (the 3-checkbox-gated process documented earlier).

In the data model: every budget row has a `greenlit: {monthKey: amount}` map. The aggregate of these is `greenlitAll`. The version is tracked as `greenlitVersion` (1, 2, 3...) — every approved amendment bumps it.

### Live Budget — "what's actually happening"

The live budget is the **current operational forecast**. It starts as a copy of greenlit, but producers update it constantly as reality unfolds: a contractor charges more than expected, a department gets cut, a phase slips. Live is freely editable by anyone with edit permissions; it doesn't need approval.

In the data model: every row also has a `live: {monthKey: amount}` map. Aggregate = `liveAll`.

### Why both?

The whole financial discipline of the app is built around the **gap between live and greenlit**:

- **Variance** (`liveAll − greenlitAll`) tells you how much the project is drifting from its commitment.
- When variance exceeds 10%, the system surfaces an "Amendment Needed" prompt — the live forecast has drifted enough that you should formally re-baseline.
- The Payment Schedule shows this gap milestone-by-milestone so finance can see when cash needs will exceed what was approved.
- The P&L Projection always uses greenlit (the deal you're committing to) as the basis for break-even math.

So: **greenlit is the contract, live is reality, variance is the gap between them.** Every page in the app is some flavor of comparing those two surfaces.


