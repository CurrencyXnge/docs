<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# 🎨 CurrencyEx Frontend Architecture

This document serves as the **Single Source of Truth** for the frontend engineering of the CurrencyEx platform. We support two primary clients: a **Web Dashboard** (for Staff & Admins) and a **Mobile App** (for Customers).

## 1. 🏗️ Tech Stack

| Component | Web (Admin/Teller) | Mobile (Customer) |
| :--- | :--- | :--- |
| **Framework** | React 18 (Vite) | React Native (Expo) |
| **Language** | TypeScript | TypeScript |
| **State Mgmt** | Redux Toolkit (Global) + React Query (Server) | Redux Toolkit |
| **Styling** | TailwindCSS + Headless UI | NativeWind (Tailwind for RN) |
| **Real-time** | Socket.io-client / Native WebSockets | Native WebSockets |
| **Forms** | React Hook Form + Zod | React Hook Form + Zod |

---

## 2. 🧩 High-Level Modules

The application is sliced into distinct functional modules based on user roles and features.

### 2.1. Role-Based Portals
*   **Super Admin Portal**:
    *   System Configuration (Fees, Limits).
    *   Master Rate Management (Override Live Rates).
    *   Staff Management (Create Teller/Cashier).
*   **Teller/Cashier Workstation**:
    *   Rapid Transaction Entry (Keyboard optimized).
    *   Physical Cash Drawer Reconciliation.
    *   KYC Document Scanning & Upload.
*   **Customer App**:
    *   Self-service Remittance.
    *   Prepaid Wallet Management.
    *   Live Rate Checking.

### 2.2. Feature Modules
*   **Market Data Widget**: A reusable ticker component subscribed to `wss://api.currencyex.com/rates`.
*   **Remittance Wizard**: A multi-step form (Recipient -> Amount -> Payment -> Review).
*   **Prepaid Service**: Layouts for Wallet Top-up, History, and Virtual Card display.

---

## 3. 📂 Repository Structure (Monorepo-style)

```text
/
├── apps/
│   ├── web-dashboard/       # React Admin/Teller App
│   └── mobile-app/          # React Native Customer App
├── packages/
│   ├── content/             # Shared Types/Interfaces
│   ├── ui/                  # Shared UI Kit (Buttons, Inputs)
│   └── utils/               # Shared logic (Currency formatting, Date parsing)
```

---

## 4. 📚 Detailed Design Specifications

To keep this architecture digestible, we have split the deep dives into specific files:

*   [**01_ROLES_ROUTING.md**](01_ROLES_ROUTING.md) - Authentication, Role Guards, and Route Structure for Admin/Teller/Cashier.
*   [**02_STATE_SOCKETS.md**](02_STATE_SOCKETS.md) - Handling Real-time Market Data, Redux Slices, and Optimistic UI updates.
*   [**03_REMITTANCE_FLOW.md**](03_REMITTANCE_FLOW.md) - Detailed breakdown of the Remittance Form state machine and Validation logic.
*   [**04_UI_LIBRARY.md**](04_UI_LIBRARY.md) - Design System tokens, shared component usage, and responsive behavior.

---

## 5. 🚀 Deployment Strategy

*   **Web**: Dockerized Nginx container serving static build assets. Deployed to K8s.
*   **Mobile**: OTA Updates via Expo EAS (for JS bundles) + Native builds for App Stores.
