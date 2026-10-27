# 31562 - \[SC - Medium] Every consecutive epoch will have same number o...

Submitted on May 21st 2024 at 11:06:37 UTC by @SAAJ for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31562

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Minter.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`Minter` contract have function `updatePeriod` where it has a condition where it reset `stepdown` to update `rewards` for next epoch.

## Vulnerability Details

The description for condition to reset `stepdown` in `updatePeriod` function clearly mentions only when the `rewards` level reached the `TAIL_EMISSIONS_RATE`.

```
// Once we reach the emissions tail stepdown is 0

if (rewards <= TAIL_EMISSIONS_RATE) {
                stepdown = 0;
            }
```

However, the condition logic is flawed as it is designed to reset `stepdown` even when `rewards` level is lower than the `TAIL_EMISSIONS_RATE` for the current epoch. This will lead to resetting of `stepdown` in every coming epoch that will have `less or equal` rewards with comparison to `TAIL_EMISSIONS_RATE`.

```
// Set rewards for next epoch
            rewards -= stepdown;
```

## Impact Details

When the `updatePeriod` is called at first epoch and if `rewards` level is lower /equal to `TAIL_EMISSIONS_RATE` it will reset `stepdown`.

This will impact 3rd epoch, as when `updatePeriod` is called in 2nd epoch it will have `stepdown` value equal to `0`.

```
            rewards -= stepdown;
```

`rewards` value for 3rd epoch will be same as the previous 2nd epoch value, as the call in 2nd epoch will have no impact on value of `rewards` based on condition of being subtracted with zero.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Minter.sol#L160>

## Recommendation

Recommendation is made to change the logic from `<=` to `>=` to clearly and truely meet the condition of resetting `stepdown`, only when `rewards` amount meets or surpassed the value of `TAIL_EMISSIONS_RATE`.

```diff
    /// @inheritdoc IMinter
    function updatePeriod() external returns (uint256) {
        require(msg.sender == address(voter), "not voter");

        uint256 period = activePeriod;

        if (block.timestamp >= period + DURATION && initializer == address(0)) {
            // Only trigger if new epoch
            period = (block.timestamp / DURATION) * DURATION;
            activePeriod = period;
            epochEmissions = epochEmission();

            uint256 veAlcxEmissions = calculateEmissions(epochEmissions, veAlcxEmissionsRate);
            uint256 timeEmissions = calculateEmissions(epochEmissions, timeEmissionsRate);
            uint256 treasuryEmissions = calculateEmissions(epochEmissions, treasuryEmissionsRate);
            uint256 gaugeEmissions = epochEmissions.sub(veAlcxEmissions).sub(timeEmissions).sub(treasuryEmissions);
            uint256 balanceOf = alcx.balanceOf(address(this));

            if (balanceOf < epochEmissions) alcx.mint(address(this), epochEmissions - balanceOf);

            // Set rewards for next epoch
            rewards -= stepdown;

            // Adjust updated emissions total
            supply += rewards;

            // Once we reach the emissions tail stepdown is 0
-           if (rewards <= TAIL_EMISSIONS_RATE) {
+           if (rewards >= TAIL_EMISSIONS_RATE) {
                stepdown = 0;
            }

            // If there are no votes, send emissions to veALCX holders
            if (voter.totalWeight() > 0) {
                alcx.approve(address(voter), gaugeEmissions);
                voter.notifyRewardAmount(gaugeEmissions);
            } else {
                veAlcxEmissions += gaugeEmissions;
            }

            // Logic to distrubte minted tokens
            IERC20(address(alcx)).safeTransfer(address(rewardsDistributor), veAlcxEmissions);
            rewardsDistributor.checkpointToken(); // Checkpoint token balance that was just minted in rewards distributor
            rewardsDistributor.checkpointTotalSupply(); // Checkpoint supply

            IERC20(address(alcx)).safeTransfer(address(timeGauge), timeEmissions);
            timeGauge.notifyRewardAmount(timeEmissions);

            IERC20(address(alcx)).safeTransfer(treasury, treasuryEmissions);

            revenueHandler.checkpoint();

            emit Mint(msg.sender, epochEmissions, supply);
        }
        return period;
    }
}
```

## Proof of Concept

This test demonstrate the no change in impact in value of `rewards` for next epoch if its value is less or equal to `TAIL_EMISSIONS_RATE` in current one. For this test values for variables are assumed to have clear idea on the outcome generated when the `updatePeriod` is called in 2nd epoch.

```
    uint256 TAIL_EMISSIONS_RATE = 2194; // ALCX tail emissions rate sans 18

    uint256 supply = 100; // assumed value of stepdown sans 18 decimals
    uint256 stepdown = 100; // assumed value of stepdown sans 18 decimals
    uint256 rewards = 2190; // assumed value of rewards sans 18 decimals

    // forge t --mt test_newEpoch -vv
    function test_newEpoch() external {
        console.log("Reward at 1st Epoch:", rewards);

        test_UpdatePeriod(); // call made for 1st epoch
        console.log("Stepdown:", stepdown);
console.log("Reward set for 2nd Epoch:", rewards);
        assertEq(rewards, 2090); // asserting reward set after function called

        uint256 next_EPOCH = 2 weeks; // assume update is called on 2nd week
        vm.warp(next_EPOCH); // making call to the function at new epoch i.e. 2nd week

        test_UpdatePeriod(); // call made for 2nd epoch
        console.log("Reward after UpdatePeriod() called on 2nd Epoch:", rewards); // Reward set for 3rd Epoch
        assertEq(rewards, 2090); // asserting reward set at 2nd Epoch

    }
```

The test passed when `updatePeriod` is called during 1st and 2nd epoch with same value of `rewards` generated each time.

```
[PASS] test_newEpoch() (gas: 23198)
Logs:
  Reward at 1st Epoch: 2190
  Stepdown: 0
  Reward set for 2nd Epoch: 2090
  Reward after UpdatePeriod() called on 2nd Epoch: 2090

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.75ms (228.17µs CPU time)

Ran 1 test suite in 86.64ms (1.75ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The value shown for `rewards` is same for 3rd epoch as it is subtracted to zero in the 2nd epoch one by meeting the condition of <= to `TAIL_EMISSIONS_RATE`.
