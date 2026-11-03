# Known Limitations Register

**TrustBridge Protocol — v5 — November 2026**

> This document catalogues all known limitations, prototype-only behaviours, and pre-production requirements in the current TrustBridge build. It is published openly to demonstrate technical awareness and professional discipline.
>
> **v5 update:** aligned with [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md) v1.3. The v1 smart contracts are now **built, tested and deployed on the Base Sepolia testnet** (not reviewed, not audited, not on mainnet). Section 3 statuses updated; the same-asset row in Section 4 corrected (rejected in the Deal contract on every plan); new **Section 7 — contract-level limitations** (CL-01 to CL-12).
>
> **v4 update:** aligned with [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md) v1.2. All architecture decisions confirmed (R-16). Contract scope is now 7 core functions plus 4 supporting functions. Adds multi-chain configuration (R-17), guarded launch and pause switch (R-18), Enterprise agent seats and reporting (R-10), two prototype UI items (U-09, U-10), and a ninth immutable decision (I-09).

**Scope:** the React/JSX prototype (Sections 1–5) and the v1 smart contracts deployed on the Base Sepolia testnet (Section 7).

---

## How to read this document

| Category | What it means |
|----------|--------------|
| **Prototype infrastructure** | Deliberate simplifications for rapid prototyping. Expected. Will be replaced in Phase 2. |
| **Known UI limitations** | Present in the current prototype. Known and understood. Do not affect the core deal flow. |
| **Pre-production requirements** | Not yet built. Must exist before any mainnet deployment with real funds. |
| **Plan tier limits** | Confirmed limits, enforced at `createDeal()` and in the UI. |
| **Confirmed resolved** | Bugs identified and fixed during development. |
| **Immutable decisions** | Not limitations — permanent architectural decisions that will never change. |

---

## Section 1 — Prototype infrastructure

| Ref | Component | Limitation | Production replacement |
|-----|-----------|-----------|----------------------|
| P-01 | `dealStore.js` | Prototype state only — not an authenticated, persistent backend. | Supabase or PlanetScale with row-level security. |
| P-02 | Dashboard, CreateDeal | Hardcoded deal ID `#0042` throughout. | Server-generated unique IDs from contract events. |
| P-03 | `PlanSwitcher.jsx` | Prototype-only plan switching — does not exist in production. | Remove entirely. Plan from authenticated user account. |
| P-04 | `Dashboard.jsx` | Simulated wallet connection via local React state. | Real wallet via Wagmi + viem. MetaMask and WalletConnect. |
| P-05 | Multiple screens | Hardcoded mock wallet addresses. | Real addresses from connected wallet provider. |
| P-06 | Share links | Invite link hardcoded to non-functional URL. | Real single-use invite links with cryptographic token. |

---

## Section 2 — Known UI limitations

| Ref | Component | Limitation | Resolution target |
|-----|-----------|-----------|------------------|
| U-01 | `PriceBandLiveIndicator` | Always shows ETH/USDT regardless of deal assets. Simulated movement hardcoded. | Before operator onboarding. Live CoinGecko for display; Chainlink for contract logic. |
| U-02 | `Dashboard.jsx` | Party B completed deals not shown as "Received". | Mainnet launch. Requires backend persistence. |
| U-03 | Multiple screens | Progress tracker only on Create Deal success panel. | Before operator onboarding. |
| U-04 | `LockFund.jsx` | Escrow Summary header still shows `Deal #0042`. | Before first operator demo. Resolves with real deal IDs. |
| U-05 | `DealRoom.jsx` | Chat not stored in an encrypted backend. | Mainnet launch. |
| U-06 | Multiple screens | No wallet-connected lock state on Deal Room, Lock & Fund, Execute. | Before operator onboarding. |
| U-07 | `Dashboard.jsx` | No search, filter or pagination. Unusable above ~10 active deals for Pro users. | Before Pro plan operator onboarding. Server-side search and filter. |
| U-08 | `Execute.jsx` | Legacy dispute screens remain in the codebase, dormant and greyed out. "Raise a Concern" is the active v1 feature. | Kept dormant until Kleros integration in v2. |
| U-09 | `CreateDeal.jsx` | Counterparty is entered as a wallet address. v1 design is an invite link with an optional address lock and Party A confirmation (D-02). | Before operator onboarding. |
| U-10 | Deal Room / Party B preview | Auto-release labels read "after both confirm". Behaviour is single-party (smoke-tested June 2026); the wording needs correcting. | Before first operator demo. |

---

## Section 3 — Pre-production requirements

