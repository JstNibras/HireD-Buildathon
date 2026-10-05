# HireD Technical Decisions — Final Submission Version

> **Database Design / ER Diagram:**  
> https://www.drawdb.app/share/nMLatUMJIR5HmSo_CppOUrzf

> **Submission note:** This document defines the final schema decisions that the implementation and DrawDB ER diagram must match. The DrawDB diagram must be updated to these final names, values, constraints, indexes, and fields before submission.

## 1. Document Overview

This document records the significant technical and database design decisions for HireD. It is intended to keep the implemented SQL schema, ER diagram, backend behavior, and workflow documentation consistent.

The supported operational flow is:

**B2B Company → Order → Trip → Driver Assignment → Trip Completion → Payroll → Driver Wallet → Monthly Invoice → Two-Person Verification → Publication**

---

## 2. Decision Summary

| Decision | Final approach |
|---|---|
| User identity | Single `users` table |
| User roles | `admin`, `partner`, `driver` |
| Company association | `users.company_id`; partner users must have a company, admin/driver users must not |
| Driver profile | Separate `drivers` table linked one-to-one to `users` |
| Database | PostgreSQL |
| Backend architecture | Modular Monolith |
| Primary keys | Integer auto-increment |
| Money type | `NUMERIC(12,2)`; invoice totals `NUMERIC(14,2)` |
| Timestamp type | `timestamptz` |
| Billing timezone | `Asia/Kolkata` (IST), using half-open monthly ranges |
| Database naming | `snake_case` |
| Lifecycle enums | Explicit PostgreSQL enums with application-safe names and values |
| Current payout configuration | `companies.driver_share_pct`, `companies.company_share_pct` |
| Payout history | `company_payout_history` with old/new share columns |
| Payroll snapshot | `trip_payroll_records` |
| Wallet | `wallets` + append-only `wallet_transactions` |
| Invoice | `invoices` + `invoice_items` |
| Invoice verification | Creator approval + second-admin approval with timestamps |
| Atomic trip close | One transaction with row lock and layered duplicate protection |
| Money rounding | Driver share rounded to 2 decimals; company share is the remainder |
| FK delete rules | Cascade only for dependent operational children; no action for financial history |
| Notifications | Queue row inserted with state change; worker sends later |
| Audit | Centralized `audit_logs` |
| Constraints outside DBML | Migration-level checks, partial indexes, triggers, and cross-state rules |
| Intentional denormalization | `trips.company_id`, wallet balance, payroll snapshots, invoice snapshots/totals |
| Duplicate trip prevention | Unique `trips.order_id` |
| Duplicate payroll prevention | Unique `trip_payroll_records.trip_id` |
| Duplicate wallet credit prevention | Partial unique index on wallet credits |
| Duplicate invoice-item prevention | Unique `invoice_items.trip_id` |
| Active assignment prevention | Partial unique index on active assignment |
| Driver document uniqueness | Unique `(driver_id, doc_type)` |
| Invoice monthly uniqueness | Unique `(company_id, period_month)` |

---

## 3. User Identity and Role Design

### Decision

Use one central `users` table for authentication and role identification.

Supported roles:

- `admin`
- `partner`
- `driver`

### Reason

A single identity table provides one authentication model while role-specific operational data remains in dedicated tables.

### Trade-offs

Authentication is centralized, but authorization must be enforced in application services and middleware.

---

## 4. Company Association Design

### Decision

Associate partner users with companies through:

```text
users.company_id → companies.id
```

The database rule is:

```sql
CHECK (
    (role = 'partner' AND company_id IS NOT NULL)
    OR
    (role IN ('admin', 'driver') AND company_id IS NULL)
)
```

### Reason

Partners operate within one B2B company. Admins are platform-wide users. Driver company ownership is represented through operational relationships rather than `users.company_id`.

### Trade-offs

The rule is explicit and prevents invalid role/company combinations, but the application must also enforce company-scoped authorization.

