# CRM Property Catalog — Full Data Dictionary

**Version:** 1.0 | **Classification:** Internal Architecture Reference
**Cross-reference:** crm_implementation_guide.md | api_mapping_document.md

---

## Table of Contents

1. How to Use This Catalog
2. Company Object
3. Contact Object
4. Deal Object
5. Service Contract Object
6. Facility Object
7. Asset Object
8. Vendor Object
9. Vendor Service Contract Object
10. Maintenance Event Object
11. Invoice Object
12. Delivery Cost Object
13. Date Dimension Reference

---

## 1. How to Use This Catalog

Each object section contains:

- **Object metadata** — type, HubSpot classification, license requirement
- **Property table** — all fields with internal name, display name, data type, group, required flag, and governance notes
- **Key summary** — primary key, foreign keys, date keys
- **Dropdown option values** — enumerated values for select fields

### Data Type Reference

| Symbol | Data Type | Description |
|---|---|---|
| UUID | Universally Unique Identifier | HubSpot-generated object ID |
| VARCHAR(n) | Variable character string | Text field with max length n |
| TEXT | Long-form text | Unlimited character text |
| DECIMAL(p,s) | Decimal number | Precision p, scale s |
| INT | Integer | Whole number |
| BOOLEAN | Boolean | True or False |
| DATE | Date | YYYY-MM-DD |
| DATETIME | Date and time | YYYY-MM-DD HH:MM:SS |
| ENUM | Enumerated list | Dropdown or radio select |
| PERCENT | Percentage decimal | Stored as 0.0 to 1.0 |

### HubSpot Field Type Reference

| HubSpot Type | Maps To | Notes |
|---|---|---|
| single_line_text | VARCHAR | Standard text input |
| multi_line_text | TEXT | Long text input |
| number | DECIMAL or INT | Numeric input |
| date | DATE | Date picker |
| datetime | DATETIME | Date and time |
| enumeration | ENUM | Dropdown |
| boolean | BOOLEAN | Checkbox |
| percent | PERCENT | 0–100 input stored as decimal |
| currency | DECIMAL(18,2) | Money field |

---

## 2. Company Object

**HubSpot Classification:** Standard Object
**License Required:** Any HubSpot tier
**Purpose:** Represents a client organization — hospital system, health network, or laboratory

### Property Group: company_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| company_id | Company ID | — | UUID | System | HubSpot auto-generated. Primary key. |
| name | Company Name | single_line_text | VARCHAR(255) | Yes | Legal organization name. Dedup key with zip. |
| parent_health_system | Parent Health System | single_line_text | VARCHAR(255) | No | Parent organization if subsidiary. |
| industry | Industry | enumeration | ENUM | Yes | See dropdown values below. |
| region | Sales Region | enumeration | ENUM | Yes | Assigns account owner territory. |
| phone | Phone | single_line_text | VARCHAR(50) | No | Main switchboard number. |
| website | Website | single_line_text | VARCHAR(255) | No | Organization website URL. |
| city | City | single_line_text | VARCHAR(100) | Yes | Headquarters city. |
| state | State | single_line_text | VARCHAR(50) | Yes | Headquarters state. |
| zip | Zip Code | single_line_text | VARCHAR(20) | Yes | Dedup key with company name. |
| address | Street Address | single_line_text | VARCHAR(255) | No | Headquarters street address. |

### Property Group: company_operational

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| bed_count | Hospital Bed Count | number | INT | No | Total licensed beds. |
| facility_count | Number of Facilities | number | INT | No | Total locations under potential management. |
| client_status | Client Status | enumeration | ENUM | Yes | Controls account routing and reporting. |
| annual_equipment_budget | Annual Equipment Budget | currency | DECIMAL(18,2) | No | Estimated annual spend on equipment. |
| account_owner | Account Owner | single_line_text | VARCHAR(150) | Yes | Assigned sales representative. |
| territory | Territory | enumeration | ENUM | Yes | Geographic sales territory assignment. |

### Property Group: company_dates

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| createdate | Created Date | datetime | DATETIME | System | HubSpot auto-populated on record creation. |
| first_contract_date | First Contract Date | date | DATE | No | Date of first signed agreement. |
| last_activity_date | Last Activity Date | datetime | DATETIME | System | HubSpot auto-updated. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | company_id |
| Dedup Key | name + zip |
| Date Keys | createdate, first_contract_date |

