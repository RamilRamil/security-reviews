# `zapIn` does not enforce agreement with `quoteZapIn`: execution can arbitrarily diverge from the quoted plan

## Summary

`StableSwapZapIn.zapIn` executes whatever `Swap[]` the caller supplies. It never recomputes the optimal path from `quoteZapIn` / `_calculateOptimalSwaps`, never compares calldata to the off-chain quote, and never checks `expectedAmountOut` against realized swap deltas. Documentation suggests passing swaps from `quoteZapIn`, but this is **not** enforced on-chain. A caller can therefore obtain **fewer LP shares** than `quoteZapIn` indicated for the same `amounts` while still satisfying a weak `_minShares` (including `0`).

## Judging
Medium -> Valid

## Finding Description

**Quote path (`quoteZapIn`, view):**

- For a non-empty pool, builds `SwapQuote[]` via `_calculateOptimalSwaps`, applies expected outputs to derive `resultingAmounts`, then calls `hooks.quoteAddLiquidity(resultingAmounts)` to return an expected **share count**.

**Execution path (`zapIn`):**

- Pulls user tokens per `_amounts`, optionally runs `poolManager.unlock` with **caller-supplied** `_swaps`, then `_addLiquidityAndGetShares` using the contract's post-swap balances.
- Validation on `_swaps` is limited to **index sanity** (valid `tokenIn` / `tokenOut`, no identical indices). There is **no** requirement that `_swaps` matches `quoteZapIn` for the same `_amounts` and iteration budget, and **no** binding to `expectedAmountOut`.

**Broken expectation / trust model:** Integrations assume "I called `quoteZapIn`, then I pass its swaps to `zapIn` and get the quoted shares." The EVM does **not** guarantee that linkage. Malformed integrations, stale frontends, or adversarial calldata can pass **empty** or **suboptimal** `_swaps`. With `_minShares == 0`, slippage checks do not rescue the user.

Relevant flow (`src/periphery/StableSwapZapIn.sol`):

- `quoteZapIn` computes `swaps = _calculateOptimalSwaps(...)` and derives shares from simulated `resultingAmounts`.
- `zapIn` only documents: "Pre-calculated swaps from quoteZapIn" — **not** enforced in bytecode.

## Impact Explanation

**Assessment: Medium.**

Impact is primarily on the **caller** (or their users downstream): final LP position can be **strictly worse** than the `quoteZapIn` result for identical `amounts`, without reverting if `_minShares` is permissive. This is not a classic "drain arbitrary third-party wallets" primitive, but it **breaks the natural product invariant** that on-chain execution matches the **documented** quote API when both are used together.

Severity is capped by the fact that damage usually requires **wrong or malicious calldata** and **lenient** `_minShares`; well-configured UIs with sensible slippage limits reduce harm.

**Reporting note:** Treat this primarily as a **break between documented quote behavior and executable `zapIn`**, not as a classic contest-style **drain** of arbitrary third-party wallets. The bundled PoC stresses that gap using **deliberately bad** `_swaps` and **`_minShares == 0`** (see below).

## Likelihood Explanation

**Moderate** in real deployments: `minShares = 0` is allowed; integrators can mismatch quote and execution by bug or race; **no** on-chain link makes the mistake **silent** rather than failing.

## Proof of Concept

PoC idea: seed a two-token pool with balanced liquidity, call `quoteZapIn` with **heavily imbalanced** `amounts` so `_calculateOptimalSwaps` returns a **non-empty** swap plan and `sharesQuoted > 0`, then call `zapIn` with the **same** `amounts` but an **empty** `Swap[]` and `_minShares = 0`. Executed LP mint is **strictly below** `sharesQuoted`.

**What this PoC proves vs. what it does not:** It relies on **intentionally suboptimal** execution (`_swaps` omitted while the quote expects balancing swaps) and **zero minimum shares** (`_minShares == 0`), so the path stays non-reverting while showing a large **quoted vs realized** LP gap. That demonstrates the **missing on-chain link** between `quoteZapIn` and `zapIn`. It does **not** show theft from unrelated users without such calldata or without permissive slippage. In a **report**, pitch this as a **contract / API guarantee defect** (quote vs execution), not as a generic **drain** finding unless you attach a separate end-user attack story.

