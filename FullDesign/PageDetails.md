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

---

## API Endpoint Specifications

I'll use REST conventions, JSON payloads, and assume standard headers (`Authorization: Bearer <token>`, `Content-Type: application/json`). All timestamps are ISO 8601. All money values are integers in the project's base currency unit (USD cents would be safer, but to mirror the prototype I'll use whole dollars as numbers).

---

### 1. `GET /studios/:id`

Returns the studio record — org-level metadata plus a list of project IDs the requester has access to.

**Request**
```http
GET /studios/std_a8f3 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "id": "std_a8f3",
  "name": "Fictions",
  "createdAt": "2025-08-12T14:22:00Z",
  "updatedAt": "2026-04-30T09:15:00Z",
  "users": [
    {
      "id": "usr_001",
      "name": "Alex Chen",
      "email": "alex@fictions.com",
      "role": "admin",
      "projectIds": null,
      "createdAt": "2025-08-12T14:22:00Z"
    },
    {
      "id": "usr_007",
      "name": "Jamie Rivera",
      "email": "jamie@fictions.com",
      "role": "developer",
      "projectIds": ["prj_2k9l"],
      "createdAt": "2026-01-04T10:00:00Z"
    }
  ],
  "rolePages": {
    "admin": null,
    "producer": ["home", "budget", "pnl", "onesheet", "amendments", "ledger", "reconcile"],
    "developer": ["home", "budget", "milestones", "invoices"]
  },
  "platforms": ["Steam", "PS5", "Xbox Series X|S", "Nintendo Switch", "Epic"],
  "platformOptions": ["Steam", "Epic", "PS5", "PS6", "Xbox Series X|S", "Nintendo Switch", "Nintendo Switch 2", "iOS", "Android", "Mac", "Linux"],
  "projectIds": ["prj_2k9l", "prj_x4mp", "prj_99ab"],
  "currentUser": {
    "id": "usr_001",
    "role": "admin",
    "permissions": {
      "editBudget": true,
      "editPhases": true,
      "manageUsers": true,
      "deleteProject": true,
      "approveAmendments": true
    }
  }
}
```

**Notes for the backend**
- `projectIds` is filtered by the auth middleware. An admin sees all; a developer sees only the projects they're assigned to.
- `rolePages: null` means "no page restriction" (admin sees everything). An array means "only these page IDs are visible to this role."
- `currentUser` is included so the frontend can render permission-gated UI without a separate `/me` round-trip.

**Error responses**
- `401 Unauthorized` — missing/invalid token
- `403 Forbidden` — token valid but doesn't have access to this studio
- `404 Not Found` — studio doesn't exist

---

### 2. `GET /studios/:id/portfolio`

Returns pre-computed per-project summaries so the portfolio page doesn't have to load full project records and run client-side aggregation.

**Request**
```http
GET /studios/std_a8f3/portfolio HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "studioId": "std_a8f3",
  "computedAt": "2026-05-01T08:00:00Z",
  "projects": [
    {
      "id": "prj_2k9l",
      "name": "Covenant",
      "studio": "Fictions",
      "icon": "⚔️",
      "startDate": "2025-08",
      "monthCount": 36,
      "targetReleaseDate": "2028-08",
      "platforms": ["Steam", "PS5", "Xbox Series X|S"],
      "pointPerson": ["Alex Chen", "Jamie Rivera"],
      "summary": {
        "rowCount": 87,
        "ledgerEntryCount": 142,
        "greenlitVersion": 3,
        "greenlitDate": "2026-02-14",
        "greenlitTotal": 9750000,
        "liveTotal": 10215000,
        "variance": 465000,
        "variancePct": 4.77,
        "ledgerSpentToDate": 3120000,
        "currentPhase": {
          "id": "alpha",
          "name": "Alpha",
          "color": "#3730a3"
        },
        "elapsedMonths": 9,
        "currentMilestone": 9,
        "budgetHealth": "On Track",
        "amendmentNeeded": false,
        "pendingAmendments": 0,
        "pendingBudgetChanges": 1,
        "pendingInvoices": 3,
        "openMonths": 27
      },
      "createdAt": "2025-08-12T14:22:00Z",
      "updatedAt": "2026-04-30T17:42:00Z"
    },
    {
      "id": "prj_x4mp",
      "name": "Nightfall",
      "studio": "Fictions",
      "icon": "🌙",
      "startDate": "2026-01",
      "monthCount": 30,
      "targetReleaseDate": "2028-06",
      "platforms": ["Steam", "PS5"],
      "pointPerson": ["Sam Park"],
      "summary": {
        "rowCount": 42,
        "ledgerEntryCount": 18,
        "greenlitVersion": 1,
        "greenlitDate": "2026-01-04",
        "greenlitTotal": 4200000,
        "liveTotal": 4380000,
        "variance": 180000,
        "variancePct": 4.29,
        "ledgerSpentToDate": 285000,
        "currentPhase": {
          "id": "concept",
          "name": "Concept",
          "color": "#818cf8"
        },
        "elapsedMonths": 4,
        "currentMilestone": 4,
        "budgetHealth": "On Track",
        "amendmentNeeded": false,
        "pendingAmendments": 0,
        "pendingBudgetChanges": 0,
        "pendingInvoices": 0,
        "openMonths": 26
      },
      "createdAt": "2026-01-04T09:00:00Z",
      "updatedAt": "2026-04-29T11:20:00Z"
    }
  ]
}
```

**Notes for the backend**
- This is the endpoint that powers the portfolio cards. **Do not** make the frontend hit `GET /projects/:id` for each card and recompute totals — that's the prototype's behavior and it doesn't scale.
- `budgetHealth` follows the prototype's rule: "On Track" if `(spentToDate / greenlitTotal × 100) ≤ (elapsedMonths / monthCount × 100) + 5`, "Over Budget" if `> 100`, otherwise "At Risk".
- `amendmentNeeded` should be `Math.abs(variancePct) > 10 && greenlitLocked` — matches the chip on the project status bar.
- Cache aggressively. These numbers only change when a project is edited; invalidate the cache on writes to `rows`, `ledger`, or `masterBudget`.

**Error responses**
- `401`, `403`, `404` as above

---

### 3. `POST /projects`

Creates a new project. The request body matches what the Setup Wizard collects; all wizard fields are optional except `studioId` and `name`. The server fills in defaults for everything else.

**Request — minimal (Blank Project)**
```http
POST /projects HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "studioId": "std_a8f3",
  "name": "Untitled Game",
  "template": "blank"
}
```

**Request — full wizard payload**
```http
POST /projects HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "studioId": "std_a8f3",
  "name": "Eclipse",
  "studio": "Fictions",
  "startDate": "2026-09",
  "monthCount": 36,
  "multiCurrency": true,
  "altCurrency": "GBP",
  "pointPerson": ["Alex Chen", "Sam Park"],
  "platforms": ["Steam", "PS5", "Xbox Series X|S"],
  "targetReleaseDate": "2029-09",
  "icon": "🌑",
  "masterBudget": {
    "developmentCosts": 7500000,
    "portingCosts": 800000,
    "externalCosts": 1200000,
    "reserve": 300000,
    "contingency": 1000000,
    "marketing": 600000
  },
  "importedRows": [
    {
      "sectionId": "devfees",
      "subsectionId": "employees",
      "meta": {
        "role": "Lead Programmer",
        "department": "Engineering",
        "name": "Pat Singh"
      },
      "live": {
        "2026-09": 18000,
        "2026-10": 18000,
        "2026-11": 18000
      }
    },
    {
      "sectionId": "devfees",
      "subsectionId": "contractors_dev",
      "meta": {
        "role": "UI Contractor",
        "department": "Design",
        "name": ""
      },
      "live": {
        "2026-10": 12000,
        "2026-11": 12000
      }
    }
  ],
  "targetMode": "both"
}
```

**Response — 201 Created**
```json
{
  "id": "prj_q7w2",
  "studioId": "std_a8f3",
  "name": "Eclipse",
  "studio": "Fictions",
  "icon": "🌑",
  "startDate": "2026-09",
  "monthCount": 36,
  "multiCurrency": true,
  "altCurrency": "GBP",
  "pointPerson": ["Alex Chen", "Sam Park"],
  "platforms": ["Steam", "PS5", "Xbox Series X|S"],
  "targetReleaseDate": "2029-09",
  "masterBudget": {
    "developmentCosts": 7500000,
    "portingCosts": 800000,
    "externalCosts": 1200000,
    "reserve": 300000,
    "contingency": 1000000,
    "marketing": 600000
  },
  "phases": [
    { "id": "concept", "name": "Concept", "color": "#818cf8", "months": 3, "description": "" },
    { "id": "prototype", "name": "Prototype", "color": "#6366f1", "months": 4, "description": "" }
  ],
  "phaseAssignment": {
    "2026-09": "concept",
    "2026-10": "concept",
    "2026-11": "concept",
    "2026-12": "prototype"
  },
  "glStartDate": "2026-09",
  "glMonthCount": 36,
  "glPhases": [
    { "id": "concept", "name": "Concept", "color": "#818cf8", "months": 3, "description": "" }
  ],
  "glPhaseAssignment": {
    "2026-09": "concept",
    "2026-10": "concept"
  },
  "rows": [
    {
      "id": "row_n3kp",
      "sectionId": "devfees",
      "subsectionId": "employees",
      "meta": { "role": "Lead Programmer", "department": "Engineering", "name": "Pat Singh" },
      "live": { "2026-09": 18000, "2026-10": 18000, "2026-11": 18000 },
      "greenlit": { "2026-09": 18000, "2026-10": 18000, "2026-11": 18000 },
      "rates": {}
    }
  ],
  "greenlitVersion": 1,
  "greenlitDate": "2026-05-01",
  "greenlitLocked": true,
  "createdAt": "2026-05-01T10:30:00Z",
  "updatedAt": "2026-05-01T10:30:00Z",
  "createdBy": "usr_001"
}
```

**Notes for the backend**
- The server is responsible for: clamping `monthCount` to 1–120, generating `phaseAssignment` from the default phases, mirroring `live → greenlit` when `targetMode === "both"`, and seeding default rows when `template === "blank"` and no `importedRows` are provided.
- `template` accepts `"blank"`, `"demo"`, or `"wizard"` (default). Demo seeds sample ledger and sales data.
- `targetMode` controls how `importedRows` are written: `"both"` → live and greenlit both populated identically; `"live"` → only `live`; `"greenlit"` → only `greenlit`.
- The new project gets `greenlitVersion: 1` and `greenlitDate: today`. The greenlit budget is locked from creation.
- `Location` header should be set: `Location: /projects/prj_q7w2`.

**Error responses**
- `400 Bad Request` — invalid payload (missing name, malformed dates, monthCount out of range)
- `401`, `403` (caller can't create projects in this studio)
- `409 Conflict` — duplicate project name within studio (if you choose to enforce that)
- `422 Unprocessable Entity` — payload structurally valid but semantically wrong (e.g., `targetMode: "both"` but no `importedRows`)

---

### 4. `DELETE /projects/:id`

Soft-deletes a project. The record stays in the database with a `deletedAt` timestamp and stops appearing in portfolio queries; an admin-only endpoint can restore it within a retention window (e.g., 30 days).

**Request**
```http
DELETE /projects/prj_q7w2 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query params**
- `?reason=Cancelled+by+publisher` — captured in the audit log

**Response — 200 OK**
```json
{
  "id": "prj_q7w2",
  "name": "Eclipse",
  "deletedAt": "2026-05-01T11:00:00Z",
  "deletedBy": "usr_001",
  "deletionReason": "Cancelled by publisher",
  "scheduledPurgeAt": "2026-05-31T11:00:00Z",
  "restoreUrl": "/projects/prj_q7w2/restore",
  "auditEntryId": "aud_x9p4"
}
```

**Side effects**
- Creates a snapshot in `studio_history` capturing the full pre-deletion state (so restore is possible).
- Writes an audit entry: `{action: "Delete project", user, projectId, projectName, snapshotId}`.
- Removes `projectId` from any user's `projectIds` array.
- All `aux` data (sales, ledger, invoice queue, notes) is also soft-deleted with the same `deletedAt`.

**Notes for the backend**
- Hard delete (`DELETE /projects/:id?hard=true`) should require a separate, more restricted permission and should only be allowed after the soft-delete retention window passes — or never, depending on your retention policy.
- The frontend's "Are you sure?" confirm modal should still happen client-side; the API doesn't need a confirmation step.

**Error responses**
- `401`, `403` (caller doesn't have `deleteProject` permission)
- `404 Not Found` — already deleted or doesn't exist
- `409 Conflict` — project is in a state that prevents deletion (e.g., active reconciliation in progress)

---

### 5. `PATCH /projects/:id`

Partial update of project metadata. Used for the icon picker, name changes, point-person updates, platform changes, target release date, and any other field that doesn't go through a more specialized endpoint.

**What this endpoint should NOT touch:** `rows`, `ledger`, `phases`, `phaseAssignment`, `masterBudget`, `greenlit*` fields. Those need their own endpoints with their own validation and audit semantics (rows go through cell-edit endpoints; greenlit changes go through the amendment workflow; ledger goes through ledger endpoints).

**Request — icon update**
```http
PATCH /projects/prj_2k9l HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "icon": "🗡️"
}
```

**Request — multiple fields**
```http
PATCH /projects/prj_2k9l HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "name": "Covenant: Director's Cut",
  "targetReleaseDate": "2028-12",
  "platforms": ["Steam", "PS5", "PS6", "Xbox Series X|S", "Nintendo Switch 2"],
  "pointPerson": ["Alex Chen", "Sam Park", "Jamie Rivera"]
}
```

**Response — 200 OK**
```json
{
  "id": "prj_2k9l",
  "name": "Covenant: Director's Cut",
  "studio": "Fictions",
  "icon": "🗡️",
  "targetReleaseDate": "2028-12",
  "platforms": ["Steam", "PS5", "PS6", "Xbox Series X|S", "Nintendo Switch 2"],
  "pointPerson": ["Alex Chen", "Sam Park", "Jamie Rivera"],
  "updatedAt": "2026-05-01T11:30:00Z",
  "updatedBy": "usr_001",
  "auditEntryId": "aud_y3k7"
}
```

**Notes for the backend**
- Whitelist the fields this endpoint accepts. Reject anything else with `400` and a clear message about which endpoint to use instead. This prevents accidental mass-assignment vulnerabilities.
- Allowed fields: `name`, `studio` (the developer studio's name, not `studioId`), `icon`, `targetReleaseDate`, `platforms`, `pointPerson`, `multiCurrency`, `altCurrency`.
- `multiCurrency` and `altCurrency` updates should trigger a cascade check — if you turn off multi-currency, what happens to existing per-cell FX rates? Recommended behavior: keep the rates stored but ignore them in totals (so turning multi-currency back on later restores the prior state).
- Every successful PATCH writes an audit entry with the field name, old value, and new value.

**Error responses**
- `400 Bad Request` — body contains a field not allowed on this endpoint
- `401`, `403` (caller doesn't have `editSettings` permission)
- `404`, `409` (project is locked or being deleted)
- `422` — value invalid (e.g., `monthCount` outside 1–120 if you allow it here)

---

### 6. `PATCH /studios/:id`

Partial update of studio-level metadata. Most importantly: studio name. Also covers `platformOptions` (the list of platforms users can choose from when creating projects) and `rolePages` (the per-role page-visibility matrix).

**Request — rename studio**
```http
PATCH /studios/std_a8f3 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "name": "Fictions Games"
}
```

**Request — update role-page matrix**
```http
PATCH /studios/std_a8f3 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "rolePages": {
    "admin": null,
    "producer": ["home", "budget", "pnl", "onesheet", "amendments", "ledger", "reconcile", "invoices", "milestones"],
    "support": ["home", "budget", "pnl", "ledger"],
    "finance": null,
    "developer": ["home", "budget", "milestones", "invoices"],
    "viewer": ["home", "dashboard", "insights"]
  }
}
```

**Request — update platform options**
```http
PATCH /studios/std_a8f3 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "platformOptions": [
    "Steam", "Epic", "GOG",
    "PS5", "PS6",
    "Xbox Series X|S",
    "Nintendo Switch", "Nintendo Switch 2",
    "iOS", "Android"
  ]
}
```

**Response — 200 OK**
```json
{
  "id": "std_a8f3",
  "name": "Fictions Games",
  "platformOptions": [
    "Steam", "Epic", "GOG",
    "PS5", "PS6",
    "Xbox Series X|S",
    "Nintendo Switch", "Nintendo Switch 2",
    "iOS", "Android"
  ],
  "rolePages": {
    "admin": null,
    "producer": ["home", "budget", "pnl", "onesheet", "amendments", "ledger", "reconcile", "invoices", "milestones"],
    "support": ["home", "budget", "pnl", "ledger"],
    "finance": null,
    "developer": ["home", "budget", "milestones", "invoices"],
    "viewer": ["home", "dashboard", "insights"]
  },
  "updatedAt": "2026-05-01T12:00:00Z",
  "updatedBy": "usr_001",
  "auditEntryId": "aud_z2m1"
}
```

**Notes for the backend**
- Allowed fields: `name`, `platformOptions`, `rolePages`. User management belongs in dedicated `/studios/:id/users` endpoints, not here.
- `rolePages: null` for a role means "all pages visible." An array means "only these page IDs visible."
- Removing a platform from `platformOptions` does **not** remove it from existing projects — projects keep their platform tags even if the option is later disabled in the studio. This matters for backwards compatibility.
- Renaming the studio doesn't change the `id`. URLs and references stay stable.

**Error responses**
- `400 Bad Request` — disallowed fields, or empty `name`
- `401`, `403` — only `manageUsers` or higher can hit this endpoint
- `404`, `409`
- `422` — `rolePages` references unknown roles or unknown view IDs

---

## Authorization Middleware

Every endpoint above runs through middleware that enforces three layers:

```
1. Authentication       → valid bearer token? Identity established.
2. Studio membership    → does this user belong to studios/:id?
3. Permission check     → does the user's role grant the permission this endpoint requires?
4. Project filter       → for list endpoints, restrict the result to projects in the user's projectIds (or all, if projectIds is null/admin).
```

**Pseudocode for the project-filter step:**

```python
def filter_projects(user, projects):
    if user.role in ("admin", "finance"):
        return projects                                 # full access
    if user.projectIds is None:
        return projects                                 # null = no restriction
    return [p for p in projects if p.id in user.projectIds]
```

**Per-endpoint permission requirements:**

| Endpoint | Required permission |
|---|---|
| `GET /studios/:id` | studio member (any role) |
| `GET /studios/:id/portfolio` | studio member; results filtered by `projectIds` |
| `POST /projects` | `createProject` |
| `DELETE /projects/:id` | `deleteProject` AND project must be in user's `projectIds` |
| `PATCH /projects/:id` | `editSettings` AND project must be in user's `projectIds` |
| `PATCH /studios/:id` | `manageUsers` |

**Audit log:** every write endpoint should produce one audit row, captured server-side from the auth context (don't trust the client to identify itself in the body). The audit entry includes: timestamp, user ID, user name (denormalized at write time so renames don't break history), role at time of action, action verb, target ID, before/after values for changed fields.

**Idempotency keys:** `POST /projects` should accept an optional `Idempotency-Key` header. If the same key is seen twice within 24 hours, return the original response instead of creating a duplicate. This protects against double-clicks on the wizard's "Create Project" button.

---

Want me to spec out the project-internal endpoints next — `GET /projects/:id`, `PATCH /projects/:id/rows/:rowId/cells`, the amendment workflow endpoints, etc.? Those are where most of the business logic lives.


This is a substantial spec across three pages. Let me work through each one with the same structure: editable fields → calculated fields → endpoints with request/response samples.

---

# Page 1: Contracted Budget

(Doing this one first because it's the simplest, and Game Budget Grid depends on understanding it.)

## Editable Fields

The page is one form with **six numeric inputs** — that's the entire editable surface.

| Field | Type | Default | Storage path |
|---|---|---|---|
| Development Costs | Integer (USD) | 6,000,000 | `project.masterBudget.developmentCosts` |
| Porting Costs | Integer (USD) | 600,000 | `project.masterBudget.portingCosts` |
| External Support Costs | Integer (USD) | 1,400,000 | `project.masterBudget.externalCosts` |
| Reserve | Integer (USD) | 250,000 | `project.masterBudget.reserve` |
| Contingency | Integer (USD) | 1,000,000 | `project.masterBudget.contingency` |
| Marketing Costs | Integer (USD) | 500,000 | `project.masterBudget.marketing` |

Inline-editable: click the value, type, blur to save. The input strips non-numeric characters (`String(value).replace(/[^0-9.-]/g, "")`).

There is also an **Export** button (top-right) and a **per-row hover state** that reveals the input — these are UI affordances, not data fields.

## Calculated Fields

Everything else on the page is derived from those six inputs plus the Game Budget Grid's live totals.

| Field | Formula | Source |
|---|---|---|
| **Game Budget** (subtotal) | `developmentCosts + portingCosts + externalCosts` | The three cost lines that have a corresponding game-grid section |
| **Total Project Costs** (KPI banner) | `developmentCosts + portingCosts + externalCosts + reserve + contingency + marketing` | Sum of all six |
| **Grid total** (per line, three lines only) | `Σ row.live[m.key]` for all rows in the linked group, across all months | From the Game Budget Grid |
| **Variance** (per line, three lines only) | `gridValue − contractedValue` | Live grid minus what was contracted; red if positive (over) |
| **Percentage of total** (per line) | `contractedValue / totalProjectCosts × 100` | Drives the colored progress bar on each line |
| **Summary table %** | Same `(line / total) × 100` | The percentage column in the bottom summary table |

**The grid-link mapping is hardcoded** (lines 3695–3702):

```
Development Costs   → group "devcosts"
Porting Costs       → group "porting"
External Support    → group "external"
Reserve             → no link
Contingency         → no link
Marketing Costs     → no link (links to Marketing Budget page instead)
```

Reserve, Contingency, and Marketing have no grid comparison because they aren't bottom-up forecasted in the main grid.

## Endpoints

### `GET /projects/:id/master-budget`

Returns the contracted budget plus the live grid totals needed for the variance display, so the page renders in one round-trip.

**Request**
```http
GET /projects/prj_2k9l/master-budget HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "masterBudget": {
    "developmentCosts": 6000000,
    "portingCosts": 600000,
    "externalCosts": 1400000,
    "reserve": 250000,
    "contingency": 1000000,
    "marketing": 500000
  },
  "computed": {
    "gameBudget": 8000000,
    "totalProjectCosts": 9750000,
    "lines": [
      {
        "key": "developmentCosts",
        "label": "Development Costs",
        "contracted": 6000000,
        "gridGroup": "devcosts",
        "gridLiveTotal": 6240000,
        "variance": 240000,
        "pctOfTotal": 61.54
      },
      {
        "key": "portingCosts",
        "label": "Porting Costs",
        "contracted": 600000,
        "gridGroup": "porting",
        "gridLiveTotal": 580000,
        "variance": -20000,
        "pctOfTotal": 6.15
      },
      {
        "key": "externalCosts",
        "label": "External Support Costs",
        "contracted": 1400000,
        "gridGroup": "external",
        "gridLiveTotal": 1395000,
        "variance": -5000,
        "pctOfTotal": 14.36
      },
      {
        "key": "reserve",
        "label": "Reserve",
        "contracted": 250000,
        "gridGroup": null,
        "gridLiveTotal": null,
        "variance": null,
        "pctOfTotal": 2.56
      },
      {
        "key": "contingency",
        "label": "Contingency",
        "contracted": 1000000,
        "gridGroup": null,
        "gridLiveTotal": null,
        "variance": null,
        "pctOfTotal": 10.26
      },
      {
        "key": "marketing",
        "label": "Marketing Costs",
        "contracted": 500000,
        "gridGroup": null,
        "gridLiveTotal": null,
        "variance": null,
        "pctOfTotal": 5.13
      }
    ]
  },
  "updatedAt": "2026-04-30T16:22:00Z"
}
```

### `PATCH /projects/:id/master-budget`

Partial update. Send only the keys that changed — the server merges into existing.

**Request**
```http
PATCH /projects/prj_2k9l/master-budget HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "developmentCosts": 6500000,
  "marketing": 600000
}
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "masterBudget": {
    "developmentCosts": 6500000,
    "portingCosts": 600000,
    "externalCosts": 1400000,
    "reserve": 250000,
    "contingency": 1000000,
    "marketing": 600000
  },
  "computed": {
    "gameBudget": 8500000,
    "totalProjectCosts": 10350000
  },
  "updatedAt": "2026-05-01T13:00:00Z",
  "updatedBy": "usr_001",
  "auditEntryId": "aud_b4n9"
}
```

**Notes**
- One audit entry per field changed: `{action: "Edit contracted budget", field: "developmentCosts", oldValue: 6000000, newValue: 6500000}`.
- Permission required: `editSettings`.
- All values must be `>= 0` integers. Reject negatives with `400`.
- This endpoint does **not** modify the grid. The Master Budget is a separate planning surface; bottom-up grid totals can drift from it (that's the whole point of variance display).

---

# Page 2: Phases

## Editable Fields

Phases are managed in two parallel timelines — **Live Phases** and **Greenlit Phases** — each with the same fields. The mode toggle at the top (`phaseMode` state) chooses which set you're editing.

**Per-phase, editable fields:**

| Field | Type | Default | Storage path |
|---|---|---|---|
| Color | Hex color (color picker) | from default palette | `phase.color` |
| Name | Text | "Concept" / "Prototype" / etc. | `phase.name` |
| Months (duration) | Integer ≥ 0 | 3, 4, 3, 5, 8, 5, 4, 3, 1 (defaults) | `phase.months` |
| Description | Multiline text | "" | `phase.description` |
| Order | Implicit (drag-to-reorder) | Default order | Index in `phases[]` array |

**Per-phase, page-level actions:**
- Drag handle: reorders phases (updates the array index).
- Expand/collapse: UI-only, no persistence.
- Delete: removes the phase entirely. Disabled when only 1 phase remains.
- Add Phase button (top of list): creates a new phase with a default name like "New Phase" and 1 month.

**Mode-specific:**
- **Sync from Live → Greenlit** (or reverse): copies all phases from one mode to the other after confirmation.
- **Lock/Unlock Phases** (Greenlit only, producer/admin): toggles `glPhasesUnlocked`. While locked, all greenlit phase fields are read-only.

## Calculated Fields

| Field | Formula | Source |
|---|---|---|
| **Total Allocated months** | `Σ phases[].months` | Sum across all phases |
| **Remaining** | `project.monthCount − totalAllocated` | Negative if over-allocated; positive if under |
| **Allocation status** | "Unallocated N months" / "N months over" / OK | Based on `remaining` sign |
| **Per-phase Start Month** | Cursor-walked: phase 1 starts at `ms[0]`; phase N starts at `ms[Σ phases[0..N-1].months]` | Walks the timeline left-to-right, consuming each phase's duration |
| **Per-phase End Month** | `ms[startIdx + months − 1]` | Last month consumed by the phase |
| **Per-phase Live Budget** | `Σ grandTotal("live", m.key)` for months in the phase's index range | From the budget grid |
| **`phaseAssignment` map** | `{monthKey: phaseId}` map built by walking phases in order | Auto-rebuilt every time phases or month count change (`buildPhaseAssignment`, line 81) |
| **Visual timeline bar widths** | `phase.months / project.monthCount × 100%` | Per-phase percentage of total timeline |

**The crucial derived structure is `phaseAssignment`.** It's not stored as user input — it's regenerated whenever phases change. Every other page in the app (Payment Schedule, Insights, Reports, Milestones, Phase Rollup) reads `phaseAssignment[monthKey]` to know which phase a month belongs to.

## Endpoints

### `GET /projects/:id/phases`

Returns both phase sets in one response — the page needs both for the toggle to work.

**Request**
```http
GET /projects/prj_2k9l/phases HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "monthCount": 36,
  "startDate": "2025-08",
  "live": {
    "phases": [
      {
        "id": "concept",
        "name": "Concept",
        "color": "#818cf8",
        "months": 3,
        "description": "Initial pitch development, story bible, and prototype planning."
      },
      {
        "id": "prototype",
        "name": "Prototype",
        "color": "#6366f1",
        "months": 4,
        "description": ""
      },
      {
        "id": "vs",
        "name": "VS",
        "color": "#4f46e5",
        "months": 3,
        "description": ""
      }
    ],
    "phaseAssignment": {
      "2025-08": "concept",
      "2025-09": "concept",
      "2025-10": "concept",
      "2025-11": "prototype"
    },
    "computed": {
      "totalAllocated": 36,
      "remaining": 0,
      "ranges": [
        {
          "id": "concept",
          "startMonth": "Aug 25",
          "endMonth": "Oct 25",
          "startIdx": 0,
          "endIdx": 2,
          "liveBudget": 540000
        },
        {
          "id": "prototype",
          "startMonth": "Nov 25",
          "endMonth": "Feb 26",
          "startIdx": 3,
          "endIdx": 6,
          "liveBudget": 920000
        }
      ]
    }
  },
  "greenlit": {
    "phases": [
      {
        "id": "concept",
        "name": "Concept",
        "color": "#818cf8",
        "months": 3,
        "description": ""
      }
    ],
    "phaseAssignment": {
      "2025-08": "concept",
      "2025-09": "concept",
      "2025-10": "concept"
    },
    "computed": {
      "totalAllocated": 36,
      "remaining": 0,
      "ranges": []
    }
  },
  "greenlitLocked": true,
  "updatedAt": "2026-04-22T11:14:00Z"
}
```

### `PUT /projects/:id/phases/:mode`

Replaces the entire phase array for a mode (`live` or `greenlit`). Use PUT (full replacement) rather than PATCH because reordering is a common operation and per-field patching gets ambiguous when array order is itself meaningful.

**Request**
```http
PUT /projects/prj_2k9l/phases/live HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "phases": [
    {
      "id": "concept",
      "name": "Concept",
      "color": "#818cf8",
      "months": 3,
      "description": "Initial pitch and story bible."
    },
    {
      "id": "prototype",
      "name": "Prototype",
      "color": "#6366f1",
      "months": 6,
      "description": "Playable prototype + greybox levels."
    },
    {
      "id": "vs",
      "name": "VS",
      "color": "#4f46e5",
      "months": 3,
      "description": ""
    }
  ]
}
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "mode": "live",
  "phases": [
    { "id": "concept", "name": "Concept", "color": "#818cf8", "months": 3, "description": "Initial pitch and story bible." },
    { "id": "prototype", "name": "Prototype", "color": "#6366f1", "months": 6, "description": "Playable prototype + greybox levels." },
    { "id": "vs", "name": "VS", "color": "#4f46e5", "months": 3, "description": "" }
  ],
  "phaseAssignment": {
    "2025-08": "concept",
    "2025-09": "concept",
    "2025-10": "concept",
    "2025-11": "prototype",
    "2025-12": "prototype"
  },
  "computed": {
    "totalAllocated": 12,
    "remaining": 24
  },
  "warnings": [
    "24 months unassigned — consider adding more phases or extending existing ones."
  ],
  "updatedAt": "2026-05-01T13:30:00Z",
  "updatedBy": "usr_001",
  "auditEntryId": "aud_p9k2"
}
```

**Notes**
- The server **always** rebuilds `phaseAssignment` from the new phase array. Don't accept `phaseAssignment` in the request — it's a derived field.
- `warnings` is a non-blocking advisory: under-allocation or over-allocation is allowed (so users can save in-progress work), but the UI should highlight it.
- `mode` path param: `"live"` or `"greenlit"`.
- For `mode: "greenlit"`, require `editPhases` permission AND check that `greenlitLocked === false` OR the user has `manageUsers` (admin override). Reject with `409 Conflict` if locked.
- Each phase needs a stable `id`. New phases sent without `id` get one assigned server-side; existing IDs are preserved.

### `POST /projects/:id/phases/:mode/sync`

Copies one mode's phases to the other.

**Request**
```http
POST /projects/prj_2k9l/phases/live/sync HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "from": "greenlit"
}
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "syncedTo": "live",
  "syncedFrom": "greenlit",
  "phaseCount": 9,
  "auditEntryId": "aud_q1n5"
}
```

### `PATCH /projects/:id/phases/greenlit/lock`

Toggles the greenlit-phases lock.

**Request**
```http
PATCH /projects/prj_2k9l/phases/greenlit/lock HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "locked": false
}
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "greenlitLocked": false,
  "unlockedAt": "2026-05-01T14:00:00Z",
  "unlockedBy": "usr_001",
  "auditEntryId": "aud_r3p7"
}
```

---

# Page 3: Game Budget Grid

This is the most complex page in the application. The data model has more dimensions, the editing patterns are richer, and the FX cascade applies to almost every read.

## Editable Fields

Editing happens at three levels: **row metadata**, **cell values**, and **per-cell FX rates**.

### Row Metadata (per row)

Each row sits in a section/subsection that determines which meta fields it has. There are two row "shapes":

**For "person/role" subsections** (`employees`, `contractors_dev`, `contractors_fictions`):

| Field | Type | Storage path |
|---|---|---|
| Role / Title | Text | `row.meta.role` |
| Department | Text (free) or dropdown | `row.meta.department` |
| Name | Text | `row.meta.name` |

**For "item/cost" subsections** (`oneoff_dev`, `operating_dev`, `oneoff_fictions`, `porting_items`, `external_items`):

| Field | Type | Storage path |
|---|---|---|
| Item | Text | `row.meta.item` |
| Category | Text (free) or dropdown | `row.meta.category` |
| Detail | Text | `row.meta.detail` |

**Plus, on every row:**

| Field | Type | Storage path |
|---|---|---|
| Section assignment | Dropdown (one of the 6 sections) | `row.sectionId` |
| Subsection assignment | Dropdown (one of the 8 subsections) | `row.subsectionId` |
| Currency override (multi-currency mode only) | "USD" / alt-currency-code | `project.rowCurrencyOverrides[row.id]` |

### Cell Values

The grid's cells are 2D: `(rowId, monthKey)`. Each row has two parallel cell maps:

| Field | Type | Storage path | When editable |
|---|---|---|---|
| Live cell | Number (USD or alt-currency native) | `row.live[monthKey]` | Always (when month not closed) |
| Greenlit cell | Number (USD or alt-currency native) | `row.greenlit[monthKey]` | Only during an active amendment draft, OR when greenlight is unlocked |

**Cell editing patterns the page supports:**
- Single-cell click + edit
- Click-drag to multi-select
- Multi-cell paste from Excel (clipboard parser auto-fills a rectangular range)
- Bulk row actions: copy across, fill, clear
- Right-click context menu with cut/copy/paste/clear

### Per-Cell FX Rates (multi-currency only)

| Field | Type | Storage path |
|---|---|---|
| Per-cell FX rate | Number (e.g. 1.08) | `row.rates[monthKey]` |
| Per-column spot rate | Number | `project.spotRates[monthKey]` |
| Global spot rate | Number | `project.globalSpotRate` |

Rates can be edited inline (in the FX panel), in batch (apply one rate to N rows), or by typing a USD amount and back-solving the rate.

### Project-Level Toolbar Controls

| Field | Type | Storage path |
|---|---|---|
| Budget mode toggle | "live" / "greenlit" | UI state, not persisted per-user |
| Currency view toggle | "usd" / "native" | UI state |
| Multi-currency on/off | Boolean | `project.multiCurrency` (set in Settings, not the grid) |

### Month Operations (cross-cutting)

| Action | Storage effect |
|---|---|
| Close a month | Adds `project.closedMonths[mKey] = {closedAt, closedBy}`. Cells in closed months become read-only. |
| Reopen a month | Removes the entry. |

## Calculated Fields

The grid is essentially a giant pivot table; almost every visible number except the leaf cells is calculated. The full primitive list is in [the analysis document](#cross-cutting-calculation-primitives), but here are the per-page derived values:

| Field | Formula |
|---|---|
| **Row total** | `Σ row[mode][m.key]` across all months |
| **Subsection total (per month)** | `Σ rowsInSubsection[mode][m.key]` |
| **Section total (per month)** | `sectionTotal(sId, mode, m.key)` |
| **Group total (per month)** | `groupTotal(gId, mode, m.key)` |
| **Grand total (per month)** | `grandTotal(mode, m.key)` |
| **Year total (per column)** | `yearTotal(mode, year, sectionId?)` — sums months where `m.year === year` |
| **Section/group/grand totals across full timeline** | Sum of per-month totals |
| **USD-converted equivalents** | Native value × `cellRate(row, m.key)` if row is alt-currency |
| **Effective totals (the values rendered)** | `eff*Total` — switches between USD and native depending on `multiCurrency` and `currencyView` |
| **`liveAll` / `greenlitAll`** | Sum of grand totals across all months for that mode |
| **`variance`** | `liveAll − greenlitAll` |
| **Variance per cell** | `live[mKey] − greenlit[mKey]` (drives red/green cell highlighting) |
| **Blended FX rate (per month)** | `Σ(native × rate) / Σ(native)` across alt rows — used for batch-rate display |
| **Phase bar (top of grid)** | Maps each month to `phaseFor(m.key)` and renders a colored band |
| **Status bar variance %** | `(variance / greenlitAll) × 100` — drives the "Amendment Needed" chip when `> 10%` |

**Critical:** the FX cascade — `cellRate → cellUSD → effGrandTotal` — means that one cell can have its rendered value affected by edits at three different levels (the cell itself, the column spot rate, the global rate). This is the single hardest piece of business logic to port to the backend correctly.

## Endpoints

### `GET /projects/:id/grid`

Returns everything the grid needs in one response: rows, both budget modes, FX state, closed months, computed totals.

**Request**
```http
GET /projects/prj_2k9l/grid HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query params**
- `?mode=live` — return only live values (skip greenlit) — minor payload size win
- `?totals=true` (default) — include computed totals

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "startDate": "2025-08",
  "monthCount": 36,
  "months": [
    { "key": "2025-08", "label": "Aug 25", "year": 2025, "phaseId": "concept" },
    { "key": "2025-09", "label": "Sep 25", "year": 2025, "phaseId": "concept" },
    { "key": "2025-10", "label": "Oct 25", "year": 2025, "phaseId": "concept" }
  ],
  "currency": {
    "multiCurrency": true,
    "altCurrency": "GBP",
    "globalSpotRate": 1.27,
    "spotRates": {
      "2025-08": 1.28,
      "2025-09": 1.275
    }
  },
  "closedMonths": {
    "2025-08": { "closedAt": "2025-09-05T10:00:00Z", "closedBy": "Alex Chen" },
    "2025-09": { "closedAt": "2025-10-04T11:30:00Z", "closedBy": "Alex Chen" }
  },
  "rows": [
    {
      "id": "row_n3kp",
      "sectionId": "devfees",
      "subsectionId": "employees",
      "meta": {
        "role": "Lead Programmer",
        "department": "Engineering",
        "name": "Pat Singh"
      },
      "currencyOverride": null,
      "live": {
        "2025-08": 18000,
        "2025-09": 18000,
        "2025-10": 18500
      },
      "greenlit": {
        "2025-08": 18000,
        "2025-09": 18000,
        "2025-10": 18000
      },
      "rates": {}
    },
    {
      "id": "row_q4mr",
      "sectionId": "devfees",
      "subsectionId": "contractors_dev",
      "meta": {
        "role": "UI Contractor",
        "department": "Design",
        "name": "External Studio Ltd"
      },
      "currencyOverride": "GBP",
      "live": {
        "2025-09": 8000,
        "2025-10": 8000
      },
      "greenlit": {
        "2025-09": 8000,
        "2025-10": 8000
      },
      "rates": {
        "2025-09": 1.275,
        "2025-10": 1.27
      }
    }
  ],
  "totals": {
    "live": {
      "byMonth": {
        "2025-08": 540000,
        "2025-09": 562000,
        "2025-10": 558000
      },
      "byGroup": {
        "devcosts": { "byMonth": { "2025-08": 320000, "2025-09": 332000 }, "total": 6240000 },
        "porting": { "byMonth": { "2025-08": 0, "2025-09": 0 }, "total": 580000 },
        "external": { "byMonth": { "2025-08": 220000, "2025-09": 230000 }, "total": 1395000 }
      },
      "bySection": {
        "devfees": { "total": 5100000 },
        "fictions": { "total": 1140000 }
      },
      "grandTotal": 8215000
    },
    "greenlit": {
      "byMonth": { "2025-08": 540000, "2025-09": 540000 },
      "byGroup": {
        "devcosts": { "total": 6000000 },
        "porting": { "total": 600000 },
        "external": { "total": 1400000 }
      },
      "grandTotal": 8000000
    },
    "variance": 215000,
    "variancePct": 2.69
  },
  "greenlitVersion": 3,
  "greenlitLocked": true,
  "greenlitEditing": false,
  "updatedAt": "2026-04-30T17:42:00Z"
}
```

### `PATCH /projects/:id/grid/cells`

Bulk cell update — the high-traffic endpoint. Accepts an array of cell edits in one request to support multi-cell paste, fill-across, and rapid editing. Server applies them transactionally.

**Request**
```http
PATCH /projects/prj_2k9l/grid/cells HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "mode": "live",
  "edits": [
    { "rowId": "row_n3kp", "monthKey": "2025-11", "value": 18500 },
    { "rowId": "row_n3kp", "monthKey": "2025-12", "value": 18500 },
    { "rowId": "row_n3kp", "monthKey": "2026-01", "value": 19000 },
    { "rowId": "row_q4mr", "monthKey": "2025-11", "value": 8000 }
  ]
}
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "mode": "live",
  "applied": 4,
  "rejected": [],
  "totals": {
    "live": {
      "grandTotal": 8268500,
      "variance": 268500,
      "variancePct": 3.36
    }
  },
  "auditEntryIds": ["aud_c5h1", "aud_c5h2", "aud_c5h3", "aud_c5h4"],
  "updatedAt": "2026-05-01T15:00:00Z"
}
```

**Response — 207 Multi-Status (partial failure)**
```json
{
  "projectId": "prj_2k9l",
  "mode": "live",
  "applied": 2,
  "rejected": [
    {
      "rowId": "row_n3kp",
      "monthKey": "2025-08",
      "value": 19000,
      "reason": "MONTH_CLOSED",
      "message": "August 2025 is closed. Reopen the month to edit."
    },
    {
      "rowId": "row_q4mr",
      "monthKey": "2025-09",
      "value": 8500,
      "reason": "MONTH_CLOSED",
      "message": "September 2025 is closed."
    }
  ],
  "totals": {
    "live": { "grandTotal": 8243500, "variance": 243500 }
  },
  "updatedAt": "2026-05-01T15:01:00Z"
}
```

**Notes**
- `mode: "live"` requires `editBudget`; `mode: "greenlit"` requires `editBudget` AND active amendment draft AND `greenlitEditing === true` (or admin override).
- `value: null` clears the cell (different from `0`, which keeps a zero entry).
- Closed-month rejections are reported per-cell, not all-or-nothing — partial application is the right behavior because a paste might span open and closed months.
- For multi-currency rows, `value` is the **native** value. The server applies the FX cascade for any computed total it returns.
- The response includes refreshed grand totals so the frontend's status bar updates without a separate fetch.

### `POST /projects/:id/grid/rows`

Add a new row.

**Request**
```http
POST /projects/prj_2k9l/grid/rows HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "sectionId": "devfees",
  "subsectionId": "employees",
  "meta": {
    "role": "Audio Designer",
    "department": "Audio",
    "name": ""
  }
}
```

**Response — 201 Created**
```json
{
  "id": "row_w8x2",
  "sectionId": "devfees",
  "subsectionId": "employees",
  "meta": { "role": "Audio Designer", "department": "Audio", "name": "" },
  "currencyOverride": null,
  "live": {},
  "greenlit": {},
  "rates": {},
  "createdAt": "2026-05-01T15:30:00Z",
  "createdBy": "usr_001",
  "auditEntryId": "aud_d7m3"
}
```

### `PATCH /projects/:id/grid/rows/:rowId`

Update row metadata, section/subsection, or currency override.

**Request**
```http
PATCH /projects/prj_2k9l/grid/rows/row_w8x2 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "meta": {
    "name": "Devon Lee"
  },
  "currencyOverride": "GBP"
}
```

**Response — 200 OK**
```json
{
  "id": "row_w8x2",
  "sectionId": "devfees",
  "subsectionId": "employees",
  "meta": { "role": "Audio Designer", "department": "Audio", "name": "Devon Lee" },
  "currencyOverride": "GBP",
  "updatedAt": "2026-05-01T15:35:00Z",
  "auditEntryId": "aud_e1k4"
}
```

**Notes**
- Meta is partial-merge: only the keys you send are updated.
- Changing `sectionId` or `subsectionId` is allowed but logged distinctly in audit because it relocates the row across the budget hierarchy.
- `currencyOverride: null` removes the override (reverts to the section's default behavior).

### `DELETE /projects/:id/grid/rows/:rowId`

Remove a row. This is destructive — the row's cell values are gone too. (No soft-delete here; users are expected to use the snapshot system if they need to recover.)

**Request**
```http
DELETE /projects/prj_2k9l/grid/rows/row_w8x2 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "id": "row_w8x2",
  "deleted": true,
  "snapshotId": "snap_g9p1",
  "auditEntryId": "aud_f2n5"
}
```

**Notes**
- Server auto-creates a snapshot before deletion (matching the prototype's "snapshot before destructive action" pattern).
- Permission: `deleteRows`.

### `PATCH /projects/:id/grid/fx`

Update FX rates — at the global, column, or per-cell level. Accepts any combination in one request.

**Request — set a column rate and a few per-cell rates**
```http
PATCH /projects/prj_2k9l/grid/fx HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "globalSpotRate": null,
  "spotRates": {
    "2025-11": 1.272,
    "2025-12": 1.265
  },
  "cellRates": [
    { "rowId": "row_q4mr", "monthKey": "2025-11", "rate": 1.275 },
    { "rowId": "row_q4mr", "monthKey": "2025-12", "rate": 1.27 }
  ]
}
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "currency": {
    "globalSpotRate": null,
    "spotRates": {
      "2025-08": 1.28,
      "2025-09": 1.275,
      "2025-11": 1.272,
      "2025-12": 1.265
    }
  },
  "updatedCells": 2,
  "totals": {
    "live": { "grandTotal": 8268500, "variance": 268500 }
  },
  "auditEntryId": "aud_h4j2"
}
```

**Notes**
- `globalSpotRate: null` means "no global rate" — the cascade falls back to `1.0`.
- `cellRates: rate: null` removes a per-cell rate.
- Updating FX rates does not change stored cell values — it changes how they're rendered when `currencyView === "usd"`. Totals will reflect the new cascade immediately.

### `POST /projects/:id/grid/months/:monthKey/close`

Close a month. Cells in closed months become read-only.

**Request**
```http
POST /projects/prj_2k9l/grid/months/2025-10/close HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "monthKey": "2025-10",
  "closedAt": "2026-05-01T16:00:00Z",
  "closedBy": "usr_001",
  "auditEntryId": "aud_i7l8"
}
```

### `DELETE /projects/:id/grid/months/:monthKey/close`

Reopen a closed month.

**Request**
```http
DELETE /projects/prj_2k9l/grid/months/2025-10/close HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "monthKey": "2025-10",
  "reopenedAt": "2026-05-01T16:30:00Z",
  "reopenedBy": "usr_001",
  "auditEntryId": "aud_j9m0"
}
```

### `POST /projects/:id/grid/import`

CSV import. Returns a parsed preview with auto-detected sections; client confirms or adjusts before commit.

**Request — preview**
```http
POST /projects/prj_2k9l/grid/import?action=preview HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: multipart/form-data

