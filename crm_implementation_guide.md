# CRM Revenue Lifecycle Architecture — Implementation Guide

**Version:** 1.0 | **Classification:** Internal Architecture Reference

---

## Document References

| Document | Purpose |
|---|---|
| property_catalog.md | Full field-level schema for all CRM objects |
| api_mapping_document.md | Ingestion pipeline from CRM to warehouse |
| warehouse_architecture.md | Star schema, fact tables, KPI catalog |
| implementation_checklist.md | Phased deployment checklist |
| erd.mmd | Entity relationship diagram |
| pipeline_flow.mmd | Lifecycle flow diagram |

---

## Table of Contents

1. System Overview
2. Architecture Objectives
3. Architecture Principles
4. Naming Convention Standard
5. Architecture Domains
6. Business Object Model
7. Association Schema
8. Pipeline Architecture
9. Contract-to-Cash Subprocess
10. Automation Workflow Definitions
11. Pipeline Governance Rules
12. Revenue Lineage Model
13. Forecasting Model
14. Data Governance Framework
15. Implementation Checklist Summary

---

## 1. System Overview

This document describes the architecture and implementation guidance for a CRM platform designed to manage the complete revenue lifecycle of a service-based organization operating in asset-intensive environments.

The architecture extends traditional CRM functionality beyond opportunity management to encompass:

- Contract lifecycle management
- Operational service delivery
- Asset lifecycle tracking
- Vendor service coordination
- Billing and revenue realization
- Cost attribution and profitability analysis

### Core Lifecycle Principle

The system maintains a single unbroken lineage from first commercial contact through to margin realization:

```
Lead → Deal → Contract → Asset → Service Event → Invoice → Payment → Margin
```

Every record in the system traces back to an originating deal. This lineage is preserved through consistent propagation of key identifiers across all downstream objects.

---

## 2. Architecture Objectives

| # | Objective | Description |
|---|---|---|
| 1 | Revenue Lineage | Every invoice and cost traces to an originating deal |
| 2 | Operational Continuity | Sales pipeline connects directly to delivery execution |
| 3 | Profitability Analytics | Margin computable at deal, contract, and asset level |
| 4 | Data Governance | Field-level validation and pipeline stage rules enforced |
| 5 | Scalable Analytics | Warehouse model supports enterprise reporting |
| 6 | System Integration | CRM connects to ERP, CMMS, and billing platforms |

---

## 3. Architecture Principles

**Principle 1 — Single Source of Truth**
The CRM is the system of engagement for commercial and operational data. All downstream systems receive data from or report back to this platform.

**Principle 2 — Lineage Preservation**
The deal_id propagates through every downstream object. No contract, invoice, or cost record exists without a traceable deal origin.

**Principle 3 — Separation of Concerns**
Standard objects handle commercial activity. Custom objects handle operational activity. These are distinct layers with defined handoff triggers.

**Principle 4 — Grain Integrity**
Every fact table in the warehouse declares an explicit grain. Aggregation never occurs at the wrong level.

**Principle 5 — Governance First**
No deal advances pipeline stage without required field validation. No contract activates without completion gates. No invoice generates without an active contract.

**Principle 6 — Idempotent Integration**
API pipelines are designed to be re-runnable without creating duplicate records. All ingestion uses upsert logic keyed on natural identifiers.

---

## 4. Naming Convention Standard

| Convention | Standard | Example |
|---|---|---|
| Field names | snake_case | contract_start_date |
| Date key format | INTEGER YYYYMMDD | 20250101 |
| Primary keys | UUID | hs_object_id |
| Surrogate keys | sk_ prefix | sk_deal |
| Foreign keys | fk_ prefix | fk_company_id |
| Staging tables | stg_ prefix | stg_deals |
| Dimension tables | dim_ prefix | dim_company |
| Fact tables | fact_ prefix | fact_invoice |
| Property groups | lowercase_underscore | deal_financials |

---

## 5. Architecture Domains

| Domain | Objects |
|---|---|
| Customer Domain | Company, Contact |
| Sales Domain | Deal |
| Contract Domain | Service Contract |
| Facility Domain | Facility |
| Asset Lifecycle Domain | Asset |
| Service Operations Domain | Maintenance Event |
| Financial Domain | Invoice, Delivery Cost |
| Vendor Domain | Vendor, Vendor Service Contract |

