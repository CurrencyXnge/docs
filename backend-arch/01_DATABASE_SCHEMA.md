<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# 🗄️ Database Design & Schema Architecture

This document details the database strategy for CurrencyEx, using **PostgreSQL** as the primary relational store and **Redis** for caching and high-speed data.

## 1. Design & Normalization Strategy

We follow **3NF (Third Normal Form)** for critical transactional data to ensure integrity.
- **Foreign Keys**: Enforced strictly to prevent orphaned records.
- **UUIDs**: Used for Primary Keys to allow easy data migration and prevent ID enumeration attacks.
- **Audit Columns**: `created_at`, `updated_at` on every table.
- **Optimistic Locking**: `version` column on `wallets` and `users` to handle concurrency.

---

## 2. Core Tables Schema

### 2.1. Authentication & Users (`users`)
Stores identity, role, and KYC status. 
**Note**: `login_id` is the primary handle. For Customers, it matches `email`. For Staff, it is an Admin-generated ID (e.g., `TELLER-8821`).

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK, Defaults to UUIDv4 | Unique User ID |
| `login_id` | VARCHAR(100)| UNIQUE, NOT NULL | **Auth Identifier** (Email or Staff ID) |
| `email` | VARCHAR(255) | Index | Contact Email (Optional for Staff) |
| `password_hash` | VARCHAR | NOT NULL | Argon2id Hash |
| `full_name` | VARCHAR(100)| NOT NULL | Legal Name (KYC) |
| `role` | VARCHAR(50) | DEFAULT 'CUSTOMER' | `SUPER_ADMIN`, `TELLER`, `CASHIER`, `CUSTOMER` |
| `kyc_status` | VARCHAR(20) | DEFAULT 'PENDING' | `PENDING`, `APPROVED` |
| `branch_id` | UUID | NULLABLE | **Staff Only**: Assigned Branch |
| `created_by` | UUID | FK -> Users | **Staff Only**: ID of Admin who created this user |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | - |

### 2.1.1. Permissions Policy (`casbin_rule`)
Stores dynamic RBAC policies loaded by the Casbin Enforcer.

| Column | Type | Index | Description |
| :--- | :--- | :--- | :--- |
| `id` | SERIAL | PK | Auto-inc |
| `ptype` | VARCHAR(100) | - | Policy Type (usually 'p') |
| `v0` | VARCHAR(100) | Index | Role (Subject) e.g., 'TELLER' |
| `v1` | VARCHAR(100) | - | URL/Resource (Object) e.g., '/api/v1/exchange' |
| `v2` | VARCHAR(100) | - | Method (Action) e.g., 'POST' |
| `v3` | VARCHAR(100) | - | Tenant/Branch (Optional) |
| `v4` | VARCHAR(100) | - | - |
| `v5` | VARCHAR(100) | - | - |

**Example Policies**:
*   `p, TELLER, /api/v1/exchange, execute` (Can swap)
*   `p, BRANCH_MANAGER, /api/v1/negotiate, execute` (Can override rate > 2%)
*   `p, TELLER, /api/v1/negotiate, execute` (Can discount fee only)

### 2.2. Financial Ledger (`wallets`)
Supports the **Prepaid Wallet** feature.
Double-entry accounting is enforced via application logic.

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK | Wallet ID |
| `user_id` | UUID | FK -> Users | Owner |
| `currency` | CHAR(3) | NOT NULL | ISO 4217 (USD, INR, GBP, AED) |
| `balance` | DECIMAL(20,8) | DEFAULT 0 | Available Funds |
| `locked` | DECIMAL(20,8) | DEFAULT 0 | Funds in pending txns |
| `last_txn_id` | UUID | NULLABLE | Idempotency Check |

**Constraint**: `UNIQUE(user_id, currency)` - A user can only have one wallet per currency.

### 2.3. Exchange Transactions (`exchange_txns`)
Records "Cashier/Teller Window" operations (Buy/Sell Currency).

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK | Transaction ID |
| `reference` | VARCHAR(20) | UNIQUE | `EX-9925` |
| `teller_id` | UUID | FK -> Users | Whos processed it |
| `customer_id` | UUID | FK -> Users | Who requested it |
| `src_currency` | CHAR(3) | NOT NULL | e.g. USD |
| `src_amount` | DECIMAL(20,8) | NOT NULL | 500.00 |
| `target_currency`| CHAR(3) | NOT NULL | e.g. AED |
| `target_amount`| DECIMAL(20,8) | NOT NULL | 1840.00 |
| `rate` | DECIMAL(12,6) | NOT NULL | Standard System Rate |
| `negotiated_rate`| DECIMAL(12,6)| NULLABLE | **Teller Override Rate** |
| `fee` | DECIMAL(10,2) | DEFAULT 0 | Service Charge |
| `status` | VARCHAR(20) | Index | `COMPLETED`, `CANCELLED` |

### 2.4. Remittance Transactions (`remit_txns`)
Records "International Remittance" flows.

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK | Transaction ID |
| `reference` | VARCHAR(20) | UNIQUE | `TXN-9928` |
| `sender_id` | UUID | FK -> Users | Customer |
| `beneficiary_id`| UUID | FK -> Beneficiaries| Who receives money |
| `send_amount` | DECIMAL(20,8) | NOT NULL | 2000.00 GBP |
| `recv_amount` | DECIMAL(20,8) | NOT NULL | 9300.00 AED |
| `rate` | DECIMAL(12,6) | NOT NULL | Standard Rate |
| `negotiated_rate`| DECIMAL(12,6)| NULLABLE | **Cashier Override Rate** |
| `fee` | DECIMAL(10,2) | NOT NULL | 15.00 GBP |
| `total_payable` | DECIMAL(20,8) | NOT NULL | 2015.00 GBP |
| `status` | VARCHAR(20) | Index | `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED` |
| `payout_provider`| VARCHAR(50) | - | `WISE`, `RIPPLE`, `SWIFT` |
| `provider_ref` | VARCHAR(100)| - | External ID for tracking |

