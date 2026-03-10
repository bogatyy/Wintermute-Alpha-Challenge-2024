# Challenge 06 - Stale AMM: Solution

## Key Addresses

- **Old TUSD token**: [`0x8dd5fbCe2f6a956C3022bA3663759011Dd51e73E`](https://etherscan.io/address/0x8dd5fbCe2f6a956C3022bA3663759011Dd51e73E) (labeled "Legacy Contract" on Etherscan)
- **New TUSD token (proxy)**: [`0x0000000000085d4780B73119b644AE5ecd22b376`](https://etherscan.io/token/0x0000000000085d4780B73119b644AE5ecd22b376)
- **Uniswap v1 TUSD Exchange**: [`0x4F30E682D0541eAC91748bd38A648d759261b8f3`](https://etherscan.io/address/0x4F30E682D0541eAC91748bd38A648d759261b8f3)

---

## a) Reason for the Stale Price

### Old TUSD is NOT worthless — both addresses are the same token

TrueUSD's migration used **delegation**, not replacement. The old contract's `delegate` was set to the new contract address, and both share the same `BalanceSheet` / `AllowanceSheet` storage contracts. Transfers on either address update the same state. Verified at block 14,060,000:

```
Old TUSD  balanceOf(exchange) = 2405.261049 TUSD
New TUSD  balanceOf(exchange) = 2405.261049 TUSD  ← identical
```

**1 old TUSD = 1 new TUSD = $1.** Old TUSD is fully transferable and has real value. This is why the profit is ~3x and not infinite.

### The pool's ETH price is stale

After the migration, all DEX aggregators and traders switched to the **new** TUSD address. The Uniswap v1 pool for the **old** address stopped receiving arbitrage flow, so its reserve ratio — and thus its implied ETH price — froze while ETH appreciated on the open market.

The pool was created in March 2019 when ETH was ~$137. By mid-2021 ETH was ~$2,900, but the pool's implied price remained far below market.

### The reference transaction

[`0x3f1b5ba...`](https://etherscan.io/tx/0x3f1b5baef6ea7f622834eabe7634bf89e3f473b62a73e357fdd04a1a5cf32ecf) — block 12,344,574, May 1, 2021 23:39 UTC.

An MEV bot ([`0x0000...9f56`](https://etherscan.io/address/0x0000000000007f150bd6f54c40a34d7c3d5e9f56)) sold **188.44 TUSD** into the pool and received **0.18955 ETH**.

| Metric | Value |
|--------|-------|
| TUSD sold | 188.44 TUSD ($188.44) |
| ETH received | 0.18955 ETH |
| ETH market price (Dune `prices.usd`) | **$2,952.18** |
| USD value of ETH received | **$559.56** |
| **Return multiple** | **2.97x** |
| Net profit | **$371.12** |
| Pool implied ETH price (pre-tx) | ~$854 (1168.27 TUSD / 1.3683 ETH) |

The pool was selling ETH at ~$854 while the market price was $2,952 — a 3.46x discount before slippage.

---

## b) Simulation Data for January 23, 2022

### Verified pool state (Alchemy archive node, `eth_getBalance` + `eth_call`)

| Date | Block | ETH Reserve | TUSD Reserve | Implied ETH Price | Market Price (Dune) |
|------|-------|-------------|--------------|-------------------|---------------------|
| May 1 2021 (pre-ref-tx) | 12,344,573 | 1.3683 | 1,168.27 | **$854** | $2,952 |
| May 1 2021 (post-ref-tx) | 12,344,574 | 1.1787 | 1,356.71 | $1,151 | $2,952 |
| **Jan 23, 2022** | **14,060,000** | **0.6716** | **2,405.26** | **$3,582** | **$2,404–$2,506** |
| **Mar 14, 2022** | **14,380,000** | **0.6716** | **2,405.26** | **$3,582** | **$2,530–$2,580** |

Jan 23 and Mar 14 reserves are **identical** — the pool was completely idle between those dates.

### How the pool got here: complete 2021 trade history (Dune)

All 12 trades in 2021 were TUSD→ETH (extracting ETH). Queried via EthPurchase events on `ethereum.logs`:

| Date | Tx | TUSD Sold | ETH Received |
|------|----|-----------|-------------|
| Jan 5, 2021 | [`0x63bce...`](https://etherscan.io/tx/0x63bceb788e6661f57d574194e908cbf0fa425517b7216863754df7e28dba1780) | 276.57 | 0.311 |
| Jan 26, 2021 | [`0x5566d...`](https://etherscan.io/tx/0x5566db82d97dcf3b61ad79d5b45f9aedc80880c4c7472b336eb637a55056d9c1) | 1.29 | 7.541 |
| Jan 26, 2021 | [`0x4ce5a...`](https://etherscan.io/tx/0x4ce5a95078ede579df88e58d81f1b6e0889eb62f76fd711e64fe9a94b46ffb7c) | **5,000.17** | **92.456** |
| **May 1, 2021** | [`0x3f1b5...`](https://etherscan.io/tx/0x3f1b5baef6ea7f622834eabe7634bf89e3f473b62a73e357fdd04a1a5cf32ecf) | **188.44** | **0.190** |
| May 17, 2021 | [`0xb68b8...`](https://etherscan.io/tx/0xb68b8162f537647fa6e712789d2a0c6025181fcbba5057511bf82390b0ba57b9) | 1,000 | 0.499 |
| May 20, 2021 | [`0xb577d...`](https://etherscan.io/tx/0xb577dbb3f5ad316efe7f2cf25d8c269ccb88575c72ea5e404f9d26d75e92a65d) | 7,116.89 | 0.510 |
| Jul 21, 2021 | [`0x24eaa...`](https://etherscan.io/tx/0x24eaa40a15abe1f10ef6334ce7c0fa7c4e65611c8944a83ebdda9bec539f2146) | 4.07 | 0.001 |
| Jul 22, 2021 | [`0xa039d...`](https://etherscan.io/tx/0xa039defba7c8e38a8f9761311dbc0f7990ea22e3338d9cf91382b98b3df9e824) | 200 | 0.130 |
| Aug 10, 2021 | [`0x62fcd...`](https://etherscan.io/tx/0x62fcd7034a74eaed561d5e5bfe557348ad805c1c72ea42f769978fd57ccb9778) | 450 | 0.203 |
| Sep 11, 2021 | [`0xc1647...`](https://etherscan.io/tx/0xc1647cab484fec0cad9c2fee793936c0446614f69158e7b676bd83def635a6a9) | 0.94 | 0.000 |
| Oct 19, 2021 | [`0xdffc2...`](https://etherscan.io/tx/0xdffc2f2229b8368b6c3251d32fe1345ebed73d85bbd7300e54af394692ba9fec) | 114.01 | 0.044 |
| **Dec 5, 2021** | [`0x0596e...`](https://etherscan.io/tx/0x0596e050aca99d894860251d919c8b0b9f0e80b464b59f0f1054f58e60911c30) | **296.09** | **0.094** |

Last trade: Dec 5, 2021. No activity since then. Each TUSD→ETH trade extracted more ETH and pushed the implied price higher. By late 2021, arbitrageurs had **overcorrected** the pool — its implied ETH price ($3,582) exceeded the market price ($2,400–$2,600).

### Swap simulation at January 23, 2022

Using Uniswap v1 constant-product formula: `output = (input × 997 × output_reserve) / (input_reserve × 1000 + input × 997)`

**TUSD → ETH (same direction as reference tx):**

| TUSD Sold | ETH Received | USD Value (@$2,404) | Return |
|-----------|-------------|---------------------|--------|
| 100 | 0.02673 ETH | $64.26 | **0.64x — LOSS** |
| 500 | 0.11529 ETH | $277.17 | **0.55x — LOSS** |
| 1,000 | 0.19680 ETH | $473.11 | **0.47x — LOSS** |

**ETH → TUSD (reverse direction):**

| ETH Sold | TUSD Received | Cost (@$2,404) | Return |
|----------|--------------|----------------|--------|
| 0.05 ETH | 166.20 TUSD | $120.20 | **1.38x — PROFIT** |
| 0.10 ETH | 310.92 TUSD | $240.40 | **1.29x — PROFIT** |
| 0.30 ETH | 741.14 TUSD | $721.20 | **1.03x — MARGINAL** |

---

## c) Could You Execute the Same Arbitrage on March 14, 2022?

**No. The same TUSD→ETH arbitrage is a loss at this date.**

The reserves at block 14,380,000 are identical to block 14,060,000 (0.6716 ETH, 2,405.26 TUSD) — no trades occurred between January and March 2022. The pool's implied ETH price remains ~$3,582 while the market price is $2,530–$2,580.

**TUSD → ETH at March 14, 2022:**

| TUSD Sold | ETH Received | USD Value (@$2,580) | Return |
|-----------|-------------|---------------------|--------|
| 100 | 0.02673 ETH | $68.96 | **0.69x — LOSS** |

The **reverse** direction (ETH → TUSD) is slightly profitable:

| ETH Sold | TUSD Received | Cost (@$2,580) | Return |
|----------|--------------|----------------|--------|
| 0.05 ETH | 166.20 TUSD | $129.00 | **1.29x** |
| 0.10 ETH | 310.92 TUSD | $258.00 | **1.21x** |

### Why the original arbitrage no longer works

The pool was **not drained** — it still holds 0.6716 ETH today (2026), unchanged since December 2021. Instead, the 12 TUSD→ETH trades during 2021 **overcorrected** the implied price from ~$854 (below market) to ~$3,582 (above market). The profitable arbitrage direction flipped:

- **April 2021**: Pool underpriced ETH at $854 vs $2,952 market → **sell TUSD, buy cheap ETH** (2.97x)
- **Jan–Mar 2022**: Pool overpriced ETH at $3,582 vs $2,400–$2,580 market → **sell ETH, buy cheap TUSD** (1.2–1.4x)

However, the remaining mispricing is tiny — maximum extractable profit is ~$50–100 — which barely covers gas costs on Ethereum mainnet. This explains why the pool has sat completely idle since December 2021.
