# Fictions Folio — BA Page-by-Page Specification

**Source used:** Uploaded Claude conversation only (`conversations.json`).  
**Scope:** Studio views, project views, developer portal, editable fields, calculated fields, and recommended backend API surfaces based on the conversation.

---

## 1. Summary Table

| # | Page / Section | Area | Purpose | Complexity | Functionality |
|---:|---|---|---|---|---|
| 1 | Studio Portfolio | Studio | Manage all projects in the studio and open/create projects. | Medium | Project cards, project creation, project metadata, navigation into project shell. |
| 2 | Studio Insights | Studio | Portfolio-level budget and project analytics. | Medium | Aggregate project KPIs, project health, cross-project rollups. |
| 3 | Studio Sales | Studio | Portfolio-level sales intelligence. | Medium | Revenue and units across games/platforms. |
| 4 | Studio Social | Studio | Social/community reporting surface. | Low | Content/social metrics placeholder-style view. |
| 5 | Studio Admin | Studio | Manage users, roles, platforms, page visibility. | High | RBAC, page permissions, studio settings, user/platform configuration. |
| 6 | Project Home | Project | Project landing page and high-level status snapshot. | Medium | Project KPI cards, alerts, recent activity, quick links. |
| 7 | Project Setup Wizard | Project Creation | Create a new project and seed default structures. | High | Project metadata, optional contracted budget, phases, import/data setup. |
| 8 | Contracted Budget / Master Budget | Project Finance | Capture top-level approved/contracted cost categories. | Medium | Development, porting, external, reserve, contingency, marketing budgets. |
| 9 | Game Budget Grid | Project Finance | Main bottom-up monthly cost planning grid. | Very High | Rows, months, live/greenlit modes, FX, multi-cell paste, CSV import/export, closed months. |
| 10 | Marketing Budget | Project Finance | Separate marketing planning grid. | High | Marketing categories, category budgets, line items, monthly planned spend. |
| 11 | Phases | Project Planning | Configure development phases and month-to-phase mapping. | Medium | Phase names, colors, durations, descriptions, assignments. |
| 12 | Dashboard | Reporting | Executive KPI dashboard. | Medium | KPI cards, monthly burn, cumulative spend, spend by section/phase. |
| 13 | Insights / Cost Reports | Reporting | Deep budget analytics and cost reporting. | High | Budget health, cumulative charts, burn charts, top items, variance drivers. |
| 14 | Variance | Reporting | Compare actual-to-date/live forecast vs greenlit budget. | Medium | 5-column variance report: actuals, remaining, projected, greenlit, variance. |
| 15 | Rollups | Reporting | Annual and phase-based spend summaries. | Medium | Annual rollup and phase rollup by group/section/subsection. |
| 16 | Payment Schedule | Reporting / Finance | Milestone-by-milestone greenlit vs live cash requirement. | Medium | Milestone totals, phase tags, difference and cumulative difference. |
| 17 | Ledger Top Sheet | Finance | Summary of ledger/invoice position. | Medium | Ledger totals, billed/paid/open amounts, category summary. |
| 18 | Ledger | Finance | Operational transaction ledger. | High | Transactions, inline editing, category tabs, CSV/QB import, allocations. |
| 19 | Marketing Ledger | Finance | Marketing-specific ledger view. | Medium | Marketing entries grouped by marketing categories. |
| 20 | Reconciliation | Finance | Allocate ledger entries to budget rows and close months. | Very High | Allocate invoice amounts, adjust grid, pending review, close month, conflict-sensitive workflow. |
| 21 | Invoice Queue | Finance / Workflow | Review uploaded invoices before ledger creation. | High | Upload, AI extraction, review modal, approve/reject, ledger entry creation. |
| 22 | P&L Projection | Greenlight / Finance | Model recoupment, royalties, ROI, and unit scenarios. | Very High | Retail inputs, fee inputs, tiered royalty engine, partner contributions, scenarios. |
| 23 | One Sheet | Greenlight / Reporting | Executive export summary of deal economics. | High | Read-only summary of P&L, budget, break-even, ROI, greenlight assumptions. |
| 24 | Amendments | Greenlight / Workflow | Draft and approve greenlight budget amendments. | Very High | 3-checkbox gate, amendment history, update contracted budget, export P&L/One Sheet. |
| 25 | Milestones | Project Tracking | Per-month milestone notes plus budget progress. | Medium | Build info, notes, Dropbox link, spend-to-date metrics. |
| 26 | Notes — Creative | Content | Maintain creative/design project notes. | Low | CRUD notes with title, date, content, tags. |
| 27 | Notes — Meeting | Content | Maintain meeting notes and parse transcripts. | Medium | CRUD notes plus transcript upload and AI summarization. |
| 28 | Documents | Content | Project document library for AI/document context. | Medium | Upload, list, download, delete; prototype stores base64 locally. |
| 29 | History & Audit | Governance | Restore snapshots and review audit trail. | High | Manual/automatic snapshots, restore, audit log filters. |
| 30 | Sales Dashboard | Sales Intelligence | Aggregate sales performance across platforms. | High | Revenue, units, wishlists, regional breakdown, platform split, trends. |
| 31 | Per-Platform Sales Pages | Sales Intelligence | Import and analyze platform-specific sales data. | High | CSV import, mapping, platform KPIs, monthly revenue/units, regional performance. |
| 32 | Sales Promotions | Sales Intelligence | Analyze promotions, effective pricing, and media spend correlation. | High | Promotion detection, median pricing, discount windows, marketing spend vs revenue. |
| 33 | Developer Portal — Budget | External Portal | Developer proposes budget changes without direct write access. | High | Scoped budget view, editable proposal cells, submit change request. |
| 34 | Developer Portal — Invoice | External Portal | Developer submits invoices for review. | Medium | File upload, invoice metadata, submission status. |
| 35 | Developer Portal — History | External Portal | Developer views own submitted budget/invoice history. | Low | Submission list and status tracking. |
| 36 | Reports alias | Routing | Dead route alias that redirects to Insights. | Low | `reports` auto-redirects to `insights`; no separate functional page. |

