<div align="center">
  <h1>SendXpress</h1>
  <p><strong>Social commerce for cross-border trade, secured by Stellar escrow.</strong></p>

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&labelColor=000)](https://opensource.org/licenses/MIT)
[![Built on Stellar](https://img.shields.io/badge/Built%20on-Stellar-yellow?style=flat-square&labelColor=000&logo=stellar&logoColor=white)](https://stellar.org)
[![Soroban](https://img.shields.io/badge/Contracts-Soroban-yellow?style=flat-square&labelColor=000)](https://soroban.stellar.org)

[Smart Contract](https://github.com/SendXpressXl/smartcontract) · [Backend](https://github.com/SendXpressXl/backend) · [Frontend](https://github.com/SendXpressXl/frontend) · [Mobile](https://github.com/SendXpressXl/mobile) · [Docs](https://github.com/SendXpressXl/docs)

</div>

---

## What this is

Cross-border trade between importers and suppliers is broken. Buyers can't trust sellers they've never met. Sellers can't ship before payment. Banks take days, charge large fees, and require both parties to have accounts. Disputes go unresolved.

SendXpress fixes the trust problem with **on-chain escrow on Stellar**. A buyer locks funds in a Soroban smart contract before the seller ships. The seller sees the funds are secured and ships. When the buyer confirms receipt, the contract releases payment automatically — no intermediary, no waiting, no trust required. If a dispute arises, a designated on-chain arbiter resolves it.

On top of the escrow mechanism sits a social layer: a product feed, real-time chat, listing discovery, and supplier reputation — so importers and suppliers can find each other, negotiate, and transact in one place.

```
Buyer locks funds → Seller ships → Buyer confirms → Contract releases payment
                                           ↓
                                   Dispute? → Arbiter resolves on-chain
```

|                    | Traditional Trade     | SendXpress              |
| ------------------ | --------------------- | ----------------------- |
| **Trust model**    | Personal trust / L/C  | On-chain escrow         |
| **Settlement**     | 3–7 banking days      | 5 seconds (Stellar)     |
| **Fees**           | 1–3% + wire fees      | ~$0.000001 per tx       |
| **Disputes**       | Lawyers / arbitration | On-chain arbiter        |
| **Bank required**  | Yes                   | No — Stellar wallet     |
| **Currency**       | One per account       | Multi-currency natively |

### How it works, concretely

1. A supplier posts products on the **social feed** — photos, descriptions, pricing, tagged listings.
2. An importer discovers the listing, opens a **real-time chat**, and negotiates price and terms.
3. The buyer initiates a deal and **locks XLM** (or USDC) into the Soroban escrow contract. The contract records the buyer, seller, amount, description, and deadline.
4. The seller sees the funds secured on-chain and **marks the shipment** (`mark_shipped`). Status moves `Created → Shipped`.
5. When goods arrive, the buyer **confirms receipt** (`confirm_received`). The contract releases funds to the seller instantly and increments their on-chain reputation score.
6. If the goods never arrive or a dispute arises, either party can **raise a dispute**. A designated arbiter reviews evidence off-chain and resolves it on-chain — funds go to the rightful party.
7. The backend indexes deal state changes, serves chat and feed data, and handles cross-chain USDC deposits via **CCTP** and fiat on-ramp via **Transak**.

---

## Repositories

| Repo | Role | Stack |
|---|---|---|
| [**smartcontract**](https://github.com/SendXpressXl/smartcontract) | Soroban escrow contract — full deal lifecycle (lock, ship, confirm, dispute, cancel), arbiter resolution, on-chain reputation | Rust, Soroban SDK |
| [**backend**](https://github.com/SendXpressXl/backend) | REST API — user management, social feed, marketplace listings, real-time chat, Stellar escrow orchestration, CCTP cross-chain deposits, Transak fiat on-ramp, milestone payments | Node.js 20, Express, Supabase/Postgres |
| [**frontend**](https://github.com/SendXpressXl/frontend) | Web app — supplier discovery feed, real-time chat, Freighter wallet, Soroban escrow flows, shipment tracking, dark/light mode | Next.js 13, TypeScript, Tailwind CSS |
| [**mobile**](https://github.com/SendXpressXl/mobile) | Native iOS app — auto-provisioned Stellar wallet (no seed phrases), social feed, chat, escrow deals, role-based UX | Swift 5.9, SwiftUI, CryptoKit, Keychain |
| [**docs**](https://github.com/SendXpressXl/docs) | Protocol documentation — architecture, API reference, contract interface, contribution guides | Markdown |

---

## Architecture

```
┌──────────────────────┐     ┌──────────────────────────┐
│  iOS App (SwiftUI)   │     │   Web App (Next.js)      │
│  CryptoKit wallet    │     │   Freighter wallet       │
│  Keychain storage    │     │   Soroban bindings       │
└──────────┬───────────┘     └────────────┬─────────────┘
           │ REST / WebSocket             │ REST / WebSocket
           └──────────────┬──────────────┘
                          ▼
           ┌──────────────────────────────┐
           │  Backend  (Express / Node)   │
           │  feed · chat · listings      │
           │  escrow · CCTP · fiat ramp   │
           │  milestone payments · RBAC   │
           └──────────┬───────────────────┘
                      │ Stellar SDK
           ┌──────────▼───────────────────┐
           │  Soroban Contract  (Rust)    │
           │  lock · ship · confirm       │
           │  dispute · arbiter · repute  │
           └──────────┬───────────────────┘
                      │
           ┌──────────▼───────────────────┐
           │  Stellar Network             │
           │  5s finality · ~$0.000001    │
           └──────────────────────────────┘
```

The **mobile** and **web** clients talk directly to the **backend** over REST and WebSockets. The backend orchestrates Stellar transactions and syncs state to Supabase/Postgres. Writes that change escrow state (lock, confirm, cancel) are signed by the user's wallet — the backend never custodies private keys.

---

## Getting started

```bash
# Smart contract — build and test the Soroban escrow
git clone https://github.com/SendXpressXl/smartcontract.git
cd smartcontract && cargo test

# Backend — API + Supabase (see README for env setup)
git clone https://github.com/SendXpressXl/backend.git
cd backend && npm install && npm run dev

# Frontend — web app
git clone https://github.com/SendXpressXl/frontend.git
cd frontend && npm install && npm run dev

# Mobile — open in Xcode 15+
git clone https://github.com/SendXpressXl/mobile.git
# Open SendXpress.xcodeproj, select a simulator, run
```

The Stellar **testnet** is the default across all repos — no mainnet setup needed for local development.

---

## Status

SendXpress is under active open-source development on Stellar testnet. The escrow contract, backend API, and iOS app are functional end-to-end. Cross-chain USDC (CCTP), fiat on-ramp (Transak), and milestone-based payments are integrated in the backend. **No professional audit has been conducted** — do not use with real funds without your own review.

## Contributing

We welcome contributions across all repos — contracts, API, frontend, and mobile.

1. Read the repo's `CONTRIBUTING.md`
2. Browse open issues and look for `good first issue`
3. Fork, branch, keep PRs focused, add tests for logic changes

All repos are licensed under [MIT](https://opensource.org/licenses/MIT).

<div align="center">
  <sub>Built on Stellar · Powered by Soroban</sub>
</div>
