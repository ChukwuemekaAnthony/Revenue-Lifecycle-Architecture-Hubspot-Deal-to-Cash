# CRM API Data Mapping Document

**Version:** 1.0 | **Classification:** Internal Architecture Reference
**Cross-reference:** property_catalog.md | warehouse_architecture.md

---

## Table of Contents

1. Integration Architecture Overview
2. API Authentication and Connection
3. Layer 1 — CRM API Endpoints and Raw Extraction
4. Layer 2 — Staging Table Schema with Type Casting
5. Layer 3 — Transformation Rules and Warehouse Load Logic
6. Key Lineage Propagation Map
7. Incremental Load Strategy
8. Error Handling and Retry Logic
9. API Rate Limits and Batching Strategy
10. Integration Checklist

---

## 1. Integration Architecture Overview

The data integration pipeline moves data from the CRM platform through three structured layers before reaching the analytical warehouse.

```
CRM Platform (Source)
        |
        | REST API (HTTPS JSON)
        |
[LAYER 1] Raw Extraction
        Staging Tables (stg_*)
        Full JSON payload stored
        No transformation applied
        |
[LAYER 2] Type Casting and Cleansing
        Typed Staging Tables (stg_typed_*)
        Data types enforced
        Nulls handled
        Dedup logic applied
        |
[LAYER 3] Transformation and Warehouse Load
        Dimension Tables (dim_*)
        Fact Tables (fact_*)
        Surrogate keys assigned
        SCD logic applied
        Date keys joined
        |
[WAREHOUSE] Analytical Layer
        Star Schema
        KPI Queries
        BI Dashboards
```

### Integration Principles

- All ingestion uses upsert logic keyed on natural identifiers
- Pipelines are idempotent — re-running produces no duplicates
- All timestamps stored in UTC
- Soft deletes only — no hard deletes from warehouse
- Failed records logged to error table, not silently dropped

---

## 2. API Authentication and Connection

### Authentication Method

**OAuth 2.0 Private App Token**

The CRM platform uses private app tokens for server-to-server integration. Do not use API keys — they are being deprecated.

### Connection Parameters

| Parameter | Value |
|---|---|
| Base URL | https://api.hubapi.com |
| Authentication | Bearer token in Authorization header |
| Content-Type | application/json |
| Rate Limit | 100 requests per 10 seconds (standard) |
| Batch Size | Up to 100 records per request |
| Pagination | After cursor (hs_object_id based) |

### Header Template

```
Authorization: Bearer {PRIVATE_APP_TOKEN}
Content-Type: application/json
```

### Retry Policy

| Condition | Action |
|---|---|
| HTTP 429 Too Many Requests | Wait for Retry-After header value, then retry |
| HTTP 500 Server Error | Exponential backoff — 30s, 60s, 120s |
| HTTP 502 or 503 | Retry 3 times with 60s delay |
| HTTP 400 Bad Request | Log to error table, do not retry |
| Network timeout | Retry 3 times with 30s delay |

---

## 3. Layer 1 — CRM API Endpoints and Raw Extraction

This layer extracts raw JSON payloads from the CRM API and stores them in staging tables without transformation.

### 3.1 Company Endpoint

**Endpoint:** GET /crm/v3/objects/companies
**Staging Table:** stg_companies_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_company_id | Natural primary key |
| properties.name | raw_name | Company name |
| properties.parent_health_system | raw_parent_health_system | |
| properties.industry | raw_industry | |
| properties.region | raw_region | |
| properties.phone | raw_phone | |
| properties.city | raw_city | |
| properties.state | raw_state | |
| properties.zip | raw_zip | |
| properties.address | raw_address | |
| properties.bed_count | raw_bed_count | |
| properties.facility_count | raw_facility_count | |
| properties.client_status | raw_client_status | |
| properties.annual_equipment_budget | raw_annual_equipment_budget | |
| properties.account_owner | raw_account_owner | |
| properties.territory | raw_territory | |
| properties.createdate | raw_createdate | |
| properties.first_contract_date | raw_first_contract_date | |
| properties.hs_lastmodifieddate | raw_last_modified | Used for incremental loads |
| _extracted_at | _extracted_at | Pipeline audit timestamp |

