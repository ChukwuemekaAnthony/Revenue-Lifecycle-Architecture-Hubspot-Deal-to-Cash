# CRM Revenue Lifecycle Architecture — Implementation Checklist

**Version:** 1.0 | **Classification:** Internal Architecture Reference
**Cross-reference:** crm_implementation_guide.md

---

## How to Use This Checklist

Work through phases in order. Each phase has a dependency on the prior phase being complete. Tasks marked as **[BLOCKER]** must be completed before the phase can advance.

Mark each item with:
- [ ] Not started
- [x] Complete
- [~] In progress

---

## Phase 1 — CRM Foundation

**Objective:** Configure the CRM platform base layer with correct account structure, standard object settings, and user access.
**Depends On:** Nothing — this is the starting point.

### 1.1 Account and License Setup

- [ ] Confirm Operations Hub Enterprise license is active — required for custom objects **[BLOCKER]**
- [ ] Confirm Sales Hub license tier
- [ ] Set up production portal — not sandbox — as primary implementation environment
- [ ] Create a sandbox portal for testing before production changes

### 1.2 User and Team Configuration

- [ ] Create user accounts for all CRM users
- [ ] Define user roles — Admin, Sales Rep, Operations, Finance, Read-Only
- [ ] Assign permissions per role
- [ ] Configure team structure matching sales territories
- [ ] Assign account owners to teams

### 1.3 Company Object Configuration

- [ ] Confirm all required properties exist on Company object
- [ ] Create custom properties: parent_health_system, bed_count, facility_count, annual_equipment_budget, client_status, territory
- [ ] Organize properties into groups: company_info, company_operational, company_dates
- [ ] Configure duplicate detection rule: company_name plus zip code
- [ ] Set required fields: name, industry, region, city, state, zip, client_status, account_owner

### 1.4 Contact Object Configuration

- [ ] Confirm all required properties exist on Contact object
- [ ] Create custom properties: department, role_type, decision_authority, last_engagement_date
- [ ] Organize into groups: contact_info, contact_role, contact_dates
- [ ] Configure duplicate detection: email exact match **[BLOCKER]**
- [ ] Set required fields: firstname, lastname, email, role_type

### 1.5 Deal Object Configuration

- [ ] Create all custom deal properties — see property_catalog.md Deal section **[BLOCKER]**
- [ ] Organize into groups: deal_info, deal_scope, deal_financials, deal_dates, deal_loss
- [ ] Set required fields per pipeline stage — see governance rules G-D-01 through G-D-05
- [ ] Configure deal name format standard
- [ ] Validate that HubSpot native amount field maps to estimated_contract_value

---

## Phase 2 — Custom Object Creation

**Objective:** Create all six custom objects with correct schema, properties, and association rules.
**Depends On:** Phase 1 complete. Operations Hub Enterprise license confirmed.

### 2.1 Service Contract Object

- [ ] Create custom object: Service Contract **[BLOCKER]**
- [ ] Create all properties per property_catalog.md Contract section
- [ ] Organize into groups: contract_info, contract_financials, contract_lineage, contract_dates
- [ ] Set required fields: contract_name, contract_status, service_model, sla_type, contract_value, fk_deal_id, fk_company_id, contract_start_date, contract_end_date
- [ ] Create association: Deal (one-to-one) — label Generated Contract **[BLOCKER]**
- [ ] Create association: Company (many-to-one) — label Has Contracts
- [ ] Create pipeline: Contract Lifecycle Pipeline with 8 stages — see implementation guide section 8.2

### 2.2 Facility Object

- [ ] Create custom object: Facility **[BLOCKER]**
- [ ] Create all properties per property_catalog.md Facility section
- [ ] Set required fields: facility_name, facility_type, address, city, state, zip_code, fk_company_id
- [ ] Create association: Company (many-to-one) — label Has Facilities
- [ ] Create association: Service Contract (many-to-many) — label Covers Facilities

### 2.3 Asset Object

- [ ] Create custom object: Asset — Medical Equipment **[BLOCKER]**
- [ ] Create all properties per property_catalog.md Asset section
- [ ] Organize into groups: asset_info, asset_lifecycle, asset_lineage, asset_dates
- [ ] Set serial_number as globally unique — enable uniqueness validation **[BLOCKER]**
- [ ] Set required fields: equipment_type, manufacturer, model_number, serial_number, lifecycle_stage, fk_facility_id, fk_contract_id, installation_date
- [ ] Create association: Facility (many-to-one) — label Contains Assets
- [ ] Create association: Service Contract (many-to-one) — label Covered By Contract