---

## 6. Business Object Model

### 6.1 Standard CRM Objects (Native — No Creation Required)

| Object | Type | Purpose |
|---|---|---|
| Company | Standard | Client organization — hospital system or health network |
| Contact | Standard | Individual stakeholder at client organization |
| Deal | Standard | Commercial opportunity and revenue origin |

### 6.2 Custom Objects (Require Operations Hub Enterprise)

| Object | Type | Purpose |
|---|---|---|
| Service Contract | Custom | Execution agreement tied to won deal |
| Facility | Custom | Physical location under service |
| Asset | Custom | Medical equipment under management |
| Vendor | Custom | Equipment manufacturer or service provider |
| Vendor Service Contract | Custom | Vendor agreement per asset |
| Maintenance Event | Custom | Service or repair event record |

> **License Note:** Custom objects require HubSpot Operations Hub Enterprise. Confirm licensing before object creation.

---

## 7. Association Schema

| From Object | To Object | Cardinality | Label |
|---|---|---|---|
| Company | Contact | One-to-Many | Has Contacts |
| Company | Deal | One-to-Many | Has Deals |
| Company | Facility | One-to-Many | Has Facilities |
| Company | Service Contract | One-to-Many | Has Contracts |
| Contact | Deal | Many-to-Many | Associated With |
| Deal | Service Contract | One-to-One | Generated Contract |
| Service Contract | Facility | One-to-Many | Covers Facilities |
| Facility | Asset | One-to-Many | Contains Assets |
| Asset | Maintenance Event | One-to-Many | Has Service Events |
| Asset | Vendor Service Contract | One-to-Many | Covered By |
| Vendor | Vendor Service Contract | One-to-Many | Provides Service |
| Service Contract | Invoice | One-to-Many | Billed As |
| Service Contract | Delivery Cost | One-to-Many | Incurred Costs |

> **Implementation Note:** All associations must be created via the HubSpot Associations API or CRM schema settings. Association labels must match exactly as defined for automation and reporting to function.

---

## 8. Pipeline Architecture

### 8.1 Sales Pipeline — Object: Deal

| Stage | Name | Probability | Required Fields Before Advancement |
|---|---|---|---|
| 1 | Lead Identified | 5% | lead_source, territory, account_owner |
| 2 | Qualification | 15% | expected_asset_volume, service_type, estimated_contract_value |
| 3 | Discovery and Assessment | 30% | estimated_asset_count, asset_complexity_score, projected_service_cost |
| 4 | Solution Design | 50% | service_scope, staffing_model, vendor_strategy |
| 5 | Proposal Submitted | 65% | contract_term_years, expected_margin, pricing_model |
| 6 | Negotiation | 80% | final_contract_value, negotiated_margin |
| 7 | Contract Pending Signature | 90% | contract_draft_sent_date |
| 8 | Closed Won | 100% | closed_won_date, contract_value — triggers contract creation |
| 9 | Closed Lost | 0% | loss_reason, competitor, closed_lost_date |

### 8.2 Contract Execution Pipeline — Object: Service Contract

| Stage | Name | Trigger |
|---|---|---|
| 1 | Contract Initiated | Closed Won automation |
| 2 | Asset Inventory Phase | Manual advancement |
| 3 | Vendor Alignment Phase | Asset inventory complete |
| 4 | Service Deployment | Vendor alignment complete |
| 5 | Active Operations | Deployment confirmed |
| 6 | Renewal Evaluation | Automation at 90 days before end date |
| 7 | Contract Renewed | Renewal executed |
| 8 | Contract Terminated | Termination notice received |

### 8.3 Service Operations Pipeline — Object: Maintenance Event

| Stage | Name |
|---|---|
| 1 | Service Request Logged |
| 2 | Technician Assigned |
| 3 | Service In Progress |
| 4 | Repair Completed |
| 5 | Validation Completed |
| 6 | Event Closed |

### 8.4 Billing Pipeline — Object: Invoice

| Stage | Name |
|---|---|
| 1 | Invoice Generated |
| 2 | Invoice Delivered |
| 3 | Payment Pending |
| 4 | Payment Received |
| 5 | Overdue |
| 6 | Collections |

