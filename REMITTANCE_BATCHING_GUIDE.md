# 💸 Remittance Routing & Batch Processing Guide

This document outlines the technical implementation of the **Smart Routing Engine** and **Batch Processing System** used in CurrencyEx to optimize remittance delivery usage.

## 1. System Overview

The Remittance Service (`RemitSvc`) employs a **Dual-Path Routing Strategy**:
1.  **Instant Path (API)**: For high-priority transactions (Premium/Fast), routed directly to API partners (e.g., Wise, Ripple) for real-time settlement.
2.  **Batch Path (File)**: For standard transactions (Saver/Normal), queued and aggregated into bulk files (CSV/ISO 20022) to save on transaction fees.

### Architecture Diagram

```mermaid
graph TD
    Request[User Request] --> RemitSvc
    RemitSvc --> Router{Routing Logic}
    
    %% Instant Path
    Router -->|Tier=FAST| InstantAdapter[Instant API Adapter]
    InstantAdapter -->|HTTP POST| ExtAPI[External Provider API]
    ExtAPI -->|200 OK| Completed((Completed))

    %% Batch Path
    Router -->|Tier=NORMAL| DB[(DB: Pending Queue)]
    
    subgraph Cron_Job [Batch Processor (Every 1 Hour)]
        Job[Ticker] --> Fetch[Fetch Pending Txns]
        Fetch --> Group[Group by Currency]
        Group --> Gen[Generate CSV/XML]
        Gen --> Upload[SFTP Upload]
    end

    Upload --> Batched((Status: BATCHED))
```

## 2. Database Schema

Key tables involved in the batching process.

### `transactions` Table
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | Unique Transaction ID |
| `user_id` | UUID | Customer ID |
| `amount` | DECIMAL | Sending Amount |
| `currency` | VARCHAR(3) | Target Currency (e.g., INR) |
| `tier` | ENUM | `'FAST'` or `'NORMAL'` |
| `status` | ENUM | `'PENDING'`, `'PROCESSING'`, `'BATCHED'`, `'COMPLETED'` |
| `batch_id` | UUID | Nullable. FK to `batches` table |

### `batches` Table
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | Unique Batch ID |
| `file_name` | VARCHAR | Generated filename (e.g., `BATCH_INR_20241024.csv`) |
| `total_count`| INT | Number of transactions included |
| `total_amt` | DECIMAL | Total sum of the batch |
| `status` | ENUM | `'CREATED'`, `'UPLOADED'`, `'ACKNOWLEDGED'` |

## 3. Batch Processing Logic (Developer Guide)

The batch processor is a background worker in Go that runs on a schedule.

### 3.1. The Batch Job Loop
Found in `internal/jobs/batch_processor.go`.

```go
func (j *BatchJob) Run() {
    // 1. Fetch all pending 'NORMAL' transactions older than 5 mins
    txns, err := j.Repo.GetPendingTransactions("NORMAL")
    if len(txns) == 0 {
        return
    }

    // 2. Group by Target Currency
    grouped := GroupByCurrency(txns) // map[string][]Transaction

    // 3. Process each group
    for currency, groupTxns := range grouped {
        // A. Generate File Content
        fileContent := GenerateCSV(groupTxns)
        fileName := fmt.Sprintf("BATCH_%s_%s.csv", currency, time.Now().Format("200601021504"))

        // B. Upload to SFTP
        err := j.SFTPClient.Upload(fileName, fileContent)
        
        // C. Update Database
        if err == nil {
            batchID := j.Repo.CreateBatchRecord(fileName, len(groupTxns))
            j.Repo.MarkTransactionsAsBatched(groupTxns, batchID)
        }
    }
}
```

### 3.2. File Generation Strategy
*   **Format**: The system supports pluggable formatters (CSV, MT103, ISO 20022).
*   **CSV Example**:
    ```csv
    Reference,Amount,Currency,BeneficiaryName,IBAN,BankCode
    TXN-1001,500.00,EUR,John Doe,DE893704...,GENO...
    TXN-1002,1200.50,EUR,Alice Smith,FR763000...,PARI...
    ```

## 4. Routing Configuration

Routing rules are defined in `config/routing_rules.yaml`. You can change thresholds dynamically.

```yaml
routing:
  default_tier: "NORMAL"
  
  # Force API routing if amount > threshold (High Value Priority)
  auto_upgrade_threshold: 
    USD: 10000.00
    GBP: 5000.00

  # Batch settings
  batch_interval: "60m" # Run every hour
  min_batch_size: 5     # Wait for at least 5 txns before creating batch
```

## 5. Error Handling & Retry

1.  **Instant Path Failure**:
    *   If the API call fails, the transaction is strictly Rolled Back.
    *   User receives an immediate error.
2.  **Batch Upload Failure**:
    *   If SFTP upload fails, the batch is NOT marked as 'UPLOADED'.
    *   Transactions remain 'PENDING'.
    *   The job retries in the next interval (Idempotency key ensures no double processing).

---
*For further assistance, refer to the `pkg/remittance` package documentation.*