### Dropdown Values

**industry:** Healthcare, Life Sciences, Laboratory, Biotech, Government Health, Other

**client_status:** Prospect, Active, Former, Inactive, On Hold

**region:** Northeast, Southeast, Midwest, Southwest, West, Central

---

## 3. Contact Object

**HubSpot Classification:** Standard Object
**License Required:** Any HubSpot tier
**Purpose:** Individual stakeholders within client organizations

### Property Group: contact_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| contact_id | Contact ID | — | UUID | System | HubSpot auto-generated. Primary key. |
| firstname | First Name | single_line_text | VARCHAR(100) | Yes | Contact first name. |
| lastname | Last Name | single_line_text | VARCHAR(100) | Yes | Contact last name. |
| email | Email | single_line_text | VARCHAR(255) | Yes | Primary dedup key. Must be unique. |
| phone | Phone | single_line_text | VARCHAR(50) | No | Direct phone number. |
| jobtitle | Job Title | single_line_text | VARCHAR(150) | No | Current role title. |
| department | Department | enumeration | ENUM | No | Organizational department. |

### Property Group: contact_role

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| role_type | Role Type | enumeration | ENUM | Yes | Determines communication routing and deal influence tracking. |
| decision_authority | Decision Authority | enumeration | ENUM | No | Level of purchasing authority. |
| hs_lead_status | Lead Status | enumeration | ENUM | No | HubSpot native field. |

### Property Group: contact_dates

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| createdate | Created Date | datetime | DATETIME | System | Auto-populated. |
| last_engagement_date | Last Engagement Date | datetime | DATETIME | No | Last meaningful interaction. |
| hs_email_last_open_date | Last Email Open Date | datetime | DATETIME | System | HubSpot auto-tracked. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | contact_id |
| Foreign Key | company_id (via association) |
| Dedup Key | email |
| Date Keys | createdate, last_engagement_date |

### Dropdown Values

**department:** Biomedical Engineering, Finance, Procurement, Clinical Operations, IT, Administration, Executive

**role_type:** Economic Buyer, Technical Evaluator, End User, Champion, Gatekeeper, Legal

**decision_authority:** Final Decision Maker, Strong Influencer, Recommender, Evaluator, No Authority

---

## 4. Deal Object

**HubSpot Classification:** Standard Object
**License Required:** Any HubSpot tier
**Purpose:** Commercial opportunity. Origin of revenue lineage. deal_id propagates to all downstream objects.

### Property Group: deal_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| deal_id | Deal ID | — | UUID | System | HubSpot auto-generated. Primary key. |
| dealname | Deal Name | single_line_text | VARCHAR(255) | Yes | Descriptive opportunity name. |
| pipeline | Pipeline | enumeration | ENUM | Yes | Must be Sales Pipeline. |
| dealstage | Deal Stage | enumeration | ENUM | Yes | Controls probability and automation. |
| lead_source | Lead Source | enumeration | ENUM | Yes | How opportunity was identified. |
| service_type | Service Type | enumeration | ENUM | Yes | Type of engagement. |
| territory | Territory | enumeration | ENUM | Yes | Geographic assignment. |
| account_owner | Account Owner | single_line_text | VARCHAR(150) | Yes | Assigned representative. |

### Property Group: deal_scope

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| estimated_asset_volume | Estimated Asset Volume | number | INT | Yes — at Qualification | Total assets expected under management. |
| asset_complexity_score | Asset Complexity Score | number | DECIMAL(5,2) | No | Internal scoring of asset mix difficulty. |
| projected_service_cost | Projected Service Cost | currency | DECIMAL(18,2) | No | Estimated cost to deliver. |
| service_scope | Service Scope | multi_line_text | TEXT | Yes — at Solution Design | Full description of delivery scope. |
| staffing_model | Staffing Model | enumeration | ENUM | No | Delivery staffing approach. |
| vendor_strategy | Vendor Strategy | enumeration | ENUM | No | Vendor management approach. |
| contract_term_years | Contract Term (Years) | number | INT | Yes — at Proposal | Duration of agreement. |
| pricing_model | Pricing Model | enumeration | ENUM | Yes — at Proposal | Billing structure type. |

