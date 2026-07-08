# README
# Security Research Portfolio — ARIA / Area44 LLC

Smart-contract and Web3 security findings, compiled from active bug-bounty/audit-contest engagements. Methodology: white-box source review (recon → reentrancy → oracle → flash-loan → access control → arithmetic → bridge → governance) with adversarial multi-pass verification before anything ships.

## Verification legend

No finding in this document is described as "confirmed" or "working" unless a fork/unit test was actually executed and passed. Everything else is explicitly labeled unexecuted.

| Label | Meaning |
|---|---|
| ✅ **PoC-PASS** | A Foundry/fork (or unit) test was written **and run**; the behavior was directly observed. Highest confidence. |
| 🔍 **Source-verified, unexecuted** | Root cause traced line-by-line against real source; a PoC was drafted but **not run** (no forge/cargo available on the review box, or repo unreachable). Logic confidence is high; runtime behavior is unconfirmed. |
| 📄 **Skeleton (unverified)** | PoC and even file:line references are placeholders — the target source was private/unavailable at write time. Treat as a hypothesis, not a finding, until wired against real code. |
| ⭕ **Negative (proven)** | A hypothesized vulnerability was investigated and shown **not to hold** — a clean, documented result, often backed by passing tests. |

---

## Submitted / disclosed findings

### DRE dreUSD — Sherlock Contest #1259 (closed 17 Jun 2026)

| ID | Title | Severity | Verification | Status |
|---|---|---|---|---|
| F-1 | Pausing the rewards distributor corrupts vault NAV | Medium | 🔍 Source-verified — PoC skeleton written, **not executed** (no forge on review box) | Submitted during live contest |
| F-2 | OFT cross-chain delivery permanently locks funds (dead `stuckFundsRecipient` path) | Medium | 🔍 Source-verified, not executed | Submitted during live contest |
| F-3 | Multi-stablecoin mint vs. USDC-only redeem enables intra-band arbitrage | Medium | 📄 Skeleton — source repo was private/unavailable at write time; function signatures and file:line **unverified** | Courtesy disclosure (contest closed, no bounty) |
| F-4 | L2 sequencer `startedAt == 0` treated as a healthy round | Low | 📄 Skeleton — same caveat as F-3 | Courtesy disclosure (contest closed, no bounty) |

F-1/F-2 went in while the contest was live. F-3/F-4 surfaced after close and were sent directly to DRE Labs as good-faith disclosure — explicitly not a paid submission, and the source doc itself flags the PoCs as unverified skeletons pending access to the real repo.

### Sentiment V2 — Sherlock Bug Bounty #37 (program not live at review time)

Full oracle-focused review of `protocol-v2@master`. Five findings, all traced against live source; none run against a fork (no forge available during the engagement).

| ID | Title | Severity | Verification |
|---|---|---|---|
| F-5 | `RedstoneOracle` staleness check is inert — compares a millisecond timestamp against a seconds threshold, so it can never revert | Medium (arguably High) | 🔍 Source-verified, PoC drafted, not executed |
| F-6 | Hyperliquid oracles: no zero-price handling, and staleness is structurally impossible (bare `uint64` mark-price precompile) | Medium → High (conditional) | 🔍 Source-verified |
| F-7 | Hyperliquid oracles price off manipulable **mark** price, not index — no TWAP or deviation cap | Medium → High (conditional) | 🔍 Source-verified |
| F-8 | `AggV3Oracle` casts Chainlink `answer` to `uint256` with no positivity check; chains into an unguarded `MetaOracle` denominator | Medium | 🔍 Source-verified |
| F-9 | `AggV3Oracle` has no L2 sequencer-uptime / grace-period check | Medium | 🔍 Source-verified |

Process note: an initial recon pass produced 9 hypotheses; a dedupe pass against the public 2024 Sherlock judging report killed 6 of 7 as already-known **before any PoC work started**. Diffing the audited snapshot against `@master` then surfaced 4 genuinely new, unaudited oracles (F-6–F-9). The target program's live status was only confirmed as closed after the review was complete — findings were repurposed as coordinated disclosure / portfolio material rather than a paid submission.

