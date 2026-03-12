# Analytical Warehouse Architecture

**Version:** 1.0 | **Classification:** Internal Architecture Reference
**Cross-reference:** api_mapping_document.md | crm_implementation_guide.md

---

## Table of Contents

1. Architecture Overview
2. Star Schema Design
3. Fact Tables — Grain Declarations and Schema
4. Dimension Tables — Schema and SCD Types
5. Date Dimension
6. Revenue Lineage Model
7. Margin Calculation Model
8. KPI Catalog
9. Common Analytical Query Patterns
10. Warehouse Governance

---

## 1. Architecture Overview

The analytical warehouse is a star schema optimized for revenue lifecycle, service operations, and profitability analysis. It is populated from the CRM integration pipeline and serves as the single source of truth for all reporting and dashboarding.

### Layer Responsibilities

| Layer | Tables | Responsibility |
|---|---|---|
| Staging | stg_* | Raw and typed CRM data |
| Dimension | dim_* | Descriptive context for analysis |
| Fact | fact_* | Measurable events and transactions |
| Date | dim_date | Unified time intelligence |

### Design Principles

- Every fact table has a declared grain — one row = one specific event
- All fact tables join to dimensions via surrogate keys only
- Natural keys (CRM UUIDs) preserved in fact tables for lineage tracing
- deal_id propagated through all revenue-bearing fact tables
- SCD Type 2 tracks historical changes for analytical accuracy
- No hard deletes — soft delete flags used

---

## 2. Star Schema Design

### Schema Overview

```
                    dim_date
                       |
        dim_company     |     dim_vendor
             \          |         /
              \         |        /
   dim_contact--[fact_deal]------
              /         |        \
             /          |         \
       dim_asset    dim_facility   dim_contract
             \          |         /
              \         |        /
         [fact_maintenance]------
                        |
                   [fact_invoice]
                        |
                   [fact_cost]
```

### Fact Table Summary

| Fact Table | Grain | Primary Dimensions |
|---|---|---|
| fact_deal | One row per deal stage transition | dim_company, dim_date, dim_contact |
| fact_invoice | One row per invoice line | dim_company, dim_contract, dim_date |
| fact_cost | One row per cost record per contract | dim_contract, dim_date |
| fact_maintenance | One row per maintenance event | dim_asset, dim_vendor, dim_facility, dim_contract, dim_date |

### Dimension Table Summary

| Dimension | SCD Type | Key Field |
|---|---|---|
| dim_company | Type 2 on client_status | sk_company |
| dim_contact | Type 1 | sk_contact |
| dim_contract | Type 2 on contract_status | sk_contract |
| dim_asset | Type 2 on lifecycle_stage | sk_asset |
| dim_facility | Type 1 | sk_facility |
| dim_vendor | Type 1 | sk_vendor |
| dim_date | Type 0 — fixed | date_key |

---

## 3. Fact Tables

### 3.1 fact_deal

**Grain:** One row per deal stage transition event.
Each time a deal changes stage, a new row is inserted.
This enables pipeline velocity, stage duration, and conversion analysis.

| Column | Data Type | Description |
|---|---|---|
| sk_deal_event | INT | Surrogate primary key for this event row |
| deal_id | VARCHAR(36) | Natural CRM deal ID — preserved for lineage |
| sk_company | INT | FK to dim_company |
| sk_contact | INT | FK to dim_contact — primary contact |
| deal_name | VARCHAR(255) | Deal name at time of event |
| pipeline | VARCHAR(100) | Pipeline name |
| deal_stage | VARCHAR(100) | Stage at this event |
| previous_stage | VARCHAR(100) | Prior stage — NULL for first event |
| estimated_contract_value | DECIMAL(18,2) | Contract value at time of event |
| final_contract_value | DECIMAL(18,2) | Negotiated value — populated at close |
| expected_margin | DECIMAL(5,4) | Margin estimate at event time |
| stage_probability | DECIMAL(5,4) | Stage probability at event time |
| weighted_value | DECIMAL(18,2) | estimated_contract_value x stage_probability |
| lead_source | VARCHAR(100) | How opportunity originated |
| service_type | VARCHAR(100) | Type of engagement |
| territory | VARCHAR(100) | Geographic assignment |
| is_closed_won | BOOLEAN | TRUE if stage = Closed Won |
| is_closed_lost | BOOLEAN | TRUE if stage = Closed Lost |
| loss_reason | VARCHAR(100) | NULL unless Closed Lost |
| created_date_key | INT | Date deal was created (YYYYMMDD) |
| stage_entry_date_key | INT | Date this stage was entered (YYYYMMDD) |
| close_date_key | INT | Expected close date (YYYYMMDD) |
| closed_won_date_key | INT | Actual won date (YYYYMMDD) |
| closed_lost_date_key | INT | Actual lost date (YYYYMMDD) |
| days_in_stage | INT | Days spent in this stage |
| total_sales_cycle_days | INT | Days from created to closed — populated at close |

