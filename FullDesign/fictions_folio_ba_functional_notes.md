# Fictions Folio — BA Functional Notes

**Scope:** This document consolidates the BA explanations requested for the following areas:

1. P&L Projection — Performance Targets
2. P&L Projection — Break-Even Units and Net Profit
3. P&L Projection — Scenario Analysis
4. Reconciliation Page
5. Variance Page
6. Milestones Page

---

# 1. P&L Projection — Performance Targets

## Purpose

The **Performance Targets** section in the P&L Projection page is used to determine how many units must be sold to achieve a target return level.

It answers:

> **“How many units do we need to sell to break even or achieve a specific ROI target?”**

Examples:

| Target | Meaning |
|---|---|
| Break-even | Profit reaches zero |
| 35% ROI | Profit reaches 35% of Fictions Recoupment |

---

## How Performance Targets are calculated

Performance Targets are not manually entered as final values. They are calculated from configured ROI targets.

Example:

```js
p.roiTargets = [0, 0.35]
```

Where:

| ROI Target | Meaning |
|---:|---|
| `0` | Break-even |
| `0.35` | 35% ROI |

---

## Step 1: Calculate Fictions Recoupment

First, the system calculates the amount Fictions must recover.

```text
Publishing Fee Base =
Development Budget
+ Porting Budget
+ External Budget
+ Reserve Budget
+ Contingency Budget
```

Then:

```text
Publishing Fee =
Publishing Fee Base × Publishing Fee Rate
```

Then:

```text
Total Costs / Recoupment =
Publishing Fee Base
+ Marketing Budget
+ Upfront Engine Fee
+ Publishing Fee
```

Then:

```text
Fictions Recoupment =
Total Costs - Partner Contribution
```

If there is no publishing partner contribution:

```text
Fictions Recoupment = Total Costs
```

---

## Step 2: Calculate Revenue Per Unit

```text
Average Price =
Retail Price × (1 - Discount Rate)
```

```text
After Platform Revenue Per Unit =
Average Price × (1 - Platform Fee - Engine Fee)
```

```text
Licensor Royalty Per Unit =
After Platform Revenue Per Unit × Licensor Royalty Rate
```

```text
Net Revenue Per Unit =
After Platform Revenue Per Unit - Licensor Royalty Per Unit
```

---

## Step 3: Run P&L Calculation for a Unit Count

The system uses:

```js
calcPnl(units)
```

For the given unit count, it calculates:

| Output | Formula / Logic |
|---|---|
| Gross Revenue | `units × averagePrice` |
| After Platform Revenue | `grossRevenue × (1 - platformFee - engineFee)` |
| Licensor Royalty | Calculated through licensor royalty waterfall |
| Net Revenue | `afterPlatformRevenue - licensorRoyalty` |
| Developer Royalties | Calculated through developer royalty waterfall |
| Publisher Royalties | Calculated through publisher royalty waterfall |
| Net Profit | `netRevenue - fictionsRecoupment - developerRoyalties - publisherRoyalties` |

---

## Step 4: Find Units Required for Target ROI

The system uses:

```js
findUnitsForROI(target)
```

This searches for the unit count where:

```text
Net Profit / Fictions Recoupment = ROI Target
```

Examples:

| Target | Formula Goal |
|---|---|
| Break-even | `Net Profit / Fictions Recoupment = 0` |
| 35% ROI | `Net Profit / Fictions Recoupment = 0.35` |

So for Break-even:

```text
Net Profit = 0
```

For 35% ROI:

```text
Net Profit = Fictions Recoupment × 35%
```

---

## Performance Targets Table

| Field | Source | Type |
|---|---|---|
| Target Name | Configured ROI target label | Static / Configured |
| ROI % | `p.roiTargets` | Static / Configured |
| Units Required | `findUnitsForROI(target)` | Calculated |
| Net Profit | `calcPnl(unitsRequired).profit` | Calculated |

Example:

| Target | Units Required | Net Profit |
|---|---:|---:|
| Break-even | Calculated | $0 |
| 35% ROI | Calculated | `Fictions Recoupment × 0.35` |

---

# 2. P&L Projection — Break-Even Units and Net Profit

## 2.1 Break-Even Units

### Purpose

Break-Even Units shows how many units must be sold for Fictions to recover its recoupment exposure.

It answers:

> **“How many copies must we sell before Fictions gets its money back?”**

---

## Formula

```text
Break-Even Units =
ceil(Fictions Recoupment / Net Revenue Per Unit)
```

Where:

