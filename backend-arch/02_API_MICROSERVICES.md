<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# 🌐 API & Microservices Architecture

This document breaks down the communication patterns, interface definitions, and service boundaries of the CurrencyEx backend.

## 1. Service Boundaries (Domain Driven Design)

We split the monolith into **Bounded Contexts**.

| Service | Responsibility | Dependencies |
| :--- | :--- | :--- |
| **AuthSvc** | Identity, JWT, MFA, Staff ID Management | DB |
| **WalletSvc** | Ledger, Internal Transfers, Prepaid Top-up | DB |
| **RateSvc** | External fetching, Normalization, Admin Overrides | Redis, External APIs |
| **RemittanceSvc** | Cross-border payments, Beneficiary Mgmt | WalletSvc, RateSvc, Queue |
| **ExchangeSvc** | Cashier Window (Cash Buy/Sell) | WalletSvc, RateSvc |

---

## 2. API Gateway Pattern (BFF)

We use a **Backend for Frontend (BFF)** pattern. The Gateway is the **only** public-facing entry point.
- **Technology**: Golang (Gin/Fiber) or Kong/Nginx.
- **Responsibilities**:
    1.  **TLS Termination**: Handles HTTPS.
    2.  **Authentication**: Validates JWT via middleware before passing to services.
    3.  **Rate Limiting**: Protects downstream services.
    4.  **Aggregation**: Combines response from Wallet+Rate services for the Dashboard.

---

## 3. Communication Protocols

### 3.1. Synchronous (gRPC / Internal HTTP)
Used for critical operations that must succeed immediately or fail.

**Example: Cashier Exchange (Buy USD)**
1.  `ExchangeSvc` calls `RateSvc` (gRPC) to validate `rate_id`.
2.  `ExchangeSvc` calls `WalletSvc` (gRPC) to:
    - Credit `USD` to Branch Drawer.
    - Debit `AED` from Branch Drawer.
3.  If successful -> Print Receipt.

### 3.2. Asynchronous (Event Driven)
Used for eventual consistency and side effects.

**Example: Remittance Completion**
1.  `RemittanceSvc` updates DB status to `COMPLETED`.
2.  `RemittanceSvc` publishes `txn.remit.completed` to **NATS JetStream**.
3.  **Subscribers**:
    - `NotifSvc`: Sends SMS to Receiver.
    - `AuditSvc`: Logs to audit trail.
    - `DataSvc`: Updates Daily Sales Report.

---

## 4. API Endpoints List (OpenAPI Spec)

**Base URL**: `https://api.currencyex.com/v1`

### 🕵️ Identity & Access
| Method | Endpoint | Description | Auth |
| :--- | :--- | :--- | :--- |
| `POST` | `/auth/login/customer` | Email/Pass Login | ❌ |
| `POST` | `/auth/login/staff` | **Staff ID + Password** Login | ❌ |
| `POST` | `/auth/refresh` | Refresh Access Token | ✅ |

### �️ Admin / System Config (Super Admin Only)
| Method | Endpoint | Description | Auth |
| :--- | :--- | :--- | :--- |
| `POST` | `/admin/staff/create` | Onboard Teller/Cashier | ✅ |
| `GET` | `/admin/rates/config` | View current spreads | ✅ |
| `POST` | `/admin/rates/override` | Force specific rate (±2%) | ✅ |
| `PUT` | `/admin/fees/global` | Update Service Charges | ✅ |
| `GET` | `/admin/reports/pnl` | Profit & Loss Report | ✅ |

### 💱 Cashier Exchange (Spot)
| Method | Endpoint | Description | Auth |
| :--- | :--- | :--- | :--- |
| `GET` | `/exchange/rates` | Get Buy/Sell Rates | ✅ |
| `POST` | `/exchange/calculate` | Calc total with Fees | ✅ |
| `POST` | `/exchange/quote/negotiate` | **Request custom rate** | ✅ (Manager) |
| `POST` | `/exchange/execute` | Commit Cash Transaction | ✅ (Teller) |

### 🌍 Remittance
| Method | Endpoint | Description | Auth |
| :--- | :--- | :--- | :--- |
| `GET` | `/remit/beneficiaries`| List saved receivers | ✅ |
| `POST` | `/remit/quote` | Lock Standard Rate | ✅ |
| `POST` | `/remit/quote/negotiate` | **Request Special Rate** | ✅ (Manager) |
| `POST` | `/remit/send` | execute transfer | ✅ |

### 💳 Prepaid Wallet
| Method | Endpoint | Description | Auth |
| :--- | :--- | :--- | :--- |
| `GET` | `/wallet/balance` | Get multi-currency balance | ✅ |
| `POST` | `/wallet/topup` | Load funds via Gateway | ✅ |
| `POST` | `/wallet/transfer` | P2P Transfer (Internal) | ✅ |