---

## 2. Cross-Cutting Business Rules

### 2.1 Core Data Objects

| Object | Key Fields |
|---|---|
| Project | `id`, `name`, `studio`, `startDate`, `monthCount`, `rows[]`, `phases[]`, `phaseAssignment`, `ledger[]`, `masterBudget`, `marketingBudget`, `multiCurrency`, `altCurrency`, `spotRates`, `globalSpotRate`, `closedMonths` |
| Budget Row | `id`, `sectionId`, `subsectionId`, `meta`, `live{monthKey}`, `greenlit{monthKey}`, `rates{monthKey}` |
| Ledger Entry | `id`, `date`, `payee`, `description`, `category`, `amount`, `currency`, `allocations[]`, `status`, `source`, `invoiceId` |
| Invoice Queue Item | `id`, `file`, `submittedBy`, `submittedAt`, `status`, `extractedData`, `reviewedBy`, `reviewedAt` |
| Sales Data | `date`, `platform`, `region`, `units`, `revenue`, `wishlists`, `refunds`, promotion/price attributes |
| Notes | `id`, `date`, `title`, `content`, `tags`, `type` |
| Documents | `id`, `name`, `size`, `type`, `uploadedBy`, `uploadedAt`, `dataUrl/storageKey` |
| Milestone Data | `monthKey`, `buildInfo`, `notes`, `dropboxFolder` |

### 2.2 FX Cascade

All USD-aware totals must resolve currency conversion using this priority:

1. Per-cell rate: `row.rates[monthKey]`
2. Column spot rate: `project.spotRates[monthKey]`
3. Global rate: `project.globalSpotRate`

The conversation identifies `effSectionTotal`, `effGroupTotal`, and `effGrandTotal` as key calculation primitives that should be centralized in the backend.

### 2.3 Greenlit vs Live Modes

| Mode | Meaning |
|---|---|
| Greenlit | Locked approved baseline; updated through greenlight/amendment workflow. |
| Live | Current working forecast; editable by authorized users. |
| Variance | Usually `Live - Greenlit`, with red/green highlighting. |

### 2.4 API Design Principle

The frontend prototype stores data in localStorage. Production API should expose REST or GraphQL endpoints that return render-ready calculated values, so the frontend does not duplicate core financial formulas.

---

# 3. Detailed Page Specifications

## 3.1 Studio Portfolio

| Field | Details |
|---|---|
| Purpose | Studio-level landing page where users select, create, and manage projects. |
| Functionality | Shows project cards, status information, project setup entry point, and navigation into a project shell. |
| Editable Fields | Project name, studio/name metadata, icon, point person, target release date, platforms, optional budget setup during creation. |
| Calculated Fields | Project count, total budget across projects, portfolio totals, basic project health indicators. |
| API Endpoints | `GET /studios/:studioId/projects`, `POST /projects`, `PATCH /projects/:id`, `DELETE /projects/:id`. |
| Request Example | `POST /projects` with `{ "name": "Covenant", "studio": "Fictions", "startDate": "2025-08", "monthCount": 36, "masterBudget": {...} }`. |
| Response Example | Returns a mountable project record with `rows`, `phases`, `phaseAssignment`, `masterBudget`, `marketingBudget`, and empty aux arrays. |
| Complexity | Medium. |

## 3.2 Studio Insights

| Field | Details |
|---|---|
| Purpose | Portfolio-level analytics across projects. |
| Functionality | Aggregates budgets, spend, project health, and high-level performance. |
| Editable Fields | None directly; reads project data. |
| Calculated Fields | Portfolio budget, spend to date, project count, by-project totals, risk/health rollups. |
| API Endpoints | `GET /studios/:studioId/insights`. |
| Request Example | `GET /studios/stu_001/insights?asOf=2026-05-01`. |
| Response Example | `{ "studioId": "stu_001", "summary": {...}, "projects": [...] }`. |
| Complexity | Medium. |

## 3.3 Studio Sales

| Field | Details |
|---|---|
| Purpose | Studio-wide view of sales performance. |
| Functionality | Aggregates revenue, units, wishlists, refunds, and platform performance across projects. |
| Editable Fields | None directly. |
| Calculated Fields | Total revenue, total units, revenue by platform/project, average revenue per unit. |
| API Endpoints | `GET /studios/:studioId/sales`. |
| Request Example | `GET /studios/stu_001/sales?from=2026-01-01&to=2026-05-01`. |
| Response Example | `{ "summary": {"revenue": 1250000, "units": 85000}, "byProject": [], "byPlatform": [] }`. |
| Complexity | Medium. |