### Property Group: deal_financials

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| amount | Estimated Contract Value | currency | DECIMAL(18,2) | Yes — at Qualification | Total contract value. Maps to HubSpot native amount field. |
| final_contract_value | Final Contract Value | currency | DECIMAL(18,2) | Yes — at Negotiation | Negotiated final value. |
| expected_margin | Expected Margin | percent | PERCENT | Yes — at Negotiation | Estimated gross margin percentage. |
| negotiated_margin | Negotiated Margin | percent | PERCENT | No | Final agreed margin. |
| hs_deal_stage_probability | Stage Probability | percent | PERCENT | System | HubSpot auto-calculated from stage. |

### Property Group: deal_dates

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| createdate | Deal Created Date | datetime | DATETIME | System | Auto-populated. |
| closedate | Expected Close Date | date | DATE | Yes | Forecasted close. Used in pipeline reporting. |
| closed_won_date | Closed Won Date | date | DATE | Yes — on Close | Date deal converted. Trigger for contract creation. |
| closed_lost_date | Closed Lost Date | date | DATE | Conditional | Required if stage = Closed Lost. |
| contract_draft_sent_date | Contract Draft Sent Date | date | DATE | Yes — at Stage 7 | Required before Contract Pending Signature. |
| stage_entry_date | Stage Entry Date | datetime | DATETIME | No | Last stage change timestamp. |

### Property Group: deal_loss

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| loss_reason | Loss Reason | enumeration | ENUM | Conditional | Required if Closed Lost. |
| competitor | Lost To Competitor | single_line_text | VARCHAR(255) | No | Competitor if known. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | deal_id |
| Foreign Key | company_id (via association) |
| Foreign Key | contact_id (via association) |
| Date Keys | createdate, closedate, closed_won_date, closed_lost_date, stage_entry_date |

### Dropdown Values

**service_type:** Full HTM Outsource, Hybrid HTM, Consulting, Capital Planning, Lab Services, Cybersecurity

**lead_source:** Referral, Outbound, Inbound, Event, Partner, RFP, Renewal

**staffing_model:** Fully Outsourced, Co-Managed, Advisory Only

**vendor_strategy:** Vendor Neutral, OEM Direct, Hybrid

**pricing_model:** Managed Service Fixed Fee, Per-Asset Subscription, Time and Materials, Milestone-Based

**loss_reason:** Price, Incumbent Preference, Scope Mismatch, Timing, No Decision, Competitor Won

---

## 5. Service Contract Object

**HubSpot Classification:** Custom Object
**License Required:** Operations Hub Enterprise
**Purpose:** Execution agreement created when deal reaches Closed Won. Carries deal_id as foreign key.

### Property Group: contract_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| contract_id | Contract ID | — | UUID | System | HubSpot auto-generated. Primary key. |
| contract_name | Contract Name | single_line_text | VARCHAR(255) | Yes | Descriptive contract title. |
| contract_status | Contract Status | enumeration | ENUM | Yes | Controls billing eligibility and reporting. |
| service_model | Service Model | enumeration | ENUM | Yes | Delivery approach. |
| sla_type | SLA Type | enumeration | ENUM | Yes | Service level agreement classification. |
| service_scope | Service Scope | multi_line_text | TEXT | Yes | Full description of contracted services. |
| delivery_manager | Delivery Manager | single_line_text | VARCHAR(150) | Yes | Responsible manager name. |
| billing_model | Billing Model | enumeration | ENUM | Yes | How and when invoices are generated. |

### Property Group: contract_financials

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| contract_value | Contract Value | currency | DECIMAL(18,2) | Yes | Total contracted revenue. Must match deal value within 5% or require override. |
| contracted_margin | Contracted Margin | percent | PERCENT | Yes | Agreed margin percentage. |
| invoiced_to_date | Invoiced To Date | currency | DECIMAL(18,2) | No | Running total of invoiced amount. Calculated field. |
| renewal_probability | Renewal Probability | percent | PERCENT | No | Likelihood of contract renewal. |

### Property Group: contract_lineage

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| fk_deal_id | Originating Deal ID | single_line_text | UUID | Yes | Foreign key to Deal. Populated automatically by workflow on Closed Won. |
| fk_company_id | Company ID | single_line_text | UUID | Yes | Foreign key to Company. Populated by workflow. |