(CSV file as form data field "file")
```

**Response — 200 OK**
```json
{
  "previewId": "prv_n3p7",
  "fileName": "Q3_budget.csv",
  "detectedMonths": ["2025-08", "2025-09", "2025-10", "2025-11"],
  "rows": [
    {
      "tempId": "tmp_001",
      "sectionId": "devfees",
      "subsectionId": "employees",
      "meta": { "role": "Lead Programmer", "department": "Engineering", "name": "" },
      "live": { "2025-08": 18000, "2025-09": 18000 },
      "confidence": "high"
    },
    {
      "tempId": "tmp_002",
      "sectionId": "external_sec",
      "subsectionId": "external_items",
      "meta": { "item": "Localization", "category": "Loc", "detail": "" },
      "live": { "2025-08": 5000 },
      "confidence": "low",
      "warnings": ["Section assignment uncertain — please confirm."]
    }
  ],
  "expiresAt": "2026-05-01T17:00:00Z"
}
```

**Request — commit**
```http
POST /projects/prj_2k9l/grid/import?action=commit HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "previewId": "prv_n3p7",
  "targetMode": "live",
  "rows": [
    { "tempId": "tmp_001", "sectionId": "devfees", "subsectionId": "employees" },
    { "tempId": "tmp_002", "sectionId": "external_sec", "subsectionId": "external_items" }
  ]
}
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "imported": 2,
  "skipped": 0,
  "snapshotId": "snap_x4n9",
  "auditEntryId": "aud_k1p2"
}
```

**Notes**
- The two-step preview/commit pattern mirrors the wizard's CSV importer flow — the user gets to review and correct before any rows are written.
- `previewId` expires after a short window (15–30 min); after that, the user has to re-upload.
- An auto-snapshot is created before commit (destructive action).
- Permission: `importCSV`.

---

## Cross-Cutting: WebSocket / Server-Sent Events

The grid is a multi-user editing surface — the brief explicitly calls for "Real-time or optimistic updates for multi-user editing" (Phase 2 deliverable). I haven't included that in the REST endpoints above, but here's the basic shape:

**Subscribe**
```
ws://api.fictions-folio.com/projects/prj_2k9l/grid/subscribe
```

**Server pushes (on any other user's edit)**
```json
{
  "type": "cell_updated",
  "rowId": "row_n3kp",
  "monthKey": "2025-11",
  "mode": "live",
  "value": 18500,
  "updatedBy": "usr_007",
  "updatedAt": "2026-05-01T15:00:00Z"
}
```

The frontend applies these updates to its local state, optimistically reconciling with any in-flight local edits (last-write-wins is the simplest model; operational-transform or CRDT is the rigorous one).

---

## Permissions Summary

| Endpoint | Required permission | Additional gates |
|---|---|---|
| `GET /projects/:id/master-budget` | studio member, project access | — |
| `PATCH /projects/:id/master-budget` | `editSettings` | — |
| `GET /projects/:id/phases` | studio member, project access | — |
| `PUT /projects/:id/phases/live` | `editPhases` | — |
| `PUT /projects/:id/phases/greenlit` | `editPhases` | `greenlitLocked === false` OR admin |
| `POST /projects/:id/phases/:mode/sync` | `editPhases` | (target mode's lock state) |
| `PATCH /projects/:id/phases/greenlit/lock` | `editSettings` | — |
| `GET /projects/:id/grid` | studio member, project access | — |
| `PATCH /projects/:id/grid/cells` (live) | `editBudget` | Month must not be closed |
| `PATCH /projects/:id/grid/cells` (greenlit) | `editBudget` | Active amendment draft |
| `POST /projects/:id/grid/rows` | `editBudget` | — |
| `PATCH /projects/:id/grid/rows/:id` | `editBudget` | — |
| `DELETE /projects/:id/grid/rows/:id` | `deleteRows` | — |
| `PATCH /projects/:id/grid/fx` | `editBudget` | `multiCurrency === true` |
| `POST .../months/:key/close` | `editLedger` (closing is a finance op) | All entries reconciled |
| `DELETE .../months/:key/close` | `editLedger` | — |
| `POST .../grid/import` | `importCSV` | — |

---

# Page 4: P&L Projection

This is the most computation-heavy page in the system. The math drives every greenlight decision and feeds the One Sheet.

## Editable Fields

The page captures inputs across five logical groups: cost structure, partner economics, fees & pricing, developer royalty tiers, and licensor royalty tiers, plus scenario inputs. All values are stored on the project as `glPnl` (the greenlit P&L state).

### Cost Structure

| Field | Type | Default | Storage path |
|---|---|---|---|
| Has Publishing Partner | Boolean toggle | false | `glPnl.hasPubPartner` |
| Publishing Partner Name | Text | "" | `glPnl.pubPartnerName` |
| Development Budget | Integer (USD) | from masterBudget | `glPnl.devBudget` |
| Porting Budget | Integer (USD) | from masterBudget | `glPnl.portingBudget` |
| External Support Budget | Integer (USD) | from masterBudget | `glPnl.externalBudget` |
| Reserve Budget | Integer (USD) | from masterBudget | `glPnl.reserveBudget` |
| Contingency Budget | Integer (USD) | from masterBudget | `glPnl.contingencyBudget` |
| Marketing Budget | Integer (USD) | from masterBudget | `glPnl.marketingBudget` |
| Upfront Engine Fee | Integer (USD) | 0 | `glPnl.upfrontEngineFee` |
| Publishing Fee Rate | Percentage (0–1) | 0.10 | `glPnl.publishingFeeRate` |

### Partner Contributions (only when `hasPubPartner = true`)

| Field | Type | Default | Storage path |
|---|---|---|---|
| Partner contribution per cost line | Integer (USD) | 0 | `glPnl.pubContrib.{devBudget, portingBudget, externalBudget, reserveBudget, contingencyBudget, marketingBudget}` |

### Fees & Pricing

| Field | Type | Default | Storage path |
|---|---|---|---|
| Platform Fee | Percentage (0–1) | 0.30 | `glPnl.platformFee` |
| Engine Fee | Percentage (0–1) | 0.05 | `glPnl.engineFee` |
| Retail Price (SRP) | Number | 59.99 | `glPnl.retailPrice` |
| Average Discount Rate | Percentage (0–1) | 0.20 | `glPnl.discountRate` |

### Developer Royalty Tiers (`glPnl.tiers[]`)

Each tier:

| Field | Type | Notes |
|---|---|---|
| Threshold Type | Enum: `start`, `multi`, `fixed`, `gnr`, `onwards` | `start` = first tier from $0; `multi` = recoupment multiplier (e.g. 1.5× = 150% of total costs); `fixed` = a hard dollar threshold; `gnr` = Gross Net Revenue threshold; `onwards` = "all remaining revenue" |
| Threshold Value | Number | Multiplier or dollar amount, depending on `thType` |
| Developer Share | Percentage (0–1) | Dev's slice of revenue in this tier |
| Publisher Share | Percentage (0–1) | Pub's slice; should sum to 1.0 with dev share |

### Licensor Royalty Tiers (`glPnl.licensorTiers[]`, when `licensorRoyaltyEnabled`)

Each tier:

| Field | Type | Notes |
|---|---|---|
| Threshold Type | Enum: `start`, `multi`, `fixed`, `gnr`, `mg` | `mg` = Minimum Guarantee — a dollar amount the licensor is owed before tier kicks in |
| Threshold Value | Number | Same logic as dev tiers |
| Licensor Share | Percentage (0–1) | Licensor's cut of after-platform revenue in this tier |

Plus:

| Field | Type | Default | Storage path |
|---|---|---|---|
| Licensor Royalty Enabled | Boolean | false | `glPnl.licensorRoyaltyEnabled` |
| Licensor Royalty Rate (legacy single-rate) | Percentage (0–1) | 0.10 | `glPnl.licensorRoyaltyRate` (used as fallback if no tiers defined) |

### Scenario Inputs

| Field | Type | Default | Storage path |
|---|---|---|---|
| Scenario unit counts | Array of integers | `[100000, 250000, 500000, 1000000, 2500000]` | `glPnl.scenarioUnits` |
| ROI targets | Array of decimals (0 = break-even, 0.30 = 30% ROI) | `[0, 0.30, 0.50, 1.0]` | `glPnl.roiTargets` |

### Page-Level Actions

- **Sync from Contracted Budget** button — overwrites the six budget fields with `masterBudget` values.
- **Export** button — opens the export overlay (HTML/PDF print).

## Calculated Fields

This is where the page earns its complexity. Every field below is recomputed on every render.

### Cost Aggregations

| Field | Formula |
|---|---|
| `pubFeeBase` | `devBudget + portingBudget + externalBudget + reserveBudget + contingencyBudget` (note: marketing and upfront engine fee are excluded from the publishing fee base) |
| `publishingFee` | `pubFeeBase × publishingFeeRate` |
| `totalCosts` (a.k.a. `recoupment`) | `pubFeeBase + marketingBudget + upfrontEngineFee + publishingFee` |
| `totalPartnerContrib` | `Σ pubContrib[k]` if partner enabled, else 0 |
| `fictionsRecoupment` | `totalCosts − totalPartnerContrib` |
| Per-line partner percentage | `pubContrib[k] / (devBudget + …) × 100` |

### Per-Unit Revenue Waterfall

| Field | Formula |
|---|---|
| `avgPrice` | `retailPrice × (1 − discountRate)` |
| `afterPlatformPerUnit` | `avgPrice × (1 − platformFee − engineFee)` |
| `licensorFirstRate` | First tier's `licensorShare` if enabled, else 0 |
| `licensorRoyaltyPerUnit` | `afterPlatformPerUnit × licensorFirstRate` (display only — actual calc uses tiered waterfall per scenario) |
| `netPerUnit` | `afterPlatformPerUnit − licensorRoyaltyPerUnit` |

### Tiered Royalty Waterfalls (`calcPnl(units)` function)

For any unit count, the function computes:

| Field | Formula |
|---|---|
| `grossRev` | `units × avgPrice` |
| `afterPlatform` | `grossRev × (1 − platformFee − engineFee)` |
| **Licensor royalty** (if enabled) | Walk `licensorTiers[]` in order. For each tier, compute its ceiling (see ceiling rules below); the tier's revenue slice is `min(afterPlatform, ceiling) − cumulativeAfterPlatform`; tier's licensor royalty = `tierSlice × tier.licensorShare`; accumulate. |
| `netRev` | `afterPlatform − licensorRoyalty` |
| **Dev royalty + Pub royalty** | Walk `tiers[]` in order against `netRev`. For each tier, compute ceiling; tier slice = `min(netRev, ceiling) − cumulativeNetRev`; `devRoyalties += tierSlice × tier.devShare`, `pubRoyalties += tierSlice × tier.pubShare`. |
| `totalExpenses` | `recoupment + devRoyalties + pubRoyalties + licensorRoyalty` |
| `profit` | `netRev − fictionsRecoupment − devRoyalties − pubRoyalties` |
| `cashOnCash` | `netRev / fictionsRecoupment` if recoupment > 0, else 0 |

**Tier ceiling rules** (lines 7480, 7492):

| `thType` | Ceiling formula |
|---|---|
| `start` | 0 (always the first tier) |
| `multi` | `recoupment × thValue` (e.g. multi 1.5 = 150% of total costs) |
| `fixed` | `thValue` (literal dollar threshold) |
| `gnr` | `thValue` (same as fixed, semantically labeled differently) |
| `mg` (licensor only) | `thValue / licensorShare` (so total licensor share equals the MG dollar amount) |
| `onwards` | Infinity (last tier catches everything remaining) |

### Break-Even & Targets

| Field | Formula |
|---|---|
| `fictionsBE` (Fictions break-even units) | `ceil(fictionsRecoupment / netPerUnit)` |
| `partnerBE` (Partner break-even units, when applicable) | Binary search (80 iterations): find units where cumulative publisher tier royalties equal `partnerContrib` |
| `findUnitsForROI(target)` | Binary search (80 iterations): find units where `profit / fictionsRecoupment = target` |
| `roiResults[]` | Map each `roiTargets[]` value through `findUnitsForROI`, then through `calcPnl` to get profit at that point |
| `scenarios[]` | Map each `scenarioUnits[]` value through `calcPnl` |

### Sync Indicator

| Field | Formula |
|---|---|
| `hasDiff` (yellow "Sync from Contracted Budget" indicator) | True if any of the six budget fields differs from its `masterBudget` analog |
| `hasData` (whether to show sync button at all) | True if `Σ masterBudget > 0` |

## Endpoints

### `GET /projects/:id/pnl`

Returns the entire P&L state plus all computed values, ready to render.

**Request**
```http
GET /projects/prj_2k9l/pnl HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "inputs": {
    "hasPubPartner": true,
    "pubPartnerName": "Stratosphere Games",
    "devBudget": 6500000,
    "portingBudget": 600000,
    "externalBudget": 1400000,
    "reserveBudget": 250000,
    "contingencyBudget": 1000000,
    "marketingBudget": 600000,
    "upfrontEngineFee": 200000,
    "publishingFeeRate": 0.10,
    "pubContrib": {
      "devBudget": 2000000,
      "portingBudget": 0,
      "externalBudget": 500000,
      "reserveBudget": 0,
      "contingencyBudget": 0,
      "marketingBudget": 600000
    },
    "platformFee": 0.30,
    "engineFee": 0.05,
    "retailPrice": 59.99,
    "discountRate": 0.20,
    "tiers": [
      { "thType": "start", "thValue": 0,    "devShare": 0.50, "pubShare": 0.50 },
      { "thType": "multi", "thValue": 1.0,  "devShare": 0.60, "pubShare": 0.40 },
      { "thType": "multi", "thValue": 2.0,  "devShare": 0.70, "pubShare": 0.30 },
      { "thType": "onwards", "thValue": 0,  "devShare": 0.80, "pubShare": 0.20 }
    ],
    "licensorRoyaltyEnabled": true,
    "licensorRoyaltyRate": 0.10,
    "licensorTiers": [
      { "thType": "start",   "thValue": 0,        "licensorShare": 0.10 },
      { "thType": "mg",      "thValue": 500000,   "licensorShare": 0.10 },
      { "thType": "onwards", "thValue": 0,        "licensorShare": 0.15 }
    ],
    "scenarioUnits": [100000, 250000, 500000, 1000000, 2500000],
    "roiTargets": [0, 0.30, 0.50, 1.0]
  },
  "computed": {
    "costs": {
      "pubFeeBase": 9750000,
      "publishingFee": 975000,
      "totalCosts": 11525000,
      "totalPartnerContrib": 3100000,
      "fictionsRecoupment": 8425000,
      "fictionsRecoupmentPct": 26.90,
      "partnerRecoupmentPct": 73.10
    },
    "perUnit": {
      "avgPrice": 47.99,
      "platformFeeAmount": 14.40,
      "engineFeeAmount": 2.40,
      "afterPlatformPerUnit": 31.19,
      "licensorRoyaltyPerUnit": 3.12,
      "netPerUnit": 28.07
    },
    "breakEven": {
      "fictionsBE": 300214,
      "partnerBE": 850000
    },
    "scenarios": [
      {
        "units": 100000,
        "grossRev": 4799000,
        "afterPlatform": 3119350,
        "licensorRoyalty": 311935,
        "netRev": 2807415,
        "devRoyalties": 1403707,
        "pubRoyalties": 1403707,
        "totalExpenses": 13243349,
        "profit": -8425000,
        "cashOnCash": 0.33
      },
      {
        "units": 1000000,
        "grossRev": 47990000,
        "afterPlatform": 31193500,
        "licensorRoyalty": 4078025,
        "netRev": 27115475,
        "devRoyalties": 14860284,
        "pubRoyalties": 6258491,
        "totalExpenses": 35721800,
        "profit": -2428300,
        "cashOnCash": 3.22
      }
    ],
    "roiResults": [
      { "roi": 0,    "units": 300214,  "profit": 0 },
      { "roi": 0.30, "units": 425000,  "profit": 2527500 },
      { "roi": 0.50, "units": 510000,  "profit": 4212500 },
      { "roi": 1.0,  "units": 720000,  "profit": 8425000 }
    ]
  },
  "sync": {
    "hasContractedData": true,
    "diff": {
      "developmentCosts": { "contracted": 6000000, "pnl": 6500000, "diff": 500000 },
      "marketing":        { "contracted": 500000,  "pnl": 600000,  "diff": 100000 }
    },
    "needsSync": true
  },
  "greenlitVersion": 3,
  "updatedAt": "2026-04-22T11:14:00Z"
}
```

### `PATCH /projects/:id/pnl`

Partial update of any input field. The server recomputes all derived values and returns them.

**Request — update pricing**
```http
PATCH /projects/prj_2k9l/pnl HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "retailPrice": 69.99,
  "discountRate": 0.25
}
```

**Request — update tiers (full replacement of the array)**
```http
PATCH /projects/prj_2k9l/pnl HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "tiers": [
    { "thType": "start", "thValue": 0,   "devShare": 0.50, "pubShare": 0.50 },
    { "thType": "multi", "thValue": 1.5, "devShare": 0.65, "pubShare": 0.35 },
    { "thType": "onwards", "thValue": 0, "devShare": 0.75, "pubShare": 0.25 }
  ]
}
```

**Request — update partner contribution**
```http
PATCH /projects/prj_2k9l/pnl HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "pubContrib": {
    "devBudget": 2500000,
    "marketingBudget": 700000
  }
}
```

**Response — 200 OK**

Returns the same shape as `GET /pnl` with refreshed `computed` block.

```json
{
  "projectId": "prj_2k9l",
  "inputs": {
    "retailPrice": 69.99,
    "discountRate": 0.25
  },
  "computed": {
    "perUnit": {
      "avgPrice": 52.49,
      "afterPlatformPerUnit": 34.12,
      "netPerUnit": 30.71
    },
    "breakEven": {
      "fictionsBE": 274369,
      "partnerBE": 780000
    }
  },
  "updatedAt": "2026-05-01T16:00:00Z",
  "auditEntryId": "aud_p2k7"
}
```

**Notes**
- Tier arrays are full-replacement. Sending `tiers: [...]` replaces the entire tier list. This avoids ambiguity about reordering.
- `pubContrib` is partial-merge — sending one key updates only that key.
- The full computed block is always returned because tier-share validation requires server-side recalc.

**Validation rules the server enforces:**
- `platformFee + engineFee ≤ 1.0` (otherwise after-platform per-unit goes negative)
- `discountRate < 1.0`
- For each tier: `devShare + pubShare = 1.0` (within ±0.01 tolerance for floating-point)
- For licensor tiers: `licensorShare ≤ 1.0` (no other share to balance against)
- First tier in both `tiers[]` and `licensorTiers[]` must be `thType: "start"` with `thValue: 0`
- Last tier must be `thType: "onwards"` (otherwise revenue beyond the highest threshold is uncomputed)

Reject with `422 Unprocessable Entity` and a list of validation errors:
```json
{
  "error": "VALIDATION_FAILED",
  "details": [
    { "field": "tiers[1].devShare + tiers[1].pubShare", "message": "Must sum to 1.0; got 0.95" },
    { "field": "tiers", "message": "Last tier must have thType 'onwards'" }
  ]
}
```

### `POST /projects/:id/pnl/sync-from-contracted`

One-click action: copy `masterBudget` values into the corresponding P&L cost fields.

**Request**
```http
POST /projects/prj_2k9l/pnl/sync-from-contracted HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "synced": {
    "developmentCosts": { "from": 6000000, "to": 6500000 },
    "marketing":        { "from": 600000,  "to": 500000 }
  },
  "computed": {
    "costs": {
      "totalCosts": 11525000,
      "fictionsRecoupment": 8425000
    }
  },
  "updatedAt": "2026-05-01T16:15:00Z",
  "auditEntryId": "aud_q4n8"
}
```

### `POST /projects/:id/pnl/calculate`

A read-only "what-if" endpoint — runs `calcPnl(units)` for arbitrary unit counts without persisting anything. Useful for the One Sheet's inline sliders or any dynamic calculator.

**Request**
```http
POST /projects/prj_2k9l/pnl/calculate HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "units": [50000, 175000, 400000],
  "roiTargets": [0.40]
}
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "scenarios": [
    {
      "units": 50000,
      "grossRev": 2399500,
      "netRev": 1403707,
      "devRoyalties": 701853,
      "pubRoyalties": 701853,
      "profit": -8425000,
      "cashOnCash": 0.17
    },
    {
      "units": 175000,
      "grossRev": 8398250,
      "netRev": 4912974,
      "devRoyalties": 2456487,
      "pubRoyalties": 2456487,
      "profit": -8425000,
      "cashOnCash": 0.58
    }
  ],
  "roiResults": [
    { "roi": 0.40, "units": 463000, "profit": 3370000 }
  ]
}
```

**Notes**
- This endpoint is stateless and does not write or audit. Safe to call repeatedly.
- Useful for dashboards that want to show "if we sell X units, we make Y" without modifying the saved P&L.

### Permissions Summary (P&L)

| Endpoint | Required permission | Additional gates |
|---|---|---|
| `GET /projects/:id/pnl` | studio member, project access | — |
| `PATCH /projects/:id/pnl` | `editSettings` | If greenlit is locked, must be in active amendment draft |
| `POST /projects/:id/pnl/sync-from-contracted` | `editSettings` | — |
| `POST /projects/:id/pnl/calculate` | studio member, project access | — |

---

# Page 5: Amendments Workflow

This page is less about input fields and more about **state transitions**. The amendment is a multi-stage record that moves through `draft → submitted → approved | rejected | deleted`. Each transition has gates and side effects.

## Amendment Lifecycle (state machine)

```
        ┌──────────────────────────────────────────────┐
        │                                              │
   [START]                                             │
        │                                              ▼
        ▼                                       ┌───────────┐
  ┌─────────┐  Submit ┌───────────┐ Approve ┌──►│  APPROVED │ ← bumps GL version
  │  DRAFT  │────────►│ SUBMITTED │─────────┘   └───────────┘
  └─────────┘         └───────────┘
        │                   │
   Discard            Reject │
        │                   │
        ▼                   ▼
   [GONE]             ┌───────────┐
                      │  REJECTED │
                      └───────────┘
                            │
                       Delete
                            ▼
                       [GONE]