---

### 3.2 fact_invoice

**Grain:** One row per invoice line item.
Each invoice record represents a discrete billing event against a contract.

| Column | Data Type | Description |
|---|---|---|
| sk_invoice | INT | Surrogate primary key |
| invoice_id | VARCHAR(36) | Natural CRM invoice ID |
| sk_company | INT | FK to dim_company |
| sk_contract | INT | FK to dim_contract |
| deal_id | VARCHAR(36) | Preserved deal lineage FK |
| invoice_number | VARCHAR(100) | Human-readable invoice reference |
| payment_status | VARCHAR(100) | Current payment status |
| billing_type | VARCHAR(100) | Invoice classification |
| invoice_amount | DECIMAL(18,2) | Billed amount before tax |
| tax_amount | DECIMAL(18,2) | Tax component |
| total_amount | DECIMAL(18,2) | Total including tax |
| invoice_date_key | INT | Invoice generation date (YYYYMMDD) |
| payment_due_date_key | INT | Payment due date (YYYYMMDD) |
| payment_received_date_key | INT | Payment receipt date (YYYYMMDD) — NULL if unpaid |
| days_to_payment | INT | payment_received_date - invoice_date — NULL if unpaid |
| is_paid | BOOLEAN | TRUE if payment_status = Payment Received |
| is_overdue | BOOLEAN | TRUE if past due and not paid |
| days_overdue | INT | Days past due — 0 if paid or not yet due |

---

### 3.3 fact_cost

**Grain:** One row per cost record submitted against a contract.

| Column | Data Type | Description |
|---|---|---|
| sk_cost | INT | Surrogate primary key |
| cost_id | VARCHAR(36) | Natural CRM cost ID |
| sk_contract | INT | FK to dim_contract |
| deal_id | VARCHAR(36) | Preserved deal lineage FK |
| cost_category | VARCHAR(100) | Labor, Vendor, Equipment, Overhead |
| cost_period | VARCHAR(50) | Period this cost covers |
| labor_cost | DECIMAL(18,2) | Personnel cost component |
| vendor_cost | DECIMAL(18,2) | Third-party service cost |
| equipment_cost | DECIMAL(18,2) | Parts and materials cost |
| allocated_overhead | DECIMAL(18,2) | Overhead allocation |
| total_cost | DECIMAL(18,2) | SUM of all components |
| cost_recorded_date_key | INT | Date cost was recorded (YYYYMMDD) |
| cost_period_start_key | INT | Period start date (YYYYMMDD) |
| cost_period_end_key | INT | Period end date (YYYYMMDD) |

---

### 3.4 fact_maintenance

**Grain:** One row per maintenance event.
Each event represents a single service or repair action on a specific asset.

| Column | Data Type | Description |
|---|---|---|
| sk_maintenance | INT | Surrogate primary key |
| maintenance_event_id | VARCHAR(36) | Natural CRM event ID |
| sk_asset | INT | FK to dim_asset |
| sk_vendor | INT | FK to dim_vendor — NULL if internal tech |
| sk_facility | INT | FK to dim_facility |
| sk_contract | INT | FK to dim_contract |
| maintenance_type | VARCHAR(100) | Preventive, Corrective, Emergency, etc. |
| work_order_number | VARCHAR(100) | Work order reference |
| event_status | VARCHAR(100) | Current status |
| downtime_hours | DECIMAL(10,2) | Equipment downtime in hours |
| repair_cost | DECIMAL(18,2) | Cost of this service event |
| billable_flag | BOOLEAN | Whether event generates invoice |
| resolution_code | VARCHAR(100) | Outcome classification |
| maintenance_date_key | INT | Date service performed (YYYYMMDD) |
| next_service_due_date_key | INT | Next scheduled service (YYYYMMDD) |
| days_since_last_service | INT | Calculated from asset last_service_date |