| Ref | Area | Requirement | Phase |
|-----|------|------------|-------|
| R-01 | Smart contracts | **Built, tested and deployed on Base Sepolia (November 2026):** factory, settlement library, module, guard and Chainlink price adapter, all 7 core and 4 supporting functions; 331 tests, Slither clean of high and medium findings. Still required: independent senior Solidity / Safe review and the audit (R-02). See [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md). | Built — review next |
| R-02 | Security audit | No audit conducted. Independent professional audit mandatory before mainnet. Budget: $15,000–$25,000. | Phase 2, before mainnet |
| R-03 | Price oracle | Chainlink integration built, with staleness, round-completeness and L2 sequencer checks; running against the live Base Sepolia feeds. CoinGecko for UI display only. | Built — testnet |
| R-04 | Real wallet integration | Wagmi + viem for wallet connection and transaction signing. No real transactions in prototype. | Phase 2 |
| R-05 | Backend persistence | Supabase or PlanetScale with row-level security. FATF Travel Rule field in schema from day one — **populated for all deals above AED 3,500 (~$950 USD)**. Admin audit log as append-only table. | Phase 2 |
| R-06 | Email notifications | Deal event notifications. No notification system in prototype. | Phase 2 |
| R-07 | Domain and SSL | `trustbridgeprotocol.xyz` registered May 2026. Typo domains before mainnet. | trustbridgeprotocol.xyz complete |
| R-08 | KYC / AML thresholds | **Two separate VARA thresholds:** (1) FATF Travel Rule: **AED 3,500 (~$950 USD)** — originator name, wallet address and beneficiary info required above this level; Free plan deals ($5,000) can exceed it, so it applies from mainnet day one. (2) CDD occasional transaction: **AED 55,000 (~$15,000 USD)** — enhanced due diligence above this level. The CDD check is enforced at the application / operator layer, not in the contracts (D-05). | Phase 3 — specified before Phase 2 |
| R-09 | VARA regulatory classification | Non-custodial DLT infrastructure argument — not a VASP. Pre-application VARA enquiry and formal UAE legal opinion required before mainnet. | Before mainnet |
| R-10 | Enterprise operator onboarding | No onboarding flow exists. Operators need: domain configuration, fee split setup, brand configuration, an operator dashboard with revenue metrics, agent seats with roles, per-agent deal attribution, consolidated reporting and export for internal audit, and an Operator Agreement defining the liability split. Company wallets (including a company Safe multisig) are used across all deals. | Spec: Phase 2. Build: Phase 3 |
| R-11 | Approved token whitelist | USDC, USDT, ETH, WBTC, DAI, WETH, enforced at `createDeal()`. UI shows a verified badge for approved tokens and a warning for unverified addresses. Enterprise token additions via a governance process (to be defined). | Built — enforced on-chain |
| R-12 | Minimum deal value | $10 USD equivalent enforced at `createDeal()` against Chainlink. Prevents gas griefing. | Built — enforced on-chain |
| R-13 | Fee precision at large deal sizes | Always round down in favour of the user; fee taken from what Party A receives; cap fixed in token units at creation; wei precision; misconfigured fee wallet reverts rather than failing silently. | Specified. Build: Phase 2 |
| R-14 | Renegotiation connection loss | If a renegotiation window expires without acceptance or decline, the deal cancels and both deposits return. No ambiguous mid-renegotiation state. | Built — tested |
| R-15 | Public operator registry | A public list of approved white-label operators on trustbridgeprotocol.xyz, protecting users against operator impersonation. | Phase 3 |
| R-16 | Architecture decisions | **Confirmed October 2026:** Safe per deal + module + guard (D-01), signature-bound invite link with optional address lock and Party A confirmation (D-02), on-chain confirmations with permissionless auto-release trigger (D-03), Free / Pro voucher / operator registry fee tiers (D-04), compliance at the application layer (D-05), renegotiation window on price breach (D-06), expiry changes only with both signatures (D-07), no break-glass mode (D-08), configurable EVM chains (D-09). | Confirmed — built to spec v1.3 |
| R-17 | Multi-chain configuration | Base first. Other EVM chains added through per-chain configuration (Safe addresses, price sources, whitelist), each followed by a chain-specific audit review. Non-EVM chains and cross-chain deals out of scope; users are pointed to independent swap and bridge services through verified links. | Phase 2 design; additional chains Phase 3+ |
| R-18 | Guarded launch and pause | A maximum deal value applies for the first months of mainnet. An admin pause stops new deals only; existing deals always continue. | Built — testnet cap $50,000; mainnet cap to decide |

---

## Section 4 — Plan tier limits