---

## 9. Contract-to-Cash Subprocess

### Handoff 1 — Closed Won to Contract Created

| Attribute | Detail |
|---|---|
| Trigger | deal.pipeline_stage = Closed Won |
| Action | Create Service Contract record |
| Data Carried Forward | deal_id, company_id, contract_value, service_scope, contract_term_years |
| Automation Type | HubSpot Workflow |
| Validation Gate | contract_value must be non-null |

### Handoff 2 — Contract Active to Asset Onboarding

| Attribute | Detail |
|---|---|
| Trigger | contract.status = Active |
| Action | Initiate asset inventory workflow |
| Data Carried Forward | contract_id, company_id, facility_id |
| Automation Type | HubSpot Workflow plus CMMS integration |
| Validation Gate | At least one facility must be associated |

### Handoff 3 — Asset Onboarded to Service Schedule

| Attribute | Detail |
|---|---|
| Trigger | asset.lifecycle_stage = Active |
| Action | Create preventive maintenance schedule |
| Data Carried Forward | asset_id, contract_id, vendor_contract_id |
| Automation Type | Workflow plus Maintenance Event creation |
| Validation Gate | asset.serial_number must be populated |

### Handoff 4 — Service Event Completed to Invoice Eligibility

| Attribute | Detail |
|---|---|
| Trigger | maintenance_event.stage = Event Closed |
| Action | Check billing eligibility against contract |
| Data Carried Forward | maintenance_event_id, contract_id, asset_id, repair_cost |
| Automation Type | HubSpot Workflow |
| Validation Gate | contract.status = Active |

### Handoff 5 — Invoice Approved to Revenue Recognized

| Attribute | Detail |
|---|---|
| Trigger | invoice.payment_status = Payment Received |
| Action | Mark revenue as realized; sync to ERP |
| Data Carried Forward | invoice_id, contract_id, deal_id, payment_received_date |
| Automation Type | Workflow plus ERP sync |
| Validation Gate | invoice.amount greater than 0 |

### Handoff 6 — Revenue Recognized to Margin Calculated

| Attribute | Detail |
|---|---|
| Trigger | Invoice closed plus cost records present |
| Formula | margin = SUM(invoice_amount) - SUM(labor_cost + vendor_cost + equipment_cost + allocated_overhead) |
| Grain | Per contract; per deal |
| Refresh Cadence | Daily warehouse job |

---

## 10. Automation Workflow Definitions

| Workflow Name | Trigger | Actions | Objects Affected |
|---|---|---|---|
| Deal Won to Contract | Stage = Closed Won | Create Service Contract; copy deal_id, company_id, contract_value | Deal, Service Contract |
| Contract Active Asset Workflow | Stage = Active Operations | Enroll in asset onboarding sequence | Service Contract, Asset |
| Warranty Expiry Alert | warranty_expiration within 60 days | Create replacement review task | Asset |
| Maintenance Due Alert | next_service_due_date within 14 days | Create maintenance event; assign technician | Asset, Maintenance Event |
| Invoice Overdue Alert | payment_due_date passed and status not Paid | Send overdue notification; update status | Invoice |
| Contract Renewal Alert | contract_end_date within 90 days | Create renewal deal; notify account owner | Service Contract, Deal |
| Asset Replacement Due | replacement_due_date within 90 days | Create capital planning task | Asset |

---

## 11. Pipeline Governance Rules

### Deal Pipeline Rules

| Rule ID | Rule |
|---|---|
| G-D-01 | Deal cannot advance past Qualification without estimated_contract_value |
| G-D-02 | Deal cannot advance past Solution Design without service_scope |
| G-D-03 | Deal cannot reach Closed Won without contract_value and expected_margin |
| G-D-04 | Closed Lost requires loss_reason — mandatory field |
| G-D-05 | Duplicate company detection required at deal creation |

### Contract Pipeline Rules

| Rule ID | Rule |
|---|---|
| G-C-01 | Contract cannot reach Active Operations without at least one associated asset |
| G-C-02 | Contract cannot reach Active Operations without contract_start_date |
| G-C-03 | contract_end_date must be after contract_start_date — system validation |
| G-C-04 | Contract value must match deal value within 5% or require override flag |