### 2.5. Beneficiaries (`beneficiaries`)
Saved recipients for quicker remittance.

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK | - |
| `user_id` | UUID | FK -> Users | Who added this beneficiary |
| `full_name` | VARCHAR(100)| NOT NULL | Receiver Name |
| `bank_name` | VARCHAR(100)| NOT NULL | e.g. Barclays, Emirates NBD |
| `account_number`| VARCHAR(50) | NOT NULL | IBAN / Acct No |
| `swift_code` | VARCHAR(20) | - | SWIFT / Sort Code |
| `relationship` | VARCHAR(50) | - | Family, Friend, Business |

### 2.6. Audit Log (`audit_logs`)
For "Transaction / Audit Log" compliance view.

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK | - |
| `user_id` | UUID | FK -> Users | Actor (Admin/Teller/User) |
| `action` | VARCHAR(50) | NOT NULL | `LOGIN`, `CREATE_TXN`, `Override_Rate` |
| `entity_id` | UUID | Index | ID of Txn or User modified |
| `details` | JSONB | - | Snapshot of change `{"old": 3.65, "new": 3.68}` |
| `ip_address` | VARCHAR(45) | - | Security tracking |
| `created_at` | TIMESTAMPTZ | Index | Timestamp |

### 2.7. Branches & Cash Management
Specific tables for physical store operations ("Teller" & "Cashier").

#### `branches`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | PK |
| `name` | VARCHAR(100) | "Downtown Branch" |
| `location` | VARCHAR(255) | Full Address |
| `manager_id` | UUID | FK -> Users (Role: BRANCH_MANAGER) |
| `base_currency` | CHAR(3) | Defaults to local currency (e.g. AED) |

#### `cash_drawers`
Tracks physical cash assigned to a specific Teller/Counter.

| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | PK |
| `branch_id` | UUID | FK |
| `teller_id` | UUID | FK -> Users | Who is responsible |
| `currency` | CHAR(3) | USD, GBP, etc. |
| `balance` | DECIMAL | Physical cash amount in drawer |
| `status` | VARCHAR | `OPEN`, `CLOSED`, `LOCKED` |
| `last_audit_at` | TIMESTAMP | Last time manager verified count |

### 2.8. System Configuration & Rates
Stores dynamic business rules like Spreads and Fees.

#### `spread_configs`
Defines the profit margin added on top of the interbank rate.

| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | SERIAL | PK |
| `pair` | VARCHAR(7) | `USD/AED` |
| `base_spread` | DECIMAL | Standard Margin (e.g. 0.02 = 2%) |
| `min_spread` | DECIMAL | Floor limit (Safety) |
| `max_spread` | DECIMAL | Ceiling limit (Safety) |
| `is_active` | BOOLEAN | Toggle pair availability |

#### `rate_overrides`
Allows Admins to force a specific rate for a window of time.

| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | SERIAL | PK |
| `pair` | VARCHAR(7) | `GBP/USD` |
| `rate_value` | DECIMAL | The forced rate (e.g. 1.28) |
| `active_from` | TIMESTAMP | Start time |
| `active_until`| TIMESTAMP | Auto-expire time |
| `created_by` | UUID | Admin who set this |

---

## 3. Redis Caching Strategy

We use Redis to offload read-heavy operations.

### Key Namespacing pattern
- **Rates**: `rate:latest:{base}:{target}` (TTL: 60s)
- **Session**: `session:{user_id}:{device_id}` (TTL: 24h)
- **OTP**: `otp:{email}:{code}` (TTL: 5m)
- **Idempotency**: `idemp:{req_hash}` (TTL: 24h)

---

## 4. Migration Strategy (Golang Migrate)

We use `golang-migrate` to handle schema versioning.
Files are stored in `internal/adapter/repository/migrations/`.

**Example: 000001_init_schema.up.sql**
```sql
BEGIN;

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (...);

COMMIT;
```

## 5. Backup, Recovery & Archival Strategy

We employ a multi-tier strategy to ensure data safety and compliance (7-10 years retention).

### 5.1. Daily Backups (Disaster Recovery)
*   **Tool**: `pg_dump` (for logical) / WAL-G (for physical).
*   **Frequency**: Automated daily snapshot at 03:00 AM.
*   **Storage**: Encrypted S3 Bucket (Glacier Deep Archive).
*   **Retention**: Keep daily for 30 days, monthly for 1 year.

### 5.2. Point-in-Time Recovery (PITR)
*   **Mechanism**: Postgres WAL (Write Ahead Log) archiving.
*   **Capability**: Restore database to *any second* in the last 7 days.
*   **Use Case**: Accidental table drop or catastrophic data corruption.

### 5.3. Cold Data Archival (Long-Term)
As transaction tables grow (millions of rows), query performance can degrade.
*   **Strategy**: "Partitioning" by Date (e.g., `remit_txns_2024`, `remit_txns_2025`).
*   **Archival Job**: A yearly Cron job moves data older than 2 years to a "Cold Storage" Database or Data Warehouse (Snowflake/BigQuery).
*   **Schema Update**:
    ```sql
    -- Example Table Partitioning
    CREATE TABLE remit_txns (
        ...
    ) PARTITION BY RANGE (created_at);
    
    CREATE TABLE remit_txns_2024 PARTITION OF remit_txns
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
    ```