---

### 3.2 Contact Endpoint

**Endpoint:** GET /crm/v3/objects/contacts
**Staging Table:** stg_contacts_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_contact_id | Natural primary key |
| properties.firstname | raw_firstname | |
| properties.lastname | raw_lastname | |
| properties.email | raw_email | Dedup key |
| properties.phone | raw_phone | |
| properties.jobtitle | raw_jobtitle | |
| properties.department | raw_department | |
| properties.role_type | raw_role_type | |
| properties.decision_authority | raw_decision_authority | |
| properties.hs_lead_status | raw_lead_status | |
| properties.createdate | raw_createdate | |
| properties.last_engagement_date | raw_last_engagement_date | |
| properties.hs_email_last_open_date | raw_email_last_open | |
| properties.associatedcompanyid | raw_company_id | Association FK |
| properties.hs_lastmodifieddate | raw_last_modified | |
| _extracted_at | _extracted_at | Pipeline audit |

---

### 3.3 Deal Endpoint

**Endpoint:** GET /crm/v3/objects/deals
**Staging Table:** stg_deals_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_deal_id | Natural primary key — propagates downstream |
| properties.dealname | raw_deal_name | |
| properties.pipeline | raw_pipeline | |
| properties.dealstage | raw_deal_stage | |
| properties.lead_source | raw_lead_source | |
| properties.service_type | raw_service_type | |
| properties.territory | raw_territory | |
| properties.account_owner | raw_account_owner | |
| properties.estimated_asset_volume | raw_estimated_asset_volume | |
| properties.asset_complexity_score | raw_asset_complexity_score | |
| properties.projected_service_cost | raw_projected_service_cost | |
| properties.service_scope | raw_service_scope | |
| properties.staffing_model | raw_staffing_model | |
| properties.vendor_strategy | raw_vendor_strategy | |
| properties.contract_term_years | raw_contract_term_years | |
| properties.pricing_model | raw_pricing_model | |
| properties.amount | raw_amount | |
| properties.final_contract_value | raw_final_contract_value | |
| properties.expected_margin | raw_expected_margin | |
| properties.negotiated_margin | raw_negotiated_margin | |
| properties.hs_deal_stage_probability | raw_stage_probability | |
| properties.createdate | raw_createdate | |
| properties.closedate | raw_closedate | |
| properties.closed_won_date | raw_closed_won_date | |
| properties.closed_lost_date | raw_closed_lost_date | |
| properties.contract_draft_sent_date | raw_contract_draft_sent | |
| properties.stage_entry_date | raw_stage_entry_date | |
| properties.loss_reason | raw_loss_reason | |
| properties.competitor | raw_competitor | |
| properties.hs_lastmodifieddate | raw_last_modified | |
| associations.company_id | raw_company_id | Association FK |
| associations.contact_ids | raw_contact_ids | JSON array |
| _extracted_at | _extracted_at | Pipeline audit |

---

### 3.4 Service Contract Endpoint

**Endpoint:** GET /crm/v3/objects/{contract_object_id}
**Staging Table:** stg_contracts_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_contract_id | Primary key |
| properties.contract_name | raw_contract_name | |
| properties.contract_status | raw_contract_status | |
| properties.service_model | raw_service_model | |
| properties.sla_type | raw_sla_type | |
| properties.service_scope | raw_service_scope | |
| properties.delivery_manager | raw_delivery_manager | |
| properties.billing_model | raw_billing_model | |
| properties.contract_value | raw_contract_value | |
| properties.contracted_margin | raw_contracted_margin | |
| properties.invoiced_to_date | raw_invoiced_to_date | |
| properties.renewal_probability | raw_renewal_probability | |
| properties.fk_deal_id | raw_deal_id | Critical lineage FK |
| properties.fk_company_id | raw_company_id | |
| properties.contract_signed_date | raw_contract_signed_date | |
| properties.contract_start_date | raw_contract_start_date | |
| properties.contract_end_date | raw_contract_end_date | |
| properties.renewal_date | raw_renewal_date | |
| properties.createdate | raw_createdate | |
| properties.hs_lastmodifieddate | raw_last_modified | |
| _extracted_at | _extracted_at | Pipeline audit |

