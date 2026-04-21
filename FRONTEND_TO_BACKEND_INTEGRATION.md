# Comprehensive Frontend to Backend Integration Audit
## Fictions Command Center v0.4 - React.js to Node.js Backend Migration

**Document Date:** April 20, 2026  
**Audit Scope:** Static React application to dynamic backend-driven architecture  
**Status:** Initial Analysis Complete

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Page/View Analysis & API Mapping](#pageview-analysis--api-mapping)
3. [Static Data Structures](#static-data-structures)
4. [Required API Endpoints & Database Schema](#required-api-endpoints--database-schema)
5. [Implementation Strategy](#implementation-strategy)
6. [Migration Phases](#migration-phases)
7. [Technical Notes per View](#technical-notes-per-view)
8. [Summary & Recommendations](#summary--recommendations)

---

## Executive Summary

This is a **monolithic React.js single-file application** (~9,000+ lines) serving as a game development budget and project management system. The application currently uses **in-memory state with browser localStorage persistence** via `window.storage.get/set`. All data structures are defined as static defaults and local state.

### Current Architecture
- **Frontend:** React 18+ with hooks, no external state management (useState/useContext)
- **Persistence:** Browser localStorage via `window.storage` API
- **Data Flow:** Unidirectional (state → UI), no API layer
- **Authentication:** Mocked via roles (`admin`, `finance`, `producer`, `stakeholder`, `devMode`)
- **File Handling:** Client-side CSV parsing, PDF export via HTML

### Migration Target
- **Frontend:** React with TanStack Query for server state, Axios for HTTP
- **Backend:** Node.js/Express with PostgreSQL
- **Authentication:** JWT + refresh tokens
- **File Storage:** AWS S3 / Azure Blob Storage
- **Real-time:** Optional WebSocket for multi-user collaboration

---

## Page/View Analysis & API Mapping

### Complexity Matrix & Effort Estimation

| # | View ID | View Name | Complexity | Data Entry Points | State Management | Data Manipulation | Est. Hours |
|---|---------|-----------|------------|-------------------|------------------|-------------------|------------|
| 1 | `home` | Project Home | Medium | 8 | Local State | Aggregations, KPIs | 12-16 |
| 2 | `dashboard` | Dashboard | Medium | 6 | Local State | Charts, Aggregations | 14-18 |
| 3 | `master_budget` | Contracted Budget | Low | 6 | Local State | Simple CRUD | 8-12 |
| 4 | `mktg_budget` | Marketing Budget | Medium | 10 | Local State | Category CRUD, aggregations | 16-20 |
| 5 | `budget` | Game Budget Grid | **High** | 50+ | Complex Local | Cell editing, selection, paste, FX | 40-50 |
| 6 | `phases` | Phase Management | Medium | 12 | Local State | Drag-drop, timeline | 14-18 |
| 7 | `rollup` | Annual/Phase Rollups | Low | 2 | Read-only | Aggregations | 8-10 |
| 8 | `payment` | Payment Schedule | Low | 2 | Read-only | Timeline aggregations | 6-8 |
| 9 | `ledger` | Game Ledger | **High** | 20+ | Local State | CRUD, sorting, filtering | 30-35 |
| 10 | `mktg_ledger` | Marketing Ledger | **High** | 20+ | Local State | CRUD, sorting, filtering | 28-32 |
| 11 | `ledger_topsheet` | Ledger Top Sheet | Medium | 4 | Read-only | Aggregations, grouping | 12-14 |
| 12 | `reconcile` | QB Reconciliation | **High** | 15 | Local State | Import, matching, allocation | 35-40 |
| 13 | `variance` | Variance Analysis | Medium | 6 | Local State | Comparisons, charts | 16-20 |
| 14 | `insights` | AI Insights | Medium | 5 | Async State | AI API calls | 18-22 |
| 15 | `history` | Change History | Medium | 8 | Read-only | Filtering, snapshots | 14-16 |
| 16 | `amendments` | Amendment Tracker | **High** | 15 | Workflow State | Approval workflow | 28-32 |
| 17 | `invoices` | Invoice Queue | **High** | 12 | Async State | AI scanning, approval | 30-35 |
| 18 | `pnl` | P&L Projection | **High** | 25 | Complex Local | Multi-tier royalties, scenarios | 35-40 |
| 19 | `onesheet` | One Sheet | Medium | 15 | Local State | Print/export | 16-20 |
| 20 | `milestones` | Milestones | Medium | 10 | Local State | Timeline, deliverables | 18-22 |
| 21 | `documents` | Documents | Medium | 6 | Local + File | Upload/download | 16-20 |
| 22 | `sales` | Sales Dashboard | **High** | 15 | Complex State | Multi-platform import, charts | 35-40 |
| 23 | `sales_promos` | Sales Intelligence | Medium | 10 | Local State | Promo calendar | 14-18 |
| 24 | `notes_*` | Notes (Creative/Meeting) | Low | 6 | Local State | Simple CRUD | 10-12 |

### Complexity Breakdown

**HIGH Complexity (8 views - 240+ hours)**
- Budget Grid, Ledgers (2×), Reconciliation, Amendments, Invoices, P&L, Sales

**MEDIUM Complexity (11 views - 180+ hours)**
- Home, Dashboard, Marketing Budget, Phases, Variance, Insights, History, One Sheet, Milestones, Documents, Sales Intelligence

**LOW Complexity (5 views - 60+ hours)**
- Master Budget, Rollups, Payment Schedule, Notes

**Total Estimated Hours: 480-550 hours**  
**Recommended Team Size:** 2-3 full-stack developers  
**Estimated Duration:** 14-18 weeks

---

## Static Data Structures

### Core Project Data (`createDefaultProject()`)

**Location:** Lines 85-127 of fictions-command-center-v0.4.jsx

```javascript
{
  // Identity
  id: uuid(),
  name: "Covenant",
  studio: "Fictions",
  icon: "",
  
  // Timeline
  startDate: "2025-08",
  monthCount: 36,
  glStartDate: "2025-08",
  glMonthCount: 36,
  
  // Phases
  phases: PHASES_DEFAULT, // 9 predefined phases
  phaseAssignment: {}, // month → phase mapping
  glPhases: PHASES_DEFAULT,
  glPhaseAssignment: {},
  
  // Budget Rows
  rows: [
    { id, sectionId, subsectionId, meta, greenlit: {}, live: {} },
    // 15 default rows across employees, contractors, one-offs
  ],
  
  // Ledger
  ledger: [],
  milestonePayment: {},
  
  // Settings
  deptColors: {},
  multiCurrency: false,
  altCurrency: "EUR",
  spotRates: {},
  globalSpotRate: "",
  rowCurrencyOverrides: {},
  
  // History & Audit
  greenlitHistory: [],
  changeHistory: [],
  cellComments: {},
  auditLog: [],
  fxHedges: [],
  
  // Greenlit State
  greenlitLocked: true,
  greenlitVersion: 1,
  greenlitDate: new Date().toISOString().slice(0, 10),
  
  // Metadata
  pointPerson: [],
  platforms: [],
  targetReleaseDate: "",
  
  // Master Budget (High-level)
  masterBudget: {
    developmentCosts: 6000000,
    portingCosts: 600000,
    externalCosts: 1400000,
    reserve: 250000,
    contingency: 1000000,
    marketing: 500000
  },
  
  // Marketing Budget
  marketingBudget: { categories: [], rows: [] }
}
```

### Static Configuration Constants

| Constant | Lines | Purpose | Sample Values |
|----------|-------|---------|----------------|
| `PHASES_DEFAULT` | 22-32 | Development lifecycle phases | Concept, Prototype, VS, Pre-Alpha, Alpha, Beta, RC, Post-Launch, Archive |
| `PHASE_COLORS` | 34 | Color palette for phases | 16 color hex codes |
| `BUDGET_GROUPS` | 36-65 | Budget hierarchy (Group → Section → Subsection) | Development Costs, Porting Costs, External Support, etc. |
| `LEDGER_CATEGORIES` | 286-310 | Ledger category tree | Developer Fees, Engine Fees, QA, Marketing Costs, etc. |
| `CURRENCIES` | 314 | Supported currencies | USD, EUR, GBP, JPY, etc. (20 currencies) |
| `FX_RATES` | 315 | Default exchange rates | Spot rates for currency conversion |
| `LEDGER_STATUSES` | 316 | Invoice workflow states | Draft, Submitted, Approved, Sent to Finance, Paid |
| `STATUS_COLORS` | 317 | Status UI styling | Color mappings per status |
| `DEFAULT_MKTG_CATS` | 284 | Marketing categories | 7 predefined categories |
| `PLATFORM_OPTIONS` | 545 | Gaming platforms | Steam, EGS, PS4/5, Xbox, Switch, iOS, Android, Netflix |

### Auxiliary State Objects

```javascript
salesData: [] // [{id, platform, date, title, units, revenue, wishlists, refunds, region, source}]
wishlistData: [] // [{id, platform, date, title, adds, deletes, purchases, gifts}]
promoData: [] // [{platform, title, startDate, endDate, originalPrice, salePrice, discountPct, notes}]
notesData: { creative: [], meeting: [] }
projectDocs: [] // [{id, name, type, size, data, uploadedAt, uploadedBy}]
milestoneData: {} // {milestoneKey: {deliverables, status, etc.}}
glPnl: {} // P&L projection settings (royalty tiers, scenarios, etc.)
glOneSheet: {} // One-sheet template fields
amendments: [] // [{id, status, requestedBy, reason, budgetDelta, etc.}]
invoiceQueue: [] // [{id, status, fileName, extractedData, reviewedBy, etc.}]
reconItems: [] // QB import items pending reconciliation
```

---

## Required API Endpoints & Database Schema

### A. Projects API

**Endpoints:**
```
GET    /api/v1/projects              - List all projects
GET    /api/v1/projects/:id          - Get single project
POST   /api/v1/projects              - Create project
PUT    /api/v1/projects/:id          - Update project settings
DELETE /api/v1/projects/:id          - Delete project
POST   /api/v1/projects/:id/snapshot - Export snapshot
```

**Database Table: `projects`**
```sql
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  studio VARCHAR(255),
  icon VARCHAR(50),
  description TEXT,
  
  -- Timeline
  start_date DATE NOT NULL DEFAULT CURRENT_DATE,
  month_count INTEGER DEFAULT 36 CHECK (month_count > 0),
  gl_start_date DATE,
  gl_month_count INTEGER,
  target_release_date DATE,
  
  -- Currency
  multi_currency BOOLEAN DEFAULT FALSE,
  alt_currency VARCHAR(3) DEFAULT 'EUR',
  global_spot_rate DECIMAL(10, 6),
  
  -- Greenlit State
  greenlit_locked BOOLEAN DEFAULT TRUE,
  greenlit_version INTEGER DEFAULT 1,
  greenlit_date DATE,
  
  -- Metadata
  point_persons TEXT[], -- Array of user IDs
  platforms TEXT[], -- Array of platform names
  
  -- Master Budget
  master_budget_dev DECIMAL(15, 2) DEFAULT 6000000,
  master_budget_porting DECIMAL(15, 2) DEFAULT 600000,
  master_budget_external DECIMAL(15, 2) DEFAULT 1400000,
  master_budget_reserve DECIMAL(15, 2) DEFAULT 250000,
  master_budget_contingency DECIMAL(15, 2) DEFAULT 1000000,
  master_budget_marketing DECIMAL(15, 2) DEFAULT 500000,
  
  -- Audit
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_by UUID REFERENCES users(id),
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP NULL
);

CREATE INDEX idx_projects_studio ON projects(studio);
CREATE INDEX idx_projects_created_by ON projects(created_by);
```

### B. Budget Rows and Cells API

**Endpoints:**
```
GET    /api/v1/projects/:id/budget/rows         - Get all rows
POST   /api/v1/projects/:id/budget/rows         - Create row
PUT    /api/v1/projects/:id/budget/rows/:rowId  - Update row metadata
DELETE /api/v1/projects/:id/budget/rows/:rowId  - Delete row
PUT    /api/v1/projects/:id/budget/rows/:rowId/cells - Bulk update cells
POST   /api/v1/projects/:id/budget/import/csv   - Import from CSV
```

**Database Tables:**
```sql
CREATE TABLE budget_rows (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  section_id VARCHAR(50) NOT NULL,
  subsection_id VARCHAR(50) NOT NULL,
  
  -- Metadata (flexible JSONB)
  meta JSONB NOT NULL, -- {role, department, name} or {item, category, detail}
  
  -- Currency
  currency_override VARCHAR(3), -- NULL = USD, or "EUR" etc.
  
  sort_order INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CHECK (section_id IS NOT NULL),
  CHECK (subsection_id IS NOT NULL)
);

CREATE INDEX idx_budget_rows_project ON budget_rows(project_id);
CREATE INDEX idx_budget_rows_section ON budget_rows(project_id, section_id, subsection_id);

CREATE TABLE budget_cells (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  row_id UUID NOT NULL REFERENCES budget_rows(id) ON DELETE CASCADE,
  month_key VARCHAR(7) NOT NULL, -- "2025-08"
  mode VARCHAR(10) NOT NULL, -- "live" or "greenlit"
  
  amount DECIMAL(15, 2) DEFAULT 0,
  fx_rate DECIMAL(10, 6), -- Optional FX rate override
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE (row_id, month_key, mode),
  CHECK (mode IN ('live', 'greenlit'))
);

CREATE INDEX idx_budget_cells_project ON budget_cells(project_id);
CREATE INDEX idx_budget_cells_month ON budget_cells(project_id, month_key, mode);
```

### C. Phases API

**Endpoints:**
```
GET    /api/v1/projects/:id/phases              - Get phases (with mode filter)
PUT    /api/v1/projects/:id/phases              - Update all phases (bulk)
PUT    /api/v1/projects/:id/phase-assignment    - Update month→phase mapping
GET    /api/v1/projects/:id/phase-timeline      - Get formatted timeline
```

**Database Tables:**
```sql
CREATE TABLE phases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  name VARCHAR(100) NOT NULL,
  color VARCHAR(7) DEFAULT '#6366f1',
  months INTEGER DEFAULT 1 CHECK (months > 0),
  description TEXT,
  
  mode VARCHAR(10) NOT NULL DEFAULT 'live', -- "live" or "greenlit"
  sort_order INTEGER DEFAULT 0,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CHECK (mode IN ('live', 'greenlit'))
);

CREATE INDEX idx_phases_project_mode ON phases(project_id, mode);

CREATE TABLE phase_assignments (
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  month_key VARCHAR(7) NOT NULL, -- "2025-08"
  phase_id UUID REFERENCES phases(id) ON DELETE SET NULL,
  mode VARCHAR(10) NOT NULL DEFAULT 'live',
  
  PRIMARY KEY (project_id, month_key, mode),
  CHECK (mode IN ('live', 'greenlit'))
);

CREATE INDEX idx_phase_assign_project_mode ON phase_assignments(project_id, mode);
```

### D. Ledger API

**Endpoints:**
```
GET    /api/v1/projects/:id/ledger                  - Get ledger entries (paginated)
POST   /api/v1/projects/:id/ledger                  - Create entry
PUT    /api/v1/projects/:id/ledger/:entryId         - Update entry
DELETE /api/v1/projects/:id/ledger/:entryId         - Delete entry
POST   /api/v1/projects/:id/ledger/:entryId/allocate - Allocate to budget cell
POST   /api/v1/projects/:id/ledger/import/qb        - Import QB export
POST   /api/v1/projects/:id/ledger/scan-invoice     - AI invoice extraction
GET    /api/v1/projects/:id/ledger/reconcile        - Get unreconciled items
```

**Database Tables:**
```sql
CREATE TABLE ledger_entries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  -- Invoice Info
  invoice_num VARCHAR(100),
  payee VARCHAR(255),
  description TEXT,
  
  -- Amount & Currency
  original_amount DECIMAL(15, 2) DEFAULT 0,
  original_currency VARCHAR(3) DEFAULT 'USD',
  total_billed DECIMAL(15, 2) DEFAULT 0,
  total_billed_finalized BOOLEAN DEFAULT FALSE,
  
  -- Budget Allocation
  milestone VARCHAR(100),
  phase_id UUID REFERENCES phases(id) ON DELETE SET NULL,
  
  -- Dates
  invoice_date DATE,
  date_paid DATE,
  
  -- Classification
  category VARCHAR(100),
  status VARCHAR(50) DEFAULT 'Draft', -- Draft, Submitted, Approved, Sent to Finance, Paid
  
  -- QB Reconciliation
  qb_fingerprint VARCHAR(255), -- For deduplication
  qb_matched_at TIMESTAMP,
  
  -- Attachment
  attachment_id UUID ,
  
  -- Audit
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CHECK (status IN ('Draft', 'Submitted', 'Approved', 'Sent to Finance', 'Paid'))
);

CREATE INDEX idx_ledger_project ON ledger_entries(project_id);
CREATE INDEX idx_ledger_status ON ledger_entries(project_id, status);
CREATE INDEX idx_ledger_qb_fp ON ledger_entries(qb_fingerprint, project_id);

CREATE TABLE ledger_allocations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ledger_entry_id UUID NOT NULL REFERENCES ledger_entries(id) ON DELETE CASCADE,
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  budget_row_id UUID NOT NULL REFERENCES budget_rows(id) ON DELETE RESTRICT,
  month_key VARCHAR(7) NOT NULL,
  amount DECIMAL(15, 2) NOT NULL,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE (ledger_entry_id, budget_row_id, month_key)
);

CREATE INDEX idx_alloc_ledger ON ledger_allocations(ledger_entry_id);
CREATE INDEX idx_alloc_row_month ON ledger_allocations(budget_row_id, month_key);

CREATE TABLE ledger_attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ledger_entry_id UUID NOT NULL REFERENCES ledger_entries(id) ON DELETE CASCADE,
  
  file_name VARCHAR(255) NOT NULL,
  file_type VARCHAR(100),
  file_size INTEGER,
  s3_url TEXT, -- S3 URL reference
  
  uploaded_by UUID REFERENCES users(id),
  uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_attach_entry ON ledger_attachments(ledger_entry_id);
```

### E. Sales Data API

**Endpoints:**
```
GET    /api/v1/projects/:id/sales                - Get sales dashboard data
POST   /api/v1/projects/:id/sales/import         - Import CSV from platform
GET    /api/v1/projects/:id/wishlists            - Get wishlist trends
POST   /api/v1/projects/:id/promos               - Create promo
GET    /api/v1/projects/:id/promos               - Get all promos
PUT    /api/v1/projects/:id/promos/:promoId      - Update promo
DELETE /api/v1/projects/:id/promos/:promoId      - Delete promo
```

**Database Tables:**
```sql
CREATE TABLE sales_data (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  platform VARCHAR(50) NOT NULL, -- steam, epic, playstation, xbox, switch
  date DATE,
  title VARCHAR(255),
  
  units INTEGER DEFAULT 0,
  revenue DECIMAL(15, 2) DEFAULT 0,
  wishlists INTEGER DEFAULT 0,
  refunds INTEGER DEFAULT 0,
  
  region VARCHAR(50),
  source VARCHAR(255), -- CSV file name
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sales_project_platform ON sales_data(project_id, platform, date);

CREATE TABLE promos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  platform VARCHAR(50),
  title VARCHAR(255),
  start_date DATE NOT NULL,
  end_date DATE,
  
  original_price DECIMAL(10, 2),
  sale_price DECIMAL(10, 2),
  discount_pct DECIMAL(5, 2),
  
  notes TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_promos_project_dates ON promos(project_id, start_date, end_date);

CREATE TABLE wishlist_data (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  platform VARCHAR(50),
  date DATE,
  title VARCHAR(255),
  
  adds INTEGER DEFAULT 0,
  deletes INTEGER DEFAULT 0,
  purchases INTEGER DEFAULT 0,
  gifts INTEGER DEFAULT 0,
  
  source VARCHAR(255),
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_wishlist_project_date ON wishlist_data(project_id, date);
```

### F. Amendments API

**Endpoints:**
```
GET    /api/v1/projects/:id/amendments          - Get amendments
POST   /api/v1/projects/:id/amendments          - Create amendment draft
PUT    /api/v1/projects/:id/amendments/:id      - Submit/approve/reject
DELETE /api/v1/projects/:id/amendments/:id      - Delete draft
```

**Database Table:**
```sql
CREATE TABLE amendments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  status VARCHAR(50) DEFAULT 'draft', -- draft, submitted, approved, rejected
  
  -- Request Info
  requested_by UUID NOT NULL REFERENCES users(id),
  requested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  reason TEXT,
  
  -- Budget Impact
  current_gl_version INTEGER,
  current_gl_total DECIMAL(15, 2),
  proposed_total DECIMAL(15, 2),
  delta DECIMAL(15, 2),
  
  group_deltas JSONB, -- [{name, greenlit, proposed, delta, sections}]
  term_changes JSONB, -- {notes, devBudgetNew, royaltyChanges}
  
  pnl_completed BOOLEAN DEFAULT FALSE,
  
  -- Review
  submitted_at TIMESTAMP,
  reviewed_by UUID REFERENCES users(id),
  reviewed_at TIMESTAMP,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CHECK (status IN ('draft', 'submitted', 'approved', 'rejected'))
);

CREATE INDEX idx_amendments_project_status ON amendments(project_id, status);
```

### G. Greenlit History API

**Endpoints:**
```
GET    /api/v1/projects/:id/greenlit/history    - Get GL version history
POST   /api/v1/projects/:id/greenlit/save       - Save new GL version
GET    /api/v1/projects/:id/greenlit/compare    - Compare versions
```

**Database Table:**
```sql
CREATE TABLE greenlit_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  version INTEGER NOT NULL,
  date DATE,
  saved_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  saved_by UUID REFERENCES users(id),
  
  -- Snapshot Data
  snapshot JSONB, -- [{rowId, label, greenlit: {month: amount}}]
  total DECIMAL(15, 2),
  prev_total DECIMAL(15, 2),
  
  -- Change Details
  changes JSONB, -- [{rowId, label, oldTotal, newTotal, diff, monthChanges}]
  phase_changes JSONB, -- [{type, name, color, details}]
  phases JSONB, -- [{id, name, color, months}]
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE (project_id, version)
);

CREATE INDEX idx_gl_history_project ON greenlit_history(project_id, version DESC);
```

### H. Additional APIs

**Endpoints:**
```
GET    /api/v1/projects/:id/documents           - List documents
POST   /api/v1/projects/:id/documents           - Upload document
DELETE /api/v1/projects/:id/documents/:docId    - Delete document

GET    /api/v1/projects/:id/notes/:type         - Get notes (creative/meeting)
POST   /api/v1/projects/:id/notes/:type         - Create note
PUT    /api/v1/projects/:id/notes/:noteId       - Update note
DELETE /api/v1/projects/:id/notes/:noteId       - Delete note

GET    /api/v1/projects/:id/audit-log           - Get audit trail
GET    /api/v1/projects/:id/pnl-settings        - Get P&L settings
PUT    /api/v1/projects/:id/pnl-settings        - Update P&L settings

GET    /api/v1/projects/:id/milestones          - Get milestones
PUT    /api/v1/projects/:id/milestones          - Update milestones

POST   /api/v1/ai/chat                          - Chat with AI assistant
POST   /api/v1/ai/scan-invoice                  - Extract invoice data
```

**Database Tables:**
```sql
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  name VARCHAR(255) NOT NULL,
  type VARCHAR(50), -- PDF, DOC, XLSX, etc.
  size INTEGER,
  s3_url TEXT NOT NULL,
  
  uploaded_by UUID REFERENCES users(id),
  uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE (project_id, name)
);

CREATE TABLE notes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  type VARCHAR(50) NOT NULL, -- "creative" or "meeting"
  title VARCHAR(255),
  content TEXT,
  date DATE DEFAULT CURRENT_DATE,
  
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CHECK (type IN ('creative', 'meeting'))
);

CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  action VARCHAR(100),
  detail TEXT,
  extra JSONB,
  
  user_id UUID REFERENCES users(id),
  user_role VARCHAR(50),
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_project_date ON audit_log(project_id, created_at DESC);

CREATE TABLE pnl_settings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE UNIQUE,
  
  retail_price DECIMAL(10, 2),
  discount_rate DECIMAL(5, 4),
  publishing_fee_rate DECIMAL(5, 4),
  engine_fee_rate DECIMAL(5, 4),
  
  licensor_royalty_enabled BOOLEAN DEFAULT FALSE,
  licensor_royalty_rate DECIMAL(5, 4),
  licensor_tiers JSONB, -- [{label, thType, thValue, licensorShare}]
  
  royalty_tiers JSONB, -- [{label, thType, thValue, devShare, pubShare, fictionsShare}]
  roi_targets JSONB, -- Array of percentages
  scenario_units JSONB, -- Array of unit counts
  
  has_pub_partner BOOLEAN DEFAULT FALSE,
  pub_partner_name VARCHAR(255),
  pub_contrib JSONB, -- {devBudget, portingBudget, externalBudget, etc.}
  
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE milestone_data (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  milestone_num INTEGER NOT NULL,
  month_key VARCHAR(7),
  
  deliverables TEXT,
  status VARCHAR(50), -- Pending, In Progress, Complete, etc.
  notes TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE (project_id, milestone_num)
);
```

---

## Implementation Strategy

### Recommended Technology Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Frontend State Management** | TanStack Query (React Query) v5 | Caching, optimistic updates, background refetch, sync across tabs |
| **HTTP Client** | Axios v1.6+ | Interceptors for auth, error handling, request/response transforms |
| **Backend Framework** | Node.js + Express.js | Matches React ecosystem, broad middleware support |
| **Runtime** | Node.js 18+ LTS | Performance, support for async/await |
| **Database** | PostgreSQL 14+ | JSONB for flexible metadata, strong relational support, ACID compliance |
| **ORM / Query Builder** | Prisma v5 or TypeORM | Type safety, migrations, zero-runtime overhead (Prisma) |
| **Authentication** | JWT + Refresh Tokens | Stateless, scalable, works with multi-instance deployments |
| **File Storage** | AWS S3 or Azure Blob Storage | Scalable, CDN-capable, no local filesystem concerns |
| **Background Jobs** | Bull.js (Redis) or pg-boss | Async invoice scanning, QB reconciliation, report generation |
| **Real-time (Optional)** | Socket.io or Hono | Multi-user collaboration, live notifications |
| **API Documentation** | OpenAPI/Swagger + Redoc | Auto-generated, interactive documentation |

### Data Fetching Pattern with TanStack Query

**File: `src/hooks/useProjectData.ts`**

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { projectApi } from '../api/project';

// Query hooks
export function useProject(projectId: string) {
  return useQuery({
    queryKey: ['project', projectId],
    queryFn: () => projectApi.getProject(projectId),
    staleTime: 5 * 60 * 1000, // 5 minutes
    gcTime: 30 * 60 * 1000, // 30 minutes (formerly cacheTime)
  });
}

export function useBudgetRows(projectId: string, enabled = true) {
  return useQuery({
    queryKey: ['project', projectId, 'rows'],
    queryFn: () => projectApi.getBudgetRows(projectId),
    enabled,
    staleTime: 2 * 60 * 1000,
  });
}

// Mutation hooks
export function useUpdateCell() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ projectId, rowId, monthKey, mode, value }) =>
      projectApi.updateCell(projectId, rowId, monthKey, mode, value),
    
    onMutate: async (variables) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({
        queryKey: ['project', variables.projectId, 'rows']
      });
      
      // Snapshot old data
      const previousRows = queryClient.getQueryData(
        ['project', variables.projectId, 'rows']
      );
      
      // Optimistically update
      queryClient.setQueryData(
        ['project', variables.projectId, 'rows'],
        (old) => {
          const newRows = [...old];
          const row = newRows.find(r => r.id === variables.rowId);
          if (row) {
            row[variables.mode] = {
              ...row[variables.mode],
              [variables.monthKey]: variables.value
            };
          }
          return newRows;
        }
      );
      
      return { previousRows };
    },
    
    onError: (err, variables, context) => {
      // Rollback on error
      if (context?.previousRows) {
        queryClient.setQueryData(
          ['project', variables.projectId, 'rows'],
          context.previousRows
        );
      }
    },
    
    onSuccess: (data, variables) => {
      // Re-fetch after success to ensure sync
      queryClient.invalidateQueries({
        queryKey: ['project', variables.projectId, 'rows']
      });
    }
  });
}

export function useBulkUpdateCells() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ projectId, updates }) =>
      projectApi.bulkUpdateCells(projectId, updates),
    onMutate: async (variables) => {
      // Batch optimistic updates
      await queryClient.cancelQueries({
        queryKey: ['project', variables.projectId, 'rows']
      });
      
      const previousRows = queryClient.getQueryData(
        ['project', variables.projectId, 'rows']
      );
      
      queryClient.setQueryData(
        ['project', variables.projectId, 'rows'],
        (old) => {
          let newRows = [...old];
          variables.updates.forEach(({ rowId, monthKey, mode, value }) => {
            const row = newRows.find(r => r.id === rowId);
            if (row) {
              row[mode] = { ...row[mode], [monthKey]: value };
            }
          });
          return newRows;
        }
      );
      
      return { previousRows };
    }
  });
}
```

### Error Handling Pattern

**File: `src/components/ErrorBoundary.tsx`**

```typescript
import { useQueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary as ReactErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }) {
  return (
    <div className="error-container">
      <h2>Something went wrong</h2>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

export function QueryErrorBoundary({ children }) {
  const { reset } = useQueryErrorResetBoundary();
  
  return (
    <ReactErrorBoundary
      onReset={reset}
      FallbackComponent={ErrorFallback}
    >
      {children}
    </ReactErrorBoundary>
  );
}

// Usage
function BudgetGrid() {
  const { data, isLoading, error, refetch } = useBudgetRows(projectId);
  
  if (isLoading) return <GridSkeleton />;
  if (error) return <ErrorState error={error} refetch={refetch} />;
  
  return <Grid data={data} />;
}
```

### Loading States Pattern

**File: `src/components/LoadingStates.tsx`**

```typescript
export const GridSkeleton = () => (
  <div className="skeleton-grid">
    {Array(10).fill(0).map((_, i) => (
      <div key={i} className="skeleton-row animate-pulse" />
    ))}
  </div>
);

export const withLoadingState = (Component, LoadingComponent) => {
  return function WithLoadingState({ isLoading, ...props }) {
    return isLoading ? <LoadingComponent /> : <Component {...props} />;
  };
};

// Usage
const BudgetGridWithLoading = withLoadingState(BudgetGrid, GridSkeleton);
```

### API Client Pattern

**File: `src/api/project.ts`**

```typescript
import axios, { AxiosInstance } from 'axios';

const API_BASE = process.env.REACT_APP_API_URL || 'http://localhost:3001/api/v1';

const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE,
  headers: { 'Content-Type': 'application/json' },
});

