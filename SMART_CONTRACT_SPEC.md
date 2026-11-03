# TrustBridge Protocol — Smart Contract Specification

**Version:** 1.3 — November 2026  
**Status:** Built to this specification and **deployed and verified on the Base Sepolia testnet** (2 November 2026). **Not reviewed, not audited, not deployed on mainnet.**  
**Target chain:** Base (primary). Other EVM chains added by configuration (see D-09)  
**Architecture:** One Safe 2-of-2 smart account per deal, with a TrustBridge module and guard

---

## Overview

TrustBridge Protocol v1 is designed to provide non-custodial deal-room escrow. Each deal gets its own Safe smart account, owned by the two parties. A small TrustBridge module can move funds out of that Safe in only a handful of rule-based ways, and a TrustBridge guard blocks every other movement. No TrustBridge-controlled key, and no single party, can move user funds outside the rules both parties agreed at deal creation.

### What changed in v1.3

v1.3 records how v1 was actually built. The architecture (D-01 to D-09) is unchanged. Every refinement made while building was recorded as a numbered build decision (A-01 to A-04, B-01 to B-37) and is now written into this specification:

- **Inputs tightened** — the auto-release window is no longer a `createDeal()` input (B-03); `partyCount` + `threshold` replace a signers list (B-04); Enterprise needs the operator's signed approval (B-02); deposit has no amount parameter (B-11); delivery confirmation carries the terms hash (B-19)
- **New on-chain checks** — same asset on both sides rejected on every plan (B-05); plan deal value limits (B-08); single-use invite keys (B-07); 35-day maximum Pro voucher (B-06); confirmations must match the contract's own hash of the current terms (B-09)
- **Amendments and renegotiation fully specified** — B-10 and B-12 to B-17, B-22
- **Money out fully specified** — fee from Party A's proceeds (B-01); price band definition (B-21); all-or-nothing release (B-26); feed down at release (B-20); expiry beats everything (B-23); mutual cancel requests (B-24); on-chain cancellation reference (B-25); operator share of the fee charged (B-27)
- **Settings safety** — L2 sequencer check (A-01); hard admin ceilings (A-02); scheduled changes lapse (A-03); price adapter has no admin (A-04)
- **Deployment** — SafeL2 singleton (B-30); settlement library at a fixed address (B-31); sequencer feed required on Base mainnet (B-36); staleness limit = the feed's published heartbeat (B-37)

**Function count:** 7 core lifecycle functions plus 4 supporting functions — all 11 built.

---

## v1 dispute resolution architecture

TrustBridge Protocol v1 does **not** include `openDispute()` or `resolveDispute()`. A dispute mechanism in which a person decides who receives funds would make TrustBridge a custodian for the duration of the dispute and would compromise the non-custodial position under UAE VARA.

Funds move only in these ways, all governed by contract rules agreed at deal creation:
- **Both parties confirm delivery** → atomic swap
- **One party confirms, the other does not respond within the window** → single-party auto-release
- **Both parties agree to cancel** → each deposit returned
- **Party A cancels before the deal is fully funded** → each deposit returned
- **Expiry, or an unresolved price breach** → `claimRefund()` returns each deposit

"Raise a Concern" exists in the UI as a timestamped record only. It makes no contract call. Kleros decentralised arbitration is on the v2 roadmap.

---

## Contracts

| Contract | Role | Holds funds? |
|----------|------|--------------|
| `TrustBridgeFactory` | Validates and records new deals, locks each deal's rules at creation, deploys each deal's Safe. Runs all 11 functions | Never |
| `DealSettlement` | Linked library holding every money-out path: price band, atomic swap, fees, independent refunds, amendments (B-18) | Never |
| Deal Safe (one per deal) | Official, unmodified Safe v1.4.1 proxy (SafeL2 singleton on L2s — B-30). Owners: Party A and Party B. Threshold: 2 | Yes — that deal's deposits only |
| `TrustBridgeModule` | Enabled on every deal Safe. The only path for funds to leave; pays only Party A, Party B or the recorded fee wallets | Never |
| `TrustBridgeGuard` | Set on every deal Safe. Blocks all owner-initiated Safe transactions, including changes to owners, threshold, modules or guard | Never |
| `TrustBridgeSafeSetup` | Enables the module and guard inside Safe setup; setup succeeds only while the factory is deploying that deal (front-running protection) | Never |
| `TrustBridgeConfig` | On-chain settings for new deals: 7-day notice for every change, immediate pause of new deals only | Never |
| Price source adapter | Swappable price component. On Base: `ChainlinkPriceAdapter` — fixed feeds, no admin (A-04) | Never |

