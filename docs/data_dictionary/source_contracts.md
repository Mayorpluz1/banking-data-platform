# Banking Data Platform - Source Contracts

## 1. Purpose

This document defines the physical source contracts for the Banking Data Platform.

It specifies how each simulated operational source exposes data to the ingestion platform, including:

- source-system ownership
- physical objects
- schemas
- keys
- datatypes
- nullability
- incremental-load strategy
- CDC behaviour
- file formats
- arrival patterns
- partitioning conventions
- reconciliation controls
- schema-evolution rules
- PII classification
- ingestion expectations

These contracts represent the boundary between operational source systems and the data platform.

All source data in this project is synthetic.

---

# 2. Source Systems

The platform contains five simulated source systems.

| Source System | Technology | Primary Domain | Ingestion Pattern |
|---|---|---|---|
| CORE_BANKING | SQL Server | Customer, Account, Branch, Product | JDBC / SQL incremental |
| CUSTOMER_KYC | REST API | Contact, KYC, Verification, Risk | Paginated API incremental |
| CARD_PROCESSOR | AWS S3 | Cards, Merchants, Authorisations, Card Transactions | File-based incremental |
| PAYMENTS_PLATFORM | Azure Blob Storage | Payments, Payment Events, Settlement | File-based incremental |
| REFERENCE_DATA | SharePoint | Controlled Reference Data | File snapshot / change detection |

---

# 3. Contract Principles

All source contracts must support the following principles.

## 3.1 Source Ownership

Each business entity has a clearly defined source of record.

Downstream systems must not arbitrarily overwrite source-owned attributes.

---

## 3.2 Source Alignment

Bronze ingestion must preserve source structure as closely as practical.

Business transformation belongs primarily in Silver and Gold.

---

## 3.3 Stable Business Keys

Every entity must expose a stable business identifier.

Examples:

