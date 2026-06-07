# QA Report

## Report summary

### Low issues

Total: **1**

| ID  | Issue |
| --- | ----- |
| L-1 | `getAssetOutAmount` / `AuctionBidder` default path use lax oracle validation while `bid` enforces strict validation |

---

## Low issues

### [L-1] `getAssetOutAmount` and `AuctionBidder.bid` (no solution) use lax oracle validation while `bid` enforces strict validation

**Files**

| File | Lines (approx.) |
| ---- | ---------------- |
| [`src/BaseAuction.sol`](https://github.com/code-423n4/2026-03-chainlink/blob/main/src/BaseAuction.sol) | 429–442, 749–766, 777–802 |
| [`src/AuctionBidder.sol`](https://github.com/code-423n4/2026-03-chainlink/blob/main/src/AuctionBidder.sol) | 75–81 |
| [`src/PriceManager.sol`](https://github.com/code-423n4/2026-03-chainlink/blob/main/src/PriceManager.sol) | 372–418 |

**Description**

`getAssetOutAmount` calls `_getAssetPrice(assetIn, false)` and `_getAssetOutAmount(..., false)`, so oracle staleness or zero price does not revert the view path; it still returns a computed amount when the auction window and balances are valid. `bid` uses `_getAssetPrice(asset, true)` and `_getAssetOutAmount(..., true)`, so the same stored prices can make `bid` revert with `Errors.StaleFeedData()` or `Errors.ZeroFeedData()` while the preview remains callable and non-zero.

`AuctionBidder.bid` with an empty `solution` approves the auction using `getAssetOutAmount(assetIn, amount, block.timestamp)` and then calls `auction.bid`. That ties the default automation path to a quote that is not aligned with the execution preconditions of `bid`. Integrators or workflows that only simulate `getAssetOutAmount` can believe a bid is executable when the next on-chain `bid` will fail until prices are refreshed, which harms composability and predictability. This is not a bypass of the auction curve invariant: `bid` still protects settlement by reverting; funds are not stolen, but liveness and integration ergonomics suffer.

**Recommendation**

- Use `_getAssetPrice(..., true)` and `_getAssetOutAmount(..., true)` inside `getAssetOutAmount`, **or**
- Add a separate strict view (e.g. `getAssetOutAmountValidated`) and use it from `AuctionBidder` for approvals.

At minimum, NatSpec for `getAssetOutAmount` and `AuctionBidder.bid` should state explicitly that callers must ensure prices are valid (e.g. via `getAssetPrice` / `isValid` and operational freshness) before relying on the preview for execution.

---

### Proof of concept

**Run:**

```solidity
    function testSubmissionValidity() public {
        _startAuction(address(mockUSDC), 100_000e6);
        skip(2 hours);

        uint256 bidAmount = 50_000e6;
        uint256 previewOut = auction.getAssetOutAmount(address(mockUSDC), bidAmount, block.timestamp);
        assertGt(previewOut, 0, "preview uses lax validation and still computes a positive assetOut amount");

        _fundBidder(previewOut * 2);

        Caller.Call[] memory emptySolution = new Caller.Call[](0);
        
        vm.startPrank(bidder);
        vm.expectRevert(Errors.StaleFeedData.selector);
        auctionBidder.bid(address(mockUSDC), bidAmount, emptySolution);
        vm.stopPrank();
    }
```

**Outline:**

1. Start a USDC auction (`_startAuction`).
2. `skip(2 hours)` without `_refreshPrices()` / `_transmitPrices` so streams exceed `stalenessThreshold`.
3. Assert `getAssetOutAmount` returns `> 0`.
4. Fund `AuctionBidder` with enough LINK; call `bid` with empty `solution` under `vm.expectRevert(Errors.StaleFeedData.selector)`.

---

### Out of scope / overlap

- Not listed under publicly known issues in `known.md` / `README.md` (FoT, rebasing, non-standard ERC20, CowSwap async tradeoffs — none describe preview vs `bid` oracle validation).
- Does **not** duplicate V12 findings F-1 through F-5 (different root cause than mutable `assetOut`, param removal, `transmit` validity window, workflow selector bypass, or allowance-after-rotation).