## 3.4 Studio Social

| Field | Details |
|---|---|
| Purpose | Studio-level social/community reporting surface. |
| Functionality | Tracks social/community metrics where available. |
| Editable Fields | Not clearly detailed in conversation; likely read-only/placeholder. |
| Calculated Fields | Aggregate social metrics if data exists. |
| API Endpoints | `GET /studios/:studioId/social`. |
| Request Example | `GET /studios/stu_001/social`. |
| Response Example | `{ "channels": [], "summary": {} }`. |
| Complexity | Low. |

## 3.5 Studio Admin

| Field | Details |
|---|---|
| Purpose | Manage studio configuration, users, roles, platforms, and visibility permissions. |
| Functionality | RBAC setup, page visibility per role, user management, platform list. |
| Editable Fields | User name/email/role, role page permissions, studio platforms, UI scale/settings. |
| Calculated Fields | Effective permissions per user and role; access matrix. |
| API Endpoints | `GET /studios/:id/admin`, `PATCH /studios/:id/users`, `PATCH /studios/:id/roles`, `PATCH /studios/:id/platforms`. |
| Request Example | `PATCH /studios/stu_001/roles/producer` with `{ "pages": ["home","budget","ledger"] }`. |
| Response Example | `{ "role": "producer", "pages": [...], "updatedAt": "2026-05-01T00:00:00Z" }`. |
| Complexity | High. |

---

## 3.6 Project Home

| Field | Details |
|---|---|
| Purpose | Project landing dashboard and navigation hub. |
| Functionality | Shows key project status, current budget situation, alerts, recent notes/activity, and shortcut cards. |
| Editable Fields | None directly, except links/actions to other pages. |
| Calculated Fields | Total live budget, greenlit budget, variance, recent notes, recent activity, alert conditions. |
| API Endpoints | `GET /projects/:id/home`. |
| Request Example | `GET /projects/prj_2k9l/home`. |
| Response Example | `{ "project": {...}, "summary": {"liveTotal": 10215000, "greenlitTotal": 9750000, "variance": 465000}, "recentNotes": [], "alerts": [] }`. |
| Complexity | Medium. |

## 3.7 Project Setup Wizard

| Field | Details |
|---|---|
| Purpose | Create and initialize a new game project. |
| Functionality | Captures metadata, optional budget figures, phase template, and may import/seed initial rows. |
| Editable Fields | Project name, studio, start date, month count, platforms, point person, release date, icon, contracted budget values. |
| Calculated Fields | Month keys from start date/month count, default phase assignment, seeded project skeleton. |
| API Endpoints | `POST /projects`. |
| Request Example | `{ "name": "Covenant", "startDate": "2025-08", "monthCount": 36, "masterBudget": {"developmentCosts":6000000,"portingCosts":600000,"externalCosts":1400000,"reserve":250000,"contingency":1000000,"marketing":500000} }`. |
| Response Example | Full project object, including rows, phases, phase assignments, master budget, empty ledger/notes/sales/invoice structures. |
| Complexity | High. |

## 3.8 Contracted Budget / Master Budget

| Field | Details |
|---|---|
| Purpose | Store top-level approved cost envelope. |
| Functionality | Captures the high-level budget baseline used by P&L, One Sheet, and comparison reporting. |
| Editable Fields | `developmentCosts`, `portingCosts`, `externalCosts`, `reserve`, `contingency`, `marketing`. |
| Calculated Fields | `gameBudget = developmentCosts + portingCosts + externalCosts`; `totalProjectCosts = gameBudget + reserve + contingency + marketing`; grid variance against live grouped totals. |
| API Endpoints | `GET /projects/:id/master-budget`, `PATCH /projects/:id/master-budget`. |
| Request Example | `PATCH /projects/prj_2k9l/master-budget` with `{ "developmentCosts": 6000000, "portingCosts": 600000, "externalCosts": 1400000, "reserve": 250000, "contingency": 1000000, "marketing": 500000 }`. |
| Response Example | `{ "masterBudget": {...}, "derived": {"gameBudget":8000000,"totalProjectCosts":9750000,"liveByGroup":{...}} }`. |
| Complexity | Medium. |

## 3.9 Game Budget Grid

| Field | Details |
|---|---|
| Purpose | Main monthly project cost planning grid. |
| Functionality | Live/greenlit budget modes, editable rows, monthly cells, FX handling, row CRUD, CSV import/export, paste from Excel, phase bars, closed month warnings. |
| Editable Fields | Row metadata: `role/item`, `department/category`, `name/detail`; monthly values in `live` or `greenlit`; row section/subsection; FX rates; spot rates; row currency overrides. |
| Calculated Fields | Row total, subsection total, section total, group total, grand total, year totals, greenlit/live variance, FX-converted USD values. |
| API Endpoints | `GET /projects/:id/budget`, `POST /projects/:id/budget/rows`, `PATCH /projects/:id/budget/rows/:rowId`, `PATCH /projects/:id/budget/cells`, `POST /projects/:id/budget/import`, `GET /projects/:id/budget/export`, `PATCH /projects/:id/fx-rates`, `POST /projects/:id/greenlight`. |
| Request Example | `PATCH /projects/prj_2k9l/budget/cells` with `{ "mode": "live", "updates": [{"rowId":"row_1","month":"2025-08","value":45000,"rate":0.735}] }`. |
| Response Example | `{ "updatedRows": [...], "totals": {"grandTotal":10215000}, "auditId":"aud_001" }`. |
| Complexity | Very High. |

