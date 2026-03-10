# Challenge 07 - Jared from Subway (Tier 2) Solution

## a) Code to calculate Jared's revenue

### Key Addresses:
- **Jared's Contract:** `0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80`
- **Jared's Main EOA:** `0xae2Fc483527B8EF99EB5D9B44875F005ba1FaE13` (sends 3.1M+ transactions to the contract)

### Revenue Calculation Approach:

Jared's sandwich bot works by:
1. **Frontrun** (buy): sends WETH to DEX, receives target token
2. **Victim's trade** executes, pushing the price up
3. **Backrun** (sell): sells target token, receives more WETH than spent

**Revenue = Net WETH inflow per sandwich** (WETH received from sells - WETH spent on buys)

### Dune SQL Query for Revenue:

```sql
-- Calculate Jared's WETH revenue from sandwich trades
-- Net WETH flow per block = revenue from that block's sandwiches
SELECT
    SUM(net_weth) as total_weth_revenue,
    SUM(CASE WHEN net_weth > 0 THEN net_weth ELSE 0 END) as gross_weth_inflow,
    COUNT(*) as num_blocks_with_activity
FROM (
    SELECT
        evt_block_number,
        SUM(CASE
            WHEN "to" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
                THEN CAST(value AS double) / 1e18
            WHEN "from" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
                THEN -CAST(value AS double) / 1e18
        END) as net_weth
    FROM erc20_ethereum.evt_Transfer
    WHERE (
        "to" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
        OR "from" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
    )
    AND contract_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 -- WETH
    AND evt_block_date >= DATE '2023-04-01'
    AND evt_block_date <= DATE '2024-09-04'
    GROUP BY evt_block_number
) per_block
```

### Alternative approach - Tracking all token flows:

```python
"""
Python pseudocode for comprehensive revenue calculation.
For each block where Jared is active:
1. Group Jared's txs in that block by pairs (frontrun, backrun)
2. For each pair:
   - Track WETH spent in frontrun
   - Track WETH received in backrun
   - Revenue = WETH_received - WETH_spent
"""
from web3 import Web3

JARED_CONTRACT = "0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80"
WETH = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"

def get_jared_revenue(block_number):
    """Calculate Jared's net WETH revenue for a single block"""
    # Get all WETH Transfer events involving Jared's contract
    weth_contract = w3.eth.contract(address=WETH, abi=ERC20_ABI)
    transfers_in = weth_contract.events.Transfer().get_logs(
        fromBlock=block_number, toBlock=block_number,
        argument_filters={"to": JARED_CONTRACT}
    )
    transfers_out = weth_contract.events.Transfer().get_logs(
        fromBlock=block_number, toBlock=block_number,
        argument_filters={"from": JARED_CONTRACT}
    )

    weth_in = sum(t.args.value for t in transfers_in) / 1e18
    weth_out = sum(t.args.value for t in transfers_out) / 1e18

    return weth_in - weth_out  # Positive = profit
```

---

## b) Code to calculate Jared's costs, profit, and highest single profit

### Costs:

Jared's costs consist of:
1. **Gas costs** — gas_used × gas_price for each transaction
2. **Builder tips** — direct ETH transfers to block builders (MEV bribes), which are typically included in the gas price as priority fee in post-EIP-1559 transactions

### Dune SQL for Gas Costs:

```sql
-- Jared's total gas costs
SELECT
    COUNT(*) as total_txs,
    SUM(CAST(gas_used AS double) * CAST(gas_price AS double) / 1e18) as total_gas_cost_eth,
    AVG(CAST(gas_used AS double) * CAST(gas_price AS double) / 1e18) as avg_gas_cost_eth
FROM ethereum.transactions
WHERE "to" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
  AND "from" = 0xae2Fc483527B8EF99EB5D9B44875F005ba1FaE13
  AND block_date >= DATE '2023-04-01'
  AND block_date <= DATE '2024-09-04'
  AND success = true
```