---

### 3.5 Facility Endpoint

**Endpoint:** GET /crm/v3/objects/{facility_object_id}
**Staging Table:** stg_facilities_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_facility_id | |
| properties.facility_name | raw_facility_name | |
| properties.facility_type | raw_facility_type | |
| properties.department | raw_department | |
| properties.address | raw_address | |
| properties.city | raw_city | |
| properties.state | raw_state | |
| properties.zip_code | raw_zip_code | |
| properties.fk_company_id | raw_company_id | |
| properties.hs_lastmodifieddate | raw_last_modified | |
| _extracted_at | _extracted_at | |

---

### 3.6 Asset Endpoint

**Endpoint:** GET /crm/v3/objects/{asset_object_id}
**Staging Table:** stg_assets_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_asset_id | |
| properties.equipment_type | raw_equipment_type | |
| properties.manufacturer | raw_manufacturer | |
| properties.model_number | raw_model_number | |
| properties.serial_number | raw_serial_number | Dedup key |
| properties.asset_tag | raw_asset_tag | |
| properties.purchase_price | raw_purchase_price | |
| properties.lifecycle_stage | raw_lifecycle_stage | SCD Type 2 |
| properties.utilization_rate | raw_utilization_rate | |
| properties.service_risk_score | raw_service_risk_score | |
| properties.condition_rating | raw_condition_rating | |
| properties.fk_facility_id | raw_facility_id | |
| properties.fk_contract_id | raw_contract_id | |
| properties.fk_company_id | raw_company_id | |
| properties.installation_date | raw_installation_date | |
| properties.warranty_expiration | raw_warranty_expiration | |
| properties.last_service_date | raw_last_service_date | |
| properties.replacement_due_date | raw_replacement_due_date | |
| properties.createdate | raw_createdate | |
| properties.hs_lastmodifieddate | raw_last_modified | |
| _extracted_at | _extracted_at | |

---

### 3.7 Vendor Endpoint

**Endpoint:** GET /crm/v3/objects/{vendor_object_id}
**Staging Table:** stg_vendors_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_vendor_id | |
| properties.vendor_name | raw_vendor_name | |
| properties.vendor_type | raw_vendor_type | |
| properties.region | raw_region | |
| properties.service_capability | raw_service_capability | |
| properties.preferred_vendor | raw_preferred_vendor | |
| properties.vendor_contact_name | raw_vendor_contact_name | |
| properties.vendor_contact_email | raw_vendor_contact_email | |
| properties.hs_lastmodifieddate | raw_last_modified | |
| _extracted_at | _extracted_at | |

---

### 3.8 Vendor Service Contract Endpoint

**Endpoint:** GET /crm/v3/objects/{vendor_contract_object_id}
**Staging Table:** stg_vendor_contracts_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_vendor_contract_id | |
| properties.service_frequency | raw_service_frequency | |
| properties.sla_terms | raw_sla_terms | |
| properties.vendor_service_cost | raw_vendor_service_cost | |
| properties.cost_model | raw_cost_model | |
| properties.fk_vendor_id | raw_vendor_id | |
| properties.fk_asset_id | raw_asset_id | |
| properties.fk_contract_id | raw_contract_id | |
| properties.hs_lastmodifieddate | raw_last_modified | |
| _extracted_at | _extracted_at | |

---

### 3.9 Maintenance Event Endpoint

**Endpoint:** GET /crm/v3/objects/{maintenance_event_object_id}
**Staging Table:** stg_maintenance_events_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_event_id | |
| properties.maintenance_type | raw_maintenance_type | |
| properties.work_order_number | raw_work_order_number | |
| properties.technician_name | raw_technician_name | |
| properties.event_status | raw_event_status | |
| properties.downtime_hours | raw_downtime_hours | |
| properties.repair_notes | raw_repair_notes | |
| properties.parts_used | raw_parts_used | |
| properties.resolution_code | raw_resolution_code | |
| properties.repair_cost | raw_repair_cost | |
| properties.billable_flag | raw_billable_flag | |
| properties.fk_asset_id | raw_asset_id | |
| properties.fk_vendor_id | raw_vendor_id | |
| properties.fk_facility_id | raw_facility_id | |
| properties.fk_contract_id | raw_contract_id | |
| properties.maintenance_date | raw_maintenance_date | |
| properties.next_service_due_date | raw_next_service_due | |
| properties.createdate | raw_createdate | |
| properties.hs_lastmodifieddate | raw_last_modified | |
| _extracted_at | _extracted_at | |