---

## 4. Dimension Tables

### 4.1 dim_company

**SCD Type:** Type 2 on client_status

| Column | Data Type | Description |
|---|---|---|
| sk_company | INT | Surrogate primary key |
| company_id | VARCHAR(36) | Natural CRM ID — alternate key |
| company_name | VARCHAR(255) | Organization name |
| parent_health_system | VARCHAR(255) | Parent organization |
| industry | VARCHAR(100) | Industry classification |
| region | VARCHAR(100) | Sales region |
| bed_count | INT | Hospital bed count |
| facility_count | INT | Number of locations |
| client_status | VARCHAR(50) | SCD2 tracked — Prospect, Active, Former |
| annual_equipment_budget | DECIMAL(18,2) | Estimated annual spend |
| created_date_key | INT | Record creation date key |
| first_contract_date_key | INT | First contract date key |
| effective_date | DATE | SCD2 effective from |
| expiry_date | DATE | SCD2 effective to — 9999-12-31 if current |
| is_current | BOOLEAN | TRUE for active version |

---

### 4.2 dim_contact

**SCD Type:** Type 1 — overwrite

| Column | Data Type | Description |
|---|---|---|
| sk_contact | INT | Surrogate primary key |
| contact_id | VARCHAR(36) | Natural CRM ID |
| sk_company | INT | FK to dim_company |
| full_name | VARCHAR(255) | Concatenated first and last name |
| email | VARCHAR(255) | Dedup key |
| job_title | VARCHAR(150) | Role title |
| department | VARCHAR(150) | Department |
| role_type | VARCHAR(100) | Buyer role classification |
| decision_authority | VARCHAR(100) | Purchasing authority level |
| created_date_key | INT | Record creation date key |

---

### 4.3 dim_contract

**SCD Type:** Type 2 on contract_status

| Column | Data Type | Description |
|---|---|---|
| sk_contract | INT | Surrogate primary key |
| contract_id | VARCHAR(36) | Natural CRM ID |
| fk_deal_id | VARCHAR(36) | Originating deal — natural key for lineage |
| sk_company | INT | FK to dim_company |
| contract_name | VARCHAR(255) | Contract title |
| contract_status | VARCHAR(50) | SCD2 tracked |
| service_model | VARCHAR(100) | Delivery model |
| sla_type | VARCHAR(100) | Service level |
| billing_model | VARCHAR(100) | Billing structure |
| contract_value | DECIMAL(18,2) | Total contracted revenue |
| contracted_margin | DECIMAL(5,4) | Target margin |
| contract_term_years | INT | Duration in years |
| contract_signed_date_key | INT | Signature date key |
| contract_start_date_key | INT | Start date key |
| contract_end_date_key | INT | End date key |
| renewal_date_key | INT | Renewal target date key |
| effective_date | DATE | SCD2 effective from |
| expiry_date | DATE | SCD2 effective to |
| is_current | BOOLEAN | TRUE for active version |

---

### 4.4 dim_asset

**SCD Type:** Type 2 on lifecycle_stage

| Column | Data Type | Description |
|---|---|---|
| sk_asset | INT | Surrogate primary key |
| asset_id | VARCHAR(36) | Natural CRM ID |
| sk_facility | INT | FK to dim_facility |
| sk_contract | INT | FK to dim_contract |
| sk_company | INT | FK to dim_company |
| serial_number | VARCHAR(150) | Globally unique dedup key |
| equipment_type | VARCHAR(150) | Device classification |
| manufacturer | VARCHAR(150) | Equipment manufacturer |
| model_number | VARCHAR(100) | Manufacturer model |
| purchase_price | DECIMAL(18,2) | Acquisition cost |
| lifecycle_stage | VARCHAR(100) | SCD2 tracked |
| utilization_rate | DECIMAL(5,4) | Utilization as decimal |
| service_risk_score | DECIMAL(5,2) | Predictive risk score |
| condition_rating | VARCHAR(50) | Physical condition |
| installation_date_key | INT | Install date key |
| warranty_expiration_date_key | INT | Warranty expiry date key |
| replacement_due_date_key | INT | Planned replacement date key |
| effective_date | DATE | SCD2 effective from |
| expiry_date | DATE | SCD2 effective to |
| is_current | BOOLEAN | TRUE for active version |

---

### 4.5 dim_facility

**SCD Type:** Type 1