// Add auth token to requests
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('accessToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle 401 and refresh token
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Attempt token refresh
      const refreshToken = localStorage.getItem('refreshToken');
      if (refreshToken) {
        try {
          const { data } = await axios.post(`${API_BASE}/auth/refresh`, {
            refreshToken,
          });
          localStorage.setItem('accessToken', data.accessToken);
          error.config.headers.Authorization = `Bearer ${data.accessToken}`;
          return apiClient(error.config);
        } catch {
          // Redirect to login
          window.location.href = '/login';
        }
      }
    }
    return Promise.reject(error);
  }
);

export const projectApi = {
  getProject: (id: string) =>
    apiClient.get(`/projects/${id}`).then(r => r.data),
  
  getBudgetRows: (projectId: string) =>
    apiClient.get(`/projects/${projectId}/budget/rows`).then(r => r.data),
  
  updateCell: (projectId: string, rowId: string, monthKey: string, mode: string, value: number) =>
    apiClient.put(`/projects/${projectId}/budget/rows/${rowId}/cells`, {
      monthKey,
      mode,
      amount: value,
    }).then(r => r.data),
  
  bulkUpdateCells: (projectId: string, updates: any[]) =>
    apiClient.put(`/projects/${projectId}/budget/rows/cells/bulk`, {
      updates,
    }).then(r => r.data),
};
```

---

## Migration Phases

### Phase 1: API Infrastructure & Database (2-3 weeks)

**Deliverables:**
- PostgreSQL database schema (all tables)
- Node.js/Express server setup
- JWT authentication + refresh token flow
- Basic CRUD routes for projects

**Tasks:**
- [ ] Initialize Node.js project with TypeScript
- [ ] Set up Prisma + migrations
- [ ] Create database schema
- [ ] Implement JWT middleware
- [ ] Build GET/POST/PUT/DELETE /projects endpoints
- [ ] Set up error handling middleware
- [ ] Set up CORS for frontend

**Estimated Hours:** 80-100

---

### Phase 2: Core Budget Module (4-5 weeks)

**Deliverables:**
- Budget rows and cells APIs
- CSV import functionality
- Budget grid frontend integration (React Query)
- Live mode fully functional

**Tasks:**
- [ ] Build budget rows CRUD endpoints
- [ ] Build budget cells bulk update endpoint
- [ ] Implement CSV parser on backend
- [ ] Create frontend hooks (useProject, useBudgetRows, useUpdateCell)
- [ ] Add optimistic updates
- [ ] Migrate BudgetGrid component to use API
- [ ] Add error states and loading states
- [ ] Test cell editing, paste, multi-select

**Estimated Hours:** 150-180

---

### Phase 3: Phases & Greenlit (2-3 weeks)

**Deliverables:**
- Phase CRUD APIs
- Phase assignment logic
- Greenlit versioning APIs
- Greenlit history snapshots

**Tasks:**
- [ ] Build phases endpoints
- [ ] Build phase-assignment endpoints
- [ ] Implement greenlit versioning
- [ ] Create greenlit history table
- [ ] Migrate Phases view to API
- [ ] Test phase editing, reordering, GL locking

**Estimated Hours:** 100-120

---

### Phase 4: Ledger & Reconciliation (3-4 weeks)

**Deliverables:**
- Ledger CRUD APIs
- QB import + reconciliation matching
- Allocation system
- File attachment storage (S3)

**Tasks:**
- [ ] Build ledger entries CRUD
- [ ] Build QB import parser
- [ ] Implement fuzzy matching algorithm
- [ ] Build allocation endpoints
- [ ] Set up S3 integration for attachments
- [ ] Migrate Ledger/Reconcile views to API
- [ ] Add QR code for invoice scanning
- [ ] Test reconciliation matching

**Estimated Hours:** 150-170

---

### Phase 5: Sales & Amendments (3-4 weeks)

**Deliverables:**
- Sales data import APIs (multi-platform)
- Amendment approval workflow
- Invoice scanning with AI
- P&L projection endpoints

**Tasks:**
- [ ] Build sales import endpoints (platform-specific parsers)
- [ ] Build amendments workflow (draft → submit → approve/reject)
- [ ] Set up Claude API integration for invoice extraction
- [ ] Build P&L settings APIs
- [ ] Build royalty tier calculation endpoints
- [ ] Migrate Sales/Amendments/Insights views to API
- [ ] Test amendment approval flow

**Estimated Hours:** 160-180

---

### Phase 6: Optimization & Testing (2-3 weeks)

**Deliverables:**
- Performance optimization
- Integration testing suite
- Error handling refinement
- Documentation & deployment guides

**Tasks:**
- [ ] Implement request caching strategies
- [ ] Add database query optimization
- [ ] Build integration test suite (Jest/Supertest)
- [ ] Performance profiling & optimization
- [ ] API documentation (Swagger/OpenAPI)
- [ ] Deployment to staging/production
- [ ] Monitor error rates in production

**Estimated Hours:** 120-150

---

**Total Phase Breakdown:**
- Phase 1: 80-100 hours
- Phase 2: 150-180 hours
- Phase 3: 100-120 hours
- Phase 4: 150-170 hours
- Phase 5: 160-180 hours
- Phase 6: 120-150 hours
- **Total: 760-900 hours** (vs. 480-550 for individual components due to infrastructure overhead)

---

## Technical Notes per View

### 1. **`budget` (Game Budget Grid) - HIGH PRIORITY**

**Challenges & Solutions:**

| Challenge | Solution | Implementation |
|-----------|----------|-----------------|
| Real-time cell editing | Debounced mutations + optimistic updates | Send updates 500ms after last keystroke |
| Multi-select with paste | Batch update endpoint | Collect updates, POST once |
| FX rate calculations | Server-side dual currency | Store native + USD, recalculate on rate change |
| Large grid rendering | Virtual scrolling + memoization | React-window for rows, useMemo for cells |
| Concurrent edits (multi-user) | Conflict resolution strategy | Last-write-wins or server-side validation |

**API Design:**
```typescript
// Single cell update (debounced)
PUT /api/v1/projects/:id/budget/rows/:rowId/cells
{
  monthKey: "2025-08",
  mode: "live",
  amount: 150000
}