### Property Group: contract_dates

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| contract_signed_date | Contract Signed Date | date | DATE | Yes | Execution date of agreement. |
| contract_start_date | Contract Start Date | date | DATE | Yes | Service commencement date. |
| contract_end_date | Contract End Date | date | DATE | Yes | Agreement termination date. Must be after start date. |
| renewal_date | Renewal Date | date | DATE | No | Target renewal execution date. |
| createdate | Record Created Date | datetime | DATETIME | System | Auto-populated on record creation. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | contract_id |
| Foreign Key | fk_deal_id |
| Foreign Key | fk_company_id |
| Date Keys | contract_signed_date, contract_start_date, contract_end_date, renewal_date |

### Dropdown Values

**contract_status:** Initiated, Pending Activation, Active, Suspended, Renewed, Terminated, Expired

**service_model:** Full Outsource, Hybrid Co-Managed, Advisory Only

**sla_type:** Premium 4-Hour Response, Standard 8-Hour Response, Next Business Day, Custom

**billing_model:** Monthly Fixed Fee, Quarterly Fixed Fee, Annual Fixed Fee, Per-Event, Milestone

---

## 6. Facility Object

**HubSpot Classification:** Custom Object
**License Required:** Operations Hub Enterprise
**Purpose:** Physical location belonging to a client organization. Assets are assigned to facilities.

### Property Group: facility_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| facility_id | Facility ID | — | UUID | System | HubSpot auto-generated. Primary key. |
| facility_name | Facility Name | single_line_text | VARCHAR(255) | Yes | Name of the physical location. |
| facility_type | Facility Type | enumeration | ENUM | Yes | Classification of location type. |
| department | Primary Department | single_line_text | VARCHAR(150) | No | Primary service department. |
| address | Street Address | single_line_text | VARCHAR(255) | Yes | Physical address. |
| city | City | single_line_text | VARCHAR(100) | Yes | City. |
| state | State | single_line_text | VARCHAR(50) | Yes | State. |
| zip_code | Zip Code | single_line_text | VARCHAR(20) | Yes | Postal code. |
| fk_company_id | Company ID | single_line_text | UUID | Yes | Foreign key to Company. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | facility_id |
| Foreign Key | fk_company_id |

### Dropdown Values

**facility_type:** Hospital, Outpatient Clinic, Surgery Center, Laboratory, Research Facility, Administrative, Warehouse

---

## 7. Asset Object

**HubSpot Classification:** Custom Object
**License Required:** Operations Hub Enterprise
**Purpose:** Single piece of medical equipment under management. Core operational entity.

### Property Group: asset_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| asset_id | Asset ID | — | UUID | System | HubSpot auto-generated. Primary key. |
| equipment_type | Equipment Type | enumeration | ENUM | Yes | Classification of device. |
| manufacturer | Manufacturer | single_line_text | VARCHAR(150) | Yes | Equipment manufacturer name. |
| model_number | Model Number | single_line_text | VARCHAR(100) | Yes | Manufacturer model designation. |
| serial_number | Serial Number | single_line_text | VARCHAR(150) | Yes | Globally unique. Dedup key. |
| asset_tag | Asset Tag | single_line_text | VARCHAR(100) | No | Internal asset tracking tag. |
| purchase_price | Purchase Price | currency | DECIMAL(18,2) | No | Original acquisition cost. |

### Property Group: asset_lifecycle

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| lifecycle_stage | Lifecycle Stage | enumeration | ENUM | Yes | Current stage of asset lifecycle. |
| utilization_rate | Utilization Rate | percent | PERCENT | No | Percentage of capacity utilized. |
| service_risk_score | Service Risk Score | number | DECIMAL(5,2) | No | Predictive risk score 0–10. |
| condition_rating | Condition Rating | enumeration | ENUM | No | Physical and functional condition. |

### Property Group: asset_lineage

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| fk_facility_id | Facility ID | single_line_text | UUID | Yes | Foreign key to Facility. |
| fk_contract_id | Contract ID | single_line_text | UUID | Yes | Foreign key to Service Contract. |
| fk_company_id | Company ID | single_line_text | UUID | Yes | Foreign key to Company. |