| Field | Meaning |
|---|---|
| Fictions Recoupment | Total amount Fictions must recover |
| Net Revenue Per Unit | Revenue available per unit after platform, engine, and licensor deductions |
| `ceil` | Rounds up to the next whole unit |

---

## Example

Assume:

| Input | Value |
|---|---:|
| Fictions Recoupment | $9,750,000 |
| Retail Price | $29.99 |
| Discount Rate | 15% |
| Platform Fee | 30% |
| Engine Fee | 5% |
| Licensor Royalty | 10% |

### Step 1: Average Price

```text
Average Price =
29.99 × (1 - 0.15)
= 29.99 × 0.85
= 25.49
```

### Step 2: After Platform Revenue Per Unit

```text
After Platform Revenue Per Unit =
25.49 × (1 - 0.30 - 0.05)
= 25.49 × 0.65
= 16.57
```

### Step 3: Licensor Royalty Per Unit

```text
Licensor Royalty Per Unit =
16.57 × 0.10
= 1.66
```

### Step 4: Net Revenue Per Unit

```text
Net Revenue Per Unit =
16.57 - 1.66
= 14.91
```

### Step 5: Break-Even Units

```text
Break-Even Units =
ceil(9,750,000 / 14.91)
= 653,923 units
```

---

## 2.2 Net Profit

### Purpose

Net Profit shows how much profit remains after Fictions recoupment and royalty payouts.

It answers:

> **“After costs and royalties, how much profit is left?”**

---

## Formula

```text
Net Profit =
Net Revenue
- Fictions Recoupment
- Developer Royalties
- Publisher Royalties
```

---

## How Net Revenue is calculated

```text
Gross Revenue =
Units Sold × Average Price
```

```text
After Platform Revenue =
Gross Revenue × (1 - Platform Fee - Engine Fee)
```

```text
Net Revenue =
After Platform Revenue - Licensor Royalty
```

---

## Example

Assume:

| Input | Value |
|---|---:|
| Units Sold | 1,000,000 |
| Average Price | $25.49 |
| Platform + Engine deduction | 35% |
| Licensor Royalty | 10% |
| Fictions Recoupment | $9,750,000 |
| Developer Royalties | $2,000,000 |
| Publisher Royalties | $0 |

### Step 1: Gross Revenue

```text
Gross Revenue =
1,000,000 × 25.49
= 25,490,000
```

### Step 2: After Platform Revenue

```text
After Platform Revenue =
25,490,000 × 65%
= 16,568,500
```

### Step 3: Licensor Royalty

```text
Licensor Royalty =
16,568,500 × 10%
= 1,656,850
```

### Step 4: Net Revenue

```text
Net Revenue =
16,568,500 - 1,656,850
= 14,911,650
```

### Step 5: Net Profit

```text
Net Profit =
14,911,650
- 9,750,000
- 2,000,000
- 0
= 3,161,650
```

So:

```text
Net Profit = $3,161,650
```

---

## Important Difference

There are two related break-even calculations:

| Item | Logic |
|---|---|
| Break-Even Units Card | `ceil(Fictions Recoupment / Net Revenue Per Unit)` |
| Performance Target Break-even | Uses `findUnitsForROI(0)` to find where `Net Profit = 0` |

The Performance Target version is more complete because it accounts for the full royalty waterfall through `calcPnl(units)`.

---

# 3. P&L Projection — Scenario Analysis

## Purpose

Scenario Analysis is used to test financial outcomes at different unit sales levels.

It answers:

> **“If the game sells X units, what revenue, royalties, profit, and return do we get?”**

---

## Scenario Input

The input is a configured list of unit counts.

Example:

```js
p.scenarioUnits = [100000, 250000, 500000, 1000000, 2500000]
```

Each value becomes one scenario row.

| Scenario Units | Meaning |
|---:|---|
| 100,000 | What happens at 100k sales |
| 250,000 | What happens at 250k sales |
| 500,000 | What happens at 500k sales |
| 1,000,000 | What happens at 1M sales |
| 2,500,000 | What happens at 2.5M sales |

---

## Main Calculation

For each unit count, the system runs:

```js
const scenarios = p.scenarioUnits.map(units => calcPnl(units));
```

Example:

```js
calcPnl(500000)
```

The result becomes one row in the Scenario Analysis table.

---

## Scenario Calculation Steps

### Step 1: Average Price

```text
Average Price =
Retail Price × (1 - Discount Rate)
```

Example:

```text
29.99 × (1 - 15%)
= 25.49
```

### Step 2: Gross Revenue

```text
Gross Revenue =
Scenario Units × Average Price
```

Example for 500,000 units:

```text
500,000 × 25.49
= 12,745,000
```

### Step 3: After Platform Revenue