```text
customer_id
account_id
card_id
payment_id
merchant_id

Where systems use different identifiers for the same business concept, mappings must be controlled explicitly.

3.4 Incremental Processing

Incremental ingestion should be preferred over repeated full reloads.

Incremental extraction must use one of:

modified timestamp
event timestamp
source sequence
file arrival
business date
change/version identifier
3.5 Watermark Safety

A source watermark must only advance after the corresponding target batch has been durably committed.

Failed or partial loads must not advance the persisted watermark.

3.6 Idempotency

Reprocessing the same source batch must not create duplicate target business records.

3.7 Schema Enforcement

Incoming data must be validated against the expected contract.

Unexpected structural differences must be detected and handled explicitly.

4. Source System: CORE_BANKING
4.1 Source Overview

System code:

CORE_BANKING

Technology:

SQL Server

Primary domains:

customer
account
account status history
branch
product

Ingestion mechanism:

Azure Data Factory SQL connector

Primary ingestion style:

initial full load
+
incremental watermark extraction
5. CORE_BANKING.customer

Physical source object:

dbo.customer

Grain:

one row per customer

Primary key:

customer_id

Incremental column:

modified_timestamp
Schema
Column	SQL Server Type	Nullable	Key	Classification
customer_id	VARCHAR(20)	No	PK	Internal Identifier
title	VARCHAR(10)	Yes		PII
first_name	VARCHAR(100)	No		PII
middle_name	VARCHAR(100)	Yes		PII
last_name	VARCHAR(100)	No		PII
date_of_birth	DATE	No		Sensitive PII
residency_country_code	CHAR(2)	No	FK/Reference	PII
nationality_country_code	CHAR(2)	Yes	FK/Reference	PII
onboarding_date	DATE	No		Confidential
customer_status	VARCHAR(20)	No	Reference	Confidential
customer_type	VARCHAR(20)	No		Confidential
preferred_language	VARCHAR(10)	Yes		PII
created_timestamp	DATETIME2(3)	No		Technical
modified_timestamp	DATETIME2(3)	No	Watermark	Technical
Incremental Extraction

Predicate pattern:

WHERE modified_timestamp > @previous_watermark
  AND modified_timestamp <= @current_window_end

A deterministic tie-breaker must be used if multiple records share the same timestamp.

Recommended extraction ordering:

ORDER BY modified_timestamp, customer_id
Expected Volume

Initial:

~100,000

Incremental:

small daily percentage of new or changed customers
6. CORE_BANKING.account

Physical source object:

dbo.account

Grain:

one row per account

Primary key:

account_id

Foreign keys:

customer_id
product_id
branch_id

Incremental column:

modified_timestamp
Schema
Column	SQL Server Type	Nullable	Key	Classification
account_id	VARCHAR(20)	No	PK	Sensitive Identifier
customer_id	VARCHAR(20)	No	FK	Sensitive Identifier
product_id	VARCHAR(20)	No	FK	Internal
branch_id	VARCHAR(20)	No	FK	Internal
sort_code	CHAR(6)	No	Business Attribute	Sensitive
account_number	CHAR(8)	No	Business Attribute	Sensitive
account_currency	CHAR(3)	No	Reference	Confidential
account_status	VARCHAR(20)	No	Reference	Confidential
open_date	DATE	No		Confidential
close_date	DATE	Yes		Confidential
ledger_balance	DECIMAL(18,2)	No		Financial
available_balance	DECIMAL(18,2)	No		Financial
overdraft_limit	DECIMAL(18,2)	Yes		Financial
created_timestamp	DATETIME2(3)	No		Technical
modified_timestamp	DATETIME2(3)	No	Watermark	Technical
Business Key Constraint

Synthetic environment must maintain uniqueness for:

sort_code + account_number
Incremental Strategy

Watermark:

modified_timestamp

Ordering:

modified_timestamp, account_id
7. CORE_BANKING.account_status_history

Physical source object:

dbo.account_status_history

Grain:

one row per account status period

Primary key:

account_status_history_id

Foreign key:

account_id
Schema
Column	Type	Nullable	Key
account_status_history_id	BIGINT	No	PK
account_id	VARCHAR(20)	No	FK
account_status	VARCHAR(20)	No	
effective_from	DATETIME2(3)	No	
effective_to	DATETIME2(3)	Yes	
is_current	BIT	No	
change_reason	VARCHAR(100)	Yes	
created_timestamp	DATETIME2(3)	No	

Incremental strategy:

created_timestamp

Validation:

one current row per account
no overlapping valid periods
effective_to > effective_from
8. CORE_BANKING.branch

Physical object:

dbo.branch

Grain:

one row per branch

Primary key:

branch_id
Schema
Column	Type	Nullable
branch_id	VARCHAR(20)	No
branch_name	VARCHAR(150)	No
region	VARCHAR(100)	No
city	VARCHAR(100)	No
postcode_area	VARCHAR(10)	Yes
active_flag	BIT	No
opened_date	DATE	No
closed_date	DATE	Yes
modified_timestamp	DATETIME2(3)	No

Expected volume:

~100 rows

Load pattern:

initial full load
+
small incremental changes
9. CORE_BANKING.product

Physical object:

dbo.product

Grain:

one row per product version

Primary key:

product_id + effective_from
Schema
Column	Type	Nullable
product_id	VARCHAR(20)	No
product_name	VARCHAR(150)	No
product_type	VARCHAR(50)	No
currency_code	CHAR(3)	No
interest_rate	DECIMAL(9,6)	Yes
monthly_fee	DECIMAL(10,2)	Yes
overdraft_allowed_flag	BIT	No
active_flag	BIT	No
effective_from	DATETIME2(3)	No
effective_to	DATETIME2(3)	Yes
modified_timestamp	DATETIME2(3)	No

Expected volume:

<50 active products

Historical versions may increase total row count.

10. CORE_BANKING Extraction Control

Each extraction must record:

pipeline_run_id
entity_name
previous_watermark
window_end
rows_extracted
extraction_start_timestamp
extraction_end_timestamp
status

Watermark must advance only after successful target completion.

11. Source System: CUSTOMER_KYC

System code:

CUSTOMER_KYC

Technology:

REST API

Primary domains:

customer contact
KYC
verification events
customer risk history

Transport:

HTTPS

Authentication simulation:

API key during local simulation

Production target pattern:

secret stored outside source code
managed through secure secret management
12. CUSTOMER_KYC API Standards

Base conceptual route:

/api/v1/

Endpoints:

/api/v1/customer-contacts
/api/v1/kyc-profiles
/api/v1/verification-events
/api/v1/customer-risk-history

Responses:

JSON

Encoding:

UTF-8
13. API Pagination

Default pagination model:

page
page_size

Example:

?page=1&page_size=500

Response metadata should include:

{
  "page": 1,
  "page_size": 500,
  "total_records": 100000,
  "has_more": true
}

Maximum page size should be configurable.

14. API Incremental Filtering

Endpoints supporting incremental extraction should accept:

updated_since
updated_until

Conceptual request:

/api/v1/customer-contacts?updated_since=2026-09-01T00:00:00Z&updated_until=2026-09-01T23:59:59Z

Ordering should be deterministic.

Recommended ordering:

updated_timestamp
+
business identifier
15. CUSTOMER_KYC.customer_contact

Endpoint:

/api/v1/customer-contacts

Grain:

one current contact record per party

Primary identifier:

party_reference
Schema
Field	JSON Type	Nullable	Classification
party_reference	string	No	Sensitive Identifier
email_address	string	Yes	PII
mobile_number	string	Yes	PII
address_line_1	string	Yes	PII
address_line_2	string	Yes	PII
city	string	Yes	PII
postcode	string	Yes	PII
country_code	string	No	PII
contact_preference	string	Yes	PII
updated_timestamp	ISO-8601 timestamp	No	Technical

Watermark:

updated_timestamp
16. CUSTOMER_KYC.kyc_profile

Endpoint:

/api/v1/kyc-profiles

Grain:

one row per KYC profile version

Primary key:

kyc_profile_id
Schema
Field	Type	Nullable	Classification
kyc_profile_id	string	No	Internal
party_reference	string	No	Sensitive Identifier
verification_status	string	No	Sensitive
risk_rating	string	No	Sensitive
pep_flag	boolean	No	Highly Sensitive
sanctions_screening_status	string	No	Highly Sensitive
source_of_funds_category	string	Yes	Highly Sensitive
verification_date	date	Yes	Sensitive
effective_from	timestamp	No	Technical
effective_to	timestamp	Yes	Technical
is_current	boolean	No	Technical
updated_timestamp	timestamp	No	Watermark

Incremental column:

updated_timestamp

Historical records must not be overwritten.

17. CUSTOMER_KYC.verification_event

Endpoint:

/api/v1/verification-events

Grain:

one row per verification event

Primary key:

verification_event_id
Fields
verification_event_id
party_reference
verification_type
verification_status
event_timestamp
provider_reference

Incremental strategy:

event_timestamp + verification_event_id

This is append-oriented event data.

18. CUSTOMER_KYC.customer_risk_history

Endpoint:

/api/v1/customer-risk-history

Grain:

one row per risk-rating period

Fields:

customer_risk_history_id
party_reference
risk_rating
effective_from
effective_to
is_current
change_reason
updated_timestamp

Watermark:

updated_timestamp
19. API Operational Failure Scenarios

The simulator may later support controlled responses such as:

400 Bad Request
401 Unauthorized
429 Too Many Requests
500 Internal Server Error
503 Service Unavailable

These should be used to demonstrate:

retry handling
authentication failure handling
transient failure recovery
rate-limit awareness

Retries must not be applied blindly to permanent errors.

20. Source System: CARD_PROCESSOR

System code:

CARD_PROCESSOR

Technology:

AWS S3

Primary domain:

cards
merchants
authorisations
posted card transactions

Ingestion pattern:

file based

Primary format:

Parquet for high-volume event datasets
CSV permitted for low-volume master data
21. S3 Logical Layout

Conceptual bucket:

banking-card-processor-source

Logical paths:

cards/
merchants/
authorisations/
card_transactions/

Business-date partitioning:

card_transactions/business_date=YYYY-MM-DD/

Example:

card_transactions/business_date=2026-09-01/
card_transactions_20260901_BATCH001.parquet
22. CARD_PROCESSOR.card

Path:

cards/

Preferred format:

Parquet

Grain:

one row per card

Fields:

card_id
processor_account_reference
masked_pan
card_type
card_status
issue_date
expiry_date
contactless_enabled_flag
created_timestamp
modified_timestamp

Primary key:

card_id

Watermark:

modified_timestamp

Security:

full PAN prohibited
CVV prohibited
PIN prohibited
23. CARD_PROCESSOR.merchant

Path:

merchants/

Preferred format:

Parquet

Fields:

merchant_id
merchant_name
merchant_category_code
merchant_country_code
merchant_city
modified_timestamp

Primary key:

merchant_id

Expected initial volume:

~30,000
24. CARD_PROCESSOR.card_authorisation

Path convention:

authorisations/business_date=YYYY-MM-DD/

Format:

Parquet

Grain:

one row per authorisation attempt
Fields
authorisation_id
card_id
merchant_id
authorisation_timestamp
transaction_amount
transaction_currency
authorisation_status
decline_reason
channel
processor_reference
business_date

Primary key:

authorisation_id

Incremental strategy:

new business-date files
25. CARD_PROCESSOR.card_transaction

Path:

card_transactions/business_date=YYYY-MM-DD/

Format:

Parquet

Grain:

one row per posted card transaction
Schema
Column	Type	Nullable
card_transaction_id	string	No
card_id	string	No
merchant_id	string	Yes
processor_account_reference	string	No
authorisation_id	string	Yes
transaction_timestamp	timestamp	No
posting_date	date	No
transaction_amount	decimal(18,2)	No
transaction_currency	string	No
billing_amount	decimal(18,2)	No
billing_currency	string	No
debit_credit_indicator	string	No
transaction_status	string	No
card_present_flag	boolean	Yes
channel	string	No
processor_reference	string	No
business_date	date	No

Primary key:

card_transaction_id
26. Card Transaction File Naming

Pattern:

card_transactions_<business_date>_<batch_id>.parquet

Example:

card_transactions_20260901_BATCH001.parquet

Duplicate source files may deliberately be generated for failure testing.

The pipeline must identify file-level duplicates.

27. Card Processor Arrival Pattern

Normal simulated arrival:

multiple batches per business day

Portfolio implementation may generate:

1-4 files per day

to demonstrate multi-file ingestion without unnecessary scale.

Late-arriving files must be supported.

28. Card Processor File Control Metadata

Each batch should have associated control metadata containing:

batch_id
business_date
source_file_name
expected_record_count
expected_total_amount
generated_timestamp

Control totals must reconcile with source file content.

29. Source System: PAYMENTS_PLATFORM

System code:

PAYMENTS_PLATFORM

Technology:

Azure Blob Storage

Primary domains:

payment
payment status events
settlement batch
settlement items

Ingestion:

file based

Formats:

JSON for payment events
Parquet or CSV for payment and settlement batches
30. Azure Blob Logical Layout

Conceptual container:

banking-payments-source

Paths:

payments/
payment_status_events/
settlement_batches/
settlement_items/
control/

Business-date structure:

payments/business_date=YYYY-MM-DD/
31. PAYMENTS_PLATFORM.payment

Grain:

one row per payment instruction

Preferred format:

Parquet

Fields:

payment_id
debtor_account_id
creditor_reference
payment_type
amount
currency_code
initiated_timestamp
completed_timestamp
payment_status
channel
payment_reference
created_timestamp
modified_timestamp
business_date

Primary key:

payment_id

Incremental pattern:

new and changed business-date files
32. PAYMENTS_PLATFORM.payment_status_event

Preferred format:

JSON Lines

File extension:

.jsonl

Grain:

one row per payment lifecycle event

Fields:

payment_status_event_id
payment_id
payment_status
event_timestamp
reason_code
source_system
business_date

Primary key:

payment_status_event_id

Event data is append-oriented.

33. PAYMENTS_PLATFORM.settlement_batch

Preferred format:

CSV

Grain:

one row per settlement batch

Fields:

settlement_batch_id
settlement_date
settlement_type
expected_item_count
expected_total_amount
settlement_currency
batch_status
received_timestamp

Primary key:

settlement_batch_id
34. PAYMENTS_PLATFORM.settlement_item

Preferred format:

Parquet

Grain:

one row per payment settlement membership

Fields:

settlement_batch_id
payment_id
settlement_amount
settlement_currency
settlement_status

Logical uniqueness:

settlement_batch_id + payment_id
35. Settlement Reconciliation

For each valid completed settlement batch:

COUNT(settlement_item)
=
expected_item_count

and:

SUM(settlement_amount)
=
expected_total_amount

A mismatch must not be silently ignored.

Possible result statuses:

PASS
FAIL_COUNT
FAIL_AMOUNT
FAIL_COUNT_AND_AMOUNT
36. Payment File Naming

Examples:

payments_20260901_BATCH001.parquet
payment_events_20260901_BATCH001.jsonl
settlement_batches_20260901.csv
settlement_items_20260901_BATCH001.parquet
37. Payment Arrival Pattern

Payments:

multiple incremental batches per business day

Settlement:

normally business-date batch oriented

Portfolio scale may simplify this while retaining the distinction between transaction activity and settlement processing.

38. Source System: REFERENCE_DATA

System code:

REFERENCE_DATA

Technology:

SharePoint

Primary domain:

business-maintained controlled reference data

Files should be small and readable by non-technical users.

Preferred format:

Excel or CSV
39. Reference Data Files

Initial controlled datasets:

country
currency
transaction_type
payment_type
account_status
card_status
risk_band
merchant_category_code
40. Standard Reference Schema

Where practical, reference datasets should use a common structure.

Example:

Column	Type	Purpose
code	string	Stable code
description	string	Business description
active_flag	boolean	Current validity
effective_from	date	Start date
effective_to	date	End date
modified_timestamp	timestamp	Change detection

Some datasets may require additional attributes.

41. REFERENCE_DATA.currency

Example schema:

currency_code
currency_name
minor_unit
active_flag
effective_from
effective_to
modified_timestamp

Primary key:

currency_code + effective_from
42. REFERENCE_DATA.country

Fields:

country_code
country_name
region
active_flag
effective_from
effective_to
modified_timestamp
43. REFERENCE_DATA.payment_type

Fields:

payment_type_code
payment_type_description
active_flag
effective_from
effective_to
modified_timestamp

Initial values may include:

FASTER_PAYMENT
DIRECT_DEBIT
STANDING_ORDER
INTERNAL_TRANSFER
44. REFERENCE_DATA.risk_band

Fields:

risk_band_code
risk_band_description
sort_order
active_flag
effective_from
effective_to
modified_timestamp

Initial codes:

LOW
MEDIUM
HIGH
45. REFERENCE_DATA.merchant_category_code

Fields:

merchant_category_code
merchant_category_description
merchant_group
active_flag
modified_timestamp

Merchant generation must reference this dataset.

46. SharePoint Change Detection

Reference files are low volume.

Initial implementation may use:

full snapshot ingestion
+
record hash comparison

rather than unnecessarily complex CDC.

The downstream platform must identify:

new records
changed records
deactivated records
47. File Arrival and SLA Metadata

Every file-based source should have an expected arrival pattern recorded in ingestion metadata.

Suggested metadata fields:

source_system_code
entity_code
expected_frequency
expected_arrival_time
late_after_minutes
file_pattern
active_flag

This will later support missing-file and late-file monitoring.

48. Source Contract Metadata

The ingestion control framework should eventually maintain contract metadata such as:

source_system_code
entity_code
source_object
source_type
load_type
watermark_column
tie_breaker_column
primary_key
file_format
source_path
target_bronze_table
schema_version
active_flag
load_sequence

This configuration will drive metadata-based ingestion.

49. Incremental Strategy Matrix
Entity	Source	Incremental Mechanism
customer	SQL Server	modified_timestamp
account	SQL Server	modified_timestamp
account_status_history	SQL Server	created_timestamp
branch	SQL Server	modified_timestamp
product	SQL Server	modified_timestamp
customer_contact	REST API	updated_timestamp
kyc_profile	REST API	updated_timestamp
verification_event	REST API	event_timestamp
customer_risk_history	REST API	updated_timestamp
card	AWS S3	file + modified_timestamp
merchant	AWS S3	file + modified_timestamp
card_authorisation	AWS S3	new files/business date
card_transaction	AWS S3	new files/business date
payment	Azure Blob	new/changed files
payment_status_event	Azure Blob	append-only event files
settlement_batch	Azure Blob	business-date files
settlement_item	Azure Blob	business-date files
reference data	SharePoint	snapshot + hash comparison
50. CDC Strategy

CDC behaviour should differ by source.

SQL Server:

watermark-based incremental extraction initially

Later project demonstration:

CDC-style operation generation may be introduced

REST API:

updated_timestamp filtering

Event endpoints:

append-oriented incremental events

File sources:

new file discovery
+
business date
+
file/batch identifier

SharePoint:

snapshot comparison

The project must not force one CDC technique onto every source.

51. Source Sequence and Tie-Breakers

Watermarks based only on timestamp may be ambiguous.

Where necessary, deterministic extraction must use:

watermark
+
tie-breaker business key or sequence

Example:

modified_timestamp
+
customer_id

or:

source_change_timestamp
+
source_sequence
52. File Idempotency

File-based ingestion must capture sufficient metadata to identify previously processed files.

At minimum:

source_system
source_file_name
file_size
business_date
batch_id

A content/file hash may also be calculated.

Reprocessing a previously completed file must not create duplicate business rows.

53. File Manifest

Each generated file should eventually register metadata such as:

run_id
batch_id
source_system
entity_name
file_name
file_path
business_date
record_count
total_amount
file_size
generated_timestamp
54. Schema Versioning

Every source contract should eventually have a logical schema version.

Example:

v1

Schema changes should be classified as:

BACKWARD_COMPATIBLE
BREAKING

Examples of compatible change:

new optional field

Potential breaking changes:

renamed mandatory column
changed datatype
removed required field
business key changed
55. Schema Drift Handling

Unexpected columns must not automatically become trusted business attributes.

Bronze may preserve additional source information where appropriate.

Silver must validate against the expected business contract.

Breaking changes should:

fail or quarantine affected processing
+
raise an operational alert

rather than silently alter downstream models.

56. Nullability Enforcement

Mandatory source attributes must be validated.

Examples:

customer.customer_id
account.account_id
card.card_id
payment.payment_id

must never be null in valid source data.

Controlled defect injection may violate these rules intentionally.

Such defects must appear in the defect manifest.

57. Referential Integrity

Valid source relationships include:

account.customer_id -> customer.customer_id

account.product_id -> product.product_id

account.branch_id -> branch.branch_id

account_status_history.account_id -> account.account_id

Cross-system relationships must be validated through mapping/conformance rather than physical foreign keys.

58. Cross-System Conformance

Example:

customer_id
CUST00018492

maps to:

party_reference
PTY-00018492

Example:

account_id
ACC000928177

maps to:

processor_account_reference
CPR-928177

Mapping must be deterministic and controlled.

Downstream conformance should not depend solely on parsing identifier text.

59. Data Classification Levels

The project will use the following logical classification categories:

PUBLIC
INTERNAL
CONFIDENTIAL
PII
SENSITIVE_PII
FINANCIAL
HIGHLY_SENSITIVE

Examples:

first_name -> PII
date_of_birth -> SENSITIVE_PII
account_number -> FINANCIAL
ledger_balance -> FINANCIAL
pep_flag -> HIGHLY_SENSITIVE
60. Card Data Security

The following data must never be generated:

full PAN
CVV
PIN
track data

Only masked synthetic representations are permitted.

Example:

************4837
61. Ingestion Metadata Added by Platform

The source contract must remain distinguishable from platform-generated technical metadata.

Bronze ingestion may add:

_ingestion_timestamp
_source_system
_source_object
_source_file
_business_date
_batch_id
_pipeline_run_id
_record_hash
_schema_version

These columns are not operational-source business attributes.

62. Bronze Naming Convention

Proposed table naming:

bronze.<source_system>_<entity>

Examples:

bronze.core_banking_customer
bronze.core_banking_account
bronze.customer_kyc_kyc_profile
bronze.card_processor_card_transaction
bronze.payments_platform_payment

Final catalogue/schema naming may later be aligned with Unity Catalog implementation.

63. Bronze Preservation Rule

Bronze must preserve:

source values
+
source grain
+
technical ingestion metadata

Bronze must not silently:

repair business values
infer missing foreign keys
collapse duplicates
overwrite invalid values
discard rejected source records without audit

Cleansing belongs in Silver.

64. Data Quality Contract

Every source entity should eventually define:

primary key rule
mandatory-field rules
domain-value rules
referential rules
temporal rules
freshness rule
duplicate rule

Financial entities additionally require:

record-count reconciliation
amount reconciliation
65. Reconciliation Contract

For record processing:

source_count
=
accepted_count
+
rejected_count
+
documented_exclusion_count

For monetary processing:

source_amount
=
accepted_amount
+
rejected_or_held_amount
+
documented_exclusion_amount

Any unexplained variance must be visible operationally.

66. Audit Contract

Every ingestion execution should produce an audit record containing:

pipeline_run_id
entity_name
source_system
load_type
window_start
window_end
start_timestamp
end_timestamp
source_row_count
target_row_count
rejected_row_count
status
error_message

File sources should additionally capture:

source_file
batch_id
business_date
67. Failure Behaviour

An entity load must not be marked successful if:

schema validation fails
critical source extraction fails
target write fails
critical reconciliation fails
watermark persistence fails

Partial failures must remain observable.

68. Retry Behaviour

Transient failures may be retried.

Examples:

temporary API 500/503
temporary network failure
temporary storage access problem

Permanent failures should not receive endless retries.

Examples:

authentication configuration error
invalid schema
business-rule violation
unsupported contract version
69. Source Contract Validation

Before any generator is considered complete, its output must be validated against this contract.

Validation must check:

column presence
datatype compatibility
nullability
primary key uniqueness
foreign key relationships
domain values
temporal logic
expected volume
expected defects
unexpected defects
70. Development Volume Principle

Production-style contracts must not require production-scale infrastructure.

The architecture should remain:

production-style design
+
portfolio-scale execution

The contract should work correctly whether the dataset contains:

1,000 rows

or:

millions of rows

without changing its business meaning.

71. Initial Delivery Scope

The first implementation will focus on:

CORE_BANKING
CUSTOMER_KYC
CARD_PROCESSOR
PAYMENTS_PLATFORM
REFERENCE_DATA

No additional source platform should be introduced unless it solves a defined engineering requirement.

72. Contract Change Governance

Changes to this document that affect:

business keys
source ownership
schema
incremental strategy
file format
reconciliation logic
PII handling

must be treated as architectural/data-contract changes.

Significant changes should be reflected through:

Git history
+
appropriate ADR where required
73. Production Engineering Principle

The ingestion platform must be driven by explicit source contracts rather than assumptions embedded inside pipeline code.

The desired pattern is:

SOURCE CONTRACT
      |
      v
INGESTION CONFIGURATION
      |
      v
METADATA-DRIVEN ORCHESTRATION
      |
      v
BRONZE
      |
      v
VALIDATION / RECONCILIATION
      |
      v
SILVER

This keeps source-specific rules controlled, auditable and maintainable.

74. Definition of Done

This source-contract design is considered complete when:

every initial source entity has an owner
physical source representation is defined
grain is defined
keys are defined
datatypes are defined
nullability expectations are defined
incremental strategy is defined
file/API conventions are defined
PII classification is defined
reconciliation requirements are defined
schema-change behaviour is defined
ingestion metadata expectations are defined
no ambiguous source ownership remains

The implementation must follow these contracts unless an approved design change updates them.