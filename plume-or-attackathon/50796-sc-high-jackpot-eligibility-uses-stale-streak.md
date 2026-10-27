# 50796 sc high jackpot eligibility uses stale streak

Submitted on Jul 28th 2025 at 16:24:35 UTC by @BeastBoy for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network)

* Report ID: #50796
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

In `handleRandomness` the code begins by computing:

```solidity
currentSpinStreak = _computeStreak(user, block.timestamp, true)
```

but then, inside the “Jackpot” branch, evaluates:

```solidity
else if (userDataStorage.streakCount < (currentWeek + 2)) {
    userDataStorage.nothingCounts += 1;
    rewardCategory = "Nothing";
}
```

Here `userDataStorage.streakCount` still holds yesterday’s value, so even if `currentSpinStreak` meets the required threshold (`currentWeek + 2`), the check fails. Only after this logic does the code execute:

```solidity
userDataStorage.streakCount = currentSpinStreak;
```

meaning the first day a user actually reaches the needed streak they are incorrectly treated as ineligible.

## Impact

{% hint style="warning" %}
Users must spin an extra day before becoming eligible for the jackpot, contrary to intended behavior and degrading user experience.
{% endhint %}

## Recommendation

{% hint style="info" %}
Assign `userDataStorage.streakCount = currentSpinStreak` before the jackpot check or change the condition to compare `currentSpinStreak` directly against the required threshold so that today’s spin counts immediately.
{% endhint %}

## Proof of Concept

{% code title="PoC: testStreakJackpotBug.sol" %}

```solidity
function testStreakJackpotBug() public {
    // Set up Week 0 (requires 2-day streak)
    vm.warp(spin.getCampaignStartDate());
    
    // Day 1: User spins for first time
    vm.deal(USER, INITIAL_SPIN_PRICE);
    uint256 nonce1 = performPaidSpin(USER);
    
    // Mock jackpot-winning randomness but expect denial due to insufficient old streak
    uint256[] memory rng1 = new uint256[](1);
    rng1[0] = 0; // Force jackpot
    
    vm.prank(SUPRA_ORACLE);
    vm.recordLogs();
    spin.handleRandomness(nonce1, rng1);
    
    Vm.Log[] memory logs1 = vm.getRecordedLogs();
    (string memory result1,) = abi.decode(logs1[logs1.length - 1].data, (string, uint256));
    assertEq(result1, "Nothing", "Day 1: Should be denied (old streak = 0)");
    
    // Verify current streak is now 1
    assertEq(spin.currentStreak(USER), 1, "Current streak should be 1 after day 1");
    
    // Day 2: User spins again (should have 2-day streak but gets denied)
    vm.warp(block.timestamp + 1 days);
    vm.deal(USER, INITIAL_SPIN_PRICE);
    uint256 nonce2 = performPaidSpin(USER);
    
    // Before the spin, verify the user SHOULD be eligible
    uint256 wouldBeStreak = spin.currentStreak(USER); // This will be 1 (old)
    // But _computeStreak(USER, block.timestamp, true) would return 2
    
    uint256[] memory rng2 = new uint256[](1);
    rng2[0] = 0; // Force jackpot
    
    vm.prank(SUPRA_ORACLE);
    vm.recordLogs();
    spin.handleRandomness(nonce2, rng2);
    
    Vm.Log[] memory logs2 = vm.getRecordedLogs();
    (string memory result2,) = abi.decode(logs2[logs2.length - 1].data, (string, uint256));
    
    // THE BUG: User should get jackpot (actual streak = 2) but gets denied (old streak = 1)
    assertEq(result2, "Nothing", "Day 2: BUG - User denied despite having 2-day streak");
    
    // Day 3: Now user finally gets jackpot
    vm.warp(block.timestamp + 1 days);
    vm.deal(USER, INITIAL_SPIN_PRICE);
    uint256 nonce3 = performPaidSpin(USER);
    
    uint256[] memory rng3 = new uint256[](1);
    rng3[0] = 0; // Force jackpot
    
    vm.prank(SUPRA_ORACLE);
    vm.recordLogs();
    spin.handleRandomness(nonce3, rng3);
    
    Vm.Log[] memory logs3 = vm.getRecordedLogs();
    (string memory result3,) = abi.decode(logs3[logs3.length - 1].data, (string, uint256));
    
    assertEq(result3, "Jackpot", "Day 3: Finally eligible (one day late)");
    
    console2.log("BUG CONFIRMED: User eligible on day 2 but denied until day 3");
}
```

{% endcode %}