### Rujira (THORChain) — Sherlock Bug Bounty #366 — *not yet submitted*

| ID | Title | Severity | Verification |
|---|---|---|---|
| M-01 | `Decimal` subtraction underflow panics `Revenue::distribute`, DoSing bRUNE swaps | Medium | 🔍 Source-verified against live GitLab source; PoC is a **drafted, unexecuted** Rust unit test (no cargo on the review box) |

Root cause: the fee-decay ratio is computed against **post-subtraction** `pending` instead of the original value. Any single distribution paying out 50–100% of `pending` makes the ratio exceed 1.0, and `Decimal::one() - ratio` underflows and panics — reverting the swap. State self-heals once a full `revenue_smear` period elapses without activity. Two additional leads (a `thorchain-swap` bad-debt window, and a `staking` donation-accounting quirk) were flagged but not confirmed — they need live pool/oracle state to resolve.

### Moonwell — Code4rena

| ID | Title | Severity | Verification |
|---|---|---|---|
| F-10 | `ChainlinkOracle` missing max-age/heartbeat validation — a frozen feed is served indefinitely | Medium (honestly, more realistically Low/Informational) | ✅ **PoC-PASS** — executed on a live Base mainnet fork against the real USDC/USD feed; 30-day warp produced no revert |

Caveats logged in the writeup: high duplicate risk (a commonly reported Compound-fork oracle pattern; the program excludes prior-audit findings), and the live market's routing through this exact oracle instance still needs reconfirming before submission.

### tokenize.it — HackenProof

| ID | Title | Severity | Verification |
|---|---|---|---|
| F-11 | `CoinvestedPosition` doesn't implement the documented sweep / `totalCredit` invariant | Low | 🔍 Source-verified by direct grep against the codebase |

---

## Clean / proven-negative engagements

A clean verdict is a real deliverable, not a non-result — these are documented "no exploitable bug here" findings, several backed by passing tests, that demonstrate the discipline of not filing unprovable or duplicate claims.

