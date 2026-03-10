# Challenge 04 - Pump It (Tier 2) Solution

## a) How much revenue did Pump generate and decomposition by action

Pump.fun generates revenue from three distinct actions:

### Revenue Decomposition (through September 4, 2024):

| Revenue Source | Amount (SOL) | Number of Events | Description |
|---|---|---|---|
| **Trading Fees (Bonding Curve)** | ~570,343 SOL | 179,538,287 trades | 1% fee on each buy/sell on the bonding curve |
| **Migration Fees (Raydium Deploy)** | ~146,154 SOL | 24,359 migrations | ~6 SOL fee when a token completes its bonding curve and migrates to Raydium |
| **Token Creation Fees** | ~32,479 SOL | 1,623,935 tokens created | 0.02 SOL per token creation |
| **Total Revenue** | **~748,975 SOL** | | |

### Details on each revenue source:

**1. Trading Fees (Bonding Curve) — ~570,343 SOL**
- Total trading volume on bonding curves: ~57,034,287 SOL
- Fee rate: 1% (100 basis points) on each trade (both buys and sells)
- This is by far the largest revenue source (~76% of total)
- Fee recipient is specified in each trade instruction as `account_feeRecipient`

**2. Migration/Deployment Fees — ~146,154 SOL**
- When a token's bonding curve fills up (reaches ~85 SOL real reserves), it "completes" and migrates to Raydium
- Pump.fun collects approximately 6 SOL per migration as a deployment fee
- This includes liquidity provision to the Raydium pool
- 24,359 tokens successfully migrated out of 1,623,935 created

**3. Token Creation Fees — ~32,479 SOL**
- Each token creation costs the creator 0.02 SOL
- 1,623,935 tokens were created in the period
- This is the smallest revenue source (~4% of total)

### Dune SQL Query (Trading Fees):

```sql
SELECT
    SUM(CAST(solAmount AS double)) / 1e9 as total_sol_volume,
    SUM(CAST(solAmount AS double)) / 1e9 * 0.01 as estimated_trading_fees_sol,
    COUNT(*) as num_trades
FROM pumpdotfun_solana.pump_evt_tradeevent
WHERE evt_block_date <= DATE '2024-09-04'
```

---

## b) What percentage of tokens were successfully deployed to Raydium?

### Key Statistics:

| Metric | Value |
|---|---|
| Total tokens created | 1,623,935 |
| Tokens successfully migrated to Raydium | 23,149 – 24,359 |
| **Percentage migrated** | **~1.43%** |

Only about **1.43%** of all tokens created on pump.fun successfully filled their bonding curve and were deployed to Raydium.

### Token that took the LEAST time to deploy to Raydium:

Several tokens completed migration extremely quickly (within seconds of creation or even in the same block), suggesting coordinated buying immediately after token creation. The fastest observed was **near-instant** (0 seconds between create and complete events in the same block).

### Token that took the MOST time to deploy to Raydium:

- **Mint:** `A9PPNVNbUkbswbhqApoEsLnXAuNC6bfiMKSyr3g3ECn9`
- **Created:** May 7, 2024
- **Completed:** August 19, 2024
- **Time to migrate:** ~104.6 days (9,035,747 seconds)

This token took over 3 months to accumulate enough buying pressure to fill its bonding curve and migrate.

### Dune SQL Query (Migration Analysis):

```sql
WITH creates AS (
    SELECT mint, MIN(evt_block_time) as create_time
    FROM pumpdotfun_solana.pump_evt_createevent
    WHERE evt_block_date <= DATE '2024-09-04'
    GROUP BY mint
),
completes AS (
    SELECT mint, MIN(evt_block_time) as complete_time
    FROM pumpdotfun_solana.pump_evt_completeevent
    WHERE evt_block_date <= DATE '2024-09-04'
    GROUP BY mint
)
SELECT
    COUNT(DISTINCT c.mint) as total_created,
    COUNT(DISTINCT m.mint) as total_migrated,
    CAST(COUNT(DISTINCT m.mint) AS double) * 100.0 / COUNT(DISTINCT c.mint) as pct_migrated
FROM creates c
LEFT JOIN completes m ON c.mint = m.mint
```

---

## c) Were there cases where the pump team had a clear incentive to buy tokens?

### Analysis:

**Yes, there is a structural incentive for the pump team to buy tokens in specific situations.**

The incentive arises from the migration fee structure:

- Pump.fun collects ~6 SOL when a token completes its bonding curve and migrates
- The bonding curve completes when real SOL reserves reach approximately 85 SOL
- If a token is at, say, 80 SOL in the bonding curve (95% full), the pump team could:
  - **Buy ~5 SOL worth of tokens** to push the curve to completion
  - **Collect ~6 SOL migration fee**
  - **Net gain: ~1 SOL minus the token value they end up holding**

### When this incentive exists:

The incentive exists when:
```
Migration Fee (6 SOL) > Cost to fill remaining bonding curve + Token holding risk
```

This is most attractive when:
1. The bonding curve is >90% full (cost to complete < 8.5 SOL)
2. Trading volume is still active (so the team's purchased tokens retain some value)
3. The 1% trading fee on the final push adds additional revenue

### Example scenario:

A token is at 83 SOL in the bonding curve (97.6% full):
- **Cost to complete:** ~2 SOL in token purchases
- **Trading fee earned on own purchase:** 0.02 SOL
- **Migration fee earned:** 6 SOL
- **Net revenue:** 6 + 0.02 - 2 = **4.02 SOL profit**

The pump team essentially has an incentive to "finish off" any bonding curve that is close to completion, especially if organic buying has stalled. This creates a potential conflict of interest where the platform operator profits from pushing tokens to Raydium that may not have enough organic demand to sustain their price post-migration.

### Evidence:

To find concrete examples, one would need to:
1. Identify the pump.fun team's wallets
2. Check if they bought tokens that were near the bonding curve completion threshold
3. Correlate with subsequent migration events

```sql
-- Find tokens where the final buyer before completion could be the pump team
-- (would need to know pump team wallet addresses)
SELECT
    te.mint,
    te.user as final_buyer,
    te.sol_amount,
    te.evt_block_time,
    ce.complete_time
FROM pumpdotfun_solana.pump_evt_tradeevent te
INNER JOIN (
    SELECT mint, MIN(evt_block_time) as complete_time
    FROM pumpdotfun_solana.pump_evt_completeevent
    GROUP BY mint
) ce ON te.mint = ce.mint
WHERE te.is_buy = true
  AND te.evt_block_time <= ce.complete_time
  AND te.evt_block_time >= ce.complete_time - INTERVAL '1' MINUTE
ORDER BY te.evt_block_time DESC
LIMIT 100
```