### Property Group: asset_dates

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| installation_date | Installation Date | date | DATE | Yes | Date asset placed in service. |
| warranty_expiration | Warranty Expiration Date | date | DATE | Yes | Triggers alert workflow at 60 days. |
| last_service_date | Last Service Date | date | DATE | No | Date of most recent maintenance event. |
| replacement_due_date | Replacement Due Date | date | DATE | No | Planned replacement date. Triggers capital planning at 90 days. |
| createdate | Record Created Date | datetime | DATETIME | System | Auto-populated. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | asset_id |
| Foreign Key | fk_facility_id |
| Foreign Key | fk_contract_id |
| Foreign Key | fk_company_id |
| Dedup Key | serial_number |
| Date Keys | installation_date, warranty_expiration, last_service_date, replacement_due_date |

### Dropdown Values

**equipment_type:** MRI System, CT Scanner, Patient Monitor, Infusion Pump, Ventilator, Ultrasound, X-Ray System, Lab Analyzer, Defibrillator, Surgical Robot, Endoscopy System, Other

**lifecycle_stage:** Procurement, Active, Maintenance Required, End of Life, Decommissioned, Replacement Pending

**condition_rating:** Excellent, Good, Fair, Poor, Needs Replacement

---

## 8. Vendor Object

**HubSpot Classification:** Custom Object
**License Required:** Operations Hub Enterprise
**Purpose:** Equipment manufacturer or third-party service provider

### Property Group: vendor_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| vendor_id | Vendor ID | — | UUID | System | HubSpot auto-generated. Primary key. |
| vendor_name | Vendor Name | single_line_text | VARCHAR(255) | Yes | Legal vendor entity name. |
| vendor_type | Vendor Type | enumeration | ENUM | Yes | Classification of vendor role. |
| region | Coverage Region | enumeration | ENUM | No | Geographic service coverage. |
| service_capability | Service Capability | multi_line_text | TEXT | No | Description of supported services and equipment types. |
| preferred_vendor | Preferred Vendor | boolean | BOOLEAN | No | Flags preferred vendor status for routing. |
| vendor_contact_name | Primary Contact Name | single_line_text | VARCHAR(150) | No | Vendor relationship contact. |
| vendor_contact_email | Primary Contact Email | single_line_text | VARCHAR(255) | No | Vendor contact email. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | vendor_id |

### Dropdown Values

**vendor_type:** OEM — Original Equipment Manufacturer, Third-Party Service Provider, Specialty Parts Supplier, Independent Service Organization

---

## 9. Vendor Service Contract Object

**HubSpot Classification:** Custom Object
**License Required:** Operations Hub Enterprise
**Purpose:** Service agreement between the organization and a vendor for a specific asset

### Property Group: vendor_contract_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| vendor_contract_id | Vendor Contract ID | — | UUID | System | Primary key. |
| service_frequency | Service Frequency | enumeration | ENUM | Yes | How often preventive maintenance occurs. |
| sla_terms | SLA Terms | multi_line_text | TEXT | Yes | Full SLA description. |
| vendor_service_cost | Vendor Service Cost | currency | DECIMAL(18,2) | Yes | Annual or per-event cost. |
| cost_model | Cost Model | enumeration | ENUM | Yes | How vendor cost is structured. |

### Property Group: vendor_contract_lineage

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| fk_vendor_id | Vendor ID | single_line_text | UUID | Yes | Foreign key to Vendor. |
| fk_asset_id | Asset ID | single_line_text | UUID | Yes | Foreign key to Asset. |
| fk_contract_id | Service Contract ID | single_line_text | UUID | Yes | Foreign key to Service Contract. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | vendor_contract_id |
| Foreign Key | fk_vendor_id |
| Foreign Key | fk_asset_id |
| Foreign Key | fk_contract_id |

### Dropdown Values

**service_frequency:** Monthly, Quarterly, Semi-Annual, Annual, Per-Event, As-Needed

**cost_model:** Annual Fixed, Per-Event, Per-Asset Annual, Time and Materials

---

## 10. Maintenance Event Object

**HubSpot Classification:** Custom Object
**License Required:** Operations Hub Enterprise
**Purpose:** Single service or repair event on an asset. Operational heartbeat of service delivery.

