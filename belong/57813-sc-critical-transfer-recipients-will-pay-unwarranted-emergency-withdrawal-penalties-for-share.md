# 57813 sc critical transfer recipients will pay unwarranted emergency withdrawal penalties for share positions they legitimately own

**Submitted on Oct 29th 2025 at 01:41:03 UTC by @Lavender88 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57813
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:** Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

{% hint style="danger" %}
Missing stake-tracking updates on ERC20 transfers can force transfer recipients to use emergency withdrawal (10% penalty) even after waiting the full minStakePeriod.
{% endhint %}

## Description

### Brief / Intro

The missing stake tracking updates on ERC20 transfers will cause a 10% penalty loss for share recipients: they cannot use normal withdrawal even after waiting the full minStakePeriod because the stakes\[] array remains empty for transferred shares.

### Vulnerability Details

In `Staking.sol`, the ERC20 `transfer()` function inherited from Solady's base implementation does not update the `stakes[]` mapping. When shares are transferred:

* `balanceOf[]` correctly updates (sender decreases, recipient increases)
* `stakes[]` does NOT update (recipient remains with empty array)

This desynchronization causes `_consumeUnlockedSharesOrRevert()` to fail for transfer recipients even after the lock period expires, forcing them into the emergency withdrawal path with its 10% penalty.

## Preconditions / Reproduction Steps

{% stepper %}
{% step %}

### 1. Create a locked stake for Alice

Alice calls `deposit()` to create a stake entry in `stakes[A]` with locked shares.
{% endstep %}

{% step %}

### 2. Transfer shares to recipient

Alice calls `transfer()` to send shares to Bob, who has an empty `stakes[B]` array.
{% endstep %}

{% step %}

### 3. Ensure non-zero lock period

`minStakePeriod` is set to a non-zero value (e.g., default 1 day).
{% endstep %}
{% endstepper %}

## Attack Path (example scenario)

Scenario: Innocent Transfer Recipient (Bob) Gets Griefed

* Alice deposits 1000 LONG tokens via `deposit()`, receiving 1000 shares\
  `stakes[alice] = [{shares: 1000, timestamp: T}]`\
  `balanceOf[alice] = 1000`
* Alice transfers 1000 shares to Bob via standard ERC20 `transfer()`\
  `balanceOf[alice] = 0`\
  `balanceOf[bob] = 1000`\
  `stakes[alice] = unchanged [{shares: 1000, timestamp: T}]`\
  `stakes[bob] = []` (empty)
* Bob waits full lock period (1 day passes) and calls `redeem(1000, bob, bob)`\
  Execution reaches `_consumeUnlockedSharesOrRevert(bob, 1000)` which loops through `stakes[bob]` (empty) => remaining = 1000 => REVERTS with `MinStakePeriodNotMet()`.
* Bob is forced to call `emergencyRedeem(1000, bob, bob)` which uses `_removeAnySharesFor()` and does not check `stakes[]` properly. Bob pays 10% penalty: 100 LONG sent to treasury, receives 900 LONG.

Result: Bob loses 100 LONG (10% of his position) despite legitimately holding shares for the required duration.

## Impact Details

Transfer recipients suffer an unwarranted 10% loss when attempting to withdraw shares they legitimately own. Examples of affected recipients:

* DEX purchasers
* Gift/transfer recipients
* Liquidity provision reward recipients

This is griefing: damage to users without profit motive for the attacker. Treasury receives unintended penalty funds.

## Mitigation

Two suggested mitigations (as proposed by the reporter):

* Option 1: Update `stakes[]` on ERC20 transfers to keep stake metadata in sync with `balanceOf[]`.
* Option 2: Disable transfers entirely (preventing transfer-based desynchronization).

## Proof of Concept

```solidity
function testTransferRecipientGriefed() public {
    // Alice stakes 1000 LONG (locked for 1 day)
    vm.prank(alice);
    uint256 shares = staking.deposit(1000e18, alice);
    
    // Alice transfers shares to Bob
    vm.prank(alice);
    staking.transfer(bob, shares);
    
    // Bob waits full lock period (1 day + buffer)
    vm.warp(block.timestamp + 1 days + 1);
    
    // Bob attempts normal withdrawal - REVERTS
    vm.prank(bob);
    vm.expectRevert(Staking.MinStakePeriodNotMet.selector);
    staking.redeem(shares, bob, bob);
    
    // Bob forced to use emergency withdrawal
    uint256 bobBalanceBefore = long.balanceOf(bob);
    vm.prank(bob);
    staking.emergencyRedeem(shares, bob, bob);
    
    uint256 bobReceived = long.balanceOf(bob) - bobBalanceBefore;
    uint256 penalty = long.balanceOf(treasury);
    
    // Verify Bob paid penalty despite waiting full period
    assertEq(penalty, 100e18, "Bob paid 10% penalty");
    assertEq(bobReceived, 900e18, "Bob lost 100 LONG");
}
```

## References

* Target contract: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
