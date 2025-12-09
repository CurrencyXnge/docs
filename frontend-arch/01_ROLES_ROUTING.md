<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# 🛡️ Roles, Routing & Permissions (Frontend)

The frontend uses a **Route Guard** pattern to ensure users only access authorized modules.

## 1. User Roles Definition

| Role | Access Level | Primary Dashboard |
| :--- | :--- | :--- |
| **SUPER_ADMIN** | Full System Access | `/admin/analytics` |
| **BRANCH_MANAGER**| Branch Reports, Override Approvals | `/branch/overview` |
| **TELLER** | Exchange Execution, Cash Handling | `/teller/pos` |
| **CASHIER** | Remittance Processing, KYC Entry | `/cashier/remit` |
| **CUSTOMER** | Self-Service (Mobile Only) | `/app/home` |

---

## 2. Route Structure (Web Dashboard)

```tsx
<Routes>
  {/* Public Routes */}
  <Route path="/login" element={<LoginPage />} />
  
  {/* Protected Routes */}
  <Route element={<AuthGuard />}>
    
    {/* Shared Staff Routes */}
    <Route path="/profile" element={<UserProfile />} />

    {/* Admin Routes */}
    <Route element={<RoleGuard allowed={['SUPER_ADMIN']} />}>
      <Route path="/admin/rates" element={<RateManager />} />
      <Route path="/admin/staff" element={<StaffManager />} />
    </Route>

    {/* Operations Routes (Teller + Cashier) */}
    <Route element={<RoleGuard allowed={['TELLER', 'CASHIER']} />}>
      <Route path="/pos/exchange" element={<ExchangePOS />} />
      <Route path="/pos/remit" element={<RemittanceForm />} />
    </Route>

  </Route>
</Routes>
```

---

## 3. AuthGuard Logic

The `AuthGuard` checks for a valid JWT in `localStorage` or `Cookie`.
1.  **Check Token**: If missing -> Redirect to `/login`.
2.  **Verify Expiry**: If expired -> Attempt Silent Refresh -> Fail to `/login`.
3.  **Fetch Profile**: Load User Role & Permissions into Redux Store.

## 4. Workstation Customization

Different roles see different "Home Screens" upon login:

*   **Super Admin**: Sees a BI Dashboard with Global Volume, P&L, and System Health.
*   **Teller**: Sees a **POS Interface** (Point of Sale) optimized for speed.
    *   Big Buttons for "Buy" / "Sell".
    *   Keyboard Shortcuts (F1 for USD, F2 for EUR).
    *   Cash Drawer Status indicator.
*   **Cashier**: Sees a Queue Management list of pending Remittance requests.

---

## 5. UI Elements Permissioning

We use a custom hook `usePermission()` to toggle specific buttons.

```tsx
const SubmitButton = () => {
  const { can } = usePermission();

  // Only Managers can approve transactions > $10k
  if (amount > 10000 && !can('APPROVE_HIGH_VALUE')) {
     return <Button disabled>Requires Approval</Button>;
  }

  return <Button onClick={submit}>Submit</Button>;
};
```
