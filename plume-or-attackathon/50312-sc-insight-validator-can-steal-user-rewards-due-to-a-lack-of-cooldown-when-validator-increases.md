# 50312 sc insight validator can steal user rewards due to a lack of cooldown when validator increases commission

**Submitted on Jul 23rd 2025 at 16:09:04 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50312
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

When a validator increases their commission, the change takes effect immediately. This means that users who staked with the validator under the old commission rate are automatically subject to the new, higher rate. As a result, user rewards (for the next segment) are affected, since they are calculated as the gross rewards minus the validator's commission.

### Vulnerability Details

The staking system works based on segments. When a validator increases their commission rate, for example to the maximum allowed (`$.maxAllowedValidatorCommission`), the code updates the commission immediately:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L317-L352>

```solidity
    function setValidatorCommission(
        uint16 validatorId,
        uint256 newCommission
    ) external onlyValidatorAdmin(validatorId) {
        // Check against the system-wide maximum allowed commission.
        if (newCommission > $.maxAllowedValidatorCommission) {
            revert CommissionExceedsMaxAllowed(newCommission, $.maxAllowedValidatorCommission);
        }

        ...

        // Now update the validator's commission rate to the new rate.
@>        validator.commission = newCommission;
@>        PlumeRewardLogic.createCommissionRateCheckpoint($, validatorId, newCommission);
        ...
    }
```

User rewards are computed per segment; the commission used is the effective commission rate at the start of that segment:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L339-L352>

```solidity
    function _calculateRewardsCore(
        PlumeStakingStorage.Layout storage $,
        address user,
        uint16 validatorId,
        address token,
        uint256 userStakedAmount,
        uint256 currentCumulativeRewardPerToken
    )
        internal
        view
        returns (uint256 totalUserRewardDelta, uint256 totalCommissionAmountDelta, uint256 effectiveTimeDelta)
    {
                ...
@>                uint256 grossRewardForSegment =
                    (userStakedAmount * rewardPerTokenDeltaForUserInSegment) / PlumeStakingStorage.REWARD_PRECISION;

                // Commission rate effective at the START of this segment
@>                uint256 effectiveCommissionRate = getEffectiveCommissionRateAt($, validatorId, segmentStartTime);

                // Use ceiling division for commission charged to user to ensure rounding up
@>                uint256 commissionForThisSegment =
                    _ceilDiv(grossRewardForSegment * effectiveCommissionRate, PlumeStakingStorage.REWARD_PRECISION);

                if (grossRewardForSegment >= commissionForThisSegment) {
@>                    totalUserRewardDelta += (grossRewardForSegment - commissionForThisSegment);
                } // else, net reward is 0 for this segment for the user.
                // Commission is still generated for the validator based on gross.
                // This was previously missing, commission should always be based on gross.
                totalCommissionAmountDelta += commissionForThisSegment;
            ...
    }
```

Because commission is applied using the commission rate effective at the start of each segment, but commission rate changes are effective immediately, a validator can increase their commission right after users stake and thereby take a larger share of future rewards for those users. The user only stops losing rewards when they unstake, but rewards accumulated while the increased commission is active are lost to the validator.

The root cause is the absence of a cooldown period that would let users decide whether to accept the new commission rate or unstake.

### Example

{% stepper %}
{% step %}

### User stakes with low-commission validator

User stakes with validator A who currently has a 1% commission.
{% endstep %}

{% step %}

### Validator increases commission

Validator A has many users attracted by the 1% commission. Validator A increases commission to the maximum allowed (e.g., 50%).
{% endstep %}

{% step %}

### New segments use higher commission immediately

For the next segment, all users who staked under 1% are now subject to 50% commission because the rate change is effective immediately.
{% endstep %}

{% step %}

### Users' rewards are reduced

User rewards for that segment are reduced by the new commission rate. The longer users remain staked, the more the validator extracts.
{% endstep %}

{% step %}

### Subtle attacks possible

A validator could temporarily increase commission at specific times then revert it, profiting from short intervals while avoiding detection.
{% endstep %}
{% endstepper %}

## Impact

Theft of unclaimed yield as validators can immediately increase the commission rate to the maximum allowed, claiming up to half (or more depending on limits) of the user's yield for subsequent segments.

