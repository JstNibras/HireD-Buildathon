# HireD Technical Decisions

> **Database Design URL:**
> https://www.drawdb.app/share/nMLatUMJIR5HmSo_CppOUrzf

## 1. Document Overview

This document records the important technical and database design
decisions for the HireD platform.

The decisions are based on the current HireD database design and support
the complete operational flow:

**B2B Company → Order → Trip → Driver Assignment → Trip Completion →
Payroll → Driver Wallet → Monthly Invoice → Verification → Publication**

The purpose of this document is to make the reasoning behind the
database structure clear and maintainable for the development team.

------------------------------------------------------------------------

## 2. Decision Summary

  Decision                            Selected Approach
  ----------------------------------- ---------------------------------------------
  User identity                       Single `users` table
  User roles                          `admin`, `partner`, `driver`
  Company association                 `users.company_id` for partner users
  Driver profile                      Separate `drivers` table linked to `users`
  Database structure                  Normalized relational design
  Company payout configuration        Stored in `companies`
  Payout history                      Stored in `company_payout_history`
  Trip financial snapshot             Stored in `trip_payroll_records`
  Driver wallet                       `wallets` + `wallet_transactions`
  Wallet history                      Append-only transaction ledger
  Assignment history                  Separate `trip_assignments` table
  Invoice structure                   `invoices` + `invoice_items`
  Invoice verification                Two-person verification
  Invoice monthly uniqueness          `company_id + period_month`
  Audit trail                         `audit_logs`
  Notifications                       `notifications`
  Duplicate trip prevention           Unique `trips.order_id`
  Duplicate payroll prevention        Unique `trip_payroll_records.trip_id`
  Duplicate invoice-item prevention   Unique `invoice_items.trip_id`
  Query optimization                  Targeted indexes for common access patterns

------------------------------------------------------------------------

## 3. User Identity and Role Design

### Decision

Use one central `users` table for authentication and role
identification.

Supported roles:

-   `admin`
-   `partner`
-   `driver`

### Reason

A single user identity table provides a common authentication structure
for all system users. The database does not use a separate `admins`
table.

Driver-specific information is stored separately in `drivers`, while
partner company association is represented through `users.company_id`.

### Trade-off

This centralizes authentication but requires role-based application
logic to determine which features and data a user can access.

------------------------------------------------------------------------

## 4. Company Association Design

### Decision

Associate partner users with companies through:

``` text
users.company_id → companies.id
```

### Reason

A partner user needs to access data belonging to their company. Admins
can have:

``` text
company_id = NULL
```

because admins operate across companies.

### Trade-off

Company ownership is centralized through the user-to-company
relationship, while operational records independently maintain their
company foreign keys.

------------------------------------------------------------------------

## 5. Driver Profile Separation

### Decision

Keep driver-specific information in a separate `drivers` table rather
than storing all driver information directly in `users`.

``` text
users 1 ─────── 1 drivers
```

`drivers.user_id` is unique.

### Reason

The `users` table is responsible for identity and authentication, while
`drivers` contains driver-specific operational information.

Related driver information is further separated into:

-   `driver_documents`
-   `driver_availability`
-   `trip_assignments`
-   `driver_expenses`
-   `wallets`

### Trade-off

This introduces additional joins, but keeps authentication data and
driver operational data separated.

------------------------------------------------------------------------

## 6. Relational Database Structure

### Decision

Use a normalized relational database structure with foreign keys, unique
constraints, indexes, and relationship tables.

### Reason

HireD contains strongly related business entities:

``` text
Company
   ↓
Order
   ↓
Trip
   ↓
Driver
   ↓
Payroll
   ↓
Wallet
   ↓
Invoice
```

Foreign-key relationships maintain these associations.

### Trade-off

A relational structure requires joins between related entities, but
provides explicit relationships and constraints for the business
workflow.

------------------------------------------------------------------------

## 7. Company Payout Configuration

### Decision

Store the current company payout split in `companies`:

``` text
driver_share_pct
company_share_pct
```

Business rule:

``` text
driver_share_pct + company_share_pct = 100
```

### Reason

The payout split is defined at company level.

### Historical Changes

Changes are recorded in `company_payout_history`, including the previous
and new percentages, the user who made the change, and the change
timestamp.

### Trade-off

The current configuration is easy to retrieve from `companies`, while
historical changes require reading `company_payout_history`.

------------------------------------------------------------------------

## 8. Historical Payout Rate Snapshot

### Decision

Store the payout percentages and calculated amounts in
`trip_payroll_records` when a trip is closed.

Stored values include:

``` text
trip_amount
driver_share_pct
company_share_pct
driver_share_amount
company_share_amount
```

### Reason

Historical financial records must not change when a company's payout
configuration changes later.

Example:

``` text
Trip Amount = ₹10,000

Driver Share = 30%
Company Share = 70%

Driver Amount  = ₹3,000
Company Amount = ₹7,000
```

### Trade-off

The payout percentage is duplicated between company configuration and
payroll records, but this is intentional because it preserves the
historical financial snapshot.

------------------------------------------------------------------------

## 9. One Payroll Record Per Closed Trip

### Decision

Make `trip_payroll_records.trip_id` unique.

### Reason

A trip should not generate multiple payroll records. The unique
constraint provides database-level protection against duplicate payroll
records for the same trip.

------------------------------------------------------------------------

## 10. Driver Wallet Design

### Decision

Represent driver wallet data using:

``` text
wallets
wallet_transactions
```

`wallets` stores the current balance. `wallet_transactions` stores
transaction history.

### Reason

The current balance needs to be quickly accessible, while transaction
history must be retained for financial traceability.

### Trade-off

The current balance and transaction history are maintained separately,
requiring the application to keep them synchronized.

------------------------------------------------------------------------

## 11. Append-Only Wallet Ledger

### Decision

Treat `wallet_transactions` as an append-only ledger.

### Reason

Keeping transaction history allows the team to trace how a driver's
wallet balance changed over time.

### Flow

``` text
Trip Closed
    ↓
Calculate Driver Share
    ↓
Create Payroll Record
    ↓
Create Wallet Transaction
    ↓
Update Wallet Balance
```

------------------------------------------------------------------------

## 12. Trip Assignment History

### Decision

Store driver assignment history in a separate `trip_assignments` table.

### Reason

A driver may be reassigned during a trip lifecycle. Instead of
overwriting the previous driver assignment, the database preserves
assignment history.

Stored information includes:

-   Trip
-   Driver
-   User who assigned the driver
-   Assignment time
-   Unassignment time

### Trade-off

The current assignment must be determined from the trip and assignment
records, but previous assignments remain available for auditing.

------------------------------------------------------------------------

## 13. One Order to One Trip

### Decision

Use a unique `trips.order_id` relationship.

### Reason

The current design defines:

``` text
One Order = One Trip
```

The unique constraint prevents multiple trip records from being created
for the same order.

------------------------------------------------------------------------

## 14. Trip Company Denormalization

### Decision

Keep `trips.company_id` even though the trip is already connected to an
order and the order is connected to a company.

### Reason

The current database design explicitly keeps `company_id` in `trips` for
fast queries.

### Trade-off

The company relationship exists through both the order and trip.
Application/database logic must keep the values consistent.

------------------------------------------------------------------------

## 15. Invoice Structure

### Decision

Separate monthly invoice headers and trip-level invoice data into:

``` text
invoices
invoice_items
```

### Reason

`invoices` stores monthly totals and verification information.
`invoice_items` stores the individual trips included in the invoice.

------------------------------------------------------------------------

## 16. Invoice Financial Snapshot

### Decision

Store trip-level financial values inside `invoice_items`.

These include:

-   Trip amount
-   Driver share percentage
-   Company share percentage
-   Driver share amount
-   Company share amount

### Reason

The invoice must preserve the financial values used when it was
generated.

### Trade-off

Some financial values are stored both in payroll and invoice items, but
this supports historical invoice accuracy.