| Column | Data Type | Description |
|---|---|---|
| sk_facility | INT | Surrogate primary key |
| facility_id | VARCHAR(36) | Natural CRM ID |
| sk_company | INT | FK to dim_company |
| facility_name | VARCHAR(255) | Location name |
| facility_type | VARCHAR(100) | Hospital, Clinic, Lab, etc. |
| city | VARCHAR(100) | City |
| state | VARCHAR(50) | State |
| zip_code | VARCHAR(20) | Postal code |

---

### 4.6 dim_vendor

**SCD Type:** Type 1

| Column | Data Type | Description |
|---|---|---|
| sk_vendor | INT | Surrogate primary key |
| vendor_id | VARCHAR(36) | Natural CRM ID |
| vendor_name | VARCHAR(255) | Vendor name |
| vendor_type | VARCHAR(100) | OEM, ISP, Parts Supplier |
| region | VARCHAR(100) | Coverage region |
| preferred_vendor | BOOLEAN | Preferred status flag |

---

## 5. Date Dimension

**SCD Type:** Type 0 — fixed. Never changes once populated.

| Column | Data Type | Description |
|---|---|---|
| date_key | INT | YYYYMMDD format — primary key |
| full_date | DATE | Calendar date |
| day_of_week | INT | 1 Sunday through 7 Saturday |
| day_name | VARCHAR(20) | Full day name |
| week_of_year | INT | ISO week number |
| month_number | INT | 1 through 12 |
| month_name | VARCHAR(20) | Full month name |
| quarter | INT | 1 through 4 |
| calendar_year | INT | Four-digit year |
| fiscal_period | INT | Fiscal month if offset from calendar |
| fiscal_quarter | INT | Fiscal quarter |
| fiscal_year | INT | Fiscal year |
| is_weekday | BOOLEAN | True if Monday through Friday |
| is_month_end | BOOLEAN | True if last day of calendar month |
| is_quarter_end | BOOLEAN | True if last day of calendar quarter |
| is_fiscal_year_end | BOOLEAN | True if last day of fiscal year |

**Populate:** Minimum 10 years — 5 years historical and 5 years forward from deployment date.

---

## 6. Revenue Lineage Model

### Identifier Propagation Through Warehouse

```
fact_deal.deal_id
    |
    | (1:1)
    v
dim_contract.fk_deal_id
    |
    |-- (1:N) --> fact_invoice.deal_id
    |                  |
    |                  +--> Revenue = SUM(invoice_amount)
    |
    |-- (1:N) --> fact_cost.deal_id
                       |
                       +--> Cost = SUM(total_cost)
```

### Revenue Lineage SQL

```sql
WITH revenue AS (
    SELECT
        i.deal_id,
        SUM(i.invoice_amount)     AS total_revenue,
        SUM(i.total_amount)       AS total_billed
    FROM fact_invoice i
    WHERE i.is_paid = TRUE
    GROUP BY i.deal_id
),
costs AS (
    SELECT
        c.deal_id,
        SUM(c.total_cost)         AS total_cost
    FROM fact_cost c
    GROUP BY c.deal_id
),
deals AS (
    SELECT DISTINCT
        d.deal_id,
        d.deal_name,
        d.closed_won_date_key,
        d.final_contract_value,
        d.territory,
        comp.company_name
    FROM fact_deal d
    JOIN dim_company comp ON comp.sk_company = d.sk_company
    WHERE d.is_closed_won = TRUE
)
SELECT
    dl.deal_id,
    dl.deal_name,
    dl.company_name,
    dl.territory,
    dl.final_contract_value,
    r.total_revenue,
    c.total_cost,
    r.total_revenue - c.total_cost                        AS gross_margin,
    ROUND((r.total_revenue - c.total_cost)
          / NULLIF(r.total_revenue, 0) * 100, 2)          AS margin_pct
FROM deals dl
LEFT JOIN revenue r ON r.deal_id = dl.deal_id
LEFT JOIN costs c   ON c.deal_id = dl.deal_id
ORDER BY gross_margin DESC;
```

---

## 7. Margin Calculation Model

### Contract-Level Margin

```
Contract Revenue  = SUM(fact_invoice.invoice_amount)
                    WHERE sk_contract = X AND is_paid = TRUE

Contract Cost     = SUM(fact_cost.total_cost)
                    WHERE sk_contract = X

Gross Margin      = Contract Revenue - Contract Cost

Margin %          = Gross Margin / Contract Revenue
```

