# Banking Domain Model

## 1. Purpose

This document defines the canonical synthetic banking data model for the Banking Data Platform project.

The model represents a fictional UK retail bank and is designed to simulate realistic operational relationships between customers, accounts, cards, payments, transactions, KYC records, branches, products and reference data.

All data is synthetic. No real bank customer data is used.

---

# 2. Source System Ownership

## Core Banking - SQL Server

Primary system of record for:

- customer
- account
- account_status_history
- branch
- product

## Customer and KYC Platform - REST API

Primary system of record for:

- customer_contact
- kyc_profile
- verification_event
- customer_risk_history

## Card Processor - AWS S3

Primary system of record for:

- card
- card_authorisation
- card_transaction
- merchant

## Payments Platform - Azure Blob Storage

Primary system of record for:

- payment
- payment_status_event
- settlement_batch
- settlement_item

## Controlled Reference Data - SharePoint

Primary source for:

- country
- currency
- transaction_type
- payment_type
- account_status
- card_status
- risk_band
- merchant_category_code

---

# 3. Core Banking Entities

## 3.1 customer

### Business Purpose

Represents the bank's core customer record.

### Grain

One row per customer.

### Source

SQL Server core banking system.

### Primary Key

customer_id

### Example Key

CUST00018492

### Columns

- customer_id: VARCHAR(20), not null
- title: VARCHAR(10)
- first_name: VARCHAR(100), not null
- middle_name: VARCHAR(100)
- last_name: VARCHAR(100), not null
- date_of_birth: DATE, not null
- residency_country_code: CHAR(2), not null
- nationality_country_code: CHAR(2)
- onboarding_date: DATE, not null
- customer_status: VARCHAR(20), not null
- customer_type: VARCHAR(20), not null
- preferred_language: VARCHAR(10)
- created_timestamp: DATETIME2, not null
- modified_timestamp: DATETIME2, not null

### PII Classification

Sensitive:

- first_name
- middle_name
- last_name
- date_of_birth
- nationality
- residency

### Valid Customer Statuses

- ACTIVE
- DORMANT
- RESTRICTED
- CLOSED

### Incremental Strategy

Use modified_timestamp as watermark.

---

## 3.2 account

### Business Purpose

Represents a customer deposit or lending account.

### Grain

One row per account.

### Source

SQL Server core banking system.

### Primary Key

account_id

### Example Key

ACC000928177

### Columns

- account_id: VARCHAR(20), not null
- customer_id: VARCHAR(20), not null
- product_id: VARCHAR(20), not null
- branch_id: VARCHAR(20), not null
- sort_code: CHAR(6), not null
- account_number: CHAR(8), not null
- account_currency: CHAR(3), not null
- account_status: VARCHAR(20), not null
- open_date: DATE, not null
- close_date: DATE
- ledger_balance: DECIMAL(18,2), not null
- available_balance: DECIMAL(18,2), not null
- overdraft_limit: DECIMAL(18,2)
- created_timestamp: DATETIME2, not null
- modified_timestamp: DATETIME2, not null

### Relationships

- many accounts to one customer
- many accounts to one product
- many accounts to one branch

### PII Classification

Sensitive:

- sort_code
- account_number

### Valid Account Statuses

- ACTIVE
- DORMANT
- BLOCKED
- CLOSED

### Incremental Strategy

modified_timestamp watermark.

---

## 3.3 account_status_history

### Business Purpose

Tracks historical status changes for accounts.

### Grain

One row per account status period.

### Primary Key

account_status_history_id

### Columns

- account_status_history_id
- account_id
- account_status
- effective_from
- effective_to
- is_current
- change_reason
- created_timestamp

### Historical Behaviour

Used to demonstrate temporal history and SCD-style processing.

---

## 3.4 branch

### Grain

One row per bank branch.

### Primary Key

branch_id

### Columns

- branch_id
- branch_name
- region
- city
- postcode_area
- active_flag
- opened_date
- closed_date