```text
After Platform Revenue =
Gross Revenue × (1 - Platform Fee - Engine Fee)
```

Example:

```text
12,745,000 × (1 - 30% - 5%)
= 12,745,000 × 65%
= 8,284,250
```

### Step 4: Licensor Royalty

If licensor royalty is enabled, the system calculates licensor royalties through the licensor royalty tiers.

Basic version:

```text
Licensor Royalty =
After Platform Revenue × Licensor Royalty Rate
```

If tiered:

```text
Licensor Royalty =
Sum of each licensor tier revenue slice × that tier's royalty %
```

### Step 5: Net Revenue

```text
Net Revenue =
After Platform Revenue - Licensor Royalty
```

Example, assuming 10% licensor royalty:

```text
8,284,250 - 828,425
= 7,455,825
```

### Step 6: Developer and Publisher Royalties

The system calculates developer and publisher royalties using the royalty waterfall.

```text
Developer Royalty =
Sum of each tier revenue slice × developer share %

Publisher Royalty =
Sum of each tier revenue slice × publisher share %
```

### Step 7: Net Profit

```text
Net Profit =
Net Revenue
- Fictions Recoupment
- Developer Royalties
- Publisher Royalties
```

### Step 8: Cash-on-Cash / Return

```text
Cash-on-Cash =
Net Revenue / Fictions Recoupment
```

ROI-style return may also be represented as:

```text
ROI =
Net Profit / Fictions Recoupment
```

---

## Scenario Analysis Table

| Units | Gross Revenue | Net Revenue | Dev Royalties | Pub Royalties | Net Profit | Cash-on-Cash / ROI |
|---:|---:|---:|---:|---:|---:|---:|
| 100,000 | Calculated | Calculated | Calculated | Calculated | Calculated | Calculated |
| 250,000 | Calculated | Calculated | Calculated | Calculated | Calculated | Calculated |
| 500,000 | Calculated | Calculated | Calculated | Calculated | Calculated | Calculated |
| 1,000,000 | Calculated | Calculated | Calculated | Calculated | Calculated | Calculated |

---

## Difference Between Scenario Analysis and Performance Targets

| Feature | How it works |
|---|---|
| Scenario Analysis | User/system gives units first, then system calculates financial outcome |
| Performance Targets | User/system gives ROI target first, then system calculates required units |

In simple words:

```text
Scenario Analysis =
“If we sell X units, what happens?”

Performance Targets =
“How many units do we need to reach Y return?”
```

---

# 4. Reconciliation Page

## Purpose

The **Reconciliation Page** is used to connect ledger or invoice entries to the correct budget rows and months.

It answers:

> **“This invoice was billed — which budget line and month should it be allocated against?”**

---

## Business Purpose

| Item | Explanation |
|---|---|
| Main purpose | Allocate ledger/invoice amounts to budget rows |
| Business user | Producer, Finance, Project Owner |
| Why it is needed | To confirm actual invoices are mapped against the planned budget |
| Final outcome | Ledger entries are fully allocated, budget cells are reconciled, and months can be closed |

---

## Example

| Invoice | Amount | Allocate To |
|---|---:|---|
| Vendor invoice for QA | $8,000 | QA budget row, April 2026 |
| Porting invoice | $12,000 | Console Porting row, May 2026 |

---

## Key Functionality

| Functionality | Description |
|---|---|
| Select ledger entry | User selects an invoice or ledger transaction |
| Allocate to budget row | User maps the amount to one or more budget rows |
| Allocate to month | User selects the applicable budget month |
| Partial allocation | One invoice can be split across multiple rows/months |
| Full allocation | Invoice is fully reconciled when total allocated equals total billed |
| Auto-match by amount | System can suggest matching based on amount |
| Close month | Once entries are reconciled, finance can close the month |

---

## What is Unallocated Amount?

**Unallocated Amount** means the part of a ledger/invoice amount that has not yet been assigned to any budget row or month.

Formula:

```text
Unallocated Amount =
Total Billed Amount - Allocated Amount
```

Where:

```text
Allocated Amount =
Sum of all allocation amounts for that ledger entry
```

---

## Example

Suppose an invoice is:

| Field | Value |
|---|---:|
| Invoice Total / Total Billed | $10,000 |
| Allocated to Budget Row A | $4,000 |
| Allocated to Budget Row B | $3,000 |

Then:

```text
Allocated Amount =
4,000 + 3,000
= 7,000
```

```text
Unallocated Amount =
10,000 - 7,000
= 3,000
```

So **$3,000 is still unallocated**.

---

## Reconciliation Status Logic