## 3.10 Marketing Budget

| Field | Details |
|---|---|
| Purpose | Plan marketing spend separately from the main game budget. |
| Functionality | Marketing categories with budget allocation and monthly line-item planning. |
| Editable Fields | Category name, category budget, line item name, line item category, monthly planned spend. |
| Calculated Fields | Contracted marketing budget, allocated budget, planned spend, remaining budget, per-category planned spend, category variance. |
| API Endpoints | `GET /projects/:id/marketing-budget`, `PATCH /projects/:id/marketing-budget/categories`, `POST /projects/:id/marketing-budget/rows`, `PATCH /projects/:id/marketing-budget/rows/:rowId`, `DELETE /projects/:id/marketing-budget/rows/:rowId`. |
| Request Example | `{ "categories": [{"id":"paid_media","name":"Paid Media","budget":200000}] }`. |
| Response Example | `{ "marketingBudget": {...}, "summary": {"contracted":500000,"allocated":500000,"planned":420000,"remaining":80000} }`. |
| Complexity | High. |

## 3.11 Phases

| Field | Details |
|---|---|
| Purpose | Define project timeline phases and map months to phases. |
| Functionality | Phase template editing, phase color/duration/description, month assignment. |
| Editable Fields | Phase name, color, duration/months, description, month-to-phase assignment. |
| Calculated Fields | Month sequence, phase timeline bands, phase totals used by rollups and reports. |
| API Endpoints | `GET /projects/:id/phases`, `PATCH /projects/:id/phases`. |
| Request Example | `{ "phases": [{"id":"alpha","name":"Alpha","color":"#3730a3","months":8}], "phaseAssignment": {"2025-08":"concept"} }`. |
| Response Example | `{ "phases": [...], "phaseAssignment": {...}, "months": [...] }`. |
| Complexity | Medium. |

## 3.12 Dashboard

| Field | Details |
|---|---|
| Purpose | Quick executive dashboard for budget health. |
| Functionality | KPI cards and charts for burn, cumulative spend, section and phase spend. |
| Editable Fields | None. |
| Calculated Fields | Total budget, ledger spent, remaining, average burn, monthly burn, cumulative spend, spend by section, spend by phase. |
| API Endpoints | `GET /projects/:id/dashboard`. |
| Request Example | `GET /projects/prj_2k9l/dashboard?asOf=2026-05-01`. |
| Response Example | `{ "kpis": {...}, "monthlyBurn": [], "cumulative": [], "bySection": [], "byPhase": [] }`. |
| Complexity | Medium. |

## 3.13 Insights / Cost Reports

| Field | Details |
|---|---|
| Purpose | Deep analytics view for diagnosing budget problems. |
| Functionality | Budget health, cumulative and monthly charts, spend by section/phase/department, top line items, over-budget rows. |
| Editable Fields | None. |
| Calculated Fields | Timeline progress, burn rate, projected total, remaining months, cumulative live/greenlit, section/phase/department totals, variance by line item. |
| API Endpoints | `GET /projects/:id/insights`, `GET /projects/:id/cost-reports`. |
| Request Example | `GET /projects/prj_2k9l/insights?asOf=2026-05-01`. |
| Response Example | `{ "summary": {...}, "charts": {"cumulative": [], "monthlyBurn": [], "bySection": [], "byPhase": []}, "topItems": [] }`. |
| Complexity | High. |

## 3.14 Variance

| Field | Details |
|---|---|
| Purpose | Five-column report comparing actuals-to-date/remaining/live projection vs greenlit. |
| Functionality | Shows actuals to date, remaining projected, projected total, greenlit total, and variance by cost group/section. |
| Editable Fields | None. |
| Calculated Fields | `elapsedMonths`, actuals-to-date from live grid for past months, remaining projected from live grid future months, projected total, greenlit total, variance, variance %. Ledger actuals are shown as a reference but not used in variance math. |
| API Endpoints | `GET /projects/:id/variance`. |
| Request Example | `GET /projects/prj_2k9l/variance?asOf=2026-05-01`. |
| Response Example | `{ "summary": {"actualsToDate":1842500,"remainingProjected":8372500,"projectedTotal":10215000,"greenlitTotal":9750000,"variance":465000}, "byGroup": [] }`. |
| Complexity | Medium. |

## 3.15 Rollups

| Field | Details |
|---|---|
| Purpose | Summarize spend by calendar year and development phase. |
| Functionality | Annual rollup and phase rollup across groups, sections, and subsections. |
| Editable Fields | None. |
| Calculated Fields | Year totals by subsection/section/group, phase totals by subsection/section/group, grand totals. |
| API Endpoints | `GET /projects/:id/rollups`. |
| Request Example | `GET /projects/prj_2k9l/rollups`. |
| Response Example | `{ "annual": {"years":[2025,2026,2027], "rows":[]}, "phase": {"phases":[], "rows":[]} }`. |
| Complexity | Medium. |

## 3.16 Payment Schedule

