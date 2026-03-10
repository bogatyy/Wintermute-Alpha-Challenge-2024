# Challenge 10 - Solana Stake: Solution

## Background

In June 2024, the Solana Foundation [announced](https://www.theblock.co/post/299244/solana-foundation-removes-certain-operators-from-delegation-program-over-malicious-sandwich-attacks) the removal of certain validators from its delegation program for facilitating sandwich attacks against users. This was a landmark enforcement action in the Solana ecosystem.

Context:
- [Mert's thread](https://x.com/0xMert_/status/1799955514786234751) explaining the situation
- Jito's decision to remove its public mempool in March 2024
- Tim Garcia's announcement on the Solana Foundation Discord

---

## a) How Validators Run Private Mempools for Sandwich Attacks

### Background: Solana's Transaction Processing Model

Unlike Ethereum, Solana does not have a traditional public mempool. Transactions are sent directly to the current and upcoming leaders (block producers) via the QUIC protocol. In theory, this makes sandwich attacks harder because there is no public staging area where pending transactions can be observed. However, validators who are the current leader have privileged access to all incoming transactions and full control over ordering within their block.

### Jito's Modified Client and the Pseudo-Mempool

**Jito Labs** developed a modified Solana validator client (`jito-solana`) that introduced MEV infrastructure to Solana:

1. **Block Engine**: Jito's off-chain infrastructure that accepted transaction bundles from searchers. When a Jito-running validator was the leader, the Block Engine would receive incoming transactions and hold them for approximately **200ms** before forwarding them to the validator.

2. **Bundle mechanism**: During this 200ms window, searchers connected to the Block Engine could observe pending transactions and submit **bundles** -- ordered groups of transactions that would be included atomically. This created a de facto mempool.

3. **Tips**: Searchers paid tips (in SOL) to validators for including their bundles, creating an economic incentive for validators to run Jito's client.

**In March 2024**, Jito removed the public mempool feature from the Block Engine due to community backlash about sandwich attacks. However, this only removed the *public* mempool -- it did not prevent validators from creating their own *private* mempools.

### How Malicious Validators Create Private Mempools

After Jito removed its public mempool, unaligned validators can still facilitate sandwich attacks through the following mechanisms:

#### Method 1: Modified Banking Stage

The `banking_stage` is the component of the Solana validator responsible for processing incoming transactions and ordering them into blocks.

```
Normal flow:
  User TX --> QUIC endpoint --> SigVerify --> Banking Stage --> Block

Malicious flow:
  User TX --> QUIC endpoint --> SigVerify --> MODIFIED Banking Stage
                                                    |
                                                    v
                                              Private Mempool
                                              (hold ~200ms)
                                                    |
                                                    v
                                              Forward to co-located
                                              MEV bot for analysis
                                                    |
                                                    v
                                              Receive sandwich bundle
                                                    |
                                                    v
                                              Include in block with
                                              controlled ordering:
                                              [front-run, victim, back-run]
```

The validator modifies the `banking_stage` to:
1. **Buffer incoming transactions** instead of processing them immediately
2. **Forward transaction data** to a co-located MEV bot (running on the same machine or a nearby server with minimal latency)
3. **Receive sandwich bundles** back from the bot
4. **Order transactions** to place the front-run before the victim and the back-run after

#### Method 2: Modified QUIC Endpoint

The validator modifies its QUIC server implementation to:
1. **Intercept transactions** at the network layer before they reach the banking stage
2. **Copy transaction data** to a sidecar MEV service
3. **Delay forwarding** to the banking stage by a few hundred milliseconds
4. This gives the MEV bot time to analyze the transaction and construct sandwich transactions

#### Method 3: Custom Transaction Scheduler

The Solana validator's scheduler determines the order of transactions within a block. A malicious validator can:
1. **Replace the default scheduler** with a custom one that prioritizes MEV extraction
2. **Group transactions by DEX pool** -- identify all swaps to the same pool
3. **Insert sandwich transactions** around profitable victim transactions
4. **Use the validator's own keypair** or a co-located bot's keypair to sign the sandwich transactions with minimal latency

#### Method 4: Private Relay Network

A validator can operate a private relay similar to Jito's Block Engine:
1. Run a **private RPC endpoint** that promises "fast inclusion" or "priority processing"
2. Users who send transactions to this endpoint unknowingly expose their transactions
3. The relay **analyzes transactions** for sandwichable swaps (large DEX trades with high slippage tolerance)
4. **Constructs and inserts sandwich bundles** when the colluding validator is the leader

### Key Technical Details

- **Leader slots**: On Solana, each validator gets consecutive leader slots (4 slots = ~1.6 seconds). The validator only has the power to sandwich during their own leader slots.
- **Transaction ordering**: Within a slot, the leader has complete discretion over transaction ordering. There is no protocol-level rule enforcing FIFO ordering.
- **Compute units**: The sandwich transactions consume compute units from the block, but since the validator controls the block, they can reserve capacity.
- **Latency advantage**: The validator's MEV bot runs co-located (same machine or same datacenter), giving it sub-millisecond latency to construct and sign sandwich transactions.

---

## b) Validators Removed and Total Stake

### The Announcement

**Tim Garcia**, Solana Foundation's Validator Relations Lead, announced on the **Solana Foundation Discord** on **June 10, 2024** (guidelines first posted May 7, 2024) that the Foundation would remove stake from validators facilitating sandwich attacks.

Key quotes from Tim Garcia:
- *"Decisions in this matter are final. Enforcement actions are ongoing as we detect operators participating in mempools which allow sandwich attacks."*
- *"Operators participating in a private mempool to sandwich attack transactions or otherwise harming Solana users will not be tolerated by the delegation program."*

### Foundation Removal Details

| Metric | Value |
|--------|-------|
| **Operators removed** | **32** |
| **Total Foundation stake removed** | **~1.5 million SOL** (~0.5% of SFDP) |
| **USD value at time** | **~$225 million** (at ~$150/SOL) |
| **Enforcement mechanism** | Withdrawal of delegated stake (no slashing) |

The Foundation **never published its specific list of 32 removed operator pubkeys publicly**. The removals were enforced silently — operators simply lost their Foundation delegation.

### Identified Validators from JIP-15 Jito Blacklist

The **JIP-15 Jito governance proposal** identified 34 validators for blacklisting from the Jito stake pool for private mempool participation (significant overlap with the Foundation's 32). These pubkeys were extracted from the proposal:

```
CJ1GZixWD1WzozqZMm3v9dY2xboRYZfQuuEHzawAthen
GRWCUtxwiSLtLGERyNyZymr77NJdko2HdDHxpVcJz6E9
jUP5hCf2fGJEz8F2j2gACezxFYCNE9zo1eTMsRQjRK9
DrCcHpAWj8a4JU99QKtwfCynzdhgQeuieAY9WadZD5Ry
5XGMWvqZSBk1fktPtxbwaMF5dhkbrtchpwd4xiXG9q8u
BfBPPqzYcqEQK9hF7AnQEJojzMtzQzB926qcPc4Y1v3L
39xF5qkfK5HBaG4Hkq6bjumUB2k4B5ozEAmfZoobhUVw
6xwWwNVXJLGhgPfBpew7UDcSjQr73McXSRK2EhdhcL1u
3NXnw51gHc2rDWTCWU4eVpP1yHyKKbJa1p6JDgMkmiDa
A1taSaBJrLMrqfWsPESYDujnZv5yD7bF35LjXoyNXhzN
HFLsfstZkJeWDgVEA5fmXeea876Ad9f1VxrbXk386bBY
HAK7iPgQTFwqEzPfVrRDbG5epFdk1dsA9qQexytsDoes
ADmLWUm2eQ3KFijFbqa4bVfiLVmW5iqjStE5b8Wbti1y
FPjq7vB2V3TiseJJSPsp47UWSfT4AwvKjiU7GEro7bX9
5VocRSwT6cqSTB8qcJ8CsSmHCmGNnohXySHWWQRfmv3a
CrLn7zEBytbmRBUGhkDyyUbGCa6H7bMCnw94Dip8QbcJ
3Rk99suwAvvJgyLpDYEJeC2YPPLw1enc5T1r6J8ZoRSr
D7ZCDE1PHe8duMjNpxwHrYbrRzcnsS7p4nD2daLzWwtr
BWkvytz3MAiLkUbMuYK5yV1VYThbBYYQYG3gdef8NLw5
725My2yzg5ZUpQtpEtivLT7JmRes2gGxF3KeGCbYACDe
BXNW9ysAB9ksEDidcNWraaFkMeA88q6xzFRSyNnGvQYC
8EfUy8zz6DF2iTMQCUe4QAnoq4jVUzfU1yvZMCr2yJ7m
AjdEzodq5LpRA63AtQaiHyTLDfjTkr6csrtYWiSDPjaK
8FPz3JG4E3HVXxGbPZVibarva4AGXSZWx3qKLUS5uFtN
D8aGMETy3q6ymADJs7YS81QY2qTQmptVox3iCfGDgSTA
DTDvrj1mKFv453DMAGRuFwg77DuLjsfVHnbLe5BJPL9D
Atw2Wond9H3DHfgg5NGqi4dxMwF3Nrgdhm39x2oK8MCW
H7JJ6aE73ufbUuDCZSQMxguQsj28XHe4VQyMk2hsVGoo
6ZUCygEHeA1MRgdxZ4V6e6gcAsJz1e2uDA3e3jv1bY7M
De4k4hrdkxFHmAx4nVRA3g5ukdg4YqmDLdwuYUcrjjud
GQ8DSRSNCFGEdCEwc6em1ma18qSPc5cXCSDnSPSznWBP
6GoijNiK3JZVAY96ykCfPhcJnrQfrQMLvrQaX5HyMVhu
2Gife8andd4BkEbT5CncpriPxmQYqbspDh8cXkN6RUSH
```

Total Jito stake affected by JIP-15: **2,453,387.62 SOL** (~15% of Jito pool). The vote passed unanimously with a 50-epoch sanction period.

### DeezNode — The Central Private Mempool Operator

The primary private mempool ("DeezMempool") was operated by **DeezNode**, offering participating validators a 50% profit split on extracted MEV:

- **Validator pubkey:** `HM5H6FAYWEMcm9PCXFbbiUFfFVLTN9UGy9AqmMQjdMRA`
- **Delegated stake:** 811,604 SOL (~$168.5M)
- **Sandwich bot program:** `vpeNALD89BZ4KxNUFjdLmFXBCwtyqBDQ85ouNoax38b`
- **30-day stats (Dec 2024):** 1.55M sandwich transactions, 88.9% success rate, 65,880 SOL profit ($13.43M)
- Accounted for roughly **half of all sandwich attacks on Solana** per Jito analysis

### Community Tracking Resources

| Resource | Coverage |
|----------|----------|
| **[a-guard/malicious-validators](https://github.com/a-guard/malicious-validators)** | **241 validators** identified, **8,575,767 SOL** total stake |
| **JIP-4 Blacklist Spreadsheet** | Initial validator blacklist with JitoSOL delegations |
| **JIP-15 Proposal** | 220+ validators targeted for blacklisting |
| **StakeWiz** | 51 validators flagged for private mempool participation |
| **Marinade Finance** | 50+ validators blacklisted, protecting ~$2B in delegated stake |
| **sandwiched.me** | Tracks per-validator sandwich rates across 8.5B analyzed trades |

### Broader Ecosystem Impact

- Sandwich bots extracted **$370M–$500M** from Solana users over a 16-month period
- **0.72%** of all Solana blocks contained sandwich attacks
- Some validators had sandwich attacks in up to **27%** of their produced blocks
- Top 5 sandwiching validators processed **41%** of MEV-heavy blocks

### Timeline

| Date | Event |
|------|-------|
| **March 2024** | Jito shuts off public mempool due to sandwich attacks during meme coin frenzy |
| **March-June 2024** | Private mempools (especially DeezMempool) emerge to fill the gap |
| **May 7, 2024** | Tim Garcia posts guidelines on Foundation Discord prohibiting malicious activities |
| **June 10, 2024** | **Solana Foundation removes 32 operators (1.5M SOL)** from delegation program |
| **June 2024** | JIP-4 proposed at Jito — authorization to blacklist validators |
| **Late 2024** | JIP-15 proposed — expanded blacklist of 220+ validators, passes unanimously |
| **2025** | Marinade blacklists 50+ validators; StakeWiz flags 51 validators |

---

## c) Code to Detect Sandwich Attacks in a Solana Block

```python
#!/usr/bin/env python3
"""
Solana Sandwich Attack Detector

Given a Solana block (slot number), this script analyzes all transactions
to identify potential sandwich attacks.

A sandwich attack consists of three transactions:
  1. FRONT-RUN: Attacker buys token X on a DEX (before victim)
  2. VICTIM: User swaps token X on the same DEX pool (pushing price up)
  3. BACK-RUN: Attacker sells token X on the same DEX pool (after victim)

Detection heuristics:
  - Same signer (attacker) for transactions 1 and 3
  - Transactions 1 and 3 bracket a victim transaction
  - All three interact with the same DEX pool
  - Transaction 1 is a buy, transaction 3 is a sell (based on token balance changes)

Usage:
  python detect_sandwich.py <slot_number>
  python detect_sandwich.py 269270524

Requirements:
  pip install solana solders requests
"""

import sys
import json
from dataclasses import dataclass, field
from typing import Optional
from solana.rpc.api import Client
from solders.pubkey import Pubkey

# Well-known DEX program IDs
DEX_PROGRAMS = {
    "675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8": "Raydium AMM V4",
    "CAMMCzo5YL8w4VFF8KVHrK22GGUsp5VTaW7grrKgrWqK": "Raydium CLMM",
    "whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc": "Orca Whirlpool",
    "9W959DqEETiGZocYWCQPaJ6sBmUzgfxXfqGeTEdp3aQP": "Orca Token Swap V2",
    "JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4": "Jupiter V6",
    "JUP4Fb2cqiRUcaTHdrPC8h2gNsA2ETXiPDD33WcGuJB": "Jupiter V4",
    "srmqPvymJeFKQ4zGQed1GFppgkRHL9kaELCbyksJtPX": "Serum/OpenBook",
    "opnb2LAfJYbRMAHHvqjCwQxanZn7ReEHp1k81EQMQvR": "OpenBook V2",
    "LBUZKhRxPF3XUpBCjp4YzTKgLccjZhTSDM9YuVaPwxo": "Meteora DLMM",
    "Eo7WjKq67rjJQSZxS6z3YkapzY3eMj6Xy8X5EQVn5UaB": "Meteora Pools",
}

# Common token mints
WSOL_MINT = "So11111111111111111111111111111111111111112"
USDC_MINT = "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
USDT_MINT = "Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB"

# RPC endpoint
SOLANA_RPC = "https://api.mainnet-beta.solana.com"


@dataclass
class TokenBalanceChange:
    """Represents a token balance change for an account in a transaction."""
    mint: str
    owner: str
    pre_amount: float
    post_amount: float
    change: float


@dataclass
class SwapInfo:
    """Extracted swap information from a transaction."""
    tx_index: int
    signature: str
    signer: str
    dex_program: str
    dex_name: str
    pool_accounts: list  # Account keys that identify the pool
    token_changes: list  # List of TokenBalanceChange
    is_buy: Optional[bool] = None  # True = buying non-SOL token, False = selling
    target_mint: Optional[str] = None  # The non-SOL/non-stable token being traded


@dataclass
class SandwichAttack:
    """A detected sandwich attack."""
    front_run: SwapInfo
    victim: SwapInfo
    back_run: SwapInfo
    attacker: str
    profit_estimate: float = 0.0


def get_block_transactions(client: Client, slot: int) -> dict:
    """Fetch all transactions in a block with full detail."""
    response = client.get_block(
        slot,
        encoding="jsonParsed",
        max_supported_transaction_version=0,
    )
    return response.value


def extract_token_balance_changes(tx) -> list:
    """Extract token balance changes from pre/post token balances."""
    changes = []

    meta = tx.transaction.meta
    if meta is None:
        return changes

    pre_balances = {}
    post_balances = {}

    # Parse pre_token_balances
    if meta.pre_token_balances:
        for bal in meta.pre_token_balances:
            key = (bal.account_index, str(bal.mint))
            owner = str(bal.owner) if bal.owner else "unknown"
            amount = float(bal.ui_token_amount.ui_amount or 0)
            pre_balances[key] = (owner, amount, str(bal.mint))

    # Parse post_token_balances
    if meta.post_token_balances:
        for bal in meta.post_token_balances:
            key = (bal.account_index, str(bal.mint))
            owner = str(bal.owner) if bal.owner else "unknown"
            amount = float(bal.ui_token_amount.ui_amount or 0)
            post_balances[key] = (owner, amount, str(bal.mint))

    # Calculate changes
    all_keys = set(pre_balances.keys()) | set(post_balances.keys())
    for key in all_keys:
        pre = pre_balances.get(key, ("unknown", 0.0, ""))
        post = post_balances.get(key, ("unknown", 0.0, ""))
        owner = post[0] if post[0] != "unknown" else pre[0]
        mint = post[2] if post[2] else pre[2]
        change = post[1] - pre[1]
        if abs(change) > 0:
            changes.append(TokenBalanceChange(
                mint=mint,
                owner=owner,
                pre_amount=pre[1],
                post_amount=post[1],
                change=change,
            ))

    return changes


def is_dex_transaction(tx) -> Optional[tuple]:
    """
    Check if a transaction interacts with a known DEX program.
    Returns (program_id, dex_name) or None.
    """
    message = tx.transaction.transaction.message
    account_keys = [str(k) for k in message.account_keys]

    # Also check inner instructions for program invocations
    for acc in account_keys:
        if acc in DEX_PROGRAMS:
            return (acc, DEX_PROGRAMS[acc])

    return None


def get_signer(tx) -> str:
    """Get the fee payer / first signer of a transaction."""
    message = tx.transaction.transaction.message
    account_keys = [str(k) for k in message.account_keys]
    return account_keys[0] if account_keys else "unknown"


def get_pool_key(tx, dex_program: str) -> str:
    """
    Extract a pool identifier from the transaction.
    For most DEXes, the pool is identified by specific accounts in the instruction.
    We use a simplified approach: hash of all non-signer accounts that interact
    with the DEX program.
    """
    message = tx.transaction.transaction.message
    account_keys = [str(k) for k in message.account_keys]

    # For Raydium AMM V4, the pool is typically accounts[1] in the swap instruction
    # For Orca Whirlpool, it's the whirlpool account
    # Simplified: use all accounts involved with the DEX as a pool fingerprint
    dex_accounts = []
    for acc in account_keys:
        if acc != dex_program and acc != get_signer(tx):
            dex_accounts.append(acc)

    # Return first few non-signer, non-program accounts as pool identifier
    # This is a heuristic; production code should parse each DEX's instruction layout
    return "|".join(sorted(dex_accounts[:5]))


def classify_swap_direction(token_changes: list, signer: str) -> tuple:
    """
    Classify whether a swap is a 'buy' or 'sell' of a non-SOL/non-stable token.
    Returns (is_buy: bool, target_mint: str)

    Buy = signer receives non-SOL token (positive change for non-SOL mint)
    Sell = signer sends non-SOL token (negative change for non-SOL mint)
    """
    stable_mints = {WSOL_MINT, USDC_MINT, USDT_MINT}

    for change in token_changes:
        if change.owner == signer and change.mint not in stable_mints:
            if change.change > 0:
                return (True, change.mint)  # Buying the token
            elif change.change < 0:
                return (False, change.mint)  # Selling the token

    return (None, None)


def extract_swap_info(tx, tx_index: int) -> Optional[SwapInfo]:
    """Extract swap information from a transaction if it's a DEX swap."""
    dex_info = is_dex_transaction(tx)
    if dex_info is None:
        return None

    meta = tx.transaction.meta
    if meta is None or meta.err is not None:
        return None  # Skip failed transactions

    dex_program, dex_name = dex_info
    signer = get_signer(tx)
    sig = str(tx.transaction.transaction.signatures[0])
    token_changes = extract_token_balance_changes(tx)
    pool_key = get_pool_key(tx, dex_program)
    is_buy, target_mint = classify_swap_direction(token_changes, signer)

    return SwapInfo(
        tx_index=tx_index,
        signature=sig,
        signer=signer,
        dex_program=dex_program,
        dex_name=dex_name,
        pool_accounts=[pool_key],
        token_changes=token_changes,
        is_buy=is_buy,
        target_mint=target_mint,
    )


def find_sandwiches(swaps: list) -> list:
    """
    Given a list of SwapInfo ordered by position in the block,
    find sandwich attack patterns.

    Pattern:
    1. Attacker buys token T on pool P (front-run)
    2. Victim swaps on pool P (pushes price)
    3. Attacker sells token T on pool P (back-run)

    Constraints:
    - front_run.signer == back_run.signer (same attacker)
    - front_run.signer != victim.signer (different from victim)
    - All three interact with overlapping pool accounts
    - front_run is a buy, back_run is a sell of the same token
    - front_run.tx_index < victim.tx_index < back_run.tx_index
    """
    sandwiches = []

    # Group swaps by signer for efficiency
    signer_swaps = {}
    for swap in swaps:
        if swap.signer not in signer_swaps:
            signer_swaps[swap.signer] = []
        signer_swaps[swap.signer].append(swap)

    # For each signer, look for buy-then-sell pairs
    for signer, signer_swap_list in signer_swaps.items():
        buys = [s for s in signer_swap_list if s.is_buy is True]
        sells = [s for s in signer_swap_list if s.is_buy is False]

        for buy in buys:
            for sell in sells:
                # Must be same target token
                if buy.target_mint != sell.target_mint:
                    continue

                # Buy must come before sell
                if buy.tx_index >= sell.tx_index:
                    continue

                # Must have overlapping pool accounts
                buy_pools = set(buy.pool_accounts)
                sell_pools = set(sell.pool_accounts)
                if not buy_pools & sell_pools:
                    continue

                # Look for victim transactions between the buy and sell
                for victim_swap in swaps:
                    if victim_swap.signer == signer:
                        continue  # Victim must be different signer

                    if not (buy.tx_index < victim_swap.tx_index < sell.tx_index):
                        continue  # Must be between buy and sell

                    # Victim should interact with same pool
                    victim_pools = set(victim_swap.pool_accounts)
                    if not buy_pools & victim_pools:
                        continue

                    # Check if victim trades the same token
                    if victim_swap.target_mint != buy.target_mint:
                        continue

                    # Calculate rough profit estimate
                    profit = 0.0
                    for change in sell.token_changes:
                        if change.owner == signer and change.mint in {
                            WSOL_MINT, USDC_MINT, USDT_MINT
                        }:
                            profit += change.change  # positive = profit
                    for change in buy.token_changes:
                        if change.owner == signer and change.mint in {
                            WSOL_MINT, USDC_MINT, USDT_MINT
                        }:
                            profit += change.change  # negative = cost

                    sandwich = SandwichAttack(
                        front_run=buy,
                        victim=victim_swap,
                        back_run=sell,
                        attacker=signer,
                        profit_estimate=profit,
                    )
                    sandwiches.append(sandwich)

    return sandwiches


def analyze_block(slot: int) -> list:
    """Main function to analyze a Solana block for sandwich attacks."""
    print(f"Analyzing slot {slot} for sandwich attacks...")

    client = Client(SOLANA_RPC)

    # Fetch block
    print("Fetching block data...")
    block = get_block_transactions(client, slot)

    if block is None:
        print(f"Could not fetch block at slot {slot}")
        return []

    transactions = block.transactions
    print(f"Block contains {len(transactions)} transactions")

    # Extract swap information from all transactions
    print("Extracting swap information...")
    swaps = []
    for i, tx in enumerate(transactions):
        swap = extract_swap_info(tx, i)
        if swap is not None:
            swaps.append(swap)

    print(f"Found {len(swaps)} DEX swap transactions")

    if len(swaps) < 3:
        print("Not enough swaps to form a sandwich")
        return []

    # Find sandwich patterns
    print("Searching for sandwich patterns...")
    sandwiches = find_sandwiches(swaps)

    return sandwiches


def print_results(sandwiches: list, slot: int):
    """Pretty-print the sandwich detection results."""
    print("\n" + "=" * 80)
    print(f"SANDWICH ATTACK DETECTION RESULTS - Slot {slot}")
    print("=" * 80)

    if not sandwiches:
        print("\nNo sandwich attacks detected in this block.")
        return

    print(f"\nDetected {len(sandwiches)} potential sandwich attack(s):\n")

    for i, s in enumerate(sandwiches, 1):
        print(f"--- Sandwich #{i} ---")
        print(f"  Attacker:   {s.attacker}")
        print(f"  DEX:        {s.front_run.dex_name}")
        print(f"  Token:      {s.front_run.target_mint}")
        print(f"  Front-run:  tx #{s.front_run.tx_index} - {s.front_run.signature[:20]}...")
        print(f"  Victim:     tx #{s.victim.tx_index} - {s.victim.signature[:20]}...")
        print(f"    Victim signer: {s.victim.signer}")
        print(f"  Back-run:   tx #{s.back_run.tx_index} - {s.back_run.signature[:20]}...")
        if s.profit_estimate != 0:
            print(f"  Est. profit: {s.profit_estimate:.6f} (in SOL/USDC units)")
        print()


def main():
    if len(sys.argv) < 2:
        print("Usage: python detect_sandwich.py <slot_number>")
        print("Example: python detect_sandwich.py 269270524")
        sys.exit(1)

    slot = int(sys.argv[1])
    sandwiches = analyze_block(slot)
    print_results(sandwiches, slot)

    # Exit code indicates whether sandwiches were found
    sys.exit(0 if not sandwiches else 1)


if __name__ == "__main__":
    main()
```

### How the Detector Works

1. **Fetch block data**: Uses `getBlock` RPC with `jsonParsed` encoding to get all transactions with decoded instruction data and token balance changes.

2. **Identify DEX swaps**: Filters transactions that interact with known DEX program IDs (Raydium, Orca, Jupiter, Meteora, OpenBook, etc.).

3. **Extract swap metadata**: For each DEX transaction, extracts:
   - The signer (fee payer)
   - Token balance changes (from `pre_token_balances` / `post_token_balances`)
   - Pool identity (from account keys in the instruction)
   - Swap direction (buy vs. sell based on token balance changes)

4. **Pattern matching**: Searches for triplets where:
   - Same signer has a **buy** (front-run) before and a **sell** (back-run) after a victim transaction
   - All three transactions interact with the **same DEX pool**
   - All three trade the **same token**
   - The victim transaction is from a **different signer**

5. **Profit estimation**: Calculates approximate profit by comparing the attacker's SOL/USDC balance changes across the front-run and back-run transactions.

### Limitations and Improvements

- **Multi-hop swaps**: Some sandwiches use multi-hop routes (e.g., via Jupiter). The detector should also check inner instructions for DEX interactions.
- **Multiple victims**: A single sandwich can wrap multiple victim transactions, not just one.
- **Cross-DEX sandwiches**: The front-run and back-run might use different DEXes or different pools. More sophisticated detectors use token flow analysis rather than pool matching.
- **False positives**: Regular arbitrage or rebalancing trades may look like sandwiches. Cross-referencing with the validator's leader schedule helps confirm intent.
- **Account obfuscation**: Sophisticated attackers use different accounts for front-run and back-run. Linking these requires analyzing fund flows or using known attacker address databases.

### Running Against Known Sandwich Blocks

To test the detector, you can use slots from known sandwiching validators identified during the June 2024 enforcement period. Look for blocks produced by validators that were subsequently removed from the Foundation's delegation program and analyze their leader slots for sandwich patterns.

```bash
# Install dependencies
pip install solana solders requests

# Run against a specific slot
python detect_sandwich.py 269270524

# To scan a range of slots for a specific validator's leader slots:
# 1. Get the validator's leader schedule
# 2. For each of their slots, run the detector
# 3. Aggregate results
```