// Bulk update (paste)
PUT /api/v1/projects/:id/budget/rows/cells/bulk
{
  updates: [
    { rowId: "...", monthKey: "2025-08", mode: "live", amount: 100000 },
    { rowId: "...", monthKey: "2025-09", mode: "live", amount: 110000 },
    ...
  ]
}

// Get with aggregations
GET /api/v1/projects/:id/budget/rows?aggregate=true
```

**Frontend Pattern:**
```typescript
const useGridUpdates = (projectId) => {
  const updateCell = useUpdateCell();
  const debouncedUpdate = useRef(null);
  
  return (rowId, monthKey, mode, value) => {
    // Debounce: cancel previous timer
    if (debouncedUpdate.current) clearTimeout(debouncedUpdate.current);
    
    // Optimistic update client-side immediately
    setLocalValue(value);
    
    // Debounce server update
    debouncedUpdate.current = setTimeout(() => {
      updateCell.mutate({ projectId, rowId, monthKey, mode, value });
    }, 500);
  };
};
```

---

### 2. **`ledger` / `mktg_ledger` & `reconcile` - HIGH PRIORITY**

**Challenges & Solutions:**

| Challenge | Solution | Implementation |
|-----------|----------|-----------------|
| QB import parsing (multiple formats) | CSV parser with format detection | Regex detection for headers, adaptive column mapping |
| Fuzzy matching | Scoring algorithm | Amount + name matching, Levenshtein distance |
| Large import files | Streaming + chunking | Process in 100-row batches, background job queue |
| Duplicate detection | Fingerprint + qbFingerprint | md5(date+num+name+amount) |

**API Design:**
```typescript
// QB Import
POST /api/v1/projects/:id/ledger/import/qb
Body: FormData with CSV file
Response: {
  jobId: "...",
  itemsFound: 500,
  suggestedMatches: [{qbId, ledgerId, score}]
}