| Field | Details |
|---|---|
| Purpose | Milestone-by-milestone view of greenlit vs live payment requirements. |
| Functionality | Displays each month as a milestone with phase, greenlit amount, live amount, difference, cumulative difference. |
| Editable Fields | None. |
| Calculated Fields | Milestone label, phase, greenlit total, live total, difference, cumulative difference. |
| API Endpoints | `GET /projects/:id/payment-schedule`. |
| Request Example | `GET /projects/prj_2k9l/payment-schedule`. |
| Response Example | `{ "milestones": [{"month":"2025-08","label":"MS 1 — Aug 25","phase":"Concept","greenlit":100000,"live":120000,"difference":20000,"cumulativeDifference":20000}] }`. |
| Complexity | Medium. |

## 3.17 Ledger Top Sheet

| Field | Details |
|---|---|
| Purpose | Executive finance summary of ledger position. |
| Functionality | Shows billed, paid, open, and category-level ledger summary. |
| Editable Fields | None. |
| Calculated Fields | Total billed, paid, unpaid/open, by-category totals, by-status totals. |
| API Endpoints | `GET /projects/:id/ledger/top-sheet`. |
| Request Example | `GET /projects/prj_2k9l/ledger/top-sheet`. |
| Response Example | `{ "summary": {"totalBilled":900000,"paid":700000,"open":200000}, "byCategory": [] }`. |
| Complexity | Medium. |

## 3.18 Ledger

| Field | Details |
|---|---|
| Purpose | Transaction-level finance ledger. |
| Functionality | Add/edit/delete ledger entries, category tabs, sorting, CSV/QuickBooks import, duplicate detection, allocation status. |
| Editable Fields | Date, payee/vendor, invoice number, category, subcategory, description, amount, currency, status, payment date, allocations. |
| Calculated Fields | Total billed, total paid, allocation status, category totals, duplicate fingerprint. |
| API Endpoints | `GET /projects/:id/ledger`, `POST /projects/:id/ledger`, `PATCH /projects/:id/ledger/:entryId`, `DELETE /projects/:id/ledger/:entryId`, `POST /projects/:id/ledger/import`. |
| Request Example | `{ "date":"2026-04-12", "payee":"Vendor", "amount":12500, "category":"QA", "currency":"USD" }`. |
| Response Example | `{ "entry": {...}, "summary": {"totalBilled":912500}, "etag":"v3" }`. |
| Complexity | High. |

## 3.19 Marketing Ledger

| Field | Details |
|---|---|
| Purpose | Marketing-specific filtered ledger. |
| Functionality | Same ledger mechanics but grouped around marketing categories. |
| Editable Fields | Same as Ledger, with marketing category mapping. |
| Calculated Fields | Marketing spend by category, paid/open marketing totals, comparison to marketing budget. |
| API Endpoints | `GET /projects/:id/marketing-ledger`, `POST /projects/:id/marketing-ledger`, `PATCH /projects/:id/ledger/:entryId`. |
| Request Example | `{ "payee":"Ad Network", "amount":30000, "marketingCategoryId":"paid_media" }`. |
| Response Example | `{ "entry": {...}, "marketingSummary": {"planned":420000,"actual":30000} }`. |
| Complexity | Medium. |

## 3.20 Reconciliation

| Field | Details |
|---|---|
| Purpose | Allocate ledger/invoice amounts to budget rows and optionally adjust the live grid through formal review. |
| Functionality | Select unreconciled ledger entries, allocate to rows/months, create review batch, close months after reconciliation. |
| Editable Fields | Selected ledger entry, target budget row, month key, allocation amount, adjustment option, close-month decision. |
| Calculated Fields | Remaining unallocated amount, allocation totals, ledger match suggestions, month reconciliation status. |
| API Endpoints | `GET /projects/:id/reconciliation`, `POST /projects/:id/reconciliation/allocate`, `POST /projects/:id/reconciliation/auto-match`, `POST /projects/:id/months/:monthKey/close`. |
| Request Example | `POST /projects/prj_2k9l/reconciliation/allocate` with `{ "ledgerEntryId":"led_1", "allocations":[{"rowId":"row_1","month":"2026-04","amount":12500}], "adjustGrid": true }` plus `If-Match` header. |
| Response Example | `{ "ledgerEntry": {...}, "changeBatch": {...}, "remainingUnallocated":0 }`; on conflict, return `409` with current entry. |
| Complexity | Very High. |

## 3.21 Invoice Queue

| Field | Details |
|---|---|
| Purpose | Holding pen for invoices awaiting review. |
| Functionality | Upload invoices, AI extraction, review/edit extracted data, approve to ledger or reject. |
| Editable Fields | Uploaded file, vendor, invoice number, invoice date, currency, line item description, line amount, include/exclude checkbox, review decision. |
| Calculated Fields | Pending/processed status, extracted totals, approved line-item total, ledger entry generated from included lines. |
| API Endpoints | `GET /projects/:id/invoices`, `POST /projects/:id/invoices`, `POST /projects/:id/invoices/:invoiceId/extract`, `PATCH /projects/:id/invoices/:invoiceId/review`, `POST /projects/:id/invoices/:invoiceId/approve`, `POST /projects/:id/invoices/:invoiceId/reject`. |
| Request Example | Multipart upload with file; review payload `{ "extractedData": {"vendor":"Vendor","invoiceNumber":"INV-1","lineItems":[{"description":"QA","amount":12500,"include":true}] } }`. |
| Response Example | `{ "invoice": {"id":"inv_1","status":"approved"}, "createdLedgerEntryId":"led_1" }`. |
| Complexity | High. |

