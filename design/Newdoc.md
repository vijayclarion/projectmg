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