| Parameter | Free | Pro | Enterprise |
|-----------|------|-----|-----------|
| Maximum deal value | $5,000 | $500,000 | Unlimited |
| Maximum timer | 7 days | 30 days | 365 days |
| Long-duration acknowledgment | N/A — 7-day max | Deals over 7 days (confirmed) | Deals over 30 days (confirmed) |
| Single-party auto-release window | 15 minutes | 15 minutes | Configurable — default 2 hours |
| Same-asset deals | Not permitted in Deal | Not permitted in Deal | Not permitted in Deal |
| Minimum deal value | $10 USD equivalent | $10 USD equivalent | $10 USD equivalent |
| Approved tokens | USDC, USDT, ETH, WBTC, DAI, WETH | Same | Same + operator additions |

Same-asset swaps are rejected by the Deal contract on every plan as an anti-money-laundering control (a swap of a token for itself has no economic purpose). Same-asset shared vaults belong to the TrustBridge Vault product.

---

## Section 5 — Confirmed resolved

| Ref | Component | Bug description | Status |
|-----|-----------|----------------|--------|
| F-01 | `TimeLimitSelector.jsx` | Timer showed 24h regardless of chip selected. | Fixed May 2026 |
| F-02 | `dealStore.js` | Adjust Deal showed original amounts, not the latest version. | Fixed May 2026 |
| F-03 | `LockFund.jsx` / `BreachResponseFlow.jsx` | Black screen on price band breach during renegotiation. | Fixed May 2026 |
| F-04 | `CreateDeal.jsx` | ExtremeConditions acknowledgment didn't unlock Create Deal. | Fixed May 2026 |
| F-05 | `LockFund.jsx` | CXL reference changed on every checkbox tick. | Fixed May 2026 |
| F-06 | `CreateDeal.jsx` / `TimeLimitSelector.jsx` | Custom time validation accepted exactly 24h and 60min. | Fixed May 2026 |
| F-07 | `LockFund.jsx` / `Execute.jsx` | Renegotiated amounts not flowing downstream. | Fixed May 2026 |
| F-08 | `LockFund.jsx` / `Execute.jsx` | Timer and price band not freezing on cancellation. | Fixed May 2026 |
| F-09 | `DealSuccessPanel.jsx` / `LockFund.jsx` | Deal expiry countdown dropped the days component. | Fixed May 2026 |
| F-10 | Dashboard / CreateDeal | No wrong-network detection. Base network check and warning banner added. | Fixed May 2026 |
| F-11 | `Execute.jsx` | No auto-release explanation. Countdown and plain-English banner added. | Fixed May 2026 |
| F-12 | `CreateDeal.jsx` | Same-asset deals allowed on Free with a meaningless price band. Error block and education note added. | Fixed May 2026 |
| F-13 | `DealRoom.jsx` | No Party B join notification. Toast and system chat message added. | Fixed May 2026 |
| F-14 | `TimeLimitSelector.jsx` | No tier-based timer maximums. Free 7d / Pro 30d / Enterprise 365d enforced. | Fixed May 2026 |
| F-15 | `CreateDeal.jsx` | No token whitelist UI. Verified badges and unverified warning added. | Fixed May 2026 |
| F-16 | `CreateDeal.jsx` | No minimum deal value. $10 minimum added. | Fixed May 2026 |

---

## Section 6 — Immutable design decisions

| Ref | Decision | Rationale |
|-----|----------|-----------|
| I-01 | 2-of-2 multisig — cannot be waived for any user, deal or circumstance | No party can move funds at will. The only one-sided outcomes are rule-based (auto-release after the window, refund at expiry), agreed at deal creation and enforced by contract logic. |
| I-02 | Time-lock auto-refund — no admin override | Admin override creates a centralised attack surface. Refund must be unstoppable. |
| I-03 | Non-custodial — TrustBridge cannot access, freeze or move user funds | Changes regulatory classification and destroys the core value proposition. |
| I-04 | Fee on completion only — never on creation, expiry or cancellation | Aligns TrustBridge incentives with users. Failed deals never cost platform fees. |
| I-05 | ConfirmDeliveryModal — mandatory, both parties, every deal, never removable | Direct architectural response to the Bybit $1.5B UI spoofing attack (February 2025). |
| I-06 | Chainlink oracle — on-chain price verification only, never CoinGecko | CoinGecko can be manipulated. Chainlink is manipulation-resistant and verifiable. |
| I-07 | Checks-effects-interactions — state updated before all transfers | Protection against reentrancy. Non-negotiable on all fund-moving functions. |
| I-08 | No discretionary dispute resolution in v1 | Preserves the non-custodial position under UAE VARA. Kleros decentralised arbitration in v2. |
| I-09 | No operator or admin power over live deals — no cancel switch, no break-glass mode | Exits are held by the parties themselves (mutual cancel) and by the rules (expiry, auto-release). Any intervention power would create an abuse and regulatory risk. |

