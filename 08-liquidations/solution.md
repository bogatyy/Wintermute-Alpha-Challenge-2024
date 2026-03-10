# Challenge 08 - Liquidations (Tier 3) Solution

## Background

The referenced transaction: `0xec4f2ab36afa4fac4ba79b1ca67165c61c62c3bb6a18271c18f42a6bdfdb533d`

### Transaction Details (from on-chain data):
- **Block:** 10,692,540 (August 19, 2020)
- **Sender (EOA):** `0x8329f48bd4c7a1cf43a434dda99a96c605d6bf7b`
- **Liquidator Contract:** `0x88886841cfccbf54adbbc0b6c9cbaceabec42b8b`
- **Gas Used:** 501,020
- **Gas Price:** 171.1 Gwei
- **Function Selector:** `0x3743cb3f` (custom, not matching any standard interface)
- **Calldata Length:** 2,116 bytes (very large for a liquidation)

### Decoded Transaction Flow (from event log analysis):

| Step | Action | Contract | Details |
|------|--------|----------|---------|
| 1 | Oracle price update | `0xc629c26d...` (Open Price Feed) | Price posted for `0xfceadafab14d46e20144f48824d0c09b1a03f2bc` |
| 2 | Second oracle update | `0x9b8eb8b3...` (Coinbase Oracle) | Additional price data signed and anchored |
| 3 | Custom event | `0x88886841...` (Liquidator) | References cDAI (`0x5d3a536e...`) — preparing liquidation |
| 4 | DAI Transfer | DAI (`0x6b175474...`) | Liquidator → cDAI: **~5,384.5 DAI** repaid |
| 5 | MakerDAO/DSS | `0x35d1b3f3...` (Vat) | DAI accounting updates (mint/burn) |
| 6 | cDAI LiquidateBorrow | cDAI (`0x5d3a536e...`) | Borrower `0x26db83c0...` liquidated |
| 7 | COMP distribution | COMP (`0xc00e94cb...`) | Rewards sent to borrower and liquidator |
| 8 | cETH Transfer | cETH (`0x4ddc2d19...`) | Borrower → Liquidator: **seized cETH collateral** |
| 9 | cDAI Redeem | cDAI (`0x5d3a536e...`) | Liquidator redeems remaining cDAI tokens |

### Key Participants:
- **Borrower:** `0x26db83c03f408135933b3cff8b7adc6a7e0adebc`
- **cToken Borrowed (repaid):** cDAI at `0x5d3a536e4d6dbd6114cc1ead35777bab948e3643`
- **cToken Collateral (seized):** cETH at `0x4ddc2d193948926d02f9b1fe9e1daa0718270ed5`
- **Underlying repaid:** DAI (`0x6b175474e89094c44da98b954eedeac495271d0f`)
- **Repay amount:** ~5,384.5 DAI (`0x12030e7985a480c0087` = 5384.5e18 wei)

Compound Governance Proposal 19 introduced the **Open Price Feed** (replacing the old admin-controlled price oracle with a decentralized Coinbase-signed price feed anchored by Uniswap TWAP). This changed the liquidation dynamics because prices now updated more predictably, making liquidation a gas-priority competition.

---

## a) What's the edge of this liquidator?

### The Liquidator's Edge: Combined Price Update + Liquidation in a Single Transaction

The key innovation in this transaction is not just gas optimization — it's that the **liquidator bundles the oracle price update with the liquidation itself**. Looking at the event logs:

1. **Steps 1-2:** The transaction first calls the Open Price Feed and Coinbase Oracle contracts to **post new price data**. This is the price update that makes the borrower's position underwater.
2. **Steps 3-9:** Immediately after updating the price, in the same transaction, the liquidator executes the liquidation.

This is a massive edge because:
- **Atomicity:** The price update and liquidation happen in the same transaction — no other liquidator can front-run between the two
- **No race condition:** Normally, a price update would be a separate transaction, and all liquidators would race to liquidate the now-underwater position. By bundling them, the liquidator eliminates competition entirely.

### Additional Advantages (from bytecode analysis):