---

## 5. Driver Profile Separation

### Decision

Keep driver-specific operational information in `drivers`, linked one-to-one to `users` by unique `drivers.user_id`.

Related tables:

- `driver_documents`
- `driver_availability`
- `trip_assignments`
- `driver_expenses`
- `wallets`

### Reason

Authentication identity and driver operations have different responsibilities.

### Trade-offs

This requires joins but improves separation of concerns and keeps the user table focused on identity.

---

## 6. Relational Database Structure and FK Delete Rules

### Decision

Use a normalized relational model with primary keys, foreign keys, unique constraints, CHECK constraints, indexes, and controlled delete behavior.

### Foreign-key delete policy

**Cascade only for dependent child rows:**

- `driver_documents`
- `driver_availability`
- `trip_assignments`
- `driver_expenses`
- `invoice_items`

**No action for financial/history records**, including:

- payroll records
- wallet transactions
- invoices
- audit logs
- company payout history

### Reason

Operational child records can safely follow their parent where they have no independent financial meaning. Financial and audit history must not disappear accidentally through parent deletion.

### Trade-offs

The stricter financial-history policy may require explicit archival/deactivation instead of deletion, but protects historical integrity.

---

## 7. Company Payout Configuration and History

### Decision

Store the current payout split in `companies`:

```text
driver_share_pct
company_share_pct
```

The split must satisfy:

```text
driver_share_pct >= 0
company_share_pct >= 0
driver_share_pct + company_share_pct = 100
```

Store history in `company_payout_history` using the actual columns:

```text
old_driver_share
old_company_share
new_driver_share_pct
new_company_share_pct
changed_by
changed_at
```

### Reason

The current configuration is easy to retrieve, while historical changes remain auditable.

### Trade-offs

Current configuration and history are stored separately, but this preserves both operational simplicity and auditability.

---

## 8. Historical Payout Rate Snapshot

### Decision

At trip close, freeze the applicable payout rates and calculated amounts in `trip_payroll_records`:

```text
trip_amount
driver_share_pct
company_share_pct
driver_share_amount
company_share_amount
```

### Reason

Changing a company's current payout configuration must never change an already-closed trip.

### Trade-offs

Financial values are intentionally repeated as historical snapshots.

---

## 9. One Payroll Record Per Closed Trip

### Decision

`trip_payroll_records.trip_id` is unique.

### Reason

A closed trip can create only one payroll record.

This is one layer of duplicate-payout protection; it is combined with a row lock and a partial unique wallet-credit index during atomic trip close.

---

## 10. Driver Wallet Design

### Decision

Use:

```text
wallets
wallet_transactions
```

`wallets.balance` is the current balance, while `wallet_transactions` is the append-only financial ledger.

### Reason

The balance supports fast reads; the ledger preserves financial history.

### Trade-offs

The balance and ledger must be updated atomically.

---

## 11. Append-Only Wallet Ledger

### Decision

`wallet_transactions` is append-only. Wallet credits created by trip close must be unique per trip.

Required migration-level index:

```sql
CREATE UNIQUE INDEX uq_wallet_credit_trip
ON wallet_transactions(trip_id)
WHERE type = 'credit';
```

### Reason

This prevents a retry or duplicate request from crediting a driver twice for the same trip.

---

## 12. Trip Assignment History

### Decision

Keep assignment history in `trip_assignments`.

The active-assignment rule is enforced by:

```sql
CREATE UNIQUE INDEX uq_active_trip_assignment
ON trip_assignments(trip_id)
WHERE unassigned_at IS NULL;
```

### Reason

A trip may be reassigned, but only one assignment can be active at a time.

### Trade-offs

Historical assignments remain queryable while the partial unique index provides database-level protection against concurrent active assignments.

---

## 13. One Order to One Trip

### Decision

Use a unique `trips.order_id`.

### Reason

