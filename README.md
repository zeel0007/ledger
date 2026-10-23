# 🏦 Backend Ledger — Double-Entry Financial Accounting & Transfer Engine

A high-reliability, double-entry financial accounting backend service built with **Node.js**, **Express.js**, and **MongoDB**. 

Designed around core fintech principles: **zero static balances**, **strict append-only ledger immutability**, **multi-document ACID transactions**, **request idempotency**, and **dynamic balance derivation via aggregation pipelines**.

---

## 📑 Table of Contents
- [Key Features](#-key-features)
- [System Architecture](#-system-architecture)
- [Data Flow Diagrams (DFD)](#-data-flow-diagrams-dfd)
  - [Level 0: Context Diagram](#level-0-context-diagram)
  - [Level 1: Subsystem Level](#level-1-subsystem-level)
  - [Level 2: Transaction Lifecycle](#level-2-transaction-lifecycle)
- [Entity-Relationship Diagram (ERD)](#-entity-relationship-diagram-erd)
- [The 4-Phase Transfer Lifecycle](#-the-4-phase-transfer-lifecycle)
- [Core Architectural Decisions](#-core-architectural-decisions)
- [API Reference](#-api-reference)
- [Getting Started](#-getting-started)
- [Environment Configuration](#-environment-configuration)

---

## ⚡ Key Features

- **Double-Entry Bookkeeping:** Every transaction produces equal and offsetting `DEBIT` and `CREDIT` entries. Money is never created or destroyed arbitrarily.
- **Zero Mutable Balances:** Account balances are not stored as static numbers. Balances are dynamically calculated using MongoDB aggregation pipelines (`$group`, `$cond`, `$subtract`) over immutable ledger entries.
- **Driver-Level Immutability:** Mongoose pre-query hooks intercept and reject any `updateOne`, `deleteOne`, `findOneAndUpdate`, or `deleteMany` operations on the ledger collection.
- **ACID Multi-Document Transactions:** Money transfers execute inside isolated MongoDB sessions (`mongoose.startSession()`), guaranteeing atomicity across multi-collection writes.
- **Request Idempotency:** Client-supplied unique `idempotencyKey` prevents duplicate transactions during network retries and double-clicks.
- **Stateful JWT Session Revocation with 3-Day TTL:** Logged-out tokens are stored in a MongoDB blacklist with an automated **3-day TTL index** (`expireAfterSeconds: 259200`), eliminating database bloat without external cron jobs.
- **Role-Based Access Control (RBAC):** Restricts system-level initial fund seeding (`/api/transactions/system/initial-funds`) to authenticated `systemUser` accounts.
- **Asynchronous Notifications:** Non-blocking email alerts dispatched via Nodemailer with Google OAuth2.

---

## 🏛️ System Architecture

```mermaid
flowchart TD
    Client(["📱 Web / Mobile Client"])

    subgraph Entry ["1. Entry & Global Pipeline"]
        Server["server.js (Process Bootstrap & DB Init)"]
        App["app.js (Express Application)"]
        GlobalMW["Global Middlewares (express.json, cookie-parser)"]
    end

    subgraph Security ["2. Security & Guard Layer"]
        AuthMW["auth.middleware.js (JWT Verification)"]
        TTLBlacklist[("Token Blacklist (TTL Index 3 Days)")]
    end

    subgraph Routing ["3. Routing Layer"]
        AuthRoutes["/api/auth"]
        AccountRoutes["/api/accounts"]
        TxnRoutes["/api/transactions"]
    end

    subgraph Controllers ["4. Business Logic & Controllers"]
        AuthController["auth.controller.js"]
        AccountController["account.controller.js"]
        TxnController["transaction.controller.js (Transfer Engine)"]
    end

    subgraph Models ["5. Data Layer & Defensive Guards"]
        UserModel["user.model.js (bcrypt hash)"]
        AccountModel["account.model.js (Aggregation Pipeline)"]
        TxnModel["transaction.model.js (Idempotency Key)"]
        LedgerModel["ledger.model.js (Immutable Pre-Hooks)"]
    end

    subgraph DB ["6. Persistence Engine (MongoDB)"]
        MongoSession["ACID Transaction Session (startSession)"]
        DBCollections[("MongoDB Collections")]
    end

    subgraph Notifications ["7. External Services"]
        EmailService["email.service.js (Nodemailer OAuth2)"]
    end

    Client -->|HTTP Request| Server
    Server --> App
    App --> GlobalMW
    GlobalMW --> AuthMW
    AuthMW -. Checks Revocation .-> TTLBlacklist
    AuthMW --> Routing

    Routing --> AuthRoutes & AccountRoutes & TxnRoutes
    AuthRoutes --> AuthController
    AccountRoutes --> AccountController
    TxnRoutes --> TxnController

    AuthController --> UserModel
    AccountController --> AccountModel
    TxnController --> TxnModel & LedgerModel & AccountModel

    TxnController --> MongoSession
    MongoSession --> DBCollections
    TxnController -. Async Background .-> EmailService
```

---

## 🔄 Data Flow Diagrams (DFD)

### Level 0: Context Diagram
```mermaid
flowchart LR
    User(["👤 Normal User"])
    SysAdmin(["🛡️ System Admin"])
    System[["0.0<br/>Backend Ledger<br/>System"]]
    EmailGateway(["📧 Email Gateway<br/>(Gmail OAuth2)"])

    User -->|"1. Register / Login Credentials"| System
    User -->|"2. Create Account Request"| System
    User -->|"3. Query Account Balance"| System
    User -->|"4. Transfer Request (from, to, amount, key)"| System

    System -->|"Auth Token & Profile"| User
    System -->|"Account Details & Derived Balance"| User
    System -->|"Transaction Receipt"| User

    SysAdmin -->|"Seed Initial Funds Request"| System
    System -->|"Funding Confirmation"| SysAdmin

    System -->|"Async Email Trigger"| EmailGateway
```

---


### Level 2: Transaction Lifecycle
```mermaid
flowchart TD
    User(["👤 Authenticated User"])

    subgraph Process3 ["3.0 Money Transfer Execution"]
        P3_1(("3.1<br/>Validate Payload &<br/>Idempotency Key"))
        P3_2(("3.2<br/>Verify Account<br/>Status (ACTIVE)"))
        P3_3(("3.3<br/>Aggregate Sender<br/>Balance ($sum Credits - Debits)"))
        P3_4(("3.4<br/>Start MongoDB ACID Session<br/>Write PENDING Txn"))
        P3_5(("3.5<br/>Write Immutable<br/>DEBIT & CREDIT Entries"))
        P3_6(("3.6<br/>Commit ACID Session &<br/>Mark COMPLETED"))
    end

    D3[("D3: Accounts Store")]
    D4[("D4: Transactions Store")]
    D5[("D5: Immutable Ledger Store")]
    P4(("4.0<br/>Notification Dispatcher"))

    User -->|"fromAccount, toAccount, amount, idempotencyKey"| P3_1

    P3_1 <-->|"Check Existing Key"| D4
    P3_1 -->|"Key Valid & New"| P3_2

    P3_2 <-->|"Read Account Status"| D3
    P3_2 -->|"Both ACTIVE"| P3_3

    P3_3 <-->|"Run Aggregation Pipeline"| D5
    P3_3 -->|"Balance >= Amount"| P3_4

    P3_4 -->|"Insert PENDING Txn (Session)"| D4
    P3_4 -->|"Session Active"| P3_5

    P3_5 -->|"Insert DEBIT (Sender) & CREDIT (Receiver)"| D5
    P3_5 -->|"Entries Created"| P3_6

    P3_6 -->|"Update Status = COMPLETED (Commit Session)"| D4
    P3_6 -->|"201 Created Response"| User
    P3_6 -->|"Async Event"| P4
```

---

## 🗄️ Entity-Relationship Diagram (ERD)

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : owns
    USER ||--o{ TOKEN_BLACKLIST : invalidates
    ACCOUNT ||--o{ LEDGER : "has entries"
    TRANSACTION ||--|{ LEDGER : produces

    USER {
        ObjectId _id PK
        string email UK
        string name
        string password
        boolean systemUser
        date createdAt
        date updatedAt
    }

    ACCOUNT {
        ObjectId _id PK
        ObjectId user FK
        string status "ACTIVE | FROZEN | CLOSED"
        string currency "INR"
        date createdAt
        date updatedAt
    }

    TRANSACTION {
        ObjectId _id PK
        ObjectId fromAccount FK
        ObjectId toAccount FK
        number amount
        string status "PENDING | COMPLETED | FAILED | REVERSED"
        string idempotencyKey UK
        date createdAt
        date updatedAt
    }

    LEDGER {
        ObjectId _id PK
        ObjectId account FK
        ObjectId transaction FK
        number amount
        string type "CREDIT | DEBIT"
    }

    TOKEN_BLACKLIST {
        ObjectId _id PK
        string token UK
        date createdAt "TTL: 3 Days"
    }
```

---

## ⏱️ The 4-Phase Transfer Lifecycle

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant Auth as Auth Middleware
    participant TxnCtrl as Transaction Controller
    participant AcctModel as Account Model
    participant DB as MongoDB (ACID Session)
    participant Email as Email Service

    Note over Client,Email: Phase 1: Pre-Flight Guards & Idempotency
    Client->>Auth: POST /api/transactions (Bearer Token / Cookie)
    Auth->>DB: Check Token in Blacklist
    DB-->>Auth: Token valid & active
    Auth->>TxnCtrl: Forward Request
    TxnCtrl->>DB: Check idempotencyKey
    DB-->>TxnCtrl: Key is unique (new transaction)
    TxnCtrl->>DB: Verify fromAccount & toAccount status == "ACTIVE"
    DB-->>TxnCtrl: Accounts valid

    Note over Client,Email: Phase 2: Dynamic Balance Derivation
    TxnCtrl->>AcctModel: getBalance(fromAccount)
    AcctModel->>DB: Aggregate: sum(CREDIT) - sum(DEBIT)
    DB-->>AcctModel: Derived Balance
    AcctModel-->>TxnCtrl: Balance >= Amount

    Note over Client,Email: Phase 3: The Atomic ACID Commit
    TxnCtrl->>DB: mongoose.startSession() & session.startTransaction()
    TxnCtrl->>DB: Create Transaction (status: "PENDING")
    TxnCtrl->>DB: Create DEBIT Ledger Entry (fromAccount)
    TxnCtrl->>DB: Create CREDIT Ledger Entry (toAccount)
    TxnCtrl->>DB: Update Transaction (status: "COMPLETED")
    TxnCtrl->>DB: session.commitTransaction()

    Note over Client,Email: Phase 4: Side Effects & Response
    TxnCtrl->>Email: sendTransactionEmail() (Non-blocking async)
    TxnCtrl-->>Client: 201 Created (Transaction payload)
```

---

## 🧠 Core Architectural Decisions

### 1. Dynamic Balance Derivation vs. Static Balance Column
* **Problem:** Storing `balance: Number` on an Account document causes concurrency race conditions, phantom overwrites, and lacks an audit trail.
* **Solution:** Account balances are dynamically aggregated from ledger entries on demand:
  $$\text{Balance} = \sum \text{CREDIT amounts} - \sum \text{DEBIT amounts}$$
  Implemented via a native MongoDB Aggregation pipeline (`$match`, `$group`, `$cond`, `$project`).

### 2. Append-Only Immutability via Mongoose Pre-Hooks
* **Problem:** Accidental developer updates or rogue scripts modifying past entries ruin financial integrity.
* **Solution:** [`ledger.model.js`](src/models/ledger.model.js) attaches Mongoose pre-hooks (`findOneAndUpdate`, `updateOne`, `deleteOne`, `updateMany`, etc.) that throw runtime exceptions on any mutation or deletion attempt.

### 3. Request Idempotency
* **Problem:** Mobile or web retries due to packet drops can double-charge users.
* **Solution:** Transfers require an `idempotencyKey` unique index. If a request is received with an existing key, the controller short-circuits and returns the existing transaction state rather than re-executing.

### 4. Automated 3-Day TTL Token Blacklisting
* **Problem:** Revoking stateless JWTs on logout without database bloat.
* **Solution:** Revoked tokens are saved to [`tokenBlackList`](src/models/blackList.model.js) with a MongoDB TTL index:
  ```javascript
  tokenBlacklistSchema.index({ createdAt: 1 }, { expireAfterSeconds: 60 * 60 * 24 * 3 });
  ```
  MongoDB's internal 60-second background thread automatically sweeps records older than 3 days, perfectly aligned with the JWT's cryptographic expiration.

---

## 📡 API Reference

### 🔐 Authentication (`/api/auth`)
| Method | Endpoint | Auth | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/auth/register` | Public | Register new user, issue JWT cookie & token payload |
| `POST` | `/api/auth/login` | Public | Authenticate user credentials & issue JWT |
| `POST` | `/api/auth/logout` | User | Blacklist active JWT into DB with 3-day TTL & clear cookie |

### 💳 Accounts (`/api/accounts`)
| Method | Endpoint | Auth | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/accounts/` | User | Create a new financial account for authenticated user |
| `GET` | `/api/accounts/` | User | Fetch all accounts belonging to the user |
| `GET` | `/api/accounts/balance/:accountId` | User | Dynamically calculate and return real-time ledger balance |

### 💸 Transactions (`/api/transactions`)
| Method | Endpoint | Auth | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/transactions/` | User | Transfer funds between accounts (ACID session, double-entry) |
| `POST` | `/api/transactions/system/initial-funds` | **System User** | Seed initial funds from system master account to a target account |

---