```

Plus a separate, parallel queue: **Budget Change Reviews** — these are not amendments but smaller cell-level edits that came in from Reconciliation auto-adjustments or from the Developer Portal. They appear on the same page but follow a different (simpler) workflow.

## Editable Fields

### Amendment Draft (when `amendmentDraft !== null`)

| Field | Type | Default | Storage path |
|---|---|---|---|
| Reason | Multiline text | "" | `amendmentDraft.reason` (required) |
| Term Changes notes | Multiline text | "" | `amendmentDraft.termChanges.notes` |
| Checklist: Update Contracted Budget | Boolean | false | `amendmentDraft.checklist.contractedBudget` |
| Checklist: Export new P&L | Boolean | false | `amendmentDraft.checklist.exportPnl` |
| Checklist: Export new One Sheet | Boolean | false | `amendmentDraft.checklist.exportOneSheet` |

The proposed budget changes themselves are not edited on this page — they live in the **greenlit budget grid** during the draft (when `greenlitEditing === true`). The Amendment page captures the rationale and the procedural gate; the actual numbers come from the grid edits.

### Submit / Reject / Approve

These are actions, not fields, but each accepts an optional note:

| Action | Body field | Permission |
|---|---|---|
| Submit draft | `reason` (already in draft) | `editSettings`, draft owner |
| Approve | optional `approvalNote` | `approveAmendments` |
| Reject | optional `rejectionReason` | `approveAmendments` |
| Delete | optional `deletionReason` | `editSettings` (owner) or admin |

### Budget Change Review Actions

| Action | Body field | Permission |
|---|---|---|
| Approve a batch | optional `note` | `approveBudgetChanges` |
| Reject a batch | required `reason` | `approveBudgetChanges` |

## Calculated Fields

### Status Bar (top of page)

| Field | Formula |
|---|---|
| `liveTot` | `Σ grandTotal("live", m.key)` over `msLive` |
| `glTot` | `Σ grandTotal("greenlit", m.key)` over `msGreenlit` |
| `variance` | `liveTot − glTot` |
| `varPct` | `(variance / glTot) × 100` |
| Pending Finance Approval count | `amendments.filter(status === "submitted").length` |
| Pending Budget Changes count | `changeLog.filter(status === "pending").length` |

### Amendment History Table

| Field | Source |
|---|---|
| Columns | "Initial Deal" + one column per approved amendment + "Current Live" |
| Initial Deal column | First approved amendment's `groupDeltas[].greenlit` (the values *before* that amendment) |
| Amendment N column | That amendment's `groupDeltas[].proposed` |
| Current Live column | Live values from grid: `Σ groupTotal(g.id, "live", m.key)` over `msLive` |
| Per-row Total Change | `lastColumn − firstColumn` per group |
| Cell highlight | Red if value increased from previous column, green if decreased |

### Per-Amendment Card (Pending and Approved lists)

| Field | Formula |
|---|---|
| `currentGLTotal` | Snapshot at submission time of `glTot` |
| `proposedTotal` | Sum of `groupDeltas[].proposed` |
| `delta` | `proposedTotal − currentGLTotal` |
| Group deltas (per group) | `{name, color, greenlit (before), proposed (after), delta}` |
| `pnlCompleted` | All three checklist items true |

### Submit Button Gate

| Condition | Required |
|---|---|
| `amendmentDraft.checklist.contractedBudget` | true |
| `amendmentDraft.checklist.exportPnl` | true |
| `amendmentDraft.checklist.exportOneSheet` | true |
| `amendmentDraft.reason.trim()` | non-empty |

If any condition fails, the button is disabled.

### "Amendment Needed" Indicator

| Condition | Where shown |
|---|---|
| `\|varPct\| > 10 && glLocked && !isDev && perms.editSettings` | Yellow chip in the project status bar (top of every page); clicking jumps to Amendments |

## Endpoints

### `GET /projects/:id/amendments`

Returns the full amendment history plus pending budget change batches.

**Request**
```http
GET /projects/prj_2k9l/amendments HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "currentGreenlit": {
    "version": 3,
    "lockedAt": "2026-02-14T15:00:00Z",
    "totalGreenlit": 9750000,
    "totalLive": 10215000,
    "variance": 465000,
    "variancePct": 4.77,
    "amendmentNeeded": false
  },
  "draft": null,
  "amendments": [
    {
      "id": "amd_a1b2",
      "status": "approved",
      "currentGLVersion": 1,
      "newGLVersion": 2,
      "requestedBy": "Alex Chen",
      "requestedAt": "2025-12-10T14:00:00Z",
      "submittedAt": "2025-12-10T14:30:00Z",
      "approvedBy": "Sam Park",
      "approvedAt": "2025-12-12T09:15:00Z",
      "reason": "Additional QA cycle required after publisher feedback on Alpha milestone.",
      "termChanges": { "notes": "Extended QA window by 2 months." },
      "currentGLTotal": 9000000,
      "proposedTotal": 9450000,
      "delta": 450000,
      "groupDeltas": [
        { "id": "devcosts", "name": "Development Costs", "color": "#6366f1", "greenlit": 6000000, "proposed": 6200000, "delta": 200000 },
        { "id": "porting",  "name": "Porting Costs",     "color": "#f59e0b", "greenlit": 600000,  "proposed": 600000,  "delta": 0 },
        { "id": "external", "name": "External Support",  "color": "#10b981", "greenlit": 1400000, "proposed": 1650000, "delta": 250000 }
      ],
      "checklist": {
        "contractedBudget": true,
        "exportPnl": true,
        "exportOneSheet": true
      },
      "pnlCompleted": true,
      "snapshotId": "snap_p3k1"
    },
    {
      "id": "amd_c5d6",
      "status": "approved",
      "currentGLVersion": 2,
      "newGLVersion": 3,
      "requestedBy": "Alex Chen",
      "requestedAt": "2026-02-10T11:00:00Z",
      "submittedAt": "2026-02-12T16:20:00Z",
      "approvedBy": "Sam Park",
      "approvedAt": "2026-02-14T15:00:00Z",
      "reason": "Marketing budget increase for E3 presence.",
      "termChanges": { "notes": "" },
      "currentGLTotal": 9450000,
      "proposedTotal": 9750000,
      "delta": 300000,
      "groupDeltas": [
        { "id": "devcosts", "name": "Development Costs", "color": "#6366f1", "greenlit": 6200000, "proposed": 6200000, "delta": 0 },
        { "id": "porting",  "name": "Porting Costs",     "color": "#f59e0b", "greenlit": 600000,  "proposed": 600000,  "delta": 0 },
        { "id": "external", "name": "External Support",  "color": "#10b981", "greenlit": 1650000, "proposed": 1650000, "delta": 0 }
      ],
      "checklist": {
        "contractedBudget": true,
        "exportPnl": true,
        "exportOneSheet": true
      },
      "pnlCompleted": true,
      "snapshotId": "snap_q7m2"
    }
  ],
  "history": {
    "columns": [
      { "label": "Initial Deal", "color": "#64748b", "values": { "Development Costs": 6000000, "Porting Costs": 600000, "External Support": 1400000 }, "total": 8000000 },
      { "label": "Amendment 1", "color": "#6366f1", "date": "Dec 10, 2025", "values": { "Development Costs": 6200000, "Porting Costs": 600000, "External Support": 1650000 }, "total": 8450000 },
      { "label": "Amendment 2", "color": "#6366f1", "date": "Feb 12, 2026", "values": { "Development Costs": 6200000, "Porting Costs": 600000, "External Support": 1650000 }, "total": 8450000 },
      { "label": "Current Live", "color": "#10b981", "values": { "Development Costs": 6240000, "Porting Costs": 580000, "External Support": 1395000 }, "total": 8215000 }
    ]
  },
  "budgetChangeQueue": [
    {
      "id": "bch_x7p2",
      "submittedBy": "Jamie Rivera",
      "submittedAt": "2026-04-28T13:45:00Z",
      "source": "dev-portal",
      "status": "pending",
      "note": "Need to bring on a contract animator for 3 months to hit Alpha.",
      "changes": [
        { "rowId": "row_n3kp", "rowName": "Animation Contractor", "monthKey": "2026-05", "oldValue": 0, "newValue": 12000 },
        { "rowId": "row_n3kp", "rowName": "Animation Contractor", "monthKey": "2026-06", "oldValue": 0, "newValue": 12000 },
        { "rowId": "row_n3kp", "rowName": "Animation Contractor", "monthKey": "2026-07", "oldValue": 0, "newValue": 12000 }
      ]
    },
    {
      "id": "bch_y9q3",
      "submittedBy": "Reconciliation",
      "submittedAt": "2026-04-29T10:00:00Z",
      "source": "reconcile",
      "status": "pending",
      "note": null,
      "changes": [
        { "rowId": "row_q4mr", "rowName": "UI Contractor", "monthKey": "2026-04", "oldValue": 8000, "newValue": 9500 }
      ]
    }
  ]
}
```

### `POST /projects/:id/amendments/draft`

Open a new amendment draft. There can be only one draft at a time per project — opening a second one fails with `409 Conflict`.

**Request**
```http
POST /projects/prj_2k9l/amendments/draft HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "reason": "Initial draft note (optional, can be edited later)"
}
```

**Response — 201 Created**
```json
{
  "draft": {
    "id": "amd_draft_w4k2",
    "status": "draft",
    "currentGLVersion": 3,
    "openedBy": "Alex Chen",
    "openedAt": "2026-05-01T17:00:00Z",
    "reason": "Initial draft note (optional, can be edited later)",
    "termChanges": { "notes": "" },
    "checklist": {
      "contractedBudget": false,
      "exportPnl": false,
      "exportOneSheet": false
    },
    "currentGLTotal": 9750000,
    "proposedTotal": 9750000,
    "delta": 0,
    "groupDeltas": [
      { "id": "devcosts", "name": "Development Costs", "greenlit": 6200000, "proposed": 6200000, "delta": 0 },
      { "id": "porting",  "name": "Porting Costs",     "greenlit": 600000,  "proposed": 600000,  "delta": 0 },
      { "id": "external", "name": "External Support",  "greenlit": 1650000, "proposed": 1650000, "delta": 0 }
    ]
  },
  "greenlitEditing": true,
  "auditEntryId": "aud_r8s2"
}
```

**Side effects**
- Sets `project.greenlitEditing = true` — unlocks the greenlit grid for editing.
- Sets `project.greenlitLocked = false` for the duration of the draft.
- Auto-creates a snapshot before opening (rollback point).

**Error responses**
- `409 Conflict` — a draft already exists. Body returns the existing draft ID and prompts the caller to discard it first.

### `PATCH /projects/:id/amendments/draft`

Update draft fields (reason, term changes, checklist).

**Request**
```http
PATCH /projects/prj_2k9l/amendments/draft HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "reason": "External Support contractor rates increased due to delayed scope. Adding 3 months of QA buffer.",
  "termChanges": {
    "notes": "QA window extended; no royalty term changes."
  },
  "checklist": {
    "contractedBudget": true,
    "exportPnl": true
  }
}
```

**Response — 200 OK**
```json
{
  "draft": {
    "id": "amd_draft_w4k2",
    "reason": "External Support contractor rates increased due to delayed scope. Adding 3 months of QA buffer.",
    "termChanges": { "notes": "QA window extended; no royalty term changes." },
    "checklist": {
      "contractedBudget": true,
      "exportPnl": true,
      "exportOneSheet": false
    },
    "submitReady": false,
    "submitBlockers": ["checklist.exportOneSheet must be true"]
  },
  "proposedTotal": 10100000,
  "delta": 350000,
  "auditEntryId": "aud_t1u4"
}
```

**Notes**
- The `proposedTotal` and `delta` reflect the **current state of the greenlit grid** — they update automatically as the user edits cells.
- `submitReady` is the server's view of whether Submit would succeed. `submitBlockers` lists what's missing.

### `DELETE /projects/:id/amendments/draft`

Discard the draft.

**Request**
```http
DELETE /projects/prj_2k9l/amendments/draft HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "discardedDraftId": "amd_draft_w4k2",
  "greenlitRestored": true,
  "snapshotRestored": "snap_v5w6",
  "auditEntryId": "aud_u3v7"
}
```

**Side effects**
- Restores the greenlit grid from the pre-draft snapshot — any edits made during the draft are reverted.
- Sets `project.greenlitEditing = false`, `project.greenlitLocked = true`.

### `POST /projects/:id/amendments/draft/submit`

Submit the draft to finance for approval.

**Request**
```http
POST /projects/prj_2k9l/amendments/draft/submit HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "amendment": {
    "id": "amd_e7f8",
    "status": "submitted",
    "currentGLVersion": 3,
    "requestedBy": "Alex Chen",
    "requestedAt": "2026-05-01T17:00:00Z",
    "submittedAt": "2026-05-01T17:30:00Z",
    "reason": "External Support contractor rates increased…",
    "currentGLTotal": 9750000,
    "proposedTotal": 10100000,
    "delta": 350000,
    "groupDeltas": [
      { "id": "devcosts", "greenlit": 6200000, "proposed": 6200000, "delta": 0 },
      { "id": "porting",  "greenlit": 600000,  "proposed": 600000,  "delta": 0 },
      { "id": "external", "greenlit": 1650000, "proposed": 2000000, "delta": 350000 }
    ],
    "snapshotId": "snap_w8x9"
  },
  "auditEntryId": "aud_v5w8"
}
```

**Side effects**
- Captures `groupDeltas` from current greenlit grid state.
- Sets `project.greenlitEditing = false` (grid re-locks until approval).
- Sets `project.greenlitLocked = true`.
- Notifies users with `approveAmendments` permission.

**Error responses**
- `422` — checklist incomplete or reason empty
- `409` — no active draft

### `POST /projects/:id/amendments/:amendmentId/approve`

Approve a submitted amendment. Bumps GL version and applies the proposed changes as the new greenlit baseline.

**Request**
```http
POST /projects/prj_2k9l/amendments/amd_e7f8/approve HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "approvalNote": "Approved with caveats — please flag if external costs exceed 2.1M."
}
```

**Response — 200 OK**
```json
{
  "amendment": {
    "id": "amd_e7f8",
    "status": "approved",
    "currentGLVersion": 3,
    "newGLVersion": 4,
    "approvedBy": "Sam Park",
    "approvedAt": "2026-05-02T09:00:00Z",
    "approvalNote": "Approved with caveats — please flag if external costs exceed 2.1M.",
    "snapshotId": "snap_w8x9"
  },
  "project": {
    "greenlitVersion": 4,
    "greenlitDate": "2026-05-02",
    "greenlitLocked": true,
    "totalGreenlit": 10100000
  },
  "auditEntryId": "aud_x1y2"
}
```

**Side effects**
- `greenlitVersion` bumped (3 → 4).
- `greenlitDate` updated to today.
- New row in `greenlitHistory[]` capturing the previous version's `greenlit{}` snapshot.
- All notification consumers fired (e.g., email to producers, dashboard chip refreshed).

### `POST /projects/:id/amendments/:amendmentId/reject`

Reject a submitted amendment.

**Request**
```http
POST /projects/prj_2k9l/amendments/amd_e7f8/reject HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "rejectionReason": "External cost increase needs publisher sign-off first. Please attach the publisher email and resubmit."
}
```

**Response — 200 OK**
```json
{
  "amendment": {
    "id": "amd_e7f8",
    "status": "rejected",
    "rejectedBy": "Sam Park",
    "rejectedAt": "2026-05-02T09:30:00Z",
    "rejectionReason": "External cost increase needs publisher sign-off first."
  },
  "auditEntryId": "aud_y3z4"
}
```

**Side effects**
- `greenlitVersion` not bumped.
- The greenlit grid is not modified — the proposed values were captured in the amendment record but never applied.
- Producer is notified to revise and resubmit.

### `DELETE /projects/:id/amendments/:amendmentId`

Delete an amendment. For approved amendments, this **reverts** the greenlight version (rare, but supported per the prototype's `deleteAmendment` flow).

**Request**
```http
DELETE /projects/prj_2k9l/amendments/amd_e7f8 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "deletionReason": "Approved in error — wrong project."
}
```

**Response — 200 OK**
```json
{
  "deletedAmendmentId": "amd_e7f8",
  "deletedAt": "2026-05-02T10:00:00Z",
  "deletedBy": "Alex Chen",
  "previousStatus": "approved",
  "greenlitVersionReverted": true,
  "currentGreenlitVersion": 3,
  "auditEntryId": "aud_z5a6"
}
```

**Notes**
- For non-approved amendments, this is a simple soft-delete.
- For approved amendments, the server walks the `greenlitHistory[]` and restores the prior version's snapshot. This is destructive enough that it should require admin or `manageUsers` permission.

### `POST /projects/:id/budget-changes/:batchId/approve`

Approve a pending budget-change batch (from Reconciliation or Dev Portal).

**Request**
```http
POST /projects/prj_2k9l/budget-changes/bch_x7p2/approve HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "note": "Approved — animator hired."
}
```

**Response — 200 OK**
```json
{
  "batchId": "bch_x7p2",
  "status": "approved",
  "appliedChanges": 3,
  "reviewedBy": "Alex Chen",
  "reviewedAt": "2026-05-02T11:00:00Z",
  "note": "Approved — animator hired.",
  "totals": {
    "live": { "grandTotal": 10251000, "variance": 501000 }
  },
  "auditEntryId": "aud_b7c8"
}
```

**Side effects**
- For each `change` in the batch, applies `newValue` to `row.live[monthKey]`.
- Marks the batch as `approved`.
- Updates the project's `liveAll` and variance.

### `POST /projects/:id/budget-changes/:batchId/reject`

Reject a pending budget-change batch.

**Request**
```http
POST /projects/prj_2k9l/budget-changes/bch_x7p2/reject HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "reason": "Need formal scope change request first — please open an amendment."
}
```

**Response — 200 OK**
```json
{
  "batchId": "bch_x7p2",
  "status": "rejected",
  "reviewedBy": "Alex Chen",
  "reviewedAt": "2026-05-02T11:15:00Z",
  "reason": "Need formal scope change request first — please open an amendment.",
  "auditEntryId": "aud_d9e0"
}
```

**Side effects**
- No grid changes are applied.
- The submitter (developer or reconciliation system) is notified.
- For dev-portal batches, the rejection reason flows back to the developer's "My Submissions" view.

### Permissions Summary (Amendments)

| Endpoint | Required permission | Additional gates |
|---|---|---|
| `GET /projects/:id/amendments` | studio member, project access | — |
| `POST /projects/:id/amendments/draft` | `editSettings` | No existing draft; greenlit must be locked |
| `PATCH /projects/:id/amendments/draft` | `editSettings`, draft owner | Draft must exist |
| `DELETE /projects/:id/amendments/draft` | `editSettings`, draft owner | Draft must exist |
| `POST .../draft/submit` | `editSettings`, draft owner | Checklist complete, reason non-empty |
| `POST .../:amendmentId/approve` | `approveAmendments` | Amendment must be `submitted` |
| `POST .../:amendmentId/reject` | `approveAmendments` | Amendment must be `submitted` |
| `DELETE .../:amendmentId` | `editSettings` (owner) for non-approved; `manageUsers` for approved | — |
| `POST /budget-changes/:id/approve` | `approveBudgetChanges` | Batch must be `pending` |
| `POST /budget-changes/:id/reject` | `approveBudgetChanges` | Batch must be `pending` |

---

## Cross-Cutting: Notification Hooks

Both pages should fire notifications on key transitions. The brief calls for "Email notifications for submissions, approvals, rejections" (Phase 3 deliverable). Suggested events:

| Event | Recipients |
|---|---|
| `amendment.submitted` | Users with `approveAmendments` for this project |
| `amendment.approved` | Amendment requester + all project members |
| `amendment.rejected` | Amendment requester |
| `budget_change.submitted` | Users with `approveBudgetChanges` for this project |
| `budget_change.approved` | Submitter |
| `budget_change.rejected` | Submitter |
| `pnl.synced_from_contracted` | (no notification — purely a producer convenience action) |

Each event should be queued for delivery (don't block the API response on email send). Webhook-style outbound delivery is also worth considering for studios that integrate with their own Slack or Teams.

---

## Data Integrity Notes for the Backend

A handful of subtle invariants the backend must enforce; the prototype trusts the frontend on these and that won't be acceptable in production:

1. **Only one draft amendment per project at a time.** Use a partial unique index: `UNIQUE WHERE status = 'draft'`.
2. **Only one greenlit version is current at a time.** When approving an amendment, atomically: bump version, snapshot prior, write new greenlit values. All in one transaction — never leave the project with an inconsistent greenlit state.
3. **Group deltas captured at submit time, not approve time.** If the producer keeps editing the greenlit grid after submitting, those edits should not silently change what finance is approving. Snapshot at submit.
4. **Tier validation on every P&L PATCH.** Floating-point share sums require a tolerance window. Use ±0.01 or store shares as integer basis points (5000 = 50.00%) to avoid the issue entirely.
5. **Audit every state transition.** Not just the final action but the user, timestamp, role, before/after values, and any note provided. The audit log is the legal record of greenlight decisions.
6. **Approval auto-syncs P&L.** When an amendment is approved, the `glPnl` cost fields should be updated to reflect the new contracted budget — otherwise the P&L on the next greenlight will start from stale numbers. Consider including this as a post-approval hook.

---

This is the operational financial workflow trio — the highest-frequency editing surface in the entire app. A producer might process dozens of invoices in a single session.

---

# Page 6: Game Ledger / Marketing Ledger

These are the same UI/code path with one boolean flag (`isMarketingLedger`) that filters which entries are shown. I'll cover both as one page; differences are called out where they exist.

## Editable Fields

Each ledger entry (`project.ledger[]`) has the following editable fields:

| Field | Type | Storage path | Notes |
|---|---|---|---|
| Invoice Number | Text | `entry.invoice` | Free-text, e.g. "INV-2026-0142" |
| Payee / Vendor | Text | `entry.payee` (with fallback to `entry.vendor`) | Required for any meaningful entry |
| Original Amount | Number | `entry.originalAmount` | The amount as it appears on the invoice (pre-conversion) |
| Original Currency | Dropdown | `entry.originalCurrency` | One of the supported currency codes (USD, EUR, GBP, JPY, CAD, AUD, CHF, CNY, KRW, BRL, MXN, SEK, NOK, DKK, PLN, CZK, INR, SGD, HKD, NZD, ZAR) |
| Total Billed (USD) | Number | `entry.totalBilled` | If currency is USD, equals `originalAmount`. If non-USD, manually entered or computed via FX. |
| Status | Dropdown | `entry.status` | Enum: `Draft`, `Sent to Finance`, `Paid`, `Disputed` |
| Date | Date | `entry.date` | Generic date field (entry creation / record date) |
| Invoice Date | Date | `entry.invoiceDate` | The date on the invoice itself; drives reconciliation month bucketing |
| Category | Picker | `entry.category` | Hierarchical: top-level group → optional sub-category. Comes from `LEDGER_CATEGORIES` (line 320). |
| Description / Memo | Text | `entry.description` or `entry.memo` | Free-text |
| Attachment | File | `entry.attachment = {name, data, type, size}` | Optional PDF/image; stored as base64 in prototype |

### Page-Level Actions

| Action | Effect |
|---|---|
| Add Row | Creates an empty entry; pre-fills `category` from current sub-tab if any |
| Import QB Export | CSV/XLSX upload; runs through fingerprinting (duplicate detection); imported entries enter the `reconItems` queue rather than directly into the ledger |
| Scan Invoice | File upload (PDF/image); calls Anthropic API; parses returned JSON into a draft entry |
| Send to Finance | Per-row action; downloads the attachment, copies an email to clipboard, sets status to "Sent to Finance" |
| Delete Row | Removes the entry |
| Right-click context menu | Cut/copy/paste/duplicate |
| Sub-tab navigation | Filters by category group; affects which entries are visible and what category gets pre-filled on Add Row |
| Export | Opens export overlay (CSV / HTML for printing) |

### Allocations (the link to the budget grid)

Allocations are technically part of a ledger entry but are **not edited on this page** — they're managed on the Reconciliation page. Each entry has:

```json
"allocations": [
  { "id": "alloc_a1b2", "rowId": "row_n3kp", "monthKey": "2026-04", "amount": 12000 },
  { "id": "alloc_c3d4", "rowId": "row_q4mr", "monthKey": "2026-04", "amount": 8000 }
]
```

The reconciliation icon on each ledger row (✓ / ◯ / blank) is derived from this array.

## Calculated Fields

| Field | Formula |
|---|---|
| **Reconciliation status icon** | `allocated >= total - 0.01` → ✓ (green); `0 < allocated < total` → ◯ (yellow); `allocated == 0` → blank |
| **`allocated` per entry** | `Σ entry.allocations[].amount` |
| **`isUSD`** | `(entry.originalCurrency \|\| "USD") === "USD"` — drives whether the Total Billed cell is editable or auto-computed |
| **Total Billed (when USD)** | Mirrors `originalAmount` (read-only display) |
| **Filtered entries** | `allSorted.filter(...)` based on (a) ledger type — `mktg_ledger` keeps only `CAT_TO_GROUP[category] === "Marketing Costs"`; `ledger` keeps everything else; (b) sub-tab if active |
| **`totalBilledSum`** (header KPI) | `Σ filteredEntries.totalBilled` (or `originalAmount` fallback) |
| **Group subtotals** (in expandable group headers) | `Σ groupEntries.totalBilled` |
| **Sort order** | `[...ledger].sort()` by `ledgerSort.col` (ascending or descending), with type-aware comparators |
| **Pending reconciliation count** | `unallocated.length` — drives the "N QB entries pending reconciliation" banner |
| **Per-entry "remaining" amount** (if partially allocated) | `totalBilled − allocated` |

### Categories Structure

`LEDGER_CATEGORIES` (line 320) is the master taxonomy. Each top-level group has sub-categories; some sub-categories have nested items. The mapping is:

```
Development Costs → Salaries → Engineering, Design, Art, Audio, Production, QA
                  → Contractors → (same departments)
                  → Operating → Software, Infrastructure, Office, ...
External Support  → QA, Localization, Ratings, Legal, ...
Porting Costs     → Console Porting, Platform Certification, ...
Marketing Costs   → Creative Materials, Promotional Materials, Media Spend,
                    Marketing T&E, PR Expenses, Events & Showcases, Other Marketing
Reserve           → (no sub-categories)
Contingency       → (no sub-categories)
```

Two lookup maps are derived from this for filtering/grouping:
- `CAT_TO_GROUP[categoryKey]` → top-level group name
- `CAT_TO_SUBGROUP[categoryKey]` → second-level sub-category name

## Endpoints

### `GET /projects/:id/ledger`

Returns ledger entries with filtering, sorting, and pagination support.

**Request**
```http
GET /projects/prj_2k9l/ledger?type=game&category=Engineering&sort=invoiceDate&order=desc&limit=50 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Query parameters**
- `type` — `game` (default; everything except marketing) or `marketing` (only marketing entries)
- `category` — filter by exact category key
- `subgroup` — filter by sub-category name
- `status` — filter by status (`Draft`, `Sent to Finance`, `Paid`, `Disputed`)
- `q` — free-text search across `payee`, `description`, `invoice` (the brief calls for this; the prototype doesn't have a UI for it but the API should expose it)
- `sort` — column name (`invoiceDate`, `payee`, `totalBilled`, `originalAmount`, `status`)
- `order` — `asc` or `desc`
- `limit` — page size (default 100, max 500)
- `cursor` — pagination cursor

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "type": "game",
  "filters": {
    "category": "Engineering",
    "status": null,
    "q": null
  },
  "entries": [
    {
      "id": "led_a1b2",
      "invoice": "INV-2026-0142",
      "payee": "Acme Studio Ltd",
      "originalAmount": 12000,
      "originalCurrency": "GBP",
      "totalBilled": 15240,
      "status": "Paid",
      "date": "2026-04-15",
      "invoiceDate": "2026-04-01",
      "category": "Engineering",
      "description": "Lead programmer — April retainer",
      "attachment": {
        "name": "Acme_April_Invoice.pdf",
        "type": "application/pdf",
        "size": 248320,
        "downloadUrl": "/projects/prj_2k9l/ledger/led_a1b2/attachment"
      },
      "allocations": [
        { "id": "alloc_p1q2", "rowId": "row_n3kp", "monthKey": "2026-04", "amount": 15240 }
      ],
      "computed": {
        "allocated": 15240,
        "remaining": 0,
        "reconciliationStatus": "fully_allocated",
        "groupName": "Development Costs",
        "subgroupName": "Salaries"
      },
      "createdAt": "2026-04-16T09:00:00Z",
      "createdBy": "Alex Chen",
      "updatedAt": "2026-04-22T14:30:00Z"
    },
    {
      "id": "led_c3d4",
      "invoice": "FREELANCE-204",
      "payee": "Pat Singh",
      "originalAmount": 8000,
      "originalCurrency": "USD",
      "totalBilled": 8000,
      "status": "Sent to Finance",
      "date": "2026-04-20",
      "invoiceDate": "2026-04-15",
      "category": "Engineering",
      "description": "Network architecture consulting (8 hrs)",
      "attachment": null,
      "allocations": [],
      "computed": {
        "allocated": 0,
        "remaining": 8000,
        "reconciliationStatus": "unallocated",
        "groupName": "Development Costs",
        "subgroupName": "Contractors"
      },
      "createdAt": "2026-04-20T11:00:00Z",
      "createdBy": "Alex Chen",
      "updatedAt": "2026-04-20T11:00:00Z"
    }
  ],
  "pagination": {
    "total": 142,
    "returned": 50,
    "nextCursor": "eyJvZmYiOjUwfQ=="
  },
  "summary": {
    "totalBilledSum": 285420,
    "byStatus": {
      "Draft": 4,
      "Sent to Finance": 12,
      "Paid": 124,
      "Disputed": 2
    },
    "byReconciliation": {
      "fully_allocated": 118,
      "partial": 8,
      "unallocated": 16
    }
  }
}
```

### `POST /projects/:id/ledger`

Create a single ledger entry (called by the "Add Row" button).

**Request**
```http
POST /projects/prj_2k9l/ledger HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "category": "Engineering",
  "payee": "",
  "originalAmount": 0,
  "originalCurrency": "USD",
  "status": "Draft"
}
```

**Response — 201 Created**
```json
{
  "id": "led_e5f6",
  "invoice": "",
  "payee": "",
  "originalAmount": 0,
  "originalCurrency": "USD",
  "totalBilled": 0,
  "status": "Draft",
  "date": "2026-05-01",
  "invoiceDate": null,
  "category": "Engineering",
  "description": "",
  "attachment": null,
  "allocations": [],
  "computed": {
    "allocated": 0,
    "remaining": 0,
    "reconciliationStatus": "unallocated",
    "groupName": "Development Costs",
    "subgroupName": "Salaries"
  },
  "createdAt": "2026-05-01T17:30:00Z",
  "createdBy": "Alex Chen",
  "auditEntryId": "aud_g7h8"
}
```

### `PATCH /projects/:id/ledger/:entryId`

Partial update of a ledger entry. This is the highest-frequency endpoint in the system — every keystroke in an inline-edit cell could fire one (with debouncing).

**Request**
```http
PATCH /projects/prj_2k9l/ledger/led_e5f6 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "payee": "Localization Studio Ltd",
  "originalAmount": 4500,
  "originalCurrency": "EUR",
  "totalBilled": 4860,
  "invoice": "LOC-2026-007",
  "invoiceDate": "2026-04-25",
  "category": "Localization",
  "description": "FIGS localization, batch 1"
}
```

**Response — 200 OK**
```json
{
  "id": "led_e5f6",
  "payee": "Localization Studio Ltd",
  "originalAmount": 4500,
  "originalCurrency": "EUR",
  "totalBilled": 4860,
  "invoice": "LOC-2026-007",
  "invoiceDate": "2026-04-25",
  "category": "Localization",
  "description": "FIGS localization, batch 1",
  "computed": {
    "allocated": 0,
    "remaining": 4860,
    "reconciliationStatus": "unallocated",
    "groupName": "External Support Costs",
    "subgroupName": "Localization"
  },
  "updatedAt": "2026-05-01T17:35:00Z",
  "auditEntryId": "aud_i9j0"
}
```

**Notes**
- Server-side debouncing: collapse rapid successive PATCHes to the same entry into one audit entry (rolling 5-second window).
- Changing `category` recomputes `groupName` and `subgroupName`. If the change moves the entry between game and marketing ledgers, log it as a category-move audit (different action verb).
- If `originalCurrency` changes from USD to non-USD, **don't** auto-compute `totalBilled` — the prototype leaves it to the user. Surface a hint via response (e.g. `warnings: ["totalBilled may need updating"]`).

### `DELETE /projects/:id/ledger/:entryId`

Hard delete (the prototype's `delLedger` is destructive). The deletion also removes any allocations referencing this entry.

**Request**
```http
DELETE /projects/prj_2k9l/ledger/led_e5f6 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "deletedEntryId": "led_e5f6",
  "removedAllocations": 0,
  "snapshotId": "snap_z1a2",
  "auditEntryId": "aud_k1l2"
}
```

**Notes**
- Auto-snapshot before deletion.
- If the entry had allocations, those amounts are no longer counted against grid cells — the grid totals don't change, but the reconciliation status of the affected cells does.

### `POST /projects/:id/ledger/:entryId/attachment`

Upload an attachment for an existing entry.

**Request**
```http
POST /projects/prj_2k9l/ledger/led_a1b2/attachment HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: multipart/form-data

(file as multipart field "file")
```

**Response — 201 Created**
```json
{
  "entryId": "led_a1b2",
  "attachment": {
    "name": "Acme_April_Invoice.pdf",
    "type": "application/pdf",
    "size": 248320,
    "downloadUrl": "/projects/prj_2k9l/ledger/led_a1b2/attachment",
    "uploadedAt": "2026-05-01T18:00:00Z",
    "uploadedBy": "Alex Chen"
  }
}
```

**Notes**
- File stored in S3 (or equivalent) per the brief's Phase 2 requirements; not as base64 in the database.
- File size cap: 10 MB. Reject larger with `413 Payload Too Large`.
- Allowed types: `application/pdf`, `image/png`, `image/jpeg`, `image/webp`.

### `GET /projects/:id/ledger/:entryId/attachment`

Download an attachment.

**Request**
```http
GET /projects/prj_2k9l/ledger/led_a1b2/attachment HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK** — binary stream with appropriate `Content-Type` and `Content-Disposition: attachment; filename="..."`.

### `POST /projects/:id/ledger/import`

Import from QuickBooks CSV/XLSX. Two-step preview/commit pattern (same as the budget grid importer).

**Request — preview**
```http
POST /projects/prj_2k9l/ledger/import?action=preview HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: multipart/form-data

(file as multipart field "file")
```

**Response — 200 OK**
```json
{
  "previewId": "lpv_b3c4",
  "fileName": "QB_Export_April_2026.csv",
  "totalRows": 47,
  "newEntries": 38,
  "duplicates": 9,
  "duplicateDetail": [
    {
      "row": 4,
      "fingerprint": "Acme Studio Ltd|2026-04-01|12000|GBP|INV-2026-0142",
      "existingEntryId": "led_a1b2"
    }
  ],
  "newEntryPreview": [
    {
      "tempId": "tmp_001",
      "invoice": "INV-2026-0150",
      "payee": "Compliance Bureau",
      "originalAmount": 5500,
      "originalCurrency": "USD",
      "totalBilled": 5500,
      "invoiceDate": "2026-04-18",
      "category": "Ratings",
      "fingerprint": "Compliance Bureau|2026-04-18|5500|USD|INV-2026-0150"
    }
  ],
  "expiresAt": "2026-05-01T18:30:00Z"
}
```

**Request — commit**
```http
POST /projects/prj_2k9l/ledger/import?action=commit HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "previewId": "lpv_b3c4",
  "skipDuplicates": true,
  "destination": "reconciliation_queue"
}
```

**Response — 200 OK**
```json
{
  "imported": 38,
  "skippedDuplicates": 9,
  "destination": "reconciliation_queue",
  "queuedReconItemIds": ["recon_d5e6", "recon_f7g8"],
  "snapshotId": "snap_h9i0",
  "auditEntryId": "aud_m3n4"
}
```

