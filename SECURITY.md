# Security Architecture

**TrustBridge Protocol — Threat Model and Required Defences**

> This document is the security specification for TrustBridge Protocol. It defines known attack vectors, required defences, and immutable security decisions. It is published openly as a signal of intent and to hold the development team accountable to the stated architecture.
>
> **Version 4 — November 2026.** Aligned with [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md) v1.3. The contracts are now **built, tested and deployed on the Base Sepolia testnet** (2 November 2026; addresses in the README). Every contract-layer defence below is implemented and covered by tests — 331 tests, a whole-lifecycle invariant campaign, Slither with no high or medium findings, and a mutation check in which all 12 injected bugs were caught. **Not yet independently reviewed or audited; not deployed on mainnet.**

---

## Critical context — the Bybit attack (February 2025)

In February 2025, Lazarus Group (North Korean state actors) stole $1.5 billion from Bybit. They did not exploit the smart contract. The contract was correct. They injected malicious JavaScript into the Safe signing UI — the frontend interface Bybit's authorised signers used to approve transactions. Signers saw one transaction on screen and signed a completely different one. By the time anyone identified the discrepancy, the funds were gone.

**This is now the primary attack vector for any Safe-powered application that moves user funds.**

TrustBridge's `ConfirmDeliveryModal` is a direct architectural response. Before any fund release, both parties independently view a human-readable summary of the exact transaction — asset amounts, wallet addresses, deal reference — and must tick an acknowledgment checkbox before the confirm button activates. The modal cannot be dismissed by clicking outside or pressing Escape. It fires for every release action, for every party, for every deal, with no exceptions.

This is documented as an immutable design decision. It cannot be removed, bypassed, or simplified in any version of TrustBridge for any reason.

---

## Threat model

### CRITICAL — Smart contract layer

