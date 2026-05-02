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
