<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# 💸 Remittance & Transaction Flow (Frontend)

The Remittance module is a complex, multi-step wizard designed to ensure accuracy and compliance.

## 1. The Wizard Steps

We use a **State Machine** pattern to manage the wizard navigation.

1.  **Sender/Customer Selection**: Search existing customer or Quick Register (KYC Lite).
2.  **Recipient Selection**: Choose saved beneficiary or Add New (validates IBAN/Swift).
3.  **Quote & Amount**:
    *   Enter Sending Amount (e.g., 1000 GBP).
    *   System fetches live rate.
    *   **Lock Rate**: Limits the user to complete step in 60s.
4.  **Compliance Check**: Source of Funds declaration (if > threshold).
5.  **Review & Confirm**: Final summary screen.
6.  **Receipt**: Print thermal receipt / Email PDF.

---

## 2. Form Architecture (React Hook Form)

We use a single large form context `FormProvider` wrapping the wizard steps.

```tsx
// schemas/remittance.ts (Zod Validation)
const RemittanceSchema = z.object({
  senderId: z.string().uuid(),
  recipient: z.object({
    name: z.string().min(2),
    bankName: z.string(),
    accountNumber: z.string().regex(/^[A-Z0-9]+$/), // IBAN validation
    swiftCode: z.string().length(8).or(z.string().length(11)),
  }),
  amount: z.number().min(10).max(50000), // Daily Limit
  sourceOfFunds: z.string().optional(),
});
```

## 3. Rate Locking Mechanism

To prevent "Slippage" (rate changing while the teller is typing), we implement a **Quote Timer**.

*   **UI**: A progress bar shrinking from 60s to 0s.
*   **Logic**:
    *   When the user enters an amount, we call `POST /remit/quote`.
    *   The API returns a `quote_id` and `expiry_at`.
    *   If the timer hits 0, the "Next" button is disabled.
    *   User must click "Refresh Rate" to get a new quote.

---

## 4. Smart Receipt Printing

For Tellers/Cashiers, the frontend integrates with thermal printers via the browser print API or a local bridge.

**Receipt Layout**:
```text
--------------------------------
      CURRENCYEX EXCHANGE
--------------------------------
Date: 2024-10-24 10:30
Txn:  TX-8821
--------------------------------
Sender:   John Doe
Receiver: Jane Smith
Bank:     Barclays UK
Acct:     **** 1234
--------------------------------
Sent:     1,000.00 AED
Rate:     0.2150
Fees:     15.00 AED
Total:    1,015.00 AED
--------------------------------
Recv:     215.00 GBP
--------------------------------
       Powered by CurrencyEx
--------------------------------
```
