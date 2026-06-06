# StreamPay

> **Enterprise-grade bulk payment infrastructure on Stellar — open-source, modular, and built for the decentralized web.**

StreamPay is an open-source organization building the tooling and infrastructure for scalable, multi-recipient crypto payment processing on the [Stellar network](https://stellar.org). It is the production-ready evolution of [StellarStream](https://github.com/stellarstream), rebuilt from the ground up as a fully separated, team-friendly system with a dedicated frontend, backend API, and on-chain smart contracts — each maintained in its own repository under this organization.

Whether you're running DAO treasury distributions, crypto payroll for a globally distributed team, token airdrops, or DeFi disbursements, StreamPay gives you the full stack to do it reliably, auditably, and at scale.

---

## Organization Structure

StreamPay is organized as three independent repositories, each with a clearly scoped responsibility. They are designed to work together but can also be adopted individually depending on your needs.

```
github.com/streampay/
├── streampay-app        → Next.js frontend — the payment UI
├── streampay-api        → Node.js backend — transaction orchestration & business logic
└── streampay-contracts  → Stellar smart contracts — on-chain payment execution
```

| Repository | Description | Primary Language |
|---|---|---|
| [`streampay-app`](https://github.com/streampay/streampay-app) | User-facing web application: recipient grid, bulk-edit tools, CSV pipeline | TypeScript / Next.js |
| [`streampay-api`](https://github.com/streampay/streampay-api) | REST API layer: transaction building, signing, submission, and job queuing | TypeScript / Node.js |
| [`streampay-contracts`](https://github.com/streampay/streampay-contracts) | Stellar smart contracts: on-chain disbursement logic and escrow primitives | Rust / Soroban |

> **Note:** StreamPay is in early development. All three repositories are actively being scaffolded. Expect breaking changes, incomplete features, and evolving interfaces. Stars, issues, and early contributors are very welcome.

---

## Why StreamPay?

Traditional payment rails are slow, expensive, and built for a world where finance is centralized. Stellar changes the equation — near-instant finality (3–5 seconds), sub-cent fees, native multi-asset support (USDC, XLM, EURT, and any custom Stellar asset), and a global network with no geographic restrictions.

But raw Stellar access — via the Horizon API or the JS SDK — requires engineering effort to use safely at scale. StreamPay abstracts that complexity into a cohesive product:

- **No infrastructure re-invention**: The API handles transaction construction, fee bumping, error retry, and submission so your team doesn't have to.
- **Bulk-first UX**: The frontend is built around processing many recipients at once — not one at a time. Bulk-edit, CSV import/export, and batch validation are first-class features.
- **Auditable by design**: Every payment batch is structured, reviewable, and exportable before anything touches the network.
- **Separation of concerns**: Frontend, backend, and contracts are decoupled. Teams can replace or extend any layer without disrupting the others.
- **Open-source and self-hostable**: No vendor lock-in, no SaaS subscription. Deploy StreamPay on your own infrastructure and own your payment pipeline end-to-end.

---

## Web3 Use Cases

StreamPay is designed to serve the operational needs of Web3 organizations at any scale:

### DAO & Protocol Treasury Distributions
Export wallet addresses from a governance snapshot, import them into StreamPay, bulk-assign the reward amount and asset in two clicks, run the pre-flight validation check, and submit the entire batch as a single operation. Treasurer workflows that used to take hours now take minutes.

### Crypto Payroll for Remote Teams
Web3-native teams paying contributors in USDC or XLM can use StreamPay as their payroll interface. Organize recipients by role or seniority tier, bulk-set salaries per tier, assign a unified memo like `"Payroll - July 2025"` across all rows, and dispatch.

### Token Airdrops
Token issuers running campaigns on Stellar can manage eligibility lists directly in the recipient grid. Segment by tier — early adopters, community contributors, ecosystem partners — bulk-assign the correct amount per segment, and export the final list for sign-off before execution.

### NFT & Creator Royalty Payments
Platforms distributing royalties across a large creator base can use the grid to manage per-creator payout amounts while stamping a consistent memo across all rows for accounting reconciliation.

### Grant Programs
Foundations and accelerators disbursing milestone-based grants can maintain a live recipient grid across funding rounds, update amounts per milestone, and export a full audit trail for treasury reporting.

### DeFi Protocol Operations
Protocols that periodically top up LP addresses, reimburse subsidies, or distribute protocol fees can pipe their eligibility data directly into StreamPay via the CSV import and submit in one coordinated batch.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     streampay-app                       │
│         Next.js · TypeScript · Tailwind CSS             │
│                                                         │
│  ┌──────────────────┐   ┌───────────────────────────┐  │
│  │  Recipient Grid  │   │    Bulk-Edit Utility Bar  │  │
│  │  (inline edit,   │   │  (amount / asset / memo   │  │
│  │   row select,    │   │   applied to all selected) │  │
│  │   CSV pipeline)  │   └───────────────────────────┘  │
│  └──────────────────┘                                   │
└───────────────────────────┬─────────────────────────────┘
                            │ REST / JSON
┌───────────────────────────▼─────────────────────────────┐
│                     streampay-api                        │
│           Node.js · TypeScript · Express                 │
│                                                         │
│  - Payment batch orchestration                          │
│  - Stellar transaction construction (Horizon SDK)       │
│  - Signing & fee bump management                        │
│  - Job queue for large batches                          │
│  - Pre-flight validation (funded accounts, balances)    │
└───────────────────────────┬─────────────────────────────┘
                            │ Stellar Horizon API
┌───────────────────────────▼─────────────────────────────┐
│                  streampay-contracts                     │
│                  Rust · Soroban SDK                      │
│                                                         │
│  - On-chain disbursement logic                          │
│  - Escrow and conditional release primitives            │
│  - Multi-sig authorization flows                        │
└───────────────────────────┬─────────────────────────────┘
                            │
                    Stellar Network
              (Testnet during development,
               Mainnet for production)
```

### How the layers interact

The **frontend** (`streampay-app`) is the operator's interface. It manages the recipient list, bulk-edit operations, and pre-submission review. When a batch is ready, it sends a structured payload to the **API**.

The **API** (`streampay-api`) receives the batch, validates each recipient against the Stellar network (account existence, trustlines, balance sufficiency), constructs the transaction envelope using the Stellar SDK, handles signing, and submits to Horizon. For large batches that exceed Stellar's 100-operations-per-transaction limit, the API splits and sequences multiple transactions automatically.

The **contracts** (`streampay-contracts`) extend what's possible on-chain. Written in Rust for the [Soroban](https://soroban.stellar.org) smart contract platform on Stellar, they handle advanced disbursement patterns — milestone-gated grant releases, escrow holds, multi-signature authorization, and streaming payment schedules — that can't be expressed with simple Horizon operations alone.

---

## Repository Guides

Each repository has its own detailed README with setup instructions, architecture decisions, and contribution guidelines. Start with the repo most relevant to your work:

- **Frontend developers** → [`streampay-app`](https://github.com/streampay/streampay-app)
- **Backend / API developers** → [`streampay-api`](https://github.com/streampay/streampay-api)
- **Smart contract / Soroban developers** → [`streampay-contracts`](https://github.com/streampay/streampay-contracts)

---

## Stellar Network Integration

StreamPay's data model maps directly to Stellar's transaction primitives across all three layers.

| StreamPay Field | Stellar Transaction Field | Notes |
|---|---|---|
| `address` | Destination account | Must be a funded Stellar account with the correct trustline |
| `amount` | Operation amount | Supports up to 7 decimal places (Stellar's native precision) |
| `asset` | Asset code + issuer | e.g. `USDC:GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN` |
| `memo` | Transaction memo | `MEMO_TEXT` type, max 28 bytes — required by most exchanges |

Each recipient row in the grid corresponds to one `payment` operation. Stellar supports up to 100 operations per transaction envelope — the API layer batches and sequences accordingly for large disbursements.

---

## Getting Started (Local Development)

To run the full StreamPay stack locally, clone all three repositories and follow the individual setup guides. A rough local setup looks like:

```bash
# 1. Frontend
git clone https://github.com/streampay/streampay-app
cd streampay-app && npm install && npm run dev
# Runs on http://localhost:3000

# 2. API
git clone https://github.com/streampay/streampay-api
cd streampay-api && npm install && npm run dev
# Runs on http://localhost:4000

# 3. Contracts (requires Rust + Soroban CLI)
git clone https://github.com/streampay/streampay-contracts
cd streampay-contracts && cargo build
# Deploy to Stellar Testnet for local testing
```

For a full environment setup including environment variables, Stellar Testnet funding, and service wiring, see the **Getting Started** section in each repo's README.

---

## Roadmap

StreamPay is in early development. The roadmap below reflects the planned build sequence across all three repos.

### Phase 1 — Foundation (Current)
- [ ] Scaffold all three repositories with base project structure
- [ ] Frontend: recipient grid with inline editing and bulk-edit bar
- [ ] Frontend: CSV import/export pipeline
- [ ] API: basic REST endpoints for batch submission
- [ ] API: Stellar Horizon integration for single-asset payments
- [ ] Contracts: initial Soroban project setup and testnet deployment

### Phase 2 — Core Payment Flow
- [ ] API: full transaction construction, signing, and submission
- [ ] API: pre-flight validation (funded accounts, trustlines, balance checks)
- [ ] API: batch splitting for lists exceeding 100 recipients
- [ ] Frontend: Stellar address format validation (checksums, federation)
- [ ] Frontend: Testnet / Mainnet environment toggle
- [ ] Frontend: real-time transaction status tracking

### Phase 3 — Advanced Features
- [ ] Contracts: milestone-gated disbursement logic
- [ ] Contracts: escrow and conditional release primitives
- [ ] Contracts: multi-signature authorization flows
- [ ] API: job queue for very large batches (1,000+ recipients)
- [ ] API: transaction history and receipt storage
- [ ] Frontend: wallet connect (Freighter, Albedo, xBull)
- [ ] Frontend: undo/redo for bulk edits
- [ ] Frontend: role-based access (approver / submitter workflow)

### Phase 4 — Production Hardening
- [ ] End-to-end test suite across all three layers
- [ ] Audit of smart contract code
- [ ] Self-hosting documentation and Docker Compose setup
- [ ] SDK / API client library for third-party integrations

---

## Contributing

StreamPay is open to contributors at every level. Since we are in early development, there is significant opportunity to shape the architecture, not just add features.

**Before opening a PR:**
1. Check the Issues tab in the relevant repository for existing discussions.
2. For anything beyond a small bug fix, open an issue first so we can align on approach before you write code.

**When contributing:**
- Follow the code style of the repository you're working in (TypeScript strict mode throughout; Rust `clippy` clean for contracts).
- Add or update types for any new data shapes.
- Write a clear PR description explaining what changed, why, and how to test it.
- Ensure your changes don't break the integration points between repos — the shared data schema is the contract between them.

**Good first issues** are labelled `good first issue` across all three repositories.

---

## Community

StreamPay is being built in the open. Discussions, questions, and feedback are welcome via GitHub Issues in the relevant repository. A Discord community will be set up as the project matures.

---

## License

All StreamPay repositories are released under the **MIT License**. See the `LICENSE` file in each repository for details.

---

*StreamPay is an independent open-source organization and is not affiliated with the Stellar Development Foundation.*