1. **Proxy/Delegatecall pattern:** The contract uses `delegatecall` to an implementation contract at `0x1513aC508B8bd1daCa2861B4b2B79a2e9CfC6faA` (now self-destructed). This allows upgrading the liquidation logic without redeploying, and the self-destruction of the implementation obscures the actual code.

2. **GasToken2 (GST2) integration:** The bytecode contains hardcoded references to GasToken2 at `0xB3f879cb30FE243b4Dfee438691c043d8CcD0f36`. After executing the liquidation, the contract calls `freeUpTo()` to destroy pre-minted GST2 tokens, receiving a **gas refund** that significantly reduces the effective gas cost. By minting GasToken when gas was cheap and burning it during high-gas-price liquidation txs, the effective cost was much lower than competitors.

3. **Gas-optimized assembly contract:** The contract uses raw EVM bytecode with:
   - MakerDAO DSAuth-style access control (`rely`/`deny`/`wards` pattern)
   - 18+ function selectors in a custom dispatcher
   - Custom function selector `0x3743cb3f` not matching any known signature
   - Fine-tuned gas stipend management (34,710 gas subtracted from available gas in internal calls)

4. **Packed calldata format:** The 2,116 bytes of calldata are far larger than a standard `liquidateBorrow()` call (~132 bytes), because the calldata contains:
   - The Coinbase-signed price data (signatures, prices, timestamps)
   - The oracle update parameters
   - The liquidation parameters (borrower, amounts, collateral)
   All packed into a single custom-encoded payload

5. **Compound Proposal 19 context:** The Open Price Feed allows anyone to post signed Coinbase price data on-chain. The liquidator monitors Coinbase's price feed off-chain, and when a price movement would make a position underwater, they:
   - Construct the signed price update message
   - Bundle it with the liquidation call
   - Submit as a single transaction with a high gas price

6. **CFG obfuscation:** Random `JUMPDEST` opcodes, conditional jumps between unrelated code flows, and dynamic jump destinations via bitwise operations — all to prevent competitors from reverse-engineering the contract.

### Why this outcompetes other liquidators:

After Proposal 19, the price update mechanism is permissionless — anyone can post Coinbase-signed prices. But only this liquidator realized they could **bundle the price update with the liquidation** in a single atomic transaction. Other liquidators:
1. Wait for someone else to post the price update
2. Then race to submit a liquidation transaction
3. Have to win a gas-price auction against all other liquidators

This liquidator bypasses step 2 entirely — they ARE the price updater AND the liquidator.

---

## b) How the obfuscated calldata works

### Calldata Structure:

The 2,116-byte calldata contains multiple components packed together:

```
[4-byte function selector: 0x3743cb3f]
[Coinbase price update data: ~600-800 bytes]
    - Signed price messages from Coinbase
    - Timestamps
    - ETH/USD price
    - Other asset prices
[Oracle anchor data: ~400-600 bytes]
    - Uniswap TWAP anchor parameters
[Liquidation parameters: ~92-132 bytes]
    - borrower: 0x26db83c03f408135933b3cff8b7adc6a7e0adebc
    - cTokenBorrow: 0x5d3a536e4d6dbd6114cc1ead35777bab948e3643 (cDAI)
    - cTokenCollateral: 0x4ddc2d193948926d02f9b1fe9e1daa0718270ed5 (cETH)
    - repayAmount: ~5384.5 DAI
```

### Why it looks "obfuscated":

1. **Custom function selector** — `0x3743cb3f` doesn't match any known standard function signature
2. **Bundled operations** — the calldata contains both oracle update data AND liquidation params in one payload
3. **No standard ABI encoding** — the contract reads calldata directly via `CALLDATALOAD`, using custom offsets rather than Solidity ABI decoding
4. **Signed price data** — the Coinbase price signatures are raw bytes that look like random data to tools expecting ABI-encoded parameters
5. **Assembly-level optimizations** — the contract skips memory allocation, ABI encoding/decoding overhead, and uses direct stack manipulation

### Decoding the contract logic (pseudocode):