------------------------------------------------------------------------

## 17. One Invoice Per Company Per Month

### Decision

Use a unique constraint on:

``` text
(company_id, period_month)
```

### Reason

The current design defines one monthly invoice for a company. This
prevents duplicate invoices for the same company and calendar month.

------------------------------------------------------------------------

## 18. Two-Person Invoice Verification

### Decision

Use two verification roles:

``` text
Verifier 1
Verifier 2
```

The invoice stores:

``` text
created_by
second_verifier_id
```

### Reason

The design requires two-person verification before publication. The same
admin must not act as both verifiers.

### Flow

``` text
Draft
  ↓
Pending Verification
  ↓
Verifier 1
  +
Verifier 2
  ↓
Verified
  ↓
Published
```

### Trade-off

Invoice publication requires coordination between two administrators,
but provides separation of verification responsibility.

------------------------------------------------------------------------

## 19. Audit Log Design

### Decision

Use a centralized `audit_logs` table.

### Reason

Important business actions need to be traceable.

The current design records events such as:

-   Payout split changes
-   Trip amount changes
-   Driver reassignment
-   Invoice verification

The log stores:

``` text
entity_type
entity_id
action
actor_id
created_at
meta_data
```

### Trade-off

A centralized audit table is flexible, but the application must
consistently create audit records for important actions.

------------------------------------------------------------------------

## 20. Notification Design

### Decision

Use a dedicated `notifications` table.

### Reason

Notification records are associated with users and contain channel,
event, payload, status, and timestamps.

The current default channel is:

``` text
email
```

------------------------------------------------------------------------

## 21. Driver Document Storage

### Decision

Store driver document metadata in `driver_documents` and keep the
document reference in `file_url`.

The current schema describes `file_url` as a private storage
key/reference.

### Reason

Driver documents are operationally related to drivers but are separated
from the driver profile.

------------------------------------------------------------------------

## 22. Driver Availability Design

### Decision

Store driver availability by date in `driver_availability` with a unique
constraint on:

``` text
(driver_id, available_date)
```

### Reason

The current design represents calendar availability as one record per
driver per day.

------------------------------------------------------------------------

## 23. Indexing Strategy

### Decision

Add indexes for common query patterns.

### Orders

``` text
(company_id, created_at)
(status)
```

### Trips

``` text
(driver_id, status)
(company_id, created_at)
(company_id, closed_at)
(status)
```

### Payroll

``` text
(company_id, closed_at)
(driver_id, closed_at)
```

### Wallet

``` text
(wallet_id, created_at)
```

### Audit

``` text
(entity_type, entity_id, created_at)
```

### Reason

These indexes support common operational queries such as company order
history, order status filtering, driver active trips, company trip
history, month-end closed-trip queries, driver payout history, wallet
transaction history, and entity audit history.

### Trade-off

Indexes improve read/query performance but add database storage and
write/update overhead.

------------------------------------------------------------------------

## 24. Duplicate Prevention Strategy

### Decision

Use database-level unique constraints for important business identities.

The current design includes unique constraints on:

``` text
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
```

### Reason

These constraints prevent duplicate records in critical parts of the
business workflow.

------------------------------------------------------------------------

## 25. Financial Data Separation

### Decision

Separate financial information into three stages.

### Stage 1 --- Trip

``` text
trips.amount
```

Stores the trip amount.

### Stage 2 --- Payroll

``` text
trip_payroll_records
```

Stores the payout calculation and frozen rates.

### Stage 3 --- Invoice

``` text
invoice_items
invoices
```

Stores trip-level invoice snapshots and monthly totals.

### Reason

This separation allows trip execution, payroll calculation, and monthly
billing to have their own records and lifecycle.

------------------------------------------------------------------------

## 26. Current Schema Items Requiring Review

The current database documentation explicitly identifies the following
items for review before final implementation.

### 26.1 `first_approved_id`

Current field:

``` text
first_approved_id timestamptz
```

The current documentation notes that the field name appears to represent
an ID while its type is a timestamp.