---

### 3.10 Invoice Endpoint

**Endpoint:** GET /crm/v3/objects/{invoice_object_id}
**Staging Table:** stg_invoices_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_invoice_id | |
| properties.invoice_number | raw_invoice_number | |
| properties.payment_status | raw_payment_status | SCD Type 2 |
| properties.billing_type | raw_billing_type | |
| properties.invoice_amount | raw_invoice_amount | |
| properties.tax_amount | raw_tax_amount | |
| properties.total_amount | raw_total_amount | |
| properties.fk_contract_id | raw_contract_id | |
| properties.fk_deal_id | raw_deal_id | Critical lineage FK |
| properties.fk_company_id | raw_company_id | |
| properties.invoice_date | raw_invoice_date | |
| properties.payment_due_date | raw_payment_due_date | |
| properties.payment_received_date | raw_payment_received_date | |
| properties.createdate | raw_createdate | |
| properties.hs_lastmodifieddate | raw_last_modified | |
| _extracted_at | _extracted_at | |

---

### 3.11 Delivery Cost Endpoint

**Endpoint:** GET /crm/v3/objects/{delivery_cost_object_id}
**Staging Table:** stg_delivery_costs_raw

| CRM API Field | Raw Stage Column | Notes |
|---|---|---|
| hs_object_id | raw_cost_id | |
| properties.cost_category | raw_cost_category | |
| properties.cost_period | raw_cost_period | |
| properties.cost_description | raw_cost_description | |
| properties.labor_cost | raw_labor_cost | |
| properties.vendor_cost | raw_vendor_cost | |
| properties.equipment_cost | raw_equipment_cost | |
| properties.allocated_overhead | raw_allocated_overhead | |
| properties.total_cost | raw_total_cost | |
| properties.fk_contract_id | raw_contract_id | |
| properties.fk_deal_id | raw_deal_id | Critical lineage FK |
| properties.cost_recorded_date | raw_cost_recorded_date | |
| properties.cost_period_start | raw_cost_period_start | |
| properties.cost_period_end | raw_cost_period_end | |
| properties.createdate | raw_createdate | |
| properties.hs_lastmodifieddate | raw_last_modified | |
| _extracted_at | _extracted_at | |

---

## 4. Layer 2 — Staging Table Schema with Type Casting

This layer applies explicit data type casting, null handling, and deduplication. All API strings are cast to proper types.

### Type Casting Rules

| Source Type | Target Type | Cast Rule | Null Handling |
|---|---|---|---|
| String number | DECIMAL(18,2) | CAST(value AS DECIMAL) | Default to 0.00 if null |
| String number | INT | CAST(value AS INT) | Default to 0 if null |
| String date | DATE | CAST(value AS DATE) format YYYY-MM-DD | NULL allowed — do not default |
| String datetime | DATETIME | CAST(value AS DATETIME) UTC | NULL allowed |
| String boolean | BOOLEAN | Map 'true' to TRUE, 'false' to FALSE | Default to FALSE |
| String percent | DECIMAL(5,4) | DIVIDE(value, 100) | Default to 0.0000 |
| Enum string | VARCHAR(100) | Trim whitespace, uppercase | Default to UNKNOWN if null |
| UUID string | VARCHAR(36) | Preserve as-is | NULL not allowed on FK fields — reject record |

### Null Handling Policy

| Field Class | Null Policy |
|---|---|
| Primary Key | Never null — reject record if null |
| Foreign Key — required | Never null — reject and log to error table |
| Foreign Key — optional | Null allowed |
| Required text field | Reject record if null |
| Optional text field | NULL stored as NULL |
| Financial amount | Default to 0.00 if null |
| Date fields | NULL stored as NULL — never default date |
| Boolean fields | Default to FALSE if null |

