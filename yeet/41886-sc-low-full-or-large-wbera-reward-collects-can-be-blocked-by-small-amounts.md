# #41886 \[SC-Low] Full or Large WBERA reward collects can be blocked by small amounts

**Submitted on Mar 19th 2025 at 06:43:21 UTC by @merlinboii for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41886
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

**The `depositWBERA` function in `StakeV2` contract is vulnerable to front-running attacks that can prevent the protocol from collecting full or large amounts of surplus `WBERA`**. While this doesn't affect the main `BERA` reward distribution, it can temporarily disrupt the protocol's ability to handle surplus rewards.

## Vulnerability Details

The `depositWBERA` function in the `StakeV2` contract allows converting `WBERA` held by the contract into `BERA` for reward distribution:

```solidity
function depositReward() public payable {
    require(msg.value > 0, "Must send value");
@>  accumulatedRewards += msg.value;
    emit RewardDeposited(msg.sender, msg.value);
}
--- SNIPPED ---
function depositWBERA(uint256 amount) external {
@>  wbera.withdraw(amount);
    this.depositReward{
            value: amount
        }();
}
```

However, this function is **vulnerable to a frontrunning attack**, where an attacker can **deposit a dust amount (e.g., 1 wei) right before a legitimate large deposit transaction**. This results in only the dust amount being processed, while the full deposit attempt fails and reverts due to insufficient available `WBERA`.

This issue should not be confused with normal transaction races, where multiple legitimate transactions may compete to deposit the full amount. In those cases, the total intended amount is still successfully converted into `BERA` over multiple transactions, even if one of them fails.

Instead, this vulnerability **specifically allows an attacker to block full `WBERA` deposits** by ensuring only an insignificant amount gets collected.

## Impact Details

* **Griefing** (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* An attacker can **indefinitely frontrun** these events to disrupt protocol operations.
* The impact is more significant if the function is called together with reward distribution, especially when a manager is smart contract and its reward distribution handler does: `{depositWBERA(wbera.balanceOf(stakeV2)); executeRewardDistribution(...);}`}.

Although, the issue can be classified as `Griefing (Medium)`, its impact is more aligned with `Contract fails to deliver promised returns, but doesn't lose value (Low)`, due to the following limitations:

* Only affects surplus `WBERA` (donations are also affected, but this is not the primary reward mechanism, as rewards are mainly distributed in `BERA`)
* Main reward distribution uses `BERA` and remains unaffected (only surplus from distribution events is impacted)
* Can be worked around using partial deposits

## Recommendation

1. Allows to input `amount` as `type(uint256).max` to trigger full balance deposit:

```diff
function depositWBERA(uint256 amount) external {
+   uint256 withdrawAmount = amount;
+   if (amount == type(uint256).max) {
+       withdrawAmount = wbera.balanceOf(address(this));
+   }
-    wbera.withdraw(amount);
+    wbera.withdraw(withdrawAmount);
    this.depositReward{
-           value: amount
+           value: withdrawAmount
        }();
}
```

2. Handle `WBERA` within the `executeRewardDistribution` so this eliminates the need for separate `depositWBERA` calls

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L138-L143>

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L190>

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L182-L201>

## Proof of Concept

## Proof of Concept

### Runnable PoC

* Put the following test into `test/StakeV2.test.sol:StakeV2_ExecuteRewardsDistrubution`

```solidity
    function test_audit_handleExcessToken1DepositWBERA() public {
        address add = makeAddr("Koala");
        stakeV2.addManager(add);
        vm.deal(add, 1000 ether);
        vm.startPrank(add);
        // Add excess wbera to staking contract to simulate debt
        wbera.deposit{
                value: 100 ether
            }();
        wbera.transfer(address(stakeV2), 100 ether);
        vm.stopPrank();
        
        uint256 amountToDeposit = wbera.balanceOf(address(stakeV2));
        
        // frontrun deposit 1 wei
        address alice = makeAddr("alice");
        vm.prank(alice);
        stakeV2.depositWBERA(1);
        
        // tx got executed after
        vm.expectRevert();
        stakeV2.depositWBERA(amountToDeposit);
    }
}
```

* Run: `forge test --match-test test_audit_handleExcessToken1DepositWBERA -vvv`

### Conceptual PoC

1. Initial stats:
   * `StakeV2` holds 1e18 `WBERA` (surplus)
   * `accumulatedRewards`: 20e18 `BERA`
2. Manager initiates **full** collection: `depositWBERA(1e18)`
3. Attacker see the transaction then frontrun with 1 wei dust deposit: `depositWBERA(1)`
   * The balance of `WBERA` becomes `1e18 - 1`
   * `accumulatedRewards`: 20e18 + 1 `BERA`
4. When the manger's tx got executed, it will revert due to insufficient funds
   * actual balance: `1e18 - 1` **less than** expected withdraw: `1e18`
5. Attacker can also disrupt and prevent LARGE deposits as well, by calculating:
   * `required deposit` = `WBERA.balanceOf(stakeV2)` - `upcoming withdrawal` + 1
   * **Griefing becomes attractive as long as `upcoming withdrawal` >> `required deposit`**
