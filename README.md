<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# CurrencyEx App Documentation

This repository hosts the design specifications, architectural diagrams, and wireframes for **CurrencyEx** — a next-generation Currency Exchange and Remittance platform.

## 📄 Overview

The documentation provides a comprehensive view of the system, including:
- **System Architecture**: Microservices design using Go, NATS, and Kubernetes.
- **Process Flows**: Sequence diagrams for Exchange and Remittance workflows.
- **UI Wireframes**: Low-fidelity digital drafts for Admin, Customer, and Teller interfaces.
- **Commercials**: Development roadmap and estimated costs.

## 🚀 How to View

The documentation is built as a standalone HTML page with embedded CSS and Mermaid.js for diagrams.

1.  **Clone the repository** (if you haven't already):
    ```bash
    git clone <repository-url>
    ```
2.  **Open the file**:
    Simply double-click `index.html` or open it in your browser:
    ```bash
    # Linux
    xdg-open index.html
    
    # Mac
    open index.html
    
    # Windows
    start index.html
    ```

## 🛠 Technology Stack

The detailed tech stack is documented within, including:
- **Backend**: Go (Golang) 1.21+
- **Frontend**: React 18 & React Native
- **Database**: PostgreSQL & Redis
- **Messaging**: NATS JetStream

## 📂 System Architecture

The documentation is split into specialized architectural guides:

### [🏗️ Backend Architecture](backend-arch/BACKEND_MASTER_DOC.md)
*   **Microservices**: Auth, Wallet, Rate, Remittance services.
*   **Database**: PostgreSQL Schema & Redis Caching.
*   **Infrastructure**: Docker, K8s, CI/CD.

### [🎨 Frontend Architecture](frontend-arch/FRONTEND_MASTER_DOC.md)
*   **Roles**: Routing for Admin, Teller, Cashier.
*   **State**: Redux Toolkit & WebSocket Integration.
*   **Components**: Reusable UI Kit & Remittance Wizard.

## 📡 Real-time Market Data Integration

The system aggregates live exchange rates from external providers (e.g., Wise, OpenExchangeRates) and creates a normalized rate stream for the internal services.

> **[📄 Read the Detailed Market Data Integration Guide](MARKET_DATA_INTEGRATION.md)**

### Key Features
*   **Polling Engine**: Fetch rates every 30s.
*   **Normalization**: Unified data model for all providers.
*   **Distribution**: Real-time push via NATS and WebSockets.

## 💸 Remittance & Smart Routing

The platform handles global money transfers using a dual-path strategy to balance speed and cost.

> **[📄 Read the Detailed Remittance Routing & Batching Guide](REMITTANCE_BATCHING_GUIDE.md)**

### Core Concepts
1.  **Instant Path**: Direct API calls to partners (Wise, Ripple) for "Fast" tier payments.
2.  **Batch Path**: Aggregates "Saver" transactions into CSV/ISO files and uploads via SFTP every hour.
3.  **Automated Failover**: Robust retry mechanisms and status tracking.

## �📂 Structure

- `index.html` - Main documentation file containing all content and diagrams.
- `styles.css` - Stylesheet for the documentation page.
