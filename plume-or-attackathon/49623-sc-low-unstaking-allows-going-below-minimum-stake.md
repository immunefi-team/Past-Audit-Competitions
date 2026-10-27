# 49623 sc low unstaking allows going below minimum stake

**Submitted on Jul 17th 2025 at 19:05:26 UTC by @Blobism for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49623
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `unstake` method in `StakingFacet` for unstaking a particular amount does not check if the remaining stake is still above the minimum stake. A staker can thus leave a dust stake amount with a validator.

### Vulnerability Details

Most places in `StakingFacet` confirm that the user stake is above `minStakeAmount`, but unstaking with a specific amount does not do this:

```solidity
function unstake(uint16 validatorId, uint256 amount) external returns (uint256 amountUnstaked) {
    if (amount == 0) {
        revert InvalidAmount(0);
    }
    return _unstake(validatorId, amount);
}
```

### Impact Details

Anyone can go below the minimum stake parameter with this method. Financial incentives for this are not immediately clear, but this does lead to storage bloat.

## References

See `plume/src/facets/StakingFacet.sol`

## Proof of Concept

<details>

<summary>PoC — apply diff and run failing test</summary>

The PoC below demonstrates how a user can end up with having just 1 wei staked, below the minimum stake.

Save the diff below to `poc.diff` then run `git apply poc.diff`. Run like this:

```bash
forge test --via-ir --match-path test/PlumeStakingDiamond.t.sol --match-test testGetAccruedCommission_Direct
```

```diff
diff --git a/plume/test/PlumeStakingDiamond.t.sol b/plume/test/PlumeStakingDiamond.t.sol
index eaa5bd5..3fc2214 100644
--- a/plume/test/PlumeStakingDiamond.t.sol
+++ b/plume/test/PlumeStakingDiamond.t.sol
@@ -1062,7 +1062,7 @@ contract PlumeStakingDiamondTest is Test {
         vm.stopPrank();
 
         // Create validator with 10% commission
-        uint256 initialStake = 10 ether;
+        uint256 initialStake = 1 ether;
         vm.startPrank(user1);
         StakingFacet(address(diamondProxy)).stake{value: initialStake}(
             DEFAULT_VALIDATOR_ID
@@ -1089,9 +1089,24 @@ contract PlumeStakingDiamondTest is Test {
         vm.startPrank(user1);
 
         // Unstake a minimal amount to trigger reward update
-        StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, 1); // Unstake 1 wei
+        StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, (1 ether) - 1); // Leave 1 wei
         vm.stopPrank();
 
+        assertEq(
+            ManagementFacet(address(diamondProxy)).getMinStakeAmount(),
+            1 ether,
+            "Min stake amount is 1 ether"
+        );
+
+        PlumeStakingStorage.StakeInfo memory stakeInfo = StakingFacet(
+            address(diamondProxy)
+        ).stakeInfo(user1);
+        assertEq(
+            stakeInfo.staked,
+            1,
+            "user1 has 1 wei staked"
+        );
+
         // Check that some commission has accrued (positive amount)
         uint256 commission = ValidatorFacet(address(diamondProxy))
             .getAccruedCommission(DEFAULT_VALIDATOR_ID, address(pUSD));
```

</details>
