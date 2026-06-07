I-01 Spot oracle 0x808 is readable but not wired into `spotNotionalUsdcFromSpotPx` (dead integration path)
Severity
Informational

Finding description and impact
`PrecompileReader.spotPx` reads HyperCore spot price precompile **0x808** (scaled per NatSpec as `10^(8 - baseSzDecimals)`). `TokenMath.spotNotionalUsdcFromSpotPx` implements the companion notional formula with a **+2** exponent shift in the divisor versus `spotNotionalUsdcFromPerpPx`, matching the documented 0x808 vs 0x807 precision gap.

Production notionals in `src/` use **0x807** (`oraclePx`) and `spotNotionalUsdcFromPerpPx` only (see `spotNotionalUsdcFromPerp` / `suppliedNotionalUsdcFromPerp`). **Nothing in `src/` calls `spotNotionalUsdcFromSpotPx`**, so this branch is not exercised and cannot currently skew `totalBackingSigned` or settlement gates.

The residual risk is **forward-looking**: if 0x808 is later connected to backing without golden vectors from official Hyperliquid specs and without cross-checks against the perp path (within floor tolerance), a systematic scale mismatch on `rawSpotPx` / `spotSzDecimals` could mis-state notionals (constant-factor bias, not rounding dust). That would be an integration/spec alignment failure, not demonstrated as incorrect Solidity in isolation.

Recommended mitigation steps
Before first use of 0x808 in backing math, add **spec-backed golden tests** for `spotNotionalUsdcFromSpotPx`, mirror the existing `TokenMathFuzz.t.sol` properties for the perp path (zeros, revert domain `weiDecimals + 2 >= spotSzDecimals`, monotonicity in balance), and if both 0x807 and 0x808 paths can value the same economic exposure, define and test an acceptable **floor-only** differential budget. Document which field supplies **`spotSzDecimals`** relative to `baseSzDecimals` and HIP-1 indices.

---

I-02 Generic `evmToL1Wei` / `l1WeiToEvm` are unused in `src/`; intentional floor asymmetry and sharp `int8` corners for future HIP-1
Severity
Informational

Finding description and impact
`TokenMath.evmToL1Wei` and `TokenMath.l1WeiToEvm` generalize EVM↔L1 wei conversion using `evmExtraWeiDecimals` (NatSpec: `L1 weiDecimals - EVM decimals`). For **`evmExtraWeiDecimals > 0`**, EVM→L1 floor-divides and L1→EVM multiplies exactly, so **full EVM→L1→EVM round-trip is not an algebraic inverse** on arbitrary amounts; that follows from the chosen integer convention, not from a compiler bug. Integrators who assume identity round-trips can mis-reconcile balances between sides.

There are **no call sites** to these two functions anywhere under `src/` (only tests). Live code uses **`usdcEvmToL1Wei` / `usdcL1WeiToEvm`** with the fixed factor 100, which matches `evmToL1Wei(..., -2)` on the safe domain.

The API still exposes **`int8`** without an on-chain enforced HIP-1 range. In particular, **`evmExtraWeiDecimals == type(int8).min` (-128)** forces evaluation of `-evmExtraWeiDecimals` in the negative-extra branch, which **panics** on unary minus overflow in Solidity 0.8; extreme positive `extra` can also revert via overflow in power/multiply steps. That is fail-closed for nonsensical metadata but shows the surface is not "any int8".

Until a trusted reader passes real `evmExtraWeiDecimals` from precompiles into these helpers, there is **no attacker-controlled execution path** in the contest snapshot and **no demonstrated fund impact**.

Recommended mitigation steps
When wiring HIP-1 tokens, validate **`evmExtraWeiDecimals`** at a single trusted choke point (aligned with real 0x80C / HIP-1 bounds), document that EVM↔L1 round-trip is **not** guaranteed for `extra != 0`, and extend **fuzz/unit** coverage beyond the current `[-6, 6]` sandbox: explicit test for **-128** panic, SafeCast boundaries for negative `extra`, and optional L1→EVM→L1 idempotence where `extra > 0` on integer domains. Keep differential checks that **`evmToL1Wei(x, -2)`** matches **`usdcEvmToL1Wei(x)`** on the safe range.

---

I-03 `totalBackingSigned` additively sums 0x801 hedge notionals, 0x811 supplied notionals, and 0x80F perp `accountValue` — bounded correctness is a HyperCore semantics assumption
Severity
Informational

Finding description and impact
`MonetrixAccountant._readL1Backing` intentionally **adds** (among other terms) whitelist spot notionals from **0x801** via `spotNotionalUsdcFromPerp`, supplied notionals from **0x811** for registered slots, and signed perp value from **0x80F**. That matches the README-style decomposition, but the contracts **do not prove** HyperCore excludes the same economic mass from being represented in more than one bucket (e.g. HIP-1 overlap between hedge spot and supplied for the same logical inventory) or that **0x80F** is pairwise non-overlapping with the explicit spot/supplied MTM terms.

This is **not** demonstrated double-counting in Solidity; it is an **external integration assumption** identical in spirit to other HL precompile reliance. Wrong HL accounting semantics would skew `surplus()` / settlement gates, not a user-accessible EVM exploit in the snapshot.

Recommended mitigation steps
Obtain **explicit HL documentation or fork evidence** for non-duplication across 0x801 vs 0x811 vs 0x80F for the Vault inventory states you deploy. Publish a concise **scenario matrix** (which inventory states you support and how each precompile slot is interpreted) as a sponsor-facing invariant note or README subsection.

---

I-04 `distributableSurplus` subtracts `RedeemEscrow.shortfall` while `requestRedeem` does not burn USDM — document the full lifecycle before interpreting Gate 3
Severity
Informational