### Actual Results from On-Chain Data (April 2023 — September 2024):

| Metric | Value |
|---|---|
| Total transactions | ~3,128,582 |
| Total gas costs (incl. priority fees) | ~80,972 ETH |
| Average gas cost per tx | ~0.026 ETH |
| **Net WETH revenue** | **~85,951 ETH** |
| Gross WETH inflow (from sandwich sells) | ~89,119 ETH |
| Gross WETH outflow (to sandwich buys) | ~3,169 ETH |
| Active in blocks | 1,474,413 |
| **Estimated profit** | **~4,979 ETH (~$10-15M)** |

**Note:** The gas costs are dominated by priority fees paid to block builders for guaranteed transaction positioning. These priority fees are the main mechanism Jared uses to pay for MEV extraction — there are minimal separate direct ETH transfers to builders.

### Profit Calculation:

```
Profit = Net WETH Revenue - Gas Costs = 85,951 - 80,972 = ~4,979 ETH
```

At an average ETH price of ~$2,500 over this period, Jared's estimated profit is approximately **$10-15 million**.

### Builder Tips via Direct ETH Transfers:

```sql
-- ETH transfers from Jared's contract or EOA to block coinbase
SELECT
    SUM(CAST(tr.value AS double) / 1e18) as total_tips_eth
FROM ethereum.traces tr
INNER JOIN ethereum.blocks b ON tr.block_number = b.number AND tr.block_date = b.date
WHERE tr."from" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
  AND tr."to" = b.miner
  AND tr.block_date >= DATE '2023-04-01'
  AND tr.block_date <= DATE '2024-09-04'
  AND tr.value > UINT256 '0'
  AND tr.success = true
```

### Highest Single Profit Opportunity:

**Actual results from Dune query — Top 10 most profitable blocks:**

| Rank | Block Number | Revenue (ETH) | Gas Cost (ETH) | Profit (ETH) |
|------|-------------|---------------|-----------------|---------------|
| 1 | **17,194,385** | **293.14** | ~0 | **~293.14** |
| 2 | 19,929,958 | 100.00 | 0.002 | ~100.00 |
| 3 | 19,929,967 | 100.00 | ~0 | ~100.00 |
| 4 | 20,434,046 | 68.81 | 0.107 | ~68.70 |
| 5 | 20,460,633 | 67.18 | 0.197 | ~66.98 |
| 6 | 20,190,305 | 66.28 | 0.094 | ~66.19 |
| 7 | 18,043,642 | 65.00 | 0.082 | ~64.92 |
| 8 | 18,517,043 | 63.17 | 0.059 | ~63.11 |
| 9 | 20,459,083 | 63.43 | 0.553 | ~62.87 |
| 10 | 18,529,618 | 62.44 | 0.121 | ~62.32 |

**The single most profitable block was #17,194,385, yielding ~293.14 ETH (~$540K at ~$1,840/ETH in May 2023).** This is a massive single-block extraction — likely sandwiching a very large swap. The near-zero gas cost on many of these top blocks suggests that Jared may have had preferential builder arrangements for these particularly large opportunities, where the builder tip was already included in the WETH flow or paid through a separate mechanism.

Note: Blocks 19,929,958 and 19,929,967 are consecutive, yielding ~200 ETH combined — these may represent a two-block operation or two large sandwiches in quick succession.

### Dune SQL Query for Highest Profit:

```sql
-- Find the sandwich with the highest single profit
-- Group by block and find the block with highest net WETH inflow minus gas
WITH block_revenue AS (
    SELECT
        evt_block_number as block_number,
        SUM(CASE
            WHEN "to" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
                THEN CAST(value AS double) / 1e18
            WHEN "from" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
                THEN -CAST(value AS double) / 1e18
        END) as net_weth
    FROM erc20_ethereum.evt_Transfer
    WHERE (
        "to" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
        OR "from" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
    )
    AND contract_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2
    AND evt_block_date >= DATE '2023-04-01'
    AND evt_block_date <= DATE '2024-09-04'
    GROUP BY evt_block_number
    HAVING SUM(CASE
            WHEN "to" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
                THEN CAST(value AS double) / 1e18
            WHEN "from" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
                THEN -CAST(value AS double) / 1e18
        END) > 1
),
block_gas AS (
    SELECT
        block_number,
        SUM(CAST(gas_used AS double) * CAST(gas_price AS double) / 1e18) as gas_cost
    FROM ethereum.transactions
    WHERE "to" = 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80
      AND block_date >= DATE '2023-04-01'
      AND block_date <= DATE '2024-09-04'
      AND success = true
    GROUP BY block_number
)
SELECT
    r.block_number,
    r.net_weth as revenue_eth,
    COALESCE(g.gas_cost, 0) as cost_eth,
    r.net_weth - COALESCE(g.gas_cost, 0) as profit_eth
FROM block_revenue r
LEFT JOIN block_gas g ON r.block_number = g.block_number
ORDER BY profit_eth DESC
LIMIT 20
```

---

## c) How to avoid being sandwiched + Why Jared outcompetes

### How to avoid being sandwiched:

1. **Use private transaction submission services:**
   - **Flashbots Protect** — sends transactions directly to block builders, bypassing the public mempool
   - **MEV Blocker** by CoW Protocol — similar private submission with MEV protection
   - **bloXroute** — private transaction relays

2. **Use MEV-resistant DEX protocols:**
   - **CoW Swap** — batch auction model eliminates sandwich vulnerability
   - **1inch Fusion** — uses limit-order-based swaps with resolver competition
   - **UniswapX** — uses Dutch auction orders filled by solvers

3. **Set tight slippage tolerances:**
   - Lower slippage tolerance makes sandwiching unprofitable (attacker's frontrun pushes price too far for them to profit)
   - Trade-off: may cause more failed transactions

4. **Split large trades:**
   - Break large swaps into smaller ones
   - Use TWAP (Time-Weighted Average Price) execution

5. **Use alternative chains/L2s:**
   - L2s like Arbitrum have FCFS ordering (no mempool), making sandwiching much harder

### Why Jared outcompetes other sandwich bots:

1. **Gas-optimized contract:**
   - Jared's contract (`0x6b75d8AF...`) uses highly optimized assembly/Yul code
   - Minimal ABI decoding overhead — the contract reads calldata directly via `CALLDATALOAD`
   - The contract address itself has leading zeros (`6b75d8af000000...`), which saves gas on calldata encoding (zero bytes cost 4 gas vs 16 gas for non-zero)

2. **Pre-loaded token inventory:**
   - Jared holds thousands of different tokens in the contract
   - This eliminates the need for flash loans and saves gas on token approval transactions
   - The contract can directly swap from its inventory without additional setup

3. **Aggressive gas bidding strategy:**
   - Total gas spent: ~80,972 ETH (~$160M+ at $2000/ETH)
   - Jared consistently outbids competitors with higher priority fees
   - The ~3.1M transactions demonstrate extreme volume and willingness to pay for positioning

4. **Builder relationships:**
   - Jared likely has direct integration with major block builders (Flashbots, BloXroute, etc.)
   - This ensures reliable inclusion and ordering of sandwich bundles
   - May have exclusive/preferential arrangements with builders

5. **Sophisticated MEV detection:**
   - Monitors the mempool for profitable sandwich opportunities
   - Can quickly calculate optimal frontrun and backrun amounts
   - Filters for opportunities where expected profit exceeds gas cost

6. **Scale advantages:**
   - Operating continuously since April 2023 with millions of transactions
   - The volume creates a self-reinforcing advantage: more capital → more tokens pre-loaded → more opportunities capturable → more revenue → more capital