### Typed Staging Tables

Each raw staging table produces a typed counterpart.

**Example: stg_typed_deals**

| Column | Data Type | Cast From | Null Handling |
|---|---|---|---|
| deal_id | VARCHAR(36) | raw_deal_id | Reject if null |
| deal_name | VARCHAR(255) | raw_deal_name | Reject if null |
| pipeline | VARCHAR(100) | raw_pipeline | Default UNKNOWN |
| deal_stage | VARCHAR(100) | raw_deal_stage | Reject if null |
| lead_source | VARCHAR(100) | raw_lead_source | NULL allowed |
| service_type | VARCHAR(100) | raw_service_type | NULL allowed |
| territory | VARCHAR(100) | raw_territory | NULL allowed |
| account_owner | VARCHAR(150) | raw_account_owner | NULL allowed |
| estimated_asset_volume | INT | raw_estimated_asset_volume | Default 0 |
| asset_complexity_score | DECIMAL(5,2) | raw_asset_complexity_score | Default 0.00 |
| projected_service_cost | DECIMAL(18,2) | raw_projected_service_cost | Default 0.00 |
| service_scope | TEXT | raw_service_scope | NULL allowed |
| contract_term_years | INT | raw_contract_term_years | Default 0 |
| amount | DECIMAL(18,2) | raw_amount | Default 0.00 |
| final_contract_value | DECIMAL(18,2) | raw_final_contract_value | Default 0.00 |
| expected_margin | DECIMAL(5,4) | raw_expected_margin / 100 | Default 0.0000 |
| negotiated_margin | DECIMAL(5,4) | raw_negotiated_margin / 100 | Default 0.0000 |
| stage_probability | DECIMAL(5,4) | raw_stage_probability / 100 | Default 0.0000 |
| createdate | DATETIME | raw_createdate | Reject if null |
| closedate | DATE | raw_closedate | NULL allowed |
| closed_won_date | DATE | raw_closed_won_date | NULL allowed |
| closed_lost_date | DATE | raw_closed_lost_date | NULL allowed |
| loss_reason | VARCHAR(100) | raw_loss_reason | NULL allowed |
| company_id | VARCHAR(36) | raw_company_id | Reject if null |
| last_modified | DATETIME | raw_last_modified | Reject if null |
| _extracted_at | DATETIME | _extracted_at | System |

---

## 5. Layer 3 — Transformation Rules and Warehouse Load Logic

### 5.1 Surrogate Key Assignment

All dimension tables receive a warehouse-generated surrogate key.

```sql
sk_{object} = ROW_NUMBER() OVER (ORDER BY natural_key)
```

Surrogate keys are stable integer sequences. Natural keys (CRM UUIDs) are preserved as alternate keys.

### 5.2 SCD Type Definitions

| Object | Field | SCD Type | Reason |
|---|---|---|---|
| Deal | deal_stage | Type 2 | Track stage history for pipeline velocity analysis |
| Deal | amount | Type 1 | Overwrite — no history needed |
| Service Contract | contract_status | Type 2 | Track status changes for lifecycle analysis |
| Asset | lifecycle_stage | Type 2 | Track lifecycle transitions |
| Asset | service_risk_score | Type 1 | Overwrite — point-in-time value |
| Invoice | payment_status | Type 2 | Track payment progression |
| Contact | role_type | Type 1 | Overwrite |
| Company | client_status | Type 2 | Track account status changes |

### SCD Type 2 Implementation Pattern

```sql
-- Insert new version when tracked field changes
INSERT INTO dim_deal
  (sk_deal, deal_id, deal_stage, ..., effective_date, expiry_date, is_current)
SELECT
  NEXT_SURROGATE_KEY(),
  deal_id,
  deal_stage,
  ...,
  CURRENT_DATE,
  '9999-12-31',
  TRUE
WHERE deal_stage <> (SELECT deal_stage FROM dim_deal
                     WHERE deal_id = incoming.deal_id AND is_current = TRUE);

-- Expire previous version
UPDATE dim_deal
SET expiry_date = CURRENT_DATE - 1, is_current = FALSE
WHERE deal_id = incoming.deal_id
  AND is_current = TRUE
  AND deal_stage <> incoming.deal_stage;
```