### Property Group: event_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| maintenance_event_id | Maintenance Event ID | — | UUID | System | Primary key. |
| maintenance_type | Maintenance Type | enumeration | ENUM | Yes | Classification of service performed. |
| work_order_number | Work Order Number | single_line_text | VARCHAR(100) | Yes | Unique work order reference. |
| technician_name | Technician Name | single_line_text | VARCHAR(150) | No | Performing technician. |
| event_status | Event Status | enumeration | ENUM | Yes | Current pipeline stage of event. |

### Property Group: event_operational

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| downtime_hours | Downtime Hours | number | DECIMAL(10,2) | No | Total equipment downtime in hours. |
| repair_notes | Repair Notes | multi_line_text | TEXT | No | Description of work performed. |
| parts_used | Parts Used | multi_line_text | TEXT | No | Parts and components replaced. |
| resolution_code | Resolution Code | enumeration | ENUM | No | Outcome classification. |

### Property Group: event_financial

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| repair_cost | Repair Cost | currency | DECIMAL(18,2) | No | Total cost of this service event. |
| billable_flag | Billable | boolean | BOOLEAN | Yes | Whether event generates an invoice. |

### Property Group: event_lineage

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| fk_asset_id | Asset ID | single_line_text | UUID | Yes | Foreign key to Asset. |
| fk_vendor_id | Vendor ID | single_line_text | UUID | No | Foreign key to Vendor if third-party. |
| fk_facility_id | Facility ID | single_line_text | UUID | Yes | Foreign key to Facility. |
| fk_contract_id | Contract ID | single_line_text | UUID | Yes | Foreign key to Service Contract. |

### Property Group: event_dates

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| maintenance_date | Maintenance Date | date | DATE | Yes | Date service was performed. |
| next_service_due_date | Next Service Due Date | date | DATE | No | Triggers maintenance alert at 14 days. |
| createdate | Record Created Date | datetime | DATETIME | System | Auto-populated. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | maintenance_event_id |
| Foreign Key | fk_asset_id |
| Foreign Key | fk_vendor_id |
| Foreign Key | fk_facility_id |
| Foreign Key | fk_contract_id |
| Date Keys | maintenance_date, next_service_due_date |

### Dropdown Values

**maintenance_type:** Preventive Maintenance, Corrective Repair, Emergency Repair, Calibration, Safety Inspection, Software Update, Installation, Decommission

**event_status:** Service Request Logged, Technician Assigned, Service In Progress, Repair Completed, Validation Completed, Event Closed

**resolution_code:** Resolved — Part Replaced, Resolved — Adjustment Made, Resolved — Software Fix, Unresolved — Escalated, Deferred — Parts Pending, Referred to OEM

---

## 11. Invoice Object

**HubSpot Classification:** Custom Object
**License Required:** Operations Hub Enterprise
**Purpose:** Billing record generated against an active contract. Carries deal_id for revenue lineage.

### Property Group: invoice_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| invoice_id | Invoice ID | — | UUID | System | Primary key. |
| invoice_number | Invoice Number | single_line_text | VARCHAR(100) | Yes | Human-readable invoice reference. |
| payment_status | Payment Status | enumeration | ENUM | Yes | Current billing status. |
| billing_type | Billing Type | enumeration | ENUM | Yes | Classification of invoice. |

### Property Group: invoice_financials

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| invoice_amount | Invoice Amount | currency | DECIMAL(18,2) | Yes | Total billed amount. Must be greater than 0. |
| tax_amount | Tax Amount | currency | DECIMAL(18,2) | No | Tax if applicable. |
| total_amount | Total Amount | currency | DECIMAL(18,2) | No | invoice_amount + tax_amount. Calculated. |

### Property Group: invoice_lineage

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| fk_contract_id | Contract ID | single_line_text | UUID | Yes | Foreign key to Service Contract. |
| fk_deal_id | Deal ID | single_line_text | UUID | Yes | Foreign key to originating Deal. Propagated from contract. |
| fk_company_id | Company ID | single_line_text | UUID | Yes | Foreign key to Company. |