// Confirm reconciliation
POST /api/v1/projects/:id/ledger/reconcile/confirm
{
  matches: [
    { qbItemId, ledgerEntryId, action: "match" | "new" }
  ]
}

// Allocate ledger entry to budget
POST /api/v1/projects/:id/ledger/:entryId/allocate
{
  budgetRowId, monthKey, amount
}
```

**QB Matching Algorithm:**
```typescript
const scoreMatch = (qbItem, ledgerEntry) => {
  let score = 0;
  
  // Exact amount match (strongest signal)
  const qbAmt = Math.abs(qbItem.amount);
  const leAmt = Math.abs(ledgerEntry.totalBilled);
  if (qbAmt === leAmt && qbAmt > 0) score += 50;
  else if (Math.abs(qbAmt - leAmt) / Math.max(qbAmt, leAmt) < 0.01) score += 40;
  
  // Name/payee match
  const qbName = qbItem.name?.toLowerCase() || '';
  const leName = ledgerEntry.payee?.toLowerCase() || '';
  if (qbName && leName && qbName === leName) score += 30;
  else {
    const words = qbName.split(/\s+/);
    const matches = words.filter(w => w.length > 2 && leName.includes(w));
    if (matches.length > 0) score += 15;
  }
  
  // Invoice number match
  if (qbItem.num && ledgerEntry.invoice && qbItem.num === ledgerEntry.invoice)
    score += 20;
  
  // Date proximity (within 3 days)
  const qbDate = new Date(qbItem.date);
  const leDate = new Date(ledgerEntry.invoiceDate);
  const daysDiff = Math.abs((qbDate - leDate) / (1000 * 60 * 60 * 24));
  if (daysDiff <= 3) score += 10;
  
  return score;
};
```

---

### 3. **`pnl` (P&L Projection) - HIGH PRIORITY**

**Challenges & Solutions:**

| Challenge | Solution | Implementation |
|-----------|----------|-----------------|
| Complex royalty tiers | Server-side computation | Pre-calculate scenarios, cache results |
| Multi-tier scenarios | Scenario matrix | Generate 9 unit scenarios × 7 ROI targets |
| Performance with large budgets | Memoization + server caching | Redis cache scenarios |

**Backend Logic Example:**
```typescript
// Calculate net revenue for scenario
const calculateNetRevenue = (unitsSold, settings) => {
  const gross = unitsSold * settings.retailPrice;
  const discounted = gross * (1 - settings.discountRate);
  
  // Platform fee
  const platformFee = discounted * settings.platformFee;
  
  // Engine fee
  const afterEngine = (discounted - platformFee) * (1 - settings.engineFee);
  
  // Licensor royalty (if applicable)
  let licensorRoyalty = 0;
  if (settings.licensorRoyaltyEnabled) {
    const tier = findLicensorTier(unitsSold, settings.licensorTiers);
    licensorRoyalty = afterEngine * tier.licensorShare;
  }
  
  const netBeforeRoyalty = afterEngine - licensorRoyalty;
  
  // Developer/Publisher/Fictions split (via royalty tiers)
  const tier = findRoyaltyTier(unitsSold, settings.royaltyTiers);
  
  return {
    gross,
    discounted,
    platformFee,
    engineFee: afterEngine - platformFee - licensorRoyalty,
    licensorRoyalty,
    devShare: netBeforeRoyalty * tier.devShare,
    pubShare: netBeforeRoyalty * tier.pubShare,
    fictionsShare: netBeforeRoyalty * tier.fictionsShare,
    roi: ((netBeforeRoyalty - budget) / budget * 100)
  };
};
```

---

### 4. **`sales` - HIGH PRIORITY**

**Challenges & Solutions:**

| Challenge | Solution | Implementation |
|-----------|----------|-----------------|
| Multi-platform CSV formats | Platform-specific parsers | Steam vs Epic vs PS vs Xbox (different headers) |
| Date parsing (MM/DD/YYYY vs YYYY-MM-DD) | Flexible date parser | Try multiple formats, fallback to ISO |
| Large datasets | Pagination + virtualization | Fetch in 1000-row chunks, React-window |
| Real-time dashboards | WebSocket updates | Emit event on new sales, client subscribes |

**Platform CSV Mapping:**
```typescript
const PLATFORM_PARSERS = {
  steam: {
    dateCol: 'Date',
    unitsCol: 'Net Units Sold',
    revenueCol: 'Net Steam Sales',
    regionCol: 'Country',
  },
  epic: {
    dateCol: 'Date',
    unitsCol: 'Units Acquired',
    revenueCol: 'Net Revenue',
    regionCol: 'Market',
  },
  playstation: {
    dateCol: 'Month Start Date',
    unitsCol: 'Sales Quantity',
    revenueCol: 'Sales Exc Tax $',
    regionCol: 'Country/Region',
  },
  xbox: {
    dateCol: 'Period',
    unitsCol: 'Acquisitions',
    revenueCol: 'Revenue',
    regionCol: 'Market',
  },
};
```

---

### 5. **`amendments` - MEDIUM PRIORITY**

**Challenges & Solutions:**

| Challenge | Solution | Implementation |
|-----------|----------|-----------------|
| Approval workflow state machine | Explicit states + transitions | Draft → Submitted → Approved/Rejected |
| Budget snapshots | Full row snapshots | Store JSON before/after |
| Version tracking | Increment GL version | Guard by lock state |

**State Machine:**
```
Draft
  ├─ [Submit] → Submitted
  ├─ [Delete] → Deleted
  └─ [Edit] → Draft