Deal Safes are created through Safe's official proxy factory from the official audited Safe contracts. Each deal's funds are isolated in its own Safe: a problem affecting one deal cannot reach another deal's funds.

ETH is handled as WETH; the application wraps ETH before deposit. The contracts handle plain ERC-20 tokens only.

### Party wallets

- Parties keep using the **same wallet for every deal** — only the vault (the deal's Safe) is new per deal.
- A party can be a personal wallet **or a smart-contract wallet**, including a company's own Safe multisig (for example, an OTC desk whose directors approve each action internally). Contracts identify parties by the calling address only, never by `tx.origin`. Covered by 23 tests with a 2-of-3 company Safe as Party A, Party B and both.
- **Payouts always go to the party address recorded in the deal.** There is no alternative payout address in v1 — a redirect field is a common target for manipulation.

---

## Deal state machine (v1)

```
createDeal()           acceptDeal()        confirmCounterparty(yes)    confirmTerms() ×2
PENDING        →       JOINED        →     NEGOTIATING          →      AGREED
                         │ confirmCounterparty(no) → back to PENDING (new invite)

depositAssets()        depositAssets()
AGREED  →  FUNDED_PARTIAL  →  LOCKED

From LOCKED:
  confirmDelivery() ×2 → release                → COMPLETED
  confirmDelivery() ×1 + window → triggerAutoRelease() → COMPLETED
  price breach at release (Pro / Enterprise)  → RENEGOTIATING
  price breach at release (Free)              → CANCELLED (refunds paid)

From RENEGOTIATING:
  amendTerms() by both parties  → LOCKED (confirmations reset)
  window lapses → claimRefund() → CANCELLED

From any state before COMPLETED:
  cancelDeal() by both parties  → CANCELLED
  expiry reached → claimRefund() → EXPIRED

From PENDING, JOINED, NEGOTIATING, AGREED or FUNDED_PARTIAL:
  cancelDeal() by Party A alone → CANCELLED
```

Deal expiry is set at `createDeal()` and changes only through `amendTerms()` signed by both parties (D-07). **Expiry always takes precedence (B-23):** once reached, release, auto-release, amendment and cancellation stop, and only `claimRefund()` works.

FUNDED_PARTIAL applies whenever any money is in the Safe but not both legs are fully funded, including after an amendment raised a party's amount (B-16).

---

## Core functions (7)

### 1. `createDeal(params, tierProof)` — Party A

| Parameter | Validation |
|-----------|-----------|
| `assetOffered`, `amountOffered` | Token on this chain's whitelist |
| `assetRequested`, `amountRequested` | Token on this chain's whitelist. **Must differ from `assetOffered` on every plan** (B-05 — anti-money-laundering) |
| `expiry` | ≥ now + 15 minutes and ≤ the plan's maximum, counted from creation (Free 7d / Pro 30d / Enterprise 365d) |
| `priceBandBps` | Free: 0 or 1500 (±15%). Pro / Enterprise: party-chosen, 0 = off |
| `inviteSigner` | Address of a one-time invite key (D-02). Each key can be used once across all deals (B-07). Required unless `lockedCounterparty` is set |
| `lockedCounterparty` | Optional. If set, only this address can accept |
| `partyCount`, `threshold` | v1: 2 and 2. Stored for future M-of-N (B-04) |
| `tierProof` | Empty = Free. Pro: voucher signed by the voucher key, naming the caller, valid ≤ 35 days (B-06). Enterprise: registered operator id plus the operator authorizer's signed approval naming Party A (B-02). Never both |

The auto-release window is **not** an input (B-03): Free and Pro use the plan's 15 minutes; Enterprise uses the operator's registered window (default 2 hours, can be off).

**Checks:** new deals not paused · deal value ≥ $10 · value of the larger leg ≤ the plan limit (Free $5,000 / Pro $500,000 / Enterprise unlimited — B-08) and ≤ the guarded-launch cap; the lower limit wins · fee tier resolved (D-04).  
**Effects:** all deal rules locked for the life of the deal, including the fee rate and the fee cap converted into the requested token at today's price (B-01) · terms version 1 · state → PENDING · emit `DealCreated`.

### 2. `acceptDeal(dealId, inviteSig)` — Party B

Two ways to join (D-02):
- **Invite link:** the link carries a one-time invite key. Party B's app signs the EIP-712 message "accept deal X for wallet Y" with it (`acceptDigest()`), and the contract checks the signature against `inviteSigner`. Because the signature names Party B's own wallet, someone who copies it from the network cannot use it to join in their place.
- **Address lock:** if `lockedCounterparty` is set, the caller must be that address.

**Effects:** Party B recorded · state → JOINED · emit `CounterpartyJoined`. Does not change expiry.

### 3. `confirmTerms(dealId, termsHash)` — each party

`termsHash` must equal `termsHashOf(dealId)` — the contract's own EIP-712 hash of the current on-chain terms, including the terms version (B-09). If anything changed after the app showed the terms, the hashes differ and the confirmation is refused, so nobody can agree to terms they were not shown. When both parties have confirmed the same version: state → AGREED · emit `TermsConfirmed`.

### 4. `depositAssets(dealId)` — each party

**No amount parameter (B-11):** the contract works out what the caller owes (the full amount, or the difference after an amendment raised it) and pulls exactly that into the deal's Safe via `SafeERC20` (token approval needed first). It then checks the Safe received exactly that amount, so fee-on-transfer or otherwise unusual tokens are refused. First party fully funded → FUNDED_PARTIAL; both → LOCKED. Emit `AssetDeposited`, then `VaultLocked`.

### 5. `releaseFunds(dealId)` — anyone, once both parties have confirmed delivery

Normally completed inside the second `confirmDelivery()`; callable by anyone afterwards.
- `nonReentrant`
- **Price band check** if `priceBandBps > 0` (B-21): the gap between the two legs' live USD values, as a percentage of the requested leg; exactly at the limit is allowed. A breach opens RENEGOTIATING (Pro / Enterprise) or cancels with refunds (Free) — D-06. Emits `PriceBandBreach` with both values and the gap
- **No live price** (stale feed or sequencer down): reverts with no state change — try again later (B-20)
- **State → COMPLETED before any transfer**
- **Fee (B-01):** completion fee = `amountRequested` × locked rate, capped at the locked cap in token units, **rounded down**. It comes out of what Party A receives; Party B receives the full `amountOffered`
- **Operator share (B-27):** on Enterprise deals the operator receives 70% of the fee actually charged, rounded down; TrustBridge receives the remainder
- **All-or-nothing (B-26):** if any payout fails, the whole release is undone; the deal stays LOCKED and refunds still work at expiry
- Emit `FundsReleased`

### 6. `claimRefund(dealId)` — anyone

Available once expiry is reached (from any state before COMPLETED), or once a renegotiation window lapses unresolved.
- `nonReentrant`
- **State → EXPIRED or CANCELLED before any transfer**
- Each party's deposit returned **independently**. If one transfer fails (for example, a token issuer has frozen that address), that refund stays recorded as unpaid and can be retried later by calling `claimRefund()` again. The other party's refund is unaffected
- Zero fee, always. Never depends on the price source
- Emit `RefundPaid` per party (or `RefundPending` if a transfer failed), and `DealExpired` on expiry

### 7. `cancelDeal(dealId)` — Party A alone, or both parties together

- **Party A alone:** any state before LOCKED. Party B cannot cancel alone (B-24)
- **Both parties (mutual cancel):** any state before COMPLETED and before expiry. The first call records a cancel request (`CancelRequested`), valid only for the current terms version; the second party's call executes. Confirming delivery withdraws your own pending request (B-24)
- `nonReentrant` · **state → CANCELLED before any transfer** · independent refunds as in `claimRefund()` · zero fee · emit `DealCancelled` with an on-chain cancellation reference derived from chain, factory and deal number; the app shows it as the short CXL code (B-25)

---

## Supporting functions (4)

### 8. `confirmCounterparty(dealId, approve, newInviteSigner)` — Party A

After Party B joins, Party A confirms "this is my counterparty" (D-02).
- **Yes:** the factory deploys the deal's Safe (owners A and B, threshold 2, module enabled, guard set) · state → NEGOTIATING · emit `CounterpartyConfirmed`
- **No:** Party B is removed and a fresh one-time `newInviteSigner` must be supplied (unless the deal has an address lock) · state → PENDING · emit `CounterpartyRejected`

No deposit is possible before this confirmation.

### 9. `confirmDelivery(dealId, termsHash)` — each party, state LOCKED

`termsHash` must equal `termsHashOf(dealId)` (B-19), so a confirmation made against a spoofed or out-of-date screen is refused. Records the party's confirmation and timestamp on-chain (D-03) and withdraws that party's own pending cancel request. The first confirmation starts the auto-release window (if enabled). The second completes the release in the same transaction; if no live price is available at that moment, the confirmation still counts and anyone can complete the release later with `releaseFunds()` (B-20). Emit `DeliveryConfirmed`.

### 10. `triggerAutoRelease(dealId)` — anyone

Requires: auto-release enabled · exactly one party has confirmed · the window has passed since that confirmation · not expired. Executes the same path, checks and fee rules as `releaseFunds()`. Callers gain no power by calling — they can only trigger the outcome the rules already decided. Emit `AutoReleaseTriggered`.

### 11. `amendTerms(dealId, newAmountOffered, newAmountRequested, newExpiry)` — both parties

Each party calls with identical new values; the first call records a proposal (`AmendmentProposed`) tied to the current terms version, and the second matching call executes. Used for:
- **Amendments** (NEGOTIATING to LOCKED) — new amounts and/or a new expiry (D-07). Executing counts as both parties confirming the new terms; there is no second confirmation round (B-10). New values must pass the limits locked at creation, with the furthest expiry counted from creation (B-12). A deposit above the new amount gets the difference returned at once (`ExcessReturned`); a shortfall is topped up through `depositAssets()`. If an excess return fails, the whole amendment is undone (B-14). Changing amounts needs a live price; expiry-only changes don't (B-17)
- **After LOCKED** — amounts can only go down (refunded at once), or the expiry change. To increase, cancel and create a new deal (B-13)
- **Renegotiation** after a price breach (D-06, B-22) — amounts only go down, the new amounts must be back inside the price band (checked at proposal), expiry-only changes are refused, and both must agree inside the window. The deal returns to LOCKED with delivery confirmations reset

Emit `TermsAmended`.

### Read-only views

`getDeal()` · `termsHashOf()` · `acceptDigest()` · `depositOwed()` · `payoutPreview()` (what each side and the fee wallets receive if released now — used by ConfirmDeliveryModal) · `priceBandStatus()` (live band position) · `amendmentHashOf()` · `proVoucherDigest()` · `operatorApprovalDigest()` (B-27).

---

## Fee tier resolution (D-04)

| Situation | Fee applied |
|-----------|------------|
| No proof supplied | Free: 1%, max $10 |
| Valid Pro voucher (signed by the TrustBridge voucher key, names the caller's wallet, not expired, validity ≤ 35 days — B-06) | Pro: 0.5%, max $25 |
| Registered Enterprise operator **plus** a signed approval from that operator's registered authorizer (a key or the operator's own Safe) naming Party A's wallet (B-02) | Operator's registered rate (0.25% standard, negotiable); operator receives 70% of the fee charged |

The fee tier is never a free-form caller parameter. The fee rate and cap are locked at creation. The voucher key and the operator registry can only affect the fee level of new deals — never any deposit.

---

## Settings and admin powers

| Setting | Change process |
|---------|---------------|
| Token whitelist, price source | Admin multisig, 7-day public notice, new deals only |
| Fee rates and caps, fee wallet, operator registry, voucher key | Admin multisig, 7-day public notice, new deals only |
| Plan timer maximums, plan value limits, minimum deal value, guarded-launch cap | Admin multisig, 7-day public notice, new deals only |
| Pause new deals | Admin multisig, immediate. Stops `createDeal()` only — every existing deal continues normally |

- **Hard ceilings (A-02):** whatever the admin sets, a fee rate can never exceed 1% and a plan duration can never exceed 365 days
- **Scheduled changes lapse (A-03):** a change not executed within 14 days after its 7-day notice must be scheduled again
- **Price adapter has no admin (A-04):** feeds are fixed at deployment; to change a feed, deploy a new adapter and switch to it with the 7-day notice. Each deal keeps the price source it was created with

**No admin function can move, freeze, redirect or delay user funds, change an existing deal's rules, or override any outcome.** There is no emergency "break-glass" mode in v1 (D-08). Larger changes are made by deploying a new factory version: new deals use the new version, and existing deals finish under the rules they started with.

---

## Multi-chain configuration (D-09)

No chain-specific addresses are written into the contract code. Each chain has its own configuration, set at deployment:
- Safe contract addresses (SafeL2 singleton on L2s — B-30)
- Price source per token (Chainlink adapter on Base; any other chain decided before its deployment)
- L2 sequencer uptime feed — **required on Base mainnet**; the deployment plan refuses mainnet without it (B-36)
- Token whitelist
- Fee wallet, voucher key and operator registry

Adding an EVM chain is a configuration and deployment task, followed by a chain-specific audit review. Non-EVM chains (for example Solana) and cross-chain deals are out of scope.

---

## Price checks

- On-chain prices come only from the configured price source — on Base, Chainlink Data Feeds. CoinGecko is used for UI display only
- Every read checks: price above zero · round complete · **freshness within the feed's published heartbeat, no extra margin** (B-37 — e.g. USDC/USD 24 hours, ETH/USD 20 minutes on Base Sepolia) · **L2 sequencer up and back up for at least 1 hour** (A-01)
- Any failed check blocks the price-dependent action ("try again later", no state change) rather than using an old price
- Pair values are derived from USD feeds
- Refunds, cancellations and expiry never depend on the price source

---

## Fee configuration

| Plan | Rate | Cap |
|------|------|-----|
| Free | 1% of the requested amount | $10 maximum |
| Pro | 0.5% of the requested amount | $25 maximum |
| Enterprise | 0.25% standard | Negotiable |
| White-label split | 70% of the fee charged to the operator / remainder to TrustBridge | Automatic on-chain |

Fee charged on completion only — never on creation, expiry or cancellation. Paid by Party A in v1, out of the asset Party A receives (B-01); Party B always receives the full agreed amount. The cap is converted into the requested token at creation and locked. Rounded down in the user's favour. A misconfigured fee wallet reverts.

---

## Token whitelist and minimum value

**v1 tokens on Base:** USDC, USDT, ETH (as WETH), WBTC, DAI, WETH. Plain ERC-20 tokens only — fee-on-transfer and rebasing tokens are unsupported.  
**Minimum deal value:** $10 USD equivalent. **Guarded launch:** a maximum deal value applies for the first months of mainnet, raised over time ($50,000 on the Base Sepolia testnet — B-34; mainnet amount to be decided).

---

## Events

| Event | Emitted by |
|-------|-----------|
| `DealCreated` | `createDeal()` |
| `CounterpartyJoined` | `acceptDeal()` |
| `CounterpartyConfirmed` / `CounterpartyRejected` | `confirmCounterparty()` |
| `TermsConfirmed` | `confirmTerms()` |
| `AssetDeposited`, `VaultLocked` | `depositAssets()` |
| `DeliveryConfirmed` | `confirmDelivery()` |
| `FundsReleased` | release |
| `AutoReleaseTriggered` | `triggerAutoRelease()` |
| `PriceBandBreach`, `RenegotiationOpened` | release checks |
| `AmendmentProposed`, `TermsAmended`, `ExcessReturned` | `amendTerms()` |
| `CancelRequested`, `DealCancelled` | `cancelDeal()` |
| `RefundPaid` / `RefundPending` | refunds |
| `DealExpired` | `claimRefund()` |
| `SettingChangeScheduled`, `SettingChanged`, `SettingChangeCancelled`, `NewDealsPaused`, `NewDealsResumed`, `AdminTransferStarted`, `AdminTransferred` | settings |

---

## Security requirements

See [SECURITY.md](SECURITY.md) for the full threat model.

**Mandatory on every fund-moving path:** `nonReentrant` · state updated before every transfer · `SafeERC20`.

**Invariants proven by tests** (whole-lifecycle invariant campaign of 358,400 random actions, about half with company-Safe parties):
- Funds in a deal Safe can only ever leave to Party A, Party B or the recorded fee recipients
- Total paid out never exceeds total deposited; no funds are created, lost or stuck
- No owner-initiated Safe transaction can succeed while the guard is set
- No admin or settings change can affect an existing deal
- One party's failed refund never blocks the other party's refund
- Every payout goes only to the party address recorded in the deal
- All party actions work when a party is a smart-contract wallet (Safe)

**Achieved before testnet deployment:** 331 tests (unit, fuzz, invariant, property) · 98.1% line coverage · Slither 0 high / 0 medium · mutation check: 12 of 12 deliberately injected bugs caught, re-run before every deployment and after every review fix.

---

## Dependencies

| Library | Usage |
|---------|-------|
| OpenZeppelin 5.1 | ReentrancyGuard, SafeERC20, EIP-712 signatures |
| Safe v1.4.1 (official contracts, unmodified) | Deal Safe proxies (SafeL2 on L2s), module and guard interfaces |
| Chainlink Data Feeds | Price source adapter on Base |
| Solidity 0.8.26 · Foundry | Development, testing, deployment |

---

## Resolved during the build (were "details to confirm" in v1.2)

- Fee leg — deducted from what Party A receives (B-01)
- Pro voucher validity — 35 days maximum, enforced on-chain (B-06)
- Base Sepolia feeds — Chainlink publishes USDC, USDT, DAI, ETH and BTC feeds; no sequencer feed exists on Base Sepolia (B-36). The testnet uses free-mint test tokens priced by those real feeds
- Guarded-launch cap — $50,000 on Base Sepolia (B-34)

## Still to decide before mainnet

- Mainnet guarded-launch cap amount and schedule
- Base mainnet token set and issuer freeze behaviour (USDT, WBTC availability on Base)
- Admin multisig and treasury wallet
- Open questions for the independent reviewer: O-01 (escaping a deal at a price spike) and O-02 (Free deals priced off-market) — see KNOWN_LIMITATIONS.md

Enterprise agent seats, roles, per-agent deal attribution and consolidated reporting are application-layer features (operator dashboard) and need no contract changes.

---

## Deployment checklist

- [x] Factory, module, guard, settlement library and Base price adapter implemented
- [x] All 11 functions and invariants tested; 95%+ coverage (98.1%)
- [x] Static analysis (Slither) — no unresolved critical, high or medium findings
- [x] Base Sepolia deployment verified on Basescan; two-party demo deal completed (2 November 2026)
- [ ] Independent senior Solidity / Safe review
- [ ] Independent security audit — report published on this repository, naming the audited version
- [ ] 100 testnet deals across every outcome path
- [ ] Mainnet admin multisig, timelock, fee wallet, whitelist and sequencer feed verified on-chain
- [ ] Mainnet guarded-launch cap set
- [ ] VARA legal opinion obtained before mainnet

---

*Specification v1.3 · November 2026 · TrustBridge Protocol · trustbridgeprotocol.xyz*  
*Responsible disclosure: craig@trustbridgeprotocol.xyz*