| Condition | Meaning | Status |
|---|---|---|
| `allocated == 0` | Nothing has been mapped yet | Unallocated |
| `0 < allocated < total` | Some amount mapped, some still pending | Partially Allocated |
| `allocated >= total - 0.01` | Full invoice mapped | Fully Allocated |

---

## Example Allocation Object

```json
{
  "ledgerId": "led_c3d4",
  "totalBilled": 8000,
  "allocations": [
    {
      "rowId": "row_q4mr",
      "monthKey": "2026-04",
      "amount": 5000
    },
    {
      "rowId": "row_n3kp",
      "monthKey": "2026-04",
      "amount": 3000
    }
  ]
}
```

In this example:

```text
Allocated =
5,000 + 3,000
= 8,000
```

```text
Unallocated =
8,000 - 8,000
= 0
```

So this invoice is fully reconciled.

---

## Editable Fields

| Field | Editable? |
|---|---|
| Selected ledger entry | Yes |
| Target budget row | Yes |
| Target month | Yes |
| Allocation amount | Yes |
| Adjustment option | Yes, if user chooses to adjust live budget |
| Month close action | Yes, if user has permission |

---

## Calculated Fields

| Field | Calculation |
|---|---|
| Allocated Amount | Sum of `allocations[].amount` |
| Unallocated Amount | `totalBilled - allocatedAmount` |
| Allocation Status | Based on allocated vs total billed |
| Remaining Unallocated | Amount still pending allocation |
| Month Reconciliation Status | Based on whether relevant entries are allocated |

---

# 5. Variance Page

## Purpose

The **Variance Page** is a read-only reporting page used to compare the project’s latest/live budget forecast against the locked Greenlit Budget.

It answers:

> **“Are we currently projected to finish over or under the approved greenlit budget?”**

---

## Business Purpose

| Purpose | Explanation |
|---|---|
| Track budget variance | Shows whether the project is over or under the greenlit baseline |
| Separate past vs future spend | Splits live budget into elapsed months and future months |
| Support finance review | Helps Finance and Producers identify cost sections that are drifting |
| Highlight risk areas | Shows which groups/sections are exceeding greenlit budget |
| Support amendment decisions | If variance is high, the team may need an amendment or budget review |

---

## What the Page Shows

The page shows KPI cards and a Variance by Cost Section table.

| Column | Meaning |
|---|---|
| Actuals to Date | Live budget values for months that have already elapsed |
| Remaining Projected | Live budget values for future months |
| Projected Total | Actuals to Date + Remaining Projected |
| Greenlit Total | Approved locked greenlit budget across all months |
| Variance | Projected Total - Greenlit Total |

---

## Important BA Note

On this page, **Actuals to Date does not mean ledger-paid actuals**.

It means:

> Past-month values from the Live Budget Grid.

Ledger actuals may be shown as a reference, but they are not used in the variance calculation.

---

## Formula

```text
Projected Total =
Actuals to Date + Remaining Projected
```

```text
Variance =
Projected Total - Greenlit Total
```

```text
Variance % =
Variance / Greenlit Total × 100
```

---

## Example

| Field | Value |
|---|---:|
| Actuals to Date | $1,500,000 |
| Remaining Projected | $4,000,000 |
| Projected Total | $5,500,000 |
| Greenlit Total | $5,000,000 |
| Variance | $500,000 over budget |

So the page tells the user:

> The project is currently projected to finish **$500,000 over** the approved greenlit budget.

---

## Editable Fields

None.

The Variance Page is read-only.

---

## Calculated Fields

| Field | Calculation |
|---|---|
| Actuals to Date | Sum of live budget values for elapsed months |
| Remaining Projected | Sum of live budget values for future months |
| Projected Total | `Actuals to Date + Remaining Projected` |
| Greenlit Total | Sum of greenlit values for all relevant months |
| Variance | `Projected Total - Greenlit Total` |
| Variance % | `Variance / Greenlit Total × 100` |

---

# 6. Milestones Page

## Purpose

The **Milestones Page** is used to track the project month by month, where each month acts like a milestone.

It answers:

> **“For this milestone or month, what build status do we have, what notes or blockers exist, where are the files, and how much budget has been consumed so far?”**

---

## Business Purpose

| Purpose | Explanation |
|---|---|
| Track milestone status | Keeps monthly build/milestone progress in one place |
| Capture project notes | Stores milestone notes, blockers, risks, and delivery information |
| Link delivery files | Stores Dropbox folder URL or related delivery location |
| Show budget progress | Displays burn, life-to-date spend, and remaining budget |
| Support project review | Helps Producer and Finance review milestone progress with cost context |

---

## Editable / Static Fields

