
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