### 5.3 Date Key Joins

All date fields in fact tables are replaced with date_key integers.

```sql
-- Convert DATE to date_key
date_key = CAST(FORMAT(date_value, 'YYYYMMDD') AS INT)

-- Example: 2025-03-15 becomes 20250315
```

Date keys join to the dim_date table for time intelligence.

### 5.4 Warehouse Table Load Mapping

#### dim_company

| Warehouse Column | Source | Transform |
|---|---|---|
| sk_company | Generated | Surrogate key |
| company_id | stg_typed_companies.company_id | Natural key |
| company_name | stg_typed_companies.company_name | Trim whitespace |
| parent_health_system | stg_typed_companies.parent_health_system | As-is |
| industry | stg_typed_companies.industry | As-is |
| region | stg_typed_companies.region | As-is |
| bed_count | stg_typed_companies.bed_count | As-is |
| facility_count | stg_typed_companies.facility_count | As-is |
| client_status | stg_typed_companies.client_status | SCD Type 2 tracked |
| annual_equipment_budget | stg_typed_companies.annual_equipment_budget | As-is |
| created_date_key | stg_typed_companies.createdate | Convert to YYYYMMDD INT |
| first_contract_date_key | stg_typed_companies.first_contract_date | Convert to YYYYMMDD INT |
| effective_date | Load date | SCD Type 2 |
| expiry_date | 9999-12-31 default | SCD Type 2 |
| is_current | TRUE default | SCD Type 2 |

#### fact_deal

| Warehouse Column | Source | Transform |
|---|---|---|
| sk_deal | Generated | Surrogate key |
| deal_id | stg_typed_deals.deal_id | Natural key |
| sk_company | dim_company.sk_company | Lookup by company_id |
| sk_deal_stage | dim_deal_stage.sk | Lookup by stage name |
| deal_name | stg_typed_deals.deal_name | As-is |
| pipeline | stg_typed_deals.pipeline | As-is |
| deal_stage | stg_typed_deals.deal_stage | As-is |
| estimated_contract_value | stg_typed_deals.amount | As-is |
| final_contract_value | stg_typed_deals.final_contract_value | As-is |
| expected_margin | stg_typed_deals.expected_margin | Stored as decimal |
| stage_probability | stg_typed_deals.stage_probability | Stored as decimal |
| weighted_value | Calculated | amount x stage_probability |
| created_date_key | stg_typed_deals.createdate | YYYYMMDD INT |
| close_date_key | stg_typed_deals.closedate | YYYYMMDD INT — NULL if null |
| closed_won_date_key | stg_typed_deals.closed_won_date | YYYYMMDD INT — NULL if null |
| closed_lost_date_key | stg_typed_deals.closed_lost_date | YYYYMMDD INT — NULL if null |
| is_closed_won | Calculated | TRUE if deal_stage = Closed Won |
| is_closed_lost | Calculated | TRUE if deal_stage = Closed Lost |

#### fact_invoice

| Warehouse Column | Source | Transform |
|---|---|---|
| sk_invoice | Generated | Surrogate key |
| invoice_id | stg_typed_invoices.invoice_id | Natural key |
| sk_company | dim_company.sk_company | Lookup |
| sk_contract | dim_contract.sk_contract | Lookup by contract_id |
| deal_id | stg_typed_invoices.deal_id | Preserved for lineage |
| invoice_number | stg_typed_invoices.invoice_number | As-is |
| payment_status | stg_typed_invoices.payment_status | SCD Type 2 |
| billing_type | stg_typed_invoices.billing_type | As-is |
| invoice_amount | stg_typed_invoices.invoice_amount | As-is |
| tax_amount | stg_typed_invoices.tax_amount | As-is |
| total_amount | stg_typed_invoices.total_amount | As-is |
| invoice_date_key | stg_typed_invoices.invoice_date | YYYYMMDD INT |
| payment_due_date_key | stg_typed_invoices.payment_due_date | YYYYMMDD INT |
| payment_received_date_key | stg_typed_invoices.payment_received_date | YYYYMMDD INT — NULL if null |
| days_to_payment | Calculated | payment_received_date - invoice_date |
| is_paid | Calculated | TRUE if payment_status = Payment Received |
| is_overdue | Calculated | TRUE if payment_due_date < today AND NOT is_paid |