Phase 1 defines:

```text
One Order = One Trip
```

The unique constraint prevents duplicate trip creation for an order.

---

## 14. Trip Company Denormalization

### Decision

Keep `trips.company_id` even though the trip also references `orders`.

The value is copied from the order **inside the order-acceptance transaction**.

### Reason

It provides fast company-scoped trip queries without repeatedly joining through orders.

### Trade-offs

It duplicates a relationship, so the accept transaction must set it from the authoritative order company.

---

## 15. Invoice Structure

### Decision

Use:

```text
invoices
invoice_items
```

`invoices` stores monthly-level information; `invoice_items` stores trip-level invoice snapshots.

---

## 16. Invoice Financial Snapshot

### Decision

Store trip-level invoice financial values in `invoice_items`:

- `total_trip_amount`
- `driver_share_pct`
- `company_share_pct`
- `driver_share_amount`
- `company_share_amount`

### Reason

An invoice must remain historically reproducible even if later configuration changes.

---

## 17. One Invoice Per Company Per Month

### Decision

Use:

```text
UNIQUE(company_id, period_month)
```

### Reason

Phase 1 defines one monthly invoice/statement per company.

---

## 18. Two-Person Invoice Verification

### Decision

Record both approvals with timestamps:

```text
created_by
first_approved_at
second_verifier_id
second_approved_at
```

Required rule:

```sql
CHECK (second_verifier_id <> created_by)
```

The invoice can become `verified` or `published` only when both approvals are present.

### Verification state rule

```text
draft
  ↓
pending_verification
  ↓
verified
  ↓
published
```

`verified` requires:

```text
first_approved_at IS NOT NULL
second_verifier_id IS NOT NULL
second_approved_at IS NOT NULL
```

`published` requires the same completed approvals and a valid verification state.

### Reason

Two-person verification separates invoice creation from final verification and records when each approval occurred.

### Trade-offs

Publication requires two administrators, but materially improves financial control and auditability.

---

## 19. Audit Log Design

### Decision

Use centralized `audit_logs` for significant business changes.

At minimum, audit:

- payout split changes
- trip amount changes
- driver assignment/reassignment
- invoice verification
- driver document replacement

Store:

```text
entity_type
entity_id
action
actor_id
created_at
meta_data
```

---

## 20. Notification Design and Queue

### Decision

Use `notifications` with:

```text
user_id
channel
event
payload
status
created_at
sent_at
read_at
entity_type
entity_id
```

The notification row is inserted in the **same transaction as the business state change**. A worker sends the notification later.

### Reason

This prevents lost notification events while ensuring email failure does not roll back the underlying business transaction.

### Trade-offs

A worker/queue adds operational complexity, but it decouples business transactions from email delivery.

---

## 21. Driver Documents

### Decision

Store private document references in `driver_documents.file_url`.

Enforce:

```sql
UNIQUE(driver_id, doc_type)
```

A document update overwrites the existing row for that document type. Both the old and new document references are recorded in `audit_logs`.

### Reason

Each driver has one current document per document type while replacements remain auditable.

### Trade-offs

The current row is simple to access, while audit history preserves previous references.

---

## 22. Driver Availability Design

### Decision

Store one availability record per driver per date:

```text
UNIQUE(driver_id, available_date)
```

### Reason

This prevents duplicate availability rows for the same driver and date.

---

## 23. Indexing Strategy

Required indexes include:

### Orders

```text
(company_id, created_at)
(status)
```

### Trips

```text
(driver_id, status)
(company_id, created_at)
(company_id, closed_at)
(status)
```

### Payroll

```text
(company_id, closed_at)
(driver_id, closed_at)
```

### Wallet

```text
(wallet_id, created_at)
```

### Audit

```text
(entity_type, entity_id, created_at)
```

### Required indexes previously missing from the documentation

