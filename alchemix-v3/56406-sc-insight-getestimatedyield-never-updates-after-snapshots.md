# 56406 sc insight getestimatedyield never updates after snapshots

**Submitted on Oct 15th 2025 at 16:28:34 UTC by @spongebob for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56406
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

`getEstimatedYield` simply returns `params.estimatedYield` and that struct field is only populated in the constructor via `_params.estimatedYield` (no setter or later writes).

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/MYTStrategy.sol#L285>

```solidity
 function getEstimatedYield() public view returns (uint256) {
        return params.estimatedYield;
    }

```

`snapshotYield` updates the mutable state `estApr/estApy` and emits `YieldUpdated`, but never copies the result back into `params.estimatedYield`.

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/MYTStrategy.sol#L196-L215>

```solidity
// Add incentives to calculation if applicable
        uint256 rewardsRatePerSec;
        if (params.additionalIncentives == true) rewardsRatePerSec = _computeRewardsRatePerSecond();

        // Combine rates
        uint256 totalRatePerSec = baseRatePerSec + rewardsRatePerSec;
        uint256 apr = totalRatePerSec * SECONDS_PER_YEAR; // simple annualization (APR)
        uint256 apy = _approxAPY(totalRatePerSec);

        // Smoothing factor
        // TODO need to figure out how to ramp this up
        // Since first call is 0 the second call will be skewed
        // perhaps no smoothing on second pass
        uint256 alpha = 7e17; // 0.7
        estApr = _lerp(estApr, apr, alpha);
        estApy = _lerp(estApy, apy, alpha);

        lastSnapshotTime = uint64(currentTime);
        lastIndex = newIndex;
```

Because no other code touches `params.estimatedYield`, external callers of `getEstimatedYield()` always see the original constructor value even after snapshots run, so the method surfaces stale data.

## Recommendation

Update `snapshotYield()` to write `params.estimatedYield = estApy;` so the getter reflects the freshest snapshot.

## Proof of Concept

Add this test to `MYTStrategy.t.sol` and run `forge test --mt testPOC_GetEstimatedYield_Returns_Stale_Data -vvvv`

```solidity
function testPOC_GetEstimatedYield_Returns_Stale_Data() external {
        // SETUP: Deploy a mock strategy with a specific initial estimatedYield
        uint256 initialEstimatedYield = 100e18; // 100% initial yield from constructor

        // Verify the initial estimatedYield is set correctly
        vm.assertEq(strategy.getEstimatedYield(), initialEstimatedYield, "Initial estimatedYield should match constructor value");

        // STEP 1: Record the initial state
        uint256 estimatedYieldBefore = strategy.getEstimatedYield();
        uint256 estApyBefore = strategy.estApy();
        uint256 estAprBefore = strategy.estApr();

        // Initially, estApy and estApr should be 0 (not yet snapshotted)
        vm.assertEq(estApyBefore, 0, "estApy should be 0 before first snapshot");
        vm.assertEq(estAprBefore, 0, "estApr should be 0 before first snapshot");

        // STEP 2: Warp time forward to allow snapshot (MIN_SNAPSHOT_INTERVAL = 1 day)
        vm.warp(block.timestamp + 1 days + 1);

        // STEP 3: Call snapshotYield() to update the yield estimates
        // This will update estApr and estApy state variables
        uint256 returnedApy = strategy.snapshotYield();

        // STEP 4: Record state after snapshot
        uint256 estimatedYieldAfter = strategy.getEstimatedYield();
        uint256 estApyAfter = strategy.estApy();
        uint256 estAprAfter = strategy.estApr();

        // VULNERABILITY: getEstimatedYield() still returns the original constructor value
        // even though snapshotYield() has updated estApy and estApr
        vm.assertEq(estimatedYieldAfter, initialEstimatedYield, "VULNERABILITY: getEstimatedYield() still returns constructor value");
        vm.assertEq(estimatedYieldAfter, estimatedYieldBefore, "VULNERABILITY: getEstimatedYield() unchanged after snapshot");

        // The estApy and estApr state variables HAVE been updated
        // (In this mock strategy, _computeBaseRatePerSecond returns 0, so estApy will be smoothed from 0)
        // But the key point is that params.estimatedYield was NOT updated

        // STEP 5: Call snapshotYield() again after another interval
        vm.warp(block.timestamp + 1 days + 1);
        uint256 returnedApy2 = strategy.snapshotYield();

        uint256 estimatedYieldAfterSecondSnapshot = strategy.getEstimatedYield();
        uint256 estApyAfterSecondSnapshot = strategy.estApy();

        // VULNERABILITY: Even after multiple snapshots, getEstimatedYield() returns stale data
        vm.assertEq(estimatedYieldAfterSecondSnapshot, initialEstimatedYield, "VULNERABILITY: getEstimatedYield() still stale after 2nd snapshot");

        // STEP 6: Demonstrate the disconnect between getEstimatedYield() and actual yield tracking
        // The function comment says "get the current snapshotted estimated yield"
        // but it actually returns the ORIGINAL constructor value, not the current snapshot

        // If we manually check params.estimatedYield, it should still be the original value
        (
            address owner,
            string memory name,
            string memory protocol,
            IMYTStrategy.RiskClass riskClass,
            uint256 cap,
            uint256 globalCap,
            uint256 estimatedYield,
            bool additionalIncentives,
            uint256 slippageBPS
        ) = strategy.params();

        vm.assertEq(estimatedYield, initialEstimatedYield, "params.estimatedYield never updated");

    }
```
