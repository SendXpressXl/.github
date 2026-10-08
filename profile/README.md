<div align="center">
  <h1>SendXpress</h1>
  <p><strong>Social commerce for cross-border trade, secured by Stellar escrow.</strong></p>

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&labelColor=000)](https://opensource.org/licenses/MIT)
[![Built on Stellar](https://img.shields.io/badge/Built%20on-Stellar-yellow?style=flat-square&labelColor=000&logo=stellar&logoColor=white)](https://stellar.org)
[![Soroban](https://img.shields.io/badge/Contracts-Soroban-yellow?style=flat-square&labelColor=000)](https://soroban.stellar.org)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-yellow?style=flat-square&labelColor=000)](https://github.com/SendXpressXl/backend/blob/main/CONTRIBUTING.md)

[Smart Contract](https://github.com/SendXpressXl/smartcontract) · [Backend](https://github.com/SendXpressXl/backend) · [Frontend](https://github.com/SendXpressXl/frontend) · [Mobile](https://github.com/SendXpressXl/mobile) · [Docs](https://github.com/SendXpressXl/docs)

</div>

---

## What this is

Cross-border trade between importers and suppliers runs on a trust deficit. Buyers can't pay before goods arrive — sellers can't ship before payment clears. Banks bridge this gap with letters of credit, but they're slow (3–7 days), expensive (1–3% + wire fees), and exclusive to businesses with bank relationships.

SendXpress replaces the letter of credit with a **Soroban smart contract on Stellar**. A buyer locks funds on-chain before the seller ships. The seller sees those funds are cryptographically secured and ships. When the buyer confirms receipt the contract releases payment in seconds — no intermediary, no waiting, no trust required between strangers. A social feed, real-time chat, on-chain marketplace, and reputation system sit on top so importers and suppliers can find each other, negotiate, and transact in one place.

```
Buyer locks funds → Seller ships → Buyer confirms → Contract releases payment
                                          ↓
                                  Dispute raised → Arbiter resolves on-chain
                                                   (split or full award)
```

|                    | Traditional (L/C)     | SendXpress              |
| ------------------ | --------------------- | ----------------------- |
| **Settlement**     | 3–7 banking days      | ~5 seconds (Stellar)    |
| **Fees**           | 1–3% + wire fees      | Configurable basis pts  |
| **Trust model**    | Bank as intermediary  | On-chain escrow         |
| **Disputes**       | Courts / arbitration  | On-chain arbiter        |
| **Bank required**  | Yes                   | No — Stellar wallet     |
| **Currency**       | Single account ccy    | Any Stellar asset       |

---

## How it works, concretely

1. A supplier creates a **marketplace listing** on-chain — product name, category (Textiles, Electronics, Agriculture, Machinery, Chemicals, FoodBeverage, Furniture, Automotive), price, accepted token, stock, minimum order quantity, and a `metadata_hash` anchoring off-chain images and descriptions.
2. An importer discovers the listing on the **social feed**, opens a **real-time chat**, and negotiates price and terms.
3. The buyer calls `create_deal` on the **escrow contract** — locking tokens (XLM or any Stellar asset), recording buyer, seller, amount, description, and a deadline. Status: `Created`.
4. The seller sees funds secured on-chain and calls `mark_shipped`. Status: `Shipped`.
5. When goods arrive, the buyer calls `confirm_received`. The contract deducts a platform fee (in basis points, e.g. 50 = 0.5%), releases the net amount to the seller, and calls the **reputation contract** to record the completed trade. Status: `Confirmed`.
6. If goods never arrive or a dispute arises, either party calls `raise_dispute`. A designated on-chain arbiter reviews evidence and calls `resolve_dispute` — specifying a split or full award. Status: `Resolved`.
7. If a deadline passes without shipment, anyone can call `expire_deal` — funds return to the buyer automatically. Status: `Expired`.
8. Off-chain, the **backend** indexes deal state changes, maintains per-second chat, handles **CCTP cross-chain USDC deposits** (so buyers on Base or other EVM chains can fund Stellar escrows), and processes **Transak fiat on-ramp webhooks** so buyers without crypto can pay by card.

---

## Smart Contracts

Three production Soroban contracts, each independently deployable:

### `sendxpress-escrow`
The core deal machine. Manages the full lifecycle: `Created → Shipped → Confirmed`, `Disputed → Resolved`, `Cancelled`, `Expired`. Includes a configurable platform fee (basis points), fee accumulation per token, fee sweep by admin, and TTL-extended persistent storage (~1 year per deal).

```
Created ──→ Shipped ──→ Confirmed  (net funds → seller, fee → platform)
   │                       │
   │                    Disputed ──→ Resolved  (arbiter splits or awards)
   │
   ├──→ Cancelled  (buyer refunded)
   └──→ Expired    (deadline passed, buyer refunded)
```

### `sendxpress-marketplace`
On-chain product listing registry. Stores name, category, price, accepted token, stock, MOQ, verified-seller badge, and a `metadata_hash` anchoring off-chain content. Listings persist for ~1 year (TTL-managed). Admin grants verified-seller badges; owners can activate/deactivate listings.

### `sendxpress-reputation`
Immutable on-chain trust scoring. Each confirmed trade increments `trades_completed` and `volume_traded`. Each dispute outcome updates `disputes_won`, `disputes_lost`, or `disputes_abandoned`. Score is computed as:

```
score = clamp(
    (trades_completed × 100)
    − (disputes_lost  × 30)
    − (disputes_abandoned × 10),
    0, 10_000    // stored as basis points → 0–100.00
)
```

Only whitelisted escrow contracts can record trade outcomes — preventing score manipulation. The admin manages the whitelist.

---

## Repositories

| Repo | Role | Stack |
|---|---|---|
| [**smartcontract**](https://github.com/SendXpressXl/smartcontract) | Three production Soroban contracts: escrow (fee model, TTL, split-dispute), marketplace (on-chain listings, verified sellers, stock), reputation (scoring formula, whitelist) | Rust, Soroban SDK |
| [**backend**](https://github.com/SendXpressXl/backend) | REST API — social feed, marketplace, real-time chat, Stellar escrow orchestration, CCTP cross-chain USDC, Transak fiat on-ramp, milestone payments, RBAC, deal state machine | Node.js 20, Express, Supabase/Postgres |
| [**frontend**](https://github.com/SendXpressXl/frontend) | Web app — supplier discovery, real-time chat, Freighter wallet, Soroban escrow flows, shipment tracking, dark/light mode | Next.js 13, TypeScript, Tailwind CSS |
| [**mobile**](https://github.com/SendXpressXl/mobile) | Native iOS app — auto-provisioned Stellar wallet (no seed phrase), social feed, chat, escrow deals, role-based UX | Swift 5.9, SwiftUI, CryptoKit, Keychain |
| [**docs**](https://github.com/SendXpressXl/docs) | Protocol documentation — architecture, API reference, contract interface, contribution guides | Markdown |

---

## Architecture

```
┌──────────────────────┐     ┌──────────────────────────┐
│   iOS  (SwiftUI)     │     │   Web  (Next.js)         │
│   CryptoKit wallet   │     │   Freighter wallet       │
│   Keychain storage   │     │   Soroban bindings       │
└──────────┬───────────┘     └────────────┬─────────────┘
           │   REST / WebSocket           │   REST / WebSocket
           └──────────────┬──────────────┘
                          ▼
           ┌──────────────────────────────────────┐
           │  Backend  (Node.js / Express)        │
           │  feed · chat · listings · users      │
           │  deal state machine · RBAC           │
           │  CCTP cross-chain · fiat on-ramp     │
           │  milestone payments · webhooks       │
           └──────────┬───────────────────────────┘
                      │  Stellar SDK (never holds keys)
           ┌──────────▼───────────────────────────┐
           │  Soroban Contracts  (Rust)           │
           │  sendxpress-escrow                   │
           │  sendxpress-marketplace              │
           │  sendxpress-reputation               │
           └──────────┬───────────────────────────┘
                      │
           ┌──────────▼───────────────────────────┐
           │  Stellar Network                     │
           │  ~5s finality · ~$0.000001 / tx      │
           └──────────────────────────────────────┘
```

All writes that change escrow state are signed by the user's own wallet — the backend never custodies private keys. The mobile app uses Apple CryptoKit to generate Ed25519 keypairs stored in iOS Keychain; the web app uses Freighter. Both clients talk directly to Soroban RPC for on-chain reads and signed writes; the backend handles off-chain concerns (feed, chat, cross-chain, fiat) independently.

---

## Getting started

```bash
# Smart contracts — build and test all three Soroban contracts
git clone https://github.com/SendXpressXl/smartcontract.git
cd smartcontract && cargo test --workspace

# Backend — REST API (see README for Supabase + env setup)
git clone https://github.com/SendXpressXl/backend.git
cd backend && npm install && npm run dev

# Frontend — web app
git clone https://github.com/SendXpressXl/frontend.git
cd frontend && npm install && npm run dev

# Mobile — open in Xcode 15+
git clone https://github.com/SendXpressXl/mobile.git
# Open SendXpress.xcodeproj, select a simulator, press Run
```

Stellar **testnet** is the default across all repos — no mainnet credentials needed for local development.

---

## Status

SendXpress is under active open-source development on Stellar testnet. All three Soroban contracts are deployed and functional; the backend, frontend, and iOS app are wired end-to-end. Cross-chain USDC (CCTP), fiat on-ramp (Transak), and milestone-based deal payments are live in the backend. **No professional audit has been conducted** — do not use with real funds without your own independent review. See each repo's `SECURITY.md` for responsible disclosure.

## Contributing

We welcome contributions across all repos — contracts, API, frontend, and mobile.

1. Pick a repo and read its `CONTRIBUTING.md`
2. Browse open issues and look for `good first issue`
3. Fork, branch, keep PRs focused, and add tests for logic changes

All repos are licensed under [MIT](https://opensource.org/licenses/MIT).

<div align="center">
  <sub>Built on Stellar · Powered by Soroban</sub>
</div>