**Notes**
- `destination`: `"reconciliation_queue"` (default — entries land in the recon queue and don't appear in ledger until allocated) or `"ledger"` (entries go directly into ledger as Draft status).
- Fingerprint formula: `payee | invoiceDate | originalAmount | originalCurrency | invoice` — matches the prototype's duplicate detection.
- `skipDuplicates: false` would import duplicates anyway (rare, but possible if user explicitly wants it).

### `POST /projects/:id/ledger/scan-invoice`

Upload an invoice file for AI extraction. Server proxies to Anthropic API (server-side key per Phase 1 brief), parses the response, returns a draft entry.

**Request**
```http
POST /projects/prj_2k9l/ledger/scan-invoice HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: multipart/form-data

(file as multipart field "file")
```

**Response — 200 OK**
```json
{
  "scanId": "scn_o1p2",
  "fileName": "ContractorInvoice_May.pdf",
  "extractedData": {
    "vendor": "Devon Lee Audio",
    "invoiceNumber": "DLA-2026-12",
    "invoiceDate": "2026-04-30",
    "currency": "USD",
    "total": 6500,
    "lineItems": [
      {
        "id": "li_q3r4",
        "description": "VO direction — Q2 sessions",
        "amount": 4500,
        "include": true,
        "suggestedCategory": "Audio"
      },
      {
        "id": "li_s5t6",
        "description": "Audio mastering — trailer cut",
        "amount": 2000,
        "include": true,
        "suggestedCategory": "Audio"
      }
    ],
    "confidence": "high"
  },
  "warnings": [],
  "createdAt": "2026-05-01T18:45:00Z"
}
```

**Notes**
- Returns a draft for review — does **not** create a ledger entry yet. The frontend opens a review modal where the user edits and confirms.
- After confirmation, the frontend calls `POST /projects/:id/ledger` (or a dedicated `POST /projects/:id/ledger/scan-invoice/:scanId/commit`) with the corrected data.
- Permissions: `submitInvoices` or `editLedger`. Note that the AI cost is non-trivial — consider rate-limiting per studio.

### `POST /projects/:id/ledger/:entryId/send-to-finance`

The "Send to Finance" workflow: marks the entry as sent and (in production) actually emails it.

**Request**
```http
POST /projects/prj_2k9l/ledger/led_a1b2/send-to-finance HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "recipients": ["finance@fictions.com"],
  "ccProducer": true,
  "note": "Please process per April payroll cycle."
}
```

**Response — 200 OK**
```json
{
  "entryId": "led_a1b2",
  "status": "Sent to Finance",
  "sentAt": "2026-05-01T18:50:00Z",
  "sentBy": "Alex Chen",
  "emailQueued": true,
  "emailMessageId": "msg_u7v8",
  "auditEntryId": "aud_w9x0"
}
```

**Notes**
- The prototype copies email text to clipboard and downloads the attachment for manual sending. In production, this endpoint queues a real email (with attachment) for delivery.
- Sets `entry.status = "Sent to Finance"` automatically.

### Permissions Summary (Ledger)

| Endpoint | Required permission | Additional gates |
|---|---|---|
| `GET /projects/:id/ledger` | studio member, project access | — |
| `POST /projects/:id/ledger` | `editLedger` | — |
| `PATCH /projects/:id/ledger/:id` | `editLedger` | — |
| `DELETE /projects/:id/ledger/:id` | `editLedger` | Allocated entries require `manageUsers` (since deletion affects reconciliation) |
| `POST .../attachment` | `editLedger` | — |
| `GET .../attachment` | studio member | — |
| `POST /ledger/import` | `importCSV` AND `editLedger` | — |
| `POST /ledger/scan-invoice` | `submitInvoices` OR `editLedger` | Rate-limited per studio (suggest 100/day) |
| `POST .../send-to-finance` | `editLedger` | Entry must have `totalBilled > 0` |

---

# Page 7: Reconciliation

The most operationally important page. Allocates ledger entries to budget grid cells `(rowId, monthKey, amount)`, supports closing months, and surfaces unallocated entries.

## Editable Fields

The "fields" on this page are not really persistent inputs — they're transient working state. The persistent edits happen via allocation creation and month closing.

### Working State (transient)

| Field | Type | Storage |
|---|---|---|
| Selected invoice | Reference | `allocModal` (one ledger entry ID at a time) |
| Selected month for grid view | Month key | `allocMonthKey` |
| Pending allocation map | `{rowId: amount}` | `pendingAlloc` (temporary; not persisted) |
| "Show all rows" toggle (per group) | Boolean | `reconShowAll[groupName]` |
| Recon tab filter | Month key or "all" | `reconTab` |

### Persistent Edits

Allocations are created in batches when the user clicks Commit. Each allocation is:

| Field | Storage path | Type |
|---|---|---|
| `id` | `allocation.id` | Generated |
| `rowId` | `allocation.rowId` | FK to grid row |
| `monthKey` | `allocation.monthKey` | YYYY-MM |
| `amount` | `allocation.amount` | Number |
| Stored as | `entry.allocations[]` | Array on the parent ledger entry |

Closed months are stored as:

| Field | Storage path |
|---|---|
| Closed month | `project.closedMonths[monthKey] = {closedAt, closedBy}` |

### Page-Level Actions

| Action | Effect |
|---|---|
| Click an unallocated invoice | Sets `allocModal`; defaults `allocMonthKey` to `entry.invoiceDate.slice(0, 7)` (or earliest open month if invoice month is closed) |
| Click a budget cell in the grid | Adds row to `pendingAlloc` with auto-suggested amount (cell remaining or invoice remaining, whichever smaller) |
| Adjust pending amount | Inline number input |
| Quick allocate by subsection | Distributes invoice remainder proportionally across subsection rows by their budgeted amounts |
| Commit | Persists allocations; if pending != remaining, triggers mismatch confirm modal |
| Adjust grid on commit (mismatch path) | Bumps `row.live[mKey]` to match total allocations; creates a `changeLog` batch (queued for amendment review) |
| Remove existing allocation | Direct delete |
| Close month | Sets `closedMonths[mKey]` |
| Reopen month | Removes `closedMonths[mKey]` |

## Calculated Fields

| Field | Formula |
|---|---|
| `allAllocations` | `flatten(ledger[].allocations[])` with `ledgerId` annotated |
| `cellAllocated[rowId_monthKey]` | `Σ allocations[].amount` keyed by row+month |
| `entryAllocated[ledgerId]` | `Σ entry.allocations[].amount` per entry |
| `unallocated[]` | `ledger.filter(e => total > 0 && allocated < total - 0.01)` |
| `fullyAllocated[]` | `ledger.filter(e => total > 0 && allocated >= total - 0.01)` |
| `totalLedger` | `Σ ledger[].totalBilled` |
| `totalAllocatedAmt` | `Σ entryAllocated[]` |
| `closedCount` | `Object.keys(closedMonths).length` |
| Per-cell `budgeted` | `row.live[selMonth]` |
| Per-cell `cellRemaining` | `budgeted − cellAllocated[rowId_selMonth]` |
| `selRemaining` (selected invoice's unallocated amount) | `selTotal − selUsed` |
| `pendingTotal` | `Σ pendingAlloc[]` |
| `mismatch diff` (commit-time) | `\|selRemaining − pendingTotal\|` |
| Quick-allocate share (per row) | `(row.budgeted / totalSubsectionBudgeted) × selRemaining`; last row gets remainder to handle rounding |
| `unallocMonths[]` | `[...new Set(unallocated.map(e => e.invoiceDate.slice(0, 7)))]` — drives the month tab list |

## Endpoints

### `GET /projects/:id/reconciliation`

Returns everything needed to render the page in one round-trip.

**Request**
```http
GET /projects/prj_2k9l/reconciliation HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "summary": {
    "unallocatedCount": 16,
    "allocated": { "amount": 145320, "ledgerTotal": 285420 },
    "monthsClosed": 9,
    "totalMonths": 36,
    "matchedEntries": { "matched": 118, "withTotal": 134 }
  },
  "unallocated": [
    {
      "id": "led_c3d4",
      "payee": "Pat Singh",
      "invoice": "FREELANCE-204",
      "invoiceDate": "2026-04-15",
      "category": "Engineering",
      "totalBilled": 8000,
      "allocated": 0,
      "remaining": 8000
    },
    {
      "id": "led_y3z4",
      "payee": "Compliance Bureau",
      "invoice": "INV-2026-0150",
      "invoiceDate": "2026-04-18",
      "category": "Ratings",
      "totalBilled": 5500,
      "allocated": 0,
      "remaining": 5500
    }
  ],
  "unallocMonths": ["2026-03", "2026-04"],
  "closedMonths": {
    "2025-08": { "closedAt": "2025-09-05T10:00:00Z", "closedBy": "Alex Chen" },
    "2025-09": { "closedAt": "2025-10-04T11:30:00Z", "closedBy": "Alex Chen" }
  },
  "cellAllocations": {
    "row_n3kp_2026-04": 15240,
    "row_n3kp_2026-03": 18000,
    "row_q4mr_2026-04": 8000
  }
}
```

### `GET /projects/:id/reconciliation/grid/:monthKey`

Returns the grid for a specific month with budgeted, allocated, and remaining values per row — used when the user selects an invoice and the allocation grid renders.

**Request**
```http
GET /projects/prj_2k9l/reconciliation/grid/2026-04 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "monthKey": "2026-04",
  "isClosed": false,
  "rows": [
    {
      "id": "row_n3kp",
      "name": "Lead Programmer — Pat Singh",
      "groupId": "devcosts",
      "groupName": "Development Costs",
      "groupColor": "#6366f1",
      "sectionId": "devfees",
      "sectionName": "Developer Fees",
      "subsectionId": "employees",
      "subsectionName": "Employees",
      "budgeted": 18500,
      "allocated": 15240,
      "remaining": 3260
    },
    {
      "id": "row_q4mr",
      "name": "UI Contractor",
      "groupId": "devcosts",
      "groupName": "Development Costs",
      "groupColor": "#6366f1",
      "sectionId": "devfees",
      "sectionName": "Developer Fees",
      "subsectionId": "contractors_dev",
      "subsectionName": "Contractors",
      "budgeted": 8000,
      "allocated": 8000,
      "remaining": 0
    },
    {
      "id": "row_w8x2",
      "name": "Audio Designer — Devon Lee",
      "groupId": "devcosts",
      "groupName": "Development Costs",
      "groupColor": "#6366f1",
      "sectionId": "devfees",
      "sectionName": "Developer Fees",
      "subsectionId": "employees",
      "subsectionName": "Employees",
      "budgeted": 0,
      "allocated": 0,
      "remaining": 0
    }
  ]
}
```

### `POST /projects/:id/reconciliation/allocate`

Commit a batch of allocations for one ledger entry. This is the central reconciliation action.

**Request — straightforward case (allocations match invoice remaining)**
```http
POST /projects/prj_2k9l/reconciliation/allocate HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "ledgerId": "led_c3d4",
  "monthKey": "2026-04",
  "allocations": [
    { "rowId": "row_q4mr", "amount": 5000 },
    { "rowId": "row_n3kp", "amount": 3000 }
  ],
  "adjustGrid": false
}
```

**Response — 200 OK**
```json
{
  "ledgerId": "led_c3d4",
  "createdAllocations": [
    { "id": "alloc_p3q4", "rowId": "row_q4mr", "monthKey": "2026-04", "amount": 5000 },
    { "id": "alloc_r5s6", "rowId": "row_n3kp", "monthKey": "2026-04", "amount": 3000 }
  ],
  "ledger": {
    "id": "led_c3d4",
    "totalBilled": 8000,
    "allocated": 8000,
    "remaining": 0,
    "reconciliationStatus": "fully_allocated"
  },
  "gridChangeLog": null,
  "auditEntryId": "aud_t7u8"
}
```

**Request — mismatch with `adjustGrid: true` (commits and updates grid + queues review)**
```http
POST /projects/prj_2k9l/reconciliation/allocate HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "ledgerId": "led_y3z4",
  "monthKey": "2026-04",
  "allocations": [
    { "rowId": "row_x4y5", "amount": 6500 }
  ],
  "adjustGrid": true,
  "note": "Compliance Bureau invoice came in $1000 over original budget — animator's overage is offset by lower QA cost."
}
```

The invoice total is $5,500 but the allocation is $6,500 — `adjustGrid: true` says "I know — bump the grid cell to match."

**Response — 200 OK**
```json
{
  "ledgerId": "led_y3z4",
  "createdAllocations": [
    { "id": "alloc_v9w0", "rowId": "row_x4y5", "monthKey": "2026-04", "amount": 6500 }
  ],
  "ledger": {
    "id": "led_y3z4",
    "totalBilled": 5500,
    "allocated": 6500,
    "remaining": -1000,
    "reconciliationStatus": "fully_allocated"
  },
  "gridChangeLog": {
    "id": "bch_a1b2",
    "status": "pending",
    "submittedBy": "Alex Chen",
    "submittedAt": "2026-05-01T19:00:00Z",
    "source": "reconcile",
    "note": "Compliance Bureau invoice came in $1000 over original budget…",
    "changes": [
      { "rowId": "row_x4y5", "rowName": "Compliance — Ratings", "monthKey": "2026-04", "oldValue": 5500, "newValue": 6500 }
    ]
  },
  "auditEntryId": "aud_x1y2"
}
```

**Notes**
- The prototype's commit logic (lines 5965–6002) only bumps cells where `totalAllocForCell > currentVal || currentVal === 0` — partial allocations don't reduce the grid. Backend should preserve this asymmetry.
- `adjustGrid: false` with a mismatch returns a `409 Conflict` and prompts the user to choose. Don't silently apply over-allocation.
- The `gridChangeLog` batch is what surfaces on the Amendments page for separate approval. The grid value is updated immediately, but the change is also logged for review.

**Validation**
- All `monthKey`s in `allocations[]` must equal the request-level `monthKey` (allocations are single-month per call).
- Month must not be closed (or override permission required).
- Each `rowId` must exist in the project.
- `amount > 0` for every allocation.
- `Σ amounts ≤ ledger.totalBilled` unless `adjustGrid: true` and user has `editBudget`.

### `DELETE /projects/:id/reconciliation/allocations/:allocationId`

Remove a single allocation.

**Request**
```http
DELETE /projects/prj_2k9l/reconciliation/allocations/alloc_p3q4 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "deletedAllocationId": "alloc_p3q4",
  "ledgerId": "led_c3d4",
  "ledger": {
    "totalBilled": 8000,
    "allocated": 3000,
    "remaining": 5000,
    "reconciliationStatus": "partial"
  },
  "auditEntryId": "aud_z3a4"
}
```

**Notes**
- Removing allocations does **not** revert any grid changes that were made during the original commit. Those are independent.
- Cannot remove allocations from closed months without first reopening the month.

### `POST /projects/:id/reconciliation/auto-match`

Auto-match allocations by amount across all unallocated entries. Returns a list of suggestions; user confirms before committing. (The brief specifies "auto-match by amount" — the prototype has this only at QB import time; this endpoint exposes it broadly.)

**Request**
```http
POST /projects/prj_2k9l/reconciliation/auto-match HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "monthKey": "2026-04",
  "tolerance": 0.01,
  "matchStrategy": "exact_amount"
}
```

**Response — 200 OK**
```json
{
  "monthKey": "2026-04",
  "suggestions": [
    {
      "ledgerId": "led_c3d4",
      "ledgerPayee": "Pat Singh",
      "ledgerTotal": 8000,
      "matchedRowId": "row_n3kp",
      "matchedRowName": "Lead Programmer — Pat Singh",
      "matchedBudgeted": 8000,
      "confidence": "high",
      "reason": "Exact amount match; payee name matches row name"
    },
    {
      "ledgerId": "led_y3z4",
      "ledgerPayee": "Compliance Bureau",
      "ledgerTotal": 5500,
      "matchedRowId": "row_x4y5",
      "matchedRowName": "Compliance — Ratings",
      "matchedBudgeted": 5500,
      "confidence": "medium",
      "reason": "Exact amount match; category match (Ratings)"
    }
  ],
  "unmatched": [
    {
      "ledgerId": "led_z9a8",
      "ledgerPayee": "Localization Studio Ltd",
      "ledgerTotal": 4860,
      "reason": "No exact amount match in this month"
    }
  ]
}
```

**Request — commit the suggestions**
```http
POST /projects/prj_2k9l/reconciliation/auto-match/commit HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "monthKey": "2026-04",
  "approvedSuggestions": ["led_c3d4", "led_y3z4"]
}
```

**Response — 200 OK**
```json
{
  "appliedCount": 2,
  "createdAllocations": 2,
  "skippedSuggestions": 0,
  "auditEntryId": "aud_b5c6"
}
```

**Notes**
- The two-step pattern (suggest → confirm) is critical. Don't auto-apply without user confirmation — wrong matches are painful to undo.
- Match strategies the backend should support:
  - `exact_amount` — invoice total equals row's budgeted value
  - `payee_name` — payee matches row's role/name field
  - `category` — invoice category matches a row whose subsection maps to that category
  - `combined` — weighted score across all of the above

### `POST /projects/:id/reconciliation/months/:monthKey/close`

Close a month. After closing, cells in this month become read-only on the grid and cannot be allocated against without reopening.

**Request**
```http
POST /projects/prj_2k9l/reconciliation/months/2026-03/close HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "monthKey": "2026-03",
  "closedAt": "2026-05-01T19:30:00Z",
  "closedBy": "Alex Chen",
  "warnings": [
    {
      "type": "unallocated_entries",
      "message": "3 ledger entries with invoiceDate in 2026-03 are not fully allocated.",
      "entryIds": ["led_d7e8", "led_f9g0", "led_h1i2"]
    }
  ],
  "auditEntryId": "aud_d7e8"
}
```

**Notes**
- Closing is permitted even with unallocated entries — but warnings are returned. The frontend should surface them and require explicit confirmation.
- A separate stricter endpoint variant `POST .../close?strict=true` could reject the close if any entries are unallocated.

### `DELETE /projects/:id/reconciliation/months/:monthKey/close`

Reopen a previously closed month.

**Request**
```http
DELETE /projects/prj_2k9l/reconciliation/months/2026-03/close HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "monthKey": "2026-03",
  "reopenedAt": "2026-05-01T19:45:00Z",
  "reopenedBy": "Alex Chen",
  "auditEntryId": "aud_f9g0"
}
```

### Permissions Summary (Reconciliation)

| Endpoint | Required permission | Additional gates |
|---|---|---|
| `GET /reconciliation` | studio member, project access | — |
| `GET /reconciliation/grid/:monthKey` | studio member, project access | — |
| `POST /reconciliation/allocate` | `editLedger` | If `adjustGrid: true`, also requires `editBudget`; month must not be closed |
| `DELETE /reconciliation/allocations/:id` | `editLedger` | Source month must not be closed |
| `POST /reconciliation/auto-match` | `editLedger` | — |
| `POST /reconciliation/auto-match/commit` | `editLedger` | — |
| `POST .../months/:key/close` | `editLedger` | — |
| `DELETE .../months/:key/close` | `editLedger` | — |

---

# Page 8: Invoice Queue

The intake pipeline for invoices uploaded by anyone. They wait here until reviewed; on approval, a ledger entry is created.

## Editable Fields

### Per-Invoice Queue Entry

| Field | Type | Storage path |
|---|---|---|
| File | File upload | `aux.invoiceQueue[].fileData` (base64 in prototype; S3 in production) |
| `submittedBy` | Text | Auto-set from auth context |
| `note` | Text | `aux.invoiceQueue[].note` (optional, set at upload time) |
| `status` | Enum | `pending` / `approved` / `rejected` |

### Invoice Review Modal (when reviewer opens an item)

The AI-extracted data is fully editable before approval:

| Field | Type | Storage path |
|---|---|---|
| Vendor | Text | `extractedData.vendor` |
| Invoice Number | Text | `extractedData.invoiceNumber` |
| Invoice Date | Date | `extractedData.date` |
| Currency | Dropdown | `extractedData.currency` |
| Total | Number | `extractedData.total` |
| Line Items (per row) | Description / Amount / Include checkbox / Suggested category | `extractedData.lineItems[]` |
| Final category assignment | Picker | Per-line-item, before commit |

### Page-Level Actions

| Action | Effect |
|---|---|
| Upload Invoice | Creates a queue entry; status `pending` |
| Review (per pending item) | Opens review modal with AI-extracted data |
| Approve (in review modal) | Creates ledger entries from the included line items; queue entry status → `approved` |
| Reject | Sets status → `rejected` |

## Calculated Fields

| Field | Formula |
|---|---|
| `pending` | `invoiceQueue.filter(q => q.status === "pending")` |
| `processed` | `invoiceQueue.filter(q => q.status !== "pending")` |
| Pending count (badge) | `pending.length` |
| Processed count (badge) | `processed.length` |
| Per-line-item `included total` | `Σ lineItems.filter(li => li.include).amount` (drives the "ledger entries to create" preview) |
| `mediaType` icon | Based on `q.mediaType` — `📄` for PDF, `🖼️` for image |
| `canReview` | `perms.approveInvoices \|\| perms.editLedger` |

## Endpoints

### `GET /projects/:id/invoices`

Returns the invoice queue with pending and processed items.

**Request**
```http
GET /projects/prj_2k9l/invoices?status=pending HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Query parameters**
- `status` — `pending` / `approved` / `rejected` / `all` (default `all`)
- `submittedBy` — filter by submitter name (useful for the Dev Portal's "My Submissions")
- `limit`, `cursor` — pagination

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "summary": {
    "pending": 3,
    "approved": 18,
    "rejected": 2
  },
  "items": [
    {
      "id": "inv_a1b2",
      "fileName": "ContractorInvoice_May.pdf",
      "mediaType": "application/pdf",
      "fileSize": 248320,
      "downloadUrl": "/projects/prj_2k9l/invoices/inv_a1b2/file",
      "submittedBy": "Jamie Rivera",
      "submittedAt": "2026-04-28T13:45:00Z",
      "submitterRole": "developer",
      "status": "pending",
      "note": "Animator invoice for May milestone deliverables.",
      "extractedData": {
        "vendor": "Devon Lee Audio",
        "invoiceNumber": "DLA-2026-12",
        "invoiceDate": "2026-04-30",
        "currency": "USD",
        "total": 6500,
        "lineItems": [
          {
            "id": "li_q3r4",
            "description": "VO direction — Q2 sessions",
            "amount": 4500,
            "include": true,
            "suggestedCategory": "Audio"
          },
          {
            "id": "li_s5t6",
            "description": "Audio mastering — trailer cut",
            "amount": 2000,
            "include": true,
            "suggestedCategory": "Audio"
          }
        ],
        "confidence": "high"
      }
    },
    {
      "id": "inv_c3d4",
      "fileName": "QA_Vendor_Invoice.pdf",
      "mediaType": "application/pdf",
      "fileSize": 184230,
      "downloadUrl": "/projects/prj_2k9l/invoices/inv_c3d4/file",
      "submittedBy": "Alex Chen",
      "submittedAt": "2026-04-29T09:00:00Z",
      "submitterRole": "admin",
      "status": "approved",
      "note": null,
      "reviewedBy": "Sam Park",
      "reviewedAt": "2026-04-29T11:30:00Z",
      "createdLedgerEntryIds": ["led_n5p6", "led_q7r8"]
    }
  ],
  "pagination": {
    "total": 23,
    "returned": 23,
    "nextCursor": null
  }
}
```

### `POST /projects/:id/invoices`

Upload a new invoice. Server stores the file, calls AI for extraction, and returns the queue item with extracted data ready for review.

**Request**
```http
POST /projects/prj_2k9l/invoices HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: multipart/form-data

(file as multipart field "file"; optional "note" text field)
```

**Response — 201 Created**
```json
{
  "id": "inv_e5f6",
  "fileName": "ContractorInvoice_May.pdf",
  "mediaType": "application/pdf",
  "fileSize": 248320,
  "downloadUrl": "/projects/prj_2k9l/invoices/inv_e5f6/file",
  "submittedBy": "Alex Chen",
  "submittedAt": "2026-05-01T20:00:00Z",
  "status": "pending",
  "note": "May milestone audio invoice",
  "extractedData": {
    "vendor": "Devon Lee Audio",
    "invoiceNumber": "DLA-2026-13",
    "invoiceDate": "2026-05-01",
    "currency": "USD",
    "total": 6500,
    "lineItems": [
      {
        "id": "li_t9u0",
        "description": "VO direction — Q2 sessions",
        "amount": 4500,
        "include": true,
        "suggestedCategory": "Audio"
      }
    ],
    "confidence": "high"
  },
  "auditEntryId": "aud_v1w2"
}
```

**Notes**
- File stored in S3 with a server-generated key. `downloadUrl` is a signed URL or a routed endpoint that streams the file with auth.
- AI extraction runs synchronously here for the prototype's UX; for very large or batch uploads, consider returning `202 Accepted` with a polling URL.
- File size cap: 10 MB. Allowed types: PDF, PNG, JPG, WEBP.
- Permission: `submitInvoices` or `editLedger`. Critically, **developers can hit this endpoint** — they can submit invoices via the Developer Portal.

### `PATCH /projects/:id/invoices/:invoiceId`

Update the extracted data before approval. Used by the review modal as the user corrects AI mistakes.

**Request**
```http
PATCH /projects/prj_2k9l/invoices/inv_e5f6 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "extractedData": {
    "vendor": "Devon Lee Audio LLC",
    "invoiceDate": "2026-04-30",
    "lineItems": [
      {
        "id": "li_t9u0",
        "description": "VO direction — Q2 sessions (revised)",
        "amount": 4800,
        "include": true,
        "suggestedCategory": "Audio"
      },
      {
        "id": "li_x3y4",
        "description": "Studio time — additional",
        "amount": 1700,
        "include": true,
        "suggestedCategory": "Audio"
      }
    ]
  }
}
```

**Response — 200 OK**
```json
{
  "id": "inv_e5f6",
  "extractedData": {
    "vendor": "Devon Lee Audio LLC",
    "invoiceDate": "2026-04-30",
    "lineItems": [
      { "id": "li_t9u0", "description": "VO direction — Q2 sessions (revised)", "amount": 4800, "include": true, "suggestedCategory": "Audio" },
      { "id": "li_x3y4", "description": "Studio time — additional", "amount": 1700, "include": true, "suggestedCategory": "Audio" }
    ],
    "total": 6500
  },
  "updatedAt": "2026-05-01T20:15:00Z",
  "auditEntryId": "aud_z5a6"
}
```

### `POST /projects/:id/invoices/:invoiceId/approve`

Approve the invoice and create ledger entries from the included line items.

**Request**
```http
POST /projects/prj_2k9l/invoices/inv_e5f6/approve HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "lineItemAssignments": [
    { "lineItemId": "li_t9u0", "category": "Audio", "createSeparateEntry": true },
    { "lineItemId": "li_x3y4", "category": "Audio", "createSeparateEntry": false }
  ],
  "consolidateInto": true,
  "approvalNote": "Approved per producer review."
}
```

**Notes on the request shape**
- `lineItemAssignments[]` — per-line-item: which category to use, and whether to create a separate ledger entry per line.
- `consolidateInto: true` — alternative behavior: roll all included line items into one ledger entry (useful when an invoice has many small items that don't need separate tracking).

**Response — 200 OK**
```json
{
  "invoiceId": "inv_e5f6",
  "status": "approved",
  "reviewedBy": "Sam Park",
  "reviewedAt": "2026-05-01T20:30:00Z",
  "approvalNote": "Approved per producer review.",
  "createdLedgerEntryIds": ["led_b7c8"],
  "ledgerEntries": [
    {
      "id": "led_b7c8",
      "payee": "Devon Lee Audio LLC",
      "invoice": "DLA-2026-13",
      "invoiceDate": "2026-04-30",
      "originalAmount": 6500,
      "originalCurrency": "USD",
      "totalBilled": 6500,
      "category": "Audio",
      "description": "VO direction — Q2 sessions (revised); Studio time — additional",
      "status": "Draft",
      "attachment": {
        "name": "ContractorInvoice_May.pdf",
        "downloadUrl": "/projects/prj_2k9l/ledger/led_b7c8/attachment"
      }
    }
  ],
  "auditEntryId": "aud_d7e8"
}
```

**Notes**
- The original invoice file is automatically attached to the new ledger entry (or to all entries if multiple were created).
- Default `status` for the created ledger entry is `Draft` — the producer can move it to `Sent to Finance` later.
- The invoice queue item is preserved with `status: approved` for audit history.

### `POST /projects/:id/invoices/:invoiceId/reject`

Reject the invoice without creating ledger entries.

**Request**
```http
POST /projects/prj_2k9l/invoices/inv_e5f6/reject HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "reason": "Wrong project — please resubmit to Project Nightfall."
}
```

**Response — 200 OK**
```json
{
  "invoiceId": "inv_e5f6",
  "status": "rejected",
  "reviewedBy": "Sam Park",
  "reviewedAt": "2026-05-01T20:35:00Z",
  "rejectionReason": "Wrong project — please resubmit to Project Nightfall.",
  "auditEntryId": "aud_f9g0"
}
```

**Notes**
- The submitter is notified (especially important for developer portal submissions).
- The file is retained for record-keeping; can be downloaded but not approved.

### `GET /projects/:id/invoices/:invoiceId/file`

Download the original invoice file.

**Request**
```http
GET /projects/prj_2k9l/invoices/inv_e5f6/file HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK** — binary stream with appropriate `Content-Type` and `Content-Disposition`.

### `POST /projects/:id/invoices/:invoiceId/rerun-ai`

Re-extract data via AI (useful when the original extraction was poor). Doesn't replace the user's edits unless they confirm.

**Request**
```http
POST /projects/prj_2k9l/invoices/inv_e5f6/rerun-ai HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "invoiceId": "inv_e5f6",
  "newExtractedData": {
    "vendor": "Devon Lee Audio",
    "invoiceNumber": "DLA-2026-13",
    "invoiceDate": "2026-04-30",
    "currency": "USD",
    "total": 6500,
    "lineItems": [
      { "id": "li_h1i2", "description": "Voice over direction", "amount": 4500, "suggestedCategory": "Audio", "include": true },
      { "id": "li_j3k4", "description": "Audio mastering", "amount": 2000, "suggestedCategory": "Audio", "include": true }
    ],
    "confidence": "high"
  },
  "currentExtractedData": {
    "vendor": "Devon Lee Audio LLC"
  },
  "applied": false
}
```

**Notes**
- Returns both the new extraction and the current state — the frontend asks the user "replace or keep?".
- Apply is a separate PATCH call.
- Rate-limited because AI calls cost money.

### Permissions Summary (Invoice Queue)

| Endpoint | Required permission | Additional gates |
|---|---|---|
| `GET /invoices` | studio member, project access | Developers see only `submittedBy === devName` |
| `POST /invoices` | `submitInvoices` OR `editLedger` | File size ≤ 10 MB; allowed types only; rate-limited |
| `PATCH /invoices/:id` | `approveInvoices` OR `editLedger` | Item must be `pending` |
| `POST /invoices/:id/approve` | `approveInvoices` OR `editLedger` | Item must be `pending`; at least one line item included |
| `POST /invoices/:id/reject` | `approveInvoices` OR `editLedger` | Item must be `pending` |
| `GET /invoices/:id/file` | studio member, project access | — |
| `POST /invoices/:id/rerun-ai` | `approveInvoices` OR `editLedger` | Rate-limited |

---

# Cross-Cutting Notes for the Backend

## 1. Real-time updates across the trio

These three pages are tightly coupled — an action on one affects the others:

- Approving an invoice creates a ledger entry → Reconciliation's "unallocated" count goes up.
- Allocating a ledger entry → the grid cell's allocated amount changes → Reconciliation's KPIs refresh.
- Closing a month → ledger entries with that invoiceDate can no longer be edited via reconcile.

Suggested WebSocket events on a single channel `projects/:id/financial`:

| Event | Triggered by |
|---|---|
| `ledger.entry_created` | New ledger entry (manual add, import, invoice approval) |
| `ledger.entry_updated` | PATCH to a ledger entry |
| `ledger.entry_deleted` | DELETE |
| `reconciliation.allocation_created` | Successful allocate |
| `reconciliation.allocation_removed` | Allocation deleted |
| `reconciliation.month_closed` | Month close |
| `reconciliation.month_reopened` | Month reopen |
| `invoice.queued` | Upload |
| `invoice.approved` | Approval |
| `invoice.rejected` | Rejection |

Frontend listens on whichever pages the user has open and updates accordingly.

## 2. Audit log granularity

Every write across the trio produces an audit entry. The granularity is high (per cell, per allocation, per status change), so:

- Don't write audits inline in the request hot path — queue them to a background worker.
- Index audit entries by `projectId`, `entityType`, `entityId`, and `timestamp` for the History page filters.
- Retain at least the prototype's 200-entry cap **per project** — but uncapped in production.

## 3. The "Total Billed" vs "Original Amount" duality

The prototype lets a non-USD invoice have an `originalAmount` (in source currency) and a separately-entered `totalBilled` (in USD). There's no enforced relationship between them — the user is trusted to compute the conversion. This is intentional (FX rates fluctuate; finance often uses a specific date's rate) but error-prone.

The backend should:

- **Accept** the asymmetry as the prototype does.
- **Surface a warning** in PATCH responses when `originalCurrency !== "USD"` and `totalBilled` doesn't match `originalAmount × spotRateAtInvoiceDate` within a reasonable tolerance.
- **Provide an endpoint** like `POST /projects/:id/ledger/:id/recompute-billed?rateSource=eod_invoice_date` for users who want automatic conversion.

## 4. Invoice file storage and lifecycle

- Files belong in S3 (or equivalent) — never the database.
- Use a project-scoped bucket prefix: `invoices/projects/<projectId>/<invoiceId>/<filename>`.
- Retention policy: keep files indefinitely for approved invoices (legal record); delete files for rejected invoices after 90 days unless flagged for retention.
- Signed download URLs expire in 1 hour; the application's GET endpoint is the canonical access point because it can re-issue URLs and check permissions on every download.

## 5. Reconciliation race conditions

Two reviewers might try to allocate the same ledger entry simultaneously. The backend must:

- Use optimistic locking on the ledger entry: `If-Match: <etag>` header on POST allocate.
- On conflict, return `409` with the current state so the frontend can re-render and the user can re-attempt.
- Same approach for closed-month state — closing a month is a fast operation and conflicts here are easy to recover from.

---

That's the operational financial trio fully spec'd. Want me to tackle the **Sales/Sales Intelligence/Per-platform pages** next (the analytics layer), or jump to the **Developer Portal** endpoints (which are the smallest but the most security-sensitive surface)?


# Page 9: Sales Dashboard

The aggregate view across all platforms.

## Editable Fields

The Sales Dashboard itself is **read-only** — it visualizes data uploaded elsewhere. The only "editable" surface is the data import that happens via per-platform pages.

| Field | Type | Storage path |
|---|---|---|
| (none — display only) | — | Reads from `aux.salesData[]` and `aux.wishlistData[]` |

## Calculated Fields

Everything on this page is derived from `salesData[]` and `wishlistData[]` aggregations.

### Top KPI Strip

| Field | Formula |
|---|---|
| `totalRevenue` | `Σ salesData[].revenue` |
| `totalUnits` | `Σ salesData[].units` |
| `totalWishlists` | `Σ salesData[].wishlists + Σ wishlistData[].adds` |
| `avgRevenuePerUnit` | `totalRevenue / totalUnits` |

### Wishlist Metrics

| Field | Formula |
|---|---|
| `totalWlAdds` | `Σ wishlistData[].adds` |
| `totalWlDeletes` | `Σ wishlistData[].deletes` |
| `totalWlPurchases` | `Σ wishlistData[].purchases` |
| `wlConvRate` | `totalWlPurchases / totalWlAdds × 100` |

### Refunds & Quality

| Field | Formula |
|---|---|
| `totalRefunds` | `Σ salesData[].refunds` |
| `refundRate` | `totalRefunds / totalUnits × 100` |

### Per-Platform Breakdown

| Field | Formula |
|---|---|
| `byPlatform[platform]` | Group-by reduction: `{revenue: Σ revenue, units: Σ units, wishlists: Σ wishlists, refunds: Σ refunds}` |
| `revPct` per platform | `(platform.revenue / totalRevenue) × 100` |
| `unitPct` per platform | `(platform.units / totalUnits) × 100` |
| `pieRevData` | Filter platforms with `revenue > 0`, format for Recharts |
| `pieUnitData` | Filter platforms with `units > 0` |

### Time-Series

| Field | Formula |
|---|---|
| `byMonth[month]` | Group-by reduction over `salesData` keyed by `e.date.slice(0, 7)`; tracks per-platform revenue + total + units |
| `monthlyData[]` | `Object.values(byMonth).sort((a, b) => a.month.localeCompare(b.month))` |
| `cumData[]` | Running cumulative revenue across `monthlyData` |
| `wlByDay[]` | Per-day wishlist `{adds, deletes, net: adds - deletes}` |
| `wlCumData[]` | Running cumulative net wishlists |
| `wlByMonth[]` | Per-month wishlist aggregation |

### Regional Breakdown

| Field | Formula |
|---|---|
| `byRegion[region]` | Group-by reduction: `{revenue, units}` |
| `regionData[]` | Top 12 regions sorted by `revenue` descending |

### ROI Comparison

| Field | Formula |
|---|---|
| `totalBudget` | `greenlitAll` (from project) |
| `budgetROI` | `((totalRevenue − totalBudget) / totalBudget) × 100` |

## Endpoints

### `GET /projects/:id/sales`

Returns aggregated sales data with all the rollups the dashboard needs. Server pre-computes everything so the frontend doesn't iterate the raw arrays.

**Request**
```http
GET /projects/prj_2k9l/sales?from=2026-01-01&to=2026-04-30 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Query parameters**
- `from` / `to` — date range filter (optional; defaults to all data)
- `platform` — filter to a single platform (optional)
- `granularity` — `month` (default) or `day`

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "filters": {
    "from": "2026-01-01",
    "to": "2026-04-30",
    "platform": null
  },
  "summary": {
    "totalRevenue": 3245890,
    "totalUnits": 87421,
    "totalWishlists": 142350,
    "totalRefunds": 1842,
    "avgRevenuePerUnit": 37.13,
    "refundRate": 2.11,
    "wlConvRate": 8.42,
    "budgetROI": -66.71,
    "totalBudget": 9750000
  },
  "byPlatform": [
    {
      "platform": "steam",
      "label": "Steam",
      "color": "#1b2838",
      "revenue": 1842500,
      "units": 52340,
      "wishlists": 98420,
      "refunds": 1102,
      "revPct": 56.77,
      "unitPct": 59.87
    },
    {
      "platform": "playstation",
      "label": "PlayStation",
      "color": "#003791",
      "revenue": 720340,
      "units": 18250,
      "wishlists": 0,
      "refunds": 312,
      "revPct": 22.19,
      "unitPct": 20.87
    },
    {
      "platform": "xbox",
      "label": "Xbox",
      "color": "#107C10",
      "revenue": 485200,
      "units": 12380,
      "wishlists": 0,
      "refunds": 248,
      "revPct": 14.95,
      "unitPct": 14.16
    }
  ],
  "monthly": [
    {
      "month": "2026-01",
      "total": 1024850,
      "units": 28420,
      "byPlatform": { "steam": 612300, "playstation": 245820, "xbox": 166730, "switch": 0, "epic": 0 }
    },
    {
      "month": "2026-02",
      "total": 845200,
      "units": 21340,
      "byPlatform": { "steam": 502800, "playstation": 198450, "xbox": 143950, "switch": 0, "epic": 0 }
    }
  ],
  "cumulative": [
    { "month": "2026-01", "cumRev": 1024850 },
    { "month": "2026-02", "cumRev": 1870050 },
    { "month": "2026-03", "cumRev": 2580340 },
    { "month": "2026-04", "cumRev": 3245890 }
  ],
  "byRegion": [
    { "region": "United States", "revenue": 1248320, "units": 32420 },
    { "region": "United Kingdom", "revenue": 412580, "units": 11240 },
    { "region": "Germany", "revenue": 384120, "units": 10680 },
    { "region": "Japan", "revenue": 248950, "units": 6820 }
  ],
  "wishlists": {
    "totalAdds": 142350,
    "totalDeletes": 12420,
    "totalPurchases": 11984,
    "convRate": 8.42,
    "monthly": [
      { "month": "2026-01", "adds": 38420, "deletes": 2840 },
      { "month": "2026-02", "adds": 28930, "deletes": 3120 }
    ],
    "cumulativeNet": [
      { "date": "2026-01-01", "cumulative": 1240 },
      { "date": "2026-01-02", "cumulative": 2380 }
    ]
  },
  "lastUploadedAt": "2026-04-30T22:00:00Z"
}
```

### Permissions

| Endpoint | Required permission |
|---|---|
| `GET /projects/:id/sales` | studio member, project access |

---

# Page 10: Per-Platform Pages

Five near-identical pages: `sales_steam`, `sales_xbox`, `sales_ps`, `sales_switch`, `sales_epic`. Each accepts platform-specific CSV uploads and shows that platform's slice of `salesData[]`.

## Editable Fields

The only editable surface is the **CSV upload** — there's no inline editing of sales data after import.

| Field | Type | Storage path |
|---|---|---|
| Sales CSV | File | Each row parsed into `aux.salesData[]` (filter by `e.platform === platformKey`) |
| Wishlist CSV (Steam only) | File | `aux.wishlistData[]` |

The platform mapping (line 8039):

```
sales_steam   → "steam"
sales_xbox    → "xbox"
sales_ps      → "playstation"
sales_switch  → "switch"
sales_epic    → "epic"
```

## Calculated Fields

Same as the Sales Dashboard, but filtered to one platform. Specifically:

| Field | Formula |
|---|---|
| Platform-filtered `salesData` | `salesData.filter(e => e.platform === platformKey)` |
| Per-platform KPIs | `Σ` over the filtered set |
| Per-platform monthly chart | Same group-by-month as the dashboard, but on the filtered set |
| Per-platform regional breakdown | Same group-by-region |

The CSV parser is platform-specific — each platform exports a different schema, so the parser needs to know which platform's format to expect.

## Endpoints

### `POST /projects/:id/sales/import/:platform`

Upload a sales CSV for a specific platform.

**Request**
```http
POST /projects/prj_2k9l/sales/import/steam HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: multipart/form-data

(file as multipart field "file")
```

**Path parameter** — `:platform` is one of `steam`, `xbox`, `playstation`, `switch`, `epic`.

**Query parameters**
- `mode` — `append` (default) or `replace_platform_data` (wipes existing rows for this platform first)

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "platform": "steam",
  "fileName": "Steam_Sales_April_2026.csv",
  "imported": {
    "salesRows": 28,
    "wishlistRows": 0
  },
  "duplicatesSkipped": 0,
  "warnings": [],
  "summary": {
    "platformTotalRevenue": 1842500,
    "platformTotalUnits": 52340,
    "platformWishlists": 98420
  },
  "importedAt": "2026-05-01T20:30:00Z",
  "importedBy": "Alex Chen",
  "snapshotId": "snap_a1b2",
  "auditEntryId": "aud_c3d4"
}
```

**Response — 200 OK with warnings (partial parse)**
```json
{
  "projectId": "prj_2k9l",
  "platform": "steam",
  "fileName": "Steam_Sales_April_2026.csv",
  "imported": {
    "salesRows": 24,
    "wishlistRows": 0
  },
  "duplicatesSkipped": 4,
  "warnings": [
    {
      "row": 8,
      "issue": "Date format unrecognized",
      "raw": "Apr-2026"
    },
    {
      "row": 12,
      "issue": "Negative revenue value (refund?)",
      "raw": "-89.99"
    }
  ],
  "summary": { "platformTotalRevenue": 1742500 }
}
```

**Notes for the backend**
- Each platform has a distinct CSV schema. Steam exports include game ID, package ID, country, units, gross revenue, refunds, and wishlist data. Xbox's format is entirely different. The backend should have a per-platform parser module.
- Duplicate detection: fingerprint = `platform | date | region | units | revenue` — skip exact duplicates on append.
- Permissions: `editSales`.
- File size cap: 10 MB. Allowed types: `.csv`, `.xlsx`, `.xls`.

### `POST /projects/:id/sales/wishlist-import/steam`

Steam-only endpoint for wishlist data (separate file from sales).

**Request**
```http
POST /projects/prj_2k9l/sales/wishlist-import/steam HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: multipart/form-data

(file as multipart field "file")
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "platform": "steam",
  "fileName": "Steam_Wishlist_April.csv",
  "imported": {
    "wishlistRows": 30
  },
  "summary": {
    "totalAdds": 38420,
    "totalDeletes": 2840,
    "totalPurchases": 4120,
    "netChange": 35580
  },
  "importedAt": "2026-05-01T20:35:00Z",
  "auditEntryId": "aud_e5f6"
}
```

### `GET /projects/:id/sales/:platform`

Returns platform-filtered sales data — the data backing the per-platform page.

**Request**
```http
GET /projects/prj_2k9l/sales/steam HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK** — Same shape as `GET /projects/:id/sales` but pre-filtered to one platform. The `byPlatform` array contains a single entry; `monthly` and `byRegion` are scoped accordingly.

### `DELETE /projects/:id/sales/:platform`

Wipe all sales data for one platform (useful when re-importing from scratch).

**Request**
```http
DELETE /projects/prj_2k9l/sales/steam HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "platform": "steam",
  "deletedRows": 487,
  "deletedWishlistRows": 142,
  "snapshotId": "snap_g7h8",
  "auditEntryId": "aud_i9j0"
}
```

**Notes**
- Auto-snapshots before deletion.
- Does not affect other platforms' data.
- Permissions: `editSales` AND `deleteRows`.

### Permissions Summary (Per-Platform)

| Endpoint | Required permission |
|---|---|
| `GET /sales/:platform` | studio member, project access |
| `POST /sales/import/:platform` | `editSales` |
| `POST /sales/wishlist-import/steam` | `editSales` |
| `DELETE /sales/:platform` | `editSales` AND `deleteRows` |

---

# Page 11: Sales Intelligence (Promos)

Two-tab analytical layer: Pricing & Promotions (auto-detects sale periods) and Paid Media (correlates ad spend with revenue).

## Editable Fields

### Promo Period Names

| Field | Type | Storage path |
|---|---|---|
| Period name (per detected period) | Text | `aux.promoData[periodKey].name` |

The periods themselves are auto-detected — only their human-readable names are user-editable.

### Other Surfaces

The Media tab is fully read-only — it visualizes ledger entries (categorized as marketing) against sales data.

## Calculated Fields

### Promos Tab

| Field | Formula |
|---|---|
| `pricesWithUnits[]` | Per month: `{month, units, revenue, effPrice: revenue/units}` |
| `medianPrice` | Median of `pricesWithUnits[].effPrice` (the baseline retail) |
| `threshold` | `medianPrice × 0.9` (10% below baseline) |
| `detectedPeriods[]` | Contiguous runs of months where `effPrice < threshold`; each becomes one period |
| `periodStats[]` per detected period | `{startMonth, endMonth, months[], avgPrice, discount: (1 - avgPrice/medianPrice) × 100, rev: Σ revenue, units: Σ units, revPerMonth, revLift: ((revPerMonth - baseRevPerMonth) / baseRevPerMonth) × 100, unitLift}` |
| `baseRevPerMonth` | Average monthly revenue across non-sale months |
| `baseUnitsPerMonth` | Average monthly units across non-sale months |
| Per-period name | From `aux.promoData[periodKey].name` (user-editable) |

### Media Tab

| Field | Formula |
|---|---|
| `mktgEntries[]` | `ledger.filter(e => CAT_TO_GROUP[e.category] === "Marketing Costs")` |
| `mktgByMonth[month]` | `{spend: Σ totalBilled, items: [...descriptions]}` keyed by `e.invoiceDate.slice(0, 7)` |
| `totalMediaSpend` | `Σ mktgEntries[].totalBilled` |
| `totalSalesRev` | `Σ salesData[].revenue` |
| `totalSalesUnits` | `Σ salesData[].units` |
| `roas` | `totalSalesRev / totalMediaSpend` (return on ad spend) |
| `cpa` | `totalMediaSpend / totalSalesUnits` (cost per acquisition) |
| `chartWithMedia[]` | `monthlyData` joined with `mktgByMonth` on month key |
| Per-month ROAS | `monthRevenue / monthSpend` |
| Per-month CPA | `monthSpend / monthUnits` |

## Endpoints

### `GET /projects/:id/sales/intelligence/promos`

Returns the promo analysis — detected periods, per-period stats, baseline, all charts data.

**Request**
```http
GET /projects/prj_2k9l/sales/intelligence/promos?threshold=0.10 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Query parameters**
- `threshold` — discount threshold for sale detection (default 0.10 = 10%)
- `from` / `to` — date range

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "config": {
    "threshold": 0.10,
    "medianPrice": 39.99,
    "saleThreshold": 35.99
  },
  "summary": {
    "detectedSaleCount": 3,
    "avgUnitLift": 124.0,
    "avgDiscount": 28.5,
    "baseRevPerMonth": 142500,
    "baseUnitsPerMonth": 3820
  },
  "monthlyAnalysis": [
    {
      "month": "2026-01",
      "units": 4250,
      "revenue": 169575,
      "effPrice": 39.90,
      "vsBase": -0.22,
      "isSale": false
    },
    {
      "month": "2026-02",
      "units": 3650,
      "revenue": 145935,
      "effPrice": 39.98,
      "vsBase": -0.02,
      "isSale": false
    },
    {
      "month": "2026-03",
      "units": 8420,
      "revenue": 252600,
      "effPrice": 30.00,
      "vsBase": -24.98,
      "isSale": true
    }
  ],
  "detectedPeriods": [
    {
      "key": "2026-03",
      "name": "Spring Sale",
      "startMonth": "2026-03",
      "endMonth": "2026-03",
      "months": ["2026-03"],
      "avgPrice": 30.00,
      "discount": 24.98,
      "rev": 252600,
      "units": 8420,
      "revPerMonth": 252600,
      "revLift": 77.26,
      "unitLift": 120.42
    },
    {
      "key": "2026-06_2026-07",
      "name": "",
      "startMonth": "2026-06",
      "endMonth": "2026-07",
      "months": ["2026-06", "2026-07"],
      "avgPrice": 27.99,
      "discount": 30.00,
      "rev": 580200,
      "units": 20730,
      "revPerMonth": 290100,
      "revLift": 103.58,
      "unitLift": 171.20
    }
  ]
}
```

### `PATCH /projects/:id/sales/intelligence/promos/:periodKey`

Update a detected period's name.

**Request**
```http
PATCH /projects/prj_2k9l/sales/intelligence/promos/2026-03 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "name": "Spring Awakening Sale"
}
```

**Response — 200 OK**
```json
{
  "periodKey": "2026-03",
  "name": "Spring Awakening Sale",
  "updatedAt": "2026-05-01T21:00:00Z",
  "auditEntryId": "aud_k1l2"
}
```

### `GET /projects/:id/sales/intelligence/media`

Returns the paid-media correlation analysis.

**Request**
```http
GET /projects/prj_2k9l/sales/intelligence/media HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "summary": {
    "totalMediaSpend": 384200,
    "totalSalesRev": 3245890,
    "totalSalesUnits": 87421,
    "roas": 8.45,
    "cpa": 4.40,
    "mediaEntries": 27
  },
  "monthly": [
    {
      "month": "2026-01",
      "mediaSpend": 84200,
      "revenue": 1024850,
      "units": 28420,
      "roas": 12.17,
      "cpa": 2.96,
      "items": ["Steam wishlist boost", "PR push — IGN", "YouTube creator program"]
    },
    {
      "month": "2026-02",
      "mediaSpend": 142500,
      "revenue": 845200,
      "units": 21340,
      "roas": 5.93,
      "cpa": 6.68,
      "items": ["TGS booth", "Influencer campaign batch 2"]
    },
    {
      "month": "2026-03",
      "mediaSpend": 78400,
      "revenue": 252600,
      "units": 8420,
      "roas": 3.22,
      "cpa": 9.31,
      "items": ["Spring Sale paid placement", "Steam featured"]
    }
  ]
}
```

### Permissions Summary (Sales Intelligence)

| Endpoint | Required permission |
|---|---|
| `GET /sales/intelligence/promos` | studio member, project access |
| `PATCH /sales/intelligence/promos/:periodKey` | `editSales` |
| `GET /sales/intelligence/media` | studio member, project access |

---

# Page 12: Developer Portal

A separate, walled-off React component (`DevPortal`, lines 11671–11907). Renders only when the URL has `?portal=<projectId>&dev=<name>`. Reads project data read-only and writes only to `aux.changeLog` and `aux.invoiceQueue`.

This is the most security-sensitive surface in the app — it's how external developers (not studio employees) submit data into the system. The brief calls out three specific gaps that need closing before production: token-scoped authentication (currently URL params only), invitation flow, and email notifications.

## Editable Fields

The portal has three tabs.

### Tab 1: Budget Grid

The developer sees the budget grid scoped to their project. **Cells are editable but never persisted directly** — every edit goes into a `proposedChanges` map (transient state on the client) and is submitted as a batch.

| Field | Type | Storage |
|---|---|---|
| Per-cell proposed value | Number | `proposedChanges[rowId__monthKey]` (transient client state) |
| Change note | Text | `changeNote` (transient; sent with batch submission) |
| Currency view | "usd" / "native" | `currencyView` (UI state) |
| Add row button | Action | Creates a new row directly via `setProject` (this is one place where the prototype lets developers mutate project state directly — see gap analysis below) |

Persistent fields per row addition:

| Field | Type | Storage path |
|---|---|---|
| Section / Subsection | Hardcoded by which "+ Add" button was clicked | `row.sectionId` / `row.subsectionId` |
| Role / Item / Department / Name | Empty initially; filled in via subsequent edits | `row.meta.{role, department, name}` or `{item, category, detail}` |

### Tab 2: Submit Invoice

| Field | Type | Storage path |
|---|---|---|
| File upload | PDF/image | Stored as base64 in `aux.invoiceQueue[].fullFileData` (S3 in production) |
| Submission note | Text | `aux.invoiceQueue[].note` |

### Tab 3: My Submissions

Read-only list of the developer's own submissions. No editable fields.

## Calculated Fields

### Tab 1: Budget Grid

| Field | Formula |
|---|---|
| `pendingCount` | `Object.keys(proposedChanges).length` (drives the "(N)" badge on the Budget tab and the "Submit N Changes" button) |
| Per-cell display value | `cellValue × cellRate(row, mKey)` if `currencyView === "usd"` and row is alt-currency; else native value |
| Per-cell rate (FX cascade) | Same as the main app: `row.rates[mKey] → spotRates[mKey] → globalSpotRate → 1.0` |
| Group total per month | `Σ rowsInGroup.live[m.key]` |
| Section total per month | `Σ rowsInSection.live[m.key]` |
| Row total | `Σ row.live[m.key]` across all months |
| Grand total | `Σ rows.Σ live[m.key]` |
| Multi-currency display | `mcEnabled = project.multiCurrency && project.altCurrency` |

### Tab 3: My Submissions

| Field | Formula |
|---|---|
| `myChanges[]` | `aux.changeLog.filter(b => b.submittedBy === devName)` |
| `myInvoices[]` | `aux.invoiceQueue.filter(q => q.submittedBy === devName)` |
| Per-batch / per-invoice status | Direct from `b.status` / `q.status` |

## Authentication & Authorization Model

Before getting to endpoints, the auth model needs explaining. The Developer Portal must be cleanly separated from the studio's auth — developers are external collaborators, not studio employees.

**Recommended model:**

1. **Producer creates an invitation** for a specific project with a specific scope.
2. The invitation generates a **portal token** (long, opaque, signed JWT or equivalent).
3. The token is delivered to the developer via email link: `https://app.fictions-folio.com/portal?token=<portalToken>`.
4. When the developer hits that URL, the token is validated server-side; if valid, a session cookie is set scoped to **only that project** with **only developer-level permissions**.
5. All portal endpoints check both the session AND the project scope on every request.