### Asset Rules

| Rule ID | Rule |
|---|---|
| G-A-01 | serial_number must be unique across the system |
| G-A-02 | Asset cannot be activated without a parent facility |
| G-A-03 | Asset cannot be activated without a parent contract |
| G-A-04 | replacement_due_date must be after installation_date |

### Invoice Rules

| Rule ID | Rule |
|---|---|
| G-I-01 | Invoice cannot be generated unless contract.status = Active |
| G-I-02 | Invoice amount must be greater than zero |
| G-I-03 | payment_due_date must be after invoice_date |
| G-I-04 | Invoice must carry deal_id from parent contract |

---

## 12. Revenue Lineage Model

### Key Identifier Propagation Chain

```
deal_id
  └── contract_id  (carries deal_id)
        ├── asset_id  (carries contract_id)
        │     └── maintenance_event_id  (carries asset_id, contract_id)
        ├── invoice_id  (carries contract_id, deal_id)
        └── cost_id  (carries contract_id, deal_id)
```

### Margin Calculation Logic

```
Contract Revenue   = SUM(invoice_amount)         WHERE fk_contract_id = X
Contract Cost      = SUM(labor_cost
                       + vendor_cost
                       + equipment_cost
                       + allocated_overhead)      WHERE fk_contract_id = X
Gross Margin       = Contract Revenue - Contract Cost
Margin Percentage  = Gross Margin / Contract Revenue
```

---

## 13. Forecasting Model

| Category | Formula |
|---|---|
| Total Pipeline Revenue | SUM(deal_value) WHERE stage NOT IN (Closed Won, Closed Lost) |
| Weighted Pipeline Revenue | SUM(deal_value x stage_probability) |
| Committed Revenue | SUM(deal_value) WHERE stage IN (Negotiation, Contract Pending Signature) |
| Closed Revenue | SUM(deal_value) WHERE stage = Closed Won AND closed_won_date IN period |
| Realized Revenue | SUM(invoice_amount) WHERE payment_status = Paid AND payment_received_date IN period |
| Contract Backlog | SUM(contract_value - invoiced_to_date) |

---

## 14. Data Governance Framework

### Ownership Model

| Object | Data Owner | Data Steward |
|---|---|---|
| Company | Revenue Operations | CRM Administrator |
| Deal | Sales Leadership | Account Owner |
| Service Contract | Delivery Management | Operations Analyst |
| Asset | Field Operations | CMMS Administrator |
| Invoice | Finance | Billing Analyst |
| Maintenance Event | Service Operations | Field Supervisor |

### Duplicate Detection Rules

| Object | Deduplication Key | Detection Method |
|---|---|---|
| Company | company_name plus zip_code | Fuzzy name match plus exact zip |
| Contact | email | Exact match |
| Asset | serial_number | Exact match — globally unique |
| Deal | company_id plus service_type plus date range | Rule-based within 90-day window |

### Data Retention Policy

| Object | Retention Period | Archival Action |
|---|---|---|
| Deal | 7 years | Archive to cold storage |
| Contract | 10 years | Archive to cold storage |
| Invoice | 10 years | Archive — regulatory requirement |
| Maintenance Event | 7 years | Archive to CMMS |
| Asset | Life of asset plus 5 years | Archive on disposal |

---

## 15. Implementation Checklist Summary

| Phase | Description | Depends On |
|---|---|---|
| Phase 1 | CRM Foundation — Standard object configuration | None |
| Phase 2 | Custom Object Creation — Schema and associations | Phase 1 |
| Phase 3 | Pipeline and Automation Setup | Phase 2 |
| Phase 4 | Data Integration and API Pipelines | Phase 3 |
| Phase 5 | Warehouse Build — Staging, dimensions, facts | Phase 4 |
| Phase 6 | Analytics and Dashboard Enablement | Phase 5 |
| Phase 7 | Governance and QA | Phase 6 |

See implementation_checklist.md for the complete phased checklist with task-level detail.

---

*End of Document — CRM Revenue Lifecycle Architecture Implementation Guide*
*Cross-references: property_catalog.md | api_mapping_document.md | warehouse_architecture.md*