| Threat | Attack description | Required defence | Status |
|--------|-------------------|-----------------|--------|
| **Reentrancy** | Malicious contract calls back into the factory or settlement code before state is updated, draining the vault repeatedly | `ReentrancyGuard` on ALL fund-moving functions. State updated BEFORE all transfers (checks-effects-interactions). Non-negotiable on every function that moves assets. | Implemented and tested |
| **Price oracle manipulation** | Attacker manipulates price feed to trigger false price band breach or prevent legitimate release | Chainlink price oracle ONLY for on-chain band verification at `releaseFunds()`. CoinGecko is UI display only — never used in contract logic. | Implemented — non-negotiable |
| **Precision loss / integer overflow** | Rounding errors in fee calculation create exploitable edge cases | Solidity 0.8.x built-in overflow protection. Fee calculation uses basis points (integer arithmetic), rounded down in the user's favour. | Implemented and tested (property tests against an independent calculation) |
| **Token whitelist bypass** | Attacker uses a malicious ERC-20 token with non-standard behaviour | Approved token whitelist enforced at `createDeal()`. `SafeERC20` handles non-standard tokens including tokens with return value issues. Each deposit is checked so the Safe receives exactly the amount owed; fee-on-transfer tokens are refused. | Implemented and tested |
| **Front-running** | Attacker front-runs `releaseFunds()` to manipulate outcome | 2-of-2 requirement means attacker would need control of one party's wallet. Commit-reveal not required for this architecture. | Acceptable risk |
| **Fee tier spoofing** | Caller declares a cheaper plan tier to pay a lower fee | Free rate by default. Pro requires a voucher signed by the TrustBridge voucher key naming the caller's wallet; Enterprise requires an operator registry entry plus a signed approval from the operator's registered authorizer naming Party A. Rate and cap locked at creation. Voucher validity capped at 35 days on-chain. | Implemented and tested |
| **Voucher key compromise** | Attacker steals the Pro voucher signing key | Worst case: some new deals pay the Pro rate instead of Free. The key cannot touch any deposit or existing deal. Key rotated via the admin multisig. | Implemented and tested |
| **Parties bypass deal rules** | Both owners sign a Safe transaction directly, moving funds outside the module | TrustBridgeGuard blocks every owner-initiated transaction on a deal Safe, including changes to owners, threshold, modules or guard. Parties who want out together use mutual cancel. | Implemented and tested |
| **Frozen token address** | A stablecoin issuer freezes one party's address, making a joint refund fail | Refunds are independent per party. A failed transfer is recorded as pending and can be retried; the other party is always refunded. The swap itself is all-or-nothing, so nobody can end up with both assets. | Implemented and tested |
| **Stale or failed oracle data** | Chainlink feed is stale or unavailable at release | Staleness (the feed's published heartbeat), round-completeness and L2 sequencer-uptime checks on every price read. A stale feed blocks the price-dependent action rather than using an old price. Refunds, cancellation and expiry never depend on the oracle. | Implemented and tested |
| **Stranded deposit on cancellation** | Party A cancels after Party B has deposited, and Party B's funds are not returned | `cancelDeal()` and `claimRefund()` return every deposit held, whichever party made it. Covered by invariant tests. | Implemented and tested |
| **Griefing / DoS** | Party A creates many deals to prevent Party B from acting | Gas costs disincentivise spam. Future: rate limiting per address. | Known limitation — to be addressed in v2 |

---

### HIGH — Frontend / signing layer

| Threat | Attack description | Required defence | Status |
|--------|-------------------|-----------------|--------|
| **UI spoofing (Bybit vector)** | Malicious JS injected into signing UI — user signs different transaction than displayed | `ConfirmDeliveryModal` — mandatory human-readable summary before every release. Both parties. Every deal. Cannot be removed. | Implemented in prototype — must be preserved in production |
| **Phishing via typo domain** | Attacker registers trust-bridge.io, trustbr1dge.xyz etc. to harvest wallet signatures | Register defensive domains. HSTS on all TrustBridge domains. EIP-712 signed messages display domain name in wallet — mismatch is immediately visible. | Partially complete — additional domains to be registered before mainnet |
| **Malicious JS dependency** | Compromised npm package injects malicious code | SRI (Subresource Integrity) hashes on all external scripts. `npm audit` in CI pipeline. Dependabot for dependency updates. Minimal dependency footprint. | To be implemented in Phase 2 |
| **Fake invite link** | Attacker sends a spoofed invite link pointing to a phishing site | Invite links contain cryptographic deal reference. Party B pre-connection preview shows terms — mismatch between expected and displayed terms is detectable. | Partially implemented — full cryptographic signing in Phase 2 |
| **Invite link interception** | A third party obtains the invite or copies an acceptance from the network | The invite carries a one-time key; acceptance is a signature naming the acceptor's own wallet, so a copied acceptance cannot be reused by anyone else. Single use. Optional address lock. Party A must confirm the counterparty before the Safe is created or any deposit is possible. | Implemented and tested |

---

### MEDIUM — Infrastructure and configuration

| Threat | Attack description | Required defence | Status |
|--------|-------------------|-----------------|--------|
| **PlanSwitcher in production** | Prototype-only plan switching tool accidentally left in production build | `PlanSwitcher.jsx` must be removed entirely from production build. Plan determined by authenticated user account only. | Known prototype limitation — documented in Known Limitations Register |
| **Hardcoded deal IDs** | Prototype uses hardcoded #0042 — collision or confusion in production | Server-generated unique IDs from deal creation events before any production deployment. | Known prototype limitation |
| **In-memory dealStore** | Prototype state lost on page refresh — audit trail not persisted | Replace dealStore.js with Supabase/PlanetScale backend with row-level security before testnet. Append-only admin audit log. | Known prototype limitation |
| **Admin key compromise** | Attacker gains control of the configuration admin key | Admin powers are limited to settings for new deals (whitelist, fees, fee wallet, operator registry, voucher key, limits), each behind a 7-day public notice, plus an immediate pause that only stops new deals. No admin function can move, freeze, redirect or delay user funds or change an existing deal. Multi-sig admin wallet. No dispute admin key and no break-glass mode in v1. | Implemented and tested |
| **Payout redirection** | Attacker tricks a party into sending their proceeds to a different address | No alternative payout address exists in v1. Every payout goes to the party address recorded in the deal. | Implemented and tested |
| **Smart-contract wallet incompatibility** | A company using a Safe multisig as its party wallet cannot complete a deal | Contracts identify parties by calling address only (no `tx.origin` or personal-wallet-only checks). Tested with a 2-of-3 company Safe as Party A, Party B and both (23 tests). | Implemented and tested |
| **Operator collusion** | A white-label operator helps one party escape a deal they agreed to | Operators have no cancel, release or refund power over any deal. Only the parties (mutual cancel) or the rules (expiry, auto-release) end a deal. | Implemented and tested |
| **Fee wallet compromise** | Platform fee wallet is a single point of failure | Multi-sig fee wallet. Fee withdrawal requires multiple signatures. | Required before mainnet (testnet uses a single test wallet) |

---

### LOW — Protocol design

| Threat | Attack description | Required defence | Status |
|--------|-------------------|-----------------|--------|
| **Dust attack** | Attacker creates many tiny deals to pollute state | $10 USD minimum deal value enforced at `createDeal()` against Chainlink. Gas costs disincentivise at current fee levels. | Implemented and tested |
| **Concern spam** | Party raises repeated concerns to pressure the counterparty | "Raise a Concern" is a UI-only timestamped record with no contract call and no effect on funds or deal state. Rate limited to 1 per deal. | Implemented in prototype |
| **Time manipulation** | Miner manipulates block timestamp to affect expiry | Solidity `block.timestamp` is within ~15 seconds of real time — acceptable given the 15-minute minimum expiry. Not exploitable at TrustBridge deal durations. | Acceptable risk |

---

## Immutable security decisions

These decisions are permanent. They cannot be changed for any user, operator, plan tier, deal size, or business reason:

### 1. ConfirmDeliveryModal — never removable

Both parties must independently verify the exact transaction details before any fund release. The modal fires for every release action, for every party, on every deal. It cannot be dismissed without an explicit choice. It cannot be bypassed for any tier or deal type.

**Rationale:** This is TrustBridge's primary defence against the Bybit attack vector. The $1.5B loss happened because users trusted what the UI showed them without verifying. This modal forces verification.

### 2. Non-custodial architecture — absolute

TrustBridge cannot access, freeze, move, or confiscate user funds under any circumstance. No admin function exists that can override deal logic while a deal is in `LOCKED` state.

Funds move only in these ways, all governed by contract rules agreed at deal creation:
- **Both parties confirm** — the atomic swap executes
- **Single-party auto-release** — one party confirms and the other does not respond within the window
- **Mutual cancel** — both parties agree; each deposit is returned
- **Party A cancels before full funding** — each deposit is returned
- **Expiry or an unresolved price breach** — `claimRefund()` is permissionless and returns each deposit

There is no dispute resolution function and no emergency break-glass mode in v1. No person at TrustBridge decides any outcome. Kleros decentralised arbitration is planned for v2.

### 3. Chainlink oracle — on-chain only

CoinGecko and similar off-chain price APIs can be manipulated, rate-limited, or go offline. Chainlink's decentralised oracle network is the only acceptable source for on-chain price band verification at `releaseFunds()`. CoinGecko is acceptable for UI display only and is clearly marked as such in all documentation.

### 4. Checks-effects-interactions — always

State is always updated before any external call or token transfer. No exceptions on any fund-moving function. This is the standard protection against reentrancy and must be enforced by the auditing firm as a pass/fail criterion.

### 5. Time-lock auto-refund — no admin override

When the deal timer expires, `claimRefund()` is callable by anyone and returns both deposits automatically. No admin function can prevent this, and no admin function can extend the timer. The expiry can never be extended by one party alone.

---

## Audit requirements

Before any mainnet deployment, TrustBridge Protocol requires a professional security audit covering:

**Scope:**
- All contracts in `trustbridge-contracts/src/`: `TrustBridgeFactory`, the `DealSettlement` library, `TrustBridgeModule`, `TrustBridgeGuard`, `TrustBridgeSafeSetup`, `TrustBridgeConfig` and `ChainlinkPriceAdapter` (the testnet-only test token is out of scope)
- State machine integrity — all valid and invalid transitions
- All fund-moving functions with special attention to reentrancy
- Fee calculation edge cases
- Oracle integration and manipulation resistance
- Access control (configuration admin functions, party-specific functions) — confirming no admin path to user funds
- Auto-release path and all refund and cancellation paths, including partial-funding states
- TrustBridgeGuard: no owner-initiated transaction can move funds or change a deal Safe's configuration
- Invite signatures, Pro vouchers and the operator registry
- Settings timelock and the pause switch (new deals only)
- Event emission completeness

**Required audit firms:** CertiK, Trail of Bits, OpenZeppelin, or equivalent recognised firm. Self-audits are not acceptable.

**Pass criteria:**
- Zero critical findings unresolved
- Zero high findings unresolved
- All medium findings resolved or documented with accepted risk
- Audit report published publicly

**The prototype will not move to production without a clean audit report.**

---

## Known prototype limitations (security-relevant)

The following prototype limitations are security-relevant and must be resolved before any real funds are involved:

| Item | Risk | Resolution |
|------|------|-----------|
| Simulated wallet connection | No real transaction signing | Replace with Wagmi + viem in Phase 2 |
| In-memory dealStore | Deal state lost on refresh | Replace with authenticated Supabase backend |
| Hardcoded deal IDs | No uniqueness guarantee | Server-generated IDs from contract events |
| PlanSwitcher.jsx present | Any user can switch to Enterprise | Remove entirely from production build |
| CoinGecko price display | Simulated — not live in prototype | Wire to live CoinGecko API for display; Chainlink for contract logic |
| No KYC threshold enforcement | Deals above UAE VARA threshold not flagged | KYC process design required before Pro plan mainnet. Enforced at the application / operator layer, not in the contracts (D-05) |
| Dispute UI code dormant | Legacy dispute screens remain in the codebase, greyed out | Not active in v1. Removed or kept dormant until Kleros integration in v2 |

See [KNOWN_LIMITATIONS.md](KNOWN_LIMITATIONS.md) for the complete register.

---

## Data security

| Data type | Retention | Storage | Encryption |
|-----------|-----------|---------|-----------|
| Deal chat messages | 2 years (completed) / 8 years (where a concern was raised) | Supabase (off-chain) | AES-256 at rest |
| Raise a Concern records | 8 years | Supabase (off-chain) | AES-256 + SHA-256 fingerprint |
| Platform fee records | 8 years | Append-only audit log | Standard |
| User email (optional) | Until deletion request | Supabase | Standard |
| IP addresses | 90 days | Hashed only | SHA-256 |
| Real identity / KYC | Not collected at MVP | — | — |
| Behavioural analytics | Never collected | — | — |

All on-chain data is public by nature. Users should be aware that deal creation, deposit, release, refund and cancellation events are permanently visible on the blockchain.

---

*Version 4 · Last updated: November 2026 · TrustBridge Protocol · trustbridgeprotocol.xyz*

*Responsible disclosure: If you identify a security vulnerability in TrustBridge Protocol, please contact craig@trustbridgeprotocol.xyz before public disclosure. Smart contracts are deployed on the Base Sepolia testnet only — there are no live funds at risk.*
