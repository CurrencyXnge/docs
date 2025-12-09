<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# ⚡ State Management & Real-Time Sockets

We utilize **Redux Toolkit (RTK)** for global client state and custom **WebSocket hooks** for live market data.

## 1. Global Store Architecture (Redux)

The store is divided into logic slices:

| Slice | Purpose | Persisted? |
| :--- | :--- | :--- |
| `auth` | User Token, Role, Permissions | ✅ (SessionStorage) |
| `rates` | Live Market Rates (Normalized) | ❌ (Ephemeral) |
| `cart` | Current Exchange/Remit Transaction Draft | ✅ (LocalStorage) |
| `ui` | Toast notifications, Modal visibility, Theme | ❌ |

---

## 2. Live Market Data (WebSockets)

We do not store live rates involves Redux persistence to avoid stale data on reload.

### Data Flow
1.  **Connection**: Client connects to `wss://api.currencyex.com/ws`.
2.  **Subscription**: Client sends `{ "action": "subscribe", "pairs": ["USD/INR", "GBP/EUR"] }`.
3.  **Update**: Server pushes diffs or full snapshots.

### Custom Hook: `useLiveRate`

```tsx
// hooks/useLiveRate.ts
export const useLiveRate = (base: string, target: string) => {
  const rates = useSelector((state) => state.rates.data);
  const status = useSelector((state) => state.rates.connectionStatus);

  const rate = rates[`${base}/${target}`];

  // Logic to show "Stale" indicator if data is > 60s old
  const isStale = Date.now() - rate?.lastUpdated > 60000;

  return { rate: rate?.value, isStale, status };
};
```

---

## 3. Optimistic Updates

For **Prepaid Wallet** operations, we use Optimistic UI patterns to make the app feel instant.

**Scenario: Validating a Top-Up**
1.  **User Action**: Clicks "Load $100".
2.  **Frontend**:
    *   Immediately increments displayed balance state: `balance += 100`.
    *   Adds a "Pending" badge.
3.  **Network**: Sends API request `POST /wallet/topup`.
4.  **Failure**:
    *   If API fails, roll back balance: `balance -= 100`.
    *   Show Error Toast.
5.  **Success**:
    *   Remove "Pending" badge.
    *   Reconcile with actual server response if needed.

---

## 4. Symbol & Currency Formatting

We use a central utility for consistent display.

```tsx
// utils/currency.ts
export const formatMoney = (amount: number, currency: string) => {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: currency,
    minimumFractionDigits: 2
  }).format(amount);
};
```