### Deal-Level Margin

```
Deal Revenue      = SUM(fact_invoice.invoice_amount)
                    WHERE deal_id = Y AND is_paid = TRUE

Deal Cost         = SUM(fact_cost.total_cost)
                    WHERE deal_id = Y

Deal Margin       = Deal Revenue - Deal Cost

Deal Margin %     = Deal Margin / Deal Revenue
```

### Asset-Level Cost Analysis

```
Asset Cost        = SUM(fact_maintenance.repair_cost)
                    WHERE sk_asset = Z

Asset Downtime    = SUM(fact_maintenance.downtime_hours)
                    WHERE sk_asset = Z

Cost Per Hour     = Asset Cost / NULLIF(Asset Downtime, 0)
```

---

## 8. KPI Catalog

Each KPI is defined with its business meaning, calculation, source tables, grain, and recommended refresh frequency.

### 8.1 Sales Pipeline KPIs

| KPI Name | Definition | Formula | Source | Grain | Refresh |
|---|---|---|---|---|---|
| Total Pipeline Value | Sum of open deal values | SUM(estimated_contract_value) WHERE NOT closed | fact_deal | All open deals | 4 hours |
| Weighted Pipeline Value | Probability-adjusted value | SUM(estimated_contract_value x stage_probability) | fact_deal | All open deals | 4 hours |
| Win Rate | Percentage of deals won | COUNT(is_closed_won) / COUNT(deal_id) | fact_deal | Period | Daily |
| Average Deal Size | Mean contract value of won deals | AVG(final_contract_value) WHERE is_closed_won | fact_deal | Period | Daily |
| Sales Cycle Length | Days from created to closed won | AVG(total_sales_cycle_days) WHERE is_closed_won | fact_deal | Period | Daily |
| Stage Conversion Rate | Deals advancing from each stage | COUNT(stage_n+1) / COUNT(stage_n) | fact_deal | By stage | Daily |
| Pipeline Velocity | Revenue generated per day | (Deals x Win Rate x Deal Size) / Sales Cycle Days | fact_deal | Period | Weekly |
| Committed Revenue | High-confidence pipeline | SUM(amount) WHERE stage IN Negotiation, Pending | fact_deal | Current period | 4 hours |

---

### 8.2 Contract Performance KPIs

| KPI Name | Definition | Formula | Source | Grain | Refresh |
|---|---|---|---|---|---|
| Active Contracts | Count of contracts in Active Operations | COUNT(contract_id) WHERE status = Active | dim_contract | Current | Daily |
| Contract Backlog | Remaining contracted revenue | SUM(contract_value - invoiced_to_date) | dim_contract, fact_invoice | Per contract | Daily |
| Renewal Rate | Contracts renewed vs expired | COUNT(Renewed) / COUNT(Expired + Renewed) | dim_contract | Period | Weekly |
| Contract Value at Risk | Expiring contracts in 90 days | SUM(contract_value) WHERE end_date within 90 days | dim_contract | Current | Daily |
| Average Contract Value | Mean contract size | AVG(contract_value) WHERE status = Active | dim_contract | Current | Weekly |
| Contract Activation Time | Days from signed to Active Operations | AVG(active_date - contract_signed_date) | dim_contract, dim_date | Per contract | Weekly |

---

### 8.3 Asset Lifecycle KPIs

| KPI Name | Definition | Formula | Source | Grain | Refresh |
|---|---|---|---|---|---|
| Assets Under Management | Total active assets | COUNT(asset_id) WHERE lifecycle_stage = Active | dim_asset | Current | Daily |
| Average Utilization Rate | Mean utilization across fleet | AVG(utilization_rate) | dim_asset | Current | Daily |
| Assets Nearing Replacement | Count approaching replacement date | COUNT(asset_id) WHERE replacement_due within 90 days | dim_asset | Current | Daily |
| High Risk Assets | Assets with elevated risk score | COUNT(asset_id) WHERE service_risk_score >= 7 | dim_asset | Current | Daily |
| Asset Age Distribution | Fleet age breakdown | DATEDIFF(today, installation_date) grouped | dim_asset, dim_date | Per asset | Weekly |
| Warranty Coverage Rate | Assets within warranty | COUNT(in_warranty) / COUNT(total) | dim_asset | Current | Daily |

---

### 8.4 Service Operations KPIs