### 2.4 Vendor Object

- [ ] Create custom object: Vendor
- [ ] Create all properties per property_catalog.md Vendor section
- [ ] Set required fields: vendor_name, vendor_type
- [ ] No direct CRM associations at object level — associated via Vendor Service Contract

### 2.5 Vendor Service Contract Object

- [ ] Create custom object: Vendor Service Contract
- [ ] Create all properties per property_catalog.md Vendor Service Contract section
- [ ] Set required fields: service_frequency, sla_terms, vendor_service_cost, fk_vendor_id, fk_asset_id, fk_contract_id
- [ ] Create association: Vendor (many-to-one) — label Provides Service
- [ ] Create association: Asset (many-to-one) — label Covered By
- [ ] Create association: Service Contract (many-to-one) — label Under Contract

### 2.6 Maintenance Event Object

- [ ] Create custom object: Maintenance Event **[BLOCKER]**
- [ ] Create all properties per property_catalog.md Maintenance Event section
- [ ] Organize into groups: event_info, event_operational, event_financial, event_lineage, event_dates
- [ ] Set required fields: maintenance_type, work_order_number, event_status, fk_asset_id, fk_facility_id, fk_contract_id, maintenance_date
- [ ] Create pipeline: Service Event Pipeline with 6 stages — see implementation guide section 8.3
- [ ] Create association: Asset (many-to-one) — label Has Service Events
- [ ] Create association: Vendor (many-to-one) — label Serviced By — nullable
- [ ] Create association: Facility (many-to-one)
- [ ] Create association: Service Contract (many-to-one)

### 2.7 Invoice Object

- [ ] Create custom object: Invoice **[BLOCKER]**
- [ ] Create all properties per property_catalog.md Invoice section
- [ ] Set required fields: invoice_number, payment_status, billing_type, invoice_amount, fk_contract_id, fk_deal_id, invoice_date, payment_due_date
- [ ] Create pipeline: Billing Pipeline with 6 stages — see implementation guide section 8.4
- [ ] Create association: Service Contract (many-to-one) — label Billed As
- [ ] Create association: Company (many-to-one)
- [ ] Validate fk_deal_id is populated automatically from contract on creation

### 2.8 Delivery Cost Object

- [ ] Create custom object: Delivery Cost
- [ ] Create all properties per property_catalog.md Delivery Cost section
- [ ] Set required fields: cost_category, fk_contract_id, fk_deal_id, cost_recorded_date
- [ ] Create association: Service Contract (many-to-one) — label Incurred Costs
- [ ] Validate fk_deal_id is populated from contract on creation

---

## Phase 3 — Pipeline and Automation Setup

**Objective:** Configure sales pipeline stages, automation workflows, and governance validation rules.
**Depends On:** Phase 2 complete — all custom objects created.

### 3.1 Sales Pipeline Configuration

- [ ] Create Sales Pipeline on Deal object **[BLOCKER]**
- [ ] Create all 9 stages per implementation guide section 8.1
- [ ] Set probability for each stage: 5, 15, 30, 50, 65, 80, 90, 100, 0
- [ ] Configure required fields per stage using conditional property logic
- [ ] Test stage advancement blocking — confirm fields enforce before progression

### 3.2 Workflow: Deal Won to Contract

- [ ] Create workflow triggered by Deal stage = Closed Won **[BLOCKER]**
- [ ] Action: Create Service Contract record
- [ ] Map fields: deal_id, company_id, contract_value, service_scope, contract_term_years
- [ ] Set initial contract_status = Contract Initiated
- [ ] Test workflow end-to-end in sandbox before production

### 3.3 Workflow: Contract Active to Asset Onboarding

- [ ] Create workflow triggered by Service Contract stage = Active Operations
- [ ] Action: Create enrollment task for asset inventory
- [ ] Notify delivery manager
- [ ] Test in sandbox

### 3.4 Workflow: Warranty Expiry Alert

- [ ] Create workflow triggered by asset.warranty_expiration within 60 days
- [ ] Action: Create task for asset replacement review
- [ ] Assign to account owner

### 3.5 Workflow: Maintenance Due Alert

- [ ] Create workflow triggered by asset.next_service_due_date within 14 days
- [ ] Action: Create Maintenance Event record
- [ ] Set event_status = Service Request Logged
- [ ] Populate fk_asset_id, fk_facility_id, fk_contract_id automatically

### 3.6 Workflow: Invoice Overdue Alert

- [ ] Create workflow triggered daily when payment_due_date is past and payment_status is not Payment Received
- [ ] Action: Update payment_status to Overdue
- [ ] Send notification to billing team

