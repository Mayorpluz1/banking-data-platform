# Synthetic Banking Data Generation Rules

## 1. Purpose

This document defines the generation rules for all synthetic banking data used in the Banking Data Platform.

The objective is to generate data that is:

- fully synthetic
- relationally consistent
- operationally plausible for a fictional UK retail bank
- suitable for data engineering workloads
- suitable for initial and incremental loading
- suitable for CDC simulation
- suitable for reconciliation
- suitable for data quality testing
- suitable for SCD Type 2 processing
- suitable for Spark performance testing
- deterministic and reproducible

The generated data must not represent real customers, accounts, cards, merchants or financial transactions.

---

# 2. Global Generation Principles

## 2.1 Synthetic Data Only

All generated records must be fictional.

The generator must never:

- copy real customer records
- use real bank account details
- generate usable card credentials
- copy real transaction histories
- represent identifiable real individuals

Synthetic records should be operationally plausible without representing real people.

---

## 2.2 Deterministic Generation

All generators must support a configurable random seed.

Example:

```python
RANDOM_SEED = 42

The same:

random seed
configuration
run identifier
generation window

must reproduce the same intended dataset.

This supports:

repeatable testing
debugging
reconciliation
automated validation
CI/CD testing
failure recovery testing

Random behaviour must not be scattered unpredictably throughout the codebase.

2.3 Configuration-Driven Generation

Business distributions and volumes must be controlled through configuration rather than duplicated as hard-coded values throughout Python modules.

Configuration should control:

record volumes
date ranges
random seed
customer distributions
account distributions
transaction distributions
payment distributions
defect rates
incremental window
output locations
performance-test scale

This allows development, testing and performance workloads to use different volumes without changing generator code.

3. Generation Modes

The generator must eventually support three execution modes.

3.1 Initial Load

Creates the historical baseline.

Example:

2023-01-01 to 2026-08-31

This produces the starting state of:

customers
accounts
account history
products
branches
KYC
cards
merchants
authorisations
transactions
payments
payment events
settlement data
reference data
3.2 Incremental Load

Creates new business activity after the initial baseline.

Examples:

new customers
changed customer details
new accounts
account status changes
new cards
card status changes
new transactions
new payments
KYC updates
settlement batches
reference-data changes
3.3 Controlled Performance Load

Creates larger datasets only for specific engineering experiments.

Examples:

Spark shuffle analysis
partition testing
join optimisation
data skew testing
Delta MERGE testing
file-size optimisation

Large datasets must not be the default generation mode.

4. Global Temporal Rules

Initial historical generation period:

2023-01-01 to 2026-08-31

Dates and timestamps must follow valid business chronology.

Examples:

customer_onboarding_date <= account_open_date

account_open_date <= card_issue_date

card_issue_date <= card_transaction_timestamp

payment_initiated_timestamp <= payment_completed_timestamp

effective_from < effective_to

account_open_date < account_close_date

Historical effective-date ranges for the same business entity must not overlap unless the record has been deliberately injected as a known data-quality defect.

Future-dated records must not appear accidentally.

5. Customer Generation
5.1 Target Volume

Initial target:

100,000 customers
5.2 Customer Identifier

Format:

CUST00000001

Example:

CUST00018492

customer_id must be:

unique
stable
non-null
deterministic
5.3 Customer Type Distribution

Suggested synthetic distribution:

PERSONAL     94%
BUSINESS      6%

These are project simulation assumptions and do not represent any specific bank's actual portfolio.

5.4 Customer Status Distribution

Suggested:

ACTIVE        91%
DORMANT        4%
RESTRICTED     2%
CLOSED         3%

The distribution must be configurable.

5.5 Customer Age Distribution

Personal customers must be adults.

Suggested range:

18-90 years

Age must not be uniformly distributed.

Suggested synthetic distribution:

18-24     10%
25-34     22%
35-44     22%
45-54     18%
55-64     14%
65+       14%

Date of birth must correspond correctly with generated age.

6. Customer Geography

The primary population should represent fictional UK retail-banking customers across:

England
Scotland
Wales
Northern Ireland

A small minority may have non-UK residency to support international-customer and KYC scenarios.

Generated addresses must be fictional.

Postcodes may follow plausible synthetic formatting but must not intentionally reproduce a real customer's complete address.

7. Customer Onboarding Behaviour

Customers must have different onboarding dates across the historical period.

The generator should avoid creating the same number of customers every day.

Synthetic onboarding should contain reasonable variation such as:

weekday differences
monthly variation
modest seasonal variation

Closed customers should generally have earlier onboarding dates than newly active customers.

8. Account Generation
8.1 Target Volume

Initial target:

150,000 accounts

This gives approximately:

1.5 accounts per customer

on average.

The relationship must not be uniform.

Suggested distribution:

1 account       ~60%
2 accounts      ~30%
3 accounts       ~8%
4+ accounts      ~2%

Not every customer should have the same number of accounts.

9. Account Identifiers

account_id format:

ACC000000001

Example:

ACC000928177

Synthetic account number:

8 numeric characters

Synthetic sort code:

6 numeric characters

The combination:

sort_code + account_number

must be unique within the synthetic environment.

These values are fictional and must not intentionally reproduce known real account details.

10. Account Product Distribution

Suggested synthetic portfolio:

CURRENT_ACCOUNT              58%
SAVINGS_ACCOUNT              30%
STUDENT_ACCOUNT               5%
BUSINESS_CURRENT_ACCOUNT      4%
OTHER_DEPOSIT_PRODUCT         3%

Product assignment must be logically compatible with customer type.

For example:

BUSINESS_CURRENT_ACCOUNT

should normally belong to a business customer rather than a personal customer.

11. Account Currency

Suggested distribution:

GBP                 96.0%
EUR                  2.0%
USD                  1.5%
OTHER_SUPPORTED      0.5%

GBP should dominate because the platform represents a fictional UK retail bank.

Currency values must exist in the controlled currency reference dataset.

12. Account Status

Suggested distribution:

ACTIVE       92%
DORMANT       3%
BLOCKED       2%
CLOSED        3%

Account status must be compatible with account history.

For example, if:

account.account_status = CLOSED

the latest current account-status-history record must normally also indicate:

CLOSED

unless a deliberate defect has been injected.

13. Account Status History

Some accounts must experience historical status changes.

Example:

ACTIVE
   ->
DORMANT
   ->
ACTIVE

or:

ACTIVE
   ->
BLOCKED
   ->
ACTIVE

or:

ACTIVE
   ->
CLOSED

Each history record must contain:

effective_from
effective_to
is_current
change_reason

For each account:

only one record may have is_current = true

under normal valid conditions.

Effective periods must not overlap.

14. Account Balance Rules

Balances must follow skewed distributions rather than uniform random values.

Current Accounts

Typical synthetic pattern:

many balances below £5,000
smaller population between £5,000 and £25,000
relatively few high balances
some negative balances where overdrafts are permitted
Savings Accounts

Typical synthetic pattern:

normally positive
broader positive balance distribution
occasional larger balances

The project does not attempt to reproduce any real bank's proprietary balance-calculation engine.

15. Ledger and Available Balance Relationship

The following fields must not be generated independently:

ledger_balance
available_balance
overdraft_limit

They must follow internally consistent logic.

Conceptually:

available funds
=
ledger position
+
available overdraft capacity
-
relevant holds

The exact synthetic implementation may be simplified, but impossible combinations should not be produced accidentally.

16. Branch Generation

Initial target:

approximately 80-150 branches

The first implementation may use approximately:

100 branches

Each branch should contain:

branch_id
branch_name
region
city
postcode_area
active_flag
opened_date
closed_date

Closed branches must have a valid closed date after their opened date.

Branch identifiers must remain stable.

17. Product Generation

Initial target:

fewer than 50 products

Products should include combinations such as:

current account
savings account
student account
business current account

Product records should support:

effective dating
active/inactive status
interest rate
monthly fee
overdraft eligibility

Product changes may later provide a slowly-changing reference-data scenario.

18. Cross-System Customer Mapping

Different source systems must intentionally use different identifiers.

Core Banking example:

customer_id = CUST00018492

Customer/KYC API example:

party_reference = PTY-00018492

The generator must produce a deterministic cross-system mapping.

Example:

CUST00018492 <-> PTY-00018492

The analytical platform must not rely on stripping prefixes as its production conformance strategy.

A controlled mapping dataset will be used to resolve identities.

This supports realistic testing of:

cross-system conformance
identifier mapping
reconciliation
referential integrity
master-data issues
19. Customer Contact Generation

Customer-contact data belongs to the REST API source.

Typical fields:

party_reference
email_address
mobile_number
address_line_1
address_line_2
city
postcode
country_code
contact_preference
updated_timestamp

All contact information must be synthetic.

Contact records should support controlled changes such as:

address change
telephone-number change
email change
communication-preference change

These changes will support incremental ingestion testing.

20. KYC Generation

Active customers should normally have an associated KYC profile.

Suggested synthetic risk distribution:

LOW        76%
MEDIUM     21%
HIGH        3%

These percentages are project assumptions and are not intended to represent an actual financial institution.

21. KYC Verification Status

Supported values:

VERIFIED
PENDING
REVIEW_REQUIRED
EXPIRED

Most active customers should be:

VERIFIED

Restricted customers should have an increased probability of:

REVIEW_REQUIRED

but status relationships should remain probabilistic rather than mechanically identical.

22. KYC PEP and Screening Scenarios

Synthetic PEP-like scenarios should be uncommon.

No PEP record may correspond to a known real person.

Screening statuses may include synthetic states such as:

CLEAR
POTENTIAL_MATCH
REVIEW_REQUIRED

These exist solely for engineering and data-quality scenarios.

23. Historical KYC Changes

A controlled subset of customers must experience historical KYC changes.

Examples:

verification renewal
risk-rating change
source-of-funds category change
verification-status change

Example history:

party_reference: PTY-00018492
risk_rating: LOW
effective_from: 2024-03-01
effective_to: 2025-10-12
is_current: false

followed by:

party_reference: PTY-00018492
risk_rating: MEDIUM
effective_from: 2025-10-12
effective_to: NULL
is_current: true

Historical periods must not overlap.

This dataset will later support SCD Type 2 processing.

24. Card Generation

Initial target:

120,000 cards

Cards should primarily be associated with eligible transactional accounts.

Some eligible accounts may have:

no active card
one card
more than one historical or replacement card
25. Card Identifier

Example:

CARD000001289

Card identifiers must be unique and stable.

26. Cross-System Account Mapping

The card processor must not directly expose the core banking account_id.

Core Banking example:

account_id = ACC000928177

Card Processor example:

processor_account_reference = CPR-928177

A deterministic controlled mapping must exist:

ACC000928177 <-> CPR-928177

Again, downstream processing must use controlled mapping logic rather than assume that string manipulation is sufficient.

27. Card Security Rules

Never generate or store:

full PAN
CVV
PIN
magnetic-stripe track data
usable payment-card credentials

Permitted synthetic masked representation:

************4837

or:

4444********4837

The resulting value must not be usable as a payment credential.

28. Card Types

Supported initial values:

DEBIT
BUSINESS_DEBIT

Card type must be compatible with account/product type.

29. Card Status

Suggested synthetic distribution:

ACTIVE        88%
BLOCKED        3%
EXPIRED        6%
CANCELLED      3%

Dates and status must be logically related.

For example, a newly issued card should not accidentally have an expiry date before its issue date.

30. Merchant Generation

Initial target:

approximately 30,000 merchants

The architecture should support later configuration between:

20,000-50,000

Each merchant should contain:

merchant_id
synthetic merchant_name
merchant_category_code
merchant_country_code
merchant_city

Merchant category must reference the controlled Merchant Category Code dataset.

31. Merchant Behaviour

Merchants must not receive identical transaction volumes.

The synthetic population should contain:

high-volume merchants
medium-volume merchants
low-volume merchants

This helps create realistic non-uniform workloads.

Merchant geography should contain:

predominantly UK merchants
smaller international population
32. Card Authorisation Generation

A card authorisation represents an attempt to obtain approval for a card transaction.

Supported outcomes:

APPROVED
DECLINED
REVERSED

Possible synthetic decline reasons:

INSUFFICIENT_FUNDS
CARD_BLOCKED
INVALID_CARD
LIMIT_EXCEEDED
SUSPECTED_FRAUD
EXPIRED_CARD

Not every authorisation should produce a posted card transaction.

This creates a realistic difference between:

authorisation activity

and:

posted financial activity
33. Card Transaction Volume

Initial target:

2,000,000 posted card transactions

The dataset may later increase toward:

5,000,000

only for targeted performance experiments.

34. Card Transaction Activity Distribution

Transaction activity must be non-uniform.

Some accounts/cards should produce:

very low activity
normal activity
high activity

A small subset should generate significantly more transactions than average.

This creates realistic skew for later Spark engineering exercises.

35. Card Transaction Amounts

Transaction amounts must use skewed distributions.

Suggested synthetic pattern:

common             £2-£100
moderate           £100-£500
less frequent      £500-£2,000
rare               >£2,000

Values must not simply be drawn from a uniform random distribution.

Transaction amounts must be positive business amounts.

Debit/credit behaviour must be represented through explicit transaction semantics rather than arbitrary negative values.

36. Card Transaction Timing

Transaction timestamps must exhibit realistic variation.

Include:

weekday/weekend differences
daytime peaks
lower overnight activity
seasonal variation
modest month-end effects
occasional late-arriving transactions

Do not distribute all timestamps uniformly across the historical period.

37. Card Transaction Channels

Suggested synthetic distribution:

POS             56%
ECOMMERCE       35%
ATM              9%

Channel assignment must be compatible with merchant and transaction context where practical.

38. Card Transaction Status

Supported initial values:

POSTED
REVERSED
REFUNDED

Where a transaction references an authorisation, the lifecycle must normally be logically consistent.

For example:

DECLINED authorisation

should not normally generate a successfully posted transaction.

39. Payment Generation

Initial target:

500,000 payments
40. Payment Types

Suggested synthetic mix:

FASTER_PAYMENT       46%
DIRECT_DEBIT         31%
STANDING_ORDER       12%
INTERNAL_TRANSFER    11%

These are project simulation assumptions.

41. Payment Amount Distribution

Payment amounts must be skewed.

Different payment types should exhibit different amount behaviour.

For example:

standing orders may show recurring amounts
direct debits may show recurring bill-like amounts
Faster Payments may have wider variation
internal transfers may include larger values

Do not assign every payment type the same amount distribution.

42. Payment Lifecycle

Normal lifecycle:

INITIATED
    ->
PROCESSING
    ->
COMPLETED

Possible alternative paths:

INITIATED
    ->
REJECTED

or:

INITIATED
    ->
PROCESSING
    ->
FAILED

or:

INITIATED
    ->
PROCESSING
    ->
COMPLETED
    ->
RETURNED

Each status transition must be represented by a corresponding payment-status event where appropriate.

43. Payment Status Distribution

The majority of payments should complete successfully.

Failures, rejections, returns and cancellations should be:

uncommon
non-zero
deliberately generated
traceable

Failure behaviour should not overwhelm the valid source population.

44. Payment Temporal Rules

Normal valid payments must satisfy:

initiated_timestamp
<= processing event
<= completed_timestamp

A returned payment must have been completed before it was returned.

Temporal violations should occur only where deliberately injected as test defects.

45. Settlement Batch Generation

Payments selected for settlement must be grouped into settlement batches.

Each batch contains:

settlement_batch_id
settlement_date
settlement_type
expected_item_count
expected_total_amount
settlement_currency
batch_status
received_timestamp
46. Settlement Items

Each settlement item must reference:

settlement_batch_id
payment_id
settlement_amount
settlement_currency
settlement_status

For valid completed batches:

COUNT(settlement_item)
=
settlement_batch.expected_item_count

and:

SUM(settlement_item.settlement_amount)
=
settlement_batch.expected_total_amount

subject only to explicitly documented exceptions.

47. Reference Data

Controlled reference datasets should include:

country
currency
transaction_type
payment_type
account_status
card_status
risk_band
merchant_category_code

Reference data should be:

low volume
version controlled where appropriate
effective dated where useful
used to validate operational source values

The generator must not independently invent domain values that are absent from the corresponding reference dataset unless deliberately injecting a domain defect.

48. Data Quality Defect Injection

The generator must deliberately introduce a small controlled population of invalid records.

Defects must never be introduced without tracking.

Suggested defect rates:

approximately 0.1%-0.5%

depending on dataset and defect type.

The exact rates must be configurable.

49. Duplicate Defects

Examples:

duplicate card_transaction_id
duplicate payment_id
duplicate customer-contact record
duplicate source file
repeated settlement item

Duplicates should include both:

exact duplicates
duplicate business keys with differing ingestion metadata

where appropriate.

50. Missing-Value Defects

Examples:

missing mandatory customer field
missing account reference
missing currency code
missing merchant identifier
missing payment status

Optional fields must not be incorrectly classified as defects.

51. Referential-Integrity Defects

Examples:

card -> unknown processor account

transaction -> unknown card

transaction -> unknown merchant

payment -> unknown debtor account

KYC profile -> unknown party reference

These defects should be rare and deliberately identifiable.

52. Domain-Validation Defects

Examples:

unsupported currency code
invalid account status
invalid card status
malformed payment status
unknown merchant category code
invalid debit/credit indicator

These records will later exercise Silver validation and quarantine processing.

53. Temporal Defects

Examples:

payment completed before initiation
card transaction before card issue date
account close date before open date
overlapping KYC history
overlapping account-status history
invalid post-close account activity

These defects must be deliberately injected rather than produced through faulty generator logic.

54. File-Level Defects

File-based sources should support controlled simulation of:

duplicate files
late-arriving files
missing expected files
empty files
malformed rows
schema drift
unexpected additional columns

These scenarios will later exercise ingestion controls.

55. Late-Arriving Data

A controlled subset of valid business events should arrive after their expected processing window.

Example:

transaction business date:
2026-06-14

source file arrival:
2026-06-16

The event itself is valid.

The engineering problem is its delayed arrival.

This distinction is important:

late data != bad data

The platform must eventually process valid late-arriving records correctly.

56. Defect Manifest

Every generation run containing injected defects must produce an expected-defect manifest.

Required fields:

run_id
source_system
entity_name
record_key
defect_type
defect_description
expected_action
generated_timestamp

Example expected actions:

QUARANTINE
REJECT
WARN
ACCEPT_WITH_FLAG

This allows comparison between:

expected defects

and:

pipeline-detected defects
57. Control Totals

Every financial batch/file should produce control totals where appropriate.

Example fields:

run_id
source_system
entity_name
file_name
business_date
expected_record_count
expected_debit_amount
expected_credit_amount
expected_net_amount
generated_timestamp

Control totals must be calculated from the generated source dataset rather than invented independently.

58. Record-Count Reconciliation

For an ingestion batch:

Source Record Count
=
Accepted Record Count
+
Rejected/Quarantined Record Count
+
Documented Exclusions

The pipeline must not silently lose records.

59. Monetary Reconciliation

For financial datasets:

Source Total Amount
=
Accepted Total Amount
+
Rejected/Held Total Amount
+
Documented Legitimate Exclusions

Any unexplained difference must cause reconciliation failure or investigation.

No financial discrepancy should be silently accepted.

60. Incremental Load Simulation

After the historical baseline, incremental generation must support:

new customers
changed customer records
new accounts
account status changes
account closures
new cards
replacement cards
card status changes
new authorisations
new card transactions
new payments
payment status changes
KYC changes
settlement batches
late-arriving records
source corrections
61. Incremental Run Metadata

Every incremental run must contain:

run_id
window_start
window_end
generation_timestamp
generation_mode
random_seed

Example:

run_id:
GEN-20260901-001

window_start:
2026-09-01T00:00:00

window_end:
2026-09-01T23:59:59
62. Watermark Behaviour

Source entities intended for incremental processing must contain an appropriate change indicator.

Examples:

modified_timestamp
updated_timestamp
event_timestamp
posting_date

A watermark represents the successfully processed source boundary.

The downstream ingestion framework must only advance its persisted watermark after successful durable processing of the intended batch.

63. CDC Simulation

Where appropriate, incremental data should include source-change semantics such as:

INSERT
UPDATE
DELETE

or logical closure where physical deletion is inappropriate.

Each CDC-style change should contain sufficient ordering information such as:

source_operation
source_change_timestamp
source_sequence

This allows deterministic handling of multiple changes to the same business key.

64. CDC Ordering

Multiple changes to the same record may occur inside one incremental window.

Example:

10:00 INSERT
13:20 UPDATE
18:05 UPDATE

The generator must preserve ordering through:

source_change_timestamp

plus a deterministic tie-breaker such as:

source_sequence

when timestamps are identical.

65. SCD Type 2 Scenarios

Historical tracking should be demonstrated on entities where business history matters.

Primary candidates:

customer KYC/risk history
account status history
selected product/reference attributes

A Type 2 record should support fields such as:

effective_from
effective_to
is_current

Only one current record should exist for a valid business key.

66. Idempotency

The generator must make pipeline rerun behaviour testable.

For a given deterministic generation specification:

same run_id
+
same seed
+
same configuration
+
same generation window

must not unexpectedly generate different business data.

The downstream platform should eventually be able to process the same source batch more than once without duplicating the valid target state.

67. Source-Specific Output Separation

Generated data must be separated according to its simulated operational source.

Target local structure:

data/
  generated/
    sql_server/
    rest_api/
    aws_s3/
    azure_blob/
    sharepoint/
    control/

This separation is important because each source will later use a different ingestion pattern.

68. SQL Server Source Output

The SQL Server simulator will eventually host relational datasets such as:

customer
account
account_status_history
branch
product

Relational integrity should normally be enforced for valid source records.

Controlled defect scenarios may be introduced through dedicated test mechanisms rather than corrupting every baseline table.

69. REST API Source Output

The API simulator will expose resources such as:

customer contact
KYC profile
verification event
customer risk history

The API must eventually support:

pagination
authentication
incremental filtering
deterministic responses
error simulation
rate-limit-like scenarios where useful

API complexity should be introduced only where it demonstrates an engineering requirement.

70. AWS S3 Source Output

The simulated S3 card-processing source will contain file-based datasets such as:

card
merchant
card authorisation
card transaction

File paths should eventually support business-date partitioning.

Example conceptual path:

card_transactions/
business_date=2026-09-01/

Exact physical conventions will be defined in the source contract.

71. Azure Blob Source Output

The simulated payment source will contain:

payment
payment status event
settlement batch
settlement item

Financial files should include associated control totals where appropriate.

72. SharePoint Source Output

SharePoint will simulate business-maintained reference datasets such as:

country
currency
payment type
account status
card status
risk band
merchant category code

These files should be small and business-readable.

Reference-data changes must be controlled and traceable.

73. Generated File Formats

File format must depend on the source-system scenario rather than using one format everywhere.

Potential formats include:

CSV
JSON
Parquet

The physical source contract will define which entity uses which format.

Format selection should serve a realistic engineering purpose.

74. Generated Data and Git

Large generated datasets must not be committed to Git.

Git should contain:

generator code
configuration
schemas
tests
small fixtures where necessary
documentation

Git should not contain millions of generated transaction records.

The appropriate generated-data paths must therefore be added to .gitignore.

75. Generator Architecture

Avoid implementing the complete solution as one large Python script.

The generator should eventually be separated into responsibilities such as:

configuration
schemas
reference generation
customer generation
account generation
KYC generation
card generation
transaction generation
payment generation
defect injection
control-total generation
manifest generation
validation
common utilities

This supports maintainability and testing.

76. Schema Validation

Every generated dataset must be validated against an explicit schema.

Validation should check:

expected columns
datatypes
nullability
key uniqueness
allowed domains
required relationships

Generation is not considered successful merely because a file was written.

77. Post-Generation Validation

Each generator run must produce validation results.

At minimum:

entity
generated_record_count
expected_record_count
primary_key_duplicates
unexpected_nulls
referential_failures
expected_injected_defects
unexpected_defects
validation_status

The generator should fail when unexpected structural problems occur.

Known deliberately injected defects must be distinguished from accidental generator defects.

78. Logging

Generator execution must produce structured logging sufficient to answer:

which run executed?
which configuration was used?
which entity was generated?
how many records were generated?
how long did generation take?
which defects were injected?
did validation pass?
where was the output written?

Avoid relying solely on ad-hoc print() statements.

79. Exception Handling

Generation failures must not be silently ignored.

Exceptions should provide useful context including:

run_id
entity
generation stage
relevant configuration
error reason

Partial output must be identifiable so that incomplete generation is not mistaken for a successful batch.

80. Testability

Generator components should be designed for automated testing.

Tests should eventually include:

deterministic-generation tests
identifier uniqueness tests
schema tests
referential-integrity tests
temporal-rule tests
distribution sanity tests
defect-injection tests
reconciliation tests
incremental-generation tests
idempotency tests
81. Small Test Fixtures

Automated tests should use small datasets.

Example:

customers: 100
accounts: 150
transactions: 1,000
payments: 500

CI/CD must not generate millions of records merely to test basic generator logic.

82. Performance-Test Dataset

A separate configuration may later generate larger datasets specifically for:

Spark shuffle analysis
skew testing
join strategy comparison
partition testing
Delta MERGE testing
file compaction testing

Performance scale must be increased only when there is a specific engineering question to answer.

83. Data Skew Simulation

A controlled subset of transaction activity should create skew.

For example:

a small percentage of merchants receive disproportionately high transaction volumes
a small percentage of accounts generate high activity

The skew must be intentional and documented.

This will later allow analysis of:

partition imbalance
shuffle behaviour
join performance
mitigation techniques
84. Schema Evolution Simulation

Later incremental runs should support controlled schema-evolution scenarios.

Examples:

new optional source column
changed column ordering
additional JSON field
deprecated field

Breaking schema changes must not be introduced accidentally.

Each schema-evolution scenario must be deliberate and documented.

85. Source File Naming

Every generated file-based batch must eventually follow a deterministic naming convention.

The exact convention will be defined in the source contract.

Names should contain sufficient information to identify attributes such as:

entity
business date
batch/run identifier

Avoid ambiguous names such as:

data.csv
final.csv
new_file.csv
86. Batch Traceability

Every generated source batch must be traceable to a generation run.

Where appropriate, technical metadata should support:

run_id
batch_id
business_date
generation_timestamp
source_system

These attributes will later connect:

source generation
->
ingestion
->
Bronze
->
Silver
->
reconciliation
->
audit
87. Bronze Metadata Compatibility

Although source records should remain source-aligned, the generation design must allow ingestion to add technical metadata such as:

_ingestion_timestamp
_source_system
_source_file
_batch_id
_pipeline_run_id
_record_hash

These are ingestion metadata and should not be confused with operational business attributes.

88. Separation of Valid Data and Test Defects

The baseline generation process must first be capable of creating valid data.

Defect injection should then operate as a controlled engineering stage.

Conceptually:

Generate valid business data
        ↓
Validate baseline
        ↓
Inject configured defects
        ↓
Record defects in manifest
        ↓
Write source output
        ↓
Produce control totals

This prevents accidental generator bugs from being mistaken for intentional bad data.

89. Expected Versus Unexpected Defects

The system must distinguish:

EXPECTED DEFECT

A deliberately injected test condition.

from:

UNEXPECTED DEFECT

A bug or unintended inconsistency in the generator.

Unexpected defects should fail validation.

This distinction is critical for trustworthy data-quality testing.

90. Reproducibility Manifest

Each successful generation run should eventually produce a manifest containing:

run_id
generation_mode
random_seed
window_start
window_end
configuration_version
generation_timestamp
entity_counts
output_locations
defect_count
validation_status

This provides an auditable record of how a synthetic dataset was produced.

91. Initial Portfolio Volumes

Default initial targets:

customers              100,000
accounts               150,000
cards                  120,000
card_transactions    2,000,000
payments               500,000
kyc_profiles          ~100,000+
merchants               30,000
branches                  ~100
products                   <50

These values must remain configurable.

92. Cost and Compute Principle

Synthetic-data scale must serve an engineering objective.

Do not generate or process unnecessarily large datasets simply to make the project appear larger.

The project principle is:

production-style engineering
+
portfolio-scale infrastructure

Scale should be increased only when required to demonstrate behaviour that cannot be meaningfully observed on smaller data.

93. Production-Quality Principle

The synthetic-data generator is part of the engineering solution rather than disposable setup code.

It must therefore eventually include:

configuration-driven behaviour
deterministic seeds
modular implementation
explicit schemas
logging
validation
exception handling
reproducibility
control totals
defect manifests
incremental-generation support
automated tests
documentation

Avoid creating one large Python script containing all generation logic.