The portal token itself is stored on the server with: `tokenId`, `projectId`, `inviteeName`, `inviteeEmail`, `permissions`, `createdBy`, `createdAt`, `expiresAt`, `revokedAt`.

## Endpoints

### Auth: Token Issuance and Validation

#### `POST /studios/:id/portal-invitations`

(Studio-side endpoint, called by producers.) Creates a new portal invitation.

**Request**
```http
POST /studios/std_a8f3/portal-invitations HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "projectId": "prj_2k9l",
  "inviteeName": "Jamie Rivera",
  "inviteeEmail": "jamie@external-studio.com",
  "expiresInDays": 365,
  "permissions": {
    "viewBudget": true,
    "submitBudgetChanges": true,
    "submitInvoices": true,
    "viewMilestones": true
  },
  "note": "Lead developer at external studio collaborating on Project Covenant."
}
```

**Response — 201 Created**
```json
{
  "invitationId": "inv_a1b2",
  "projectId": "prj_2k9l",
  "inviteeName": "Jamie Rivera",
  "inviteeEmail": "jamie@external-studio.com",
  "portalUrl": "https://app.fictions-folio.com/portal?token=eyJhbGciOi…",
  "tokenExpiresAt": "2027-05-01T21:00:00Z",
  "createdAt": "2026-05-01T21:00:00Z",
  "createdBy": "Alex Chen",
  "emailQueued": true,
  "auditEntryId": "aud_m3n4"
}
```

**Notes**
- The `portalUrl` is shown to the producer once and emailed to the developer; the raw token is not shown again after this response.
- An audit entry on the studio + project records the invitation.
- Permissions: `manageUsers` (admin-level).

#### `GET /studios/:id/portal-invitations`

List active invitations for the studio.

**Response — 200 OK**
```json
{
  "invitations": [
    {
      "invitationId": "inv_a1b2",
      "projectId": "prj_2k9l",
      "projectName": "Covenant",
      "inviteeName": "Jamie Rivera",
      "inviteeEmail": "jamie@external-studio.com",
      "permissions": {
        "viewBudget": true,
        "submitBudgetChanges": true,
        "submitInvoices": true,
        "viewMilestones": true
      },
      "createdAt": "2026-05-01T21:00:00Z",
      "createdBy": "Alex Chen",
      "lastUsedAt": "2026-05-02T09:30:00Z",
      "expiresAt": "2027-05-01T21:00:00Z",
      "revokedAt": null
    }
  ]
}
```

#### `DELETE /studios/:id/portal-invitations/:invitationId`

Revoke an invitation. Future requests with that token return `401`.

**Response — 200 OK**
```json
{
  "invitationId": "inv_a1b2",
  "revokedAt": "2026-05-01T21:30:00Z",
  "revokedBy": "Alex Chen",
  "auditEntryId": "aud_o5p6"
}
```

#### `POST /portal/sessions`

Validate a token and start a portal session. This is the developer's entry point.

**Request**
```http
POST /portal/sessions HTTP/1.1
Content-Type: application/json

{
  "token": "eyJhbGciOi…"
}
```

**Response — 200 OK**
```json
{
  "session": {
    "sessionId": "psn_q7r8",
    "projectId": "prj_2k9l",
    "projectName": "Covenant",
    "studioName": "Fictions",
    "inviteeName": "Jamie Rivera",
    "permissions": {
      "viewBudget": true,
      "submitBudgetChanges": true,
      "submitInvoices": true,
      "viewMilestones": true
    },
    "expiresAt": "2026-05-01T22:00:00Z"
  }
}
```

**Notes**
- Server sets an HttpOnly, Secure session cookie scoped to the portal subdomain.
- The session expires shorter than the token (1 hour vs. up to 1 year). The token is the long-lived credential; sessions are short-lived.
- All subsequent portal endpoints use the session cookie, not the token.

**Error responses**
- `401 Unauthorized` — token invalid, expired, or revoked
- `429 Too Many Requests` — too many failed attempts (brute force protection)

### Tab 1: Budget Grid

#### `GET /portal/projects/:id/budget`

Returns the budget grid scoped to the developer's project. This is a **read-only** version of the studio's `GET /projects/:id/grid` — same data shape, but greenlit values may be filtered or hidden depending on portal permissions.

**Request**
```http
GET /portal/projects/prj_2k9l/budget HTTP/1.1
Cookie: portal_session=psn_q7r8
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "projectName": "Covenant",
  "startDate": "2025-08",
  "monthCount": 36,
  "months": [
    { "key": "2025-08", "label": "Aug 25", "year": 2025, "phaseId": "concept" }
  ],
  "currency": {
    "multiCurrency": false,
    "altCurrency": null,
    "globalSpotRate": null,
    "spotRates": {}
  },
  "rows": [
    {
      "id": "row_n3kp",
      "sectionId": "devfees",
      "subsectionId": "employees",
      "meta": { "role": "Lead Programmer", "department": "Engineering", "name": "Pat Singh" },
      "live": { "2025-08": 18000, "2025-09": 18000 }
    }
  ],
  "totals": {
    "live": {
      "byMonth": { "2025-08": 540000, "2025-09": 562000 },
      "grandTotal": 8215000
    }
  },
  "greenlitVersion": 3,
  "permissions": {
    "viewBudget": true,
    "viewGreenlit": false,
    "submitBudgetChanges": true,
    "addRow": true
  }
}
```

**Notes**
- Greenlit values can be hidden (`viewGreenlit: false`) — many developer agreements specify that the studio's contracted budget isn't disclosed to the dev, only the live forecast.
- This endpoint is **always read-only** — no live or greenlit values can be patched via this URL.

#### `POST /portal/projects/:id/budget/changes`

Submit a batch of proposed budget changes.

**Request**
```http
POST /portal/projects/prj_2k9l/budget/changes HTTP/1.1
Cookie: portal_session=psn_q7r8
Content-Type: application/json

{
  "changes": [
    { "rowId": "row_n3kp", "monthKey": "2026-05", "proposedValue": 19500 },
    { "rowId": "row_n3kp", "monthKey": "2026-06", "proposedValue": 19500 },
    { "rowId": "row_q4mr", "monthKey": "2026-05", "proposedValue": 12000 }
  ],
  "note": "Lead programmer rate increase per contract amendment effective May 1. Adding contract animator for 1 month."
}
```

**Response — 201 Created**
```json
{
  "batchId": "bch_s9t0",
  "projectId": "prj_2k9l",
  "submittedBy": "Jamie Rivera",
  "submittedAt": "2026-05-01T21:30:00Z",
  "status": "pending",
  "source": "dev-portal",
  "changes": [
    {
      "rowId": "row_n3kp",
      "rowName": "Lead Programmer — Pat Singh",
      "monthKey": "2026-05",
      "previousValue": 18500,
      "proposedValue": 19500
    },
    {
      "rowId": "row_n3kp",
      "rowName": "Lead Programmer — Pat Singh",
      "monthKey": "2026-06",
      "previousValue": 18500,
      "proposedValue": 19500
    },
    {
      "rowId": "row_q4mr",
      "rowName": "UI Contractor",
      "monthKey": "2026-05",
      "previousValue": 8000,
      "proposedValue": 12000
    }
  ],
  "note": "Lead programmer rate increase…",
  "notificationsSent": ["alex@fictions.com", "sam@fictions.com"]
}
```

**Notes**
- The submission goes into `aux.changeLog[]` with `status: "pending"`, where it's surfaced on the producer's Amendments page for approval.
- This endpoint **does not modify** the live grid — even after submission, the grid still shows the old values until a producer approves.
- The notification list is users with `approveBudgetChanges` for that project.
- Permission required (on token): `submitBudgetChanges`.

#### `POST /portal/projects/:id/budget/rows`

Add a new row from the developer side. Unlike the studio version, this also goes through the approval queue rather than directly mutating the project.

**Request**
```http
POST /portal/projects/prj_2k9l/budget/rows HTTP/1.1
Cookie: portal_session=psn_q7r8
Content-Type: application/json

{
  "sectionId": "devfees",
  "subsectionId": "employees",
  "meta": { "role": "Audio Designer", "department": "Audio", "name": "Devon Lee" },
  "proposedValues": {
    "2026-05": 8500,
    "2026-06": 8500,
    "2026-07": 8500
  },
  "note": "New hire starting May 1 to handle audio for Alpha milestone."
}
```

**Response — 201 Created**
```json
{
  "batchId": "bch_u1v2",
  "projectId": "prj_2k9l",
  "submittedBy": "Jamie Rivera",
  "submittedAt": "2026-05-01T22:00:00Z",
  "status": "pending",
  "source": "dev-portal",
  "type": "row_addition",
  "proposedRow": {
    "tempId": "tmp_w3x4",
    "sectionId": "devfees",
    "subsectionId": "employees",
    "meta": { "role": "Audio Designer", "department": "Audio", "name": "Devon Lee" },
    "proposedValues": { "2026-05": 8500, "2026-06": 8500, "2026-07": 8500 }
  },
  "note": "New hire starting May 1…",
  "notificationsSent": ["alex@fictions.com"]
}
```

**Notes**
- **This is a deliberate divergence from the prototype**, which lets the dev portal directly add rows. The prototype's behavior (line 11705) writes the row to project state immediately. That is a security gap; the production behavior should funnel through the approval queue like cell edits.
- The producer sees the proposed row addition on the Amendments page; approving creates the row, rejecting discards it.

### Tab 2: Submit Invoice

#### `POST /portal/projects/:id/invoices`

Upload an invoice from the dev portal. Same shape as the studio's invoice upload, but the resulting queue entry is tagged `source: "dev-portal"`.

**Request**
```http
POST /portal/projects/prj_2k9l/invoices HTTP/1.1
Cookie: portal_session=psn_q7r8
Content-Type: multipart/form-data

(file as multipart field "file"; optional "note" text field)
```

**Response — 201 Created**
```json
{
  "invoiceId": "inv_y5z6",
  "projectId": "prj_2k9l",
  "fileName": "Animator_May_Invoice.pdf",
  "submittedBy": "Jamie Rivera",
  "submittedAt": "2026-05-01T22:30:00Z",
  "status": "pending",
  "note": "Animator invoice for May milestone deliverables.",
  "extractedData": {
    "vendor": "Devon Lee Audio",
    "invoiceNumber": "DLA-2026-13",
    "invoiceDate": "2026-04-30",
    "currency": "USD",
    "total": 6500,
    "lineItems": [
      { "id": "li_a7b8", "description": "VO direction", "amount": 4500, "include": true, "suggestedCategory": "Audio" }
    ],
    "confidence": "high"
  },
  "notificationsSent": ["alex@fictions.com", "sam@fictions.com"]
}
```

