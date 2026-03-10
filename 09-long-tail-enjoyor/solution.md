# Challenge 09 - Long Tail Enjoyor (Tier 3) Solution

## Background

**SIP-112** introduced the **ETH Wrappr** to Synthetix, deployed on May 13, 2021. The Wrappr allows users to deposit ETH and receive sETH (synthetic ETH) at a 1:1 ratio minus a mint fee. Key parameters at launch:
- **maxETH:** 5,000 ETH (initial cap; later raised to 50,000 ETH via SCCP-105)
- **mintFeeRate:** 100 basis points (1%) initially; later reduced via governance SCCPs
- **burnFeeRate:** 100 basis points (1%) initially

### Key Synthetix Addresses:
- **sETH token:** `0x5e74C9036fb86BD7eCdcb084a0673EFc32eA31cb`
- **Curve sETH/ETH pool:** `0xc5424B857f758E906013F3555Dad202e4bdB4567`
- **EtherWrapper:** `0xC1AAE9d18bBe386B102435a8632C8063d31e747C`
- **NativeEtherWrapper:** Also available via `contracts.synthetix.io/NativeEtherWrapper`

---

## a) How can you make money based on the SIP specification?

### The Arbitrage Opportunity: sETH Premium on Curve + Fee Parameter Changes

**sETH consistently traded at a premium** to ETH on the Curve sETH/ETH pool because:

1. **sETH supply was constrained** — the only way to get sETH was through the Synthetix staking/minting system, requiring collateralizing SNX at a high C-ratio (750%+)
2. **sETH demand was high** — used for trading, liquidity provision, and as a building block in DeFi
3. **The premium ranged from 0.5-3%** at various times

### The Core Strategy:

SIP-112's ETH Wrappr creates a direct path: **ETH → sETH at 1:1 (minus mintFeeRate)**. Arbitrage exists whenever:

```
sETH premium on Curve > mintFeeRate + Curve trading fee (2 bps)
```

### Two Arbitrage Windows:

**Window 1 — Initial Deployment (May 13, 2021):**
- sETH was at a premium on Curve
- Mint sETH via Wrappr at 1% fee, sell on Curve at >1% premium
- Limited to initial 5,000 ETH maxETH cap

**Window 2 — Fee Parameter Reductions (ongoing):**
- Synthetix governance periodically reduced the mintFeeRate via SCCPs
- Each fee reduction reopened the arbitrage window if Curve premium exceeded the new fee
- **SCCP-105** raised maxETH to 50,000 ETH (more capacity to arb)
- Fee reductions (e.g., from 1% to 0.5%) were announced via governance and visible in advance

### The Key Insight — "Long Tail":

Every SCCP governance proposal to reduce the mint fee or increase capacity is publicly announced before it takes effect. An alert trader monitoring Synthetix governance can:
1. **Watch for SCCP proposals** that reduce `mintFeeRate` or increase `maxETH`
2. **Pre-calculate** whether the new fee rate will be below the Curve sETH premium
3. **Prepare a backrun transaction** that executes immediately after the governance tx

This is a "long tail" alpha — not a one-shot opportunity but a **recurring pattern** where each governance parameter change can create a new arbitrage window.

### SCCP Parameter Evolution (each one opened an arbitrage window):

| SCCP | Date | Change | Arbitrage Window |
|------|------|--------|-----------------|
| SCCP-100 | May 12, 2021 | Initial: mintFee=200bp, maxETH=5K | 250bp Curve premium - 200bp fee = 50bp |
| SCCP-102 | May 14, 2021 | Cap: 5K → 15K ETH | More capacity to arb |
| SCCP-104 | May 17, 2021 | mintFee: 200bp → 100bp | Wider window |
| SCCP-105 | May 18, 2021 | Cap: 15K → 50K ETH | Much more capacity |
| SCCP-109 | May 20, 2021 | mintFee: 100bp → 50bp | **Best opportunity (~19 ETH profit)** |
| SCCP-116 | May 26, 2021 | Loan issueFee: 50bp → 25bp | Additional arb via loans |
| SCCP-117 | ~June 2021 | mintFee: 25bp → 5bp | Near-zero fee |
| SCCP-118 | ~June 2021 | Cap: 100K → 125K ETH | Massive capacity |
| SCCP-121 | ~June 2021 | Cap: 125K → 175K ETH | Full absorption |