These values are manually entered by the user.

| Field | Purpose |
|---|---|
| Build Information | Build version, platform, branch, release candidate info, etc. |
| Milestone Notes | Deliverables, goals, blockers, status updates |
| Dropbox Folder URL | Link to milestone files/assets/build drop location |

These are stored conceptually as:

```text
milestoneData[monthKey].buildInfo
milestoneData[monthKey].notes
milestoneData[monthKey].dropboxFolder
```

---

## Is the data calculated?

Partly yes.

The Milestones Page contains both:

| Data Type | Fields |
|---|---|
| Editable / Static | Build Information, Milestone Notes, Dropbox Folder URL |
| Calculated / Derived | Milestone number, phase, monthly burn, LTD spend, remaining budget, progress percentage |

---

## Calculated Fields

| Field | How it is calculated |
|---|---|
| Milestone Number | Based on month index; Month 1 = Milestone 1 |
| Month Label | Derived from project timeline month |
| Phase Badge | Pulled from project phase assignment for that month |
| Monthly Burn | Total live budget value for that month |
| LTD Spend | Sum of monthly burn from first month through current milestone |
| Remaining Budget | Total Budget - LTD Spend |
| Progress % | `LTD Spend / Total Budget × 100` |

---

## Example

Assume the live budget has:

| Month / Milestone | Monthly Burn |
|---|---:|
| Milestone 1 | $100,000 |
| Milestone 2 | $150,000 |
| Milestone 3 | $200,000 |

For Milestone 3:

```text
Monthly Burn =
$200,000
```

```text
LTD Spend =
100,000 + 150,000 + 200,000
= 450,000
```

```text
Remaining Budget =
Total Budget - 450,000
```

So the Milestones Page is not only a notes page. It also gives budget progress for each milestone/month.

---

## BA Summary

| Question | Answer |
|---|---|
| Purpose | Track monthly milestone status, build information, blockers, links, and budget progress |
| Is it editable? | Yes, for build info, notes, and Dropbox URL |
| Is it calculated? | Yes, for burn, LTD spend, remaining budget, progress %, phase, and milestone number |
| Main data source | Budget Grid live values and milestone data |
| Complexity | Medium |

---

# 7. Combined BA Summary

| Page / Section | Purpose | Editable Data | Calculated Data |
|---|---|---|---|
| P&L Performance Targets | Find units needed to hit ROI targets | ROI target config | Units required, profit at target |
| Break-Even Units | Find units required to recover Fictions exposure | P&L input assumptions | Break-even units |
| Net Profit | Show profit after recoupment and royalties | P&L input assumptions | Net profit |
| Scenario Analysis | Show financial outcome at selected unit counts | Scenario unit counts | Revenue, royalties, profit, return |
| Reconciliation | Allocate ledger/invoice amounts to budget rows/months | Allocation rows, amounts, month, target row | Allocated amount, unallocated amount, status |
| Variance | Compare live projection against greenlit baseline | None | Actuals to date, remaining projected, variance |
| Milestones | Track milestone notes and monthly budget progress | Build info, notes, Dropbox URL | Monthly burn, LTD spend, remaining budget, progress % |

---

# 8. Recommended Backend Notes

## P&L Calculation Service

The backend should centralize:

- Fictions Recoupment
- Average Price
- Net Revenue Per Unit
- Licensor Royalty waterfall
- Developer Royalty waterfall
- Publisher Royalty waterfall
- Net Profit
- Break-Even Units
- Scenario Analysis
- Performance Targets

Recommended endpoint:

```http
POST /projects/:projectId/pnl/calculate
```

---

## Reconciliation Service

The backend should protect reconciliation with optimistic locking.

Recommended endpoint:

```http
POST /projects/:projectId/reconciliation/allocate
```

Important rules:

- One ledger entry can be allocated to multiple rows/months.
- Allocation total should not exceed total billed amount.
- Fully allocated entries can be marked reconciled.
- Month close should only happen after all required entries are reconciled.
- Conflict-sensitive actions should use `If-Match` / ETag validation.

---

## Variance Reporting Service

Recommended endpoint:

```http
GET /projects/:projectId/variance
```

Important rules:

- Variance uses Live Budget Grid values, not ledger-paid actuals.
- Past months = Actuals to Date.
- Future months = Remaining Projected.
- Greenlit values are the locked baseline.

---

## Milestones Service

Recommended endpoint:

```http
GET /projects/:projectId/milestones
PATCH /projects/:projectId/milestones/:monthKey
```

Important rules:

- Build info, notes, and Dropbox URL are user-maintained.
- Budget progress values are calculated from the live budget grid.
