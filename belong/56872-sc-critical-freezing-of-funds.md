# 56872 sc critical freezing of funds&#x20;

**Submitted on Oct 21st 2025 at 12:37:02 UTC by @shadowHunter for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56872
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Permanent freezing of funds

## Description

### Brief/Intro

The `Staking` contract mints shares to represent users’ staked positions. These shares are freely transferable ERC20 tokens because of ERC4626 inheritance.

The vault tracks staked positions in the `stakes[address]` array to enforce:

* Minimum stake periods (`minStakePeriod`)
* Reward distribution (proportional to shares)
* Emergency withdrawal penalties

Transferring shares to another address does not update the internal `stakes[]` array, resulting in a permanent desync between the internal staking record and the transferable ERC20 shares.

{% stepper %}
{% step %}

### Vulnerability Details — sequence of actions

* Assume Alice and Bob are two users.
* Alice deposits 100 `LONG` tokens → receives 100 `sLONG` shares.
* Alice transfers 100 `sLONG` shares to Bob.
* Bob now holds the shares but has no corresponding entries in `stakes[Bob]`.
* Bob attempts a normal withdraw → fails due to missing unlocked stake.
* Bob can only use `emergencyWithdraw` → receives 90% of assets, losing 10% as a penalty.
  {% endstep %}

{% step %}

### Impact Details

* The new holder of transferred shares cannot withdraw normally — the vault sees no unlocked shares.
* Emergency withdrawal is the only recovery path, but it always imposes the configured penalty (10% by default).
* Users effectively lose access to their funds or pay a financial cost simply due to a transfer.

This can lead to permanent freezing of funds.
{% endstep %}
{% endstepper %}

## Recommendation

Prevent permanent freezing by making shares non-transferable. For example:

{% hint style="info" %}
Override ERC20 `transfer` and `transferFrom` to revert so shares cannot be moved without updating the internal bookkeeping.
{% endhint %}

```solidity
function transfer(address, uint256) public pure override returns (bool) {
    revert("NON_TRANSFERABLE");
}

function transferFrom(address, address, uint256) public pure override returns (bool) {
    revert("NON_TRANSFERABLE");
}
```

## Proof of Concept

<details>

<summary>Click to expand the full PoC test and observed output</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.27;

import "forge-std/Test.sol";
import {Staking} from "../contracts/v2/periphery/Staking.sol";
import {LONG} from "../contracts/v2/tokens/LONG.sol";
import "openzeppelin-contracts/contracts/proxy/Clones.sol";

contract StakingTransferTest is Test {
    Staking stakingImplementation;
    Staking vault;

    LONG token;

    address owner = address(this);
    address treasury = address(0xDEAD);
    address alice = address(0xA11CE);
	address bob = address(0xB0B);

    function setUp() public {
        // --- Deploy and initialize LONG ---
		LONG longImpl = new LONG();
		token = LONG(Clones.clone(address(longImpl)));
        //token = new LONG();
        token.initialize(alice, owner, owner); // initial supply to alice, owner/admin roles set

        // --- Deploy Staking implementation ---
        stakingImplementation = new Staking();

        // --- Deploy a clone vault ---
        vault = Staking(Clones.clone(address(stakingImplementation)));

        // --- Initialize vault ---
        vault.initialize(owner, treasury, address(token));

        // --- Alice approves the vault ---
        vm.startPrank(alice);
        token.approve(address(vault), type(uint256).max);
        vm.stopPrank();
    }

   /// @notice Helper to get stake array length for a user
    function getStakeLength(address user) internal view returns (uint256 len) {
        try vault.stakes(user, 0) returns (uint256, uint256) {
            uint256 i = 0;
            while (true) {
                try vault.stakes(user, i) returns (uint256, uint256) {
                    unchecked { ++i; }
                } catch {
                    break;
                }
            }
            len = i;
        } catch {
            len = 0;
        }
    }

    function testTransferBreaksStakeBookkeeping() public {
        // Alice deposits
        vm.startPrank(alice);
        vault.deposit(100 ether, alice);
        vm.stopPrank();

        // Fast forward beyond minStakePeriod
        vm.warp(block.timestamp + 2 days);

        uint256 aliceShares = vault.balanceOf(alice);
        assertGt(aliceShares, 0, "alice should have shares");

        // Alice transfers all shares to Bob
        vm.prank(alice);
        vault.transfer(bob, aliceShares);

        // Bob attempts to redeem shares (should fail: no unlocked stakes)
        vm.prank(bob);
        vm.expectRevert(); // MinStakePeriodNotMet
        vault.redeem(aliceShares, bob, bob);

    }
}
```

Observed output: As expected Redeem reverts (vm.expectRevert())

```
[PASS] testTransferBreaksStakeBookkeeping() (gas: 191754)
```

</details>