**Notes**
- AI extraction runs server-side using the studio's API key (the developer never sees the key).
- Permission required: `submitInvoices`.
- File size cap: 10 MB; allowed types: PDF, PNG, JPG, WEBP.
- Unlike the studio side, the developer **cannot edit the extracted data** — they can only submit and let the producer review. (This avoids gaming the AI's extraction.)

### Tab 3: My Submissions

#### `GET /portal/projects/:id/submissions`

Returns the developer's own submissions — both budget changes and invoices — across pending, approved, and rejected states.

**Request**
```http
GET /portal/projects/prj_2k9l/submissions HTTP/1.1
Cookie: portal_session=psn_q7r8
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "submittedBy": "Jamie Rivera",
  "budgetChanges": [
    {
      "batchId": "bch_s9t0",
      "submittedAt": "2026-05-01T21:30:00Z",
      "status": "pending",
      "type": "cell_changes",
      "changeCount": 3,
      "totalDelta": 5500,
      "note": "Lead programmer rate increase…",
      "reviewedBy": null,
      "reviewedAt": null
    },
    {
      "batchId": "bch_h7i8",
      "submittedAt": "2026-04-22T15:00:00Z",
      "status": "approved",
      "type": "cell_changes",
      "changeCount": 2,
      "totalDelta": 1500,
      "note": "Contract rate adjustments per April amendment.",
      "reviewedBy": "Alex Chen",
      "reviewedAt": "2026-04-23T10:30:00Z",
      "reviewNote": "Approved per contract."
    },
    {
      "batchId": "bch_j9k0",
      "submittedAt": "2026-04-15T09:00:00Z",
      "status": "rejected",
      "type": "row_addition",
      "proposedRow": { "meta": { "role": "Animation Director" } },
      "note": "Adding animation director.",
      "reviewedBy": "Alex Chen",
      "reviewedAt": "2026-04-15T11:00:00Z",
      "rejectionReason": "Need to discuss in next review meeting before adding new headcount."
    }
  ],
  "invoices": [
    {
      "invoiceId": "inv_y5z6",
      "fileName": "Animator_May_Invoice.pdf",
      "submittedAt": "2026-05-01T22:30:00Z",
      "status": "pending",
      "note": "Animator invoice for May milestone deliverables.",
      "reviewedBy": null,
      "reviewedAt": null
    },
    {
      "invoiceId": "inv_l1m2",
      "fileName": "Audio_April_Invoice.pdf",
      "submittedAt": "2026-04-28T13:00:00Z",
      "status": "approved",
      "reviewedBy": "Sam Park",
      "reviewedAt": "2026-04-29T09:30:00Z",
      "createdLedgerEntryIds": ["led_n3o4"]
    }
  ]
}
```

**Notes**
- Pagination is generally not needed at the per-developer level (their own submission list won't be large), but cursor support is fine to have.
- Rejected submissions show the rejection reason — critical for the dev to understand what to revise.

### Permissions Summary (Developer Portal)

| Endpoint | Required permission (on the token) | Additional gates |
|---|---|---|
| `POST /studios/:id/portal-invitations` | (studio side) `manageUsers` | Studio admin only |
| `GET /studios/:id/portal-invitations` | (studio side) `manageUsers` | — |
| `DELETE /studios/:id/portal-invitations/:id` | (studio side) `manageUsers` | — |
| `POST /portal/sessions` | (no auth) | Token validation |
| `GET /portal/projects/:id/budget` | `viewBudget` | Token must be scoped to this project |
| `POST /portal/projects/:id/budget/changes` | `submitBudgetChanges` | — |
| `POST /portal/projects/:id/budget/rows` | `submitBudgetChanges` AND `addRow` | — |
| `POST /portal/projects/:id/invoices` | `submitInvoices` | File size and type validation |
| `GET /portal/projects/:id/submissions` | `viewBudget` (or any portal access) | Returns only `submittedBy === inviteeName` |

---

# Cross-Cutting Concerns

## 1. Portal isolation

The developer portal must run on a **separate subdomain or path** with its own cookie scope:

- Studio side: `app.fictions-folio.com` — cookie `studio_session`
- Portal side: `portal.fictions-folio.com` — cookie `portal_session`

This prevents any session-bleeding scenario where a developer's portal token could accidentally elevate against the main app.

The brief explicitly notes this: *"the Developer Portal currently renders a standalone component with its own header and tabs. In production, this should be a separate route with proper auth scoping."*

## 2. Rate limiting

| Endpoint family | Limit |
|---|---|
| `POST /portal/sessions` (token validation) | 10/minute per IP, 100/day per token |
| `POST /portal/projects/:id/invoices` (AI calls) | 50/day per portal session |
| `POST /portal/projects/:id/budget/changes` | 100/day per portal session |
| `POST /sales/import/:platform` (CSV uploads) | 20/hour per studio |
| Sales Intelligence GET endpoints | 60/minute per session |

The AI invoice scanning is the most expensive endpoint by far — the rate limit is both a cost guard and a protection against malicious uploads.

## 3. Data scope on the portal side

A portal token can only see:
- The project it's scoped to
- Live budget data (greenlit optionally hidden)
- Its own submissions
- Phase / milestone read-only data

It explicitly **cannot** see:
- Other projects in the studio
- The full ledger (private financial data)
- Sales data (until and unless a future scope flag enables it)
- Other developers' submissions
- The Amendments page (the workflow inside which their proposals are reviewed)
- Audit logs

Every portal endpoint should `403` if asked for data outside this scope, even if the data exists for the project.

## 4. Notifications

Per the brief's Phase 3 requirement, every portal action should produce a notification:

| Event | Recipients | Channel |
|---|---|---|
| Portal invitation created | Invitee | Email with portal URL |
| Budget change submitted | Producers with `approveBudgetChanges` | Email + in-app notification |
| Invoice uploaded | Producers with `approveInvoices` | Email + in-app |
| Budget change approved | Submitter | Email + in-app (visible in My Submissions) |
| Budget change rejected | Submitter | Email + in-app, including rejection reason |
| Invoice approved | Submitter | Email + in-app |
| Invoice rejected | Submitter | Email + in-app with reason |

## 5. Sales data import scaling

Sales CSVs from popular games can be enormous — Steam exports for a single quarter on a major release can be hundreds of thousands of rows. The endpoint design above assumes synchronous processing, but for large files:

- Files over 5 MB should switch to async processing: return `202 Accepted` with a job ID, then poll `GET /jobs/:id` for status.
- Server-side, use streaming CSV parsers (don't load entire file into memory).
- Use bulk insert semantics in the database (Postgres `COPY`, batched INSERTs of 1000 rows).
- Index `salesData` on `(projectId, platform, date)` because every query filters on at least the first two.

---

That covers Sales (Dashboard, per-platform, Intelligence) and the Developer Portal. The remaining unspec'd surfaces are:

- **Notes / Documents / Milestones** — content management, simpler shapes
- **Variance / Cost Reports / Insights / Rollup / Payment Schedule / Top Sheet / Master Budget** — reporting endpoints (mostly GET-only, derived from existing data)
- **History & Audit** — read endpoints for snapshots and audit log

These are the **reporting layer** of the application. They have a common shape: read-only views over data that's edited elsewhere. The "editable fields" sections will be short for most of them; the calculation logic is where the value lies.

A note before diving in: the "Master Budget" page is the same as the "Contracted Budget" page (`view === "master_budget"`). I covered that fully earlier in the conversation — `GET /projects/:id/master-budget` and `PATCH /projects/:id/master-budget`. I'll skip re-spec'ing it and reference back. Top Sheet was also partially covered earlier as `view === "ledger_topsheet"` — I'll add the API spec here.

---

# Page 13: Variance

The 5-column variance report explicitly called for in the brief.

## Editable Fields

**None.** The page is purely read-only. All values are derived from the budget grid, the ledger, and today's date.

## Calculated Fields

| Field | Formula |
|---|---|
| `elapsedMonths` | Count of months in `msLive` whose `key + "-01"` ≤ today |
| `actuals` (per group/section) | `Σ groupTotal/sectionTotal("live", m.key)` for months indexed `0..elapsedMonths−1` |
| `remaining` (per group/section) | `Σ groupTotal/sectionTotal("live", m.key)` for months indexed `elapsedMonths..end` |
| `projected` | `actuals + remaining` |
| `greenlit` (per group/section) | `Σ groupTotal/sectionTotal("greenlit", m.key)` over all months |
| `variance` | `projected − greenlit` |
| `variancePct` | `variance / greenlit × 100` |
| `ledgerActuals` (footnote) | `Σ ledger[].totalBilled` — for cross-reference, not used in the variance math |

**Important business semantic** (worth restating): "Actuals to Date" here means *forecasted live spend for past months*, not what's been invoiced. The footnote acknowledges this distinction by showing the ledger total separately.

## Endpoints

### `GET /projects/:id/variance`

Returns the full variance breakdown ready to render. The frontend doesn't recompute anything.

**Request**
```http
GET /projects/prj_2k9l/variance HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query parameters**
- `asOf` — override "today" with a specific date (useful for historical variance reports). Default: today.

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "asOf": "2026-05-01",
  "timeline": {
    "elapsedMonths": 9,
    "totalMonths": 36,
    "remainingMonths": 27,
    "elapsedPct": 25.0
  },
  "summary": {
    "actualsToDate": 1842500,
    "remainingProjected": 8372500,
    "projectedTotal": 10215000,
    "greenlitTotal": 9750000,
    "variance": 465000,
    "variancePct": 4.77
  },
  "byGroup": [
    {
      "id": "devcosts",
      "name": "Development Costs",
      "color": "#6366f1",
      "actualsToDate": 1248000,
      "remainingProjected": 4992000,
      "projectedTotal": 6240000,
      "greenlitTotal": 6000000,
      "variance": 240000,
      "variancePct": 4.00,
      "sections": [
        {
          "id": "devfees",
          "title": "Developer Fees",
          "actualsToDate": 1102500,
          "remainingProjected": 3997500,
          "projectedTotal": 5100000,
          "greenlitTotal": 4900000,
          "variance": 200000,
          "variancePct": 4.08
        },
        {
          "id": "fictions",
          "title": "Paid by Fictions",
          "actualsToDate": 145500,
          "remainingProjected": 994500,
          "projectedTotal": 1140000,
          "greenlitTotal": 1100000,
          "variance": 40000,
          "variancePct": 3.64
        }
      ]
    },
    {
      "id": "porting",
      "name": "Porting Costs",
      "color": "#f59e0b",
      "actualsToDate": 0,
      "remainingProjected": 580000,
      "projectedTotal": 580000,
      "greenlitTotal": 600000,
      "variance": -20000,
      "variancePct": -3.33,
      "sections": [
        {
          "id": "porting_sec",
          "title": "Porting",
          "actualsToDate": 0,
          "remainingProjected": 580000,
          "projectedTotal": 580000,
          "greenlitTotal": 600000,
          "variance": -20000,
          "variancePct": -3.33
        }
      ]
    },
    {
      "id": "external",
      "name": "External Support Costs",
      "color": "#10b981",
      "actualsToDate": 594500,
      "remainingProjected": 800500,
      "projectedTotal": 1395000,
      "greenlitTotal": 1400000,
      "variance": -5000,
      "variancePct": -0.36,
      "sections": [
        {
          "id": "external_sec",
          "title": "External Support",
          "actualsToDate": 594500,
          "remainingProjected": 800500,
          "projectedTotal": 1395000,
          "greenlitTotal": 1400000,
          "variance": -5000,
          "variancePct": -0.36
        }
      ]
    }
  ],
  "ledgerCrossReference": {
    "ledgerActualsTotal": 1652340,
    "ledgerVsLiveActualsDiff": -190160,
    "note": "Ledger total differs from live actuals because the ledger reflects only invoiced amounts; live actuals include unbilled forecasted spend."
  }
}
```

### Permissions

| Endpoint | Required permission |
|---|---|
| `GET /variance` | studio member, project access |

---

# Page 14: Cost Reports (Insights)

In the prototype, `view === "reports"` auto-redirects to `view === "insights"` (line 667). They're conceptually one page now.

## Editable Fields

**None.** Read-only.

The Export PDF button (top of page) triggers `window.print()` — purely client-side, not an API call.

## Calculated Fields

A dense set of derived analytics. All formulas come from the prototype.

### Budget Health Section

| Field | Formula |
|---|---|
| `elapsedMonths` | Count of `msLive` months ≤ today |
| `spentToDate` | `Σ ledger[].totalBilled` |
| `greenlitToDate` | `Σ grandTotal("greenlit", m.key)` for months `0..elapsedMonths−1` |
| `remainingLive` | `liveAll − spentToDate` |
| `burnRate` | `spentToDate / elapsedMonths` |
| `projectedTotal` | `burnRate × monthCount` |
| `estCompletion` | `ceil(remainingLive / burnRate)` if both > 0, else `remainingMonths` |
| `pctComplete` | `(spentToDate / greenlitAll) × 100` |
| `pctElapsed` | `(elapsedMonths / monthCount) × 100` |
| `varianceToDate` | `spentToDate − greenlitToDate` |
| Health band | `pctComplete > 100` red; `> 80` yellow; else green |

### Phase Spend Report

| Field | Formula |
|---|---|
| Per phase | `{name, color, monthCount: countOfMonthsInPhase, greenlit: Σ grandTotal("greenlit"), live: Σ grandTotal("live"), diff: live − greenlit, pctOfBudget: live / liveAll × 100}` |
| Phase status indicator | Inline progress bar tied to `pctOfBudget` |

### Section Breakdown

| Field | Formula |
|---|---|
| Per section | `{group, section, live, greenlit, diff, pct: (live / greenlit) × 100}` |
| Filter | Only sections where `live > 0 OR greenlit > 0` |

### Top Cost Items

| Field | Formula |
|---|---|
| `rowCosts[]` | Map every row to `{label, dept, live: rowTot(r, "live"), greenlit: rowTot(r, "greenlit")}`, filter `live > 0`, sort descending by `live` |
| Top 10 | Slice of `rowCosts[0..10]` |
| Per-item variance | `live − greenlit` |

### Charts

| Field | Formula |
|---|---|
| `monthlyActuals[]` | Per month: `{name, live, greenlit, elapsed: i < elapsedMonths}` — drives the monthly trend bar chart |
| `burnData[]` | Per month: cumulative `{Actual: cumulative live if elapsed else null, Greenlit: cumulative greenlit, Projected: burnRate × (i+1)}` — drives the cumulative burn line chart |

### Insights-specific (vs. legacy Reports)

The Insights page also shows:

| Field | Formula |
|---|---|
| `groupSpends[]` | Per group: `{name, color, live, gl}` filtered to non-zero |
| `deptSpends{}` | `{dept: Σ row.live[m.key]}` aggregated by `meta.department \|\| meta.category` |
| `phaseSpends[]` | Per phase: `{name, months, live, gl}` filtered to non-zero |
| `overBudgetRows[]` | Top 5 rows where `liveTotal > greenlitTotal && greenlitTotal > 0`, sorted by `diff` |

## Endpoints

### `GET /projects/:id/insights`

Returns everything the Insights / Reports page needs in one round-trip. This endpoint replaces both `view === "reports"` and `view === "insights"` (since they share data).

**Request**
```http
GET /projects/prj_2k9l/insights?asOf=2026-05-01 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "asOf": "2026-05-01",
  "timeline": {
    "elapsedMonths": 9,
    "totalMonths": 36,
    "remainingMonths": 27,
    "pctElapsed": 25.0
  },
  "budgetHealth": {
    "totalBudget": 10215000,
    "greenlitTotal": 9750000,
    "spentToDate": 1652340,
    "greenlitToDate": 2437500,
    "remainingLive": 8562660,
    "burnRate": 183593,
    "projectedTotal": 6609333,
    "estCompletion": 47,
    "pctComplete": 16.95,
    "varianceToDate": -785160,
    "healthBand": "on_track",
    "isOverBudget": false,
    "isOverPacing": false
  },
  "byGroup": [
    {
      "id": "devcosts",
      "name": "Development Costs",
      "color": "#6366f1",
      "live": 6240000,
      "greenlit": 6000000,
      "pctOfTotal": 61.09
    },
    {
      "id": "porting",
      "name": "Porting Costs",
      "color": "#f59e0b",
      "live": 580000,
      "greenlit": 600000,
      "pctOfTotal": 5.68
    },
    {
      "id": "external",
      "name": "External Support Costs",
      "color": "#10b981",
      "live": 1395000,
      "greenlit": 1400000,
      "pctOfTotal": 13.66
    }
  ],
  "byPhase": [
    {
      "id": "concept",
      "name": "Concept",
      "color": "#818cf8",
      "monthCount": 3,
      "greenlit": 540000,
      "live": 562000,
      "diff": 22000,
      "diffPct": 4.07,
      "pctOfBudget": 5.50
    },
    {
      "id": "prototype",
      "name": "Prototype",
      "color": "#6366f1",
      "monthCount": 4,
      "greenlit": 920000,
      "live": 945000,
      "diff": 25000,
      "diffPct": 2.72,
      "pctOfBudget": 9.25
    },
    {
      "id": "alpha",
      "name": "Alpha",
      "color": "#3730a3",
      "monthCount": 8,
      "greenlit": 2840000,
      "live": 3120000,
      "diff": 280000,
      "diffPct": 9.86,
      "pctOfBudget": 30.54
    }
  ],
  "byDepartment": [
    { "name": "Engineering", "spend": 2480000, "color": "#6366f1" },
    { "name": "Art", "spend": 1640000, "color": "#f43f5e" },
    { "name": "Design", "spend": 980000, "color": "#a855f7" },
    { "name": "Audio", "spend": 420000, "color": "#10b981" },
    { "name": "QA", "spend": 380000, "color": "#06b6d4" }
  ],
  "bySection": [
    {
      "groupName": "Development Costs",
      "sectionName": "Developer Fees",
      "live": 5100000,
      "greenlit": 4900000,
      "diff": 200000,
      "pctVsGreenlit": 104.08
    },
    {
      "groupName": "Development Costs",
      "sectionName": "Paid by Fictions",
      "live": 1140000,
      "greenlit": 1100000,
      "diff": 40000,
      "pctVsGreenlit": 103.64
    }
  ],
  "topCostItems": [
    {
      "rowId": "row_n3kp",
      "label": "Lead Programmer",
      "dept": "Engineering",
      "live": 648000,
      "greenlit": 612000,
      "diff": 36000
    },
    {
      "rowId": "row_q4mr",
      "label": "UI Contractor",
      "dept": "Design",
      "live": 320000,
      "greenlit": 280000,
      "diff": 40000
    }
  ],
  "overBudgetRows": [
    {
      "rowId": "row_q4mr",
      "label": "UI Contractor",
      "dept": "Design",
      "live": 320000,
      "greenlit": 280000,
      "diff": 40000,
      "diffPct": 14.29
    }
  ],
  "charts": {
    "monthlyTrend": [
      { "month": "2025-08", "name": "Aug 25", "live": 180000, "greenlit": 180000, "elapsed": true },
      { "month": "2025-09", "name": "Sep 25", "live": 187000, "greenlit": 180000, "elapsed": true }
    ],
    "cumulativeBurn": [
      { "month": "2025-08", "name": "Aug 25", "actual": 180000, "greenlit": 180000, "projected": 183593 },
      { "month": "2025-09", "name": "Sep 25", "actual": 367000, "greenlit": 360000, "projected": 367186 }
    ]
  }
}
```

### Permissions

| Endpoint | Required permission |
|---|---|
| `GET /insights` | studio member, project access |

---

# Page 15: Rollup

Two read-only summary tables: by **calendar year** and by **development phase**.

## Editable Fields

**None.** Pure aggregation view.

## Calculated Fields

### Annual Rollup Table

| Field | Formula |
|---|---|
| `years[]` | `[...new Set(ms.map(m => m.year))]` |
| Subsection × year cell | `Σ rowsInSubsection.Σ live[m.key]` for months where `m.year === yr` |
| Section × year cell | `yearTotal("live", yr, sec.id)` |
| Group × year cell | `yearGroupTotal("live", yr, grp.id)` |
| Section row total | `Σ years.yearTotal("live", yr, sec.id)` |
| Group row total | `Σ years.yearGroupTotal("live", yr, grp.id)` |
| Grand total per year | `yearTotal("live", yr)` |
| Grand total | `liveAll` |

### Phase Rollup Table

| Field | Formula |
|---|---|
| Subsection × phase cell | Filter months where `phaseAssignment[m.key] === pId`, then `Σ rowsInSubsection.Σ live[m.key]` |
| Section × phase cell | `phaseTotal("live", p.id, sec.id)` |
| Group × phase cell | `phaseGroupTotal("live", p.id, grp.id)` |
| Phase total (across all groups) | `phaseTotal("live", p.id)` |

The structure is hierarchical: Group → Section → Subsection. Each level expands when its container has multiple sub-items.

## Endpoints

### `GET /projects/:id/rollup`

Returns both rollup matrices in one response.

**Request**
```http
GET /projects/prj_2k9l/rollup?mode=live HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query parameters**
- `mode` — `live` (default) or `greenlit`. The prototype shows live; the API can serve either.

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "mode": "live",
  "annual": {
    "years": [2025, 2026, 2027, 2028],
    "rows": [
      {
        "type": "group",
        "groupId": "devcosts",
        "name": "Development Costs",
        "color": "#6366f1",
        "values": { "2025": 1248000, "2026": 2496000, "2027": 1872000, "2028": 624000 },
        "total": 6240000,
        "children": [
          {
            "type": "section",
            "sectionId": "devfees",
            "name": "Developer Fees",
            "values": { "2025": 1020000, "2026": 2040000, "2027": 1530000, "2028": 510000 },
            "total": 5100000,
            "children": [
              {
                "type": "subsection",
                "subsectionId": "employees",
                "name": "Employees",
                "values": { "2025": 648000, "2026": 1296000, "2027": 972000, "2028": 324000 },
                "total": 3240000
              },
              {
                "type": "subsection",
                "subsectionId": "contractors_dev",
                "name": "Contractors",
                "values": { "2025": 372000, "2026": 744000, "2027": 558000, "2028": 186000 },
                "total": 1860000
              }
            ]
          },
          {
            "type": "section",
            "sectionId": "fictions",
            "name": "Paid by Fictions",
            "values": { "2025": 228000, "2026": 456000, "2027": 342000, "2028": 114000 },
            "total": 1140000,
            "children": []
          }
        ]
      }
    ],
    "grandTotal": {
      "values": { "2025": 1923000, "2026": 4082500, "2027": 3035500, "2028": 1174000 },
      "total": 10215000
    }
  },
  "phase": {
    "phases": [
      { "id": "concept", "name": "Concept", "color": "#818cf8" },
      { "id": "prototype", "name": "Prototype", "color": "#6366f1" },
      { "id": "alpha", "name": "Alpha", "color": "#3730a3" }
    ],
    "rows": [
      {
        "type": "group",
        "groupId": "devcosts",
        "name": "Development Costs",
        "color": "#6366f1",
        "values": { "concept": 420000, "prototype": 720000, "alpha": 2400000 },
        "total": 6240000,
        "children": [
          {
            "type": "section",
            "sectionId": "devfees",
            "name": "Developer Fees",
            "values": { "concept": 350000, "prototype": 600000, "alpha": 2000000 },
            "total": 5100000,
            "children": [
              {
                "type": "subsection",
                "subsectionId": "employees",
                "name": "Employees",
                "values": { "concept": 220000, "prototype": 380000, "alpha": 1280000 },
                "total": 3240000
              }
            ]
          }
        ]
      }
    ],
    "grandTotal": {
      "values": { "concept": 562000, "prototype": 945000, "alpha": 3120000 },
      "total": 10215000
    }
  }
}
```

### Permissions

| Endpoint | Required permission |
|---|---|
| `GET /rollup` | studio member, project access |

---

# Page 16: Payment Schedule

Milestone-by-milestone payment ledger comparing greenlit vs live.

## Editable Fields

**None.** Read-only view.

## Calculated Fields

| Field | Formula |
|---|---|
| Per milestone | `{milestone: idx + 1, label: m.label, phase: phaseObj(phaseFor(m.key))}` |
| `g` (greenlit) | `effGrandTotal("greenlit", m.key)` (USD-aware via FX cascade) |
| `l` (live) | `effGrandTotal("live", m.key)` |
| `diff` | `l − g` |
| `cumDiff` | Running sum of `diff` from milestone 1 |

## Endpoints

### `GET /projects/:id/payment-schedule`

Returns the milestone-by-milestone schedule.

**Request**
```http
GET /projects/prj_2k9l/payment-schedule HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query parameters**
- `currency` — `usd` (default) or `native`. For multi-currency projects, controls which view is returned.

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "currency": "usd",
  "milestones": [
    {
      "milestone": 1,
      "monthKey": "2025-08",
      "label": "Aug 25",
      "phase": { "id": "concept", "name": "Concept", "color": "#818cf8" },
      "greenlit": 180000,
      "live": 180000,
      "diff": 0,
      "cumulativeDiff": 0
    },
    {
      "milestone": 2,
      "monthKey": "2025-09",
      "label": "Sep 25",
      "phase": { "id": "concept", "name": "Concept", "color": "#818cf8" },
      "greenlit": 180000,
      "live": 187000,
      "diff": 7000,
      "cumulativeDiff": 7000
    },
    {
      "milestone": 9,
      "monthKey": "2026-04",
      "label": "Apr 26",
      "phase": { "id": "prealpha", "name": "Pre-Alpha", "color": "#4338ca" },
      "greenlit": 285000,
      "live": 312000,
      "diff": 27000,
      "cumulativeDiff": 142000
    }
  ],
  "summary": {
    "totalGreenlit": 9750000,
    "totalLive": 10215000,
    "totalDiff": 465000,
    "milestonesOverBudget": 14,
    "milestonesUnderBudget": 8,
    "milestonesOnBudget": 14
  }
}
```

### Permissions

| Endpoint | Required permission |
|---|---|
| `GET /payment-schedule` | studio member, project access |

---

# Page 17: Top Sheet (Ledger Top Sheet)

Master budget vs. ledger actuals, broken down by category group.

## Editable Fields

**None.** Read-only.

The page has a sub-category drill-down (click to expand groups), but those are UI-only states.

## Calculated Fields

### Per Category Group

| Field | Formula |
|---|---|
| `groupEntries[]` | `ledger.filter(e => CAT_TO_GROUP[e.category] === cat.group)` |
| `groupSpent` | `Σ groupEntries.totalBilled` |
| `groupBudgeted` | Special-cased per group: <br>– Marketing: `Σ marketingBudget.categories[].budget` (fallback to `masterBudget.marketing`) <br>– Reserve: `masterBudget.reserve` <br>– Contingency: `masterBudget.contingency` <br>– Otherwise: corresponding `masterBudget.{developmentCosts, portingCosts, externalCosts}` |
| `groupRemaining` | `budgeted − spent` |
| `groupUtilization` | `(spent / budgeted) × 100` |
| Color band | Green ≤ 80%, yellow ≤ 100%, red > 100% |

### Per Sub-Category

| Field | Formula |
|---|---|
| `subEntries` | `groupEntries.filter(e => e.category === subKey \|\| CAT_TO_SUBGROUP[e.category] === subKey)` |
| `subSpent` | `Σ subEntries.totalBilled` |
| `subCount` | `subEntries.length` |
| `subBudgeted` | Marketing only: matching `marketingBudget.categories[].budget`. Otherwise 0. |
| `nestedItems[]` | For non-marketing groups: `cat.items[].sub` items, each with their own spent/count |

### Uncategorized

| Field | Formula |
|---|---|
| `categorized` | `Σ subs.spent` (excluding nested items to avoid double-counting) |
| `uncategorized` | `groupSpent − categorized` |

### Total Row

| Field | Formula |
|---|---|
| `totalBudgeted` | `Σ topSheetData[].budgeted` |
| `totalSpent` | `Σ topSheetData[].spent` |
| `totalRemaining` | `totalBudgeted − totalSpent` |
| `totalUtilization` | `(totalSpent / totalBudgeted) × 100` |

## Endpoints

### `GET /projects/:id/top-sheet`

Returns the full top-sheet breakdown.

**Request**
```http
GET /projects/prj_2k9l/top-sheet HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "summary": {
    "totalBudgeted": 9750000,
    "totalSpent": 1652340,
    "totalRemaining": 8097660,
    "utilizationPct": 16.95
  },
  "groups": [
    {
      "groupName": "Development Costs",
      "budgeted": 6000000,
      "spent": 1248000,
      "remaining": 4752000,
      "utilizationPct": 20.80,
      "entryCount": 84,
      "healthBand": "on_track",
      "subs": [
        {
          "label": "Salaries",
          "budgeted": 0,
          "spent": 820000,
          "count": 42,
          "isNested": false,
          "nestedItems": [
            { "label": "Engineering", "budgeted": 0, "spent": 380000, "count": 18, "parentSub": "Salaries" },
            { "label": "Design", "budgeted": 0, "spent": 220000, "count": 12, "parentSub": "Salaries" },
            { "label": "Art", "budgeted": 0, "spent": 220000, "count": 12, "parentSub": "Salaries" }
          ]
        },
        {
          "label": "Contractors",
          "budgeted": 0,
          "spent": 320000,
          "count": 28,
          "isNested": false,
          "nestedItems": []
        },
        {
          "label": "Operating",
          "budgeted": 0,
          "spent": 108000,
          "count": 14,
          "isNested": false,
          "nestedItems": []
        }
      ],
      "uncategorized": 0
    },
    {
      "groupName": "External Support Costs",
      "budgeted": 1400000,
      "spent": 245340,
      "remaining": 1154660,
      "utilizationPct": 17.52,
      "entryCount": 18,
      "healthBand": "on_track",
      "subs": [
        { "label": "QA", "budgeted": 0, "spent": 142000, "count": 8 },
        { "label": "Localization", "budgeted": 0, "spent": 78000, "count": 6 },
        { "label": "Ratings", "budgeted": 0, "spent": 25340, "count": 4 }
      ],
      "uncategorized": 0
    },
    {
      "groupName": "Marketing Costs",
      "budgeted": 500000,
      "spent": 159000,
      "remaining": 341000,
      "utilizationPct": 31.80,
      "entryCount": 27,
      "healthBand": "on_track",
      "subs": [
        { "label": "Creative Materials", "budgeted": 100000, "spent": 48000, "count": 8 },
        { "label": "Promotional Materials", "budgeted": 75000, "spent": 22000, "count": 6 },
        { "label": "Media Spend", "budgeted": 200000, "spent": 84200, "count": 9 },
        { "label": "Marketing T&E", "budgeted": 50000, "spent": 4800, "count": 4 },
        { "label": "PR Expenses", "budgeted": 25000, "spent": 0, "count": 0 },
        { "label": "Events & Showcases", "budgeted": 30000, "spent": 0, "count": 0 },
        { "label": "Other Marketing", "budgeted": 20000, "spent": 0, "count": 0 }
      ],
      "uncategorized": 0
    },
    {
      "groupName": "Porting Costs",
      "budgeted": 600000,
      "spent": 0,
      "remaining": 600000,
      "utilizationPct": 0,
      "entryCount": 0,
      "healthBand": "on_track",
      "subs": [],
      "uncategorized": 0
    },
    {
      "groupName": "Reserve",
      "budgeted": 250000,
      "spent": 0,
      "remaining": 250000,
      "utilizationPct": 0,
      "entryCount": 0,
      "healthBand": "on_track",
      "subs": [],
      "uncategorized": 0
    },
    {
      "groupName": "Contingency",
      "budgeted": 1000000,
      "spent": 0,
      "remaining": 1000000,
      "utilizationPct": 0,
      "entryCount": 0,
      "healthBand": "on_track",
      "subs": [],
      "uncategorized": 0
    }
  ]
}
```

### Permissions

| Endpoint | Required permission |
|---|---|
| `GET /top-sheet` | studio member, project access |

---

# Page 18: Master Budget (Contracted Budget)

This is `view === "master_budget"`. Already covered earlier with `GET /projects/:id/master-budget` and `PATCH /projects/:id/master-budget`. To recap:

- **Editable fields:** six numeric inputs — `developmentCosts`, `portingCosts`, `externalCosts`, `reserve`, `contingency`, `marketing`.
- **Calculated fields:** Game Budget (sum of first three), Total Project Costs (sum of all six), per-line grid total (from budget grid for the linked groups), per-line variance, percentage of total.
- **Endpoints:** `GET /projects/:id/master-budget`, `PATCH /projects/:id/master-budget`.

See the earlier turn for full request/response samples.

---

# Cross-Cutting Concerns for the Reporting Layer

## 1. Caching strategy

These are read-heavy endpoints with complex aggregation. Cache aggressively:

| Endpoint | Cache TTL | Invalidation |
|---|---|---|
| `GET /variance` | 5 minutes | On any cell write, ledger write, or amendment approval |
| `GET /insights` | 5 minutes | Same |
| `GET /rollup` | 10 minutes | On any cell write or phase change |
| `GET /payment-schedule` | 10 minutes | On any cell write or phase change |
| `GET /top-sheet` | 5 minutes | On any ledger write or master budget update |
| `GET /master-budget` | 5 minutes | On master budget update or any cell write (grid totals included) |

Use ETags so the frontend can do conditional requests:

```
GET /projects/prj_2k9l/insights HTTP/1.1
If-None-Match: "abc123"
```

Server returns `304 Not Modified` if unchanged.

## 2. The `asOf` parameter

Several endpoints accept `asOf=YYYY-MM-DD` for historical reporting (e.g. "What did our variance look like at the end of Q1?"). The semantics:

- "Elapsed months" is computed against `asOf` instead of today
- Ledger entries are filtered to `invoiceDate <= asOf`
- Live and greenlit values are taken at the **current state** of those grids (not snapshotted to that date — for true point-in-time, the request would have to specify a `snapshotId`)

This distinction matters: `asOf` shifts the "as of" *point of view*, but the underlying budget data is current. For full historical accuracy, pair `asOf` with a `snapshotId`:

```
GET /projects/prj_2k9l/insights?asOf=2026-01-31&snapshotId=snap_q3close
```

## 3. Currency handling

Every reporting endpoint should respect the project's `multiCurrency` flag and the FX cascade:

- USD-only projects: all values in USD, no special handling.
- Multi-currency projects: default response is USD-converted. Add `?currency=native` to get values in the alt currency for alt-currency rows.

The cascade is already applied server-side via `effGrandTotal` etc. — the API consumer never needs to know about per-cell rates unless they're editing.

## 4. PDF export

The brief mentions "PDF export generation server-side (currently clipboard/copy workarounds due to sandbox limits)" as a Phase 4 deliverable. Each reporting endpoint should have a PDF variant:

```
GET /projects/prj_2k9l/insights/export?format=pdf
GET /projects/prj_2k9l/variance/export?format=pdf
GET /projects/prj_2k9l/top-sheet/export?format=pdf
GET /projects/prj_2k9l/payment-schedule/export?format=pdf
GET /projects/prj_2k9l/rollup/export?format=pdf
```

Returns `application/pdf` binary stream. The PDF is generated server-side using a templating engine (the prototype's `window.print()` is just a UX hack to dodge sandbox limits).

For programmatic exports:

```
GET /projects/prj_2k9l/insights/export?format=xlsx
```

Returns an Excel workbook with one sheet per major report section.

## 5. Permissions are uniform

All reporting endpoints just need `studio member + project access`. There is no role-based filtering on the report data itself — if a user can see the project at all, they can see its variance/insights/rollups.

The exception is the **Developer Portal** — developers should *not* have access to these reporting endpoints. The portal exposes only its own narrow set of endpoints (covered in the previous turn). If a developer somehow hits `/projects/:id/insights`, the portal session middleware should return `403`.

## 6. The shared aggregation engine

If you look across the responses for Variance, Insights, Rollup, Payment Schedule, and Top Sheet, you'll notice they all reduce to the same primitives:

- `Σ row[mode][m.key]` filtered by various dimensions (section, group, phase, year, month, ledger category)
- Counts of months by predicate
- FX-cascade USD conversions

**Recommendation:** build one server-side aggregation engine — a single function that takes `(projectId, dimensions, filters, mode)` and returns a pivoted result. Every reporting endpoint becomes a thin caller of that engine. This avoids the kind of math drift the prototype has (where the Insights page and the Reports page have nearly-identical-but-not-quite formulas duplicated inline).

---

That covers the reporting layer. The remaining surfaces are:

- **Notes / Documents / Milestones / History & Audit** — content and metadata management
- **Studio-level views** (Insights, Sales, Social, Admin) beyond the Portfolio I covered earlier
- **Settings / User Management** modals (deeper than what `PATCH /studios/:id` covers)

These four pages share a common shape: each is a content/metadata management surface that wraps a list-shaped data structure. They're simpler than the financial pages but they collectively account for a lot of the day-to-day project memory.

---

# Page 19: Notes (Creative & Meeting)

Two views (`view === "notes_creative"` and `view === "notes_meeting"`) that share one component branched by a `noteType` flag (line 8205). They have identical UI; meeting notes additionally support transcript upload.

## Editable Fields

### Per Note

| Field | Type | Default | Storage path |
|---|---|---|---|
| Title | Text | empty | `notesData[type][note].title` |
| Date | Date (YYYY-MM-DD) | today | `notesData[type][note].date` |
| Content | Multiline text (Markdown-friendly) | empty | `notesData[type][note].content` |
| Tags | Array of text | `[]` | `notesData[type][note].tags` |

### Page-Level Actions

| Action | Effect |
|---|---|
| + New Note | Adds a blank note with today's date, opens it in the editor |
| Delete (per note) | Confirmation prompt, then removes from `notesData[type]` |
| Open in editor | Selects a note for full-pane editing (master/detail layout) |
| Upload Transcript (Meeting only) | File upload (`.txt`, `.vtt`, `.srt`); calls AI to summarize/extract action items; pre-fills a new note with the extracted content |

### Transient State

| Field | Storage |
|---|---|
| Currently selected note | `noteEditing` (note ID) |
| Transcript processing flag | `transcriptProcessing` (boolean, UI-only) |

## Calculated Fields

| Field | Formula |
|---|---|
| `sorted[]` | `[...notes].sort((a, b) => (b.date \|\| "").localeCompare(a.date \|\| ""))` — newest first |
| Note count badge | `notes.length` |
| Per-note preview | `n.content?.slice(0, 60)` (used on the Home page's "Recent Notes" card) |
| `recentNotes[]` (for Home page) | `[...creative, ...meeting].sort by date.slice(0, 5)` |
| Editing state | Computed from URL/state — note panel shows when `noteEditing !== null` |

The page does no derived math — notes are a leaf data structure.

## Endpoints

### `GET /projects/:id/notes`

Returns all notes (both creative and meeting in one call) so the Notes pages and the Home page's "Recent Notes" card can both populate.

**Request**
```http
GET /projects/prj_2k9l/notes?type=meeting&limit=50 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query parameters**
- `type` — `creative` / `meeting` / `all` (default `all`)
- `limit` — page size (default 100)
- `cursor` — pagination cursor
- `q` — free-text search across `title` and `content`
- `tag` — filter by tag

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "summary": {
    "creativeCount": 14,
    "meetingCount": 32,
    "totalCount": 46
  },
  "notes": [
    {
      "id": "nte_a1b2",
      "type": "meeting",
      "date": "2026-04-30",
      "title": "Weekly producer sync — April 30",
      "content": "## Agenda\n- Alpha milestone status\n- Audio integration blockers\n- Marketing asset review\n\n## Decisions\n- Slip Alpha by 2 weeks; no scope changes\n- Approve external audio contractor for May–July\n\n## Action items\n- [ ] Pat: send updated alpha schedule\n- [ ] Sam: confirm audio contractor budget delta",
      "tags": ["weekly-sync", "alpha", "audio"],
      "createdAt": "2026-04-30T15:30:00Z",
      "createdBy": "Alex Chen",
      "updatedAt": "2026-04-30T16:15:00Z"
    },
    {
      "id": "nte_c3d4",
      "type": "creative",
      "date": "2026-04-22",
      "title": "Concept brief: Final boss arena",
      "content": "Visual reference: think arena gladiator combat with a cathedral aesthetic…",
      "tags": ["concept", "art-direction", "boss-design"],
      "createdAt": "2026-04-22T11:00:00Z",
      "createdBy": "Sam Park",
      "updatedAt": "2026-04-22T11:00:00Z"
    }
  ],
  "pagination": {
    "total": 32,
    "returned": 32,
    "nextCursor": null
  }
}
```

### `POST /projects/:id/notes`

Create a new note.

**Request**
```http
POST /projects/prj_2k9l/notes HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "type": "meeting",
  "title": "",
  "date": "2026-05-01",
  "content": "",
  "tags": []
}
```

**Response — 201 Created**
```json
{
  "id": "nte_e5f6",
  "type": "meeting",
  "date": "2026-05-01",
  "title": "",
  "content": "",
  "tags": [],
  "createdAt": "2026-05-01T22:00:00Z",
  "createdBy": "Alex Chen",
  "updatedAt": "2026-05-01T22:00:00Z",
  "auditEntryId": "aud_g7h8"
}
```

### `PATCH /projects/:id/notes/:noteId`

Update one or more fields of a note. High-frequency endpoint (the editor saves on blur or after debounce).

**Request**
```http
PATCH /projects/prj_2k9l/notes/nte_e5f6 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "title": "Audio review with Devon",
  "content": "## Discussion\n- VO direction approach for Q2 sessions\n- Mastering pipeline for trailer cut\n\n## Decisions\n- Approve $4500 for VO direction\n- Studio time billed separately at $1700",
  "tags": ["audio", "review", "may-milestone"]
}
```

**Response — 200 OK**
```json
{
  "id": "nte_e5f6",
  "type": "meeting",
  "date": "2026-05-01",
  "title": "Audio review with Devon",
  "content": "## Discussion\n- VO direction approach…",
  "tags": ["audio", "review", "may-milestone"],
  "updatedAt": "2026-05-01T22:15:00Z",
  "auditEntryId": "aud_i9j0"
}
```

**Notes**
- Server-side debouncing: collapse rapid PATCHes to one note within a 5-second window into a single audit entry.
- Tags are full-replacement, not partial-merge — sending `tags: []` clears all tags.

### `DELETE /projects/:id/notes/:noteId`

Delete a note.

**Request**
```http
DELETE /projects/prj_2k9l/notes/nte_e5f6 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "deletedNoteId": "nte_e5f6",
  "auditEntryId": "aud_k1l2"
}
```

**Notes**
- Hard delete (no soft-delete in the prototype). For production, consider 30-day soft-delete with a `restoreUrl` in the response.

### `POST /projects/:id/notes/transcript`

Upload a meeting transcript file; AI parses and returns a summarized note (not yet persisted — user reviews and confirms).

**Request**
```http
POST /projects/prj_2k9l/notes/transcript HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: multipart/form-data

(file as multipart field "file"; optional "meetingDate" text field)
```

**Response — 200 OK**
```json
{
  "transcriptId": "trn_m3n4",
  "fileName": "weekly-sync-april-30.vtt",
  "fileSize": 142800,
  "extractedNote": {
    "title": "Weekly producer sync — April 30",
    "date": "2026-04-30",
    "content": "## Attendees\n- Alex Chen, Sam Park, Pat Singh, Jamie Rivera\n\n## Agenda\n- Alpha milestone status\n- Audio integration blockers\n- Marketing asset review\n\n## Decisions\n- Slip Alpha by 2 weeks; no scope changes\n- Approve external audio contractor for May–July\n\n## Action items\n- [ ] Pat: send updated alpha schedule\n- [ ] Sam: confirm audio contractor budget delta\n- [ ] Jamie: prepare audio contractor onboarding",
    "tags": ["weekly-sync", "alpha", "audio"]
  },
  "confidence": "high",
  "transcriptUrl": "/projects/prj_2k9l/notes/transcript/trn_m3n4/raw"
}
```

**Notes**
- The file is stored server-side (S3) and accessible via `transcriptUrl`.
- AI extraction is synchronous for files under ~5 MB; larger files return `202 Accepted` with a polling URL.
- Permission: `editNotes`.
- Allowed types: `.txt`, `.vtt`, `.srt`. The prototype's parser handles all three formats.
- Rate-limited per project (suggest 20/day) because of AI cost.

### `POST /projects/:id/notes/transcript/:transcriptId/commit`

Commit the AI-extracted note (after user review/edit).

**Request**
```http
POST /projects/prj_2k9l/notes/transcript/trn_m3n4/commit HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "title": "Weekly producer sync — April 30",
  "date": "2026-04-30",
  "content": "## Attendees\n- Alex Chen…",
  "tags": ["weekly-sync", "alpha"],
  "attachTranscript": true
}
```

**Response — 201 Created**
```json
{
  "id": "nte_o5p6",
  "type": "meeting",
  "date": "2026-04-30",
  "title": "Weekly producer sync — April 30",
  "content": "## Attendees…",
  "tags": ["weekly-sync", "alpha"],
  "transcriptUrl": "/projects/prj_2k9l/notes/nte_o5p6/transcript",
  "createdAt": "2026-05-01T22:30:00Z",
  "createdBy": "Alex Chen",
  "auditEntryId": "aud_q7r8"
}
```

**Notes**
- Setting `attachTranscript: true` keeps the original transcript file linked to the note (downloadable via `transcriptUrl`).

### Permissions Summary (Notes)

| Endpoint | Required permission |
|---|---|
| `GET /notes` | studio member, project access |
| `POST /notes` | `editNotes` |
| `PATCH /notes/:id` | `editNotes` |
| `DELETE /notes/:id` | `editNotes` |
| `POST /notes/transcript` | `editNotes` (rate-limited) |
| `POST /notes/transcript/:id/commit` | `editNotes` |

---

# Page 20: Documents

Project file library. The AI assistant can reference documents when "Docs On" is toggled.

## Editable Fields

| Field | Type | Storage path |
|---|---|---|
| File upload | PDF / image / text / Word | `projectDocs[].data` (base64 in prototype, S3 in production) |

### Page-Level Actions

| Action | Effect |
|---|---|
| Upload file | Stores doc; checks 4 MB cap (client-side); writes to `projectDocs[]` |
| Download (per doc) | Triggers browser download from base64 data URL |
| Delete (per doc) | Removes from `projectDocs[]` |

There are **no editable metadata fields** on uploaded documents in the prototype — no descriptions, tags, or categories. The only writable property is the file itself plus its auto-captured upload metadata.

## Calculated Fields

| Field | Formula |
|---|---|
| `formatSize(bytes)` | `bytes > 1MB ? "{X.X} MB" : "{X} KB"` |
| `typeIcon(type)` | Mapping: `pdf → 📄`, `image → 🖼️`, `text → 📝`, `word → 📃`, else `📎` |
| File count | `projectDocs.length` |
| Total storage used | `Σ projectDocs[].size` |

## Endpoints

### `GET /projects/:id/documents`

List project documents.

**Request**
```http
GET /projects/prj_2k9l/documents HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query parameters**
- `type` — filter by MIME type prefix (`pdf`, `image`, etc.)
- `q` — free-text search on filename
- `limit`, `cursor` — pagination

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "summary": {
    "documentCount": 8,
    "totalBytes": 12480200,
    "totalSizeFormatted": "11.9 MB"
  },
  "documents": [
    {
      "id": "doc_a1b2",
      "name": "Publishing_Agreement_v3_signed.pdf",
      "type": "application/pdf",
      "size": 2480200,
      "sizeFormatted": "2.4 MB",
      "downloadUrl": "/projects/prj_2k9l/documents/doc_a1b2/file",
      "uploadedAt": "2026-02-14T15:00:00Z",
      "uploadedBy": "Alex Chen"
    },
    {
      "id": "doc_c3d4",
      "name": "Concept_Art_Pack_April.zip",
      "type": "application/zip",
      "size": 8240000,
      "sizeFormatted": "7.9 MB",
      "downloadUrl": "/projects/prj_2k9l/documents/doc_c3d4/file",
      "uploadedAt": "2026-04-15T10:00:00Z",
      "uploadedBy": "Sam Park"
    },
    {
      "id": "doc_e5f6",
      "name": "Greenlight_Deck_v3.pdf",
      "type": "application/pdf",
      "size": 1760000,
      "sizeFormatted": "1.7 MB",
      "downloadUrl": "/projects/prj_2k9l/documents/doc_e5f6/file",
      "uploadedAt": "2026-02-10T11:30:00Z",
      "uploadedBy": "Alex Chen"
    }
  ],
  "pagination": {
    "total": 8,
    "returned": 8,
    "nextCursor": null
  }
}
```

### `POST /projects/:id/documents`

Upload a new document.

**Request**
```http
POST /projects/prj_2k9l/documents HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: multipart/form-data

(file as multipart field "file")
```

**Response — 201 Created**
```json
{
  "id": "doc_g7h8",
  "name": "Publishing_Agreement_v4_redlines.pdf",
  "type": "application/pdf",
  "size": 1240000,
  "sizeFormatted": "1.2 MB",
  "downloadUrl": "/projects/prj_2k9l/documents/doc_g7h8/file",
  "uploadedAt": "2026-05-01T22:45:00Z",
  "uploadedBy": "Alex Chen",
  "auditEntryId": "aud_i9j0"
}
```

**Notes for the backend**
- File size cap: prototype uses 4 MB (client-side). Production should raise this to **at least 25 MB** to accommodate concept art packs, signed PDFs with attachments, etc. Reject larger with `413 Payload Too Large`.
- Allowed types: PDF, PNG, JPG, WEBP, GIF, TXT, MD, DOC, DOCX, XLS, XLSX, ZIP. Server-side MIME sniffing should verify (don't trust the client's `Content-Type`).
- Files stored in S3 with key `projects/<projectId>/documents/<docId>/<filename>`.
- Permission: `editNotes` (the prototype gates documents with the same permission as notes).

### `GET /projects/:id/documents/:docId/file`

Download a document. Returns the raw file with appropriate headers.

**Request**
```http
GET /projects/prj_2k9l/documents/doc_g7h8/file HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK** — binary stream with `Content-Type: application/pdf` and `Content-Disposition: attachment; filename="Publishing_Agreement_v4_redlines.pdf"`.

### `DELETE /projects/:id/documents/:docId`

Delete a document.

**Request**
```http
DELETE /projects/prj_2k9l/documents/doc_g7h8 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "deletedDocumentId": "doc_g7h8",
  "auditEntryId": "aud_k1l2"
}
```

**Notes**
- The S3 file is deleted along with the database record.
- Consider keeping a 30-day soft-delete window for accidental deletions.

### `POST /projects/:id/documents/:docId/extract-text`

Extract text content from a document for AI context. Used when the user toggles "Docs On" in the AI chat — the backend pre-extracts text rather than sending raw binary to the AI on every chat message.

**Request**
```http
POST /projects/prj_2k9l/documents/doc_a1b2/extract-text HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "documentId": "doc_a1b2",
  "extractedAt": "2026-05-01T23:00:00Z",
  "characterCount": 84200,
  "tokenCount": 21050,
  "preview": "PUBLISHING AGREEMENT — between Fictions ('Publisher') and External Studio Ltd ('Developer')…"
}
```

**Notes**
- Server caches the extracted text. Subsequent AI queries with this doc in context use the cached extraction.
- For very large docs, extraction may chunk and the AI uses retrieval instead of full text.

### Permissions Summary (Documents)

| Endpoint | Required permission |
|---|---|
| `GET /documents` | studio member, project access |
| `POST /documents` | `editNotes` (file size + type validated) |
| `GET /documents/:id/file` | studio member, project access |
| `DELETE /documents/:id` | `editNotes` |
| `POST /documents/:id/extract-text` | studio member, project access |

---

# Page 21: Milestones

One row per month of the live timeline (months ↔ milestones are 1:1 in this app). Each milestone has free-form notes about the build, plus auto-derived budget metrics.

## Editable Fields

### Per Milestone

| Field | Type | Default | Storage path |
|---|---|---|---|
| Build Information | Multiline text | empty | `milestoneData[monthKey].buildInfo` |
| Milestone Notes | Multiline text | empty | `milestoneData[monthKey].notes` |
| Dropbox Folder URL | URL | empty | `milestoneData[monthKey].dropboxFolder` |

### Transient State

| Field | Storage |
|---|---|
| Expanded panel | `milestoneData[monthKey]._expanded` (UI-only; per-row expand/collapse) |

## Calculated Fields

| Field | Formula |
|---|---|
| Milestone number | `idx + 1` (1-based) |
| Phase | `project.phaseAssignment[m.key]` → `project.phases.find(id).name` and `.color` |
| `monthBurn` | `effGrandTotal("live", m.key)` — current month's spend |
| `ltdSpend` | `Σ effGrandTotal("live", mm.key)` for months `0..idx` (inclusive) — life-to-date spend |
| `totalBudget` | `Σ effGrandTotal("live", m.key)` over `msLive` |
| `remainingBudget` | `totalBudget − ltdSpend` |
| Progress bar percentage | `min(100, ltdSpend / totalBudget × 100)` |
| Pct of total | `(ltdSpend / totalBudget × 100).toFixed(1)` |

The page is purely a budget rollup with editable note fields layered on top. It doesn't have its own concept of "milestone delivery date" or "milestone payment amount" — those concepts live in the Payment Schedule.

## Endpoints

### `GET /projects/:id/milestones`

Returns all milestones with their metadata and computed budget metrics.

**Request**
```http
GET /projects/prj_2k9l/milestones HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "summary": {
    "totalMilestones": 36,
    "totalBudget": 10215000,
    "elapsedMilestones": 9,
    "milestonesWithNotes": 7
  },
  "milestones": [
    {
      "milestoneNumber": 1,
      "monthKey": "2025-08",
      "label": "Aug 25",
      "phase": {
        "id": "concept",
        "name": "Concept",
        "color": "#818cf8"
      },
      "monthBurn": 180000,
      "ltdSpend": 180000,
      "ltdSpendPct": 1.76,
      "remainingBudget": 10035000,
      "isElapsed": true,
      "buildInfo": "v0.1.0 — kickoff build, story bible draft 1",
      "notes": "Project kickoff. Team onboarded; Confluence space created.",
      "dropboxFolder": "https://dropbox.com/scl/fo/abc123/concept-month-1",
      "updatedAt": "2025-09-02T10:00:00Z"
    },
    {
      "milestoneNumber": 9,
      "monthKey": "2026-04",
      "label": "Apr 26",
      "phase": {
        "id": "prealpha",
        "name": "Pre-Alpha",
        "color": "#4338ca"
      },
      "monthBurn": 312000,
      "ltdSpend": 1842500,
      "ltdSpendPct": 18.04,
      "remainingBudget": 8372500,
      "isElapsed": true,
      "buildInfo": "Pre-Alpha build 0.7.3 — all systems online; balancing pass in progress",
      "notes": "Audio integration delayed by 2 weeks. Marketing asset pipeline up and running.",
      "dropboxFolder": "https://dropbox.com/scl/fo/xyz789/prealpha-april",
      "updatedAt": "2026-04-30T18:00:00Z"
    },
    {
      "milestoneNumber": 10,
      "monthKey": "2026-05",
      "label": "May 26",
      "phase": {
        "id": "prealpha",
        "name": "Pre-Alpha",
        "color": "#4338ca"
      },
      "monthBurn": 285000,
      "ltdSpend": 2127500,
      "ltdSpendPct": 20.83,
      "remainingBudget": 8087500,
      "isElapsed": false,
      "buildInfo": "",
      "notes": "",
      "dropboxFolder": "",
      "updatedAt": null
    }
  ]
}
```

### `PATCH /projects/:id/milestones/:monthKey`

Update milestone metadata. The `monthKey` path parameter is the month's YYYY-MM identifier — milestones don't have separate IDs, they're keyed by month.

**Request**
```http
PATCH /projects/prj_2k9l/milestones/2026-05 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "buildInfo": "Pre-Alpha build 0.8.0 target — combat polish + audio integration",
  "notes": "Critical milestone — first showable to publishing partner.",
  "dropboxFolder": "https://dropbox.com/scl/fo/m5n6/prealpha-may"
}
```

**Response — 200 OK**
```json
{
  "monthKey": "2026-05",
  "milestoneNumber": 10,
  "buildInfo": "Pre-Alpha build 0.8.0 target — combat polish + audio integration",
  "notes": "Critical milestone — first showable to publishing partner.",
  "dropboxFolder": "https://dropbox.com/scl/fo/m5n6/prealpha-may",
  "updatedAt": "2026-05-01T23:15:00Z",
  "updatedBy": "Alex Chen",
  "auditEntryId": "aud_m3n4"
}
```

**Notes**
- All three fields are optional in the body; partial-merge.
- The endpoint creates the `milestoneData[monthKey]` entry if it doesn't exist yet.

### `DELETE /projects/:id/milestones/:monthKey`

Clear all metadata for a milestone (keeps the milestone itself — that's just a month). Sets all fields to empty.

**Request**
```http
DELETE /projects/prj_2k9l/milestones/2026-05 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "monthKey": "2026-05",
  "cleared": true,
  "auditEntryId": "aud_o5p6"
}
```

### Permissions Summary (Milestones)

| Endpoint | Required permission |
|---|---|
| `GET /milestones` | studio member, project access |
| `PATCH /milestones/:monthKey` | `editNotes` OR `editBudget` (the prototype isn't strict here) |
| `DELETE /milestones/:monthKey` | `editNotes` |

---

# Page 22: History & Audit

Two stacked features behind one tab: **Snapshots** (point-in-time backups with restore) and **Audit Log** (chronological record of every action).

## Editable Fields

The page itself has no editable user-input fields. Snapshots are auto-created (or manually created via a button), and audit entries are auto-written by the system. The only "input" is:

| Field | Type | Storage |
|---|---|---|
| Audit log filter | Text | `auditFilter` (UI state, transient) |
| Currently expanded audit row | ID | `expandedAudit` (UI state, transient) |

### Page-Level Actions

| Action | Effect |
|---|---|
| Save Snapshot Now | Creates a manual snapshot tagged "Manual snapshot" |
| Restore (per snapshot) | Confirms, saves a "Pre-restore backup" snapshot, then restores project state from the chosen snapshot |
| Tab toggle | Snapshots / Audit Log |
| Filter audit | Free-text filter across action, detail, and user fields |

## Calculated Fields

### Snapshots Tab

| Field | Formula |
|---|---|
| `history[]` | `[...changeHistory].reverse()` (newest first) |
| Per-snapshot diff vs current | `compareSnap(snap)` returns `{rowDiff, liveDiff, ledgerDiff}` between snapshot and current project |
| `rowDiff` | `currentRowCount − snapshotRowCount` |
| `liveDiff` | `currentLiveTotal − snapshotLiveTotal` (sum across all rows × all months) |
| `ledgerDiff` | `currentLedgerCount − snapshotLedgerCount` |
| "LATEST" badge | First entry only (`i === 0`) |
| "Restore" pill | Color-coded based on `entry.action` ("Restored to:" or "Pre-restore backup") |

### Audit Log Tab

| Field | Formula |
|---|---|
| `logs[]` | `[...auditLog].reverse()` (newest first) |
| `filtered[]` | If `auditFilter` non-empty: `logs.filter(l => action.includes(filter) \|\| detail.includes(filter) \|\| user.includes(filter))` |
| Action chip color | Lookup map: "Edit cell" → dim, "Add row" → green, "Delete row" → red, "Save greenlight" → blue, "Add ledger entry" → purple, "Amendment" → yellow |
| User initial badge | `l.user?.[0]?.toUpperCase()` |
| Display cap | `filtered.slice(0, 100)` — only first 100 rendered |
| `hasMore` indicator | If `filtered.length > 100` |

## Endpoints

### `GET /projects/:id/history/snapshots`

List snapshots.

**Request**
```http
GET /projects/prj_2k9l/history/snapshots?limit=20 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query parameters**
- `limit` — page size (default 50; the prototype doesn't cap, but production should)
- `cursor` — pagination cursor

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "summary": {
    "totalSnapshots": 24,
    "oldestAt": "2025-08-12T14:22:00Z",
    "newestAt": "2026-05-01T17:00:00Z"
  },
  "snapshots": [
    {
      "id": "snap_a1b2",
      "action": "Manual snapshot",
      "date": "2026-05-01T17:00:00Z",
      "user": "Alex Chen",
      "userRole": "admin",
      "isLatest": true,
      "isRestore": false,
      "stats": {
        "rowCount": 87,
        "liveTotal": 10215000,
        "ledgerEntryCount": 142,
        "greenlitVersion": 3
      },
      "diffVsCurrent": {
        "rowDiff": 0,
        "liveDiff": 0,
        "ledgerDiff": 0
      }
    },
    {
      "id": "snap_c3d4",
      "action": "Save greenlight v3",
      "date": "2026-02-14T15:00:00Z",
      "user": "Sam Park",
      "userRole": "finance",
      "isLatest": false,
      "isRestore": false,
      "stats": {
        "rowCount": 84,
        "liveTotal": 9450000,
        "ledgerEntryCount": 98,
        "greenlitVersion": 2
      },
      "diffVsCurrent": {
        "rowDiff": 3,
        "liveDiff": 765000,
        "ledgerDiff": 44
      }
    },
    {
      "id": "snap_e5f6",
      "action": "Pre-restore backup",
      "date": "2026-01-22T11:00:00Z",
      "user": "Alex Chen",
      "userRole": "admin",
      "isLatest": false,
      "isRestore": true,
      "stats": {
        "rowCount": 78,
        "liveTotal": 8920000,
        "ledgerEntryCount": 78,
        "greenlitVersion": 2
      },
      "diffVsCurrent": {
        "rowDiff": 9,
        "liveDiff": 1295000,
        "ledgerDiff": 64
      }
    }
  ]
}
```

### `POST /projects/:id/history/snapshots`

Create a manual snapshot.

**Request**
```http
POST /projects/prj_2k9l/history/snapshots HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "label": "Pre-publisher review snapshot"
}
```

**Response — 201 Created**
```json
{
  "id": "snap_g7h8",
  "action": "Pre-publisher review snapshot",
  "date": "2026-05-01T23:30:00Z",
  "user": "Alex Chen",
  "userRole": "admin",
  "stats": {
    "rowCount": 87,
    "liveTotal": 10215000,
    "ledgerEntryCount": 142,
    "greenlitVersion": 3
  },
  "auditEntryId": "aud_q7r8"
}
```

**Notes**
- The server snapshots the entire project state (rows, ledger, masterBudget, marketingBudget, phases, glPnl, glOneSheet, milestoneData, etc.) into a single immutable record.
- Snapshots can be very large (the full project document). Store them in compressed form, ideally in object storage (S3) referenced by the database row, not inline.

### `GET /projects/:id/history/snapshots/:snapshotId`

Get a single snapshot's full content (for inspection or comparison).

**Request**
```http
GET /projects/prj_2k9l/history/snapshots/snap_a1b2 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "id": "snap_a1b2",
  "action": "Manual snapshot",
  "date": "2026-05-01T17:00:00Z",
  "user": "Alex Chen",
  "snapshot": {
    "rows": [],
    "ledger": [],
    "masterBudget": { "developmentCosts": 6500000 },
    "marketingBudget": {},
    "phases": [],
    "phaseAssignment": {},
    "glPhases": [],
    "glPnl": {},
    "glOneSheet": {},
    "milestoneData": {},
    "closedMonths": {},
    "greenlitVersion": 3,
    "greenlitDate": "2026-02-14"
  },
  "size": 248320
}
```

**Notes**
- The full snapshot can be many MB. Consider returning a download URL instead of inline JSON for very large snapshots.
- `GET /history/snapshots/:id?download=true` returns the snapshot as a downloadable JSON file.

### `POST /projects/:id/history/snapshots/:snapshotId/restore`

Restore the project from a snapshot. **Auto-creates a "Pre-restore backup" snapshot first.**

**Request**
```http
POST /projects/prj_2k9l/history/snapshots/snap_c3d4/restore HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "confirmRestore": true,
  "note": "Reverting to GL v2 state to investigate variance discrepancy."
}
```

**Response — 200 OK**
```json
{
  "restoredFromSnapshotId": "snap_c3d4",
  "preRestoreBackupSnapshotId": "snap_i9j0",
  "restoredAt": "2026-05-01T23:45:00Z",
  "restoredBy": "Alex Chen",
  "auditEntryId": "aud_s9t0",
  "project": {
    "rowCount": 84,
    "liveTotal": 9450000,
    "ledgerEntryCount": 98,
    "greenlitVersion": 2
  }
}
```

**Notes**
- This is the most destructive operation in the entire app — it replaces virtually everything about the project. Require `confirmRestore: true` in the body as a safeguard against accidental calls.
- Permission: `manageUsers` or admin (the prototype gates restore behind admin-equivalent permission).
- After restore, all open WebSocket connections for this project should be sent a `project.restored` event so connected clients refresh.

### `DELETE /projects/:id/history/snapshots/:snapshotId`

Delete a specific snapshot (cleanup; rarely used).

**Request**
```http
DELETE /projects/prj_2k9l/history/snapshots/snap_e5f6 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "deletedSnapshotId": "snap_e5f6",
  "auditEntryId": "aud_u1v2"
}
```

**Notes**
- Cannot delete the most recent snapshot or any snapshot referenced by an approved amendment.
- Permission: admin only.

### `GET /projects/:id/history/audit`

List audit entries with filtering.

**Request**
```http
GET /projects/prj_2k9l/history/audit?q=ledger&user=Alex&from=2026-04-01&limit=100 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query parameters**
- `q` — free-text filter across `action`, `detail`, `user` (matches the prototype's filter input)
- `user` — exact user filter
- `action` — exact action filter (e.g. "Edit cell")
- `actionType` — category filter: `cell_edit`, `row_change`, `ledger`, `amendment`, `greenlight`, `recon`, `import`
- `from` / `to` — date range
- `limit` — page size (default 100)
- `cursor` — pagination cursor

**Response — 200 OK**
```json
{
  "projectId": "prj_2k9l",
  "summary": {
    "totalEntries": 4280,
    "filteredCount": 87,
    "oldestAt": "2025-08-12T14:22:00Z",
    "newestAt": "2026-05-01T23:45:00Z"
  },
  "entries": [
    {
      "id": "aud_a1b2",
      "date": "2026-05-01T15:00:00Z",
      "user": "Alex Chen",
      "userRole": "admin",
      "action": "Edit cell",
      "actionType": "cell_edit",
      "detail": "Lead Programmer - Pat Singh, May 2026",
      "extra": {
        "rowId": "row_n3kp",
        "rowName": "Lead Programmer",
        "dept": "Engineering",
        "mode": "live",
        "mKey": "2026-05",
        "oldVal": 18500,
        "newVal": 19000
      }
    },
    {
      "id": "aud_c3d4",
      "date": "2026-05-01T13:00:00Z",
      "user": "Sam Park",
      "userRole": "finance",
      "action": "Approve amendment",
      "actionType": "amendment",
      "detail": "Amendment 3 to GL v3 — External Support contractor rates increased",
      "extra": {
        "amendmentId": "amd_e7f8",
        "previousVersion": 3,
        "newVersion": 4,
        "delta": 350000
      }
    },
    {
      "id": "aud_e5f6",
      "date": "2026-04-30T18:00:00Z",
      "user": "Alex Chen",
      "userRole": "admin",
      "action": "Add ledger entry",
      "actionType": "ledger",
      "detail": "Acme Studio Ltd $15,240",
      "extra": {
        "ledgerId": "led_a1b2",
        "category": "Engineering"
      }
    }
  ],
  "pagination": {
    "total": 87,
    "returned": 87,
    "nextCursor": null
  }
}
```

### `GET /projects/:id/history/audit/:entryId`

Get full details of a single audit entry (for the expanded-row view).

**Request**
```http
GET /projects/prj_2k9l/history/audit/aud_a1b2 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "id": "aud_a1b2",
  "date": "2026-05-01T15:00:00Z",
  "user": "Alex Chen",
  "userRole": "admin",
  "userIp": "203.0.113.42",
  "userAgent": "Mozilla/5.0 …",
  "action": "Edit cell",
  "actionType": "cell_edit",
  "detail": "Lead Programmer - Pat Singh, May 2026",
  "extra": {
    "rowId": "row_n3kp",
    "rowName": "Lead Programmer",
    "dept": "Engineering",
    "mode": "live",
    "mKey": "2026-05",
    "oldVal": 18500,
    "newVal": 19000,
    "delta": 500
  },
  "relatedEntries": [
    { "id": "aud_a1b3", "action": "Edit cell", "date": "2026-05-01T15:00:01Z" },
    { "id": "aud_a1b4", "action": "Edit cell", "date": "2026-05-01T15:00:02Z" }
  ]
}
```

**Notes**
- `userIp` and `userAgent` are captured server-side (not from the prototype, but standard production practice).
- `relatedEntries` shows other audit entries created within a few seconds — useful for tracing batch operations like multi-cell pastes.

### `POST /projects/:id/history/audit/export`

Export audit log as CSV or JSON for compliance / legal review.

**Request**
```http
POST /projects/prj_2k9l/history/audit/export HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "format": "csv",
  "from": "2026-01-01",
  "to": "2026-04-30",
  "actionTypes": ["amendment", "greenlight", "ledger"]
}
```

**Response — 200 OK** — binary stream with appropriate `Content-Type` (`text/csv` or `application/json`) and `Content-Disposition: attachment; filename="audit-prj_2k9l-2026-Q1.csv"`.

**Notes**
- Permission: admin or finance only (audit exports are sensitive).
- Large date ranges may return `202 Accepted` and asynchronously prepare the file.

### Permissions Summary (History & Audit)

| Endpoint | Required permission |
|---|---|
| `GET /history/snapshots` | studio member, project access |
| `POST /history/snapshots` | `editSettings` |
| `GET /history/snapshots/:id` | studio member, project access |
| `POST /history/snapshots/:id/restore` | `manageUsers` or admin |
| `DELETE /history/snapshots/:id` | admin only; can't delete latest or referenced snapshots |
| `GET /history/audit` | studio member, project access |
| `GET /history/audit/:id` | studio member, project access |
| `POST /history/audit/export` | admin or finance |

---

# Cross-Cutting Concerns

## 1. Audit log retention

The prototype caps audit log at 200 entries per project (line 4885). Production must be **uncapped**:

- Audit entries are append-only — never updated, never deleted (except via admin tools for compliance).
- Partition the audit table by `(projectId, date)` for query performance.
- Index on `(projectId, date DESC)`, `(projectId, user, date DESC)`, `(projectId, actionType, date DESC)`.
- For very high-volume projects, consider a separate audit database optimized for write throughput.

## 2. Snapshot storage strategy

Snapshots can be 100KB+ each, and a busy project might accumulate hundreds. Suggestions:

- Store snapshot metadata in Postgres (id, date, user, action, stats).
- Store snapshot **content** in S3 as compressed JSON (gzip cuts size by ~80%).
- Reference S3 objects from the metadata row.
- Implement a retention policy: keep all snapshots from the last 90 days, then keep one per week for 90–365 days, then one per month indefinitely. Tunable per studio.
- Snapshots referenced by approved amendments are **never** purged (they're the legal record of what was approved).

## 3. AI integration consistency

Three pages call AI:
- Notes (transcript summarization)
- Documents (text extraction for AI context)
- Invoices (ledger / invoice queue — covered earlier)

All should go through one shared `POST /projects/:id/ai/extract` internal endpoint that:
- Holds the API key
- Enforces per-studio rate limits
- Caches results
- Logs usage for billing

## 4. WebSocket events

These four pages benefit from real-time updates:

| Event | Triggered by | Subscribers |
|---|---|---|
| `note.created` / `note.updated` / `note.deleted` | Notes endpoints | Anyone viewing the Notes page |
| `document.uploaded` / `document.deleted` | Documents endpoints | Anyone viewing Documents; AI chat if "Docs On" |
| `milestone.updated` | Milestone PATCH | Anyone viewing Milestones; Home page (current milestone card) |
| `snapshot.created` | Auto or manual snapshot | Anyone viewing History |
| `audit.entry_created` | Any write op | Anyone viewing Audit Log (with optional filter match) |

## 5. Permissions notes

Notes/Documents/Milestones use the same permission (`editNotes`) in the prototype. Production should consider splitting:

- `editNotes` — create/edit/delete notes
- `editDocuments` — upload/delete documents
- `editMilestones` — update milestone metadata

This lets a studio give a junior PM permission to update milestones without giving them write access to potentially-sensitive notes (e.g. meeting notes with personnel discussions).

## 6. The audit-log-driven Insights

Several pages I've already covered (Insights, Variance, Top Sheet) benefit from the audit log being a **first-class data source**:

- "Show me when our variance crossed 10%" — query audit for the timestamp of the cell edit that pushed it over.
- "Who approved the marketing budget increase?" — direct audit query.
- "What was the second-most-edited row last quarter?" — aggregate audit by `extra.rowId`.

The Insights page in production should likely have a "Recent Activity" section that surfaces audit log highlights — that's a free analytics feature once the audit data is properly indexed.

---

That covers Notes, Documents, Milestones, and History & Audit. The remaining unspec'd surfaces are:

- **Studio-level views beyond Portfolio** — Studio Insights (cross-project rollup), Studio Sales (cross-project sales), Studio Social (community signals), Studio Admin (users, roles, platforms)
- **Settings / User Management modals** — deeper than `PATCH /studios/:id`
- **AI assistant endpoints** (currently scattered references, no formal spec)

Want me to do those next? Or pivot to **the database schema** that all these endpoints would map to? At this point we have enough endpoints spec'd that a Postgres ERD is largely mechanical.


## Studio-Level Views

The studio shell has five views: `portfolio` (covered earlier), `insights`, `sales`, `social`, and `admin`. Plus user management modals reachable from anywhere. Let me work through each.

---

# Page 23: Studio Insights

Cross-project rollup. Aggregates every project in the studio into one financial dashboard.

## Editable Fields

**None.** Pure aggregation view.

## Calculated Fields

Each metric is computed by iterating `allProjects[]`, loading each project's full record from cache or storage, and summing across.

### Top KPIs

| Field | Formula |
|---|---|
| `totalProjects` | `allProjects.length` |
| `activeProjects` | Count where `today` falls between `startDate` and `start + monthCount` (i.e. project is mid-development) |
| `totalGreenlit` | `Σ projects.Σ row.greenlit[m.key]` across all projects, all rows, all months |
| `totalLive` | `Σ projects.Σ row.live[m.key]` |
| `totalLedgerSpent` | `Σ projects.Σ ledger[].totalBilled` |
| `totalVariance` | `totalLive − totalGreenlit` |
| `totalVariancePct` | `(totalVariance / totalGreenlit) × 100` |

### Per-Project Breakdown

| Field | Formula (per project) |
|---|---|
| `projectGreenlit` | `Σ row.greenlit[m.key]` |
| `projectLive` | `Σ row.live[m.key]` |
| `projectSpent` | `Σ ledger[].totalBilled` |
| `projectVariance` | `live − greenlit` |
| `projectHealth` | `pctSpent ≤ pctElapsed + 5%` → "On Track"; `pctSpent > 100%` → "Over Budget"; else "At Risk" |
| `projectPhase` | `phaseFor(currentMonth)` |

### Portfolio Health

| Field | Formula |
|---|---|
| `onTrackCount` | Count of projects where `health === "On Track"` |
| `atRiskCount` | Count where `health === "At Risk"` |
| `overBudgetCount` | Count where `health === "Over Budget"` |
| `pendingAmendmentsTotal` | `Σ projects.amendments.filter(s === "submitted").length` |
| `pendingInvoicesTotal` | `Σ projects.invoiceQueue.filter(s === "pending").length` |

### Cross-Project Charts

| Field | Formula |
|---|---|
| `monthlyBurnByProject[]` | Per month across the union of all projects' timelines: `{month, [projectId]: monthBurn}` for stacked bar chart |
| `cumulativeStudioBurn[]` | Running sum of all projects' `Σ live` over time |
| `groupSpendBreakdown[]` | Aggregate by group across studio: `{groupId, name, color, totalLive, totalGreenlit}` |

## Endpoints

### `GET /studios/:id/insights`

Returns the studio-level rollup. Server pre-computes everything across projects so the frontend doesn't load N project records.

**Request**
```http
GET /studios/std_a8f3/insights HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query parameters**
- `from` / `to` — date range filter (only counts months in range)
- `projectIds` — array filter (analyze a subset of the portfolio)
- `currency` — `usd` (default; everything converted) or `native` (per-project currency)

**Response — 200 OK**
```json
{
  "studioId": "std_a8f3",
  "computedAt": "2026-05-01T23:50:00Z",
  "summary": {
    "totalProjects": 6,
    "activeProjects": 4,
    "completedProjects": 1,
    "preProductionProjects": 1,
    "totalGreenlitBudget": 32850000,
    "totalLiveBudget": 34250000,
    "totalLedgerSpent": 8420000,
    "totalVariance": 1400000,
    "totalVariancePct": 4.26,
    "totalRowCount": 412,
    "totalLedgerEntries": 894
  },
  "portfolioHealth": {
    "onTrack": 3,
    "atRisk": 2,
    "overBudget": 1,
    "pendingAmendments": 4,
    "pendingInvoices": 12,
    "pendingBudgetChanges": 8
  },
  "byProject": [
    {
      "id": "prj_2k9l",
      "name": "Covenant",
      "icon": "⚔️",
      "phase": { "id": "alpha", "name": "Alpha", "color": "#3730a3" },
      "elapsedMonths": 9,
      "totalMonths": 36,
      "greenlit": 9750000,
      "live": 10215000,
      "spent": 1652340,
      "variance": 465000,
      "variancePct": 4.77,
      "health": "On Track"
    },
    {
      "id": "prj_x4mp",
      "name": "Nightfall",
      "icon": "🌙",
      "phase": { "id": "concept", "name": "Concept", "color": "#818cf8" },
      "elapsedMonths": 4,
      "totalMonths": 30,
      "greenlit": 4200000,
      "live": 4380000,
      "spent": 285000,
      "variance": 180000,
      "variancePct": 4.29,
      "health": "On Track"
    }
  ],
  "byGroup": [
    {
      "id": "devcosts",
      "name": "Development Costs",
      "color": "#6366f1",
      "totalLive": 21420000,
      "totalGreenlit": 20800000,
      "pctOfTotal": 62.54
    },
    {
      "id": "porting",
      "name": "Porting Costs",
      "color": "#f59e0b",
      "totalLive": 2480000,
      "totalGreenlit": 2400000,
      "pctOfTotal": 7.24
    }
  ],
  "monthlyBurn": [
    {
      "month": "2025-08",
      "byProject": { "prj_2k9l": 180000 },
      "total": 180000
    },
    {
      "month": "2026-04",
      "byProject": {
        "prj_2k9l": 312000,
        "prj_x4mp": 142000,
        "prj_99ab": 240000
      },
      "total": 694000
    }
  ],
  "cumulativeBurn": [
    { "month": "2025-08", "total": 180000 },
    { "month": "2026-04", "total": 8420000 }
  ]
}
```

**Notes**
- Aggressive caching: TTL of 5 minutes. Invalidate on any project-level write.
- For studios with many projects, this can become a heavyweight query — consider a materialized view that's incrementally maintained.

### Permissions

| Endpoint | Required permission |
|---|---|
| `GET /studios/:id/insights` | studio member |

---

# Page 24: Studio Sales

Cross-project sales rollup. Same shape as project-level Sales Dashboard but aggregated across the studio's catalog.

## Editable Fields

**None.** Read-only aggregation.

## Calculated Fields

Same primitives as project Sales Dashboard, but iterating across all projects.

| Field | Formula |
|---|---|
| `studioTotalRevenue` | `Σ projects.Σ salesData[].revenue` |
| `studioTotalUnits` | `Σ projects.Σ salesData[].units` |
| `studioTotalWishlists` | `Σ projects.Σ salesData[].wishlists + wishlistData[].adds` |
| Per-project sales | Each project's totals |
| `byPlatformAcrossPortfolio` | Group-by platform across all sales rows in all projects |
| `topGrossing[]` | Projects sorted by revenue descending |
| `topByUnits[]` | Projects sorted by units descending |
| `byMonth` | Same time-series as project-level, but unioned across projects |
| `byRegion` | Aggregated across all projects |
| Per-project ROI | `(projectRevenue − projectGreenlit) / projectGreenlit × 100` |
| Studio ROI | `(studioTotalRevenue − studioGreenlit) / studioGreenlit × 100` |

## Endpoints

### `GET /studios/:id/sales`

Returns studio-wide sales aggregation.

**Request**
```http
GET /studios/std_a8f3/sales?from=2026-01-01&to=2026-04-30 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Optional query parameters**
- `from` / `to` — date range
- `platform` — filter to one platform across the portfolio
- `projectIds` — subset filter

**Response — 200 OK**
```json
{
  "studioId": "std_a8f3",
  "filters": { "from": "2026-01-01", "to": "2026-04-30", "platform": null },
  "summary": {
    "totalRevenue": 4287340,
    "totalUnits": 124820,
    "totalWishlists": 248920,
    "totalRefunds": 2480,
    "studioROI": -86.95,
    "totalGreenlitInvested": 32850000
  },
  "byProject": [
    {
      "id": "prj_2k9l",
      "name": "Covenant",
      "icon": "⚔️",
      "revenue": 3245890,
      "units": 87421,
      "wishlists": 142350,
      "refunds": 1842,
      "roi": -66.71,
      "greenlit": 9750000,
      "releaseStatus": "released"
    },
    {
      "id": "prj_99ab",
      "name": "Veilfall",
      "icon": "🌫️",
      "revenue": 1041450,
      "units": 37399,
      "wishlists": 106570,
      "refunds": 638,
      "roi": -88.43,
      "greenlit": 9000000,
      "releaseStatus": "released"
    }
  ],
  "byPlatform": [
    { "platform": "steam", "revenue": 2640200, "units": 78420, "pctRev": 61.59 },
    { "platform": "playstation", "revenue": 920140, "units": 25340, "pctRev": 21.46 },
    { "platform": "xbox", "revenue": 487000, "units": 13420, "pctRev": 11.36 },
    { "platform": "switch", "revenue": 240000, "units": 7640, "pctRev": 5.60 }
  ],
  "monthly": [
    {
      "month": "2026-01",
      "total": 1124850,
      "units": 31420,
      "byProject": { "prj_2k9l": 1024850, "prj_99ab": 100000 }
    },
    {
      "month": "2026-04",
      "total": 980340,
      "units": 28230,
      "byProject": { "prj_2k9l": 745890, "prj_99ab": 234450 }
    }
  ],
  "byRegion": [
    { "region": "United States", "revenue": 1842500, "units": 51240 },
    { "region": "United Kingdom", "revenue": 624800, "units": 18420 },
    { "region": "Germany", "revenue": 528200, "units": 15680 }
  ],
  "topGrossing": [
    { "id": "prj_2k9l", "name": "Covenant", "revenue": 3245890 },
    { "id": "prj_99ab", "name": "Veilfall", "revenue": 1041450 }
  ]
}
```

### Permissions

| Endpoint | Required permission |
|---|---|
| `GET /studios/:id/sales` | studio member |

---

# Page 25: Studio Social

Community/marketing signals. The prototype uses placeholder data here — it's a stub for a future feature that would integrate with social platforms.

## Editable Fields

In the current prototype: **none**. The page is a placeholder.

For production, the editable surface would likely be:

| Field | Type | Storage |
|---|---|---|
| Social account configuration (per platform) | `{platform, handle, apiKey, enabled}` | `studio.socialAccounts[]` |
| Tracked subreddits / hashtags | Array | `studio.socialTracking` |
| Sentiment dashboard date range | Date range | UI state |

## Calculated Fields

| Field | Formula |
|---|---|
| `followerCounts` per platform | Pulled from each platform's API (Twitter/X, Discord, YouTube, TikTok, Reddit, Steam community) |
| `followerGrowth` (week-over-week) | `(thisWeek − lastWeek) / lastWeek × 100` |
| `mentionVolume` | Total mentions across tracked terms in date range |
| `sentimentScore` | Weighted average from sentiment analysis service |
| `topPosts[]` | Posts ranked by engagement (likes + comments + shares) |
| `engagementRate` per platform | `(likes + comments + shares) / impressions × 100` |
| Per-game breakdown | Same metrics scoped to game-specific channels |

## Endpoints

### `GET /studios/:id/social`

Returns aggregated social metrics across configured platforms.

**Request**
```http
GET /studios/std_a8f3/social?from=2026-04-01&to=2026-04-30 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "studioId": "std_a8f3",
  "filters": { "from": "2026-04-01", "to": "2026-04-30" },
  "summary": {
    "totalFollowers": 248920,
    "weeklyGrowth": 2.4,
    "totalMentions": 14820,
    "avgSentiment": 0.72,
    "topPlatform": "twitter"
  },
  "byPlatform": [
    {
      "platform": "twitter",
      "handle": "@FictionsGames",
      "followers": 84200,
      "growth": 3.1,
      "mentions": 8420,
      "engagement": 4.2,
      "sentiment": 0.68
    },
    {
      "platform": "discord",
      "serverId": "discord_n7p2",
      "members": 42800,
      "growth": 5.8,
      "messagesSent": 18200,
      "activeMembers": 8420
    },
    {
      "platform": "youtube",
      "channelId": "UC_xyz",
      "subscribers": 28200,
      "growth": 1.2,
      "totalViews": 482000,
      "avgViewDuration": 240
    },
    {
      "platform": "reddit",
      "subreddits": ["r/FictionsGames", "r/Covenant"],
      "totalSubscribers": 12400,
      "weeklyPosts": 240,
      "topPostScore": 1842
    }
  ],
  "topMentions": [
    {
      "platform": "twitter",
      "url": "https://twitter.com/...",
      "author": "@some_user",
      "content": "...",
      "engagement": 8420,
      "sentiment": 0.92,
      "timestamp": "2026-04-22T18:30:00Z"
    }
  ],
  "byGame": [
    {
      "projectId": "prj_2k9l",
      "name": "Covenant",
      "totalMentions": 8420,
      "sentiment": 0.78,
      "growth": 12.4
    }
  ]
}
```

### `PATCH /studios/:id/social/accounts`

Configure social account integrations.

**Request**
```http
PATCH /studios/std_a8f3/social/accounts HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "accounts": [
    {
      "platform": "twitter",
      "handle": "@FictionsGames",
      "enabled": true
    },
    {
      "platform": "discord",
      "serverId": "discord_n7p2",
      "enabled": true
    }
  ]
}
```

**Response — 200 OK**
```json
{
  "studioId": "std_a8f3",
  "accounts": [
    {
      "platform": "twitter",
      "handle": "@FictionsGames",
      "enabled": true,
      "connectedAt": "2026-05-01T23:55:00Z",
      "lastSyncedAt": "2026-05-01T23:58:00Z"
    },
    {
      "platform": "discord",
      "serverId": "discord_n7p2",
      "enabled": true,
      "connectedAt": "2026-05-01T23:55:00Z",
      "lastSyncedAt": null
    }
  ],
  "auditEntryId": "aud_w3x4"
}
```

### Permissions

| Endpoint | Required permission |
|---|---|
| `GET /studios/:id/social` | studio member |
| `PATCH /studios/:id/social/accounts` | `manageUsers` |

**Production note:** This is one of the largest unbuilt areas of the application. The prototype only shows a placeholder. Building it out requires API integrations with each social platform (OAuth flows, rate limits, token management), a sentiment analysis service, and a background job system to keep the metrics fresh. It's effectively a sub-product. The brief acknowledges this is "future scope."

---

# Page 26: Studio Admin

Where users, roles, project access, and platform options are managed. The prototype's admin page is dense — it has multiple sub-areas controlled by `adminTab` state.

## Editable Fields

### User Management

Per user (`studio.users[]`):

| Field | Type | Default | Storage path |
|---|---|---|---|
| Name | Text | "" | `user.name` (editable) |
| Email | Email | "" | `user.email` (production-only; prototype doesn't have this) |
| Role | Dropdown | "support" | `user.role` (one of `admin`, `producer`, `support`, `finance`, `developer`, `viewer`) |
| Project access | Multi-select / "All" | `null` (= all) | `user.projectIds` (`null` = all projects, array = specific projects) |
| Active flag | Boolean | true | `user.active` (production; for soft-deactivation) |

### Role-Page Visibility Matrix

| Field | Type | Storage path |
|---|---|---|
| Per-role page visibility | Array of view IDs | `studio.rolePages[role]` (`null` = all pages, array = whitelist) |

This is the matrix where admins control which roles see which pages — e.g. allowing Support to see Insights but hiding the Ledger.

### Platform Options

| Field | Type | Storage path |
|---|---|---|
| Platform list | Array of strings | `studio.platformOptions[]` |
| Default platform set on new projects | Array | `studio.defaultPlatforms[]` (production; prototype doesn't have this) |

### Page-Level Actions

| Action | Effect |
|---|---|
| Add User | Opens user creation modal |
| Delete User | Removes user from `studio.users[]`; confirmation prompt |
| Assign User to Projects | Opens multi-select modal |
| Edit Role-Page Matrix | Per-role-per-page checkbox grid |
| Add / Remove Platform | Updates `studio.platformOptions[]` |

## Calculated Fields

| Field | Formula |
|---|---|
| User count by role | `Σ users.filter(u => u.role === r)` per role |
| Active user count | `Σ users.filter(u => u.active)` |
| User's project count | `u.projectIds === null ? "All" : u.projectIds.length` |
| Pending invitations | Count from portal invitations table |
| Recently active | Users with `lastLoginAt` within last 30 days |

## Endpoints

### `GET /studios/:id/users`

List users with their roles and project assignments.

**Request**
```http
GET /studios/std_a8f3/users HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "studioId": "std_a8f3",
  "summary": {
    "totalUsers": 12,
    "activeUsers": 10,
    "byRole": {
      "admin": 2,
      "producer": 4,
      "support": 2,
      "finance": 2,
      "developer": 2,
      "viewer": 0
    }
  },
  "users": [
    {
      "id": "usr_001",
      "name": "Alex Chen",
      "email": "alex@fictions.com",
      "role": "admin",
      "active": true,
      "projectIds": null,
      "projectAccess": "all",
      "createdAt": "2025-08-12T14:22:00Z",
      "lastLoginAt": "2026-05-01T09:00:00Z",
      "permissions": {
        "editBudget": true,
        "editPhases": true,
        "editLedger": true,
        "editSales": true,
        "editNotes": true,
        "editSettings": true,
        "manageUsers": true,
        "approveAmendments": true,
        "approveInvoices": true,
        "approveBudgetChanges": true,
        "createProject": true,
        "deleteProject": true,
        "deleteRows": true,
        "importCSV": true,
        "submitInvoices": true
      }
    },
    {
      "id": "usr_007",
      "name": "Jamie Rivera",
      "email": "jamie@external-studio.com",
      "role": "developer",
      "active": true,
      "projectIds": ["prj_2k9l"],
      "projectAccess": "limited",
      "createdAt": "2026-01-04T10:00:00Z",
      "lastLoginAt": "2026-04-30T14:00:00Z",
      "permissions": {
        "editBudget": false,
        "editLedger": false,
        "submitBudgetChanges": true,
        "submitInvoices": true,
        "viewBudget": true,
        "viewMilestones": true
      }
    }
  ]
}
```

### `POST /studios/:id/users`

Create a new user.

**Request**
```http
POST /studios/std_a8f3/users HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "name": "Devon Lee",
  "email": "devon@fictions.com",
  "role": "producer",
  "projectIds": ["prj_2k9l", "prj_x4mp"],
  "sendInvitationEmail": true
}
```

**Response — 201 Created**
```json
{
  "id": "usr_y5z6",
  "name": "Devon Lee",
  "email": "devon@fictions.com",
  "role": "producer",
  "active": true,
  "projectIds": ["prj_2k9l", "prj_x4mp"],
  "createdAt": "2026-05-02T00:00:00Z",
  "createdBy": "Alex Chen",
  "invitationSent": true,
  "invitationExpiresAt": "2026-05-09T00:00:00Z",
  "auditEntryId": "aud_a1b2"
}
```

**Notes**
- Sets a temporary invitation token. The user clicks the email link to set their password (or SSO links).
- If `projectIds: null`, the user has access to all projects.
- Permission: `manageUsers`.

### `PATCH /studios/:id/users/:userId`

Update user fields. Most common changes are role or project access.

**Request — change role**
```http
PATCH /studios/std_a8f3/users/usr_y5z6 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "role": "support"
}
```

**Request — update project access**
```http
PATCH /studios/std_a8f3/users/usr_y5z6 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "projectIds": ["prj_2k9l", "prj_x4mp", "prj_99ab"]
}
```

**Request — deactivate**
```http
PATCH /studios/std_a8f3/users/usr_y5z6 HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "active": false
}
```

**Response — 200 OK**
```json
{
  "id": "usr_y5z6",
  "name": "Devon Lee",
  "role": "support",
  "active": false,
  "projectIds": ["prj_2k9l", "prj_x4mp", "prj_99ab"],
  "updatedAt": "2026-05-02T00:15:00Z",
  "auditEntryId": "aud_c3d4"
}
```

**Notes**
- `projectIds: null` reverts to "all projects."
- Deactivating a user keeps their record (and audit history) intact but prevents login.
- Cannot deactivate the last admin in the studio (return `409`).

### `DELETE /studios/:id/users/:userId`

Hard-delete a user. Use with caution — audit history references this user.

**Request**
```http
DELETE /studios/std_a8f3/users/usr_y5z6 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "deletedUserId": "usr_y5z6",
  "deletedAt": "2026-05-02T00:30:00Z",
  "auditEntryRetention": "preserved",
  "auditEntryId": "aud_e5f6"
}
```

**Notes**
- The user record is removed but audit entries that reference them denormalize the user's name and role at the time of the action — so audit history stays intact.
- Production should prefer **deactivation** over deletion. Add an explicit `?hard=true` query param for actual deletion.
- Cannot delete the last admin.

### `POST /studios/:id/users/:userId/reset-password`

Trigger a password reset email.

**Request**
```http
POST /studios/std_a8f3/users/usr_y5z6/reset-password HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "userId": "usr_y5z6",
  "resetTokenSent": true,
  "expiresAt": "2026-05-02T01:00:00Z"
}
```

### `POST /studios/:id/users/:userId/resend-invitation`

Resend an invitation email if the user hasn't yet activated their account.

**Request**
```http
POST /studios/std_a8f3/users/usr_y5z6/resend-invitation HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "userId": "usr_y5z6",
  "invitationResent": true,
  "newExpiresAt": "2026-05-09T00:30:00Z"
}
```

### `GET /studios/:id/role-permissions`

Returns the role definitions (which permissions each role grants) — read by the admin UI to show what each role can do.

**Request**
```http
GET /studios/std_a8f3/role-permissions HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "studioId": "std_a8f3",
  "roles": [
    {
      "id": "admin",
      "name": "Admin",
      "description": "Full access to everything.",
      "permissions": {
        "editBudget": true,
        "editPhases": true,
        "editLedger": true,
        "editSales": true,
        "editNotes": true,
        "editSettings": true,
        "manageUsers": true,
        "approveAmendments": true,
        "approveInvoices": true,
        "approveBudgetChanges": true,
        "createProject": true,
        "deleteProject": true,
        "deleteRows": true,
        "importCSV": true,
        "submitInvoices": true
      }
    },
    {
      "id": "producer",
      "name": "Producer",
      "description": "Manages day-to-day operations of assigned projects.",
      "permissions": {
        "editBudget": true,
        "editPhases": true,
        "editLedger": true,
        "editNotes": true,
        "createProject": true,
        "submitInvoices": true,
        "importCSV": true
      }
    },
    {
      "id": "support",
      "name": "Support",
      "description": "Read access plus the ability to add notes and view financials.",
      "permissions": {
        "editNotes": true
      }
    },
    {
      "id": "finance",
      "name": "Finance",
      "description": "Approves financial changes and manages reconciliation.",
      "permissions": {
        "editLedger": true,
        "approveAmendments": true,
        "approveInvoices": true,
        "approveBudgetChanges": true,
        "deleteProject": true
      }
    },
    {
      "id": "developer",
      "name": "Developer",
      "description": "External developers — submit changes and invoices for review.",
      "permissions": {
        "submitBudgetChanges": true,
        "submitInvoices": true,
        "viewBudget": true,
        "viewMilestones": true
      }
    },
    {
      "id": "viewer",
      "name": "Viewer",
      "description": "Read-only access for stakeholders.",
      "permissions": {}
    }
  ]
}
```

**Notes**
- Roles are **system-defined** in the prototype — admins can't create new roles or modify existing ones, only assign users to them. Production might allow custom roles; in that case this endpoint becomes editable.
- The `viewer` role here closes the gap I identified earlier (the prototype has 5 roles; the brief specifies 6). Adding it is a small change to `DEFAULT_ROLES`.

### `GET /studios/:id/role-pages`

Returns the per-role page visibility matrix.

**Request**
```http
GET /studios/std_a8f3/role-pages HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "studioId": "std_a8f3",
  "rolePages": {
    "admin": null,
    "producer": null,
    "support": ["home", "dashboard", "insights", "variance", "notes_creative", "notes_meeting"],
    "finance": null,
    "developer": ["home", "budget", "milestones", "invoices"],
    "viewer": ["home", "dashboard", "insights", "variance"]
  },
  "availablePages": [
    { "id": "home", "name": "Home" },
    { "id": "dashboard", "name": "Dashboard" },
    { "id": "budget", "name": "Game Budget" },
    { "id": "master_budget", "name": "Contracted Budget" },
    { "id": "phases", "name": "Phases" },
    { "id": "milestones", "name": "Milestones" },
    { "id": "pnl", "name": "P&L Projection" },
    { "id": "onesheet", "name": "One Sheet" },
    { "id": "amendments", "name": "Amendments" },
    { "id": "rollup", "name": "Rollup" },
    { "id": "payment", "name": "Payment Schedule" },
    { "id": "variance", "name": "Variance" },
    { "id": "insights", "name": "Insights" },
    { "id": "ledger", "name": "Game Ledger" },
    { "id": "mktg_ledger", "name": "Marketing Ledger" },
    { "id": "ledger_topsheet", "name": "Top Sheet" },
    { "id": "reconcile", "name": "Reconciliation" },
    { "id": "invoices", "name": "Invoice Queue" },
    { "id": "sales", "name": "Sales Dashboard" },
    { "id": "sales_steam", "name": "Steam Sales" },
    { "id": "notes_creative", "name": "Creative Notes" },
    { "id": "notes_meeting", "name": "Meeting Notes" },
    { "id": "documents", "name": "Documents" },
    { "id": "history", "name": "History" }
  ]
}
```

### `PATCH /studios/:id/role-pages`

Update the role-page visibility matrix.

**Request**
```http
PATCH /studios/std_a8f3/role-pages HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "rolePages": {
    "support": ["home", "dashboard", "insights", "variance", "notes_creative", "notes_meeting", "ledger"],
    "viewer": ["home", "dashboard", "insights", "variance", "ledger_topsheet"]
  }
}
```

**Response — 200 OK**
```json
{
  "studioId": "std_a8f3",
  "rolePages": {
    "admin": null,
    "producer": null,
    "support": ["home", "dashboard", "insights", "variance", "notes_creative", "notes_meeting", "ledger"],
    "finance": null,
    "developer": ["home", "budget", "milestones", "invoices"],
    "viewer": ["home", "dashboard", "insights", "variance", "ledger_topsheet"]
  },
  "updatedAt": "2026-05-02T00:45:00Z",
  "auditEntryId": "aud_g7h8"
}
```

**Notes**
- `null` for a role means "all pages visible."
- Sending only a subset of roles in the body partial-updates only those roles.
- Permission: `manageUsers`.

### `PATCH /studios/:id/platform-options`

Already covered earlier in `PATCH /studios/:id` — but worth a dedicated endpoint for the admin UI's "manage platforms" tab.

**Request**
```http
PATCH /studios/std_a8f3/platform-options HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "platformOptions": [
    "Steam", "Epic", "GOG",
    "PS5", "PS6",
    "Xbox Series X|S",
    "Nintendo Switch", "Nintendo Switch 2",
    "iOS", "Android"
  ],
  "defaultPlatforms": ["Steam", "PS5", "Xbox Series X|S"]
}
```

**Response — 200 OK**
```json
{
  "studioId": "std_a8f3",
  "platformOptions": [
    "Steam", "Epic", "GOG",
    "PS5", "PS6",
    "Xbox Series X|S",
    "Nintendo Switch", "Nintendo Switch 2",
    "iOS", "Android"
  ],
  "defaultPlatforms": ["Steam", "PS5", "Xbox Series X|S"],
  "updatedAt": "2026-05-02T00:50:00Z",
  "auditEntryId": "aud_i9j0"
}
```

### Permissions Summary (Admin)

| Endpoint | Required permission |
|---|---|
| `GET /users` | `manageUsers` |
| `POST /users` | `manageUsers` |
| `PATCH /users/:id` | `manageUsers` |
| `DELETE /users/:id` | `manageUsers` (cannot delete last admin) |
| `POST /users/:id/reset-password` | `manageUsers` OR self |
| `POST /users/:id/resend-invitation` | `manageUsers` |
| `GET /role-permissions` | studio member |
| `GET /role-pages` | studio member |
| `PATCH /role-pages` | `manageUsers` |
| `PATCH /platform-options` | `manageUsers` |

---

# Page 27: Settings & User Profile Modals

These modals are reachable from the studio menu hamburger and from various other places. They wrap project-level settings and user-level preferences.

## Editable Fields

### Project Settings Modal (per project, line ~3296)

| Field | Type | Storage path |
|---|---|---|
| Project name | Text | `project.name` |
| Studio name | Text | `project.studio` (developer studio name; not the parent Fictions studio) |
| Project icon | Emoji picker | `project.icon` |
| Start date | Month picker | `project.startDate` |
| Month count | Number | `project.monthCount` |
| Multi-currency on/off | Boolean | `project.multiCurrency` |
| Alt currency | Dropdown | `project.altCurrency` |
| Point persons | Multi-select | `project.pointPerson[]` |
| Target release date | Month picker | `project.targetReleaseDate` |
| Platforms | Multi-select | `project.platforms[]` |
| Greenlit lock toggle | Boolean | `project.greenlitLocked` |

### User Profile Modal (per user, accessible by self)

| Field | Type | Storage path |
|---|---|---|
| Display name | Text | `user.name` |
| Email | Email | `user.email` (production) |
| Avatar | Image upload | `user.avatarUrl` (production) |
| Notification preferences | Checkboxes | `user.notificationPrefs` (production) |
| UI scale (75–125%) | Slider | Stored in `localStorage.fcc_ui_scale` (per-device) |
| Theme | Light / dark / auto | Stored in localStorage (production: `user.theme`) |
| Default landing page | Dropdown | `user.defaultLandingPage` (production) |

### Studio Settings Modal

| Field | Type | Storage path |
|---|---|---|
| Studio name | Text | `studio.name` |
| Studio logo | Image upload | `studio.logoUrl` (production) |
| Default currency | Dropdown | `studio.defaultCurrency` (production) |
| Fiscal year start | Month dropdown | `studio.fiscalYearStart` (production) |
| Time zone | Dropdown | `studio.timezone` (production) |

## Calculated Fields

The settings modals are mostly inputs without derived values. A few exceptions:

| Field | Formula |
|---|---|
| "Last modified" timestamp on each setting | Read from `updatedAt` |
| Permission preview | Resolves the role's permission set so the modal can show "you have permission to do X, Y, Z" |
| Project access summary (for user profile) | Count of projects user has access to |
| Storage usage (for studio settings) | Total bytes used across all projects' documents and snapshots |

## Endpoints

### `GET /me`

Returns the current user's profile (whoever is making the request).

**Request**
```http
GET /me HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "id": "usr_001",
  "studioId": "std_a8f3",
  "name": "Alex Chen",
  "email": "alex@fictions.com",
  "role": "admin",
  "avatarUrl": "https://cdn.fictions-folio.com/avatars/usr_001.jpg",
  "projectIds": null,
  "permissions": {
    "editBudget": true,
    "editPhases": true,
    "editLedger": true,
    "editSettings": true,
    "manageUsers": true,
    "approveAmendments": true,
    "approveInvoices": true,
    "createProject": true,
    "deleteProject": true
  },
  "preferences": {
    "theme": "dark",
    "uiScale": 100,
    "defaultLandingPage": "portfolio",
    "notifications": {
      "email": {
        "amendmentSubmitted": true,
        "amendmentApproved": true,
        "amendmentRejected": true,
        "invoiceSubmitted": true,
        "weeklyDigest": false
      },
      "inApp": {
        "all": true
      }
    }
  },
  "lastLoginAt": "2026-05-01T09:00:00Z",
  "createdAt": "2025-08-12T14:22:00Z"
}
```

### `PATCH /me`

Update the current user's own profile.

**Request**
```http
PATCH /me HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "name": "Alex Chen-Park",
  "preferences": {
    "theme": "light",
    "uiScale": 110,
    "notifications": {
      "email": {
        "weeklyDigest": true
      }
    }
  }
}
```

**Response — 200 OK**
```json
{
  "id": "usr_001",
  "name": "Alex Chen-Park",
  "preferences": {
    "theme": "light",
    "uiScale": 110,
    "notifications": {
      "email": {
        "amendmentSubmitted": true,
        "amendmentApproved": true,
        "amendmentRejected": true,
        "invoiceSubmitted": true,
        "weeklyDigest": true
      },
      "inApp": { "all": true }
    }
  },
  "updatedAt": "2026-05-02T01:00:00Z"
}
```

**Notes**
- Users can update their own `name`, `email` (with verification flow), `avatarUrl`, and all `preferences` fields.
- Users **cannot** update their own `role`, `projectIds`, or `active` flag — those go through the admin endpoints.
- `notifications` is partial-merge to deep keys (the spread semantics).

### `POST /me/avatar`

Upload a new avatar.

**Request**
```http
POST /me/avatar HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: multipart/form-data

(file as multipart field "file")
```

**Response — 200 OK**
```json
{
  "avatarUrl": "https://cdn.fictions-folio.com/avatars/usr_001.jpg",
  "updatedAt": "2026-05-02T01:05:00Z"
}
```

### `POST /me/change-password`

Change own password.

**Request**
```http
POST /me/change-password HTTP/1.1
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "currentPassword": "...",
  "newPassword": "..."
}
```

**Response — 200 OK**
```json
{
  "passwordChangedAt": "2026-05-02T01:10:00Z",
  "sessionsInvalidated": 2,
  "currentSessionPreserved": true
}
```

**Notes**
- All other active sessions for this user are invalidated. Only the current session continues.
- Email notification sent confirming the change (security best practice).

### `GET /me/sessions`

List all active sessions for the current user.

**Request**
```http
GET /me/sessions HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "sessions": [
    {
      "id": "sess_a1b2",
      "isCurrent": true,
      "createdAt": "2026-05-01T09:00:00Z",
      "lastActiveAt": "2026-05-02T01:00:00Z",
      "expiresAt": "2026-05-08T09:00:00Z",
      "ipAddress": "203.0.113.42",
      "userAgent": "Chrome 132 on macOS",
      "location": "San Francisco, CA"
    },
    {
      "id": "sess_c3d4",
      "isCurrent": false,
      "createdAt": "2026-04-28T18:00:00Z",
      "lastActiveAt": "2026-04-30T22:00:00Z",
      "expiresAt": "2026-05-05T18:00:00Z",
      "ipAddress": "192.0.2.10",
      "userAgent": "Mobile Safari on iOS",
      "location": "San Francisco, CA"
    }
  ]
}
```

### `DELETE /me/sessions/:sessionId`

Revoke a specific session.

**Request**
```http
DELETE /me/sessions/sess_c3d4 HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "revokedSessionId": "sess_c3d4",
  "revokedAt": "2026-05-02T01:15:00Z"
}
```

### `POST /me/logout`

Log out of the current session.

**Request**
```http
POST /me/logout HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "sessionId": "sess_a1b2",
  "loggedOutAt": "2026-05-02T01:20:00Z"
}
```

### `POST /me/logout-all`

Log out of all sessions.

**Request**
```http
POST /me/logout-all HTTP/1.1
Authorization: Bearer eyJhbGc...
```

**Response — 200 OK**
```json
{
  "sessionsRevoked": 3,
  "loggedOutAt": "2026-05-02T01:25:00Z"
}
```

### Permissions Summary (Settings & Profile)

| Endpoint | Required permission |
|---|---|
| `GET /me` | self (any authenticated user) |
| `PATCH /me` | self |
| `POST /me/avatar` | self |
| `POST /me/change-password` | self |
| `GET /me/sessions` | self |
| `DELETE /me/sessions/:id` | self |
| `POST /me/logout` | self |
| `POST /me/logout-all` | self |

---

# Cross-Cutting Concerns

## 1. Authentication strategy for the studio shell

The studio-level views are the "main app" — they should run on `app.fictions-folio.com` (separate from `portal.fictions-folio.com`). Authentication options the brief calls for:

- **Email + password** with session cookies (HttpOnly, Secure, SameSite=Lax)
- **OAuth (Google, GitHub, Microsoft)** for studio users
- **SSO (SAML / OIDC)** for enterprise customers — Phase 4 feature

All studio API endpoints validate the session cookie. Refresh tokens stored in HttpOnly cookies; access tokens short-lived in memory.

## 2. Multi-studio support (future)

The current design assumes one studio per user. The data model supports multiple — a user could belong to several studios as a freelance producer. Not in the prototype but worth designing for.

If supported, `GET /me` would return:
```json
{
  "studios": [
    { "studioId": "std_a8f3", "name": "Fictions", "role": "admin" },
    { "studioId": "std_b7c4", "name": "External Publisher", "role": "support" }
  ],
  "currentStudioId": "std_a8f3"
}
```

The user picks which studio to work in; their session is scoped to that studio at any time.

## 3. The Viewer role gap

The prototype has 5 roles; the brief calls for 6, including **Viewer**. Adding it requires:

1. Add `viewer` to `DEFAULT_ROLES` (in code)
2. Empty permission set
3. Update role-pages defaults to filter to read-only views
4. Ensure UI components respect missing permissions (most already do)
5. Update Studio Admin's role dropdown to include Viewer

This is one of the simplest gaps to close — likely a 1-day task.

## 4. Notification system

The Settings modal references notification preferences. Building this requires:

- A background job system (e.g. BullMQ on Redis) for queuing notifications.
- Email delivery service (e.g. Postmark, SES).
- WebSocket events for in-app notifications.
- Per-event-type preferences: each user can opt in/out of email or in-app for each event class.
- A notification log per user so users can see what they've been notified about even after dismissing.
- Daily/weekly digest emails (rolled-up summaries) as an alternative to per-event emails.

This is one of the bigger Phase 3 deliverables.

## 5. Audit trail for admin actions

Every action in the Studio Admin section produces an audit entry, especially:

- Creating, deleting, deactivating users
- Changing user roles
- Modifying project access
- Updating role-page matrix
- Resetting passwords
- Modifying platform options

These audit entries should be **studio-level** (visible across projects) and queryable via a separate `GET /studios/:id/admin-audit` endpoint with the same shape as project audit log.

## 6. Studio Insights aggregation cost

`GET /studios/:id/insights` is potentially expensive — for a 20-project studio, it iterates 20 projects' worth of cell data. Strategies:

- Pre-aggregate at write time: maintain a `studio_metrics` materialized view that's updated on every project write.
- Cache the response with a short TTL (5 minutes) and invalidate on any project write.
- For very large studios, consider an analytics database (ClickHouse, Snowflake) for the cross-project queries.

## 7. Soft vs hard delete patterns

Across all admin endpoints:

- **Users**: prefer deactivation (`active: false`) over deletion. Audit history preserved.
- **Projects**: soft-delete with 30-day restore window (covered earlier in `DELETE /projects/:id`).
- **Settings/configurations**: never hard-delete; audit changes always.

A consistent pattern: every entity that has audit history should soft-delete by default, with an admin-only `?hard=true` for genuine purges.

---

That covers the studio-level views and settings. The remaining unspec'd surfaces, which are smaller:

- **AI assistant endpoints** (the chat UI scattered throughout the app)
- **Database schema design** (Postgres ERD derived from all the endpoints)
- **Migration strategy** from prototype localStorage to production backend
- **Deployment / infrastructure architecture**

Want me to do the AI endpoints next (smallest), or shift to the database schema (most useful for kicking off backend implementation)?