## Recommendation

{% hint style="warning" %}
Add a cooldown period before validator commission rate changes become effective (for example, 7 days). This gives users time to decide whether they accept the new commission or wish to unstake their funds.
{% endhint %}

## Proof of Concept

Add the following test to `PlumeStakingDiamond.t.sol`:

```solidity
function testValidatorCanSteal_userRewards_byIncreasingCommissionToTheMax() public {
        // Set a very specific reward rate for predictable results
        uint256 rewardRate = 1e18; // 1 PUSD per second
        vm.startPrank(admin);
        address[] memory tokens = new address[](1);
        tokens[0] = address(pUSD);
        uint256[] memory rates = new uint256[](1);
        rates[0] = rewardRate;
        RewardsFacet(address(diamondProxy)).setRewardRates(tokens, rates);

        // Make sure treasury is properly set
        RewardsFacet(address(diamondProxy)).setTreasury(address(treasury));

        // Ensure treasury has enough PUSD by transferring tokens
        uint256 treasuryAmount = 100 ether;
        pUSD.transfer(address(treasury), treasuryAmount);
        vm.stopPrank();

        // Set a 1% commission rate for the validator
        vm.startPrank(validatorAdmin);
        uint256 newCommission = 1e16;
        ValidatorFacet(address(diamondProxy)).setValidatorCommission(
            DEFAULT_VALIDATOR_ID,
            newCommission
        );
        vm.stopPrank();

        // user stake with validator that has 1% commission
        uint256 initialStake = 10 ether;
        vm.startPrank(user1);
        StakingFacet(address(diamondProxy)).stake{value: initialStake}(
            DEFAULT_VALIDATOR_ID
        );
        vm.stopPrank();

        // Move time forward to accrue rewards
        vm.roll(block.number + 1000);
        vm.warp(block.timestamp + 1000);

        // Trigger reward/commission accrual updates 
        ValidatorFacet(address(diamondProxy)).forceSettleValidatorCommission(DEFAULT_VALIDATOR_ID);

        // user claim his rewards
        vm.prank(user1);
        uint256 userRewards = RewardsFacet(address(diamondProxy)).claim(address(pUSD), DEFAULT_VALIDATOR_ID);
        console2.log("User claimed %e in rewards paying 1% commission", userRewards);
        uint256 validatorCommission = ValidatorFacet(address(diamondProxy)).getAccruedCommission(DEFAULT_VALIDATOR_ID, address(pUSD));
        console2.log("Validator accrued commission is: %e", validatorCommission);


        // validator bump his comission to the maximum
        vm.startPrank(validatorAdmin);
        newCommission = 50e16; // 50%
        ValidatorFacet(address(diamondProxy)).setValidatorCommission(
            DEFAULT_VALIDATOR_ID,
            newCommission
        );
        vm.stopPrank();
        console2.log("validator bumped his comission to 50%");


        // Move time forward to accrue rewards - same time as before so we can precisely see the difference for the same time delta
        vm.roll(block.number + 1000);
        vm.warp(block.timestamp + 1000);
    
        // user claim his rewards
        vm.prank(user1);
        uint256 newUserRewards = RewardsFacet(address(diamondProxy)).claim(address(pUSD), DEFAULT_VALIDATOR_ID);
        console2.log("User claimed only %e when he should have received %e", newUserRewards, userRewards);

        validatorCommission = ValidatorFacet(address(diamondProxy)).getAccruedCommission(DEFAULT_VALIDATOR_ID, address(pUSD));
        console2.log("Validator unfairly accrued commission is: %e", validatorCommission);
    }
```

Run:

```
forge test --mt testValidatorCanSteal_userRewards_byIncreasingCommissionToTheMax --via-ir -vv
```

Example output:

```
[PASS] testValidatorCanSteal_userRewards_byIncreasingCommissionToTheMax() (gas: 976250)
Logs:
  User claimed 9.9e21 in rewards paying 1% commission
  Validator accrued commission is: 1e20
  validator bumped his comission to 50%
  User claimed only 5e21 when he should have received 9.9e21
  Validator unfairly accrued commission is: 5.1e21

  Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.15ms (1.25ms CPU time)
```