```text
invoices(status)
invoice_items(invoice_id)
trip_assignments(trip_id, assigned_at)
driver_expenses(trip_id)
driver_documents(driver_id, doc_type)
company_payout_history(company_id, changed_at)
```

### Reason

These support frequent operational filtering, month-end queries, assignment history, expense lookup, document lookup, invoice processing, and audit/history queries.

---

## 24. Duplicate and Uniqueness Rules

The final duplicate-prevention rules include:

```text
users.email
users.phone
companies.name
drivers.user_id
orders.order_no
trips.trip_no
trips.order_id
trip_payroll_records.trip_id
wallets.driver_id
invoices.invoice_no
invoices(company_id, period_month)
invoice_items.trip_id
driver_availability(driver_id, available_date)
driver_documents(driver_id, doc_type)
```

Additionally, migration-level partial unique indexes enforce:

```text
wallet_transactions(trip_id) WHERE type = 'credit'
trip_assignments(trip_id) WHERE unassigned_at IS NULL
```

---

## 25. Financial Data, Rounding and Atomic Trip Close

### 25.1 Financial stages

Financial data is separated into:

```text
Trip → Payroll → Invoice
```

- `trips.amount` is the accepted trip amount.
- `trip_payroll_records` freezes payout rates and calculated shares.
- `invoice_items` snapshots trip-level billing values.
- `invoices` stores monthly totals.

### 25.2 Money rounding

Driver share is calculated as:

```text
driver_share_amount =
ROUND(trip_amount × driver_share_pct / 100, 2)
```

Company share is calculated as:

```text
company_share_amount =
trip_amount - driver_share_amount
```

This guarantees that:

```text
driver_share_amount + company_share_amount = trip_amount
```

exactly, while keeping the calculation deterministic and reproducible.

### 25.3 Atomic trip close and idempotency

Trip close is one database transaction with a row lock on the trip.

The transaction performs, in order:

1. Lock the trip row.
2. Validate the current trip state.
3. Validate the OTP.
4. Validate/confirm expenses.
5. Close the trip.
6. Calculate and round the driver/company shares.
7. Create the unique payroll record.
8. Create the unique wallet credit.
9. Update the wallet balance.
10. Create required notification/audit records.
11. Commit.

Duplicate protection is layered:

- row lock on the trip
- unique `trip_payroll_records.trip_id`
- partial unique wallet-credit index

A **wrong OTP** increments `otp_attempts` in its own committed statement; it must not roll back the attempt count.

### Reason

Trip close is the core financial operation. It must either complete consistently or prevent duplicate payout.

### Trade-offs

The transaction is more complex than independent updates, but it provides atomicity, deterministic financial results, and strong retry protection.

---

## 26. Schema Corrections Applied

This section records the final corrections that must be reflected in the implementation and DrawDB ER diagram. These are no longer open review items.

### 26.1 Invoice approval timestamp

Replace the incorrect `first_approved_id` concept with:

```text
first_approved_at timestamptz
second_approved_at timestamptz
```

Keep:

```text
created_by
second_verifier_id
```

and enforce:

```sql
CHECK (second_verifier_id <> created_by)
```

### 26.2 Final lifecycle values

Use these canonical values:

- Order: `received`, `accepted`, `rejected`
- Trip: `unassigned`, `assigned`, `ongoing`, `closed`
- Driver: `pending`, `active`, `inactive`
- Payout: `pending`, `completed`
- Partner payout/payment: `pending`, `completed`

The previous `cmopleted` typo is removed.

### 26.3 Final enum names

Use explicit enum names:

```text
order_status_t
trip_status_t
driver_status_t
payout_status_t
partner_pay_status_t
```

Do not use auto-generated names such as:

```text
status_assigned_unassigned_ongoing_closed_t
status_pending_accepted_rejected_t
status_Active_pending_inactive_t
```

### 26.4 Driver login field

Rename:

```text
is_loged
```

to:

```text
is_online
```