| Target | Scope | Verdict |
|---|---|---|
| Moonwell `MErc20Delegator` (mcbETH) | Delegatecall proxy, Base | ✅ Clean — 7 passing fork tests; the "scary"-looking public `delegateToImplementation` proven non-escalating |
| Uniswap UNI token | mint / transfer / transferFrom / permit | ✅ Clean — 4 passing fork tests + live `cast` calls; no theft/inflation path reachable |
| Intuition V2 (deployed scope) | TRUST token + Hub/Spoke bridges | ⭕ Clean — no unprivileged Critical/High path; value flow is role-gated |
| ENS contracts-v2 (Namechain) | 61 contracts, all severities | ⭕ Clean — see anti-hallucination case study below |
| ENS `ens-contracts` | Root/Registrar controllers + DNSSEC verifier | ⭕ Clean — access control correct; verifier fails closed |
| ENS metadata-service (web) | SVG/XSS/SSRF | ⭕ Clean — see case study below |
| Variational Omni (Arbitrum perps) | `SettlementPoolFactory`/`SettlementPool` | ⭕ Inconclusive — blocked on private/embargoed source; the only drain path found is single-EOA owner centralization (out of scope) |
| GMX V2 — liquidation & ADL path | `LiquidationUtils`/`AdlUtils`/`PositionUtils` | ⭕ Clean — an initial "funding fees excluded from liquidation calc" lead was falsified against literal source; fees are included via `totalCostAmount` |
| GMX V2 — Data Streams reference-price validation | `Oracle.sol` / `ChainlinkDataStreamProvider` | ⭕ Clean, by-design — synthetic markets intentionally have no reference feed; Data Streams reports are independently signed and verified. One informational note filed (the report doesn't honor `expiresAt`) |
| Inverse Finance JuniorDola | `jDola` / `FiRMSlashingModule` / `WithdrawalEscrow` | ⭕ Clean — two near-miss false positives caught and disproven (slash-cap underflow, "unbounded" slash trigger); informational hardening notes only. **Scope unconfirmed** — the Immunefi scope page 404'd during review, so confirm JuniorDola is actually in-bounty before submitting anything from this engagement |

### Case study: catching hallucinations before they ship (ENS engagement)

During the ENS review, ARIA's finder agents raised three plausible-sounding "Critical" candidates that a separate adversarial coordinator pass killed by tracing each to its real sink:

1. **"Critical stored XSS via ENS name in SVG"** — killed: normalized names can't contain SVG metacharacters (ENSIP-15), and the rendering iframe is sandboxed without `allow-scripts`.
2. **"Critical reentrancy in `_regenerate()`"** — killed: the burn leg uses `to = address(0)` (no callback fires), and the mint callback is the last operation with no state writes after it.
3. **"High DoS via `_hasZeroNybbles` borrow propagation"** — killed by static proof: the SWAR bit-trick's cross-nybble borrow artifact can be shown never to flip the function's boolean result.

Result: 0 false reports shipped to Immunefi, 0 submittable bugs — the correct outcome for a heavily-audited target. Full writeup: `ARIA-anti-hallucination-ENS.md`.

---

## Engagement ledger

| Engagement | Platform | Findings | Status |
|---|---|---|---|
| DRE dreUSD | Sherlock #1259 | F-1, F-2 submitted; F-3, F-4 disclosed | Contest closed |
| Sentiment V2 | Sherlock BB #37 | F-5–F-9 | Program not live → portfolio/disclosure |
| Rujira | Sherlock BB #366 | M-01 + 2 unconfirmed leads | Not yet submitted |
| Moonwell | Code4rena | F-10 + 1 clean contract (N1) | F-10 pending submission |
| tokenize.it | HackenProof | F-11 | — |
| Uniswap, Intuition, ENS (×3), Variational, GMX V2 (×2), Inverse Finance | Various | Clean / proven-negative | Documented, no submission |

---

## Source documents

| File | Covers |
|---|---|
| `CONSOLIDATED-FINDINGS-PORTFOLIO.pdf` | Master snapshot (as of 2026-06-21) — F-1 through F-11 plus all proven-negatives |
| `DISCLOSURE-F3-F4.md` | dreUSD F-3/F-4 courtesy-disclosure email draft |
| `CASE-STUDY-2026-06-17-sentiment-v2.md` | Full Sentiment V2 engagement writeup + process lessons |
| `F1-redstone-stale-check-broken.md`, `F1-redstone-stale-submission.md` | Sentiment V2 F-5 detail + Sherlock-format submission |
| `F2-F5-new-oracle-surface.md` | Sentiment V2 F-6–F-9 consolidated |
| `F2-hyperliquid-zero-and-staleness-submission.md`, `F3-hyperliquid-mark-price-manipulation-submission.md`, `F4-aggv3-nonpositive-answer-submission.md`, `F5-aggv3-no-sequencer-check-submission.md` | Sentiment V2 F-6–F-9 individual Sherlock-format submissions |
| `R1-rujira-bug-bounty.md` | Rujira M-01 + unconfirmed leads |
| `R1-juniordola-review.md` | Inverse Finance JuniorDola clean review |
| `F1-datastreams-no-ref-feed-review.md`, `F2-liquidation-adl-review.md` | GMX V2 clean reviews |
| `ARIA-anti-hallucination-ENS.md` | ENS anti-hallucination case study |

---

*Compiled from the "Findings" self-email dated 2026-07-08 (15 attachments). Severity and verification status reflect each source document as written; where documents disagreed on verification level (e.g. the consolidated portfolio calling F-3/F-4 "source-verified" vs. the disclosure draft itself flagging them as unverified skeletons), the more conservative claim was used.*

Findings and Capabilities 