#### fact_cost

| Warehouse Column | Source | Transform |
|---|---|---|
| sk_cost | Generated | Surrogate key |
| cost_id | stg_typed_costs.cost_id | Natural key |
| sk_contract | dim_contract.sk_contract | Lookup |
| deal_id | stg_typed_costs.deal_id | Preserved for lineage |
| cost_category | stg_typed_costs.cost_category | As-is |
| labor_cost | stg_typed_costs.labor_cost | As-is |
| vendor_cost | stg_typed_costs.vendor_cost | As-is |
| equipment_cost | stg_typed_costs.equipment_cost | As-is |
| allocated_overhead | stg_typed_costs.allocated_overhead | As-is |
| total_cost | Calculated | SUM(labor + vendor + equipment + overhead) |
| cost_recorded_date_key | stg_typed_costs.cost_recorded_date | YYYYMMDD INT |

#### fact_maintenance

| Warehouse Column | Source | Transform |
|---|---|---|
| sk_maintenance | Generated | Surrogate key |
| maintenance_event_id | stg_typed_maintenance.event_id | Natural key |
| sk_asset | dim_asset.sk_asset | Lookup |
| sk_vendor | dim_vendor.sk_vendor | Lookup — nullable |
| sk_facility | dim_facility.sk_facility | Lookup |
| sk_contract | dim_contract.sk_contract | Lookup |
| maintenance_type | stg_typed_maintenance.maintenance_type | As-is |
| work_order_number | stg_typed_maintenance.work_order_number | As-is |
| downtime_hours | stg_typed_maintenance.downtime_hours | As-is |
| repair_cost | stg_typed_maintenance.repair_cost | As-is |
| billable_flag | stg_typed_maintenance.billable_flag | As-is |
| resolution_code | stg_typed_maintenance.resolution_code | As-is |
| maintenance_date_key | stg_typed_maintenance.maintenance_date | YYYYMMDD INT |
| next_service_due_date_key | stg_typed_maintenance.next_service_due | YYYYMMDD INT |

---

## 6. Key Lineage Propagation Map

The following illustrates how the deal_id flows through the warehouse.

```
stg_typed_deals.deal_id
      |
      | (1:1 via workflow)
      v
stg_typed_contracts.deal_id  -->  fact_contract.deal_id
      |
      | (1:N)
      |-----> stg_typed_invoices.deal_id  -->  fact_invoice.deal_id
      |
      |-----> stg_typed_costs.deal_id     -->  fact_cost.deal_id
      |
      | (1:N via contract_id)
      v
stg_typed_assets.contract_id  -->  dim_asset.fk_contract_id
      |
      | (1:N)
      v
stg_typed_maintenance.contract_id  -->  fact_maintenance.sk_contract
```

### Lineage Validation Query

Run after each load to verify lineage integrity:

```sql
-- Check for invoices without deal lineage
SELECT COUNT(*) AS orphaned_invoices
FROM fact_invoice
WHERE deal_id IS NULL OR deal_id = '';

-- Check for costs without deal lineage
SELECT COUNT(*) AS orphaned_costs
FROM fact_cost
WHERE deal_id IS NULL OR deal_id = '';

-- Check contracts without originating deal
SELECT COUNT(*) AS orphaned_contracts
FROM dim_contract
WHERE fk_deal_id IS NULL OR fk_deal_id = '';
```

Expected result: 0 for all queries. Any non-zero result triggers a pipeline alert.

---

## 7. Incremental Load Strategy

### Full Load — Initial Only

Run once on first deployment. Extracts all records from CRM with no date filter.

### Incremental Load — Daily

Uses hs_lastmodifieddate for change detection.