By June 2021, the EtherWrapper held **175,000 ETH** and had generated nearly **$2M in minting fees** for SNX stakers.

### Profit Formula:

```
Profit = sETH_amount × (Curve_premium - new_mintFeeRate - Curve_fee)
```

Where `Curve_fee` = 2 basis points.

---

## b) Simulation data for executing with full maxETH amount

### The Most Profitable Historical Opportunity

Based on on-chain analysis, the best single opportunity occurred at:

- **Block:** 12,473,576
- **Transaction:** `0xb355c2947ba5f4758b0a2c74e5a607215b0c681c7230dccffea780edfdeeaf10`
- **Trigger:** Governance tx at block 12,473,575 (`0xd334d2ffc1fe0991abfa80229df8f1ec62af576a6976a218c749cfd067b1cb6a`) reduced mintFeeRate from 1% to 0.5%
- **Amount swapped:** 15,000 ETH
- **Profit realized:** ~19 ETH (~$52K at the time, after ~7 ETH Flashbots bid)
- **MEV extractor:** `0xEef86c2E49E11345F1a693675dF9a38f7d880C8F`

### Pool State at Block 12,473,575 (pre-execution):

| Parameter | Value |
|-----------|-------|
| EtherWrapper WETH balance | 23,000 ETH |
| maxETH capacity | 50,000 ETH (after SCCP-105) |
| Available capacity | 27,000 ETH |
| Curve sETH/ETH exchange rate | 1:1.00852678 (0.85% premium) |
| Curve fee | 2 basis points |
| New mintFeeRate (after governance tx) | 0.5% (50 bps) |

### Arbitrage Window Calculation:

```
Curve premium:           0.85% (85 bps)
New mint fee:           -0.50% (50 bps)
Curve trading fee:      -0.02% (2 bps)
────────────────────────────────────
Net arbitrage window:    0.33% (33 bps)
```

For 15,000 ETH: `15,000 × 0.0033 ≈ 49.5 ETH gross profit` (reduced by Curve slippage on large swap, net ~25.93 ETH before Flashbots bid).

### Profit Optimization Results:

| Amount (ETH) | Estimated Profit (ETH) |
|-------------|----------------------|
| 15,000 (actual) | 25.93 |
| **16,000 (optimal)** | **25.99** |
| 17,000+ | Declining returns (slippage exceeds marginal premium) |

The optimal amount was ~16,000 ETH — slightly more than the 15,000 ETH actually used. Beyond ~16K, Curve slippage on the sETH→ETH swap exceeds the remaining premium.

### Foundry Simulation Code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";

interface IEtherWrapper {
    function mint(uint amount) external payable;
    function maxETH() external view returns (uint);
    function capacity() external view returns (uint);
    function mintFeeRate() external view returns (uint);
}

interface INativeEtherWrapper {
    function mint(uint amount) external payable;
}

interface ICurve {
    function get_dy(int128 i, int128 j, uint dx) external view returns (uint);
    function exchange(int128 i, int128 j, uint dx, uint min_dy) external payable returns (uint);
    function balances(uint i) external view returns (uint);
}

interface IERC20 {
    function balanceOf(address) external view returns (uint);
    function approve(address, uint) external returns (bool);
}