### 3.7 Workflow: Contract Renewal Alert

- [ ] Create workflow triggered by contract_end_date within 90 days
- [ ] Action: Create renewal Deal record
- [ ] Link to originating contract
- [ ] Notify account owner

### 3.8 Workflow: Asset Replacement Due

- [ ] Create workflow triggered by asset.replacement_due_date within 90 days
- [ ] Action: Create capital planning task

### 3.9 Governance Rule Testing

- [ ] Test G-D-01: Block deal from advancing past Qualification without estimated_contract_value
- [ ] Test G-D-03: Block deal from Closed Won without contract_value and expected_margin
- [ ] Test G-D-04: Confirm loss_reason is required on Closed Lost
- [ ] Test G-C-01: Block contract from Active Operations without associated asset
- [ ] Test G-A-01: Confirm serial_number uniqueness enforcement
- [ ] Test G-I-01: Confirm invoice cannot generate without active contract
- [ ] Test G-I-02: Confirm invoice amount must be greater than zero

---

## Phase 4 — Data Integration and API Pipelines

**Objective:** Build and validate the data ingestion pipeline from CRM to warehouse.
**Depends On:** Phase 3 complete. Warehouse environment provisioned.

### 4.1 Environment Setup

- [ ] Provision warehouse environment — staging schema, dimensions schema, facts schema
- [ ] Create private app token in CRM portal — store in secrets manager **[BLOCKER]**
- [ ] Retrieve custom object type IDs for all 6 custom objects **[BLOCKER]**
- [ ] Create watermark table in warehouse
- [ ] Create pipeline_errors table per api_mapping_document.md schema

### 4.2 Layer 1 — Raw Staging Tables

- [ ] Create stg_companies_raw
- [ ] Create stg_contacts_raw
- [ ] Create stg_deals_raw
- [ ] Create stg_contracts_raw
- [ ] Create stg_facilities_raw
- [ ] Create stg_assets_raw
- [ ] Create stg_vendors_raw
- [ ] Create stg_vendor_contracts_raw
- [ ] Create stg_maintenance_events_raw
- [ ] Create stg_invoices_raw
- [ ] Create stg_delivery_costs_raw
- [ ] Add _extracted_at audit column to all raw tables
- [ ] Test API connection for all 11 endpoints **[BLOCKER]**
- [ ] Run full initial load for all objects

### 4.3 Layer 2 — Typed Staging and Cleansing

- [ ] Create typed staging tables for all objects
- [ ] Implement type casting rules per api_mapping_document.md Layer 2
- [ ] Implement null handling rules — reject, default, or allow per field class
- [ ] Implement dedup logic for Company and Contact
- [ ] Implement percent field division: divide by 100 on cast
- [ ] Route rejected records to pipeline_errors table
- [ ] Validate zero records in error table on clean test run

### 4.4 Layer 3 — Warehouse Load

- [ ] Create surrogate key sequences for all dimension tables
- [ ] Implement SCD Type 2 logic for: deal_stage, contract_status, asset.lifecycle_stage, invoice.payment_status, company.client_status
- [ ] Implement date key conversion: all date fields to YYYYMMDD INT
- [ ] Populate dim_date table with minimum 10-year range
- [ ] Load all dimension tables
- [ ] Load all fact tables
- [ ] Run lineage validation queries — confirm zero orphaned records **[BLOCKER]**
- [ ] Confirm weighted_value calculation in fact_deal
- [ ] Confirm is_paid and is_overdue flags in fact_invoice
- [ ] Confirm total_cost calculation in fact_cost

### 4.5 Incremental Load Setup

- [ ] Configure incremental load using hs_lastmodifieddate watermark
- [ ] Set load frequency per api_mapping_document.md section 7 schedule
- [ ] Implement upsert logic — no duplicate records on rerun
- [ ] Test incremental load: update a record in CRM, confirm warehouse reflects change within one cycle
- [ ] Set up pipeline monitoring and alerting

---

## Phase 5 — Warehouse Build and Validation

**Objective:** Validate the complete warehouse schema, confirm all relationships, and run data quality checks.
**Depends On:** Phase 4 complete — all tables populated with initial data.

### 5.1 Dimension Validation

- [ ] Confirm dim_company has no duplicate company_id with is_current = TRUE
- [ ] Confirm dim_asset has no duplicate serial_number with is_current = TRUE
- [ ] Confirm dim_contract all have valid fk_deal_id
- [ ] Confirm all dim tables have sk surrogate keys populated
- [ ] Confirm SCD Type 2 tables have proper effective_date and expiry_date