## 3.22 P&L Projection

| Field | Details |
|---|---|
| Purpose | Model deal economics, recoupment, royalties, and ROI scenarios. |
| Functionality | Inputs for costs, retail price, discounts, platform/engine fees, developer/licensor royalty tiers, publishing partner contributions, scenario units, ROI targets. |
| Editable Fields | Retail price, discount rate, platform fee, engine fee, upfront engine fee, publishing fee rate, cost inputs, marketing budget, partner enablement and contributions, royalty tiers, scenario units, ROI targets. |
| Calculated Fields | Publishing fee, total costs/recoupment, partner contribution, Fictions recoupment, average price, after-platform revenue/unit, licensor royalty, net per unit, tiered royalty waterfalls, profit, break-even units, ROI targets. |
| API Endpoints | `GET /projects/:id/pnl`, `PATCH /projects/:id/pnl`, `POST /projects/:id/pnl/calculate`, `POST /projects/:id/pnl/sync-from-master-budget`. |
| Request Example | `{ "retailPrice": 29.99, "discountRate": 0.15, "platformFee": 0.30, "engineFee": 0.05, "scenarioUnits": [50000,100000,250000] }`. |
| Response Example | `{ "inputs": {...}, "summary": {"recoupment":9750000,"netPerUnit":16.14,"breakEvenUnits":604089}, "scenarios": [] }`. |
| Complexity | Very High. |

## 3.23 One Sheet

| Field | Details |
|---|---|
| Purpose | Executive summary for greenlight and stakeholder review. |
| Functionality | Presents P&L assumptions, budget totals, break-even, ROI, deal structure, and export-ready summary. |
| Editable Fields | Mostly none; inherits P&L and project data. |
| Calculated Fields | All major P&L summary outputs, total costs, recoupment, ROI, break-even, partner split. |
| API Endpoints | `GET /projects/:id/one-sheet`, `POST /projects/:id/one-sheet/export`. |
| Request Example | `GET /projects/prj_2k9l/one-sheet`. |
| Response Example | `{ "project": {...}, "budget": {...}, "pnlSummary": {...}, "export": {"html":"..."} }`. |
| Complexity | High. |

## 3.24 Amendments

| Field | Details |
|---|---|
| Purpose | Manage greenlight amendments after baseline changes. |
| Functionality | Draft amendment, compare live vs greenlit/contracted, complete 3-checkbox gate, save new greenlit version, update amendment history. |
| Editable Fields | Amendment title/notes, checkbox flags: update contracted budget, export P&L, export One Sheet; selected changes. |
| Calculated Fields | Amendment number, category deltas, changed rows/cells, revised totals, variance vs prior greenlight, history snapshots. |
| API Endpoints | `GET /projects/:id/amendments`, `POST /projects/:id/amendments/draft`, `POST /projects/:id/amendments/:id/approve`, `POST /projects/:id/amendments/:id/reject`, `GET /projects/:id/amendments/history`. |
| Request Example | `{ "notes":"Increase QA and porting budget", "updateContractedBudget":true, "exportPnl":true, "exportOneSheet":true }`. |
| Response Example | `{ "amendment": {"id":"amd_2","version":2,"status":"approved"}, "greenlitVersion":2, "snapshotId":"snap_10" }`. |
| Complexity | Very High. |

## 3.25 Milestones

| Field | Details |
|---|---|
| Purpose | Track build/milestone notes with budget progress by month. |
| Functionality | Per-milestone build information, milestone notes, Dropbox folder link, expanded row UI. |
| Editable Fields | Build Information, Milestone Notes, Dropbox Folder URL. |
| Calculated Fields | Milestone number, phase, month burn, life-to-date spend, total budget, remaining budget, progress %, pct of total. |
| API Endpoints | `GET /projects/:id/milestones`, `PATCH /projects/:id/milestones/:monthKey`. |
| Request Example | `{ "buildInfo":"Alpha build delivered", "notes":"Known issues list attached", "dropboxFolder":"https://..." }`. |
| Response Example | `{ "monthKey":"2026-04", "metadata": {...}, "metrics": {"monthBurn":300000,"ltdSpend":2500000,"remainingBudget":7715000} }`. |
| Complexity | Medium. |

## 3.26 Notes — Creative & Meeting

| Field | Details |
|---|---|
| Purpose | Maintain project notes, split into creative and meeting streams. |
| Functionality | New/edit/delete notes; meeting notes support transcript upload and AI summary/action extraction. |
| Editable Fields | Title, date, content, tags; uploaded transcript for meeting notes. |
| Calculated Fields | Newest-first sorting, note count, recent notes, content preview. |
| API Endpoints | `GET /projects/:id/notes?type=creative|meeting`, `POST /projects/:id/notes`, `PATCH /projects/:id/notes/:noteId`, `DELETE /projects/:id/notes/:noteId`, `POST /projects/:id/notes/transcript`. |
| Request Example | `{ "type":"meeting", "title":"Weekly Sync", "date":"2026-05-01", "content":"...", "tags":["production"] }`. |
| Response Example | `{ "note": {...}, "recentNotes": [] }`. |
| Complexity | Creative: Low; Meeting: Medium. |

