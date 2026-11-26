# \#98088 [SC-Low] LOW-59 fix is incomplete: `CfdEngine` and `CfdEngineAdmin` still allow `renounceOwnership()`, which makes degraded mode permanent

**Submitted on Nov 5th 2026 at 00:33:20 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98088
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

### Brief/Intro

After Cecuro LOW-59 ("Owner can leave emergency recovery ownerless"), the team made `renounceOwnership()` revert on `HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator`. Their stated reason: "governance must retain emergency recovery authority". `CfdEngine` (via `CfdTreasury`) and `CfdEngineAdmin` were not changed. The LOW-59 remediation said to override it "in every governance-bearing perps contract". `CfdEngine.clearDegradedMode()` is the only code path that sets `degradedMode = false`, and it is `onlyOwner`. If the engine owner renounces, degraded mode can never be cleared, even after the pool is recapitalized and solvent.

### Vulnerability Details

The fix exists in three contracts:

```solidity
// HousePool.sol:265 (same in OrderRouterAdmin.sol:125, EmergencyPauseCoordinator.sol:102)
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

The engine side has no override:

```solidity
// CfdTreasury.sol:10
@> abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step {   // no renounceOwnership override
// CfdEngineAdmin.sol:17
@> contract CfdEngineAdmin is Ownable2Step {                          // no renounceOwnership override
```

`degradedMode = true` is set at `CfdEngine.sol:1366`. The only reset is:

```solidity
// CfdEngine.sol:658-667
@> function clearDegradedMode() external onlyOwner {
       if (!degradedMode) revert CfdEngine__NotDegraded();
       if (_availableCashForFreshPoolPayouts() < _maxLiability()) revert CfdEngine__StillInsolvent();
       degradedMode = false;
       emit DegradedModeCleared();
   }
```

`degradedMode = false` appears nowhere else in `packages/perps/src`. After `engine.renounceOwnership()`, `owner() == address(0)`, so `clearDegradedMode()` reverts for every caller. `engineAdmin.renounceOwnership()` also removes `proposeRiskConfig`, and the engine cannot rotate its admin (`setDependencies` is one-time, `CfdEngine.sol:346-381`). The docs describe the owner as the recovery authority: `SECURITY.md:246`, `SECURITY.md:1504-1511` (runbook step 5: "Clear degraded mode only after raw solvency is restored"), `EMERGENCY_RESPONSE_GUIDE.md:87` ("Governance owns recovery").

Difference from LOW-59: LOW-59 named only `OrderRouterAdmin`, `HousePool`, `EmergencyPauseCoordinator`. This report covers the engine contracts the fix missed, and the engine-specific consequence (degraded mode permanent).

### Impact Details

No funds are lost. Closes, liquidations and free-margin withdrawals still work. The market can never accept new risk again: every open fails with `DEGRADED_MODE` even though the pool is solvent. The engine cannot be replaced (one-time wiring, non-upgradeable), so recovery needs a full redeploy. Needs an owner mistake or key compromise, the same precondition the team accepted for LOW-59. `MarginClearinghouse.sol:30` has the same inheritance; renouncing there permanently disables `rescueToken` (`MarginClearinghouse.sol:324`).

### Mitigation

```diff
 // CfdTreasury.sol, CfdEngineAdmin.sol, MarginClearinghouse.sol
+    function renounceOwnership() public view override onlyOwner {
+        revert OwnableInvalidOwner(address(0));
+    }
```

### References

- `packages/perps/src/CfdTreasury.sol:10` and `CfdEngine.sol:46`: Ownable2Step base, no override
- `packages/perps/src/CfdEngineAdmin.sol:17`: no override
- `packages/perps/src/CfdEngine.sol:658-667`: `clearDegradedMode`, sole reset path
- `packages/perps/src/CfdEngine.sol:1366`: degraded latch
- `packages/perps/src/CfdEngine.sol:346-381`: one-time `setDependencies`
- `packages/perps/src/MarginClearinghouse.sol:30,324`: same pattern, `rescueToken`
- `HousePool.sol:265`, `OrderRouterAdmin.sol:125`, `EmergencyPauseCoordinator.sol:102`: LOW-59 fix pattern


## Proof of Concept

## Proof of Concept

**Conditions under which this insight is valuable**
- The LOW-59 fix at `a82d4fe` covers `HousePool.sol:265`, `OrderRouterAdmin.sol:125` and `EmergencyPauseCoordinator.sol:102` only. The LOW-59 remediation asked for the override "in every governance-bearing perps contract". This report shows that fix is incomplete for the engine contracts.

Save as `packages/perps/test/insights_privileged/RenounceEngineOwnership.t.sol` and run `forge test --match-path test/insights_privileged/RenounceEngineOwnership.t.sol -vv` from `packages/perps`. It reuses the repo's `BasePerpTest` harness and the risk params of the existing degraded-mode fixture.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

contract RenounceEngineOwnershipInsight is BasePerpTest {

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0, maxSkewRatio: 1e18, maintMarginBps: 100, initMarginBps: 150, fadMarginBps: 300,
            baseCarryBps: 500, minBountyAssets: 5e6, bountyBps: 10, keeperShareBps: 5000, protocolShareBps: 0
        });
    }

    function test_EngineRenounceLeavesDegradedModeUnrecoverable() public {
        address longT = address(0xB011);
        address shortT = address(0xBEA2);
        address newT = address(0xCAFE);
        _fundTrader(longT, 100_000e6);
        _fundTrader(shortT, 100_000e6);
        _fundTrader(newT, 100_000e6);

        // 1. Profitable close reveals insolvency -> degraded mode latches.
        _open(shortT, CfdTypes.Side.SHORT, 999_000e18, 50_000e6, 1e8);
        _open(longT, CfdTypes.Side.LONG, 500_000e18, 50_000e6, 1e8);
        _close(longT, CfdTypes.Side.LONG, 500_000e18, 20_000_000);
        assertTrue(engine.degradedMode(), "setup: degraded");

        // 2. Governance recapitalizes; solvency restored.
        settlementToken.mint(address(this), 500_000e6);
        settlementToken.approve(address(engine), 500_000e6);
        engine.recapitalizePool(500_000e6);
        vm.prank(address(juniorVault));
        pool.reconcile();
        // Control: with an owner, recovery succeeds right now.
        uint256 snap = vm.snapshotState();
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "control: owner can recover");
        vm.revertToState(snap);
        assertTrue(engine.degradedMode());

        // 3. LOW-59 fix: these refuse renunciation ...
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableInvalidOwner.selector, address(0)));
        pool.renounceOwnership();
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableInvalidOwner.selector, address(0)));
        routerAdmin.renounceOwnership();
        // ... but the engine and its admin still accept it.
        engine.renounceOwnership();
        engineAdmin.renounceOwnership();
        assertEq(engine.owner(), address(0));
        assertEq(engineAdmin.owner(), address(0));

        // 4. The sole degraded-mode exit is unreachable.
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engine.clearDegradedMode();
        ICfdEngineAdminHost.EngineRiskConfig memory stricter = _engineRiskConfig();
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engineAdmin.proposeRiskConfig(stricter);

        // 5. New opens stay rejected although the pool is solvent.
        CfdTypes.Order memory o = CfdTypes.Order({
            account: newT, sizeDelta: 10_000e18, marginDelta: 1000e6, targetPrice: 1e8,
            commitTime: uint64(block.timestamp), commitBlock: uint64(block.number), orderId: 0,
            side: CfdTypes.Side.LONG, isClose: false
        });
        uint256 depth = pool.totalAssets();
        vm.expectRevert(
            abi.encodeWithSelector(
                ICfdEngineTypes.CfdEngine__TypedOrderFailure.selector,
                CfdEnginePlanTypes.ExecutionFailurePolicyCategory.ProtocolStateInvalidated,
                uint8(CfdEnginePlanTypes.OpenRevertCode.DEGRADED_MODE),
                false
            )
        );
        vm.prank(address(router));
        engine.processOrderTyped(o, 1e8, depth, uint64(block.timestamp));
        assertTrue(engine.degradedMode(), "degraded mode is now permanent");
    }
}
```

Result (actually run, `forge test --match-path test/insights_privileged/RenounceEngineOwnership.t.sol -vv`):

```
Ran 1 test for test/insights_privileged/RenounceEngineOwnership.t.sol:RenounceEngineOwnershipInsight
[PASS] test_EngineRenounceLeavesDegradedModeUnrecoverable() (gas: 2264672)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.66ms (3.79ms CPU time)
```