The documentation suggests reviewing whether the intended field should
instead be:

``` text
first_approved_at timestamptz
```

This is a review item, not a silent schema change.

### 26.2 Order Status Consistency

The current enum contains:

``` text
pending
accepted
rejected
```

while the table note describes:

``` text
received
accepted
rejected
```

These values should be made consistent before implementation.

### 26.3 Partner Payment Status Typo

The current enum contains:

``` text
pending
cmopleted
```

`cmopleted` appears to be a spelling error and should be reviewed.

### 26.4 Driver Status Casing

The enum currently uses:

``` text
Active
pending
inactive
```

while the table note uses:

``` text
pending
active
inactive
```

The casing should be made consistent.

### 26.5 Payout Split Validation

The documented business rule is:

``` text
driver_share_pct + company_share_pct = 100
```

The implementation should ensure that payout percentages cannot violate
this rule or use negative values.

### 26.6 Active Trip Assignment

The schema stores assignment history using `unassigned_at`.

Application/database logic should ensure that a trip does not have
multiple active assignments at the same time.

------------------------------------------------------------------------

## 27. Alternatives Considered

### 27.1 Separate Admin Table

**Alternative:** Create a dedicated `admins` table.

**Selected:** Use the central `users` table with `role = admin`.

**Reason:** Authentication and role identification are centralized in
one table and a separate admins table is not required by the current
design.

### 27.2 Store Driver Data Directly in Users

**Alternative:** Put all driver-specific fields directly into `users`.

**Selected:** Use `users → drivers`.

**Reason:** Driver operational information is separated from common
authentication data.

### 27.3 Store Current and Historical Payout Rates in One Table

**Alternative:** Only keep the current payout split in `companies`.

**Selected:** Use `companies`, `company_payout_history`, and
`trip_payroll_records`.

**Reason:** The design needs both company-level configuration history
and trip-level frozen financial values.

### 27.4 Store Wallet Balance Without Transactions

**Alternative:** Maintain only `wallets.balance`.

**Selected:** Maintain `wallets.balance` and `wallet_transactions`.

**Reason:** The transaction ledger provides historical wallet activity
and traceability.

### 27.5 Overwrite Driver Assignment

**Alternative:** Store only the current driver in `trips.driver_id`.

**Selected:** Keep `trips.driver_id` together with `trip_assignments`.

**Reason:** The current driver can be accessed directly while
assignment/reassignment history is preserved separately.

------------------------------------------------------------------------

## 28. Decision Principles

The database decisions follow these principles:

1.  **Centralize authentication** through one users table.
2.  **Separate role-specific operational data** from authentication
    data.
3.  **Keep company ownership explicit** on operational entities.
4.  **Preserve historical financial values** through snapshots.
5.  **Maintain wallet transaction history** rather than only a balance.
6.  **Preserve assignment history** instead of overwriting operational
    history.
7.  **Separate invoice headers from invoice items**.
8.  **Require two-person invoice verification**.
9.  **Use database constraints for duplicate prevention**.
10. **Use indexes for common operational queries**.
11. **Maintain auditability** for important business actions.
12. **Review known schema inconsistencies before final implementation**.

------------------------------------------------------------------------

## 29. Final Technical Decision Summary

The HireD database is designed around:

``` text
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

The major technical decisions are:

-   A single `users` table for authentication and roles.
-   Separate operational tables for companies and drivers.
-   Company-level payout configuration with historical tracking.
-   Trip-level payroll snapshots to preserve historical financial
    accuracy.
-   A wallet balance plus append-only transaction ledger.
-   Separate driver assignment history.
-   Monthly invoices with trip-level invoice snapshots.
-   Two-person invoice verification.
-   Centralized audit logs and notifications.
-   Database constraints and indexes for integrity and performance.

The current schema also contains explicitly identified review items.
These should be resolved before final implementation rather than being
silently changed in this document.

## 30. ER Diagram

The database design and ER diagram are maintained in DrawDB:

https://www.drawdb.app/share/nMLatUMJIR5HmSo_CppOUrzf