```
FUNCTION 0x3743cb3f(calldata):
    // Phase 1: Parse and post price data to Open Price Feed
    priceData = CALLDATALOAD(4)      // Read signed Coinbase price data
    CALL(OpenPriceFeed, "postPrices(bytes)", priceData)

    // Phase 2: Anchor prices via Uniswap TWAP
    anchorData = CALLDATALOAD(offset1)
    CALL(UniswapAnchoredView, "validate(bytes)", anchorData)

    // Phase 3: Execute liquidation
    borrower = CALLDATALOAD(offset2) >> 96     // 20 bytes
    cTokenBorrow = CALLDATALOAD(offset3) >> 96  // 20 bytes (cDAI)
    cTokenCollateral = CALLDATALOAD(offset4) >> 96 // 20 bytes (cETH)
    repayAmount = CALLDATALOAD(offset5)          // 32 bytes

    // Get underlying token (DAI) from cTokenBorrow (cDAI)
    underlying = STATICCALL(cTokenBorrow, "underlying()")

    // Approve DAI for cDAI
    CALL(underlying, "approve(address,uint256)", cTokenBorrow, repayAmount)

    // Liquidate: repay DAI debt, seize cETH collateral
    CALL(cTokenBorrow, "liquidateBorrow(address,uint256,address)",
         borrower, repayAmount, cTokenCollateral)

    // Redeem seized cETH for ETH
    seizedBalance = STATICCALL(cTokenCollateral, "balanceOf(address)", THIS)
    CALL(cTokenCollateral, "redeem(uint256)", seizedBalance)
```

---

## c) Solidity bot code + simulation data

### LiquidationBot.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ICToken {
    function liquidateBorrow(address borrower, uint repayAmount, address cTokenCollateral) external returns (uint);
    function underlying() external view returns (address);
    function redeem(uint redeemTokens) external returns (uint);
    function balanceOf(address owner) external view returns (uint);
}

interface IERC20 {
    function approve(address spender, uint amount) external returns (bool);
    function transfer(address to, uint amount) external returns (bool);
    function balanceOf(address owner) external view returns (uint);
    function transferFrom(address from, address to, uint amount) external returns (bool);
}

interface ICEth {
    function liquidateBorrow(address borrower, address cTokenCollateral) external payable;
    function redeem(uint redeemTokens) external returns (uint);
    function balanceOf(address owner) external view returns (uint);
}

interface IUniswapAnchoredView {
    function postPrices(bytes calldata messages, bytes calldata signatures, string[] calldata symbols) external;
}

interface IComptroller {
    function getAccountLiquidity(address account) external view returns (uint, uint, uint);
}