## 3.27 Documents

| Field | Details |
|---|---|
| Purpose | Project document library and optional AI context source. |
| Functionality | Upload/list/download/delete documents; documents can be used by AI context when toggled. |
| Editable Fields | Uploaded files; delete action. |
| Calculated Fields | File size display, type icon, uploaded metadata, AI context inclusion state. |
| API Endpoints | `GET /projects/:id/documents`, `POST /projects/:id/documents`, `GET /projects/:id/documents/:docId/download`, `DELETE /projects/:id/documents/:docId`. |
| Request Example | Multipart upload with file metadata. |
| Response Example | `{ "document": {"id":"doc_1","name":"brief.pdf","size":1200000,"downloadUrl":"signed-url"} }`. |
| Complexity | Medium. |

## 3.28 History & Audit

| Field | Details |
|---|---|
| Purpose | Preserve and inspect project history. |
| Functionality | Manual/automatic snapshots, restore snapshot, audit log filtering. |
| Editable Fields | Snapshot label/notes; restore action. |
| Calculated Fields | Snapshot count, latest snapshot, filtered audit rows, audit action grouping. |
| API Endpoints | `GET /projects/:id/history`, `POST /projects/:id/snapshots`, `POST /projects/:id/snapshots/:snapshotId/restore`, `GET /projects/:id/audit-log`. |
| Request Example | `{ "label":"Before CSV import", "notes":"Manual checkpoint" }`. |
| Response Example | `{ "snapshot": {"id":"snap_1","createdAt":"2026-05-01T00:00:00Z"}, "auditLog": [] }`. |
| Complexity | High. |

## 3.29 Sales Dashboard

| Field | Details |
|---|---|
| Purpose | Aggregate sales intelligence across all platforms. |
| Functionality | Revenue, units, wishlists, average revenue/unit, platform splits, cumulative revenue, budget vs revenue, regional breakdown, wishlist time series. |
| Editable Fields | None directly; imports happen on platform pages. |
| Calculated Fields | Total revenue, total units, total wishlists, avg revenue/unit, revenue share %, units share %, cumulative revenue, ROI vs greenlit, regional totals. |
| API Endpoints | `GET /projects/:id/sales`. |
| Request Example | `GET /projects/prj_2k9l/sales?from=2026-01-01&to=2026-05-01`. |
| Response Example | `{ "summary": {"revenue":850000,"units":45000,"wishlists":120000}, "byPlatform": [], "regions": [], "wishlistSeries": [] }`. |
| Complexity | High. |

## 3.30 Per-Platform Sales Pages

| Field | Details |
|---|---|
| Purpose | Import and analyze sales by platform such as Steam, Xbox, PlayStation, Switch, Epic. |
| Functionality | Platform-specific import, revenue/units/wishlist analytics, region and monthly breakdowns. |
| Editable Fields | Uploaded CSV/import mapping; imported rows if editing is supported. |
| Calculated Fields | Platform revenue, units, refunds, net revenue, regional sales, monthly sales, wishlists, conversion metrics. |
| API Endpoints | `GET /projects/:id/sales/:platform`, `POST /projects/:id/sales/:platform/import`, `DELETE /projects/:id/sales/:platform/imports/:importId`. |
| Request Example | Multipart CSV upload to `POST /projects/prj_2k9l/sales/steam/import`. |
| Response Example | `{ "importId":"imp_1", "rowsImported":1200, "summary":{"revenue":500000,"units":30000} }`. |
| Complexity | High. |

## 3.31 Sales Promotions

| Field | Details |
|---|---|
| Purpose | Detect promotional periods and analyze pricing/media-spend correlation. |
| Functionality | Effective pricing analysis, median detection, promo period identification, media spend vs revenue. |
| Editable Fields | Promotion metadata if manually maintained; marketing spend comes from ledger/marketing data. |
| Calculated Fields | Median price, effective price, discount %, promotional windows, revenue uplift, media spend correlation. |
| API Endpoints | `GET /projects/:id/sales/promotions`, `POST /projects/:id/sales/promotions`, `PATCH /projects/:id/sales/promotions/:promoId`. |
| Request Example | `{ "name":"Spring Sale", "platform":"steam", "startDate":"2026-03-01", "endDate":"2026-03-14", "discountPct":30 }`. |
| Response Example | `{ "promotions": [], "analysis": {"medianPrice":24.99,"revenueDuringPromo":120000,"mediaSpend":30000} }`. |
| Complexity | High. |

## 3.32 Developer Portal — Budget

| Field | Details |
|---|---|
| Purpose | Let external developers submit budget changes without direct project write access. |
| Functionality | Scoped project budget grid, proposed changes, submit to approval queue. |
| Editable Fields | Proposed monthly values, notes/comment for submission. |
| Calculated Fields | Difference from current live values, proposed totals, submission status. |
| API Endpoints | `GET /developer-portal/:token/budget`, `POST /developer-portal/:token/budget-changes`. |
| Request Example | `{ "changes":[{"rowId":"row_1","month":"2026-04","proposedValue":50000}], "notes":"Updated staffing forecast" }`. |
| Response Example | `{ "submissionId":"sub_1","status":"pending_review" }`. |
| Complexity | High. |

