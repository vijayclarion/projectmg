# Fictions Folio — Comprehensive Business & Logic Analysis

**Source artifacts**
- Technical: `fictions-command-center-v0.4 (1).jsx` (~11,935 lines, single-file React 18 prototype, `APP_VERSION = "0.013d"`)
- Business: `Fictions-Folio-Developer-Brief.docx` (Version 0.013, dated April 2026)
- Application root: `AppRouter` → routes between `StudioApp` (default), `DevPortal` (when URL has `?portal=<id>&dev=<name>`), and a "Portal Preview" wrapper.

**Architecture in one paragraph.** `StudioApp` is the multi-project shell: it manages the studio record, user accounts, role-page mappings, and a `portfolio` of project IDs persisted to `window.storage` under `fcc_studio` and `fcc_proj_<id>` keys. When a user opens a project, `StudioApp` mounts `ProjectPanel` (the ~9,200-line monolith from line 642 to 9,921) which holds the 24 in-project views and all budget/financial state. `DevPortal` is a parallel, walled-off renderer that reads the same `fcc_proj_<id>` and `fcc_aux_<id>` keys read-only and writes only to a `changeLog` / `invoiceQueue` queue inside `aux`.

---

## Navigation Index

The application has three navigation tiers. The first table is the studio shell; the second is the per-project shell (the bulk of the application); the third is the developer-facing walled garden.

### Studio-level views (`StudioApp.studioView`)