Copy the file below into the audited `stableswap-hooks` tree (e.g. `test/POC_ZapInQuoteMismatch.sol`) and run `forge test --mc POC_ZapInQuoteMismatchStandalone`. It does **not** import any `test/testUtils/` or `test/scenarios/` helper; only `src/` and vendor packages.

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.30;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Currency} from "@uniswap/v4-core/src/types/Currency.sol";
import {IPoolManager} from "@uniswap/v4-core/src/interfaces/IPoolManager.sol";
import {PoolManager} from "@uniswap/v4-core/src/PoolManager.sol";
import {HookMiner} from "@uniswap/v4-periphery/src/utils/HookMiner.sol";

import {Base} from "src/Base.sol";
import {StableSwapHooks} from "src/StableSwapHooks.sol";
import {StableSwapHooksFactory} from "src/factories/StableSwapHooksFactory.sol";
import {StableSwapZapIn, Swap, SwapQuote} from "src/periphery/StableSwapZapIn.sol";

contract Mint18 is ERC20 {
    constructor(string memory n, string memory s) ERC20(n, s) {}

    function mint(address to, uint256 a) external {
        _mint(to, a);
    }
}

contract FactoryHarness is StableSwapHooksFactory {
    constructor(
        IPoolManager _poolManager,
        address _owner,
        address _protocolFeeCollector,
        address _hookFeeCollector,
        bytes32 _creationCodeHash
    ) StableSwapHooksFactory(_poolManager, _owner, _protocolFeeCollector, _hookFeeCollector, _creationCodeHash) {}

    function mineSalt(
        Currency[] calldata _currencies,
        Base.RateOracleConfig[] calldata _rateOracles,
        uint256 _lpFeePercentage,
        uint256 _baseAmp,
        bytes calldata _creationCode
    ) external view returns (address hookAddress, bytes32 salt) {
        bytes memory constructorArgs = abi.encode(poolManager, _currencies, _rateOracles, _lpFeePercentage, _baseAmp);
        (hookAddress, salt) = HookMiner.find(address(this), HOOK_FLAGS, _creationCode, constructorArgs);
    }
}