### 26.5 Trip expense confirmation

Add:

```text
expenses_confirmed_at timestamptz NULL
```

to `trips`.

### 26.6 Timestamp defaults

All standard `created_at` and `updated_at` fields use:

```sql
DEFAULT now()
```

`updated_at` is maintained by the application's update behavior and/or the migration-level updated-at trigger.

### 26.7 Invoice payment status

Add:

```text
payment_status
paid_at timestamptz NULL
```

to `invoices`.

`payment_status` tracks invoice settlement independently of invoice workflow status.

### 26.8 Company payout history nullable old values

The previous payout values are nullable because a first-time configuration may not have an earlier value:

```text
old_driver_share NULL
old_company_share NULL
```

The new values remain:

```text
new_driver_share_pct
new_company_share_pct
```

### 26.9 Notification entity reference and read state

Add:

```text
read_at timestamptz NULL
entity_type
entity_id
```

to `notifications`.

### 26.10 Payout split constraint

Enforce at database/migration level:

```sql
CHECK (
    driver_share_pct >= 0
    AND company_share_pct >= 0
    AND driver_share_pct + company_share_pct = 100
)
```

### 26.11 Active assignment constraint

Enforce:

```sql
CREATE UNIQUE INDEX uq_active_trip_assignment
ON trip_assignments(trip_id)
WHERE unassigned_at IS NULL;
```

### 26.12 Required missing indexes

Ensure these exist:

```text
invoices(status)
invoice_items(invoice_id)
trip_assignments(trip_id, assigned_at)
driver_expenses(trip_id)
driver_documents(driver_id, doc_type)
company_payout_history(company_id, changed_at)
```

### 26.13 Required uniqueness

Ensure:

```text
UNIQUE(driver_id, doc_type)
```

and:

```sql
CREATE UNIQUE INDEX uq_wallet_credit_trip
ON wallet_transactions(trip_id)
WHERE type = 'credit';
```

### 26.14 Invoice approval state constraints

Migration-level logic must prevent:

- `verified` without both approvals
- `published` without both approvals

The same admin cannot be both verifiers.

---

## 27. Primary Key and ID Strategy

### Decision

Use integer auto-increment primary keys for database entities.

```text
id integer PK auto-increment
```

### Reason

This is simple, efficient for relational references, and consistent across the schema.

### Alternatives

- UUID primary keys
- Application-generated identifiers

### Trade-offs

Integer IDs are efficient, but they should not be treated as security-sensitive public identifiers.

---

## 28. Money and Financial Data Type

### Decision

Use:

```text
NUMERIC(12,2)
```

for standard monetary fields and:

```text
NUMERIC(14,2)
```

for invoice totals where required by the schema.

Use deterministic rounding:

```text
driver share = ROUND(amount × pct / 100, 2)
company share = amount - driver share
```

### Reason

Fixed-precision numeric values avoid floating-point financial errors and make results reproducible.

### Alternatives

- floating-point types
- integer minor units such as paise

### Trade-offs

`NUMERIC` is slightly more computationally expensive than floating point, but financial correctness is more important.

---

## 29. Timestamp and Billing Timezone Strategy

### Decision

Use `timestamptz` for event timestamps.

For monthly billing, compute month boundaries in:

```text
Asia/Kolkata (IST)
```

using a half-open range:

```text
[start_of_month, start_of_next_month)
```

### Reason

The system needs timezone-aware event storage while billing periods must follow the business calendar in India.

### Alternatives

- `timestamp without time zone`
- UTC-only monthly boundaries without converting to IST

### Trade-offs

The application must handle timezone conversion correctly, but the approach avoids ambiguity at month boundaries.

---

## 30. Database Naming Convention

### Decision

Use `snake_case` for tables, columns, indexes, and database identifiers.

Examples:

```text
company_payout_history
trip_payroll_records
driver_share_pct
second_verifier_id
created_at
```

### Reason

