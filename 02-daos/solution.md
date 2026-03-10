# Challenge 02 - DAOs (Tier 1) Solution

## a) Identify DAOs vulnerable to economic governance attacks

### What makes a DAO vulnerable?

A DAO is vulnerable to an economic governance attack when:
1. **Cost of acquiring quorum** < **Value extractable from treasury**
2. The governance token has sufficient DEX liquidity to acquire a large position
3. The governance system allows direct treasury transfers or critical parameter changes
4. There is no timelock or it can be bypassed
5. Low voter participation means fewer tokens needed to reach quorum

### Vulnerable DAOs identified:

#### 1. **Wildcat Finance (or similar small-cap DAOs with large treasuries)**

Many smaller DeFi protocols have governance tokens with low market caps relative to their treasuries. An attacker could:
- Acquire enough tokens on DEXes to meet quorum
- Propose a treasury drain
- Vote the proposal through

#### 2. **Build Finance DAO** (attacked in Feb 2022)
- This DAO was actually attacked: an attacker acquired governance tokens, passed a proposal to mint new tokens, and effectively took over the protocol
- Treasury at risk: ~$470K
- Cost: Minimal (low token liquidity meant cheap acquisition)

#### 3. **Beanstalk** (attacked April 2022 via flash loan governance)
- Attacker used a flash loan to acquire enough governance tokens (STALK) to pass a proposal in a single transaction
- ~$182M stolen
- Cost: Flash loan fees only (~$0)

#### 4. **Sushi (SushiSwap)**
- Low governance participation (~5-10% of tokens vote)
- Large treasury (tens of millions in various tokens)
- SUSHI token has reasonable DEX liquidity
- Quorum requirements may be achievable at ~$10-50M cost
- Treasury at risk: $30-100M+

#### 5. **Tornado Cash Governance** (attacked May 2023)
- Attacker deployed a malicious proposal contract that granted them 1.2M TORN tokens
- Used these to control governance and drain the treasury
- Cost: Gas + deployment costs

### General Attack Vector:

```
Attack Cost = tokens_for_quorum * token_price * (1 + slippage)
Potential Profit = treasury_value - attack_cost
Vulnerable if: Potential Profit > 0
```

### Estimating Cost for SushiSwap:

Using Dune data:
- SUSHI total supply: 250M
- Average governance participation: ~5-10% of circulating supply
- To pass a proposal: need > 50% of participating votes
- Tokens needed: ~5-12.5M SUSHI
- SUSHI price (mid-2024): ~$0.80-1.50
- Cost estimate: $4M - $18.75M
- Slippage on DEX acquisition would add 20-50%+
- Effective cost: $5M - $28M
- Treasury value: $30-50M in various tokens
- **Margin: Potentially profitable**

---

## b) Estimate potential benefit for the attacker

### For each vulnerable DAO:

#### Sushi:
- **Treasury at risk:** ~$50M+ (multi-sig treasury containing ETH, stablecoins, SUSHI, LP positions)
- **Attack type:** Propose transfer of treasury to attacker address
- **Timelock protection:** Yes, Sushi has a 2-day timelock, which gives defenders time to react
- **Net risk:** Reduced by timelock — but if community is inactive, timelock alone doesn't prevent execution

#### Build Finance (historical):
- **Treasury stolen:** ~$470K
- **Attack cost:** < $10K (low liquidity token)
- **Profit:** ~$460K

#### Beanstalk (historical):
- **Treasury stolen:** ~$182M (via flash loan governance attack)
- **Attack cost:** Flash loan fees (~$0)
- **Profit:** ~$182M (though most was later recovered via an exploit bounty negotiation)

#### General small-cap DAOs:
Many DAOs have:
- Treasury: $1M - $50M
- Token FDV: $5M - $100M
- Quorum: 1-10% of supply
- Required capital: $50K - $5M
- **Profitable attack when treasury > 2x attack cost**

### Key risk factors to evaluate:

| Factor | High Risk | Low Risk |
|--------|-----------|----------|
| Voter participation | < 5% | > 30% |
| Token liquidity | High DEX volume | Low/OTC only |
| Timelock | None or < 24h | > 48h |
| Treasury/MCap ratio | > 0.5 | < 0.1 |
| Proposal threshold | Low | High |
| Multi-sig protection | No | Yes |

### Live Governance Data (Dune `dao.proposals` — Jan–Sept 2024):

| DAO | Proposals | Avg Voters | Avg Participation | Executed |
|-----|-----------|-----------|-------------------|----------|
| **Compound** | 112 | ~27 | **5.5%** | 92 |
| **ENS** | 8 | ~150 | **14.3%** | 8 |
| **Uniswap** | 14 | ~785 | **4.7%** | 14 |

Key takeaway: Even major DAOs like **Compound** have only ~27 average voters per proposal and ~5.5% participation rate. This means a determined attacker needs to outweigh only the active voters, not the full token supply.

### Dune SQL Query for DAO Vulnerability Analysis:

```sql
-- Find DAOs with low voter participation and their proposal stats
SELECT
    dao_name,
    COUNT(*) as total_proposals,
    AVG(votes_total) as avg_votes_total,
    AVG(number_of_voters) as avg_num_voters,
    AVG(participation) as avg_participation,
    SUM(CASE WHEN status = 'Executed' THEN 1 ELSE 0 END) as executed_proposals,
    MAX(created_at) as last_proposal
FROM dao.proposals
WHERE created_at >= TIMESTAMP '2024-01-01'
AND created_at <= TIMESTAMP '2024-09-04'
GROUP BY dao_name
HAVING COUNT(*) >= 2
ORDER BY avg_participation ASC
LIMIT 30
```

### Conclusion

The most vulnerable DAOs share common traits: low participation, high treasury-to-market-cap ratios, liquid governance tokens, and weak or absent timelocks. Historical examples (Build Finance, Beanstalk, Tornado Cash) demonstrate that governance attacks are practical and profitable. The shift towards veToken models (vote-escrowed locking), optimistic governance, and security councils has been partly driven by these risks.