Target portfolio volume:

Approximately 80-150 branches.

---

## 3.5 product

### Grain

One row per banking product.

### Columns

- product_id
- product_name
- product_type
- currency_code
- interest_rate
- monthly_fee
- overdraft_allowed_flag
- active_flag
- effective_from
- effective_to

### Example Product Types

- CURRENT_ACCOUNT
- SAVINGS_ACCOUNT
- STUDENT_ACCOUNT
- BUSINESS_CURRENT_ACCOUNT

---

# 4. Customer and KYC Platform

## 4.1 customer_contact

### Grain

One current contact record per customer in the API source.

### API Key

party_reference

### Important Design Decision

The REST API does not expose customer_id directly.

Example:

SQL Server:
customer_id = CUST00018492

API:
party_reference = PTY-00018492

A cross-system customer mapping must therefore be created during conformance.

### Columns

- party_reference
- email_address
- mobile_number
- address_line_1
- address_line_2
- city
- postcode
- country_code
- contact_preference
- updated_timestamp

### PII Classification

Highly sensitive.

---

## 4.2 kyc_profile

### Grain

One KYC version per customer risk-assessment period.

### Columns

- kyc_profile_id
- party_reference
- verification_status
- risk_rating
- pep_flag
- sanctions_screening_status
- source_of_funds_category
- verification_date
- effective_from
- effective_to
- is_current
- updated_timestamp

### Valid Verification Statuses

- VERIFIED
- PENDING
- REVIEW_REQUIRED
- EXPIRED

### Valid Risk Ratings

- LOW
- MEDIUM
- HIGH

### Historical Requirement

Meaningful changes must be retained rather than overwritten.

Candidate SCD Type 2 entity.

---

## 4.3 verification_event

### Grain

One row per KYC verification event.

### Columns

- verification_event_id
- party_reference
- verification_type
- verification_status
- event_timestamp
- provider_reference

---

## 4.4 customer_risk_history

### Grain

One row per risk-rating period.

### Columns

- customer_risk_history_id
- party_reference
- risk_rating
- effective_from
- effective_to
- is_current
- change_reason

---

# 5. Card Processing Domain

## 5.1 card

### Grain

One row per issued card.

### Source

AWS S3 card processor.

### Important Key Design

The card system does not expose the core account_id.

Instead:

processor_account_reference = CPR-928177

This must be mapped back to the core banking account.

### Columns

- card_id
- processor_account_reference
- masked_pan
- card_type
- card_status
- issue_date
- expiry_date
- contactless_enabled_flag
- created_timestamp
- modified_timestamp

### Security Rule

Never generate or store:

- full PAN
- CVV
- PIN

Only synthetic masked PAN-like values are permitted.

---

## 5.2 merchant

### Grain

One row per merchant.

### Columns

- merchant_id
- merchant_name
- merchant_category_code
- merchant_country_code
- merchant_city

Target volume:

20,000-50,000 merchants.

---

## 5.3 card_authorisation

### Grain

One row per card authorisation attempt.

### Columns

- authorisation_id
- card_id
- merchant_id
- authorisation_timestamp
- transaction_amount
- transaction_currency
- authorisation_status
- decline_reason
- channel
- processor_reference

### Valid Statuses

- APPROVED
- DECLINED
- REVERSED

---

## 5.4 card_transaction

### Grain

One row per posted card transaction.

### Columns

- card_transaction_id
- card_id
- merchant_id
- processor_account_reference
- authorisation_id
- transaction_timestamp
- posting_date
- transaction_amount
- transaction_currency
- billing_amount
- billing_currency
- debit_credit_indicator
- transaction_status
- card_present_flag
- channel
- processor_reference

### Example Channels

- POS
- ECOMMERCE
- ATM

### Incremental Pattern

File-based daily/hourly incremental load.

---

# 6. Payments Domain

## 6.1 payment

### Grain

One row per payment instruction.