```
Filter: hs_lastmodifieddate >= last_successful_load_timestamp
```

Watermark table tracks last successful load per object:

```
pipeline_watermarks
  object_name     VARCHAR(100)
  last_load_time  DATETIME
```

### Load Frequency by Object

| Object | Frequency | Reason |
|---|---|---|
| Company | Daily | Low change frequency |
| Contact | Daily | Low change frequency |
| Deal | Every 4 hours | Stage changes throughout day |
| Service Contract | Daily | Moderate change frequency |
| Asset | Daily | Lifecycle stage updates |
| Maintenance Event | Every 4 hours | Operational activity |
| Invoice | Every 4 hours | Payment status changes |
| Delivery Cost | Daily | Periodic cost entries |
| Vendor | Weekly | Rare changes |

---

## 8. Error Handling and Retry Logic

### Error Table Schema

```
pipeline_errors
  error_id          UUID
  pipeline_name     VARCHAR(100)
  object_type       VARCHAR(100)
  record_id         VARCHAR(36)
  error_type        VARCHAR(100)
  error_message     TEXT
  raw_payload       TEXT
  occurred_at       DATETIME
  resolved          BOOLEAN
  resolved_at       DATETIME
```

### Error Types

| Error Type | Description | Action |
|---|---|---|
| NULL_REQUIRED_FIELD | Required field is null | Log, reject record, continue |
| DUPLICATE_NATURAL_KEY | Duplicate PK on insert | Log, skip record, continue |
| FK_NOT_FOUND | Foreign key not in dimension | Log, hold record for retry |
| TYPE_CAST_FAILURE | Cannot cast to target type | Log, reject record |
| API_RATE_LIMIT | HTTP 429 received | Pause, retry after backoff |
| API_SERVER_ERROR | HTTP 500 received | Retry 3 times, then alert |

---

## 9. API Rate Limits and Batching Strategy

### Standard Rate Limits

| Tier | Limit |
|---|---|
| Standard | 100 requests per 10 seconds |
| Burst | 150 requests per 10 seconds for up to 30 seconds |

### Batch Request Strategy

Use the Batch Read API for large object sets:

**Endpoint:** POST /crm/v3/objects/{object}/batch/read

```json
{
  "properties": ["field1", "field2"],
  "inputs": [
    {"id": "record_id_1"},
    {"id": "record_id_2"}
  ]
}
```

Maximum 100 records per batch request.

### Pagination Pattern

Use the after cursor for full object scans:

```
GET /crm/v3/objects/deals?limit=100&after={cursor}
```

Response includes paging.next.after cursor. Continue until no cursor returned.

---

## 10. Integration Checklist

### Layer 1 — Raw Extraction Setup

- [ ] Private app token created and stored in secrets manager
- [ ] Custom object IDs retrieved for all custom objects
- [ ] Raw staging tables created in warehouse
- [ ] Pipeline audit column _extracted_at added to all staging tables
- [ ] Watermark table created and initialized
- [ ] API connection tested for all 11 endpoints
- [ ] Pagination implemented for all object extracts
- [ ] Batch read API implemented where applicable

### Layer 2 — Type Casting and Cleansing

- [ ] Typed staging tables created for all objects
- [ ] Type casting rules applied per field
- [ ] Null handling rules implemented
- [ ] Dedup logic implemented for Company and Contact
- [ ] Error table created and error routing implemented
- [ ] Percent fields divided by 100 on cast

### Layer 3 — Warehouse Load

- [ ] Surrogate key sequences created for all dimension tables
- [ ] SCD Type 2 logic implemented for tracked fields
- [ ] Date key conversion applied to all date fields
- [ ] dim_date table populated with 10-year range minimum
- [ ] deal_id lineage propagation verified
- [ ] Lineage validation queries scheduled post-load
- [ ] Weighted value calculation confirmed in fact_deal
- [ ] is_paid and is_overdue flags confirmed in fact_invoice
- [ ] total_cost calculation confirmed in fact_cost

---

*End of Document — API Data Mapping Document*
*Cross-references: property_catalog.md | warehouse_architecture.md*
