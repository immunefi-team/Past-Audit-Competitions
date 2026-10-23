# 50082 sc low protocol lets validators operate with dust amounts making attacks risk free

**Submitted on Jul 21st 2025 at 15:10:02 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50082
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency

## Description

### Brief/Intro

The `_unstake` function doesn't check if the remaining stake amount is below the minimum stake amount, allowing validators to operate actively while having a dust amount of funds staked (i.e., 1 wei).

### Vulnerability Details

The protocol does not enforce the minimum stake amount when a user partially unstakes. It allows stakers/validators to remain with stake amounts below `minStakeAmount`.

Vulnerable snippet:

```solidity
https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L366-L390
```

```solidity
    function _unstake(uint16 validatorId, uint256 amount) internal returns (uint256 amountToUnstake) {
        PlumeStakingStorage.Layout storage $s = PlumeStakingStorage.layout();

        // @audit - Validations does not validate min stake amount
        _validateValidatorForUnstaking(validatorId);
        if (amount == 0) {
            revert InvalidAmount(amount);
        }
        if ($s.userValidatorStakes[msg.sender][validatorId].staked < amount) {
            revert InsufficientFunds($s.userValidatorStakes[msg.sender][validatorId].staked, amount);
        }

        ...
        return amount;
    }
```

With the current logic, validators can operate with dust amounts, increasing the likelihood of validators acting maliciously. The invariant that every active validator holds at least the minimum stake amount is broken.

Examples of consequences:

* Two malicious validators could skip voting in the slashing process to avoid each other being slashed.
* Admin deactivates the two validators above; new/existing validators can repeat the same behavior when their stake/funds are dust/below the minimum stake amount.
* Malicious validators acting together can vote to slash legitimate validators, reducing their funds to zero.

## Impact

The protocol permits partial unstaking that drops validators below minimum stake requirements, enabling validators to act maliciously with minimal financial risk.

## Recommendation

Enforce the minimum stake amount when a user partially unstakes. Suggested patch:

```diff
    function _unstake(uint16 validatorId, uint256 amount) internal returns (uint256 amountToUnstake) {
        PlumeStakingStorage.Layout storage $s = PlumeStakingStorage.layout();
        ...

+        if ($s.userValidatorStakes[msg.sender][validatorId].staked - amount < $s.minStakeAmount) {
+            revert StakeAmountTooSmall($s.userValidatorStakes[msg.sender][validatorId].staked - amount, $s.minStakeAmount);
+        }    

        ...
    }
```

{% hint style="info" %}
Add a custom error (e.g., `StakeAmountTooSmall`) if not already present, and ensure the check is consistent with any edge-cases (e.g., full unstake vs partial unstake).
{% endhint %}

## Proof of Concept

Add the PoC below into `PlumeStakingDiamond.t.sol`:

```solidity
    function testStakerCanLeaveValidatorWithDustAmountStaked() public {
        uint256 amount = 1e18;
        vm.startPrank(user1);
        StakingFacet(address(diamondProxy)).stake{value: amount}(
            DEFAULT_VALIDATOR_ID
        );
        assertEq(StakingFacet(address(diamondProxy)).amountStaked(), amount);

        // Unstake
        StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, 0.9e18);
        assertEq(StakingFacet(address(diamondProxy)).amountCooling(), 0.9e18);
        assertEq(StakingFacet(address(diamondProxy)).amountStaked(), 0.1e18);

        // print total staked for this validator
        (,uint256 validatorTotalStaked,) = ValidatorFacet(address(diamondProxy)).getValidatorInfo(DEFAULT_VALIDATOR_ID);
        console2.log("minStakeAmount: %e", ManagementFacet(address(diamondProxy)).getMinStakeAmount());
        console2.log("Validator has a total staked of %e", validatorTotalStaked);

        vm.stopPrank();
    }
```

Run:

```
forge test --mt testStakerCanLeaveValidatorWithDustAmountStaked -vv --via-ir
```

Output:

```
[PASS] testStakerCanLeaveValidatorWithDustAmountStaked() (gas: 554340)
Logs:
  minStakeAmount: 1e18
  Validator has a total staked of 1e17
```
