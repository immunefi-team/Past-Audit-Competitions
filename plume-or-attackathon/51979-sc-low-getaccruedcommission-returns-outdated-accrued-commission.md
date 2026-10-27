# 51979 sc low getaccruedcommission returns outdated accrued commission

* **Submitted on:** Aug 6th 2025 at 23:58:13 UTC by @holydevoti0n for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #51979
* **Report Type:** Smart Contract
* **Severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>

## Description

### Vulnerability Details

To get the total amount of commission accrued per token/validator, the `getAccruedCommission` function from `ValidatorFacet` must be called. The issue is that the accrued commission value returned is outdated because the function returns the stored `validatorAccruedCommission` value without ensuring it is updated up to the current timestamp (updates are only triggered by some other functions like stake/unstake, etc).

Relevant code:

```solidity
/**
 * @notice Get the amount of commission accrued for a specific token by a validator but not yet claimed.
 * @return The total accrued commission for the specified token.
 */
function getAccruedCommission(uint16 validatorId, address token) public view returns (uint256) {
    PlumeStakingStorage.Layout storage $s = PlumeStakingStorage.layout();
    if (!$s.validatorExists[validatorId]) {
        revert ValidatorDoesNotExist(validatorId);
    }
    if (!$s.isRewardToken[token]) {
        revert TokenDoesNotExist(token);
    }

    return $s.validatorAccruedCommission[validatorId][token];
}
```

Example scenario: if a validator accrues 10 tokens in commission over 1 hour, `getAccruedCommission` will only return that 10 tokens if `validatorAccruedCommission[validatorId][token]` has been updated since the accrual. Since `getAccruedCommission` does not itself enforce an update, it can return 0 (or another stale value) instead of the true current accrued amount.

### Impact Details

`getAccruedCommission` can return an incorrect (stale) amount because it does not take into account commission accrued up to the current timestamp.

## Recommendation

{% hint style="info" %}
Call `updateRewardPerTokenForValidator` (or the appropriate updater) before returning the accrued commission so the returned value is up to date.
{% endhint %}

Suggested change:

```diff
function getAccruedCommission(uint16 validatorId, address token) public view returns (uint256) {
        PlumeStakingStorage.Layout storage $s = PlumeStakingStorage.layout();
        if (!$s.validatorExists[validatorId]) {
            revert ValidatorDoesNotExist(validatorId);
        }
        if (!$s.isRewardToken[token]) {
            revert TokenDoesNotExist(token);
        }

+       PlumeRewardLogic.updateRewardPerTokenForValidator($s, token, validatorId);

        return $s.validatorAccruedCommission[validatorId][token];
    }
```

## Proof of Concept

Add the following test in `PlumeStakingDiamond.t.sol`:

```solidity
function testAccruedCommission_returnIncorrectValue() public {
    ManagementFacet managementFacet = ManagementFacet(address(diamondProxy));
    RewardsFacet rewardsFacet = RewardsFacet(address(diamondProxy));
    ValidatorFacet validatorFacet = ValidatorFacet(address(diamondProxy));
    StakingFacet stakingFacet = StakingFacet(address(diamondProxy));

    // 1. SETUP
    // Create a reward token and fund the treasury.
    MockPUSD rewardToken = new MockPUSD();
    address token = address(rewardToken);
    vm.startPrank(admin);
    rewardsFacet.addRewardToken(token, 1e18, 1e24); // 0.01 rewards per second
    treasury.addRewardToken(token);
    rewardToken.transfer(address(treasury), 10e22);
    vm.stopPrank();

     vm.startPrank(validatorAdmin);
        uint256 newCommission = 50e16;
        ValidatorFacet(address(diamondProxy)).setValidatorCommission(
            DEFAULT_VALIDATOR_ID,
            newCommission
        );
    vm.stopPrank();

    // User stakes with the default validator.
    uint16 validatorId = DEFAULT_VALIDATOR_ID;
    vm.prank(user1);
    stakingFacet.stake{value: 100e18}(validatorId);

    // 2. ACCRUE REWARDS
    // Let 1 day pass to accrue rewards while the validator is active.
    vm.warp(block.timestamp + 1 days);


    // fetch validatorId accrued commission
    uint256 accruedCommission = validatorFacet.getAccruedCommission(validatorId, token);
    assertEq(accruedCommission, 0, "Should be zero");
}
```

Run:

```
forge test --mt testAccruedCommission_returnIncorrectValue --via-ir
```

Output:

```
Ran 1 test for test/PlumeStakingDiamond.t.sol:PlumeStakingDiamondTest
[PASS] testAccruedCommission_returnIncorrectValue() (gas: 1483911)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.00ms (281.50µs CPU time)
```

***