Submitted
  ├─ [Approve] → Approved → [Auto-GL Save] → GL v++
  ├─ [Reject] → Rejected
  └─ [Recall] → Draft

Approved / Rejected
  └─ [Delete] → Deleted
```

---

## Summary & Recommendations

### Project Scope Overview

| Category | Count | Hours | Priority |
|----------|-------|-------|----------|
| High Complexity Views | 8 | 240+ | **CRITICAL** |
| Medium Complexity Views | 11 | 180+ | **HIGH** |
| Low Complexity Views | 5 | 60+ | **MEDIUM** |
| Database Tables | 15 | - | **CRITICAL** |
| API Endpoints | 50+ | - | **CRITICAL** |

### Total Effort

- **Backend Development:** 380-450 hours
- **Frontend Integration:** 200-250 hours
- **Testing & QA:** 100-120 hours
- **Documentation & DevOps:** 80-100 hours
- **Total:** 760-920 hours (18-23 weeks, 1 FTE)

### Recommended Approach

#### Option A: Sequential (Safe, Predictable)
- **Duration:** 18-22 weeks
- **Team:** 1-2 Full Stack Developers
- **Phases:** 6 sequential phases
- **Risk:** Low (clear milestones)
- **Cost:** Lowest

#### Option B: Parallel (Faster, Higher Risk)
- **Duration:** 12-15 weeks
- **Team:** 2-3 Full Stack Developers + 1 QA
- **Phases:** Phase 1 (shared), then parallel Phase 2-5
- **Risk:** Medium (integration complexity)
- **Cost:** Higher (more team members)

#### Option C: Hybrid (Balanced)
- **Duration:** 14-17 weeks
- **Team:** 2 Full Stack Developers + 1 Contractor (QA/Docs)
- **Phases:** Phase 1-2 sequential, Phase 3-5 with some parallelization
- **Risk:** Low-Medium
- **Cost:** Moderate

### Recommended: **Option C (Hybrid)**

**Rationale:**
- Phases 1-2 must be sequential (dependency on infrastructure)
- Phases 3-5 can be parallelized (ledger/amendments/sales are independent)
- One dedicated QA keeps integration smooth
- Cost-effective without excessive risk

### Success Criteria

- [ ] 100% downtime during migration: **0 hours** (parallel infrastructure)
- [ ] Backward compatibility: **100%** (preserve all data)
- [ ] API test coverage: **>80%**
- [ ] Frontend test coverage: **>60%**
- [ ] Performance: Page load **<2 seconds**, Cell update **<500ms**
- [ ] Data integrity: **Zero lost** transactions during migration
- [ ] User training: Minimal (UI remains the same)

---

## Appendix: File Locations Reference

### Frontend Components (Lines in JSX)
- Project Home: 3423-3625
- Dashboard: 3625-3685
- Master Budget: 3685-3806
- Marketing Budget: 3806-4061
- Budget Grid: 4061-5189
- Phases: 4955-5189
- Rollups: 5189-5340
- Payment: 5340-5702
- Ledger: 5702-5898
- Reconciliation: 5898-6298
- Variance: 6298-6411
- Amendments: 6685-7071
- Invoices: 7071-7456
- P&L: 7456-7807
- Sales: 8522-end

### Data Structure Definitions
- Default Project: Lines 85-127
- Budget Groups: Lines 36-65
- Phases: Lines 22-32
- Ledger Categories: Lines 286-310
- Currency Configuration: Lines 314-317

---

**Document Generated:** April 20, 2026  
**This audit provides a comprehensive blueprint for transitioning the Fictions Command Center from a static React application to a dynamic, backend-driven architecture with full API coverage, database schema, and implementation guidance.**

