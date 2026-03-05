# Stratax Contracts - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Missing Chainlink staleness and round-completeness checks in getPrice allow stale prices to reach critical financial calculations](#M-01)
    - ### [M-02. calculateUnwindParams and _executeUnwindOperation use different formulas, causing swap calldata mismatch that permanently locks user positions](#M-02)
    - ### [M-03. _executeUnwindOperation uses liqThreshold instead of ltv, causing user to receive 6.25% less collateral on every unwind](#M-03)
- ## Low Risk Findings
    - ### [L-01. Slippage check in _call1InchSwap can be bypassed when 1inch router returns empty bytes and the contract holds a pre-existing token balance](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #57

### Dates: Feb 12th, 2026 - Feb 19th, 2026

[See more contest details here](https://codehawks.cyfrin.io/c/2026-02-stratax-contracts)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 3
- Low: 1



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Missing Chainlink staleness and round-completeness checks in getPrice allow stale prices to reach critical financial calculations            



## Description

* `StrataxOracle::getPrice` fetches the latest price from a Chainlink feed and returns it to `Stratax.sol`, where it is used to calculate flash loan amounts, collateral withdrawal sizes, and liquidation thresholds. The only validation performed is that the returned `answer` is positive.

* `latestRoundData()` returns five values: `roundId`, `answer`, `startedAt`, `updatedAt`, and `answeredInRound`. The implementation destructures only `answer` and silently ignores the remaining four. This means a frozen feed (no updates due to node outage or circuit breaker) or an unfinished round will return an accepted price with no on-chain signal.

```solidity
    function getPrice(address _token) public view returns (uint256 price) {
        address priceFeedAddress = priceFeeds[_token];
        require(priceFeedAddress != address(0), "Price feed not set for token");

        AggregatorV3Interface priceFeed = AggregatorV3Interface(priceFeedAddress);

// @>   (, int256 answer,,,) = priceFeed.latestRoundData();
        require(answer > 0, "Invalid price from oracle");

        price = uint256(answer);
    }
```

## Risk

**Likelihood**:

* A Chainlink node infrastructure outage or deliberate circuit breaker activation freezes `updatedAt` while market prices continue to move — a scenario with historical precedent (LUNA crash, May 2022).

* A Chainlink round transition leaves `answeredInRound < roundId`, meaning the on-chain answer was computed in a previous round and the current round is unresolved.

**Impact**:

* A frozen high collateral price causes `_executeUnwindOperation` to calculate too little collateral to withdraw. The 1inch swap returns fewer funds than needed to repay the flash loan, reverting with `"Insufficient funds to repay flash loan"` and locking the user's position while Aave can liquidate it using its own live oracle.

* A stale price when opening a position allows creating undercollateralized leverage or incorrectly blocks a healthy position from being opened.

* A stale price during a forced unwind (Stratax.sol:570–571) leads to extracting the wrong collateral amount, potentially leaving residual debt or over-extracting collateral from the user.

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {StrataxOracle} from "../../src/StrataxOracle.sol";

contract MockStaleFeed {
    int256  public price;
    uint256 public updatedAt;
    uint80  public roundId;
    uint80  public answeredInRound;

    constructor(int256 _price, uint256 _staleDuration) {
        price           = _price;
        updatedAt       = block.timestamp - _staleDuration;
        roundId         = 1;
        answeredInRound = 1;
    }

    function decimals() external pure returns (uint8) { return 8; }

    function latestRoundData()
        external
        view
        returns (uint80, int256, uint256, uint256, uint80)
    {
        return (roundId, price, 0, updatedAt, answeredInRound);
    }

    function setRoundData(uint80 _roundId, uint80 _answeredInRound) external {
        roundId         = _roundId;
        answeredInRound = _answeredInRound;
    }
}

contract StaleOraclePoCTest is Test {
    StrataxOracle oracle;
    MockStaleFeed feed;
    address constant TOKEN = address(0xBEEF);

    function setUp() public {
        vm.warp(1 days);
        oracle = new StrataxOracle();
        feed   = new MockStaleFeed(3000e8, 45 minutes);
        oracle.setPriceFeed(TOKEN, address(feed));
    }

    /// @notice getPrice() accepts 45-minute-old data without reverting.
    function test_stalePriceAccepted() public view {
        uint256 price = oracle.getPrice(TOKEN);
        assertEq(price, 3000e8, "Stale price returned with no revert");
    }

    /// @notice getPrice() accepts data from an incomplete round (answeredInRound < roundId).
    function test_incompleteRoundAccepted() public {
        feed.setRoundData(5, 3);
        uint256 price = oracle.getPrice(TOKEN);
        assertEq(price, 3000e8, "Incomplete round accepted as valid");
    }
}
```

## Recommended Mitigation

```diff
+   uint256 public constant MAX_PRICE_AGE = 3600; // adjust per feed heartbeat

    function getPrice(address _token) public view returns (uint256 price) {
        address priceFeedAddress = priceFeeds[_token];
        require(priceFeedAddress != address(0), "Price feed not set for token");

        AggregatorV3Interface priceFeed = AggregatorV3Interface(priceFeedAddress);

-       (, int256 answer,,,) = priceFeed.latestRoundData();
+       (uint80 roundId, int256 answer,, uint256 updatedAt, uint80 answeredInRound) = priceFeed.latestRoundData();
        require(answer > 0, "Invalid price from oracle");
+       require(block.timestamp - updatedAt <= MAX_PRICE_AGE, "Stale price");
+       require(answeredInRound >= roundId, "Incomplete round");

        price = uint256(answer);
    }
```
## <a id='M-02'></a>M-02. calculateUnwindParams and _executeUnwindOperation use different formulas, causing swap calldata mismatch that permanently locks user positions            



## Description

* `Stratax::calculateUnwindParams` is the public view function users and frontends call to determine how much collateral will be withdrawn during an unwind, so they can prepare the corresponding 1inch swap calldata. `Stratax::_executeUnwindOperation` is the internal function that performs the actual withdrawal.

* The two functions compute `collateralToWithdraw` using entirely different formulas. `calculateUnwindParams` applies a flat 5% slippage buffer on top of the price-ratio amount. `_executeUnwindOperation` applies an LTV-precision scaling using `liqThreshold`. For typical Aave parameters (ltv=7500, liqThreshold=8000), the execution function withdraws ~19% more collateral than the view function predicted. The `_collateralToWithdraw` parameter accepted by `unwindPosition` is stored in `UnwindParams` but is never read by `_executeUnwindOperation` — it is silently dead code.

```solidity
    // calculateUnwindParams (lines 464-468) — what the user sees:
    collateralToWithdraw = (debtTokenPrice * debtAmount * 10 ** collateralDec)
                         / (collateralTokenPrice * 10 ** debtDec);
// @> collateralToWithdraw = (collateralToWithdraw * 1050) / 1000; // +5% slippage

    // _executeUnwindOperation (lines 575-577) — what actually executes:
// @> uint256 collateralToWithdraw = (
// @>     _amount * debtTokenPrice * (10 ** collateralDec) * LTV_PRECISION
// @> ) / (collateralTokenPrice * (10 ** debtDec) * liqThreshold);
```

## Risk

**Likelihood**:

* Every user who calls `calculateUnwindParams` to prepare 1inch swap calldata and then calls `unwindPosition` with that calldata triggers the mismatch — this is the documented and intended usage pattern.

* The discrepancy is always present regardless of market conditions, position size, or token pair, because it is structural: the two formulas differ by a constant factor of `(LTV_PRECISION / liqThreshold) / 1.05`.

**Impact**:

* The contract withdraws ~19% more collateral from Aave than the user's 1inch calldata was built for. If 1inch interprets the calldata as an exact-input swap, the extra collateral cannot be swapped, the flash loan repayment falls short, and `unwindPosition` reverts — the user's position is permanently locked until they construct calldata for the correct amount, which `calculateUnwindParams` cannot provide.

* Even if the swap does not revert, the extra collateral withdrawn but not swapped remains stranded in the contract and is not returned to the user, causing a direct financial loss of ~$190 per $1000 USDC of debt at typical parameters.

* The `_collateralToWithdraw` parameter in `unwindPosition` is dead code: it is accepted, validated, and stored, but `_executeUnwindOperation` ignores it entirely and recalculates its own value. Any value the user passes — including the output of `calculateUnwindParams` — has no effect on execution.

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";

contract UnwindMismatchPoCTest is Test {
    uint256 constant DEBT_AMOUNT       = 1000e6;
    uint256 constant USDC_PRICE        = 1e8;
    uint256 constant WETH_PRICE        = 3000e8;
    uint256 constant AAVE_LIQ_THRESHOLD = 8000;
    uint256 constant LTV_PRECISION     = 1e4;

    /// @notice Proves the two formulas yield different amounts (~19% gap).
    function test_formulaMismatch() public pure {
        // calculateUnwindParams formula
        uint256 base = (USDC_PRICE * DEBT_AMOUNT * 1e18) / (WETH_PRICE * 1e6);
        uint256 viewEstimate = (base * 1050) / 1000; // +5% slippage

        // _executeUnwindOperation formula
        uint256 internalActual = (DEBT_AMOUNT * USDC_PRICE * 1e18 * LTV_PRECISION)
            / (WETH_PRICE * 1e6 * AAVE_LIQ_THRESHOLD);

        uint256 discrepancyBps = ((internalActual - viewEstimate) * 10000) / viewEstimate;

        assertGt(internalActual, viewEstimate, "execution withdraws more than view predicts");
        assertGt(discrepancyBps, 1800, "discrepancy is at least 18%");

        // User's calldata covers viewEstimate WETH; contract withdraws internalActual WETH.
        // Extra WETH that cannot be swapped by the prepared calldata:
        uint256 unswappedWeth = internalActual - viewEstimate;
        uint256 stuckUsd = (unswappedWeth * 3000) / 1e18;

        assertGt(stuckUsd, 190, "at least $190 stranded per $1000 USDC unwind");
    }
}
```

## Recommended Mitigation

```diff
-   // Account for 5% slippage in swap
-   collateralToWithdraw = (collateralToWithdraw * 1050) / 1000;

+   // Match the formula used in _executeUnwindOperation
+   (,, uint256 liqThreshold,,,,,,,) =
+       aaveDataProvider.getReserveConfigurationData(_collateralToken);
+   collateralToWithdraw = (debtAmount * debtTokenPrice
+       * 10 ** IERC20(_collateralToken).decimals() * LTV_PRECISION)
+       / (collateralTokenPrice * 10 ** IERC20(_borrowToken).decimals() * liqThreshold);
```

Both functions must use the same formula so that the calldata prepared from the view function matches what the execution function will actually withdraw. Alternatively, make `_executeUnwindOperation` read `unwindParams.collateralToWithdraw` directly instead of recalculating.
## <a id='M-03'></a>M-03. _executeUnwindOperation uses liqThreshold instead of ltv, causing user to receive 6.25% less collateral on every unwind            



## Description

* `Stratax::_executeUnwindOperation` calculates how much collateral to withdraw from Aave after repaying a portion of debt. The formula is intended to withdraw exactly the collateral that backed the repaid debt, scaled by the asset's LTV ratio. The code comment on line 574 explicitly names the denominator variable as `ltv`.

* The destructuring of `getReserveConfigurationData` captures the third return value (index 2 = `liquidationThreshold`) and names it `liqThreshold`, while the second return value (index 1 = `ltv`) is skipped. For WETH on Aave (ltv=7500, liqThreshold=8000), using `liqThreshold` in the denominator inflates it by `8000/7500`, which reduces `collateralToWithdraw` by exactly 6.25% on every call. The shortfall remains locked in Aave on the contract's Aave balance and is not returned to the user.

```solidity
    // Comment says "ltv", but code reads liqThreshold (index 2, not index 1):
// @> (,, uint256 liqThreshold,,,,,,,) =
        aaveDataProvider.getReserveConfigurationData(unwindParams.collateralToken);

    // Calculate collateral to withdraw: (...) / (collateralPrice * debtDec * ltv)  ← comment
    uint256 collateralToWithdraw = (
        _amount * debtTokenPrice * (10 ** IERC20(unwindParams.collateralToken).decimals()) * LTV_PRECISION
// @> ) / (collateralTokenPrice * (10 ** IERC20(_asset).decimals()) * liqThreshold);
```

## Risk

**Likelihood**:

* Every call to `unwindPosition` or `executeOperation` on any collateral token where `ltv != liqThreshold` triggers the loss — which is true for all standard Aave v3 assets (e.g. WETH: ltv=7500, liqThreshold=8000).

* The loss is deterministic and accumulates with every partial unwind; a user performing 10 partial unwinds on a $10,000 position loses approximately $83 in collateral.

**Impact**:

* The user receives 6.25% less collateral than they are entitled to on each unwind. The shortfall (e.g. ~0.028 WETH per 1000 USDC of debt) stays on the contract's Aave deposit and cannot be individually recovered.

* Cumulative losses scale linearly with the number of unwind iterations and the size of the position, representing a direct and silent financial loss for every user who closes or partially unwinds a position.

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";

contract LiqThresholdPoCTest is Test {
    uint256 constant DEBT_AMOUNT        = 1000e6;
    uint256 constant USDC_PRICE         = 1e8;
    uint256 constant WETH_PRICE         = 3000e8;
    uint256 constant AAVE_LTV           = 7500;
    uint256 constant AAVE_LIQ_THRESHOLD = 8000;
    uint256 constant LTV_PRECISION      = 1e4;

    /// @notice Proves liqThreshold underestimates withdrawal by exactly 6.25%.
    function test_liqThresholdUnderWithdraws() public pure {
        // What the user should receive (correct: uses ltv = 7500)
        uint256 expected = (DEBT_AMOUNT * USDC_PRICE * 1e18 * LTV_PRECISION)
            / (WETH_PRICE * 1e6 * AAVE_LTV);

        // What the contract actually withdraws (bug: uses liqThreshold = 8000)
        uint256 actual = (DEBT_AMOUNT * USDC_PRICE * 1e18 * LTV_PRECISION)
            / (WETH_PRICE * 1e6 * AAVE_LIQ_THRESHOLD);

        uint256 lossBps = ((expected - actual) * 10000) / expected;

        assertLt(actual, expected, "liqThreshold underestimates collateral withdrawal");
        assertEq(lossBps, 625, "loss is exactly 625 bps = 6.25%");
    }
}
```

## Recommended Mitigation

```diff
-   (,, uint256 liqThreshold,,,,,,,) =
+   (, uint256 liqThreshold,,,,,,,,) =
        aaveDataProvider.getReserveConfigurationData(unwindParams.collateralToken);
```

This changes the destructuring to capture index 1 (`ltv`) instead of index 2 (`liquidationThreshold`), matching the intent described in the comment on line 574.

# Low Risk Findings

## <a id='L-01'></a>L-01. Slippage check in _call1InchSwap can be bypassed when 1inch router returns empty bytes and the contract holds a pre-existing token balance            



## Description

* `Stratax::_call1InchSwap` executes a swap through the 1inch router and enforces a minimum return amount to protect the user against bad swap rates. When the router returns properly encoded data, `returnAmount` is decoded directly from the response.

* When the router call succeeds but returns empty bytes, the function falls back to `IERC20(_asset).balanceOf(address(this))` as `returnAmount`. This value includes any tokens already held by the contract before the swap, not only the amount actually received from it. If the pre-existing balance is large enough to push the total above `_minReturnAmount`, the slippage check passes even when the actual swap output is below the user's tolerance.

```solidity
    function _call1InchSwap(bytes memory _swapParams, address _asset, uint256 _minReturnAmount)
        internal
        returns (uint256 returnAmount)
    {
        (bool success, bytes memory result) = address(oneInchRouter).call(_swapParams);
        require(success, "1inch swap failed");

        if (result.length > 0) {
            (returnAmount,) = abi.decode(result, (uint256, uint256));
        } else {
// @>       returnAmount = IERC20(_asset).balanceOf(address(this));
        }
        require(returnAmount >= _minReturnAmount, "Insufficient return amount from swap");
    }
```

## Risk

**Likelihood**:

* The 1inch router returns empty bytes on a successful call — this is a legitimate response from certain router versions and integration modes, not an error condition.

* The contract accumulates a residual token balance from prior operations (partial unwinds, leftover swap amounts, tokens sent directly), which is a realistic state for any actively-used contract.

**Impact**:

* A swap that returns fewer tokens than the user's `_minReturnAmount` proceeds without revert, silently delivering less value than the user specified as acceptable.

* In the extreme case, the router transfers zero tokens to the contract (complete swap failure with an empty-bytes success response), and the check still passes as long as the pre-existing balance exceeds `_minReturnAmount`, leaving the user's debt unrepaid and their collateral stuck.

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {Stratax} from "../../src/Stratax.sol";
import {UpgradeableBeacon} from "@openzeppelin/contracts/proxy/beacon/UpgradeableBeacon.sol";
import {BeaconProxy} from "@openzeppelin/contracts/proxy/beacon/BeaconProxy.sol";

contract MockERC20 {
    mapping(address => uint256) public balanceOf;
    function mint(address to, uint256 amount) external { balanceOf[to] += amount; }
    function decimals() external pure returns (uint8) { return 18; }
    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount; balanceOf[to] += amount; return true;
    }
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        balanceOf[from] -= amount; balanceOf[to] += amount; return true;
    }
    function approve(address, uint256) external pure returns (bool) { return true; }
}

/// @notice 1inch mock: transfers actualOutput tokens and returns empty bytes.
contract MockOneInchEmpty {
    MockERC20 public token;
    uint256 public actualOutput;
    constructor(MockERC20 _token, uint256 _out) { token = _token; actualOutput = _out; }
    fallback() external {
        if (actualOutput > 0) token.transfer(msg.sender, actualOutput);
        assembly { return(0, 0) }
    }
}

contract StrataxHarness is Stratax {
    function exposed_call1InchSwap(bytes memory _swapParams, address _asset, uint256 _min)
        external returns (uint256)
    { return _call1InchSwap(_swapParams, _asset, _min); }
}

contract SlippageBypassPoCTest is Test {
    uint256 constant PRE_EXISTING = 200e18;
    uint256 constant ACTUAL_OUTPUT = 600e18;
    uint256 constant MIN_RETURN = 700e18;

    /// @notice actual_output (600) < MIN_RETURN (700), but check passes due to pre-existing 200.
    function test_slippageBypass() public {
        MockERC20 token = new MockERC20();
        MockOneInchEmpty router = new MockOneInchEmpty(token, ACTUAL_OUTPUT);
        token.mint(address(router), ACTUAL_OUTPUT);

        StrataxHarness impl = new StrataxHarness();
        UpgradeableBeacon beacon = new UpgradeableBeacon(address(impl), address(this));
        bytes memory init = abi.encodeWithSelector(
            Stratax.initialize.selector, address(0), address(0), address(router), address(token), address(0)
        );
        StrataxHarness harness = StrataxHarness(address(new BeaconProxy(address(beacon), init)));

        // Pre-existing balance seeds the contract
        token.mint(address(harness), PRE_EXISTING);

        // actual_output < MIN_RETURN, but should NOT revert — slippage check bypassed
        uint256 returned = harness.exposed_call1InchSwap(abi.encodeWithSignature("swap()"), address(token), MIN_RETURN);

        assertLt(ACTUAL_OUTPUT, MIN_RETURN, "actual output is below user minimum");
        assertEq(returned, PRE_EXISTING + ACTUAL_OUTPUT, "returnAmount inflated by pre-existing balance");
        assertTrue(returned >= MIN_RETURN, "slippage check passed despite bad swap rate");
    }
}
```

## Recommended Mitigation

```diff
    function _call1InchSwap(bytes memory _swapParams, address _asset, uint256 _minReturnAmount)
        internal
        returns (uint256 returnAmount)
    {
+       uint256 balanceBefore = IERC20(_asset).balanceOf(address(this));
        (bool success, bytes memory result) = address(oneInchRouter).call(_swapParams);
        require(success, "1inch swap failed");

        if (result.length > 0) {
            (returnAmount,) = abi.decode(result, (uint256, uint256));
        } else {
-           returnAmount = IERC20(_asset).balanceOf(address(this));
+           returnAmount = IERC20(_asset).balanceOf(address(this)) - balanceBefore;
        }
        require(returnAmount >= _minReturnAmount, "Insufficient return amount from swap");
    }
```