### Property Group: invoice_dates

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| invoice_date | Invoice Date | date | DATE | Yes | Date invoice was generated. |
| payment_due_date | Payment Due Date | date | DATE | Yes | Must be after invoice_date. |
| payment_received_date | Payment Received Date | date | DATE | Conditional | Required when payment_status = Payment Received. |
| createdate | Record Created Date | datetime | DATETIME | System | Auto-populated. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | invoice_id |
| Foreign Key | fk_contract_id |
| Foreign Key | fk_deal_id |
| Foreign Key | fk_company_id |
| Date Keys | invoice_date, payment_due_date, payment_received_date |

### Dropdown Values

**payment_status:** Invoice Generated, Invoice Delivered, Payment Pending, Payment Received, Overdue, In Collections, Voided

**billing_type:** Managed Service Fee, Per-Event Charge, Milestone Payment, Reimbursable Expense, Corrective Billing

---

## 12. Delivery Cost Object

**HubSpot Classification:** Custom Object
**License Required:** Operations Hub Enterprise
**Purpose:** Cost record against a contract for margin calculation. Carries deal_id for profitability lineage.

### Property Group: cost_info

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| cost_id | Cost ID | — | UUID | System | Primary key. |
| cost_category | Cost Category | enumeration | ENUM | Yes | Classification of cost. |
| cost_period | Cost Period | single_line_text | VARCHAR(50) | No | Month or quarter this cost covers. |
| cost_description | Cost Description | multi_line_text | TEXT | No | Description of cost. |

### Property Group: cost_breakdown

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| labor_cost | Labor Cost | currency | DECIMAL(18,2) | No | Personnel and labor charges. |
| vendor_cost | Vendor Cost | currency | DECIMAL(18,2) | No | Third-party vendor service charges. |
| equipment_cost | Equipment Cost | currency | DECIMAL(18,2) | No | Parts, components, consumables. |
| allocated_overhead | Allocated Overhead | currency | DECIMAL(18,2) | No | Prorated overhead allocation. |
| total_cost | Total Cost | currency | DECIMAL(18,2) | No | Sum of all cost components. Calculated field. |

### Property Group: cost_lineage

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| fk_contract_id | Contract ID | single_line_text | UUID | Yes | Foreign key to Service Contract. |
| fk_deal_id | Deal ID | single_line_text | UUID | Yes | Foreign key to originating Deal. |

### Property Group: cost_dates

| Internal Name | Display Name | HubSpot Type | Data Type | Required | Governance Notes |
|---|---|---|---|---|---|
| cost_recorded_date | Cost Recorded Date | date | DATE | Yes | Date cost was entered. |
| cost_period_start | Cost Period Start | date | DATE | No | Start of billing period. |
| cost_period_end | Cost Period End | date | DATE | No | End of billing period. |
| createdate | Record Created Date | datetime | DATETIME | System | Auto-populated. |

### Key Summary

| Key Type | Field |
|---|---|
| Primary Key | cost_id |
| Foreign Key | fk_contract_id |
| Foreign Key | fk_deal_id |
| Date Keys | cost_recorded_date, cost_period_start, cost_period_end |

### Dropdown Values

**cost_category:** Labor, Vendor Service, Parts and Equipment, Overhead, Travel, Administrative

---

## 13. Date Dimension Reference

**Purpose:** Analytical layer only — not a CRM object. Used in the data warehouse for time intelligence.

| Column Name | Data Type | Description |
|---|---|---|
| date_key | INT (YYYYMMDD) | Surrogate primary key |
| full_date | DATE | Calendar date |
| day_of_week | INT | 1 = Sunday through 7 = Saturday |
| day_name | VARCHAR(20) | Monday, Tuesday, etc. |
| week_of_year | INT | ISO week number |
| month_number | INT | 1 through 12 |
| month_name | VARCHAR(20) | January, February, etc. |
| quarter | INT | 1 through 4 |
| calendar_year | INT | Four-digit year |
| fiscal_period | INT | Fiscal month if different from calendar |
| fiscal_quarter | INT | Fiscal quarter |
| fiscal_year | INT | Fiscal year |
| is_weekday | BOOLEAN | True if Monday through Friday |
| is_holiday | BOOLEAN | True if recognized holiday |

---

*End of Document — CRM Property Catalog*
*Cross-references: crm_implementation_guide.md | api_mapping_document.md*