---

## Section 7 — Contract-level limitations

What the v1 contracts deliberately do not do, or cannot prevent. IDs are stable; "Raised" gives the build stage. Open questions O-01 and O-02 are put to the independent reviewer.

| ID | Limitation | Why it is accepted | Mitigation | Raised |
|---|---|---|---|---|
| CL-01 | **Tokens sent straight to a deal Safe** (not through `depositAssets`) can never be recovered. The guard blocks owners; the module only pays recorded deal amounts | Any recovery path is a new way for funds to leave a deal Safe — the thing the design exists to prevent | The app only ever deposits through the contract; user guidance | Stage 2 |
| CL-02 | **A Pro voucher can't be revoked early.** Worst case it runs to its 35-day end | Revocation needs a live check against a registry or the voucher key, adding a dependency to every createDeal | 35-day on-chain maximum (B-06); a compromised voucher key only grants lower fees, never funds; key rotatable with 7-day notice | Stage 2 |
| CL-03 | **Issuer freezes the deal Safe itself** (not a party's wallet). Every transfer of that token out of the Safe fails, so that leg can be neither released nor refunded until the issuer lifts the freeze. The other token's leg is still refundable at expiry (refunds are independent) | No contract can override an issuer's freeze. The deal Safe is a fresh address that never transacts on its own, so a targeted freeze is unlikely | Refunds stay on record and are retryable forever; documented for operators and users | Stage 5 |
| CL-04 | **Breach-timing escape (O-01).** Anyone can call release at a moment the price is outside the band, forcing renegotiation (Pro/Enterprise) or cancellation (Free). A party who regrets the deal could use a brief spike to get out. Nobody can take funds: deposits return to their owners | Inherent to D-06 (band checked at release). Options: accept and document; require the breach to persist; ask the reviewer | **Open — reviewer opinion requested** (Register O-01) | Stage 4 |
| CL-05 | **Off-market Free deals (O-02).** A Free deal agreed more than 15% from market will always cancel at release | Free's band is fixed at ±15% (D-06) | App warning at Create Deal (ExtremeConditionsWarning); optional on-chain check at creation in spec v1.3 — **Open** (Register O-02) | Stage 4 |
| CL-06 | **Late first confirmation.** If a party confirms delivery less than the auto-release window before expiry, the deal expires before auto-release can run and both are refunded | Expiry beats everything (B-23) — the simpler, safer rule | App warns when confirming close to expiry | Stage 4 |
| CL-07 | **Price unavailable at release** (stale Chainlink feed or sequencer down) blocks release on deals with a price band until a live price returns; if that outlasts expiry, both are refunded | Releasing without a price would defeat the band (D-06) | Confirmations are kept (B-20); anyone can complete release later; refunds never need a price | Stage 4 |
| CL-08 | **No on-chain identity, KYC or sanctions screening.** The contract can't tell who controls a wallet | By design (D-05): screening sits at the app / operator layer, not in the contract | Operator authorizer approves wallets for Enterprise deals; app-level KYC | Stage 2 |
| CL-09 | **Deposit and fee tokens must be plain ERC-20s** on the whitelist (USDC, USDT, WBTC, DAI, WETH; ETH is used as WETH — CL-10). Fee-on-transfer, rebasing and callback tokens are unsupported | The deposit receipt check rejects fee-on-transfer; rebasing would break "Safe holds exactly the record" | Whitelist changes need the admin's 7-day notice | Stage 1–3 |
| CL-10 | **Native ETH is not supported**; ETH must be wrapped (WETH) | The module can only make ERC-20 `transfer` calls — one less attack surface | App wraps ETH before deposit | Stage 2 |
| CL-11 | **Two parties only in v1**, although `partyCount` / `threshold` are stored for future M-of-N (B-04) | v1 scope | Storage ready; logic would need review | Stage 2 |
| CL-12 | **Deal Safe owner addresses are public on-chain** from the moment the Safe deploys | Safe requires owners on-chain | Partial-address display in the app is negotiation-stage privacy only | Baseline |

---

## For grant reviewers and developers

Sections 1 and 2 are expected prototype behaviour. Section 3 is the Phase 2 scope, with R-16 recording the confirmed architecture decisions. Section 5 shows active development discipline. Section 6 confirms the security decisions are fixed. Section 7 lists what the contracts themselves cannot prevent. The stress test found no fatal architectural problems — the gaps were specification or UI issues.

---

*Document v5 — November 2026 · Craig Evans (founder) · trustbridgeprotocol.xyz*  
*Responsible disclosure: craig@trustbridgeprotocol.xyz — smart contracts on the Base Sepolia testnet only, no live funds at risk*
