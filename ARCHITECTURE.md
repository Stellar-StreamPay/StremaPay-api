# StreamPay — Architecture Overview

This document describes the system design of StreamPay: why it is structured the way it is, how the three repositories relate to each other, how data moves through the stack, and the key technical decisions made at each layer.

It is intended for contributors who want to understand the full picture before working on any individual part of the system, and for maintainers making architectural decisions as the project grows.

---

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [System Overview](#system-overview)
- [Repository Responsibilities](#repository-responsibilities)
  - [streampay-app — Frontend](#streampay-app--frontend)
  - [streampay-api — Backend](#streampay-api--backend)
  - [streampay-contracts — Smart Contracts](#streampay-contracts--smart-contracts)
- [Data Flow](#data-flow)
  - [Simple Batch Payment](#simple-batch-payment)
  - [Contract-Mediated Payment](#contract-mediated-payment)
- [Shared Data Schema](#shared-data-schema)
- [Stellar-Specific Design Decisions](#stellar-specific-design-decisions)
  - [Horizon API vs Soroban](#horizon-api-vs-soroban)
  - [Transaction Batching](#transaction-batching)
  - [Memo Handling](#memo-handling)
  - [Asset Representation](#asset-representation)
- [Inter-Repo Integration Points](#inter-repo-integration-points)
- [Local Development Topology](#local-development-topology)
- [Security Considerations](#security-considerations)
- [Future Architecture Considerations](#future-architecture-considerations)

---

## Design Philosophy

StreamPay is built around three principles:

**Separation of concerns.** The frontend, backend, and contracts each have a single, clearly scoped job. The frontend is for humans — it is the interface through which operators build, review, and dispatch payment batches. The API is for orchestration — it handles the complexity of talking to the Stellar network safely and reliably. The contracts are for trust — they enforce rules on-chain that neither the frontend nor the API can override or circumvent.

Keeping these concerns in separate repositories enforces this separation in practice. A frontend developer should never need to understand Soroban to build a UI feature. A contract developer should never need to understand Next.js routing to write disbursement logic.

**Stellar-native design.** StreamPay is not a generic payment abstraction layer that happens to support Stellar. It is designed specifically around Stellar's primitives — operations, transaction envelopes, memos, trustlines, and Soroban contracts. This means the data model, validation logic, and batching strategy all reflect how Stellar actually works, rather than mapping a generic concept onto it.

**Auditable by default.** Every payment batch passes through a structured review step before anything touches the network. The system is designed so that operators can see exactly what will happen — to whom, how much, in which asset, with what memo — before committing. This is especially important for treasury operations and payroll where errors are costly and visible.

---

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                       streampay-app                         │
│            Next.js · TypeScript · Tailwind CSS              │
│                                                             │
│   Recipient Grid  ──►  Bulk-Edit Bar  ──►  Batch Review     │
│        ▲                                        │           │
│        │  CSV Import / Export                   │           │
│        └────────────────────────────────────────┘           │
└─────────────────────────────┬───────────────────────────────┘
                              │  REST / JSON  (HTTP)
                              │  POST /api/batches
┌─────────────────────────────▼───────────────────────────────┐
│                       streampay-api                         │
│              Node.js · TypeScript · Express                 │
│                                                             │
│   Validation  ──►  TX Construction  ──►  Signing            │
│        │                                    │               │
│   Job Queue (large batches)          Fee Bump Management    │
└──────────────┬──────────────────────────────┬───────────────┘
               │  Stellar Horizon API          │  Soroban RPC
               │  (simple payments)            │  (contract calls)
┌──────────────▼──────────────┐   ┌────────────▼──────────────┐
│     Stellar Network         │   │    streampay-contracts     │
│     (Testnet / Mainnet)     │   │    Rust · Soroban SDK      │
│                             │   │                            │
│  Payment operations         │   │  Disbursement logic        │
│  Account management         │   │  Escrow primitives         │
│  Trustline checks           │   │  Multi-sig authorization   │
└─────────────────────────────┘   └────────────────────────────┘
```

---

## Repository Responsibilities

### `streampay-app` — Frontend

**What it owns:** Everything the operator sees and touches. The recipient grid, bulk-edit utility bar, CSV import/export, batch review screen, and transaction status tracking.

**What it does not own:** Any logic that touches the Stellar network directly. The frontend never constructs transaction envelopes, never holds signing keys, and never submits to Horizon or Soroban. Its only network responsibility is sending structured batch payloads to the API and polling for status updates.

**Key internal components:**

- `RecipientGrid` — The main data table. Each row is one recipient. Supports inline editing of all fields, checkbox-based row selection, and dynamic add/remove.
- `BulkEditBar` — A floating utility bar that activates when two or more rows are selected. Allows a single value (amount, asset, or memo) to be applied across all selected rows simultaneously.
- `BatchReview` — A read-only summary of the full payment batch shown before submission. The operator confirms here before the payload is sent to the API.
- `CSVPipeline` — Handles import (parsing a CSV into the recipient grid) and export (serializing the current grid state to a downloadable CSV).

**State management:** Recipient state is managed centrally in the root page component and passed down as props. There is no global state library in the initial implementation — the data model is simple enough that React's built-in state is sufficient. This decision will be revisited if the grid needs to support 1,000+ recipients with real-time status updates.

---

### `streampay-api` — Backend

**What it owns:** All orchestration between the frontend and the Stellar network. Receiving batch payloads, validating them against live network state, constructing and signing transaction envelopes, submitting to Horizon, handling errors and retries, and returning status to the frontend.

**What it does not own:** UI logic, CSV parsing, or on-chain state. The API is stateless between requests where possible; persistent state (transaction history, job queue) lives in a database.

**Key responsibilities:**

- **Pre-flight validation** — Before constructing any transaction, the API checks every recipient address: does the account exist on the network? Does it have the required trustline for the payment asset? Is the source account's balance sufficient for the full batch? Failures are returned to the frontend as structured errors before any transaction is attempted.

- **Transaction construction** — The API builds Stellar transaction envelopes using the [Stellar JS SDK](https://github.com/stellar/js-stellar-sdk). Each envelope contains up to 100 `payment` operations (Stellar's per-transaction limit). For batches larger than 100 recipients, the API splits the list and constructs multiple envelopes automatically.

- **Signing** — Transaction envelopes are signed server-side using a configured keypair. In a self-hosted deployment, the operator provides their own signing key via environment variable. A more secure multi-sig or HSM-based signing flow is planned for Phase 3.

- **Submission and retry** — Signed transactions are submitted to the Stellar Horizon API. The API handles transient failures (network timeouts, sequence number conflicts) with an exponential backoff retry strategy. Permanent failures (insufficient balance, bad destination) are surfaced immediately without retry.

- **Job queue** — For batches large enough to require multiple transactions, the API uses a job queue to sequence submissions. This prevents overwhelming Horizon and allows partial failures to be isolated and retried without re-processing successful transactions.

- **Contract invocation** — For advanced disbursement patterns (milestone-gated, escrow, multi-sig), the API calls `streampay-contracts` via the Soroban RPC endpoint instead of submitting a plain Horizon payment operation.

---

### `streampay-contracts` — Smart Contracts

**What it owns:** On-chain logic that enforces rules neither the frontend nor the API can override. Written in Rust using the [Soroban SDK](https://soroban.stellar.org) for Stellar's smart contract platform.

**What it does not own:** Business logic that doesn't need to be trustless. If a rule can be enforced reliably off-chain (like CSV validation or address formatting), it stays in the API. Contracts are reserved for logic where the trustless guarantee matters — where the operator, the API, or any other party should not be able to bypass the rule unilaterally.

**Planned contract modules:**

- **Disbursement contract** — Accepts a list of recipients and amounts, verifies authorization, and executes the payments atomically. Either all payments succeed or none do. This provides stronger atomicity guarantees than sequencing individual Horizon operations from the API.

- **Escrow contract** — Holds funds on behalf of a recipient until a release condition is met. Used for milestone-based grants where payment is contingent on verified delivery.

- **Multi-sig authorization** — Requires M-of-N signatures from a defined set of authorized addresses before a disbursement can proceed. Used for DAO treasury operations where no single party should have unilateral control over funds.

- **Streaming payments** *(future)* — Releases funds to a recipient continuously over a defined time period rather than in a single lump sum. Useful for ongoing contributor compensation or vesting schedules.

---

## Data Flow

### Simple Batch Payment

A standard bulk payment with no contract involvement:

```
1. Operator builds recipient list in streampay-app
        (adds rows, edits inline, uses bulk-edit bar, imports CSV)

2. Operator clicks "Review Batch"
        → App validates locally (no empty addresses, no zero amounts)
        → BatchReview screen renders a read-only summary

3. Operator confirms → App sends POST /api/batches
        Payload: { recipients: Recipient[], memo?: string, asset: string }

4. streampay-api receives the payload
        → Validates all recipient addresses against Stellar network
        → Checks trustlines for non-XLM assets
        → Checks source account balance

5. Validation passes → API constructs transaction envelope(s)
        → Splits into chunks of ≤100 recipients if needed
        → Adds memo to each envelope
        → Signs with operator keypair

6. API submits to Stellar Horizon
        → Handles retries on transient failures
        → Sequences multiple envelopes via job queue if needed

7. API returns status to streampay-app
        → App displays per-recipient success / failure states
        → Operator can export a receipt CSV
```

### Contract-Mediated Payment

For disbursements that require on-chain enforcement (escrow, multi-sig, milestone):

```
1–3. Same as above, with additional contract parameters in the payload
        e.g. { type: 'escrow', releaseCondition: '...', authorizers: [...] }

4. API validates payload and contract parameters

5. API constructs a Soroban contract invocation
        → Calls the relevant contract function with recipient list and conditions
        → Signs the transaction

6. API submits via Soroban RPC (not Horizon payment operation)

7. Contract executes on-chain
        → Enforces conditions atomically
        → Emits events readable by the API and frontend

8. API polls for contract execution result and returns status to frontend
```

---

## Shared Data Schema

The `Recipient` interface is the primary data structure shared across all three layers. It is defined in TypeScript for the frontend and API, and its fields map to Stellar transaction primitives.

```typescript
interface Recipient {
  id: string        // Internal identifier — not sent to Stellar
  address: string   // Stellar account address (G...) or federation address
  amount: number    // Payment amount — up to 7 decimal places
  asset: string     // Asset code: 'XLM', 'USDC', 'EURT', or custom Stellar asset
  memo: string      // Transaction memo — MEMO_TEXT, max 28 bytes
  selected: boolean // UI state only — stripped before sending to API
}

// API batch payload
interface BatchPayload {
  recipients: Omit<Recipient, 'id' | 'selected'>[]
  type: 'direct' | 'escrow' | 'multisig'   // direct = plain Horizon payment
  asset: string                              // Fallback asset if per-row asset is absent
  memo?: string                              // Batch-level memo override
}

// API batch response
interface BatchResult {
  batchId: string
  status: 'submitted' | 'partial' | 'failed'
  transactions: TransactionResult[]
}

interface TransactionResult {
  hash: string
  status: 'success' | 'failed'
  recipients: { address: string; status: 'success' | 'failed'; error?: string }[]
}
```

The `selected` field is UI-only state and is stripped by the frontend before any payload is sent to the API. The `id` field is an internal React key and is also stripped at the boundary.

---

## Stellar-Specific Design Decisions

### Horizon API vs Soroban

Stellar has two ways to execute payments: plain operations submitted via the Horizon REST API, and smart contract calls executed via the Soroban platform.

StreamPay uses **Horizon for simple bulk payments** — it is faster, cheaper, and well-understood. The Horizon path handles the majority of StreamPay's use cases (payroll, airdrops, DAO distributions).

StreamPay uses **Soroban for advanced disbursement patterns** — escrow, milestone-gating, multi-sig — where the trustless enforcement of on-chain logic justifies the additional complexity and cost.

The API layer abstracts this choice. The frontend sends a `type` field in the batch payload (`'direct'` vs `'escrow'` vs `'multisig'`), and the API routes to the appropriate execution path. The frontend does not need to know whether a given payment goes through Horizon or Soroban.

### Transaction Batching

Stellar allows up to **100 operations per transaction envelope**. A batch of 350 recipients therefore requires 4 transaction envelopes submitted in sequence.

The API handles this transparently. From the frontend's perspective, a batch of any size is a single POST request. The API splits, sequences, and tracks each envelope independently. If envelope 2 of 4 fails, the API returns a partial result indicating which recipients succeeded and which failed, allowing the operator to retry only the failed subset.

Sequence numbers are managed carefully — each envelope must use the correct sequence number for the source account, and the API fetches a fresh sequence number before constructing each envelope to avoid conflicts.

### Memo Handling

Stellar transaction memos are **per-transaction, not per-operation**. This means a single memo applies to all 100 recipients in an envelope — you cannot attach a different memo to each individual payment operation within one transaction.

This is an important constraint for operators using StreamPay. In practice it means:

- Batch-level memos (e.g. `"Payroll - July 2025"`) work perfectly.
- Recipient-level memos require each recipient to be in their own transaction, which is expensive and slow for large batches.

StreamPay's current approach: the `memo` field in the recipient grid is used to detect uniqueness. If all selected recipients share the same memo, it is applied at the transaction level. If recipients have differing memos, the API groups them by memo value and creates separate envelopes per group. This is surfaced to the operator in the batch review screen before submission.

### Asset Representation

Stellar assets are identified by a **code + issuer address** pair. `USDC` alone is not a unique identifier — it must be `USDC:GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN` (Circle's issuer address on Mainnet) to be unambiguous.

The frontend stores the short code (`USDC`, `XLM`, `EURT`) for readability in the grid. The API maintains a registry of known asset codes mapped to their canonical issuer addresses, which it uses during transaction construction. Operators can also provide a full `code:issuer` string in the asset field to use assets not in the default registry.

`XLM` (Stellar's native asset) requires no issuer and is handled as a special case.

---

## Inter-Repo Integration Points

The three repositories communicate through well-defined interfaces. Changes to these interfaces require coordination across repos.

**`streampay-app` → `streampay-api`**

- Interface: REST HTTP (JSON)
- Primary endpoint: `POST /api/batches` — submits a payment batch
- Secondary endpoints: `GET /api/batches/:id` — polls batch status
- Schema: `BatchPayload` (defined above)
- Versioning: API responses include a `version` field; the frontend checks compatibility on startup

**`streampay-api` → `streampay-contracts`**

- Interface: Soroban RPC
- The API constructs a contract invocation transaction, signs it, and submits via the Soroban RPC endpoint
- The contract address and ABI are configured in the API's environment variables, allowing contract upgrades without API code changes
- Contract events are polled by the API to determine execution status

**Shared type definitions**

The `BatchPayload`, `Recipient`, and `BatchResult` interfaces are the canonical contract between the frontend and API. Any change to these shapes must be reflected in both repos simultaneously and should be accompanied by a version bump in the API.

A shared types package (`@streampay/types`) is planned for Phase 2 to enforce this formally. Until then, the definitions in this document are the source of truth.

---

## Local Development Topology

When running StreamPay locally, all three services run simultaneously. The frontend proxies API requests to avoid CORS issues during development.

```
Browser
  └── http://localhost:3000        streampay-app (Next.js dev server)
        └── /api/* proxy ──────►  http://localhost:4000   streampay-api
                                        └── Stellar Testnet (Horizon)
                                        └── Soroban Testnet RPC
                                              └── streampay-contracts
                                                  (deployed to Testnet)
```

**Stellar Testnet** is used for all local development. Test accounts are funded using [Friendbot](https://friendbot.stellar.org), Stellar's Testnet faucet. No real assets are involved at any point during development.

Environment variables required per service are documented in each repository's `.env.example` file. The three critical values for local development are:

- `STELLAR_NETWORK` — `testnet` or `mainnet`
- `HORIZON_URL` — Testnet: `https://horizon-testnet.stellar.org`
- `SOURCE_KEYPAIR_SECRET` — The signing keypair for the API (use a Testnet-only keypair locally, never a Mainnet key)

---

## Security Considerations

Several areas warrant explicit attention from contributors:

**Signing key management.** The API holds the signing keypair for transaction submission. In a self-hosted deployment this is provided via environment variable. This key should never be committed to version control, logged, or exposed in API responses. A secrets manager (AWS Secrets Manager, HashiCorp Vault) should be used in production deployments.

**Input validation.** All recipient data from the frontend must be validated server-side in the API before it is used to construct transactions. The frontend's validation is UX convenience — the API's validation is the security boundary. Never trust the frontend payload.

**Pre-flight checks.** The API's pre-flight validation (account existence, trustlines, balance) prevents wasted transaction fees and protects against sending to unfunded accounts. These checks must happen against the live network state immediately before transaction construction — a check result cached from minutes earlier can be stale.

**Smart contract auditing.** `streampay-contracts` will handle real funds in production. The contracts must be formally audited before any Mainnet deployment. No contract code should be deployed to Mainnet before a third-party security review.

**Rate limiting.** The API must implement rate limiting on the batch submission endpoint to prevent abuse in any hosted deployment.

---

## Future Architecture Considerations

As StreamPay matures, several architectural decisions will need to be revisited:

- **Shared types package** — A published `@streampay/types` npm package will replace the informal type agreement between `streampay-app` and `streampay-api`.
- **WebSocket status updates** — Currently the frontend polls for batch status. For large batches, a WebSocket connection from API to frontend would give real-time per-recipient status updates without polling overhead.
- **Database layer** — Transaction history and job queue state will require a persistent database in the API. PostgreSQL is the likely choice.
- **Multi-tenancy** — Supporting multiple organizations using one hosted StreamPay instance will require an authentication layer, isolated signing key management per tenant, and scoped data access.
- **Frontend state at scale** — For recipient lists exceeding ~500 rows, React's built-in state management may introduce performance issues. Virtualized rendering and a more formal state management approach (Zustand or Jotai) will be evaluated at that threshold.

---

*This document reflects the intended architecture as of the early development phase. It will be updated as the system evolves. If you notice a discrepancy between this document and the actual code, please open an issue.*