| KPI Name | Definition | Formula | Source | Grain | Refresh |
|---|---|---|---|---|---|
| Total Service Events | Count of maintenance events in period | COUNT(maintenance_event_id) | fact_maintenance | Period | 4 hours |
| Mean Time Between Failures | Average days between corrective events | AVG(days_since_last_service) WHERE maintenance_type = Corrective | fact_maintenance | Per asset | Weekly |
| Corrective vs Preventive Ratio | Balance of reactive to planned work | COUNT(Corrective) / COUNT(Preventive) | fact_maintenance | Period | Weekly |
| Total Downtime Hours | Cumulative equipment downtime | SUM(downtime_hours) | fact_maintenance | Period | Daily |
| Average Repair Cost | Mean cost per service event | AVG(repair_cost) | fact_maintenance | Period | Weekly |
| First-Time Fix Rate | Events resolved on first visit | COUNT(resolved_first) / COUNT(total_events) | fact_maintenance | Period | Weekly |
| Vendor Response Compliance | Events met within SLA | COUNT(within_sla) / COUNT(total_vendor_events) | fact_maintenance | Period | Weekly |
| Billable Event Rate | Events that generate invoices | COUNT(billable_flag = TRUE) / COUNT(total) | fact_maintenance | Period | Weekly |

---

### 8.5 Revenue KPIs

| KPI Name | Definition | Formula | Source | Grain | Refresh |
|---|---|---|---|---|---|
| Total Revenue Recognized | Paid invoices in period | SUM(invoice_amount) WHERE is_paid AND invoice_date IN period | fact_invoice | Period | Daily |
| Revenue by Contract | Revenue per active contract | SUM(invoice_amount) GROUP BY sk_contract | fact_invoice | Per contract | Daily |
| Revenue by Region | Revenue by geographic territory | SUM(invoice_amount) JOIN dim_company GROUP BY region | fact_invoice, dim_company | Period | Daily |
| Days Sales Outstanding | Average days to collect | AVG(days_to_payment) WHERE is_paid | fact_invoice | Rolling 90 days | Daily |
| Overdue Revenue | Unpaid past due invoices | SUM(invoice_amount) WHERE is_overdue | fact_invoice | Current | 4 hours |
| Revenue Growth Rate | Period over period change | (Current Revenue - Prior Revenue) / Prior Revenue | fact_invoice | Monthly | Monthly |
| Invoice Collection Rate | Paid vs total invoiced | SUM(paid) / SUM(total_invoiced) | fact_invoice | Period | Daily |

---

### 8.6 Profitability KPIs

| KPI Name | Definition | Formula | Source | Grain | Refresh |
|---|---|---|---|---|---|
| Gross Margin by Deal | Revenue minus cost per deal | SUM(revenue) - SUM(cost) GROUP BY deal_id | fact_invoice, fact_cost | Per deal | Daily |
| Gross Margin by Contract | Revenue minus cost per contract | SUM(revenue) - SUM(cost) GROUP BY sk_contract | fact_invoice, fact_cost | Per contract | Daily |
| Margin Percentage by Region | Margin rate by territory | Gross Margin / Revenue GROUP BY territory | fact_invoice, fact_cost, dim_company | Period | Weekly |
| Cost per Asset Managed | Total cost divided by assets | SUM(total_cost) / COUNT(active_assets) | fact_cost, dim_asset | Period | Weekly |
| Labor as Percent of Revenue | Labor cost relative to revenue | SUM(labor_cost) / SUM(revenue) | fact_cost, fact_invoice | Period | Weekly |
| Vendor Cost Efficiency | Vendor cost vs contract value | SUM(vendor_cost) / SUM(contract_value) | fact_cost, dim_contract | Per contract | Weekly |
| Contract Margin vs Target | Actual versus contracted margin | (Actual Margin % - contracted_margin) | fact_invoice, fact_cost, dim_contract | Per contract | Daily |

---

## 9. Common Analytical Query Patterns

### Pipeline Summary by Stage

```sql
SELECT
    d.deal_stage,
    COUNT(DISTINCT d.deal_id)              AS deal_count,
    SUM(d.estimated_contract_value)        AS total_value,
    SUM(d.weighted_value)                  AS weighted_value,
    AVG(d.estimated_contract_value)        AS avg_deal_size
FROM fact_deal d
WHERE d.is_closed_won = FALSE
  AND d.is_closed_lost = FALSE
GROUP BY d.deal_stage
ORDER BY d.stage_probability DESC;
```