contract POC_ZapInQuoteMismatchStandalone is Test {
    uint256 internal constant LP_FEE = 500;
    uint256 internal constant AMP = 100;
    uint256 internal constant PROTO_FEE = 100000;
    uint256 internal constant HOOK_FEE = 200000;

    function test_POC_quoteZapIn_strictly_more_shares_than_zapIn_with_no_swaps() public {
        address admin = address(0xA11CE);
        address seedLp = address(0x51EE);
        address zapUser = address(0x2A11);
        address pCol = address(0xC01);
        address hCol = address(0xC02);

        IPoolManager pm = IPoolManager(address(new PoolManager(admin)));
        FactoryHarness factory = new FactoryHarness(
            pm, admin, pCol, hCol, keccak256(type(StableSwapHooks).creationCode)
        );

        Mint18 tA = new Mint18("PoC18A", "P18A");
        Mint18 tB = new Mint18("PoC18B", "P18B");
        Currency c0 = Currency.wrap(address(tA) < address(tB) ? address(tA) : address(tB));
        Currency c1 = Currency.wrap(address(tA) < address(tB) ? address(tB) : address(tA));

        Currency[] memory currencies = new Currency[](2);
        currencies[0] = c0;
        currencies[1] = c1;

        Base.RateOracleConfig[] memory ro = new Base.RateOracleConfig[](2);
        ro[0] = Base.RateOracleConfig({oracle: address(0), selector: bytes4(0)});
        ro[1] = Base.RateOracleConfig({oracle: address(0), selector: bytes4(0)});

        bytes memory code = type(StableSwapHooks).creationCode;
        (, bytes32 salt) = factory.mineSalt(currencies, ro, LP_FEE, AMP, code);
        StableSwapHooks hooks = StableSwapHooks(factory.deploy(currencies, ro, LP_FEE, AMP, salt, code));

        StableSwapZapIn zap = new StableSwapZapIn(address(factory), keccak256(type(StableSwapHooks).creationCode));

        vm.startPrank(admin);
        hooks.setProtocolFeePercentage(PROTO_FEE);
        hooks.setHookFeePercentage(HOOK_FEE);
        vm.stopPrank();

        // --- Seed pool (first deposit, balanced large amounts) ---
        uint256 seedAmt = 1e24;
        Mint18(Currency.unwrap(c0)).mint(seedLp, seedAmt);
        Mint18(Currency.unwrap(c1)).mint(seedLp, seedAmt);
        vm.startPrank(seedLp);
        IERC20(Currency.unwrap(c0)).approve(address(hooks), seedAmt);
        IERC20(Currency.unwrap(c1)).approve(address(hooks), seedAmt);
        uint256[] memory seedAmounts = new uint256[](2);
        seedAmounts[0] = seedAmt;
        seedAmounts[1] = seedAmt;
        uint256[] memory seedMin = new uint256[](2);
        hooks.addLiquidity(seedAmounts, seedMin, 0);
        vm.stopPrank();

        // --- Imbalanced zap amounts: quote should propose non-zero swaps ---
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 5000e18;
        amounts[1] = 100e18;

        Mint18(Currency.unwrap(c0)).mint(zapUser, amounts[0]);
        Mint18(Currency.unwrap(c1)).mint(zapUser, amounts[1]);

        uint256 maxIter = 32;
        (uint256 sharesQuoted,, SwapQuote[] memory swapQuotes) = zap.quoteZapIn(address(hooks), amounts, maxIter);
        assertGt(swapQuotes.length, 0, "PoC setup: quote must request balancing swaps");
        assertGt(sharesQuoted, 0, "PoC setup: quoted shares must be positive");

        vm.startPrank(zapUser);
        IERC20(Currency.unwrap(c0)).forceApprove(address(zap), type(uint256).max);
        IERC20(Currency.unwrap(c1)).forceApprove(address(zap), type(uint256).max);

        uint256 lpBefore = IERC20(address(hooks)).balanceOf(zapUser);
        Swap[] memory noSwaps = new Swap[](0);
        zap.zapIn(address(hooks), amounts, noSwaps, 0);
        uint256 sharesExecuted = IERC20(address(hooks)).balanceOf(zapUser) - lpBefore;
        vm.stopPrank();

        assertLt(sharesExecuted, sharesQuoted, "same amounts but no swaps: executed LP strictly below quote");
    }
}
```

**Passing test demonstrates:** for fixed `amounts`, **`quoteZapIn` promises strictly more shares** than `zapIn` when execution **deliberately omits** the quoted swap path — with **no revert** if `_minShares == 0`. This matches the PoC assumptions in the disclaimer above (not a silent third-party drain by default).

## Recommendation

Pick one or combine:

1. **Derive swaps inside `zapIn`:** call the same internal planner used by `quoteZapIn` (or a shared internal function) so execution **cannot** diverge from the intended algorithm for given `amounts` and `maxIterations`. Expose a thin wrapper if a custom path is still needed for advanced users.

2. **Require proof-of-plan:** accept calldata swaps only if they match a **recomputed** plan (e.g. byte-equality or hash) produced in the same transaction before any user funds move.

3. **Disallow `minShares == 0`** on `zapIn` (documenting risk off-chain only is insufficient for strong safety).

4. **Document trust model** if intentional: state explicitly that `zapIn` is **low-level** and callers **must** pass exact quote output; ship a single high-level function that bundles quote+execute.


# First deposit can mint zero LP shares to the user while locking full `amounts` in reserves

## Summary

On the **first** liquidity deposit, if the computed geometric mean of scaled amounts equals `MINIMUM_LIQUIDITY` (1000), the code subtracts 1000 shares for dead-address liquidity but leaves the user with **0** minted shares. Tokens are still pulled into pool reserves and `MINIMUM_LIQUIDITY` is minted to `DEAD_ADDRESS`. If `minShares == 0`, the `newShares < minShares` check does not revert.

## Judging
Medium -> Informational

## Finding Description

In `_calculateAddLiquidity`, when `totalSupply() == 0`, the implementation:

1. Computes `shares = StableSwapMath.geometricMean(scaledAmounts)`.
2. Reverts only if `shares < MINIMUM_LIQUIDITY` (`InsufficientInitialLiquidity`).
3. Otherwise executes `shares -= MINIMUM_LIQUIDITY` and returns `(shares, actualAmounts)`.

When `shares == MINIMUM_LIQUIDITY` exactly, step 3 yields **`newShares == 0`**. The revert guard in step 2 does **not** fire because the comparison is strict (`<`), not `<=`.

Later, `_handleAddLiquidityCallback`:

- Mints `MINIMUM_LIQUIDITY` to `DEAD_ADDRESS` on the initial deposit.
- Pulls **full** `actualAmounts` into reserves via PoolManager.
- Calls `_mint(sender, newShares)` with **`newShares == 0`**.

**Security / invariant angle:** A successful add-liquidity path should not leave a depositor with **zero** claim on the liquidity they just provided when amounts are non-zero and the call completes without revert. Here the guarantee “depositors receive LP shares proportional / meaningful relative to what they put in” is broken at the boundary `geometricMean == MINIMUM_LIQUIDITY`. The protocol still increases reserves as if the user became an LP, but the user receives no redeemable shares (only dead supply exists for the minimum-liquidity leg).

**Propagation of “malicious” or careless input:** The callee supplies `amounts[]` on first mint such that `geometricMean(scaledAmounts) == 1000`, and `minShares == 0` (misconfigured router, buggy UI, or scripted integration). No privileged role is required; the hook executes the arithmetic as written.

Relevant code (`src/Liquidity.sol`):

```solidity
// First-branch geometry
shares = StableSwapMath.geometricMean(scaledAmounts);
if (shares < MINIMUM_LIQUIDITY) {
    revert InsufficientInitialLiquidity();
}
shares -= MINIMUM_LIQUIDITY; // when shares == 1000 -> 0
```

and callback mint ordering:

```solidity
if (isInitialDeposit) {
    _mint(DEAD_ADDRESS, MINIMUM_LIQUIDITY);
}
// ...
_mint(sender, newShares); // can be 0
```

## Impact Explanation

**Assessment: Medium.**

For a user that hits this edge with non-zero `amounts`, the **economic impact is severe locally**: they transfer tokens into the pool but receive **no** LP balance, while the pool’s reserves increase. They cannot later burn shares they never received. This is not the classic ERC-4626 “donation inflation” attack on the *next* depositor; it is a **first-mint boundary bug** with **high loss for the affected depositor** in that transaction.

It is not rated **High/Critical** as a protocol-wide theft primitive **from this description alone**: exploitation typically requires **crafted first-deposit sizing** and often **`minShares == 0`**, so it is not an unconditional drain of arbitrary third-party wallets. It remains a **material integrity bug** for LP minting semantics.

## Likelihood Explanation

**Low to moderate.**

- The condition `geometricMean(scaledAmounts) == 1000` is an **exact integer boundary**; it is rare under random deposit sizes but **straightforward to construct** (e.g. two-token pool with 1:1 scaling: amounts `(1000, 1000)` give `sqrt(1000 * 1000) = 1000`).
- **`minShares == 0`** should be discouraged at the application layer, but the **on-chain API allows it**, so misconfiguration or malicious calldata is realistic in the wild.
- Together, the scenario is **unlikely by accident**, **likely if intentionally triggered** or via bad defaults.

## Proof of Concept

**Setup assumptions:** two-currency pool on the **first** deposit (`totalSupply() == 0`); both tokens use the same decimals and oracle-free rates so `StableSwapMath.scaleTo(amount, rate) == amount` (e.g. two 18-decimal tokens with default rate oracles). With **mixed** decimals (e.g. 18 / 6), the same raw amounts `(1000, 1000)` generally **do not** yield `geometricMean(scaled) == 1000`; the PoC must use an equal-scaling pair.

Below is a **single Foundry test file**: copy into the audited `stableswap-hooks` tree (e.g. save as `test/POC_FirstDepositZeroShares.sol`), run `forge test --mc POC_FirstDepositZeroSharesStandalone`. It **does not import** any other file from `test/testUtils/` or `test/scenarios/`; only `src/`, OpenZeppelin, and Uniswap v4 packages are required.

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.30;

import {Test} from "forge-std/Test.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Currency} from "@uniswap/v4-core/src/types/Currency.sol";
import {IPoolManager} from "@uniswap/v4-core/src/interfaces/IPoolManager.sol";
import {PoolManager} from "@uniswap/v4-core/src/PoolManager.sol";
import {HookMiner} from "@uniswap/v4-periphery/src/utils/HookMiner.sol";

import {Base} from "src/Base.sol";
import {StableSwapHooks} from "src/StableSwapHooks.sol";
import {StableSwapHooksFactory} from "src/factories/StableSwapHooksFactory.sol";

/// @dev Minimal ERC20 for PoC (default 18 decimals).
contract Mint18 is ERC20 {
    constructor(string memory n, string memory s) ERC20(n, s) {}

    function mint(address to, uint256 a) external {
        _mint(to, a);
    }
}

/// @dev CREATE2 salt mining (same pattern as factory test harness, inlined so this file stands alone).
contract FactoryHarness is StableSwapHooksFactory {
    constructor(
        IPoolManager _poolManager,
        address _owner,
        address _protocolFeeCollector,
        address _hookFeeCollector,
        bytes32 _creationCodeHash
    ) StableSwapHooksFactory(_poolManager, _owner, _protocolFeeCollector, _hookFeeCollector, _creationCodeHash) {}

    function mineSalt(
        Currency[] calldata _currencies,
        Base.RateOracleConfig[] calldata _rateOracles,
        uint256 _lpFeePercentage,
        uint256 _baseAmp,
        bytes calldata _creationCode
    ) external view returns (address hookAddress, bytes32 salt) {
        bytes memory constructorArgs = abi.encode(poolManager, _currencies, _rateOracles, _lpFeePercentage, _baseAmp);
        (hookAddress, salt) = HookMiner.find(address(this), HOOK_FLAGS, _creationCode, constructorArgs);
    }
}

contract POC_FirstDepositZeroSharesStandalone is Test {
    uint256 internal constant LP_FEE = 500;
    uint256 internal constant AMP = 100;
    uint256 internal constant PROTO_FEE = 100000;
    uint256 internal constant HOOK_FEE = 200000;

    function test_POC_firstDeposit_zeroUserShares_equalDecimals() public {
        address admin = address(0xA11CE);
        address lpUser = address(0xB0B);
        address pCol = address(0xC01);
        address hCol = address(0xC02);

        IPoolManager pm = IPoolManager(address(new PoolManager(admin)));
        FactoryHarness factory = new FactoryHarness(
            pm, admin, pCol, hCol, keccak256(type(StableSwapHooks).creationCode)
        );

        Mint18 tA = new Mint18("PoC18A", "P18A");
        Mint18 tB = new Mint18("PoC18B", "P18B");
        Currency c0 = Currency.wrap(address(tA) < address(tB) ? address(tA) : address(tB));
        Currency c1 = Currency.wrap(address(tA) < address(tB) ? address(tB) : address(tA));

        Currency[] memory currencies = new Currency[](2);
        currencies[0] = c0;
        currencies[1] = c1;

        Base.RateOracleConfig[] memory ro = new Base.RateOracleConfig[](2);
        ro[0] = Base.RateOracleConfig({oracle: address(0), selector: bytes4(0)});
        ro[1] = Base.RateOracleConfig({oracle: address(0), selector: bytes4(0)});

        bytes memory code = type(StableSwapHooks).creationCode;
        (, bytes32 salt) = factory.mineSalt(currencies, ro, LP_FEE, AMP, code);
        StableSwapHooks hooks = StableSwapHooks(factory.deploy(currencies, ro, LP_FEE, AMP, salt, code));

        vm.startPrank(admin);
        hooks.setProtocolFeePercentage(PROTO_FEE);
        hooks.setHookFeePercentage(HOOK_FEE);
        vm.stopPrank();

        uint256 a = 1000;
        Mint18(Currency.unwrap(c0)).mint(lpUser, a);
        Mint18(Currency.unwrap(c1)).mint(lpUser, a);

        vm.startPrank(lpUser);
        Mint18(Currency.unwrap(c0)).approve(address(hooks), a);
        Mint18(Currency.unwrap(c1)).approve(address(hooks), a);

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = a;
        amounts[1] = a;
        uint256[] memory minAmt = new uint256[](2);

        hooks.addLiquidity(amounts, minAmt, 0);
        vm.stopPrank();

        assertEq(hooks.balanceOf(lpUser), 0, "user gets no LP");
        assertEq(hooks.totalSupply(), hooks.MINIMUM_LIQUIDITY());
        assertEq(hooks.balanceOf(0x000000000000000000000000000000000000dEaD), hooks.MINIMUM_LIQUIDITY());
        assertEq(hooks.reserves(0), a);
        assertEq(hooks.reserves(1), a);
    }
}
```

**Expected outcome when the test passes:** first `addLiquidity` with `amounts = (1000, 1000)`, `minShares = 0`, leaves the user with **0** LP; **only** `MINIMUM_LIQUIDITY` is minted (to the dead address); reserves for both currencies equal **1000**.

## Recommendation

Reject the boundary where the user’s post-burn share amount is zero while `actualAmounts` are non-zero, or require a strictly larger initial geometric mean.

**Option A (minimal, after subtraction):**

```solidity
shares -= MINIMUM_LIQUIDITY;
if (shares == 0) revert InsufficientInitialLiquidity();
```

**Option B (stricter, before subtraction):**

```solidity
if (shares <= MINIMUM_LIQUIDITY) {
    revert InsufficientInitialLiquidity();
}
shares -= MINIMUM_LIQUIDITY;
```

**Option C (defense in depth):** additionally disallow `minShares == 0` on first mint at the periphery, but **fix the hook invariant** so the core cannot mint zero user shares for non-zero deposits.

Option B matches the intent “first LP mint must leave meaningful user supply after locking minimum liquidity to dead shares.”