## 3.33 Developer Portal — Invoice

| Field | Details |
|---|---|
| Purpose | Let developers submit invoices for review. |
| Functionality | Invoice upload and status creation in invoice queue. |
| Editable Fields | File, optional invoice metadata/comment. |
| Calculated Fields | Upload status, extraction status, queue status. |
| API Endpoints | `POST /developer-portal/:token/invoices`, `GET /developer-portal/:token/invoices/:invoiceId`. |
| Request Example | Multipart upload with invoice file and `{ "notes":"April milestone invoice" }`. |
| Response Example | `{ "invoiceId":"inv_1","status":"pending","submittedAt":"2026-05-01T00:00:00Z" }`. |
| Complexity | Medium. |

## 3.34 Developer Portal — History

| Field | Details |
|---|---|
| Purpose | Let developers track their own budget and invoice submissions. |
| Functionality | List submission history and statuses. |
| Editable Fields | None. |
| Calculated Fields | Status grouping, latest submission, submitted/approved/rejected counts. |
| API Endpoints | `GET /developer-portal/:token/history`. |
| Request Example | `GET /developer-portal/tok_abc/history`. |
| Response Example | `{ "budgetChanges": [], "invoices": [], "summary": {"pending":2,"approved":5,"rejected":1} }`. |
| Complexity | Low. |

---

# 4. Key Gaps / BA Observations from Conversation

| ID | Area | Gap / Observation | Severity |
|---|---|---|---|
| G1 | Access Control | Brief specifies six roles: Admin, Producer, Support, Finance, Developer, Viewer; code reportedly defines five and misses Viewer. | High |
| G2 | Backend | localStorage must be replaced by API/database. | High |
| G3 | Auth | Developer Portal currently uses URL params only; production needs scoped token/auth. | High |
| G4 | AI / Invoice | Anthropic key should move server-side; invoice files should use object storage. | High |
| G5 | Reconciliation | Auto-match by amount is partial; no full one-click match-all UI. | Medium |
| G6 | Ledger | Ledger has tabs/sort but no free-text search box according to conversation. | Low |
| G7 | Reports | `reports` is dead/redirect alias to `insights`. | Low |
| G8 | P&L | P&L math is duplicated across P&L, One Sheet, and print/export modal; backend should centralize calculation. | High |
| G9 | Marketing Budget | Client-side auto-seeding/migration of marketing categories should be moved to backend/project creation. | Medium |
| G10 | Reconciliation | Race conditions require optimistic locking on ledger allocation and month close. | High |

---

# 5. Recommended Endpoint Families

| Domain | Endpoint Family |
|---|---|
| Studio | `/studios/:id`, `/studios/:id/projects`, `/studios/:id/users`, `/studios/:id/roles`, `/studios/:id/platforms` |
| Project | `/projects`, `/projects/:id`, `/projects/:id/home`, `/projects/:id/settings` |
| Budget | `/projects/:id/budget`, `/projects/:id/budget/rows`, `/projects/:id/budget/cells`, `/projects/:id/master-budget`, `/projects/:id/marketing-budget` |
| Planning | `/projects/:id/phases`, `/projects/:id/milestones` |
| Greenlight | `/projects/:id/pnl`, `/projects/:id/one-sheet`, `/projects/:id/amendments`, `/projects/:id/greenlight` |
| Finance | `/projects/:id/ledger`, `/projects/:id/reconciliation`, `/projects/:id/invoices`, `/projects/:id/payment-schedule` |
| Reporting | `/projects/:id/dashboard`, `/projects/:id/insights`, `/projects/:id/variance`, `/projects/:id/rollups`, `/projects/:id/cost-reports` |
| Sales | `/projects/:id/sales`, `/projects/:id/sales/:platform`, `/projects/:id/sales/promotions` |
| Content | `/projects/:id/notes`, `/projects/:id/documents`, `/projects/:id/history`, `/projects/:id/audit-log` |
| Developer Portal | `/developer-portal/:token/budget`, `/developer-portal/:token/budget-changes`, `/developer-portal/:token/invoices`, `/developer-portal/:token/history` |

---

# 6. Notes on Request/Response Standard

All authenticated internal endpoints should use:

```http
Authorization: Bearer <token>
Content-Type: application/json
```

Financial mutation endpoints should return:

```json
{
  "data": {},
  "summary": {},
  "auditId": "aud_001",
  "etag": "v1",
  "updatedAt": "2026-05-01T00:00:00Z"
}
```

Conflict-sensitive endpoints, especially reconciliation, should support optimistic locking:

```http
If-Match: <etag>
```

Conflict response:

```json
{
  "error": "conflict",
  "message": "Ledger entry was modified by another reviewer.",
  "current": {}
}
```

---

# 7. BA Conclusion

Fictions Folio is a high-complexity financial planning and publishing operations platform. The core complexity is concentrated in:

1. Game Budget Grid
2. Reconciliation
3. P&L Projection
4. Amendments
5. Invoice Queue
6. Sales Intelligence

The most important backend design recommendation is to centralize all financial calculations — FX conversion, totals, variance, P&L, royalties, recoupment, and reporting outputs — into shared backend services so the UI renders consistent values across all pages.