### Monthly Revenue Trend

```sql
SELECT
    dt.calendar_year,
    dt.month_name,
    SUM(i.invoice_amount)                  AS revenue,
    COUNT(DISTINCT i.sk_contract)          AS contracts_billed
FROM fact_invoice i
JOIN dim_date dt ON dt.date_key = i.invoice_date_key
WHERE i.is_paid = TRUE
GROUP BY dt.calendar_year, dt.month_number, dt.month_name
ORDER BY dt.calendar_year, dt.month_number;
```

### Asset Service Frequency by Equipment Type

```sql
SELECT
    a.equipment_type,
    a.manufacturer,
    COUNT(m.sk_maintenance)                AS total_events,
    SUM(m.downtime_hours)                  AS total_downtime,
    SUM(m.repair_cost)                     AS total_cost,
    AVG(m.repair_cost)                     AS avg_cost_per_event
FROM fact_maintenance m
JOIN dim_asset a ON a.sk_asset = m.sk_asset AND a.is_current = TRUE
GROUP BY a.equipment_type, a.manufacturer
ORDER BY total_cost DESC;
```

### Contract Profitability Ranked

```sql
SELECT
    c.contract_name,
    comp.company_name,
    c.contract_value,
    SUM(i.invoice_amount)                  AS total_revenue,
    SUM(co.total_cost)                     AS total_cost,
    SUM(i.invoice_amount) - SUM(co.total_cost) AS gross_margin,
    ROUND((SUM(i.invoice_amount) - SUM(co.total_cost))
          / NULLIF(SUM(i.invoice_amount), 0) * 100, 2) AS margin_pct
FROM dim_contract c
JOIN dim_company comp    ON comp.sk_company    = c.sk_company AND comp.is_current = TRUE
JOIN fact_invoice i      ON i.sk_contract      = c.sk_contract
JOIN fact_cost co        ON co.sk_contract     = c.sk_contract
WHERE c.is_current = TRUE
GROUP BY c.contract_name, comp.company_name, c.contract_value
ORDER BY gross_margin DESC;
```

---

## 10. Warehouse Governance

### Refresh Schedule

| Object | Frequency | Window |
|---|---|---|
| dim_date | One-time plus annual extension | Off-hours |
| dim_company | Daily | 2:00 AM |
| dim_contact | Daily | 2:15 AM |
| dim_contract | Daily | 2:30 AM |
| dim_asset | Daily | 2:45 AM |
| dim_facility | Daily | 3:00 AM |
| dim_vendor | Weekly Sunday | 3:00 AM |
| fact_deal | Every 4 hours | :00 minutes |
| fact_invoice | Every 4 hours | :15 minutes |
| fact_cost | Daily | 4:00 AM |
| fact_maintenance | Every 4 hours | :30 minutes |

### Data Quality Checks — Run After Each Load

```sql
-- Orphaned invoices without deal lineage
SELECT COUNT(*) FROM fact_invoice WHERE deal_id IS NULL;

-- Orphaned costs without deal lineage
SELECT COUNT(*) FROM fact_cost WHERE deal_id IS NULL;

-- Assets without active contract
SELECT COUNT(*) FROM dim_asset
WHERE fk_contract_id IS NULL AND is_current = TRUE;

-- Invoices with invalid date keys
SELECT COUNT(*) FROM fact_invoice
WHERE invoice_date_key NOT IN (SELECT date_key FROM dim_date);

-- Contracts with value exceeding originating deal by more than 10%
SELECT c.contract_id, c.contract_value, d.final_contract_value
FROM dim_contract c
JOIN fact_deal d ON d.deal_id = c.fk_deal_id AND d.is_closed_won = TRUE
WHERE c.contract_value > d.final_contract_value * 1.10
  AND c.is_current = TRUE;
```

### Soft Delete Policy

No records are hard deleted from the warehouse. Deletions from the source system are handled as follows:

```sql
-- Mark as deleted rather than removing
UPDATE dim_{object}
SET is_deleted = TRUE,
    deleted_at = CURRENT_TIMESTAMP
WHERE {natural_key} = deleted_record_id;
```

All analytical queries filter WHERE is_deleted = FALSE unless explicitly analyzing deletions.

---

*End of Document — Analytical Warehouse Architecture*
*Cross-references: api_mapping_document.md | crm_implementation_guide.md*