contract LiquidationBot {
    address public immutable owner;

    // Compound v2 addresses
    address constant COMPTROLLER = 0x3d9819210A31b4961b30EF54bE2aeD79B9c9Cd3B;
    address constant CDAI = 0x5d3a536E4D6DbD6114cc1Ead35777bAB948E3643;
    address constant CETH = 0x4Ddc2D193948926D02f9B1fE9e1daa0718270ED5;
    address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;

    constructor() {
        owner = msg.sender;
    }

    /// @notice Liquidate a Compound v2 position using packed calldata
    /// @dev Calldata format: [borrower:20][cTokenBorrow:20][cTokenCollateral:20][repayAmount:32]
    fallback() external payable {
        address borrower;
        address cTokenBorrow;
        address cTokenCollateral;
        uint256 repayAmount;

        assembly {
            // Load packed addresses (20 bytes each)
            borrower := shr(96, calldataload(0))
            cTokenBorrow := shr(96, calldataload(20))
            cTokenCollateral := shr(96, calldataload(40))
            repayAmount := calldataload(60)
        }

        // Get underlying token
        address underlying = ICToken(cTokenBorrow).underlying();

        // Approve underlying for liquidation
        IERC20(underlying).approve(cTokenBorrow, repayAmount);

        // Execute liquidation
        uint result = ICToken(cTokenBorrow).liquidateBorrow(borrower, repayAmount, cTokenCollateral);
        require(result == 0, "Liquidation failed");

        // Redeem seized collateral
        uint seizedCTokens = ICToken(cTokenCollateral).balanceOf(address(this));
        if (seizedCTokens > 0) {
            ICToken(cTokenCollateral).redeem(seizedCTokens);
        }
    }

    /// @notice Higher-level function that also posts oracle prices before liquidating
    /// Mimics the bundled price update + liquidation pattern
    function liquidateWithPriceUpdate(
        bytes calldata priceMessages,
        bytes calldata priceSignatures,
        string[] calldata symbols,
        address oracle,
        address borrower,
        address cTokenBorrow,
        address cTokenCollateral,
        uint256 repayAmount
    ) external {
        require(msg.sender == owner, "only owner");

        // Step 1: Post price update to the Open Price Feed
        IUniswapAnchoredView(oracle).postPrices(priceMessages, priceSignatures, symbols);

        // Step 2: Execute liquidation
        address underlying = ICToken(cTokenBorrow).underlying();
        IERC20(underlying).approve(cTokenBorrow, repayAmount);
        uint result = ICToken(cTokenBorrow).liquidateBorrow(borrower, repayAmount, cTokenCollateral);
        require(result == 0, "Liquidation failed");

        // Step 3: Redeem seized collateral
        uint seizedCTokens = ICToken(cTokenCollateral).balanceOf(address(this));
        if (seizedCTokens > 0) {
            ICToken(cTokenCollateral).redeem(seizedCTokens);
        }
    }

    /// @notice Withdraw profits
    function withdraw(address token) external {
        require(msg.sender == owner);
        uint bal = IERC20(token).balanceOf(address(this));
        if (bal > 0) {
            IERC20(token).transfer(owner, bal);
        }
        uint ethBal = address(this).balance;
        if (ethBal > 0) {
            payable(owner).transfer(ethBal);
        }
    }

    /// @notice Fund the contract with tokens needed for liquidation
    function fund(address token, uint amount) external {
        IERC20(token).transferFrom(msg.sender, address(this), amount);
    }

    receive() external payable {}
}
```

### Foundry Simulation Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";

interface IComptroller {
    function getAccountLiquidity(address account) external view returns (uint, uint, uint);
    function liquidateCalculateSeizeTokens(
        address cTokenBorrowed,
        address cTokenCollateral,
        uint actualRepayAmount
    ) external view returns (uint, uint);
}

interface ICToken {
    function liquidateBorrow(address borrower, uint repayAmount, address cTokenCollateral) external returns (uint);
    function underlying() external view returns (address);
    function borrowBalanceCurrent(address account) external returns (uint);
    function balanceOf(address owner) external view returns (uint);
    function redeem(uint redeemTokens) external returns (uint);
}

interface IERC20 {
    function balanceOf(address) external view returns (uint);
    function approve(address, uint) external returns (bool);
    function transfer(address, uint) external returns (bool);
}

contract LiquidationSimTest is Test {
    // Compound v2 Addresses
    address constant COMPTROLLER = 0x3d9819210A31b4961b30EF54bE2aeD79B9c9Cd3B;
    address constant CDAI = 0x5d3a536E4D6DbD6114cc1Ead35777bAB948E3643;
    address constant CETH = 0x4Ddc2D193948926D02f9B1fE9e1daa0718270ED5;
    address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;

    // From the actual transaction:
    address constant LIQUIDATOR_CONTRACT = 0x88886841CfCcBf54AdBbC0b6C9CBAcEAbEC42B8b;
    address constant BORROWER = 0x26dB83C03f408135933b3CFF8b7aDC6a7e0ADebc;
    uint256 constant REPAY_AMOUNT = 5384500000000000000000; // ~5384.5 DAI

    function setUp() public {
        // Fork at block 10,692,539 (one block BEFORE the liquidation tx)
        // This lets us see the state just before the price update + liquidation
        vm.createSelectFork("mainnet", 10692539);
    }

    function testCheckBorrowerState() public {
        // Check borrower's account liquidity
        (uint err, uint liquidity, uint shortfall) = IComptroller(COMPTROLLER)
            .getAccountLiquidity(BORROWER);

        emit log_named_uint("Error", err);
        emit log_named_uint("Liquidity (USD, 1e18)", liquidity);
        emit log_named_uint("Shortfall (USD, 1e18)", shortfall);

        // Before the price update, borrower may not be underwater yet
        // The price update in the tx is what makes them liquidatable
    }

    function testSimulateLiquidation() public {
        // Give ourselves DAI to repay
        deal(DAI, address(this), REPAY_AMOUNT);

        uint daiBalance = IERC20(DAI).balanceOf(address(this));
        emit log_named_uint("Our DAI balance", daiBalance);

        // Approve cDAI to spend our DAI
        IERC20(DAI).approve(CDAI, REPAY_AMOUNT);

        // Check if borrower is underwater (may need price update first)
        (uint err, uint liquidity, uint shortfall) = IComptroller(COMPTROLLER)
            .getAccountLiquidity(BORROWER);
        emit log_named_uint("Borrower shortfall before", shortfall);

        if (shortfall > 0) {
            // Execute liquidation: repay DAI debt, seize cETH collateral
            uint result = ICToken(CDAI).liquidateBorrow(BORROWER, REPAY_AMOUNT, CETH);
            emit log_named_uint("Liquidation result (0=success)", result);

            // Check seized cETH
            uint cethBalance = ICToken(CETH).balanceOf(address(this));
            emit log_named_uint("Seized cETH", cethBalance);

            // Redeem cETH for ETH
            if (cethBalance > 0) {
                uint ethBefore = address(this).balance;
                ICToken(CETH).redeem(cethBalance);
                uint ethAfter = address(this).balance;
                emit log_named_uint("ETH received from redeem", ethAfter - ethBefore);
            }
        } else {
            emit log_string("Borrower is not underwater at this block - needs price update first");
            emit log_string("The original tx bundled a price update to MAKE the position underwater");
        }
    }

    // Check the borrower's borrow balance
    function testBorrowerDebt() public {
        uint borrowBalance = ICToken(CDAI).borrowBalanceCurrent(BORROWER);
        emit log_named_uint("Borrower DAI debt", borrowBalance);

        uint cethBalance = ICToken(CETH).balanceOf(BORROWER);
        emit log_named_uint("Borrower cETH collateral", cethBalance);
    }

    receive() external payable {}
}
```

