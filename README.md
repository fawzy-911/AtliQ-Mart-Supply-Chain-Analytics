# AtliQ Mart — Supply Chain OTIF Diagnostic Platform

**A production-grade Power BI solution built to diagnose and root-cause contractual OTIF (On-Time-In-Full) failures across a multi-city FMCG distribution network — engineered to separate warehouse fulfillment failure from logistics transit failure, at SKU-level granularity, against individually contract-bound SLA targets per account.**

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black)

![Status](https://img.shields.io/badge/Status-Validated-success)

---

## Business Impact Statement

Order fulfillment failure in FMCG distribution is not a service-quality footnote — it is a direct P&L exposure. Retail trading agreements are typically **contract-bound on OTIF compliance**, and non-compliance below negotiated thresholds routinely triggers chargebacks, retroactive penalty clauses, reduced shelf allocation, and — for repeated breaches — delisting. Beyond direct penalties, each OTIF failure compounds working-capital cost: unsold inventory sitting against a missed delivery window, expedited-freight premiums to recover a botched shipment, and account-management hours spent firefighting instead of growing volume.

> **Note:** The dollar-impact figures below are placeholders. Populate with your organization's actual chargeback schedule, average order value, and affected-account revenue once available — a hiring manager will value a stated methodology for calculating exposure more than an unsupported number.

| Exposure Driver | Mechanism | Est. Impact (placeholder) |
|---|---|---|
| Contractual chargebacks | Penalty clauses tied to OTIF % below SLA | (OTIF Breaches × Avg. Order Value) × Penalty Rate |
| Lost shelf space / delisting risk | Retailer reallocates SKU facings to compliant suppliers | `$__ annualized, at-risk accounts` |
| Working capital drag | Expedited freight + re-pick cost on failed orders | `$__ / quarter` |

**This dashboard exists to convert that exposure from a lagging, quarterly-review conversation into a leading, account-level, root-cause-attributed one.**

---

## Project Workflow & Methodology

End-to-end process, raw ingestion → executive decision surface:

```
1. INGEST      → Raw CSV exports (fact_order_lines, dim_customers, dim_products,
                  dim_targets_orders, dim_date) loaded into Power Query
2. VALIDATE    → Schema audit: grain confirmation, date-format reconciliation,
                  duplicate-key detection (customer_name vs. customer_id)
3. TRANSFORM   → M-script date-type coercion, order-grain roll-up table construction
4. MODEL       → Star schema assembly; active/inactive relationship architecture
5. CALCULATE   → DAX measure layer: line-grain diagnostics + order-grain contractual KPIs
6. VALIDATE    → Cross-check every measure against independently recomputed source-data
                  truth before any visual is trusted (see Known Limitations)
7. VISUALIZE   → Single-page executive view: hero KPIs → trend → root-cause split →
                  account-level breach matrix → SKU-level shortfall detail
8. DOCUMENT    → DAX logic, data caveats, and modeling decisions recorded alongside
                  the deliverable, not left as tribal knowledge
```

This is a deliberately **validation-first** methodology: every KPI on the final canvas was independently recomputed from source data and reconciled against the DAX output before being considered production-ready. Discrepancies found during that process are documented in **Known Limitations**, not silently corrected and hidden.

---

## The "Why" Behind the Analysis

### Data Cleaning & Transformation

Source dates in `fact_order_lines` arrive as long-form text (`"Tuesday, March 1, 2022"`), while `dim_date` is stored as `dd-mmm-yy`. These are structurally incompatible for a relational join without explicit type coercion. Power Query M-scripts handle:

- Locale-aware date parsing and conversion to native `Date` type across all three date fields (`order_placement_date`, `agreed_delivery_date`, `actual_delivery_date`)
- Construction of a **derived order-grain table (`Orders`)** via `Table.Group` on `order_id`, applying `List.Min()` to the line-level `On Time` and `In Full` flags — this reconstructs the contractual AND-rule at order grain without relying on a pre-aggregated source table, making the roll-up auditable and independently reproducible from raw line data at any point
- Key-integrity validation: confirmed `customer_id` — not `customer_name` — as the only safe grouping key (15 recurring trade names map to 35 distinct customer accounts across three cities, each with independently negotiated SLA targets; grouping by name alone silently blends financially distinct accounts)

### Data Modeling: Star Schema & the Three-Date Relationship Problem

```
                    dim_date
                       │
        ┌──────────────┼──────────────┐
   (inactive)      (ACTIVE)       (inactive)
placement_date  actual_delivery  agreed_delivery
        │              │              │
        └──────────────┼──────────────┘
                        ▼
dim_customers ──▶ fact_order_lines ◀── dim_products
        │
dim_targets_orders
```

`fact_order_lines` contains three plausible date join-keys to a single `dim_date` dimension. Power BI enforces a single **active** relationship per fact-to-dimension pair when multiple candidate keys exist — a second or third simultaneous active path produces ambiguous filter propagation and is not supported without explicit control.

**Design decision — active relationship: `dim_date ↔ actual_delivery_date`.**
Rationale: SLA compliance is contractually assessed against the *realized* delivery event, not the order-intake date or the promised date — this is the axis on which retailer chargeback clauses are typically evaluated, and therefore the axis the default report canvas must filter against.

**Inactive relationships, activated contextually via `USERELATIONSHIP()`:**

```dax
Orders Placed (by Order Date) =
CALCULATE (
    DISTINCTCOUNT ( fact_order_lines[order_id] ),
    USERELATIONSHIP ( fact_order_lines[order_placement_date], dim_date[date] )
)

On Time % (by Promise Date) =
CALCULATE (
    [On Time %],
    USERELATIONSHIP ( fact_order_lines[agreed_delivery_date], dim_date[date] )
)
```

This architecture isolates three independent analytical lenses — demand intake trend, promise-date/lead-time integrity, and realized SLA performance — from a single fact table, without duplicating the fact table or the date dimension.

### Business Logic: The Strict AND-Rule as Source of Truth

OTIF is a **binary, all-or-nothing contractual pass/fail condition**, not a derived average. An order is compliant only if *every* line within it is simultaneously on-time and in-full — a single short-shipped SKU or a single late line fails the entire order, because a retailer cannot merchandise a partial case fill against a live planogram.

**Critical distinction — this is the single most consequential modeling decision in this build:**

```dax
-- ❌ NAIVE (statistically invalid — assumes independence between failure modes):
OTIF % (Wrong) = [On Time %] * [In Full %]

-- ✅ SOURCE OF TRUTH (direct joint-condition count at order grain):
OTIF % =
DIVIDE (
    CALCULATE (
        COUNTROWS ( Orders ),
        Orders[Order_On_Time] = 1,
        Orders[Order_In_Full] = 1
    ),
    COUNTROWS ( Orders ),
    0
)
```

Multiplying independent on-time and in-full rates assumes the two failure modes are statistically uncorrelated — an assumption that does not hold in operational reality and materially **overstates** true compliance. In this build, the naive multiplicative approach produced a headline figure roughly 10 percentage points more favorable than the validated joint-condition result — the difference between an accurate crisis signal and a materially misleading one at the executive level.

---

## Strategic Diagnostic Insights

This dashboard is architected as a **root-cause diagnostic instrument**, not a static scorecard. The core design thesis: OTIF failure has two operationally and organizationally distinct root causes, and conflating them produces the wrong remediation.

| Failure Mode | Operational Origin | Organizational Owner | Dashboard Signal |
|---|---|---|---|
| **In-Full failure** | Warehouse allocation precision, safety-stock buffer, pick/pack accuracy | Inventory & Warehouse Ops | `In-Full %` isolated per account, benchmarked against contracted `In-Full Target %` |
| **On-Time failure** | Dispatch scheduling, transit capacity, route/corridor congestion, or unrealistic promise-date entry | Logistics & Transport | `On-Time %` isolated per account, cross-referenced against `On-Time % (by Promise Date)` to separate "we promised badly" from "we missed what we promised" |

The **Root Cause Split** and **Customer Breach Matrix** visuals are built specifically to let a reviewer identify, per account and without a drill-through click, which failure mode is dominant — converting "service levels are bad" into "Account X needs a warehouse-allocation intervention; Account Y needs a transit-corridor review," in a single glance.

> Fill in your final validated figures below before publishing.

| KPI | Actual Result | Average Contracted Target | Gap vs. Target |
|---|---|---|---|
| **On-Time %** | **59.03%** | 86.09% | **-27.06%** |
| **In-Full %** | **52.78%** | 76.51% | **-23.73%** |
| **OTIF %** | **29.02%** | 65.91% | **-36.89%** |

**Headline finding:** The massive collapse in the combined OTIF metric (29% actual vs. 66% target) demonstrates the compounding failure effect. The supply chain is simultaneously suffering from severe inventory shortages (failed In-Full) and logistics bottlenecks (failed On-Time), requiring immediate intervention on both the warehouse picking line and the delivery dispatch network.

**Headline finding:** *[e.g., "X% of breaching accounts are In-Full-driven, Y% are On-Time-driven — evidence that a single company-wide 'improve service' initiative is the wrong remediation; this requires two parallel, differently-owned workstreams."]*

---

## Technical Appendix

### Code Logic — Order-Grain Roll-Up via `MIN()` Aggregation

Order-level compliance cannot be evaluated directly against `fact_order_lines`, because the fact table's native grain is one row per SKU per order — a filter context at that grain cannot enforce a joint condition across sibling lines. The `Orders` table resolves this by grouping `fact_order_lines` on `order_id` in Power Query and applying `List.Min()` to each line's binary `On Time` and `In Full` flags:

```dax
Order_On_Time = List.Min( [On Time] )   -- 0 if ANY line in the order was late
Order_In_Full = List.Min( [In Full] )   -- 0 if ANY line in the order was short-shipped
```

`MIN()` is the correct aggregator here — not `AVERAGE()`, not `MAX()` — because it is the only function that enforces "compliant only if zero lines failed," which is the literal contractual definition of OTIF. This produces a physical, single-pass-aggregatable table rather than a virtual `SUMMARIZE()` re-scan at every query, which materially improves report performance on a shared workbook under concurrent executive filtering.

### Known Limitations

Documented deliberately, not omitted — data integrity in a portfolio artifact means surfacing what was found, including what needed correcting:

- **Calendar coverage gap:** 441 line records (0.77% of the dataset) carry an `actual_delivery_date` beyond `dim_date`'s original calendar bound. Left unresolved, this silently blanks affected rows out of any `actual_delivery_date`-filtered visual. Remediated by extending `dim_date` to cover the full observed date range plus buffer.
- **Grouping-key integrity:** 35 distinct customer accounts share only 15 unique trade names across three cities. Any visual grouped by `customer_name` alone blends financially and contractually distinct accounts into a single misleading average. Remediated by enforcing `customer_id` as the sole grouping key, with `city` displayed alongside name for human-readable disambiguation.
- **Dimensional scope of the order-grain table:** the `Orders` roll-up table aggregates away `product_id` in the `Table.Group` step by design — meaning order-grain measures (`On Time %`, `In Full %`, `OTIF %`) **cannot** be sliced by Product or Category, since that relationship no longer exists post-aggregation. Category/SKU-level diagnostics are correctly served instead by the line-grain measure (`Line Fill Rate %`), which retains the `dim_products` relationship. This is a structural consequence of the modeling decision, not an oversight, and is documented here so it isn't rediscovered as a "bug" later.
- **Multiplicative OTIF anti-pattern:** an earlier build iteration computed `OTIF % = On Time % × In Full %`, which was caught during validation and corrected to the joint-condition `MIN()`-based calculation described above — see Business Logic.

---

## Technical Stack

- **Power BI Desktop** — semantic modeling, DAX authoring, report design
- **Power Query (M)** — ingestion, type coercion, order-grain roll-up construction
- **DAX** — explicit measure layer across line-grain and order-grain KPI sets, target-gap variance logic, `USERELATIONSHIP`-based contextual date switching
- **Star Schema Data Modeling** — single fact table, conformed dimensions, single-direction filter propagation, controlled active/inactive relationship management

---

## Dashboard Preview

*See `/assets` for full-resolution screenshots: dashboard overview, data model relationship view, and DAX measure library.*

---

## Repository Structure

See `/data`, `/assets`, and `/docs` for source data documentation, visual exports, and extended DAX/business-logic reference material respectively.

---

## Contact

*[Your name / LinkedIn / email here]*
