# Challenge 03 - Stale Oracle (Tier 1) Solution

## a) Are the prices stale according to the view of Compound v1?

**Yes, the prices are stale.** After Compound v1 was deprecated on June 3, 2019, the oracle prices stopped being actively updated. However, they remained at their last set values (non-zero) for a significant period:

### Key evidence from on-chain data:

**Oracle price snapshots around deprecation (from `moneymarket_call_assetprices`):**

| Asset | Address | Price (in ETH, 1e18 scale) | Status |
|-------|---------|---------------------------|--------|
| WETH  | 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 | 1.0 ETH (1000000000000000000) | Never changes |
| SAI (old DAI) | 0x89d24a6b4ccb1b6faa2625fe562bdd9a23260359 | ~0.00530-0.00590 ETH | Slowly updated via oracle feed |

The prices were denominated in ETH (WETH price = 1e18 = 1 ETH). The SAI/DAI price fluctuated slightly, indicating the oracle was still being intermittently updated in May 2019, but these prices became frozen/stale after deprecation.

**Timeline of key events:**
- **Sept 25, 2018:** `_suspendMarket` called for REP (0x1985...), ZRX (0xe41d...), BAT (0x0d87...), old TUSD (0x8dd5...)
- **Dec 15, 2018:** `_suspendMarket` called for SAI (0x89d2...)
- **June 3, 2019:** Official deprecation announced
- **June 4, 2019:** Last borrow ever taken (block 7895001) — borrowing was still functional!
- **Aug 29, 2020:** Last supply received (block 10753729)
- **Sept 8, 2020:** Oracle changed (block 10823857)
- **Dec 10, 2022:** Another `_suspendMarket` call for SAI
- **May 25, 2025:** Last withdrawal (block 22557681) — users still withdraw!

---

## b) Were markets paused in some way? Simulation data for borrowing on June 5, 2019

### Were markets paused?

**Partially.** The admin used `_suspendMarket()` to suspend specific assets:

1. **REP** (0x1985365e9f78359a9b6ad760e32412f4a445e862) — suspended Sept 25, 2018
2. **ZRX** (0xe41d2489571d322189246dafa5ebde1f4699f498) — suspended Sept 25, 2018
3. **BAT** (0x0d8775f648430679a709e98d2b0cb6250d2887ef) — suspended Sept 25, 2018
4. **Old TUSD** (0x8dd5fbce2f6a956c3022ba3663759011dd51e73e) — suspended Sept 25, 2018
5. **SAI/Old DAI** (0x89d24a6b4ccb1b6faa2625fe562bdd9a23260359) — suspended Dec 15, 2018

**However, ETH/WETH was NOT suspended.** There is no `_suspendMarket` call for WETH (0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2) in the decoded data.

**Critical finding:** The last borrow was on **June 4, 2019** (block 7895001) — one day AFTER the official deprecation. This proves that borrowing was still possible after the deprecation announcement.

### Could you borrow on June 5, 2019?

**It depends on the asset.** The suspended assets (REP, ZRX, BAT, old TUSD, SAI) could not be borrowed. But WETH borrowing might have been possible if:
1. The oracle still returned a valid price (it did — WETH price was 1e18)
2. The borrower had existing collateral in the protocol
3. The market was not suspended

The key constraint was **liquidity**: to borrow, there must be supply available. Since new supplies dwindled after deprecation, the available borrows were limited to existing liquidity.

### Simulation Data

To simulate borrowing any asset on June 5, 2019:

**Block number:** ~7,896,000 (June 5, 2019)
**Contract:** `0x3FDA67f7583380E67ef93072294a7fAc882FD7E7` (Compound v1 MoneyMarket)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ICompoundV1 {
    function supply(address asset, uint amount) external returns (uint);
    function borrow(address asset, uint amount) external returns (uint);
    function withdraw(address asset, uint requestedAmount) external returns (uint);
    function assetPrices(address asset) external view returns (uint);
    function markets(address asset) external view returns (
        bool isSupported,
        uint blockNumber,
        address interestRateModel,
        uint supplyRateMantissa,
        uint supplyIndex,
        uint totalSupply,
        uint borrowRateMantissa,
        uint borrowIndex,
        uint totalBorrows
    );
    function getAccountLiquidity(address account) external view returns (int);
}

// Fork at block 7896000 (June 5, 2019)
// RPC: Use an archive node endpoint
// Command: forge test --fork-url <RPC_URL> --fork-block-number 7896000

contract CompoundV1BorrowTest {
    ICompoundV1 constant compound = ICompoundV1(0x3FDA67f7583380E67ef93072294a7fAc882FD7E7);
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant SAI = 0x89d24A6b4CcB1B6fAA2625fE562bDD9a23260359;

    function checkMarketState() external view returns (
        uint wethPrice,
        uint saiPrice,
        bool wethSupported,
        uint wethTotalSupply,
        uint wethTotalBorrows,
        bool saiSupported,
        uint saiTotalSupply,
        uint saiTotalBorrows
    ) {
        wethPrice = compound.assetPrices(WETH);
        saiPrice = compound.assetPrices(SAI);

        (wethSupported,,,,, wethTotalSupply,,, wethTotalBorrows) = compound.markets(WETH);
        (saiSupported,,,,, saiTotalSupply,,, saiTotalBorrows) = compound.markets(SAI);
    }

    function attemptBorrow() external {
        // Step 1: Supply WETH as collateral
        // Need to approve WETH first
        IERC20(WETH).approve(address(compound), type(uint).max);
        uint supplyResult = compound.supply(WETH, 10 ether);

        // Step 2: Try to borrow SAI against WETH collateral
        // This will fail with error code if market is suspended
        uint borrowResult = compound.borrow(SAI, 1 ether);

        // Return codes:
        // 0 = NO_ERROR (success)
        // 4 = MARKET_NOT_SUPPORTED
        // Other = various error codes
    }
}
```

**Expected results on June 5, 2019:**
- `assetPrices(WETH)` returns `1000000000000000000` (1e18 = 1 ETH) — price is valid but stale
- `assetPrices(SAI)` returns ~`5300000000000000` (~0.0053 ETH ≈ $1.40/SAI at ~$265/ETH) — stale
- SAI market `isSupported` may return `false` (was suspended Dec 2018)
- WETH market `isSupported` should return `true` (never suspended)
- Borrowing SAI would fail because the market was suspended
- Supplying WETH would succeed — as proven by the last supply in Aug 2020
- Borrowing WETH might succeed if there is available liquidity (total supply > total borrows)

### Conclusion

The prices **are stale** but **non-zero**, meaning the protocol could theoretically process transactions. The markets were **partially paused** using `_suspendMarket()` for specific assets (REP, ZRX, BAT, TUSD, SAI), but **not all assets were paused**. The admin relied on the combination of:
1. Suspending less liquid markets
2. Changing the oracle (Sept 2020)
3. The natural drain of liquidity as users withdrew
4. Not explicitly pausing the protocol globally (there is no global pause function)

The fact that the last borrow occurred on June 4, 2019 (1 day after deprecation) demonstrates that the protocol was not immediately paused and stale prices could theoretically have been exploited, though limited liquidity would cap any potential attack.