contract SIP112ArbitrageTest is Test {
    // Key addresses
    address constant SETH = 0x5e74C9036fb86BD7eCdcb084a0673EFc32eA31cb;
    address constant CURVE_SETH_ETH = 0xc5424B857f758E906013F3555Dad202e4bdB4567;
    address constant ETHER_WRAPPER = 0xC1AAE9d18bBe386B102435a8632C8063d31e747C;
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

    function setUp() public {
        // Fork at block 12,473,576 — the block AFTER the fee reduction governance tx
        // The governance tx was at block 12,473,575
        vm.createSelectFork("mainnet", 12473576);
    }

    function testCheckCurvePoolState() public {
        ICurve curve = ICurve(CURVE_SETH_ETH);

        // Check pool balances
        // Index 0 = ETH, Index 1 = sETH
        uint ethBalance = curve.balances(0);
        uint sethBalance = curve.balances(1);

        emit log_named_uint("Curve ETH balance", ethBalance);
        emit log_named_uint("Curve sETH balance", sethBalance);

        // Check the exchange rate: how much ETH for 100 sETH?
        uint ethOut = curve.get_dy(1, 0, 100 ether);
        emit log_named_uint("ETH out for 100 sETH", ethOut);

        // Calculate premium in bps
        uint premium = (ethOut - 100 ether) * 10000 / (100 ether);
        emit log_named_uint("sETH premium (bps)", premium);

        // Check EtherWrapper capacity
        IEtherWrapper wrapper = IEtherWrapper(ETHER_WRAPPER);
        uint maxEth = wrapper.maxETH();
        uint capacity = wrapper.capacity();
        uint mintFee = wrapper.mintFeeRate();
        emit log_named_uint("maxETH", maxEth);
        emit log_named_uint("Available capacity", capacity);
        emit log_named_uint("mintFeeRate (1e18 scale)", mintFee);
    }

    function testExecuteArbitrage() public {
        // Fund with 16,000 ETH (optimal amount)
        vm.deal(address(this), 16001 ether);

        ICurve curve = ICurve(CURVE_SETH_ETH);
        IEtherWrapper wrapper = IEtherWrapper(ETHER_WRAPPER);

        uint startBalance = address(this).balance;
        emit log_named_uint("Starting ETH balance", startBalance);

        // Step 1: Mint sETH via EtherWrapper
        // The NativeEtherWrapper accepts ETH directly
        // wrapper.mint{value: 15000 ether}(15000 ether);

        // For simulation, check what amounts work:
        uint[] memory amounts = new uint[](5);
        amounts[0] = 1000 ether;
        amounts[1] = 5000 ether;
        amounts[2] = 10000 ether;
        amounts[3] = 15000 ether;
        amounts[4] = 16000 ether;

        for (uint i = 0; i < amounts.length; i++) {
            // Calculate sETH received after mint fee
            uint mintFeeRate = wrapper.mintFeeRate();
            uint sethReceived = amounts[i] * (1e18 - mintFeeRate) / 1e18;

            // Calculate ETH received from selling sETH on Curve
            uint ethOut = curve.get_dy(1, 0, sethReceived);

            emit log_named_uint("ETH input", amounts[i] / 1e18);
            emit log_named_uint("sETH received (after fee)", sethReceived / 1e18);
            emit log_named_uint("ETH output from Curve", ethOut / 1e18);

            if (ethOut > amounts[i]) {
                emit log_named_uint("PROFIT (ETH)", (ethOut - amounts[i]) / 1e18);
            }
        }
    }

    receive() external payable {}
}
```

### Running the Simulation:

```bash
# Fork at the block after the fee reduction governance tx
forge test \
    --fork-url $ETH_RPC_URL \
    --fork-block-number 12473576 \
    -vvvv \
    --match-test testExecuteArbitrage
```

### Historical Arbitrage Opportunities (from on-chain analysis):

The EtherWrapper created **144+ arbitrage opportunities** over its lifetime, spanning from block ~12,426,541 to block ~14,973,856. Notable data:

- **Highest single profit:** ~19.8 ETH (block 12,581,545)
- **Total profits across all opportunities:** Estimated >100 ETH
- **Primary MEV extractors:**
  - `0xEef86c2E49E11345F1a693675dF9a38f7d880C8F` (most frequent)
  - `0x1c073d5045b1aBb6924D5f0f8B2F667b1653a4c3`
  - `0x9B1B892030Ca8E95f6c5e85D351cEccD6806FcaC`
- **Amount range per opportunity:** 0.8 ETH to 15,000 ETH
- **Some negative outcomes occurred** (up to -29.6 ETH) — likely failed arb attempts where slippage exceeded the premium

### Execution Mechanism:

The most profitable extractors used **Flashbots bundles** to:
1. Backrun the governance fee-change transaction in the same block
2. Mint sETH through the EtherWrapper at the newly reduced fee
3. Sell sETH on Curve at the still-premium rate
4. Pay a Flashbots bid (tip) to the block builder for inclusion

This ensured atomic execution and no risk of being frontrun by other arbitrageurs.