### Simulation Commands:

```bash
# Fork at block just before the liquidation
forge test \
    --fork-url $ETH_RPC_URL \
    --fork-block-number 10692539 \
    -vvv \
    --match-test testCheckBorrowerState

# Simulate the full liquidation
forge test \
    --fork-url $ETH_RPC_URL \
    --fork-block-number 10692539 \
    -vvv \
    --match-test testSimulateLiquidation

# Check borrower's debt and collateral
forge test \
    --fork-url $ETH_RPC_URL \
    --fork-block-number 10692539 \
    -vvv \
    --match-test testBorrowerDebt
```

### Complete Simulation Parameters:

| Parameter | Value |
|-----------|-------|
| Fork block | 10,692,539 (one before the liquidation) |
| Liquidator contract | `0x88886841cfccbf54adbbc0b6c9cbaceabec42b8b` |
| Borrower | `0x26db83c03f408135933b3cff8b7adc6a7e0adebc` |
| cToken borrowed | cDAI: `0x5d3a536e4d6dbd6114cc1ead35777bab948e3643` |
| cToken collateral | cETH: `0x4ddc2d193948926d02f9b1fe9e1daa0718270ed5` |
| Underlying repaid | DAI: `0x6b175474e89094c44da98b954eedeac495271d0f` |
| Repay amount | ~5,384.5 DAI |
| Comptroller | `0x3d9819210a31b4961b30ef54be2aed79b9c9cd3b` |
| Open Price Feed | `0xc629c26dced4277419cde234012f8160a0278a79` |
| Coinbase Oracle | `0x9b8eb8b3d6e2e0db36f41455185fef7049a35cae` |
| COMP token | `0xc00e94cb662c3520282e6f5717214004a7f26888` |

### Note on the Bundled Price Update:

The borrower at `0x26db83c0...` was likely close to but not yet underwater at block 10,692,539. The liquidator's transaction at block 10,692,540 first posted new price data (likely an ETH price drop) via the Open Price Feed, which made the borrower's position underwater, and then immediately liquidated them. This is why checking the borrower's shortfall at the prior block may show `shortfall = 0` — the price update within the same transaction is what triggers the liquidation eligibility.