Consistent naming makes SQL and schema navigation predictable.

### Alternatives

- camelCase
- PascalCase

### Trade-offs

The application may use camelCase properties, requiring mapping at the query/ORM layer.

---

## 31. Enum vs VARCHAR Policy

### Decision

Use PostgreSQL enums for stable lifecycle states and controlled payout states:

```text
trip_status_t
  unassigned
  assigned
  ongoing
  closed

order_status_t
  received
  accepted
  rejected

driver_status_t
  pending
  active
  inactive

payout_status_t
  pending
  completed

partner_pay_status_t
  pending
  completed
```

`invoices.status` is also an enum:

```text
invoice_status_t
  draft
  pending_verification
  verified
  published
```

Use `varchar` for intentionally flexible/non-lifecycle fields:

```text
users.role
wallet_transactions.type
orders.source
notifications.channel
notifications.event
notifications.status
```

These varchar exceptions must be validated by the application because they are intentionally not database enums.

### Reason

Stable lifecycle states benefit from database-level integrity. Flexible fields may evolve without requiring enum migrations.

### Alternatives

- enums for every controlled field
- varchar for every status/value

### Trade-offs

Enums provide stronger integrity but make value changes schema changes. Varchar exceptions provide flexibility but require application validation.

---

## 32. Why PostgreSQL?

### Decision

Use PostgreSQL as the primary SQL database.

### Reason

PostgreSQL directly supports the required relational features: foreign keys, transactions, unique constraints, CHECK constraints, partial unique indexes, fixed-precision `NUMERIC`, `timestamptz`, JSONB audit metadata, and enums.

### Alternatives

- MySQL
- SQL Server
- Oracle
- SQLite

### Trade-offs

PostgreSQL has a strong feature set for transactional business systems, but the team must operate and maintain a PostgreSQL environment.

---

## 33. Why Modular Monolith?

### Decision

Use a Modular Monolith: one deployable backend application with clear internal business modules.

Suggested modules:

```text
auth
users
companies
orders
trips
drivers
expenses
payroll
wallet
invoices
reports
notifications
```

### Reason

HireD has multiple business domains but does not require independently deployed microservices in Phase 1. A Modular Monolith simplifies deployment, local development, database transactions, testing, and operational management while maintaining clear boundaries.

### Alternatives

- microservices
- unstructured monolith
- distributed/serverless services

### Trade-offs

All modules share one deployment unit and database boundary, so module boundaries must be respected to avoid tight coupling.

---

## 34. Alternatives Considered

### 34.1 Separate Admin Table

**Decision:** Use `users.role = admin`.

**Reason:** Authentication and role identification remain centralized.

**Trade-offs:** Less table duplication, but role authorization must be enforced correctly.

### 34.2 Store Driver Data Directly in Users

**Decision:** Use `users → drivers`.

**Reason:** Driver operational data is separate from authentication identity.

**Trade-offs:** Additional joins, but cleaner responsibility boundaries.

### 34.3 Store Only Current Payout Rates

**Decision:** Use `companies` + `company_payout_history` + `trip_payroll_records`.

**Reason:** Current configuration, change history, and historical trip snapshots have different purposes.

**Trade-offs:** Some payout values are intentionally repeated for historical correctness.

### 34.4 Store Wallet Balance Without Transactions

**Decision:** Use `wallets.balance` plus `wallet_transactions`.

**Reason:** Fast balance reads and complete financial history are both required.

**Trade-offs:** Balance and ledger must be updated atomically.

### 34.5 Overwrite Driver Assignment

**Decision:** Keep `trips.driver_id` plus `trip_assignments`.

**Reason:** Current assignment is fast to access while reassignment history is preserved.

**Trade-offs:** More data and a partial unique index are required, but auditability is stronger.

---

## 35. Decision Principles

