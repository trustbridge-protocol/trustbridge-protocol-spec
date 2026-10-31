# TrustBridge Protocol

**Safe-powered deal-room escrow infrastructure for P2P and OTC crypto trading · Base · Dubai, UAE**

> "Trust without a middleman."

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Status: Prototype](https://img.shields.io/badge/Status-Prototype-orange.svg)]()
[![Contracts: In development](https://img.shields.io/badge/Contracts-In%20development%2C%20not%20audited-orange.svg)]()
[![Chain: Base](https://img.shields.io/badge/Chain-Base-blue.svg)]()
[![Powered by Safe](https://img.shields.io/badge/Powered%20by-Safe-green.svg)]()

---

## What is TrustBridge Protocol?

Two parties who don't trust each other still have to decide who moves first. In P2P and OTC trading that standoff is usually settled with rituals: test transactions, handshakes, a middleman holding the funds.

TrustBridge replaces those rituals with a shared vault and fixed rules. Each deal gets its own **Safe 2-of-2 smart account**, owned by the two parties. Both deposit, both confirm, and the assets swap in one transaction. If the deal doesn't complete, each party gets their own deposit back.

TrustBridge is designed to be non-custodial. It holds no keys, makes no decisions about who gets paid, and has no power to move, freeze or redirect anyone's funds.

It is built as **white-label deal-room infrastructure for independent crypto operators** (OTC desks, brokers, P2P communities), as well as for direct use.

**Current status**

- **Prototype:** React/JSX deal flow (22 components, 5 screens) at [app.trustbridgeprotocol.xyz](https://app.trustbridgeprotocol.xyz). Simulated wallets, no real funds.
- **Smart contracts:** in development to [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md) v1.2 in a private repository. AI-assisted engineering under founder direction. **Not reviewed, not audited, not deployed.** An independent senior Solidity / Safe review and a recognised security audit come before any mainnet deployment.

---

## Two products

| Product | What it does | Safe architecture | Status |
|---------|-------------|-------------------|--------|
| **TrustBridge Deal** | Time-limited P2P / OTC exchange. Both parties deposit the agreed assets into the deal's own Safe; the swap happens when both confirm. Refund on expiry. Price-band protection via Chainlink | One Safe 2-of-2 per deal, with a TrustBridge module and guard | v1 — in development |
| **TrustBridge Vault** | Longer-term shared vault for business partners, DAOs, joint ventures and vesting. Proportional return, yield integration, milestone-based release | Safe M-of-N, any threshold | Planned — Phase 4 |

---

## How it works

```
┌──────────────────────────────────────────────────────────────┐
│  TrustBridgeFactory                                          │
│  Checks and records each deal, locks its rules at creation,  │
│  deploys the deal's Safe. Never holds funds.                 │
└───────────────┬──────────────────────────────────────────────┘
                │ one per deal
┌───────────────▼──────────────────────────────────────────────┐
│  Deal Safe — official, unmodified Safe, 2-of-2               │
│  Owners: Party A and Party B. Holds that deal's deposits only│
│                                                              │
│   TrustBridgeModule — the only way funds can leave, and only │
│     to Party A, Party B or the recorded fee wallets          │
│   TrustBridgeGuard  — blocks every owner-signed transaction, │
│     even with both owners signing                            │
└───────────────┬──────────────────────────────────────────────┘
                │
┌───────────────▼───────────────┐  ┌───────────────────────────┐
│  Price source adapter         │  │  TrustBridgeConfig        │
│  Chainlink on Base, with      │  │  Settings for NEW deals   │
│  staleness and L2 sequencer   │  │  only, 7-day public       │
│  checks. Never CoinGecko      │  │  notice. No power over    │
│                               │  │  funds or live deals      │
└───────────────────────────────┘  └───────────────────────────┘
```

Each deal's funds are isolated in its own Safe, so a problem in one deal can't reach another.

### Funds move in only these ways

All governed by rules both parties agreed when the deal was created:

- **Both parties confirm delivery** → atomic swap
- **One party confirms and the other doesn't respond** within the auto-release window → the same swap
- **Both parties agree to cancel** → each deposit returned
- **Party A cancels before the deal is fully funded** → each deposit returned
- **Expiry, or a price breach that isn't resolved** → each deposit returned

Refunds are paid to each party independently. If one transfer fails (for example, a stablecoin issuer has frozen that address), that refund stays on record and can be retried, and the other party is still paid. The swap itself is all-or-nothing.

### Deal lifecycle (v1)

```
PENDING → JOINED → NEGOTIATING → AGREED → FUNDED_PARTIAL → LOCKED → COMPLETED
           │                                                 │
           └ Party A rejects → back to PENDING (new invite)  ├ price breach (Pro / Enterprise) → RENEGOTIATING
                                                             │      both agree new terms → LOCKED
                                                             │      window lapses → CANCELLED (refunds)
                                                             └ price breach (Free) → CANCELLED (refunds)

Any state before COMPLETED:  both parties cancel → CANCELLED · expiry reached → EXPIRED (refunds)
```

**Joining a deal.** Party A shares an invite link. Party B's acceptance is a signature naming Party B's own wallet, so a copied acceptance is useless to anyone else, and each invite works once. Party A then confirms "this is my counterparty" before the deal's Safe is created or anything can be deposited. Party A can also lock the deal to one known address.

> **Dispute resolution:** v1 has no discretionary dispute mechanism. Nobody at TrustBridge decides any outcome. That is deliberate: it keeps the non-custodial position clean. "Raise a Concern" exists in the app as a timestamped record only, with no effect on funds. Kleros decentralised arbitration is planned for v2.

---

## Smart contract functions (v1)

**7 core + 4 supporting.** Full parameters and security requirements: [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md).

| Function | Who | What it does |
|----------|-----|-------------|
| `createDeal()` | Party A | Checks the tokens, amounts, expiry, plan limits and $10 minimum; locks the deal's rules, fee and price band for its whole life |
| `acceptDeal()` | Party B | Joins via the invite signature or the address lock |
| `confirmCounterparty()` | Party A | Approves Party B → the deal's Safe is created, with module and guard. Or rejects → new invite |
| `confirmTerms()` | Each party | Confirms the exact on-chain terms; refused if the terms changed after they were shown |
| `depositAssets()` | Each party | Pulls exactly what's owed into the deal's Safe; no amount for the caller to get wrong |
| `amendTerms()` | Both parties | New amounts and/or expiry; takes effect only when both sign the identical change. Also used to renegotiate after a price breach |
| `confirmDelivery()` | Each party | Records the confirmation on-chain; the second one releases the funds |
| `releaseFunds()` | Anyone | Completes a release both parties have confirmed: price-band check, then the atomic swap |
| `triggerAutoRelease()` | Anyone | After the window, completes a release only one party confirmed. The caller gains nothing |
| `claimRefund()` | Anyone | After expiry or an unresolved breach, returns each deposit; zero fee. Retries a refund that failed |
| `cancelDeal()` | Party A, or both | Party A alone before LOCKED; both parties at any stage before completion. Zero fee |

---

## Security

See [SECURITY.md](SECURITY.md) for the full threat model.

- **ConfirmDeliveryModal** shows both parties a human-readable summary of exactly what moves, before every release. It's the direct response to the Bybit $1.5B UI-spoofing attack (February 2025). On-chain, every confirmation is also bound to the exact terms, so a confirmation made against a spoofed or out-of-date screen is refused
- **Chainlink only** for on-chain price checks, with staleness checks. A stale feed pauses price-dependent actions; it never affects refunds, cancellation or expiry
- **Reentrancy protection and checks-effects-interactions** on every fund-moving path
- **No admin path to user funds.** Settings changes apply only to new deals, after a 7-day public notice. An immediate pause stops new deals only; existing deals always carry on. There is no emergency "break-glass" mode
- **No alternative payout address.** Payouts go only to the party addresses recorded in the deal
- **Works with company wallets.** Parties can be personal wallets or smart-contract wallets, including an OTC desk's own Safe multisig

### Immutable design decisions

| Decision | Why it cannot change |
|----------|---------------------|
| 2-of-2 multisig — cannot be waived | Neither party can act alone; the only one-sided outcomes are rule-based and agreed at creation |
| Time-lock auto-refund — no admin override | An override would be a central point of attack |
| Non-custodial — TrustBridge cannot access user funds | Core value proposition and regulatory position |
| Fee on completion only — never on creation, expiry or cancellation | A failed deal never costs a platform fee |
| ConfirmDeliveryModal — mandatory, never removable | Primary defence against UI spoofing |
| Chainlink only for on-chain prices — never CoinGecko | Manipulation-resistant and verifiable |
| Checks-effects-interactions on every transfer | Reentrancy protection |
| No discretionary dispute resolution in v1 | Preserves the non-custodial position |
| No operator or admin power over live deals | Exits belong to the parties and the rules |

---

## Plan tiers

| Parameter | Free | Pro | Enterprise |
|-----------|------|-----|------------|
| Platform fee (completion only) | 1%, max $10 | 0.5%, max $25 | 0.25% standard (operator-registered) |
| Who pays the fee | Party A | Party A | Party A |
| Party B | Always free | Always free | Always free |
| Max deal value (enforced on-chain) | $5,000 | $500,000 | Unlimited |
| Minimum deal value | $10 | $10 | $10 |
| Max deal duration | 7 days | 30 days | 365 days |
| Single-party auto-release | 15 minutes | 15 minutes | Operator-set (default 2 hours), or off |
| Price band | ±15%, fixed | Party-chosen, or off | Party-chosen, or off |
| On a price breach | Cancels and refunds both | 30-minute renegotiation window | 2-hour renegotiation window |
| Same asset on both sides | Not permitted in Deal | Not permitted in Deal | Not permitted in Deal |
| Approved tokens | USDC, USDT, ETH (as WETH), WBTC, DAI, WETH | Same | Same + operator additions |

Same-asset swaps are rejected in the Deal contract on every plan as an anti-money-laundering control: a swap of a token for itself has no economic purpose. Same-asset shared vaults are part of the TrustBridge Vault product.

White-label operators receive 70% of the completion fee on their deals, split automatically on-chain.

---

## Chains

**Base first.** Other EVM chains can be added through per-chain configuration (Safe addresses, price sources, token whitelist), each followed by a chain-specific audit review, with no code changes. Non-EVM chains (such as Solana) and cross-chain deals are out of scope for v1.

---

## AML/CFT thresholds (UAE VARA)

| Threshold | Obligation |
|-----------|-----------|
| AED 3,500 (~$950 USD) | FATF Travel Rule: originator and beneficiary information required |
| AED 55,000 (~$15,000 USD) | Customer due diligence: full identity verification required |

These checks sit at the application / operator layer, not in the smart contracts. The Travel Rule applies from mainnet day one.

---

## Known limitations

See [KNOWN_LIMITATIONS.md](KNOWN_LIMITATIONS.md) for the full register of prototype limitations, pre-production requirements and resolved issues. TrustBridge publishes it openly as a matter of discipline.

---

## Roadmap

| Phase | Status | Milestone |
|-------|--------|-----------|
| Phase 1 — Validate | ✅ Complete | Prototype built, market validated |
| Phase 2 — Build | 🔄 In progress | Smart contracts to spec v1.2 → independent review → audit → Base Sepolia testnet |
| Phase 3 — Launch | ⏳ Planned | Mainnet with a guarded-launch deal cap, white-label operators |
| Phase 4 — Expand | ⏳ Planned | TrustBridge Vault, M-of-N, further EVM chains |

---

## Repositories

This is the **public specification repository**:

- `README.md` — this file
- `SMART_CONTRACT_SPEC.md` — function specification, parameters and security requirements
- `SECURITY.md` — security architecture and threat model
- `KNOWN_LIMITATIONS.md` — limitations, pre-production requirements, resolved issues
- `LICENSE` — MIT

Private repositories (available for review under NDA):

- **trustbridge-contracts**: the v1 smart contracts (Foundry). Private until audited, then published under MIT
- **trustbridge**: the React/JSX prototype

---

## Contact

**Craig Evans** — Founder, TrustBridge Protocol  
Dubai, UAE · October 2026  
[trustbridgeprotocol.xyz](https://trustbridgeprotocol.xyz) · craig@trustbridgeprotocol.xyz

**Responsible disclosure:** craig@trustbridgeprotocol.xyz  
Smart contracts are not yet deployed. No live funds are at risk.