### 5.2 Fact Table Validation

- [ ] Confirm fact_deal: all deals have sk_company joined correctly
- [ ] Confirm fact_invoice: all records have deal_id populated
- [ ] Confirm fact_cost: all records have deal_id populated
- [ ] Confirm fact_maintenance: all records have sk_asset, sk_facility, sk_contract
- [ ] Run all data quality check queries from warehouse_architecture.md section 10 **[BLOCKER]**
- [ ] Confirm all quality checks return zero

### 5.3 Revenue Lineage Validation

- [ ] Select 3 closed won deals
- [ ] Trace each deal_id through contract, invoice, and cost tables
- [ ] Confirm gross margin calculation is mathematically correct
- [ ] Confirm margin_pct is within expected range for those deals

### 5.4 KPI Baseline Run

- [ ] Run all Sales Pipeline KPI queries — confirm non-null results
- [ ] Run all Revenue KPI queries
- [ ] Run all Profitability KPI queries
- [ ] Document baseline values for QA comparison

---

## Phase 6 — Analytics and Dashboard Enablement

**Objective:** Connect the warehouse to the BI platform and build core reporting dashboards.
**Depends On:** Phase 5 complete — warehouse validated.

### 6.1 BI Platform Connection

- [ ] Connect BI platform to warehouse
- [ ] Import or create connections for all fact and dimension tables
- [ ] Configure date table — mark dim_date as date table with date_key as key
- [ ] Test basic aggregation: total revenue, deal count — confirm matches warehouse query

### 6.2 Core Dashboards

- [ ] Build Sales Pipeline Dashboard
  - [ ] Pipeline by stage bar chart
  - [ ] Weighted pipeline trend
  - [ ] Win rate by period
  - [ ] Average deal size trend
  - [ ] Sales cycle length trend

- [ ] Build Revenue Dashboard
  - [ ] Monthly revenue trend
  - [ ] Revenue by contract
  - [ ] Revenue by region
  - [ ] DSO (Days Sales Outstanding) trend
  - [ ] Overdue invoice tracker

- [ ] Build Profitability Dashboard
  - [ ] Gross margin by deal
  - [ ] Gross margin by contract
  - [ ] Margin vs target variance
  - [ ] Cost breakdown by category
  - [ ] Deal profitability ranking

- [ ] Build Operations Dashboard
  - [ ] Asset count by lifecycle stage
  - [ ] Maintenance events by type
  - [ ] Mean time between failures
  - [ ] Downtime hours trend
  - [ ] Assets nearing replacement

### 6.3 Dashboard Validation

- [ ] Reconcile dashboard totals against warehouse query results for the same period
- [ ] Confirm all filters function correctly
- [ ] Confirm date slicer operates on fiscal year if applicable
- [ ] Conduct user acceptance review with data owners

---

## Phase 7 — Governance and QA

**Objective:** Establish ongoing data governance, access control, and quality monitoring.
**Depends On:** Phase 6 complete — dashboards live.

### 7.1 Data Governance

- [ ] Assign data owners per governance framework in implementation guide section 14
- [ ] Assign data stewards per object
- [ ] Document escalation path for data quality issues
- [ ] Publish data retention policy to relevant teams

### 7.2 Quality Monitoring

- [ ] Schedule lineage validation queries to run daily after each load
- [ ] Configure pipeline alerts for any non-zero lineage validation results
- [ ] Configure alert for pipeline_errors exceeding threshold
- [ ] Establish monthly data quality review meeting

### 7.3 Access Control

- [ ] Configure row-level security on BI dashboards where required
- [ ] Restrict warehouse access by role
- [ ] Confirm CRM users can only see records appropriate to their role
- [ ] Document access control matrix

### 7.4 Documentation Handover

- [ ] Confirm all five architecture documents are published and accessible **[BLOCKER]**
- [ ] Conduct walkthrough session with data stewards
- [ ] Conduct walkthrough session with dashboard users
- [ ] Record architecture overview session for future onboarding
- [ ] Establish change management process for schema updates

### 7.5 Final Sign-Off

- [ ] Revenue lineage confirmed end-to-end by finance owner
- [ ] Pipeline governance rules confirmed by sales leadership
- [ ] Warehouse quality checks all passing
- [ ] Dashboards signed off by business stakeholders
- [ ] Implementation complete — move to BAU support model

---

*End of Document — Implementation Checklist*
*Cross-references: crm_implementation_guide.md | property_catalog.md | api_mapping_document.md | warehouse_architecture.md*