1. Centralize authentication in `users`.
2. Separate role-specific operational data.
3. Keep company ownership explicit on operational entities.
4. Preserve historical financial values through snapshots.
5. Maintain an append-only wallet ledger.
6. Prevent duplicate financial operations at the database level.
7. Preserve assignment history.
8. Require two-person invoice verification.
9. Use fixed-precision financial types and deterministic rounding.
10. Use timezone-aware timestamps and IST billing boundaries.
11. Protect financial and audit history from accidental cascades.
12. Insert notification events atomically with business state changes.
13. Keep intentional denormalization explicit and controlled.
14. Use database constraints wherever the relational model can enforce the rule.
15. Keep migration-level constraints documented where DrawDB/DBML cannot express them.
16. Keep the implemented schema and ER diagram synchronized.

---

## 36. Final Technical Decision Summary

The final HireD database and backend design is:

```text
Company
   ↓
Order
   ↓
Trip
   ↓
Driver Assignment
   ↓
Trip Completion
   ↓
Payroll
   ↓
Driver Wallet
   ↓
Monthly Invoice
   ↓
Two-Person Verification
   ↓
Publication
```

The major decisions are:

- PostgreSQL as the primary SQL database.
- Modular Monolith backend architecture.
- Integer auto-increment IDs.
- `NUMERIC` financial fields with deterministic rounding.
- `timestamptz` timestamps and IST monthly billing boundaries.
- `snake_case` database naming.
- Explicit lifecycle enums with stable names.
- Centralized users and explicit partner company association.
- Current payout configuration plus historical payout records.
- Frozen payroll rates and invoice snapshots.
- Atomic, idempotent trip close with row locking.
- Wallet balance plus append-only ledger and unique wallet credits.
- One active assignment per trip.
- Two-person invoice verification with both approval timestamps.
- Invoice payment tracking through `payment_status` and `paid_at`.
- Private driver documents with one current row per document type and audit history.
- Notification queue records created atomically with state changes.
- Controlled foreign-key deletion rules protecting financial history.
- Intentional denormalization documented explicitly.
- Migration-level constraints and partial indexes documented where DBML cannot express them.

### Intentional denormalization summary

The following repeated values are deliberate:

- `trips.company_id` — copied from the order during acceptance for fast company-scoped queries.
- `wallets.balance` — maintained as a current balance in addition to the ledger.
- Payroll payout rates and amounts — frozen historical snapshot.
- `invoice_items` financial values — invoice snapshot.
- `invoices` monthly totals — stored to avoid recalculating the complete invoice on every read.

### Final implementation requirement

The backend migrations, actual SQL schema, seed data, API validation, workflow logic, and DrawDB ER diagram must all match this document. No previous field name, enum value, typo, or unresolved review item should remain in the final submission.

---

## 37. ER Diagram

The ER diagram is maintained in DrawDB:

https://www.drawdb.app/share/nMLatUMJIR5HmSo_CppOUrzf

Before submission, update the DrawDB diagram so it exactly reflects the final schema decisions in Sections 26–36, including:

- renamed enums and canonical enum values
- `first_approved_at` and `second_approved_at`
- `is_online`
- `expenses_confirmed_at`
- invoice `payment_status` and `paid_at`
- notification `read_at`, `entity_type`, and `entity_id`
- nullable `old_driver_share` and `old_company_share`
- all required unique constraints and indexes
- all required foreign keys and delete rules

### Constraints implemented in migrations rather than represented fully by DBML

DrawDB/DBML cannot express every implementation constraint. The migration layer must therefore contain and document:

- payout split CHECK constraints
- role/company association CHECK
- second verifier differs from creator CHECK
- invoice verified/published approval requirements
- trip state transition checks
- partial unique wallet-credit index
- partial unique active-assignment index
- `UNIQUE(driver_id, doc_type)`
- updated-at trigger/behavior

The ER diagram remains the structural source of truth for entities and relationships, while migrations are the source of truth for database features that DBML cannot represent directly.
