# 50477 sc high validator loses all accrued commission when reward token is removed

**Submitted on Jul 25th 2025 at 08:19:56 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50477
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

Incorrect logic causes a validator to lose all accrued commission when a reward token is removed.

### Vulnerability Details

When requesting commission, the modifier `_validateIsToken` prevents the validator from claiming his commission for a reward token that was removed.

Relevant code:

```solidity
    modifier _validateIsToken(
        address token
    ) {
@>        if (!PlumeStakingStorage.layout().isRewardToken[token]) {
            revert TokenDoesNotExist(token);
        }
        _;
    }

    function requestCommissionClaim(
        uint16 validatorId,
        address token
    )
        external
        onlyValidatorAdmin(validatorId)
        nonReentrant
        _validateValidatorExists(validatorId)
@>        _validateIsToken(token)
    {
      ...
    }
```

The root cause is that the modifier should check if the token is a historical token (i.e., claimable even after removal), not whether it is currently an active reward token.

### Impact

The validator loses all accrued commission for the reward token that was removed, causing permanent freezing of those funds.

### Recommendation

Change the modifier to check if the token is a historical token, not the reward token.

Suggested patch:

```diff
    modifier _validateIsToken(
        address token
    ) {
-        if (!PlumeStakingStorage.layout().isRewardToken[token]) {
-            revert TokenDoesNotExist(token);
-        }
+        if (!PlumeStakingStorage.layout().isHistoricalRewardToken[token]) {
+            revert TokenDoesNotExist(token);
+        }
        _;
    }
```

{% hint style="warning" %}
Ensure that tokens removed from active rewards but for which users/validators have accrued balances remain claimable by checking a historical/legacy registry rather than the current active reward list.
{% endhint %}

## Proof of Concept

<details>

<summary>Context &#x26; PoC test (expand)</summary>

Context

* Validator accrues commission over time.
* One of the reward tokens he accrued commission for is removed.
* Validator is unable to claim his commission for the removed reward token.

PoC test to add in `PlumeStakingDiamond.t.sol`:

```solidity
function testRequestComission_whenRewardTokenIsRemoved() public {
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

        // Set a 10% commission rate for the validator
        vm.startPrank(validatorAdmin);
        uint256 newCommission = 10e16;
        ValidatorFacet(address(diamondProxy)).setValidatorCommission(
            DEFAULT_VALIDATOR_ID,
            newCommission
        );
        vm.stopPrank();

        // Create validator with 10% commission
        uint256 initialStake = 10 ether;
        vm.startPrank(user1);
        StakingFacet(address(diamondProxy)).stake{value: initialStake}(
            DEFAULT_VALIDATOR_ID
        );
        vm.stopPrank();

        // Move time forward to accrue rewards
        vm.roll(block.number + 100);
        vm.warp(block.timestamp + 100);

        // Trigger reward updates by having a user interact with the system
        // This will internally call updateRewardsForValidator
        vm.startPrank(user2);
        StakingFacet(address(diamondProxy)).stake{value: 1 ether}(
            DEFAULT_VALIDATOR_ID
        );
        vm.stopPrank();

        // Move time forward again
        vm.roll(block.number + 1);
        vm.warp(block.timestamp + 1);

        // Interact again to update rewards once more
        vm.prank(user1);
        // Unstake a minimal amount to trigger reward update
        StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, 1); // Unstake 1 wei

        // Check that some commission has accrued (positive amount)
        uint256 commission = ValidatorFacet(address(diamondProxy))
            .getAccruedCommission(DEFAULT_VALIDATOR_ID, address(pUSD));
        assertGt(commission, 0, "Commission should be greater than 0");

        // print how much commission validator accrued
        console2.log("Validator accrued comission is %e", ValidatorFacet(address(diamondProxy)).getAccruedCommission(DEFAULT_VALIDATOR_ID, address(pUSD)));

        // remove reward token
        vm.prank(admin);
        RewardsFacet(address(diamondProxy)).removeRewardToken(address(pUSD));

        // Try to request comission, but transaction reverts
        // as the reward token was removed
        vm.prank(validatorAdmin);
        vm.expectRevert(abi.encodeWithSelector(TokenDoesNotExist.selector, address(pUSD)));
        ValidatorFacet(address(diamondProxy)).requestCommissionClaim(
            DEFAULT_VALIDATOR_ID,
            address(pUSD)
        );
    }
```

Run: `forge test --mt testRequestComission_whenRewardTokenIsRemoved --via-ir`

Output:

```javascript
Ran 1 test for test/PlumeStakingDiamond.t.sol:PlumeStakingDiamondTest
[PASS] testRequestComission_whenRewardTokenIsRemoved() (gas: 1462778)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.25ms (448.13µs CPU time)
```

</details>
