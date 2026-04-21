# API Endpoint → Database Table CRUD Mapping
## Fictions Command Center v0.4

**Legend:** **R** = Read (SELECT) | **C** = Create (INSERT) | **U** = Update (UPDATE) | **D** = Delete (DELETE)

---

## Table of Contents
1. [Authentication API](#1-authentication-api)
2. [Projects API](#2-projects-api)
3. [Budget Rows & Cells API](#3-budget-rows--cells-api)
4. [Greenlit API](#4-greenlit-api)
5. [Phases API](#5-phases-api)
6. [Ledger API](#6-ledger-api)
7. [Reconciliation API](#7-reconciliation-api)
8. [Sales & Wishlist API](#8-sales--wishlist-api)
9. [Promos API](#9-promos-api)
10. [Amendments API](#10-amendments-api)
11. [Invoice Queue API](#11-invoice-queue-api)
12. [Documents API](#12-documents-api)
13. [Notes API](#13-notes-api)
14. [P&L Settings API](#14-pl-settings-api)
15. [One Sheet API](#15-one-sheet-api)
16. [Milestones API](#16-milestones-api)
17. [Master Budget API](#17-master-budget-api)
18. [Marketing Budget API](#18-marketing-budget-api)
19. [Change History & Audit API](#19-change-history--audit-api)
20. [Cell Comments API](#20-cell-comments-api)
21. [FX & Currency API](#21-fx--currency-api)
22. [Month Close API](#22-month-close-api)
23. [Change Log (Dev Mode) API](#23-change-log-dev-mode-api)
24. [AI Services API](#24-ai-services-api)
25. [Summary Matrix](#25-summary-matrix-all-tables--all-endpoints)

---

## 1. Authentication API

| # | Method | Endpoint | `users` | `projects` | Notes |
|---|--------|----------|---------|------------|-------|
| 1 | POST | `/api/v1/auth/login` | **R** | — | Validate credentials, return JWT + refresh token |
| 2 | POST | `/api/v1/auth/refresh` | **R** | — | Validate refresh token, issue new JWT |
| 3 | GET | `/api/v1/auth/me` | **R** | **R** | Return current user profile + accessible projects |
| 4 | POST | `/api/v1/auth/register` | **C** | — | Create new user account |
| 5 | PUT | `/api/v1/auth/password` | **U** | — | Change password |

---

## 2. Projects API

| # | Method | Endpoint | `projects` | `budget_rows` | `budget_cells` | `phases` | `phase_assignments` | `audit_log` | Notes |
|---|--------|----------|-----------|---------------|----------------|----------|---------------------|-------------|-------|
| 6 | GET | `/api/v1/projects` | **R** | — | — | — | — | — | List all accessible projects |
| 7 | GET | `/api/v1/projects/:id` | **R** | — | — | — | — | — | Get project details + settings |
| 8 | POST | `/api/v1/projects` | **C** | **C** | **C** | **C** | **C** | **C** | Create project with defaults (15 default rows, 9 default phases) |
| 9 | PUT | `/api/v1/projects/:id` | **U** | — | — | — | — | **C** | Update project settings (name, studio, dates, platforms, etc.) |
| 10 | DELETE | `/api/v1/projects/:id` | **D** | **D** | **D** | **D** | **D** | **C** | Soft-delete project (cascades) |
| 11 | POST | `/api/v1/projects/:id/snapshot` | **R** | **R** | **R** | **R** | **R** | — | Export full project state as JSON snapshot |

---

## 3. Budget Rows & Cells API

| # | Method | Endpoint | `budget_rows` | `budget_cells` | `cell_comments` | `change_history` | `audit_log` | Notes |
|---|--------|----------|---------------|----------------|-----------------|------------------|-------------|-------|
| 12 | GET | `/api/v1/projects/:id/budget/rows` | **R** | **R** | **R** | — | — | Get all rows with cell values & comment counts |
| 13 | POST | `/api/v1/projects/:id/budget/rows` | **C** | — | — | — | **C** | Create new budget row |
| 14 | PUT | `/api/v1/projects/:id/budget/rows/:rowId` | **U** | — | — | — | **C** | Update row metadata (name, dept, etc.) |
| 15 | DELETE | `/api/v1/projects/:id/budget/rows/:rowId` | **D** | **D** | **D** | — | **C** | Delete row + its cells + its comments |
| 16 | POST | `/api/v1/projects/:id/budget/rows/:rowId/duplicate` | **C** | **C** | — | — | **C** | Duplicate row with all cell values |
| 17 | PUT | `/api/v1/projects/:id/budget/rows/reorder` | **U** | — | — | — | — | Drag-drop reorder (update sort_order) |
| 18 | PUT | `/api/v1/projects/:id/budget/rows/:rowId/cells` | — | **C**/**U** | — | — | **C** | Upsert cells for a row (single or bulk month update) |
| 19 | POST | `/api/v1/projects/:id/budget/import/csv` | **C**/**D** | **C**/**D** | — | **C** | **C** | CSV import — replace mode deletes existing; append mode adds |

---

## 4. Greenlit API

| # | Method | Endpoint | `budget_cells` | `greenlit_history` | `change_history` | `audit_log` | `phases` | `phase_assignments` | Notes |
|---|--------|----------|----------------|--------------------|------------------|-------------|----------|---------------------|-------|
| 20 | POST | `/api/v1/projects/:id/greenlit/save` | **R**/**U** | **C** | **C** | **C** | **R** | **R** | Save new GL version: copy live→greenlit cells, snapshot rows |
| 21 | GET | `/api/v1/projects/:id/greenlit/history` | — | **R** | — | — | — | — | Get all GL version snapshots |
| 22 | GET | `/api/v1/projects/:id/greenlit/compare` | — | **R** | — | — | — | — | Compare two GL versions side-by-side |

---

## 5. Phases API

| # | Method | Endpoint | `phases` | `phase_assignments` | `audit_log` | Notes |
|---|--------|----------|----------|---------------------|-------------|-------|
| 23 | GET | `/api/v1/projects/:id/phases` | **R** | **R** | — | Get phases with mode filter (live/greenlit) |
| 24 | PUT | `/api/v1/projects/:id/phases` | **C**/**U**/**D** | **U** | **C** | Bulk update phases (add/edit/remove/reorder); auto-rebuilds assignments |
| 25 | PUT | `/api/v1/projects/:id/phase-assignment` | — | **U** | — | Manually override month→phase mapping |
| 26 | GET | `/api/v1/projects/:id/phase-timeline` | **R** | **R** | — | Get formatted phase timeline for display |

---

## 6. Ledger API

| # | Method | Endpoint | `ledger_entries` | `ledger_allocations` | `ledger_attachments` | `budget_rows` | `budget_cells` | `audit_log` | Notes |
|---|--------|----------|------------------|----------------------|----------------------|---------------|----------------|-------------|-------|
| 27 | GET | `/api/v1/projects/:id/ledger` | **R** | **R** | **R** | **R** | — | — | Paginated ledger entries with allocations & attachments; row names for allocation labels |
| 28 | POST | `/api/v1/projects/:id/ledger` | **C** | — | — | — | — | **C** | Create ledger entry |
| 29 | PUT | `/api/v1/projects/:id/ledger/:entryId` | **U** | — | — | — | — | **C** | Update entry fields (payee, amount, status, etc.) |
| 30 | DELETE | `/api/v1/projects/:id/ledger/:entryId` | **D** | **D** | **D** | — | — | **C** | Delete entry + cascade allocations & attachments |
| 31 | POST | `/api/v1/projects/:id/ledger/:entryId/duplicate` | **C** | **C** | — | — | — | **C** | Duplicate entry with allocations |
| 32 | POST | `/api/v1/projects/:id/ledger/:entryId/allocate` | — | **C**/**U** | — | **R** | **R** | **C** | Allocate ledger entry to budget cells (linkedCells) |
| 33 | POST | `/api/v1/projects/:id/ledger/:entryId/attachment` | — | — | **C** | — | — | — | Upload file attachment (→ S3) |
| 34 | DELETE | `/api/v1/projects/:id/ledger/:entryId/attachment/:attachId` | — | — | **D** | — | — | — | Remove attachment |

---

## 7. Reconciliation API

| # | Method | Endpoint | `ledger_entries` | `ledger_allocations` | `budget_rows` | `budget_cells` | `closed_months` | `change_history` | `change_log` | `audit_log` | Notes |
|---|--------|----------|------------------|----------------------|---------------|----------------|-----------------|------------------|--------------|-------------|-------|
| 35 | POST | `/api/v1/projects/:id/ledger/import/qb` | **R**/**C**/**U** | — | — | — | — | — | — | **C** | Import QB CSV: match existing entries (update status/fingerprint), create new ones |
| 36 | GET | `/api/v1/projects/:id/ledger/reconcile` | **R** | **R** | **R** | **R** | **R** | — | — | — | Get unreconciled items with suggested allocations |
| 37 | POST | `/api/v1/projects/:id/reconcile/commit` | **R**/**U** | **C**/**U** | **C** | **U** | — | **C** | **C** | **C** | Commit reconciliation: allocate entries, update/create budget cells, optionally create new rows |
| 38 | PUT | `/api/v1/projects/:id/months/:monthKey/close` | — | — | — | — | **C**/**U** | **C** | — | **C** | Close month (mark as reconciled) |
| 39 | PUT | `/api/v1/projects/:id/months/:monthKey/reopen` | — | — | — | — | **U** | — | — | **C** | Reopen previously closed month |

---

## 8. Sales & Wishlist API

| # | Method | Endpoint | `sales_data` | `wishlist_data` | Notes |
|---|--------|----------|--------------|-----------------|-------|
| 40 | GET | `/api/v1/projects/:id/sales` | **R** | **R** | Aggregated sales dashboard (all platforms) |
| 41 | GET | `/api/v1/projects/:id/sales/:platform` | **R** | **R** | Platform-specific sales data (steam, xbox, ps, switch, epic) |
| 42 | POST | `/api/v1/projects/:id/sales/import` | **C** | — | Import sales CSV for a platform |
| 43 | DELETE | `/api/v1/projects/:id/sales/:platform` | **D** | — | Clear all sales data for a platform |
| 44 | GET | `/api/v1/projects/:id/wishlists` | — | **R** | Get wishlist trend data |
| 45 | POST | `/api/v1/projects/:id/wishlists/import` | — | **C** | Import wishlist CSV for a platform |
| 46 | DELETE | `/api/v1/projects/:id/wishlists/:platform` | — | **D** | Clear wishlist data for a platform |

---

## 9. Promos API

| # | Method | Endpoint | `promos` | `sales_data` | `wishlist_data` | `ledger_entries` | Notes |
|---|--------|----------|----------|--------------|-----------------|------------------|-------|
| 47 | GET | `/api/v1/projects/:id/promos` | **R** | **R** | **R** | **R** | Get promos with correlated sales/wishlist/marketing data |
| 48 | POST | `/api/v1/projects/:id/promos` | **C** | — | — | — | Create promo period |
| 49 | PUT | `/api/v1/projects/:id/promos/:promoId` | **U** | — | — | — | Update promo details |
| 50 | DELETE | `/api/v1/projects/:id/promos/:promoId` | **D** | — | — | — | Delete promo |

---

## 10. Amendments API

| # | Method | Endpoint | `amendments` | `budget_cells` | `budget_rows` | `greenlit_history` | `change_history` | `change_log` | `audit_log` | Notes |
|---|--------|----------|--------------|----------------|---------------|--------------------|------------------|--------------|-------------|-------|
| 51 | GET | `/api/v1/projects/:id/amendments` | **R** | — | — | **R** | — | **R** | — | List amendments with change log entries |
| 52 | POST | `/api/v1/projects/:id/amendments` | **C** | — | — | — | — | — | **C** | Create amendment draft |
| 53 | PUT | `/api/v1/projects/:id/amendments/:amdId` (submit) | **U** | — | — | — | — | — | **C** | Submit amendment for review (status→submitted) |
| 54 | PUT | `/api/v1/projects/:id/amendments/:amdId` (approve) | **U** | **U** | **U** | — | **C** | **U** | **C** | Approve amendment: apply budget deltas to greenlit cells/rows |
| 55 | PUT | `/api/v1/projects/:id/amendments/:amdId` (reject) | **U** | — | — | — | — | — | **C** | Reject amendment (status→rejected) |
| 56 | DELETE | `/api/v1/projects/:id/amendments/:amdId` | **D** | — | — | — | — | — | **C** | Delete draft amendment |
| 57 | PUT | `/api/v1/projects/:id/change-log/:batchId/approve` | — | **U** | — | — | — | **U** | **C** | Approve dev-mode budget change batch |
| 58 | PUT | `/api/v1/projects/:id/change-log/:batchId/reject` | — | — | — | — | — | **U** | **C** | Reject dev-mode budget change batch |

---

## 11. Invoice Queue API

| # | Method | Endpoint | `invoice_queue` | `ledger_entries` | `ledger_allocations` | `audit_log` | Notes |
|---|--------|----------|-----------------|------------------|----------------------|-------------|-------|
| 59 | GET | `/api/v1/projects/:id/invoices` | **R** | — | — | — | List invoice queue items |
| 60 | POST | `/api/v1/projects/:id/invoices` | **C** | — | — | — | Upload invoice file to queue |
| 61 | POST | `/api/v1/projects/:id/invoices/:invId/scan` | **U** | — | — | — | AI OCR scan → extract data (via Anthropic API) |
| 62 | PUT | `/api/v1/projects/:id/invoices/:invId/approve` | **U** | **C** | **C** | **C** | Approve invoice: create ledger entries per line item |
| 63 | PUT | `/api/v1/projects/:id/invoices/:invId/reject` | **U** | — | — | **C** | Reject invoice |

---

## 12. Documents API

| # | Method | Endpoint | `documents` | `audit_log` | Notes |
|---|--------|----------|-------------|-------------|-------|
| 64 | GET | `/api/v1/projects/:id/documents` | **R** | — | List project documents |
| 65 | POST | `/api/v1/projects/:id/documents` | **C** | **C** | Upload document (→ S3, max 4MB) |
| 66 | GET | `/api/v1/projects/:id/documents/:docId/download` | **R** | — | Download document file |
| 67 | DELETE | `/api/v1/projects/:id/documents/:docId` | **D** | **C** | Delete document + S3 object |

---

## 13. Notes API

| # | Method | Endpoint | `notes` | `audit_log` | Notes |
|---|--------|----------|---------|-------------|-------|
| 68 | GET | `/api/v1/projects/:id/notes/:type` | **R** | — | Get notes by type (creative / meeting) |
| 69 | POST | `/api/v1/projects/:id/notes/:type` | **C** | — | Create note |
| 70 | PUT | `/api/v1/projects/:id/notes/:noteId` | **U** | — | Update note content/title/tags/pinned |
| 71 | DELETE | `/api/v1/projects/:id/notes/:noteId` | **D** | — | Delete note |

---

## 14. P&L Settings API

| # | Method | Endpoint | `pnl_settings` | `projects` | Notes |
|---|--------|----------|----------------|-----------|-------|
| 72 | GET | `/api/v1/projects/:id/pnl-settings` | **R** | **R** | Get P&L model (royalty tiers, scenarios, pub partner, etc.) + master budget for sync |
| 73 | PUT | `/api/v1/projects/:id/pnl-settings` | **C**/**U** | — | Upsert P&L settings (all fields) |

---

## 15. One Sheet API

| # | Method | Endpoint | `gl_one_sheet` | `pnl_settings` | `budget_rows` | `budget_cells` | `projects` | Notes |
|---|--------|----------|----------------|----------------|---------------|----------------|-----------|-------|
| 74 | GET | `/api/v1/projects/:id/onesheet` | **R** | **R** | **R** | **R** | **R** | Get one-sheet with computed financials |
| 75 | PUT | `/api/v1/projects/:id/onesheet` | **C**/**U** | — | — | — | — | Update one-sheet fields |

---

## 16. Milestones API

| # | Method | Endpoint | `milestone_data` | `budget_rows` | `budget_cells` | `phases` | `phase_assignments` | Notes |
|---|--------|----------|------------------|---------------|----------------|----------|---------------------|-------|
| 76 | GET | `/api/v1/projects/:id/milestones` | **R** | **R** | **R** | **R** | **R** | Get milestones with budget totals per month |
| 77 | PUT | `/api/v1/projects/:id/milestones` | **C**/**U** | — | — | — | — | Update milestone deliverables, status, notes |

---

## 17. Master Budget API

| # | Method | Endpoint | `projects` | Notes |
|---|--------|----------|-----------|-------|
| 78 | GET | `/api/v1/projects/:id/master-budget` | **R** | Get master budget caps (stored in projects table) |
| 79 | PUT | `/api/v1/projects/:id/master-budget` | **U** | Update master budget (devCosts, porting, external, reserve, contingency, marketing) |

---

## 18. Marketing Budget API

| # | Method | Endpoint | `marketing_budget_categories` | `marketing_budget_rows` | `audit_log` | Notes |
|---|--------|----------|-------------------------------|-------------------------|-------------|-------|
| 80 | GET | `/api/v1/projects/:id/marketing-budget` | **R** | **R** | — | Get all marketing categories + rows |
| 81 | POST | `/api/v1/projects/:id/marketing-budget/categories` | **C** | — | — | Add marketing category |
| 82 | PUT | `/api/v1/projects/:id/marketing-budget/categories/:catId` | **U** | — | — | Rename category or update budget |
| 83 | DELETE | `/api/v1/projects/:id/marketing-budget/categories/:catId` | **D** | **D** | — | Delete category + its rows |
| 84 | POST | `/api/v1/projects/:id/marketing-budget/rows` | — | **C** | — | Add row to category |
| 85 | PUT | `/api/v1/projects/:id/marketing-budget/rows/:rowId` | — | **U** | — | Update row item name or monthly values |
| 86 | DELETE | `/api/v1/projects/:id/marketing-budget/rows/:rowId` | — | **D** | — | Delete marketing budget row |

---

## 19. Change History & Audit API

| # | Method | Endpoint | `change_history` | `audit_log` | `budget_rows` | `budget_cells` | `ledger_entries` | `phases` | Notes |
|---|--------|----------|------------------|-------------|---------------|----------------|------------------|----------|-------|
| 87 | GET | `/api/v1/projects/:id/history` | **R** | — | — | — | — | — | Get change history (snapshots list) |
| 88 | POST | `/api/v1/projects/:id/history/snapshot` | **C** | **C** | **R** | **R** | **R** | **R** | Create manual snapshot of full project state |
| 89 | POST | `/api/v1/projects/:id/history/:snapshotId/restore` | **R** | **C** | **U** | **U** | **U** | **U** | Restore project from snapshot (replaces all data) |
| 90 | GET | `/api/v1/projects/:id/audit-log` | — | **R** | — | — | — | — | Get audit trail (paginated) |

---

## 20. Cell Comments API

| # | Method | Endpoint | `cell_comments` | Notes |
|---|--------|----------|-----------------|-------|
| 91 | GET | `/api/v1/projects/:id/budget/rows/:rowId/comments/:monthKey` | **R** | Get comments for a specific cell |
| 92 | POST | `/api/v1/projects/:id/budget/rows/:rowId/comments/:monthKey` | **C** | Add comment to a cell |
| 93 | DELETE | `/api/v1/projects/:id/comments/:commentId` | **D** | Delete a comment |

---

## 21. FX & Currency API

| # | Method | Endpoint | `spot_rates` | `fx_hedges` | `budget_rows` | `projects` | Notes |
|---|--------|----------|-------------|-------------|---------------|-----------|-------|
| 94 | GET | `/api/v1/projects/:id/fx/rates` | **R** | — | — | **R** | Get spot rates + global rate + alt currency |
| 95 | PUT | `/api/v1/projects/:id/fx/rates` | **U** | — | — | **U** | Update spot rates per month, global rate |
| 96 | GET | `/api/v1/projects/:id/fx/hedges` | — | **R** | — | — | Get FX hedge contracts |
| 97 | PUT | `/api/v1/projects/:id/fx/hedges` | — | **C**/**U** | — | — | Upsert FX hedge entries |
| 98 | PUT | `/api/v1/projects/:id/budget/rows/:rowId/currency` | — | — | **U** | — | Set per-row currency override |

---

## 22. Month Close API

| # | Method | Endpoint | `closed_months` | `change_history` | `audit_log` | Notes |
|---|--------|----------|-----------------|------------------|-------------|-------|
| 99 | GET | `/api/v1/projects/:id/months/status` | **R** | — | — | Get all months with open/closed status |
| 100 | PUT | `/api/v1/projects/:id/months/:monthKey/close` | **C**/**U** | **C** | **C** | Close month (same as #38, listed here for completeness) |
| 101 | PUT | `/api/v1/projects/:id/months/:monthKey/reopen` | **U** | — | **C** | Reopen month (same as #39) |

---

## 23. Change Log (Dev Mode) API

| # | Method | Endpoint | `change_log` | `pending_changes` | `budget_cells` | `audit_log` | Notes |
|---|--------|----------|--------------|--------------------|--------------------|-------------|-------|
| 102 | GET | `/api/v1/projects/:id/change-log` | **R** | **R** | — | — | Get pending & processed change batches |
| 103 | POST | `/api/v1/projects/:id/change-log` | — | **C** | — | — | Submit budget change request (dev-mode user) |
| 104 | PUT | `/api/v1/projects/:id/change-log/:batchId/approve` | **U** | **D** | **U** | **C** | Approve batch: apply changes to budget cells |
| 105 | PUT | `/api/v1/projects/:id/change-log/:batchId/reject` | **U** | **D** | — | **C** | Reject batch: discard pending changes |

---

## 24. AI Services API

| # | Method | Endpoint | `invoice_queue` | `notes` | `audit_log` | Notes |
|---|--------|----------|-----------------|---------|-------------|-------|
| 106 | POST | `/api/v1/ai/chat` | — | — | **C** | AI chat — proxied to Anthropic Claude API (budget analysis) |
| 107 | POST | `/api/v1/ai/scan-invoice` | **U** | — | **C** | AI invoice OCR — extracts line items from uploaded document |
| 108 | POST | `/api/v1/ai/process-transcript` | — | **U** | — | AI meeting transcript → structured notes with action items |

---

## 25. Summary Matrix: All Tables × All Endpoints

### Table CRUD Coverage

| # | Table | Read By (Endpoint #) | Created By (Endpoint #) | Updated By (Endpoint #) | Deleted By (Endpoint #) |
|---|-------|----------------------|-------------------------|-------------------------|-------------------------|
| 1 | `users` | 1, 2, 3 | 4 | 5 | — |
| 2 | `projects` | 3, 6, 7, 11, 72, 74, 78, 94 | 8 | 9, 79, 95 | 10 |
| 3 | `budget_rows` | 12, 27, 36, 37, 74, 76, 88 | 8, 13, 16, 19, 37 | 14, 17, 54, 89, 98 | 15, 19 |
| 4 | `budget_cells` | 12, 20, 32, 36, 37, 74, 76, 88 | 8, 16, 18, 19 | 18, 20, 37, 54, 57, 89, 104 | 15, 19 |
| 5 | `phases` | 20, 23, 26, 76, 88 | 8, 24 | 24, 89 | 24 |
| 6 | `phase_assignments` | 20, 23, 25, 26, 76 | 8, 24 | 24, 25 | — |
| 7 | `ledger_entries` | 27, 35, 36, 47, 88 | 28, 31, 35, 62 | 29, 35, 89 | 30 |
| 8 | `ledger_allocations` | 27, 32, 36 | 31, 32, 37, 62 | 32, 37 | 30 |
| 9 | `ledger_attachments` | 27 | 33 | — | 30, 34 |
| 10 | `greenlit_history` | 21, 22, 51 | 20 | — | — |
| 11 | `change_history` | 87, 89 | 19, 20, 37, 38, 54, 88, 100 | — | — |
| 12 | `audit_log` | 90 | 8, 9, 10, 13, 14, 15, 16, 18, 19, 20, 24, 28, 29, 30, 31, 32, 35, 37, 38, 39, 52, 53, 54, 55, 56, 57, 58, 62, 63, 65, 67, 88, 89, 100, 101, 104, 105, 106, 107 | — | — |
| 13 | `cell_comments` | 12, 91 | 92 | — | 15, 93 |
| 14 | `spot_rates` | 94 | — | 95 | — |
| 15 | `fx_hedges` | 96 | 97 | 97 | — |
| 16 | `closed_months` | 36, 99 | 38, 100 | 38, 39, 100, 101 | — |
| 17 | `sales_data` | 40, 41, 47 | 42 | — | 43 |
| 18 | `wishlist_data` | 40, 41, 44, 47 | 45 | — | 46 |
| 19 | `promos` | 47 | 48 | 49 | 50 |
| 20 | `amendments` | 51 | 52 | 53, 54, 55 | 56 |
| 21 | `invoice_queue` | 59 | 60 | 61, 62, 63, 107 | — |
| 22 | `documents` | 64, 66 | 65 | — | 67 |
| 23 | `notes` | 68 | 69 | 70, 108 | 71 |
| 24 | `pnl_settings` | 72, 74 | 73 | 73 | — |
| 25 | `gl_one_sheet` | 74 | 75 | 75 | — |
| 26 | `milestone_data` | 76 | 77 | 77 | — |
| 27 | `marketing_budget_categories` | 80 | 81 | 82 | 83 |
| 28 | `marketing_budget_rows` | 80 | 84 | 85 | 83, 86 |
| 29 | `dept_colors` | 12 | — | (stored in projects JSONB) | — |
| 30 | `change_log` | 51, 102 | 37 | 54, 55, 57, 58, 104, 105 | — |
| 31 | `pending_changes` | 102 | 103 | — | 104, 105 |

---

### Endpoint Count per Table

| Table | Total Endpoints | R | C | U | D |
|-------|----------------|---|---|---|---|
| `audit_log` | 39 | 1 | 38 | 0 | 0 |
| `budget_cells` | 17 | 8 | 4 | 7 | 2 |
| `budget_rows` | 13 | 7 | 5 | 5 | 2 |
| `projects` | 10 | 8 | 1 | 3 | 1 |
| `ledger_entries` | 9 | 5 | 4 | 3 | 1 |
| `phases` | 7 | 5 | 2 | 2 | 1 |
| `change_history` | 9 | 2 | 7 | 0 | 0 |
| `phase_assignments` | 7 | 5 | 2 | 2 | 0 |
| `ledger_allocations` | 7 | 3 | 4 | 2 | 1 |
| `change_log` | 7 | 2 | 1 | 6 | 0 |
| `closed_months` | 5 | 2 | 2 | 4 | 0 |
| `amendments` | 5 | 1 | 1 | 3 | 1 |
| `invoice_queue` | 5 | 1 | 1 | 4 | 0 |
| `sales_data` | 4 | 3 | 1 | 0 | 1 |
| `wishlist_data` | 5 | 4 | 1 | 0 | 1 |
| `notes` | 4 | 1 | 1 | 2 | 1 |
| `documents` | 4 | 2 | 1 | 0 | 1 |
| `promos` | 4 | 1 | 1 | 1 | 1 |
| `marketing_budget_categories` | 4 | 1 | 1 | 1 | 1 |
| `marketing_budget_rows` | 4 | 1 | 1 | 1 | 2 |
| `cell_comments` | 3 | 2 | 1 | 0 | 2 |
| `greenlit_history` | 3 | 2 | 1 | 0 | 0 |
| `pnl_settings` | 3 | 2 | 1 | 1 | 0 |
| `users` | 4 | 3 | 1 | 1 | 0 |
| `gl_one_sheet` | 2 | 1 | 1 | 1 | 0 |
| `milestone_data` | 2 | 1 | 1 | 1 | 0 |
| `pending_changes` | 3 | 1 | 1 | 0 | 2 |
| `spot_rates` | 2 | 1 | 0 | 1 | 0 |
| `fx_hedges` | 2 | 1 | 1 | 1 | 0 |
| `ledger_attachments` | 3 | 1 | 1 | 0 | 2 |

---

**Total: 108 endpoints across 31 database tables**

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