| # | View key | Label | Section in this document |
|---|---|---|---|
| S1 | `portfolio` | Portfolio (project list, create/edit/delete) | [Studio: Portfolio](#studio-portfolio) |
| S2 | `insights` | Cross-project insights | [Studio: Insights](#studio-insights) |
| S3 | `sales` | Cross-project sales rollup | [Studio: Sales](#studio-sales) |
| S4 | `social` | Social / community signals | [Studio: Social](#studio-social) |
| S5 | `admin` | User management, role-page matrix, platforms (admin/finance only) | [Studio: Admin](#studio-admin) |

### Per-project views (`ProjectPanel.view`) — grouped as in the navigation bar

| # | Group | View key | Label |
|---|---|---|---|
| 1 | Home | `home` | Project Home → [analysis](#01-home) |
| 2 | Greenlight | `pnl` | P&L Projection → [analysis](#02-pnl) |
| 3 | Greenlight | `onesheet` | One Sheet → [analysis](#03-onesheet) |
| 4 | Greenlight | `amendments` | Amendments → [analysis](#04-amendments) |
| 5 | Budget | `dashboard` | Dashboard → [analysis](#05-dashboard) |
| 6 | Budget | `master_budget` | Contracted Budget → [analysis](#06-master_budget) |
| 7 | Budget | `budget` | Game Budget Grid → [analysis](#07-budget) |
| 8 | Budget | `mktg_budget` | Marketing Budget → [analysis](#08-mktg_budget) |
| 9 | Budget | `rollup` | Rollups → [analysis](#09-rollup) |
| 10 | Budget | `payment` | Payment Schedule → [analysis](#10-payment) |
| 11 | Budget | `variance` | Variance → [analysis](#11-variance) |
| 12 | Budget | `insights` | Budget Insights → [analysis](#12-insights) |
| 13 | Milestones | `phases` | Phases → [analysis](#13-phases) |
| 14 | Milestones | `milestones` | Milestones → [analysis](#14-milestones) |
| 15 | Ledger | `ledger_topsheet` | Ledger Top Sheet → [analysis](#15-ledger_topsheet) |
| 16 | Ledger | `ledger` | Game Ledger → [analysis](#16-ledger) |
| 17 | Ledger | `mktg_ledger` | Marketing Ledger → [analysis](#17-mktg_ledger) |
| 18 | Ledger | `reconcile` | Reconciliation → [analysis](#18-reconcile) |
| 19 | Ledger | `invoices` | Invoice Queue → [analysis](#19-invoices) |
| 20 | Sales | `sales` | Sales Dashboard → [analysis](#20-sales) |
| 21 | Sales | `sales_steam` / `sales_xbox` / `sales_ps` / `sales_switch` / `sales_epic` | Per-platform pages → [analysis](#21-sales_platform) |
| 22 | Sales | `sales_promos` | Sales Intelligence → [analysis](#22-sales_promos) |
| 23 | Notes | `notes_creative` / `notes_meeting` | Notes pages → [analysis](#23-notes) |
| 24 | Notes | `documents` | Documents library → [analysis](#24-documents) |
| 25 | History (admin overlay) | `history` | History & Audit Log → [analysis](#25-history) |
| 26 | Reports (legacy alias) | `reports` (auto-redirects to `insights`) | Cost Reports → [analysis](#26-reports) |

### Developer Portal tabs (`DevPortal.tab`)

| # | Tab key | Label | Section |
|---|---|---|---|
| D1 | `budget` | Budget Grid (proposes changes, never writes them) | [Dev Portal: Budget](#d1-devportal-budget) |
| D2 | `invoice` | Submit Invoice | [Dev Portal: Invoice](#d2-devportal-invoice) |
| D3 | `history` | My Submissions | [Dev Portal: History](#d3-devportal-history) |

---

## Cross-cutting calculation primitives

Almost every page in the application stands on the same handful of selectors. Documenting them once here lets every per-page section refer to them by name instead of re-derived line-by-line.

| Symbol | Definition (JSX line) | Meaning |
|---|---|---|
| `BUDGET_GROUPS` | const, lines 38–70 | Hard-coded three-tier hierarchy: **Group → Section → Subsection**. Three groups (`devcosts`, `porting`, `external`), six sections, eight subsections. Defines every row's permitted home. |
| `PHASES_DEFAULT` | const, lines 23–33 | Nine-phase template: Concept, Prototype, VS, Pre-Alpha, Alpha, Beta, RC, Post-Launch, Archive — used to seed both `project.phases` (live) and `project.glPhases` (greenlit). |
| `msLive` / `msGreenlit` / `ms` | `useMemo`, lines 1647–1649 | Month arrays built by `buildMonths(startDate, monthCount)`. `ms` is the active one based on `budgetMode`. |
| `sectionTotal(sId, mode, mKey)` | line 1660 | Sum of `row[mode][mKey]` across all rows in section `sId`. **Native-currency, raw value.** |
| `groupTotal(gId, mode, mKey)` | line 1663 | Same, summed across all sections in a group. |
| `grandTotal(mode, mKey)` | line 1668 | All rows for one month. |
| `cellRate(row, mKey)` | line 1748 | **FX cascade:** per-cell `row.rates[mKey]` → column `project.spotRates[mKey]` → global `project.globalSpotRate` → `1.0`. |
| `cellUSD(row, mode, mKey)` | line 1749 | Native cell value, multiplied by `cellRate` only if the row is flagged as alt-currency. |
| `blendedRate(mKey)` | line 1750 | Weighted-average FX rate for a month: `Σ(native × rate) / Σ(native)` across alt rows in the live budget. Used only for batch-rate UI. |
| `effSectionTotal` / `effGroupTotal` / `effGrandTotal` | lines 1775–1777 | The "effective" totals that the rest of the app renders: USD-converted versions when `multiCurrency` is on, raw native when off. **Every dashboard, P&L, payment-schedule, and reconciliation number ultimately resolves through one of these three.** |
| `rowTot(row, mode)` | line 1671 | Sum of one row across all months in the active timeline. |
| `liveAll` / `greenlitAll` | lines 1700–1701 | Sum of `grandTotal` across all months. **The "what is the budget" number used everywhere.** |
| `liveAllUSD` / `greenlitAllUSD` / `varianceUSD` | lines 1780–1782 | Multi-currency versions of the same. |
| `phaseFor(mKey)` | line 1651 | Returns the phase ID assigned to a month, mode-aware (`phaseAssignment` for live, `glPhaseAssignment` for greenlit). |
| `entryAllocated[id]` and `cellAllocated[rowId_monthKey]` | computed inside `view === "reconcile"`, lines 5905–5907 | The two pivot maps that drive reconciliation: how much of a ledger entry is allocated, and how much has been allocated to each grid cell. |

**The cascade is the single most important business rule in the app.** Almost every visible "USD total" is `eff*Total → cellUSD → cellRate → spotRate`. Misreading or breaking that cascade silently miscounts every KPI. The backend must reproduce it identically.

---

## 01. Home <a name="01-home"></a>

**JSX:** `view === "home"` (lines 3423–3623)

### Business Purpose

The single-screen "command center" landing page for a project. Producers, finance, and developers all open here first; it has to answer "is this project healthy, on schedule, and worth opening any other tab?" without scrolling. It is the most heavily KPI-dense view in the app and acts as a dispatcher into the rest of the navigation.

### Field Inventory

- **Header:** project name (`project.name`), studio (`project.studio`), current phase badge, milestone counter (`Milestone N of M`), Budget Health pill (On Track / At Risk / Over Budget).
- **Top KPI strip (6 cards, conditional on contracted budget existing):** Contracted Budget, Greenlit Budget, Live Budget, Ledger Spent to Date, Remaining, Variance.
- **Budget Progress bar:** percentage consumed plus elapsed-timeline percentage and average monthly burn.
- **Monthly Spend chart:** Recharts `BarChart` of `grandTotal("live")` and `grandTotal("greenlit")` per month.
- **Project Snapshot card:** Current Phase, Milestone, Ledger Entries count, Ledger Paid total, Budget Rows count, GL Versions count.
- **Sales Overview card:** Total Revenue, Units Sold, Wishlists, ROI, plus an "active promotions" badge.
- **Recent Notes card:** the five most recent notes (creative + meeting combined).
- **Current Milestone card:** monthly burn for the current month, plus build-info and notes excerpts.

### Data Logic & Lineage

| Field | Type | Source / formula |
|---|---|---|
| `currentPhase` | Derived | Finds the month object matching `today`, looks up `project.phaseAssignment[m.key]`, then resolves to `phases.find(p => p.id === phaseId).name`. |
| `currentMilestone` | Derived | `elapsedMonths` (the count of `msLive` whose `key + "-01"` is on or before today). Milestones in this app are 1-to-1 with calendar months. |
| `ledgerPaid` / `spentToDate` | Derived | `Σ ledger.totalBilled` (with `originalAmount` fallback). |
| `burnRate` | Calculated | `ledgerPaid / elapsedMonths`. |
| `remainingBudget` | Calculated | `greenlitAll − spentToDate`. |
| `pctComplete` | Calculated | `spentToDate / greenlitAll × 100`. |
| `budgetHealth` | Calculated (categorical) | If `pctComplete ≤ (elapsedMonths / msLive.length × 100) + 5` → "On Track". If `> 100` → "Over Budget". Otherwise "At Risk". |
| Top KPI strip (Greenlit / Live / Variance) | Derived | Direct from `greenlitAll`, `liveAll`, `variance` — see [primitives](#cross-cutting-calculation-primitives). |
| Contracted Budget card | Calculated | Sum of all six `project.masterBudget` line items: development + porting + external + reserve + contingency + marketing. **Only rendered when this sum > 0** — the page restructures itself when no contracted budget exists. |
| Sales card values | Derived | Aggregates of `salesData[]` and `wishlistData[]` (auxiliary state, lives in `aux` storage). ROI uses `(totalRevenue − greenlitAll) / greenlitAll`. |
| Active promotions | Derived | Filter on `promoData` where today's date falls in the promo window. |

### Gap Analysis

- **No gap.** This page is brief-aligned: it consolidates "Dashboard with KPI cards" plus snapshot of current state.
- One subtle behavior worth flagging to the backend team: Health is computed off `greenlitAll` (not contracted budget) — meaning the health pill silently changes if the greenlit version is amended. Backend must preserve this exact source of truth.

---

## 02. P&L Projection <a name="02-pnl"></a>

**JSX:** `view === "pnl"` (lines 7456–7805)

### Business Purpose

The deal-modeling cockpit. Inputs are the proposed budget (which may sync from Contracted Budget), publishing partner economics, retail pricing, platform/engine fees, and tiered royalty structures. Outputs are per-unit revenue waterfalls, break-even unit counts, scenario tables, and ROI targets. This is the page Greenlight committee approves against; everything in **One Sheet** is a re-render of these numbers in a presentation skin.

### Field Inventory

- **Costs card** (top-left): publishing-partner toggle (`hasPubPartner`), partner name (`pubPartnerName`), six budget inputs — `devBudget`, `portingBudget`, `externalBudget`, `reserveBudget`, `contingencyBudget`, `marketingBudget` — plus `upfrontEngineFee`, `publishingFeeRate`. Per-line **partner contribution** inputs (`pubContrib[key]`) appear when partner toggle is on.
- **Recoupment summary:** Fictions Recoupment, Partner Recoupment, two break-even cards (Fictions B/E units, Partner B/E units).
- **Fees & Pricing card:** `platformFee`, `engineFee`, `retailPrice` (SRP), `discountRate` (avg discount). Inline **per-unit revenue waterfall**: Retail → Avg Selling Price → minus platform fee → minus engine fee → minus licensor royalty → Net Revenue Per Unit.
- **Performance Targets table:** each row is one ROI target (`p.roiTargets`, e.g. break-even, 35% ROI), columns are units required and net profit at that point.
- **Developer Royalty Tiers card:** N tiers, each with a threshold type (`thType`), threshold value (`thValue`), developer share (`devShare`), publisher share (`pubShare`).
- **Licensor Royalty Tiers card:** present when `licensorRoyaltyEnabled`, same shape but only `licensorShare`.
- **Scenario table:** per `p.scenarioUnits` value, full P&L breakdown.
- **"Sync from Contracted Budget" button** (top-right): copies `masterBudget` keys into `glPnl` keys when they differ.

### Data Logic & Lineage

This page is mostly **calculated**; the inputs are the only static fields.

| Field | Source / formula |
|---|---|
| `pubFeeBase` | `devBudget + portingBudget + externalBudget + reserveBudget + contingencyBudget` (does **not** include marketing or upfront engine fee). |
| `publishingFee` | `pubFeeBase × publishingFeeRate`. |
| `totalCosts` (a.k.a. `recoupment`) | `pubFeeBase + marketingBudget + upfrontEngineFee + publishingFee`. **Total recoupable amount across both Fictions and the partner.** |
| `totalPartnerContrib` | `Σ pubContrib[k]` if partner enabled, else 0. |
| `fictionsRecoupment` | `totalCosts − totalPartnerContrib`. **The number Fictions must recover from net revenue.** |
| `avgPrice` | `retailPrice × (1 − discountRate)`. |
| `afterPlatformPerUnit` | `avgPrice × (1 − platformFee − engineFee)`. |
| `licensorRoyaltyPerUnit` | If enabled, `afterPlatformPerUnit × (first licensor tier share)`. |
| `netPerUnit` | `afterPlatformPerUnit − licensorRoyaltyPerUnit`. **Marginal revenue Fictions and the developer split per copy sold.** |
| `calcPnl(units)` (the central function, line 7473) | For a unit count: computes gross revenue, after-platform, then runs a **tiered licensor royalty waterfall**, then a **tiered dev/pub royalty waterfall**, then `profit = netRev − fictionsRecoupment − devRoyalties − pubRoyalties`. |
| Tier ceilings | Per tier `t`: `multi` → `recoupment × t.thValue`; `fixed` / `gnr` → `t.thValue`; `mg` → `t.thValue / licensorShare`; `onwards` → infinity. |
| `findUnitsForROI(target)` | 80-iteration binary search on units to hit `profit / fictionsRecoupment = target`. |
| Performance Targets table | Maps `p.roiTargets` through `findUnitsForROI`, then evaluates `calcPnl` at that unit count. |
| Scenario table | Maps each `p.scenarioUnits` through `calcPnl`. |
| Break-even cards | `fictionsBE = ceil(fictionsRecoupment / netPerUnit)`. Partner B/E is a separate binary search that sums tiered publisher royalties until they equal `partnerContrib`. |
| "Sync from Contracted Budget" indicator | Compares each `masterBudget` key to its `glPnl` analog and shows a yellow pill if any differ. |

### Gap Analysis

- **Brief is fully reflected.** All required mechanics are present: tiered dev royalties (recoup multipliers, fixed amounts, GNR thresholds, onwards), tiered licensor royalties calculated on after-platform revenue, publishing partner contributions and royalty splits, scenario analysis, ROI targets.
- **Implementation note for backend:** the recoupment ceiling for tier-based math uses the **pre-partner-contribution** `recoupment` (`totalCosts`), but the profit calculation deducts `fictionsRecoupment` (post-partner). This is intentional — tiers are sized by total deal value, profit is measured against Fictions's exposure — but it must be ported exactly or every greenlight P&L will silently disagree with the prototype.
- **Missing per spec table comparison:** None.

---

## 03. One Sheet <a name="03-onesheet"></a>

**JSX:** `view === "onesheet"` (lines 7807–8079)

### Business Purpose

A printable, presentation-grade summary of the deal — the document used in greenlight meetings and to share with partners. Reads from `glOneSheet` for narrative text fields and **re-derives all economic numbers from `glPnl`** so the One Sheet can never drift from the P&L Projection.

### Field Inventory

- **Header:** `projectName`, `developer`, `logline`, `targetStartDate`, `targetShipDate`, `targetPlatforms` (multi-select dropdown of 13 platforms).
- **Budget / Costs table:** rows = the seven cost lines from P&L (Dev / Porting / External / Reserve / Contingency / Marketing / Publishing Fee / Upfront Engine Fee), columns = Fictions, Publisher, Total (only shown when `hasPubPartner`).
- **Pricing summary:** Retail SRP, Avg Selling Price.
- **Performance summary:** Break-Even, 30% ROI units.
- **Bullet sections** (each editable, free-form): typically Goals, Risks, Differentiators, Key Deliverables, etc. (`BulletSection` component, line 7851).
- **Print mode** (`osPrinting`): switches to a static, A4-friendly render.

### Data Logic & Lineage

| Field | Source |
|---|---|
| Project name, developer | Static — falls back to `project.name` and `project.studio` if unset. |
| `logline`, dates, platforms, bullets | Static — stored in `glOneSheet`. |
| All numeric fields (budgets, fees, recoupment, break-even, ROI) | **Calculated** — re-runs the full P&L math from `glPnl` (lines 7813–7846 mirror the P&L view). |
| `partnerBE2` | Binary search on units to find when cumulative publisher royalties equal `partnerContrib`. |
| 30% ROI cell | Inline binary search (line 7986) — same logic as P&L's `findUnitsForROI(0.3)`. |

### Gap Analysis

- **No functional gap.** Brief asks for "One Sheet" as part of the greenlight package; the page satisfies it.
- **Code smell, not a gap:** the P&L math is duplicated between `pnl` and `onesheet` views (and copied a third time in the print modal). When the backend builds an exports endpoint, the math should live in **one** server-side function the API serves to both screens.

---

## 04. Amendments <a name="04-amendments"></a>

**JSX:** `view === "amendments"` (lines 6685–7070)

### Business Purpose

The change-control workflow for greenlit budgets. Once a budget is greenlit and locked at version *N*, any material change requires a formal amendment: producer drafts → finance reviews → approval bumps the version to *N+1* and snapshots the previous one. This page also surfaces **pending budget changes from developers** (which are a softer queue and don't require a new GL version).

### Field Inventory

- **Status header:** GL version, GL date, total amendment count, count of pending budget changes.
- **Amendment History table:** columns = Initial Deal, Amendment 1, Amendment 2, …, Current Live. Rows = the three budget groups + Total. Cells highlight in red/green when they change between consecutive amendments.
- **Budget Change Reviews card** (when developer has submitted changes): one card per pending batch with submitter, timestamp, line-item delta table, Approve / Reject buttons.
- **Amendment Draft form** (when a draft is open):
  - Reason (required textarea)
  - Term changes notes
  - **3-checkbox amendment gate:** (1) Update Contracted Budget line items; (2) Export new P&L; (3) Export new One Sheet. Submit button is disabled until all three are checked.
- **Pending Finance Approval list:** one card per submitted amendment with reason, current GL total, proposed total, delta, term changes, Approve / Reject / Delete buttons (gated by `perms.approveAmendments`).
- **Approved amendment history:** read-only cards.

### Data Logic & Lineage

| Field | Source |
|---|---|
| `liveTot` / `glTot` / `variance` | Direct from `grandTotal` over `msLive` and `msGreenlit`. |
| `varPct` | `variance / glTot × 100`. The status bar drives the "Amendment Needed" chip elsewhere when `|varPct| > 10`. |
| Amendment History table values | For each approved amendment, `groupDeltas[].greenlit` (the prior version's group total) and `groupDeltas[].proposed` (the new version's). The **Initial Deal** column reads from the first approved amendment's `greenlit` field. The **Current Live** column reads from `groupTotal(g.id, "live", m.key)` — i.e. *current state of the live grid*, not the latest amendment. |
| `pendingBudgetChanges` | `changeLog.filter(b => b.status === "pending")` — these come from the **Reconciliation page** (when adjusting cells during recon) and from the **Developer Portal** (queued via `aux.changeLog`). |
| Approval action | On approve: bumps `glVersion`, copies the amendment's proposed group totals into the greenlit grid, snapshots into `greenlitHistory`, logs an audit entry. |
| Submit button gate | All three checklist booleans + non-empty reason. |

### Gap Analysis

- **Implemented as specified.** The "3-checkbox gate" is exactly the brief's requirement.
- **One important nuance:** approved amendments capture group-level totals only (`groupDeltas`). Cell-level greenlit values are still in `greenlit{}` on each row. This means the system trusts that cell-level edits done while drafting an amendment match the proposed group totals — there's no validation step. Backend should enforce a check at the API layer.

---

## 05. Dashboard <a name="05-dashboard"></a>

**JSX:** `view === "dashboard"` (lines 3625–3683)

### Business Purpose

Compact financial dashboard subordinate to Home — emphasizes greenlit-vs-live trajectory more than current-state KPIs. This is the page producers refresh during weekly burn reviews.

### Field Inventory

- **5 KPI cards:** Contracted Budget (conditional), Greenlit Total, Live Total, Variance, Ledger Paid.
- **Cumulative Spend area chart:** Greenlit vs Live, cumulative.
- **Spend by Section pie chart:** colored by `BUDGET_GROUPS[].color`.
- **Monthly Greenlit vs Live bar chart:** per-month side-by-side.

### Data Logic & Lineage

| Field | Source |
|---|---|
| Contracted Budget | Sum of all six `masterBudget` keys (same formula as Home). |
| Greenlit Total | `greenlitAll`. |
| Live Total | `liveAll`, also rendered as `% of greenlit`. |
| Variance | `liveAll − greenlitAll`. |
| Ledger Paid | `ledgerTotal` = `Σ project.ledger.totalBilled`. |
| `cumChart` data | `useMemo` running cumulative through `ms.map(grandTotal)` — line 2871. |
| `pieData` | One slice per `BUDGET_GROUPS` with value = sum of `groupTotal("live")` across `ms`. |
| `monthlyChart` | Per-month `{live, greenlit}` from `grandTotal`. |

### Gap Analysis

- No gap. Brief's "Dashboard with KPI cards, monthly burn chart, cumulative chart, spend by section, spend by phase" — phase chart is on the Insights page, not here. The split is fine; just note that "spend by phase" is on `insights`, not `dashboard`.

---

## 06. Contracted Budget (Master Budget) <a name="06-master_budget"></a>

**JSX:** `view === "master_budget"` (lines 3685–3803)

### Business Purpose

The **source of truth for contractual cost ceilings** — i.e. what was negotiated in the deal memo. These are the numbers the P&L Projection's "Sync" button reads from. Live grid totals are surfaced alongside as a sanity check ("Grid: $X" and "Var: $Y") but are not authoritative; this page lets producers adjust the contracted figures and see whether the bottom-up grid still reconciles.

### Field Inventory

- **KPI Banner:** Total Project Costs, Game Budget.
- **6 line-item cards** (each editable):
  - Development Costs (linked to `devcosts` group)
  - Porting Costs (linked to `porting` group)
  - External Support Costs (linked to `external` group)
  - Reserve (no grid link)
  - Contingency (no grid link)
  - Marketing Costs (no grid link to game grid; linked to Marketing Budget instead)
- Each card shows: contracted value (editable), grid total (read-only), variance (live − contracted), percentage of total project costs, and a progress bar.
- **Summary table:** per-line dollar amounts and percentages; subtotals for Game Budget and Total Project Costs.

### Data Logic & Lineage

| Field | Source |
|---|---|
| Six contracted values | Static — stored in `project.masterBudget.{developmentCosts, portingCosts, externalCosts, reserve, contingency, marketing}`. Default seed: 6M / 600K / 1.4M / 250K / 1M / 500K (line 133). |
| `liveByGroup[grpId]` | Calculated: sum across the group's sections of `Σ row.live[m.key]` over all months. |
| `gameBudget` | `developmentCosts + portingCosts + externalCosts`. |
| `totalProjectCosts` | `gameBudget + reserve + contingency + marketing`. |
| `lineVar` | `gridVal − contracted` for the three groups that have a grid link. |
| Progress bar percentage | `contracted / totalProjectCosts × 100`. |

### Gap Analysis

- No gap. **One subtle business rule worth surfacing:** Marketing's "grid value" is shown as `null` on this page — it has no comparison line. The marketing comparison happens on the Marketing Budget page itself (`Allocated` vs `Contracted`). Backend should preserve the asymmetry: `Marketing` lives in its own grid, the other three groups in the main game grid.

---

## 07. Game Budget Grid <a name="07-budget"></a>

**JSX:** `view === "budget"` (lines 4061–4504)

### Business Purpose

The bottom-up cost-planning grid — the **heart of the application**. Producers and developers populate hundreds of rows × 36 months of forecast data here. Two parallel modes (Live vs Greenlit) live side-by-side; the live grid is amendable, the greenlit grid is locked except during an active amendment. This is also where multi-currency cells get tagged and FX rates get set.

### Field Inventory

#### Toolbar
- Mode toggle: Live Budget / Greenlit Budget (`budgetMode` state).
- Currency view toggle (visible only when `multiCurrency`): USD / native (`currencyView`).
- Greenlit version badge: "GL v{n}", last greenlit date.
- Buttons: Fullscreen toggle, Hide Cols toggle, Export, Import CSV (perm-gated), Export Snapshot dropdown (PDF), "Open New Greenlight" / "Save Greenlight" / "Cancel" depending on amendment state.

#### Grid
- **Frozen left columns:** row label fields (`role`/`item`, `department`/`category`, `name`/`detail`) — defined per subsection in `BUDGET_GROUPS[].sections[].subsections[].fields`.
- **N month columns** (`ms.length`, default 36): each cell is a `BudgetCell` (line 213).
- **Year-total columns:** auto-inserted between Decembers and Januaries.
- **Total column:** row total across all months.
- **Group/section/subsection header rows:** sticky, expandable, show running totals.
- **Phase bar:** colored band above the month headers showing which months belong to which phase.
- **Closed-month indicators:** lock icons + grey shading for months in `project.closedMonths`.

#### Inline FX panel (visible when `fxBatchMonth` is set)
- Per-row table of: Native amount, Rate, USD — all editable.
- Batch apply: one rate to all selected rows, or one USD total to all selected rows (back-solves the rate).
- Multi-select via row checkboxes.

#### Selection / context features
- Click-drag to select cells; multi-cell paste from Excel (clipboard parser).
- Right-click context menu: cut/copy/paste/clear/delete row.
- Bulk action bar (visible when multiple rows selected): copy values, fill across, delete rows.
- Row drawer (visible when `sel` is set): per-cell history, comments (`cellComments[rowId_monthKey]`).

### Data Logic & Lineage

This is the most data-heavy view. Most fields are **static** at the cell level; everything else is rolled up.

| Field | Source / formula |
|---|---|
| Cell value | Static — `row[mode][mKey]` (number). |
| Cell display value | If `currencyView === "usd"` and row is alt-currency, computes `value × cellRate(row, mKey)`; otherwise the raw value. |
| Cell rate (FX) | `cellRate` — see [primitives](#cross-cutting-calculation-primitives). |
| Variance highlighting (red over / green under) | Per cell: compares live vs greenlit; cell border or background tinted. The status bar uses `variance` aggregated. |
| Section / group / row totals | `sectionTotal`, `groupTotal`, `rowTot` — applied per month, per year, and grand total. |
| Year totals | `yearTotal` and `yearGroupTotal`. |
| Closed-month edits | Calls `updateCell`, which checks `closedMonths[mKey]` and either applies (if open) or surfaces a warning modal. |
| Phase bar widths | `project.phaseAssignment` mapped through phase colors. |
| Greenlit edit gate | `greenlitEditing` state — set by the amendment workflow. While locked, all greenlit cells are `readOnly`. |

### Gap Analysis

- **Brief-aligned in detail.** Frozen columns, collapsible sections, phase bars, milestone numbering, alternating row colors, keyboard nav, multi-cell Excel paste — all present.
- **Multi-currency:** brief specifies "single alt-currency". The implementation matches — there is one alt currency per project, set on the Settings modal; rows are individually flagged via `rowCurrencyOverrides`.
- **Closed months:** `closedMonths[mKey] = {closedAt, closedBy}` — implemented exactly as specified.
- **No gap.**

---

## 08. Marketing Budget <a name="08-mktg_budget"></a>

**JSX:** `view === "mktg_budget"` (lines 3806–4058)

### Business Purpose

The marketing-spend planner, deliberately separated from the main game budget. Marketing has a different category structure (PR, Media, Creative, Events, etc.) and different stakeholders. A Contracted Marketing Budget sets the ceiling; sub-categories slice that into line items planned month-by-month.

### Field Inventory

- **4 KPI cards:** Contracted Budget, Allocated (sum of category budgets), Planned Spend (sum of monthly cells), Remaining.
- **Tab toggle:** "By Category" view (one card per category, with row editor) vs "Full Grid" view (matrix).
- **Category card (per category):** name (editable), category budget (editable), planned vs budget bar, list of line items.
- **Line item:** item name (editable), monthly cells (editable).
- **Default categories** (auto-seeded if empty): Creative Materials, Promotional Materials, Media Spend, Marketing T&E, PR Expenses, Events & Showcases, Other Marketing (line 319).

### Data Logic & Lineage

| Field | Source |
|---|---|
| Categories | Static — `project.marketingBudget.categories[]` = `[{id, name, budget}]`. |
| Line items | Static — `project.marketingBudget.rows[]` = `[{id, categoryId, item, monthly: {mKey: number}}]`. |
| Contracted Budget KPI | `project.masterBudget.marketing` — coming from the Contracted Budget page. |
| Allocated KPI | `Σ categories[].budget`. |
| Planned Spend KPI | `Σ rows[].Σ monthly[mKey]`. |
| Remaining KPI | `(masterMktg || allocated) − planned`. |
| Per-category Planned | `Σ catRows.Σ monthly[mKey]`. |
| Per-category Variance | `catPlanned − category.budget`. |
| Auto-seed migration | If categories are empty OR contain old names (PR, Paid Media, etc.) without new ones, the page re-seeds with `DEFAULT_MKTG_CATS`. |

### Gap Analysis

- **No gap.** Brief specifies "Marketing Budget as a separate grid with its own category structure" — implemented.
- **Note for backend:** the auto-seeding migration (line 3812) is a client-side patch that should not survive into production. The backend should seed categories on project creation and never mutate them silently afterward.

---

## 09. Rollups <a name="09-rollup"></a>

**JSX:** `view === "rollup"` (lines 5189–5337)

### Business Purpose

Two read-only summary tables at different aggregations: by **calendar year** and by **development phase**. Used in board packs and investor reports — proves where the money goes over time and over the work.

### Field Inventory

- **Annual Rollup table:** rows = budget groups → sections → subsections; columns = each year in the timeline + Total.
- **Phase Rollup table:** rows = same; columns = each phase + Total.

### Data Logic & Lineage

Every cell is **calculated**.

| Field | Source |
|---|---|
| Subsection × year cell | `Σ rowsInSubsection.Σ live[m.key]` for months `m.year === yr`. |
| Section × year cell | `yearTotal("live", yr, sec.id)`. |
| Group × year cell | `yearGroupTotal("live", yr, grp.id)`. |
| Grand total × year | `yearTotal("live", yr)`. |
| Subsection × phase cell | Filter months by `project.phaseAssignment[m.key] === pId`, then sum. |
| Section × phase cell | `phaseTotal("live", p.id, sec.id)`. |
| Group × phase cell | `phaseGroupTotal("live", p.id, grp.id)`. |

### Gap Analysis

- No gap. Brief explicitly calls for "Rollups (summary views across cost groups)".

---

## 10. Payment Schedule <a name="10-payment"></a>

**JSX:** `view === "payment"` (lines 5340–5382)

### Business Purpose

Milestone-by-milestone payment ledger that compares what was promised at greenlight against what is now planned. Used to manage developer milestone payments and surface upcoming cash needs.

### Field Inventory

Single table with six columns:
- Milestone (e.g. "MS 1 — Aug 25")
- Phase (color-tagged badge)
- Greenlit (`effGrandTotal("greenlit", m.key)`)
- Live (`effGrandTotal("live", m.key)`)
- Difference (Live − Greenlit, red if positive, green if negative)
- Cumulative Difference (running sum)

### Data Logic & Lineage

| Field | Source |
|---|---|
| Milestone label | `MS {idx+1} — {month label}`. |
| Phase | `project.phases.find(id === phaseFor(m.key))`. |
| Greenlit / Live | `effGrandTotal` (USD-aware via the FX cascade). |
| Difference | `live − greenlit` per milestone. |
| Cumulative Difference | Running sum of `difference` from milestone 1. |

### Gap Analysis

- No gap. Brief: "Payment Schedule (milestone-by-milestone with greenlit vs. live comparison)" — exact match.

---

## 11. Variance <a name="11-variance"></a>

**JSX:** `view === "variance"` (lines 6298–6408)

### Business Purpose

The **5-column variance report** explicitly called out in the brief. Splits live spend into "elapsed" (actuals to date, taken from the live grid for past months) vs "remaining" (still-future projected) and compares the projected total against the locked greenlit total.

### Field Inventory

- **5 KPI cards:** Actuals to Date, Remaining Projected, Projected Total, Greenlit Total, Variance.
- **Variance by Cost Section table:** rows = group rows + section detail rows; five numeric columns matching the KPIs.
- **Footnote:** notes that "Actuals to Date" comes from the live grid, not the ledger; provides ledger total for reference.

### Data Logic & Lineage

| Field | Source |
|---|---|
| `elapsedMonths` | Count of months whose `key + "-01"` is on or before today. |
| `actuals` (per group/section) | `Σ groupTotal/sectionTotal("live", m.key)` for months indexed `0..elapsedMonths−1`. |
| `remaining` | Same sum for months indexed `elapsedMonths..end`. |
| `projected` | `actuals + remaining`. |
| `greenlit` | `Σ groupTotal/sectionTotal("greenlit", m.key)` over all months. |
| `variance` | `projected − greenlit`. |
| `ledgerActuals` (footnote only) | `Σ ledger.totalBilled`. |

### Gap Analysis

- **Brief-aligned exactly.** Five columns named the way the brief specifies them: Actuals to Date, Remaining Projected, Projected Total, Greenlit Total, Variance.
- **Important business semantic:** "Actuals to Date" here is **forecasted live spend for past months**, not invoiced/paid amounts. The footnote acknowledges this and gives the ledger total separately. Some finance teams may interpret "actuals" as "ledger-paid" — backend should expose both as separate fields.

---

## 12. Insights (Budget Insights) <a name="12-insights"></a>

**JSX:** `view === "insights"` (lines 4505–4789)

### Business Purpose

The deep analytics page — five-card KPI overview, budget-health bar comparing spend pacing to timeline pacing, and a dense gallery of charts: cumulative spend, monthly burn, spend by section (pie), spend by phase (horizontal bars), spend by department, top 10 line items, top 5 over-budget rows. This is where producers diagnose budget problems in detail.

### Field Inventory

- **KPI strip (5 cards):** Contracted (conditional), Total Budget (live with greenlit subtitle), Ledger Spent to Date, Remaining, Avg Burn Rate (with projected total).
- **Budget Health bar:** spend percentage with a "Now" timeline marker overlay.
- **Cumulative Spend chart:** Live vs Greenlit area.
- **Monthly Burn chart:** Live vs Greenlit bars.
- **Spend by Section** pie + legend.
- **Spend by Phase** horizontal bars with variance percentages.
- **Spend by Department** horizontal bars (uses department colors).
- **Top 10 Line Items** list (live total, greenlit total, variance).
- **Top 5 Over-Budget Rows** alert list.

### Data Logic & Lineage

| Field | Source |
|---|---|
| `elapsedCount`, `pctTimeline`, `burnRate`, `projectedTotal`, `remainingMonths`, `remainingBurnRate` | All derived from current date and `msLive.length`. |
| `rowSpends` | Project rows, each annotated with `total = Σ live[m.key]`. |
| `groupSpends` | Per group: live total, greenlit total. |
| `deptSpends` | `{ dept: Σ live[m.key] }` aggregated across rows by `meta.department || meta.category`. |
| `phaseSpends` | Per phase: filter `msLive` by `phaseAssignment === p.id`, then sum. |
| `overBudgetRows` | Top 5 rows where `liveTotal > greenlitTotal && greenlitTotal > 0`, sorted by `diff`. |
| `cumData`, `burnData` | Running totals across `msLive`. |

### Gap Analysis

- **Note for backend:** the Reports view (`view === "reports"`) auto-redirects to `insights` (line 667). Backend should not need to expose two endpoints — this is a UI-only alias.

---

## 13. Phases <a name="13-phases"></a>

**JSX:** `view === "phases"` (lines 4955–5187)

### Business Purpose

Dual-mode phase editor: one timeline for live (always editable), one for greenlit (locked except during amendments). Each phase has a name, color, duration in months, and an editable description. Phases are ordered, drag-to-reorder, and assignment to specific months is computed automatically based on order + duration.

### Field Inventory

- **Mode toggle:** Live Phases / Greenlit Phases.
- **Sync button:** copy live↔greenlit (with confirmation).
- **Lock/unlock banner** (greenlit-only).
- **Allocation summary:** Allocated / Total months, with over/under indicator.
- **Visual timeline bar:** colored segments proportional to phase duration.
- **Per-phase row (drag-handle, color picker, name input, month-count input, period display, expand/collapse, delete):**
  - Expanded panel: Duration, Period, Live-budget total for the phase, Description textarea (multiline).

### Data Logic & Lineage

| Field | Source |
|---|---|
| Phase definitions | Static — `project.phases[]` (live) and `project.glPhases[]` (greenlit). |
| Phase month assignment | `project.phaseAssignment` and `project.glPhaseAssignment` — computed by `buildPhaseAssignment(phases, monthKeys)` (line 81). Maps `monthKey → phaseId` by walking phases in order, consuming `phase.months` months each. |
| `phaseRanges[].startMonth/endMonth` | Cursor walks through `currentMs` based on cumulative phase months. |
| Per-phase Live Budget total | `Σ grandTotal("live", m.key)` for months in the phase's index range. |
| `phasesLocked` | `phaseMode === "greenlit" && !greenlitEditing && !glPhasesUnlocked`. |
| Sync direction | "from greenlit": copies `glPhases` → `phases`; rebuilds `phaseAssignment`. |

### Gap Analysis

- **No gap.** The brief talks generally about phase management; the dual-mode editor is the most thorough possible interpretation.

---

## 14. Milestones <a name="14-milestones"></a>

**JSX:** `view === "milestones"` (lines 8079–8202)

### Business Purpose

Per-month editable milestone records — one row per month of the live timeline. Each milestone holds free-form notes about the build (build version, branch), status notes (deliverables/blockers), and a Dropbox folder link. The numbers (burn, LTD spend, remaining) are derived from the budget grid.

### Field Inventory

Per milestone:
- Milestone number, month label, phase badge.
- Monthly Burn (`effGrandTotal("live", m.key)`).
- LTD Spend (`Σ effGrandTotal("live", mm.key)` for months 0..idx).
- Remaining Budget (`totalBudget − ltdSpend`).
- Expanded panel:
  - Three KPI cards (same as above).
  - Progress bar.
  - Build Information textarea.
  - Milestone Notes textarea.
  - Dropbox Folder URL input + open link.

### Data Logic & Lineage

| Field | Source |
|---|---|
| Phase | Resolved via `project.phaseAssignment[m.key]`. |
| Monthly Burn | `effGrandTotal("live", m.key)`. |
| LTD Spend | Cumulative running total of monthly burn through this milestone. |
| Remaining Budget | `totalBudget − ltdSpend` where `totalBudget = Σ effGrandTotal("live")`. |
| Build Info / Notes / Dropbox Folder | Static — `milestoneData[m.key].{buildInfo, notes, dropboxFolder}`. |

### Gap Analysis

- The brief mentions "milestone numbering" as a budget-grid feature — this page is a richer counterpart to that, not in conflict with it.
- No gap.

---

## 15. Ledger Top Sheet <a name="15-ledger_topsheet"></a>

**JSX:** `view === "ledger_topsheet"` (lines 5702–5896)

### Business Purpose

The single page where finance/management can see, per category group, **how much we've spent vs. how much we agreed to spend**. Drives the Top Sheet that often gets exported for executive review. Combines master budget allocations, marketing sub-budgets, and ledger spend into one drill-down table.

### Field Inventory

- **4 KPI cards:** Total Budgeted, Total Spent, Remaining, % Utilized.
- **Category breakdown table:** rows = top-level groups → expandable sub-categories → expandable nested items; columns = Entries count, Budgeted, Spent, Remaining, Utilization (bar + %).

### Data Logic & Lineage

| Field | Source |
|---|---|
| Group entries | `ledger.filter(e => CAT_TO_GROUP[e.category] === cat.group)`. |
| `groupSpent` | `Σ entries.totalBilled`. |
| `groupBudgeted` | Special-cased per group: Marketing → sum of `marketingBudget.categories[].budget` (fallback to `masterBudget.marketing`); Reserve → `masterBudget.reserve`; Contingency → `masterBudget.contingency`; otherwise `masterBudget.{developmentCosts, portingCosts, externalCosts}`. |
| Sub-category subs[] | Per item in `cat.items`: filter entries by `category === subKey || CAT_TO_SUBGROUP[e.category] === subKey`. |
| Sub `budgeted` | Marketing sub-cat: matching `marketingBudget.categories[].budget`; otherwise 0. |
| Uncategorized | `groupSpent − Σ subs.spent`. |
| Utilization bar | `min(100, spent/budgeted × 100)`. Color: green ≤ 80, yellow ≤ 100, red > 100. |

### Gap Analysis

- **Strong page; the only gap is conceptual:** the "ledger" here is the unallocated dollar total (`totalBilled`), not the *allocated to grid* total. So a fully-allocated entry and an unallocated entry both count the same way against budgeted. Backend should expose both views — Top Sheet by ledger, Top Sheet by grid allocations — and let the UI choose.

---

## 16. Game Ledger <a name="16-ledger"></a>

**JSX:** `view === "ledger"` (lines 5384–5701, shared with `mktg_ledger`)

### Business Purpose

The transactional record of money spent — one row per invoice or expense entry. Supports inline editing of every field, sortable columns, sub-tabs by category group, multi-currency original amounts, file attachments, status tracking, and CSV/QuickBooks import with duplicate detection (fingerprinting).

### Field Inventory

Per ledger entry (`project.ledger[]`):
- Reconciliation status icon (✓ fully allocated, ◯ partial, blank none) — derived
- Invoice number
- Payee (vendor)
- Original Amount (in original currency)
- Original Currency (USD/EUR/GBP/...)
- Total Billed (USD)
- Status (Draft / Sent to Finance / Paid / Disputed — color-coded)
- Date / Invoice Date
- Category (with picker + LedgerCategorySelect, line 343)
- Description / Memo
- Attachment (file)
- Allocations (array; each allocation has `rowId`, `monthKey`, `amount`, optional `phase`)

Toolbar: Sub-tabs (All + LEDGER_TABS or MKTG_TABS), Add Row, Import QB Export, Scan Invoice (AI), Export.

### Data Logic & Lineage

| Field | Source |
|---|---|
| Most fields | Static — entered manually or imported. |
| Reconciliation status | Calculated per row: `allocated >= total - 0.01` → fully allocated. |
| Total Billed (USD) | If currency is USD, equals Original Amount. Otherwise typed in or computed via FX. |
| Filtered entries | `preFiltered` (split by mktg vs game via `CAT_TO_GROUP[category]`), then by sub-tab if any. |
| `totalBilledSum` | `Σ filteredEntries.totalBilled`. |
| Sort | By column with asc/desc toggle. |
| Duplicate detection on import | "Fingerprinting" — payee + amount + invoice number + date hash. Imported entries enter a separate `reconItems` queue first. |
| AI invoice extraction | `Anthropic Claude Sonnet API` called with the uploaded PDF/image; returns `{vendor, invoiceNumber, date, lineItems[], total, currency}`. |

### Gap Analysis

- **Brief mentions "ledger with sortable columns, category tabs, search/filter, and inline editing"** — search/filter is partial: the page has sub-tabs but no free-text search input. Backend should expose a `search?q=` API even if the UI catches up later.
- **Brief mentions "AI-powered extraction (Anthropic Claude API)"** — implemented but with the API key client-side. Brief explicitly says this needs to move server-side.

---

## 17. Marketing Ledger <a name="17-mktg_ledger"></a>

**JSX:** `view === "mktg_ledger"` (same code path as `ledger`, branched at line 5385 with `isMarketingLedger = true`)

### Business Purpose

Same transaction grid as Game Ledger but pre-filtered to entries where `CAT_TO_GROUP[category] === "Marketing Costs"`. Shows the seven marketing sub-tabs from `MKTG_TABS` (line 5386) — Creative Materials, Promotional Materials, Media Spend, Marketing T&E, PR Expenses, Events & Showcases, Other Marketing.

### Field Inventory & Logic

Identical to Game Ledger; only differences:
- Heading: "Marketing Ledger".
- `preFiltered` keeps only marketing-category entries.
- Sub-tabs: `MKTG_TABS` instead of `LEDGER_TABS`.

### Gap Analysis

- No gap. Same notes about search and AI key apply.

---

## 18. Reconciliation <a name="18-reconcile"></a>

**JSX:** `view === "reconcile"` (lines 5898–6296)

### Business Purpose

The **most operationally important page in the financial workflow.** Every ledger entry must be allocated to one or more `(rowId, monthKey)` budget cells before a month can be closed. This page is where that mapping happens. It supports auto-match by amount, manual click-to-allocate, "quick allocate by subsection" (proportional split), and per-month closing.

### Field Inventory

- **4 KPI cards:** Unallocated count, Allocated total / Ledger total, Months Closed N/M, Matched count.
- **STEP 1 — Select Invoice:** filterable list of unallocated entries (tabs by invoice month).
- **STEP 2 — Allocation Grid:** appears when an invoice is selected. Shows budget rows for the selected month; each row has Budgeted / Already Allocated / Pending input.
- **Pending allocation footer:** Pending Total, Difference vs invoice remaining, Commit / Cancel buttons.
- **Mismatch confirmation modal:** if pending ≠ remaining, asks whether to (a) adjust the grid to match, or (b) commit anyway.
- **Quick allocate by subsection** dropdown: distributes invoice remainder proportionally across rows in a subsection.
- **Month close controls:** per-month "Close" / "Reopen" buttons (audit-logged).

### Data Logic & Lineage

| Field | Source |
|---|---|
| `allAllocations` | Flatten of `ledger[].allocations[]`. |
| `cellAllocated[rowId_monthKey]` | `Σ allocations.amount` keyed by row+month. |
| `entryAllocated[ledgerId]` | `Σ entry.allocations.amount` per entry. |
| `unallocated` | Entries where `total > 0 && allocated < total - 0.01`. |
| `fullyAllocated` | Entries where `allocated ≥ total - 0.01`. |
| `pendingAlloc` (transient) | `{rowId: amount}` map of in-progress allocation. |
| `quickAllocSubsection` | For a subsection: total budgeted = `Σ rows.live[selMonth]`; each row gets `budgeted/totalBudgeted × selRemaining` proportional share. |
| Commit | If `adjustGrid` is true, also bumps `row.live[mKey]` to match total allocations and creates a `changeLog` batch (which then needs separate approval — surfaces on the Amendments page). |

### Gap Analysis

- **Brief specifies "auto-match by amount"** — partially implemented via the QB import duplicate-detection path. The reconcile page itself doesn't have a one-click "match all by amount" button — it requires manual selection of each invoice. Backend should add an API endpoint that returns auto-match suggestions.
- **The "adjust grid" path on commit creating a `changeLog` batch is a subtle but important business rule**: reconciliation can mutate the live budget, but only through the formal review queue. Producers can't bypass review by hiding the change inside an allocation.

---

## 19. Invoice Queue <a name="19-invoices"></a>

**JSX:** `view === "invoices"` (lines 7071–7141)

### Business Purpose

Holding pen for invoices uploaded by anyone (developers in the portal, producers via the ledger page). Invoices wait here until a reviewer (perm: `approveInvoices` or `editLedger`) clicks Review, sees the AI-extracted data, edits it, and either approves it (creating a ledger entry) or rejects it.

### Field Inventory

- **Pending list:** uploaded file icon, file name, submitted-by, submitted-at, status pill, Review button.
- **Processed list:** same fields, plus reviewed-by and reviewed-at, plus approve/reject status.
- **Upload Invoice button:** file picker (PDF/image) — gated by `submitInvoices` or `editLedger` permission.
- **Invoice Review Modal** (when item is being reviewed): per-line-item editor — vendor, invoice number, date, currency, line items each with description/amount/include checkbox; the auto-extracted JSON is editable before commit.

### Data Logic & Lineage

| Field | Source |
|---|---|
| `invoiceQueue` | Static auxiliary state; persisted to `aux.invoiceQueue` storage key. |
| `pending` / `processed` | Filter on `q.status === "pending"`. |
| AI extraction | On upload, sends the file to the Anthropic API and stores the structured response in `extractedData`. |
| Approval | Creates a new `ledger[]` entry from `extractedData.lineItems.filter(li => li.include)` and marks the invoice as approved. |

### Gap Analysis

- **No functional gap.** Brief: "Invoice queue with AI-powered extraction (Anthropic Claude API) for uploaded PDF/image invoices" — exactly implemented.
- **Backend note:** the brief explicitly calls out moving the API key server-side and doing file storage in S3 or equivalent. Currently uploaded files live as base64 data URLs in localStorage, which is a hard ceiling on file size (4 MB enforced in the docs page; should be enforced here too).

---

## 20. Sales Dashboard <a name="20-sales"></a>

**JSX:** `view === "sales"` (lines 8522–8079)

### Business Purpose

The aggregate sales view across all platforms. Shows revenue, units, wishlists, refunds, ROI vs greenlit, regional breakdown, monthly trends, and platform pies. Empty-state when no sales data exists.

### Field Inventory

- **4 KPI cards:** Revenue, Units Sold, Wishlists, Avg Rev/Unit.
- **Per-platform summary cards** (one card per platform with sales): revenue + share %, units + share %.
- **Pie charts:** Revenue by Platform, Units by Platform.
- **Cumulative Revenue** line chart.
- **Budget vs Revenue** chart.
- **Regional breakdown** table (top 12 regions by revenue).
- **Wishlist time series:** daily and monthly add/delete charts.

### Data Logic & Lineage

| Field | Source |
|---|---|
| `salesData[]` | Imported from per-platform CSVs — each entry has `{date, platform, region, units, revenue, wishlists, refunds, ...}`. |
| `wishlistData[]` | Wishlist add/delete/purchase records from Steam (and any other platform that exposes them). |
| All KPIs | Aggregations over `salesData` and `wishlistData`. |
| `byPlatform` / `byMonth` / `byRegion` | Group-by reductions over `salesData`. |
| `budgetROI` | `(totalRevenue − greenlitAll) / greenlitAll × 100`. |

### Gap Analysis

- No gap on dashboard semantics. Brief: "Platform-level revenue tracking with monthly charts, wishlist data" — implemented.

---

## 21. Per-Platform Sales Pages <a name="21-sales_platform"></a>

**JSX:** `view ∈ {sales_steam, sales_xbox, sales_ps, sales_switch, sales_epic}` (lines 8038–8062)

### Business Purpose

Five near-identical pages — one per supported store. Each accepts platform-specific CSV uploads (Steam packages, Xbox sales reports, etc.), parses them with PapaParse, and displays the platform's slice of `salesData[]`.

### Field Inventory

- Page header with platform-colored badge.
- "Upload Sales CSV" file input (perm-gated by `editSales`).
- Platform-filtered KPI strip (revenue, units, wishlists for this platform only).
- Platform-filtered monthly bar chart and breakdown table.

### Data Logic & Lineage

- Identical to `sales` view but filtered to `salesData.filter(e => e.platform === platMap[view])`.

### Gap Analysis

- **Brief lists "Sales" as one page but the navigation has six entries (Dashboard + 5 platforms + Intelligence).** This is a UI-only expansion — no business gap.

---

## 22. Sales Intelligence (Promos) <a name="22-sales_promos"></a>

**JSX:** `view === "sales_promos"` (lines 8465–8520)

### Business Purpose

Two-tab analytical layer over sales data:
1. **Pricing & Promotions:** auto-detects sale periods by finding months where `effPrice` drops 10%+ below the `medianPrice` baseline. Each detected period gets a row with revenue lift, unit lift, and a name input.
2. **Paid Media:** correlates marketing-ledger spend against same-month revenue. Computes ROAS (return on ad spend) and CPA (cost per acquisition).

### Field Inventory

#### Promos tab
- 4 KPI cards: Base Price, Detected Sales count, Avg Unit Lift, Avg Discount.
- Effective Price & Revenue chart (revenue bars + price line).
- Detected Sale Periods cards (each editable by name).
- Monthly Price Analysis table.

#### Media tab
- 4 KPI cards: Total Media Spend, ROAS, CPA, Media Entries.
- Revenue vs. Media Spend chart.
- Monthly Media Spend Detail table.

### Data Logic & Lineage

| Field | Source |
|---|---|
| `medianPrice` | Median of monthly `effPrice` (= `revenue / units`). |
| `threshold` | `medianPrice × 0.9` — 10% below baseline. |
| `detectedPeriods` | Contiguous months where `effPrice < threshold`. |
| `periodStats` | Per period: revenue, units, avg price, discount %, lift vs baseline. |
| `mktgEntries` | `ledger.filter(e => CAT_TO_GROUP[e.category] === "Marketing Costs")`. |
| `roas` | `totalSalesRev / totalMediaSpend`. |
| `cpa` | `totalMediaSpend / totalSalesUnits`. |

### Gap Analysis

- **No gap.** Brief: "Effective pricing analysis with median detection and promotional period identification, Media spend vs. revenue correlation" — implemented exactly.

---

## 23. Notes (Creative & Meeting) <a name="23-notes"></a>

**JSX:** `view === "notes_creative"` and `view === "notes_meeting"` (lines 8205–8380)

### Business Purpose

Lightweight per-project notebook split into two streams: creative (design/concept memos) and meeting (recurring sync notes). Meeting notes additionally support **transcript upload** (.txt, .vtt, .srt) — the file is parsed and the AI tries to summarize/extract action items.

### Field Inventory

Per note: id, date, title, content (multiline body), tags. UI shows a master/detail layout with the note list on the left and the editor on the right.

### Data Logic & Lineage

- All fields static. No calculations.
- Transcript upload (meeting only): file → text extraction → AI summarization → pre-fills the new note.

### Gap Analysis

- **No gap.** Brief: "Zoom transcript parsing for meeting notes" — implemented.

---

## 24. Documents <a name="24-documents"></a>

**JSX:** `view === "documents"` (lines 8381–8463)

### Business Purpose

Project document library. Files are stored as base64 in localStorage (4 MB cap) and can be referenced by the AI assistant when "Docs On" is toggled — at which point the docs become part of the AI context window.

### Field Inventory

- Upload area + file list (name, size, type icon, uploaded-by, uploaded-at).
- Per-doc actions: download, delete.

### Data Logic & Lineage

- Static. Files stored in `projectDocs[]` aux state.

### Gap Analysis

- **Brief calls for S3 or equivalent** — local base64 is prototype-only. The 4 MB ceiling is enforced client-side and will need to migrate.

---

## 25. History & Audit Log <a name="25-history"></a>

**JSX:** `view === "history"` (lines 4789–4951)

### Business Purpose

Two stacked features behind one tab:
1. **Snapshots:** automatic and manual point-in-time backups of the project state. Every destructive action (import, bulk delete, greenlight save) auto-saves a snapshot first; users can also save manual snapshots and restore.
2. **Audit Log:** chronological list of every user action — cell edits, row adds/deletes, ledger changes, amendments, etc. Filterable by action / user / detail. Each entry expandable to show old vs new values.

### Field Inventory

- Tab toggle.
- **Snapshots tab:** list of snapshot cards with action label, timestamp, user, diff vs current ("+5 rows since", "+$120K budget since"), Restore button.
- **Audit tab:** filter input + table of (Timestamp, User, Action chip, Detail). Each row expandable.

### Data Logic & Lineage

| Field | Source |
|---|---|
| Snapshots | `project.changeHistory[]` — each entry has `{id, date, user, action, snapshot: full project clone}`. |
| Snapshot diff | `compareSnap(snap)`: `rowDiff`, `liveDiff`, `ledgerDiff` between snap and current. |
| Audit log | `project.auditLog[]` — last 200 entries retained, each `{id, date, user, role, action, detail, extra}`. |
| Action color codes | Lookup map at line 4904. |

### Gap Analysis

- **Brief:** "Audit logging to database (currently in-memory project state)" — confirms this is a prototype shortcut. Backend must move both snapshots and audit to DB tables and remove the 200-entry cap.

---

## 26. Reports (Cost Reports) <a name="26-reports"></a>

**JSX:** `view === "reports"` (lines 6411–6683). **Note:** `setView("reports")` auto-translates to `setView("insights")` per line 667 — meaning the navigation never actually routes here. This view is dead code from the prior version of the app.

### Business Purpose (legacy)

Was the formal printable cost report — exec summary, budget health, phase spend, cost breakdown by section, top cost items, monthly trend.

### Gap Analysis

- **The view exists in code but is unreachable.** Backend doesn't need to support a `reports` route. The Insights page (`view === "insights"`) is the supported analog.

---

## Studio-Level Views

### Studio: Portfolio <a name="studio-portfolio"></a>

**JSX:** `studioView === "portfolio"` (lines 10535–10582)

The project list page that opens before any single project is selected. Shows each project as a card with: icon (emoji), name, studio, current phase, key KPIs (greenlit, live, variance), open / delete actions. Has a "+ New Project" wizard, an "Import" path, and an icon picker (massive emoji catalog at line 9936).

**Data:** `allProjects[]` is the in-memory list; per-project full data lives at `fcc_proj_<id>` in storage and is loaded lazily into `projectCache`.

**Gap:** None — implemented.

### Studio: Insights <a name="studio-insights"></a>

**JSX:** `studioView === "insights"` (lines 10583–10729)

Cross-project rollup: combined KPIs (total budgets across projects, total spend, average burn), portfolio-level charts, project comparison table.

**Data:** Iterates `studio.projects`, loads each via storage, runs the same per-project totals, then aggregates.

**Gap:** None.

### Studio: Sales <a name="studio-sales"></a>

**JSX:** `studioView === "sales"` (lines 10730–10983)

Cross-project sales rollup. Loads `aux.salesData` for every project, merges them into a single dataset, and applies project filters and time filters (from/to date pickers, all/last 7/30/90/1y).

**Data:** `studioSalesData = {sales[], wishlists[], projNames}`. Every sales row gets `projectId` and `projectName` annotated for filtering.

**Gap:** None.

### Studio: Social <a name="studio-social"></a>

**JSX:** `studioView === "social"` (lines 10984–11067)

Surfaces external community signals (e.g. SteamDB followers, Twitter mentions, Reddit). Calls third-party APIs (`socialCache`, `socialLoading`) for each project. Display-only.

**Gap:** Brief does not mention social signals. This is bonus functionality.

### Studio: Admin <a name="studio-admin"></a>

**JSX:** `studioView === "admin"` (lines 11068–11670+)

Three sub-tabs: Users (roles, project assignments), Pages (per-role page-visibility matrix), Platforms (sales platforms enabled for the studio). Only visible to users with `manageUsers` permission.

**Data:** `studio.users[]`, `studio.rolePages` (a map `{roleKey: [viewIds]}`), `studio.platforms[]`.

**Gap:** Brief specifies **6 permission tiers** (Admin, Producer, Support, Finance, Developer, Viewer). The code at line 9915 defines **5 only** — the **Viewer role is missing**. This is the most concrete brief-vs-code gap in the analysis.

---

## Developer Portal

The portal renders only when `?portal=<projectId>&dev=<name>` is in the URL. It is a separate React component (`DevPortal`, lines 11671–11907) that reads `fcc_proj_<id>` and `fcc_aux_<id>` directly from storage, but **only writes to `aux.changeLog` and `aux.invoiceQueue`** — never to the main project data.

### D1. Dev Portal: Budget <a name="d1-devportal-budget"></a>

Read-only budget grid using the same `BUDGET_GROUPS` structure. Cells are inputs that capture **proposed values** in `proposedChanges` state — they don't mutate the project. Submitting creates a `changeLog` batch with status `pending` that surfaces back on the producer's Amendments page (and triggers a notification).

### D2. Dev Portal: Invoice <a name="d2-devportal-invoice"></a>

File upload + optional note. File is stored as base64 in `aux.invoiceQueue` with `submittedBy`, `submittedAt`, `status: "pending"`. Producer sees it in the Invoices page.

### D3. Dev Portal: History <a name="d3-devportal-history"></a>

Filtered view of `aux.changeLog` and `aux.invoiceQueue` where `submittedBy === devName`. Read-only.

### Gap Analysis (Developer Portal)

- **Brief is explicit:** "Secure Developer Portal with proper auth (currently URL params only — no real security)". This is acknowledged as a known gap and is in the Phase 3 deliverables.
- **Brief calls for** "Invitation system: Producer generates a portal link with scoped access token". Currently the link is just `?portal=<id>&dev=<name>` with no token validation.
- **Brief calls for** "Email notifications for submissions, approvals, rejections". Not implemented (no email infrastructure exists in the prototype).

---

## Consolidated Gap Analysis

The brief and the prototype are tightly aligned. Most of the "gaps" listed below are explicitly acknowledged in Section 4 of the brief as Phase 1–4 deliverables. Items marked **NEW** are gaps not explicitly listed in the brief that this analysis surfaced.

| # | Area | Brief states | Code reality | Severity |
|---|---|---|---|---|
| G1 | Permission tiers | 6 tiers (Admin, Producer, Support, Finance, Developer, **Viewer**) | 5 tiers — Viewer is missing from `DEFAULT_ROLES` (line 9915) | **NEW — High** |
| G2 | Authentication | Microsoft Entra ID SSO | None — local user records only | Acknowledged (Phase 1) |
| G3 | Storage | PostgreSQL with relational schema | localStorage via `window.storage` adapter | Acknowledged (Phase 1) |
| G4 | API key for AI | Server-side | Client-side, embedded in JSX | Acknowledged (architecture notes) |
| G5 | File storage | S3 or equivalent | Base64 in localStorage, 4 MB hard cap | Acknowledged (Phase 2) |
| G6 | Developer Portal auth | Scoped invitation tokens | URL params only (`?portal=...&dev=...`) | Acknowledged (Phase 3) |
| G7 | Email notifications | For submissions/approvals/rejections | Not implemented | Acknowledged (Phase 3) |
| G8 | Snapshot/backup | Automated, server-managed | Manual + opportunistic in-memory snapshots | Acknowledged (Phase 4) |
| G9 | Audit log | DB-backed, unbounded | In-memory, 200-entry cap | Acknowledged (Phase 4) |
| G10 | QuickBooks integration | API-based | CSV import only | Acknowledged (Phase 4) |
| G11 | PDF export | Server-side rendering | Browser print + clipboard workarounds | Acknowledged (Phase 4) |
| G12 | Multi-tenant | Studio can manage many projects | Single studio in `fcc_studio` key only | Acknowledged (Phase 1) |
| G13 | Auto-match in Reconciliation | "Auto-match by amount" | Implemented for QB import duplicate detection only — no one-click match-all in the recon UI | **NEW — Low** |
| G14 | Search/filter on Ledger | "Search/filter" listed as a feature | Sub-tabs and column sort exist; no free-text search box | **NEW — Low** |
| G15 | "Reports" page | Listed in features ("Cost Reports with phase spend...") | Code exists at `view === "reports"` but auto-redirects to `insights` — dead route | **NEW — Cosmetic** |
| G16 | One Sheet calculations duplication | Should be derived from P&L | Math is reimplemented inline twice — should consolidate at the API layer | **NEW — Architecture** |
| G17 | Reconciliation grid changes bypass review | Implicit — all cell changes need review | Reconcile commit can directly mutate `row.live[mKey]` (creates a `changeLog` batch but cell is already changed) | **NEW — Medium** |
| G18 | Marketing Budget category seeding | Should be controlled | Client-side migration auto-rewrites old categories on view mount (line 3812) | **NEW — Architecture** |

---

## Notes for the backend developer

1. **The FX cascade is the single most important business rule.** Reproduce `cellRate → cellUSD → effGrandTotal` exactly. Add a `GET /projects/:id/totals?mode=live&month=2025-08&currency=usd` style endpoint that runs the cascade server-side; do not let the frontend re-implement the math after migration.
2. **Greenlit version is the unit of change control.** When a new amendment is approved, snapshot the complete `greenlit{}` map across every row, the `glPhases`, `glPhaseAssignment`, `glPnl`, and `glOneSheet` together — they form one logical version.
3. **The three-layer change control workflow needs to be preserved:**
   - Live cell edits by editors → applied immediately, audit-logged.
   - Live cell edits via Reconciliation auto-adjust → applied immediately AND queued in `changeLog` for retroactive review.
   - Greenlit cell edits → only allowed during a draft amendment, only finalized through the 3-checkbox-gated approval flow.
   - Developer-portal proposed edits → never applied, always wait in `changeLog` for producer approval.
4. **"Calendar month" and "milestone" are interchangeable in this system.** Don't introduce a separate milestone concept; the brief is consistent that they're 1:1.
5. **Currency rules:** `multiCurrency` is a project-level boolean; `altCurrency` is a single string code (one alt currency only). Per-row currency is set via `rowCurrencyOverrides[rowId]`. The default-alt convention for `devfees` rows (alt unless explicitly USD-overridden) is encoded in `rowIsAlt` (line 1746) — preserve that asymmetry.
6. **The brief says "single-file React frontend prototype serves as the production specification."** When in doubt, the JSX wins.