### Columns

- payment_id
- debtor_account_id
- creditor_reference
- payment_type
- amount
- currency_code
- initiated_timestamp
- completed_timestamp
- payment_status
- channel
- payment_reference
- created_timestamp
- modified_timestamp

### Example Payment Types

- FASTER_PAYMENT
- DIRECT_DEBIT
- STANDING_ORDER
- INTERNAL_TRANSFER

### Payment Lifecycle

INITIATED
    ->
PROCESSING
    ->
COMPLETED

Alternative outcomes:

FAILED
REJECTED
CANCELLED
RETURNED

---

## 6.2 payment_status_event

### Grain

One row per payment lifecycle event.

### Columns

- payment_status_event_id
- payment_id
- payment_status
- event_timestamp
- reason_code
- source_system

---

## 6.3 settlement_batch

### Grain

One row per settlement batch.

### Columns

- settlement_batch_id
- settlement_date
- settlement_type
- expected_item_count
- expected_total_amount
- settlement_currency
- batch_status
- received_timestamp

---

## 6.4 settlement_item

### Grain

One row per payment included in a settlement batch.

### Columns

- settlement_batch_id
- payment_id
- settlement_amount
- settlement_currency
- settlement_status

### Reconciliation Requirement

For every completed settlement batch:

sum(settlement_item.settlement_amount)

must reconcile to:

settlement_batch.expected_total_amount

subject to documented exceptions.

---

# 7. Reference Data

Reference datasets are controlled through SharePoint.

## Required Tables

- country
- currency
- transaction_type
- payment_type
- account_status
- card_status
- risk_band
- merchant_category_code

These datasets should be low volume and slowly changing.

---

# 8. Cross-System Key Mapping

The project must not assume that all operational systems use identical keys.

Required mappings include:

customer_id
    <->
party_reference

account_id
    <->
processor_account_reference

Example:

customer_id:
CUST00018492

party_reference:
PTY-00018492

account_id:
ACC000928177

processor_account_reference:
CPR-928177

Silver-layer conformance will create controlled mappings between these identifiers.

---

# 9. Data Quality Defects to Simulate

The generator must deliberately inject a controlled low percentage of realistic defects.

Examples:

- duplicate transaction IDs
- duplicate files
- missing account references
- unknown merchant category codes
- invalid currency codes
- malformed payment statuses
- delayed transaction files
- late KYC updates
- payment records without matching settlement items
- settlement count mismatch
- settlement amount mismatch
- closed-account transactions
- null mandatory fields
- duplicate customer contact records

The percentage should remain low enough that the source data still resembles a functioning banking environment.

---

# 10. Reconciliation Controls

For transaction/payment feeds:

Source row count
=
Accepted row count
+
Rejected or quarantined row count
+
Documented exclusions

For monetary feeds:

Source total amount
=
Accepted total amount
+
Rejected/held total amount
+
Documented exclusions

No unexplained financial difference should be silently accepted.

---

# 11. Synthetic Data Volume

Initial portfolio targets:

- customers: 100,000
- accounts: 150,000
- cards: 120,000
- card transactions: 2,000,000
- payments: 500,000
- KYC profiles: approximately 100,000+
- merchants: 20,000-50,000
- branches: approximately 100
- products: fewer than 50

The transaction dataset may later grow toward 5 million records only for targeted Spark performance tests.

---

# 12. Engineering Constraints

The project must support:

- initial loads
- incremental loads
- CDC-style changes
- watermarking
- idempotent reruns
- late-arriving records
- schema validation
- duplicate handling
- rejected/quarantine processing
- source-to-target reconciliation
- historical tracking
- audit logging
- lineage
- PII governance
- least privilege
- CI/CD
- failure recovery

---

# 13. Data Privacy Principle

All customer names, addresses, identifiers, account information and financial events are fictional.

The project must never import, copy or imitate identifiable real customer data.

Synthetic records should be operationally plausible without representing real individuals.