Finding description and impact
Gate 3 in `settleDailyPnL` uses **`distributableSurplus() = surplus() - shortfall()`** where `surplus()` includes **`USDM.totalSupply()`**. Redeem flow **locks** USDM on the Vault and increases escrow obligations on **`requestRedeem`**, with **`claimRedeem` -> `usdm.burn`** only after cooldown. Readers can **mis-interpret** whether the same economic shortfall is represented in supply, escrow, and distributable math unless the timing is spelled out.

This is a **documentation / audit clarity** item unless a specific double-counting identity is proven across those metrics in a separate, formal balance-sheet analysis.

Recommended mitigation steps
Add a short **balance-sheet narrative** (README or `docs/`): deposit, stake, `requestRedeem`, `claimRedeem`, `shortfall`, and how each affects `totalBackingSigned`, `totalSupply`, and Gate 3 at each step.

---

I-05 `notifyVaultSupply` stores `(spotToken, perpIndex)` without on-chain cross-check against `MonetrixConfig` maps
Severity
Informational

Finding description and impact
`MonetrixAccountant.notifyVaultSupply` is **`onlyVault`** and records the tuple on first sight per `spotToken` (idempotent thereafter). The Accountant does **not** assert `perpIndex == spotToPerp(spotToken)` (BTC-perp index `0` is a deliberate special case for map lookups). Current **`MonetrixVault`** call sites derive a consistent pair from config + hedge checks, so the snapshot has **no untrusted writer**, but a **future Vault change** or **stale registry after governor whitelist edits** could make 0x811 notionals use a wrong oracle/`szDecimals` path until an Operator fixes the registry.

Recommended mitigation steps
Optional on-chain **`require(perpIndex == config.spotToPerp(spotToken))`** when `spotToken != USDC` (with explicit handling for USDC), or off-chain monitoring on `VaultSuppliedRegistered` vs `tradeableAssets` whenever the tradeable whitelist changes.

---

I-06 Read precompile wrappers enforce minimum `returndata` length; HL does not version read response layouts in GitBook
Severity
Informational

Finding description and impact
`PrecompileReader` rejects obviously short returns before `abi.decode`. Official Hyperliquid docs point to the attached **`L1Read.sol`** as the wire reference, not a separate byte-length spec. Minimum-size checks **do not rule out** a future compatible-length response with **reordered semantics** for dynamic ABI structs (`PerpAssetInfo`, `TokenInfo`). Fail-closed **revert** still dominates **silent truncation** on short buffers.

This parallels I-01 as **integration / upgrade hygiene**, not a shown production miscalculation in the snapshot.

Recommended mitigation steps
Diff upstream **`L1Read.sol` / hyper-evm-lib** on each HL release; extend regression or fork tests if precompile layouts change. Cross-check against GitBook [Interacting with HyperCore](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore).

---

I-07 `maxTVL` is enforced on `deposit` only; yield distribution mints USDM without comparing to `maxTVL`
Severity
Informational

Finding description and impact
`MonetrixVault.deposit` requires `usdm.totalSupply() + amount <= maxTVL` when `maxTVL > 0`. `distributeYield` mints **`userShare`** to the Vault and forwards it to **`sUSDM.injectYield`** with **no `maxTVL` guard**, so **`totalSupply()` can exceed `maxTVL`** after deposits hit the cap and yield accrues.

That may be **intentional** (cap on deposit inflows only), but **`README.md` does not state the policy**, so governance and users can misread the parameter.

Recommended mitigation steps
Document whether `maxTVL` is a cap on **new deposits only** or a hard ceiling on **all issued USDM**; if the latter is desired, add a check or a separate yield cap in `distributeYield` (or equivalent).

---

I-08 `ActionEncoder.sendSpotSend` applies USDC-specific EVM 6-dp to L1 8-dp scaling to `amount` for arbitrary `token` index; no `src/` call sites
Severity
Informational

Finding description and impact
`sendSpotSend` always routes **`amount`** through **`TokenMath.usdcEvmToL1Wei`** (D6{USDC} to D8{L1} wei for USDC). The **`token`** argument is forwarded raw. **Nothing under `src/` invokes this helper today**, so production cannot mis-send a non-USDC token with the wrong scale **yet**.

Residual risk matches I-01: **first integration** with HIP-1 or another token index without a dedicated encoder would silently encode wrong L1 wei.

Recommended mitigation steps
Rename or **`require(token == USDC_TOKEN_INDEX)`** before the USDC scaler when first used, or add a parallel helper keyed by HIP-1 `weiDecimals`.

---

I-09 `TokenMath` NatSpec references a missing `docs/invariants/precision-invariants.md`
Severity
Informational

Finding description and impact
The file header in `TokenMath.sol` points readers to **`docs/invariants/precision-invariants.md`**, which **does not exist** in the repo snapshot. This impedes reviewers validating dimensional / rounding intent described in NatSpec.

Recommended mitigation steps
Restore the document or update the NatSpec link to an existing specification (README section, external URL, or in-repo path).

---

I-10 Redeem, bridge, `shortfall`, and `bridgeRetentionAmount` are operationally coupled; stress behaviour depends on Operator and Governor tuning
Severity
Informational

Finding description and impact
User redeem liquidity, L1 bridging, and escrow **shortfall** interact through **`netBridgeable`**, **`fundRedemptions`**, and related paths. Under stress, **user-visible timing** of payouts can degrade unless Operator funds escrow and bridges on schedule; many vectors are already classified as **trusted Operator** in sponsor scope.

This item records **documentation / runbook** debt rather than a new Solidity bug class.

Recommended mitigation steps
After any change to bridge, escrow, or retention parameters, **reconcile user-facing documentation** with the current code paths; publish operator monitoring guidance (shortfall vs vault EVM balance vs `bridgeRetentionAmount`).
