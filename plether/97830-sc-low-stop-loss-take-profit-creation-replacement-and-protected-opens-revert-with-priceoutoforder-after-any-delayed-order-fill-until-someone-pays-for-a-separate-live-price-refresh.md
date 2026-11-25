# \#97830 [SC-Low] Stop-loss/take-profit creation, replacement and protected opens revert with `PriceOutOfOrder` after any delayed-order fill, until someone pays for a separate live price refresh

**Submitted on Nov 3rd 2026 at 13:30:42 UTC by @enowk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97830
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In Plether Perps, any keeper fill of any trader's delayed order moves the Engine's mark price ahead of the Oracle's last accepted live price. The project documents this as a normal state. In that state, `PositionProtectionBook` calls an Oracle view that reverts with `PletherOracle__PriceOutOfOrder`. The three trader entry points for stop-loss/take-profit (TP/SL) then revert for everyone: `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection`. This happens even though the accepted price is still fresh and the Book's own guard explicitly accepts the state. They keep reverting until someone sends a paid Pyth update through `OrderRouter.updateMarkPrice`, and the next fill breaks them again. No attacker is needed, because ordinary keeper order flow creates the state. In production, traders would regularly be unable to add, tighten or attach a stop-loss unless they first pay for a refresh in a separate transaction and no fill lands in between. No funds are lost.

## Vulnerability Details

**Background: two prices.** Plether prices a basket of Pyth feeds. Two on-chain prices matter here:
- **Engine mark** (`CfdEngine.lastMarkPrice()` / `lastMarkTime()`): the accounting price. Live refreshes and delayed-order fills both advance it.
- **Accepted live basket** (`PletherOracle.getAcceptedBasketMark()`): the last basket accepted from a live Pyth update sent through the Router, for example `OrderRouter.updateMarkPrice(bytes[])`. Order fills do not write it.

Orders are delayed. A trader commits an order, and a permissionless keeper later fills it from the first Pyth tick published after the commit (a "historical" tick). TP/SL orders ("protections") live in `PositionProtectionBook`. Creating or replacing one validates the trigger prices against a current Oracle observation.

**Intended behaviour.**
- `CANONICAL_ENTRYPOINTS.md` lists `createPositionProtection` and `commitOpenOrderWithProtection` as the trader entry points for TP/SL ([L119-L120](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L119-L120)).
- `ORACLE_SYNCHRONIZATION.md` says "historical execution may advance the Engine without advancing the accepted live basket" ([L150-L151](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md?plain=1#L150-L151)). So "Engine mark ahead of the accepted basket" is a normal, expected state.
- The project's comment on finding MED-28 of the prior audit ([Cecuro report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)) describes the intended admission rule:
  > New protection, replacement and attached-open staging validate trigger geometry against the current accepted oracle observation […] The integrated synchronization guard also rejects arming when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark. A qualifying mark must be installed before arming can proceed.

  Under this rule, only the opposite state is rejected: the accepted mark *ahead of* the Engine. The state in this report meets every stated condition:
  - The accepted observation is current: it is within its freshness limit, as the PoC asserts.
  - It is not ahead of the Engine mark.
  - A qualifying mark is installed by the Book's own definition: `_acceptedMarkInstalled()` returns true when `acceptedTime < markTime` (code below).

  Admission still reverts.

**1. Order execution advances only the Engine mark.** When a delayed order is filled from its historical Pyth tick, the Router installs that tick as the Engine mark:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1058-L1059
```solidity
        if (result.publishTime > engine_.lastMarkTime()) {
            engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
```

The accepted live basket is written in one place only, inside `PletherOracle._resolveCurrentBasket` ([L796-L799](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L796-L799)). That function is reachable only from the Router's live update entry points ([L522-L539](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L522-L539)), never from historical order execution. After any fill whose tick is newer than the current mark, `acceptedTime < engine.lastMarkTime()`.

**2. Protection admission reads a view that rejects exactly that state.** `_freshProtectionObservation` calls `getLatestPrice` before running the Book's own guard:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L880-L885
```solidity
        IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
        currentMarkPrice = current.markPrice;
        currentPublishTime = current.publishTime;
        if (!_acceptedMarkInstalled()) {
            revert PositionProtectionBook__OracleMarkAhead();
        }
```

`getLatestPrice` builds its snapshot with publish-order enforcement switched on, and that enforcement reverts whenever the snapshot is older than the Engine mark:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L578
```solidity
        snapshot = _snapshotFromBasket(basket, policy, true);
```

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L892-L896
```solidity
        if (enforcePublishOrder) {
            uint64 markTime = engine.lastMarkTime();
            if (snapshot.publishTime < markTime) {
                revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
            }
```

The Book's own guard never gets to run on this path. It accepts this state, consistent with the MED-28 rule quoted above:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L893-L894
```solidity
        uint64 markTime = ENGINE.lastMarkTime();
        return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
```

**Origin.** Two commits combine to cause this:
- Commit [`81919218`](https://github.com/immunefi-team/audit-comp-plether/commit/819192182833597641504b38cce7751bbcacecb8) ("synchronize historical Pyth and enforce signed live basket") introduced the separate accepted live basket.
- Commit [`eadc5c58`](https://github.com/immunefi-team/audit-comp-plether/commit/eadc5c587e2f2f1ec26c51132472ab876d6edc15) ("harden trader protection…", the MED-28 fix) added the `getLatestPrice` call to protection admission. Its diff for `PositionProtectionBook.sol` shows the call being added. The parent version ([`PositionProtectionBook.sol` at `eadc5c58^`](https://github.com/immunefi-team/audit-comp-plether/blob/eadc5c587e2f2f1ec26c51132472ab876d6edc15%5E/packages/perps/src/PositionProtectionBook.sol)) has no `getLatestPrice` call.

**3. Sequence** (all accounts unprivileged):
1. Alice holds a LONG, and a live refresh has just synced the accepted basket with the Engine mark. Creating protection works in this state (PoC control).
2. Bob, an unrelated trader, commits an ordinary order. Any keeper executes it from the next Pyth tick after Bob's commit time. The Engine mark moves to that tick; the accepted basket is unchanged and still within its freshness limit.
3. Alice calls `createPositionProtection`. It reverts `PletherOracle__PriceOutOfOrder(acceptedTime, markTime)`, and keeps reverting in later blocks.
4. Carol calls `commitOpenOrderWithProtection`, and Alice calls `replacePositionProtection` to tighten an existing stop. Both revert the same way.
5. The zero-fee replay `OrderRouter.updateMarkPrice(new bytes[](0))` reverts the same way, so the state can only be cleared with a paid live Pyth update.
6. After a paid refresh, one more fill before the trader's next transaction brings the revert back.

**4. Why existing checks do not stop it.**
- The Book's lenient guard runs after the reverting call.
- Freshness is not the cause. In live markets, the `MarkRefresh` policy that `getLatestPrice` uses has the same limit as order execution, `orderExecutionStalenessLimit`, 60 s by default ([`OracleFreshnessPolicyLib.sol` L65-L66](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OracleFreshnessPolicyLib.sol#L65-L66), [`PletherOracle.sol` L76](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L76)). The PoC asserts that the accepted basket is within that limit when the revert happens.
- No protection entry point accepts price data except `triggerPositionProtection`, which wraps an `updateMarkPrice` refresh into the same call ([L371-L378](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L371-L378)). It serves only already-armed protections. Create, replace and protected open have no such refresh.

**5. Preconditions.** The market is live, and at least one order has been filled from a tick newer than the last live refresh. This is the ordinary result of keeper order flow. No capital, role or special timing is needed.

**6. Recommended fix.** In `PositionProtectionBook._freshProtectionObservation`, do not call the order-enforcing view when the Engine mark is ahead of the accepted basket. If `getAcceptedBasketMark()` returns `acceptedTime < ENGINE.lastMarkTime()`, use the Engine mark (price and time) as the observation. This matches `_acceptedMarkInstalled()` and the MED-28 rule. Alternatives:
- Expose a non-enforcing accepted-basket read for the Book.
- Add create/replace/protected-open variants that take `bytes[]` and perform the same atomic `updateMarkPrice` refresh that `triggerPositionProtection` already uses.
- Advance the accepted basket during historical execution.

## Impact Details

The program's Low impact is "Contract fails to deliver promised returns, but doesn't lose value". The documented TP/SL entry points, the protocol's risk-control feature, fail during normal operation. This is the closest listed impact; the harm is a recurring loss of that feature, not a loss of funds.
- **Who is affected:** every trader who tries to add, tighten or attach a stop-loss/take-profit between an order fill and the next live refresh.
  - A trader using a plain (non-EIP-7702) EOA can recover by sending a paid `updateMarkPrice` in one transaction and the protection call in the next. That only works if no fill lands in between.
  - A smart account that bundles both calls in one transaction succeeds.
  - The project's own client sends `createPositionProtection` as a single call with no refresh ([`packages/perps-aa-client/src/protection.ts`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa-client/src/protection.ts)). That package is out of scope and is cited only to show the affected path is used.
- **Funds:** none are lost or locked. Closes, liquidations and already-armed protections keep working.
- **Duration and cost:** each attempt requires the trader to pay one Pyth update fee plus gas for an extra transaction. The state returns with the next fill.
  - How often a trader hits it depends on the ratio of fill frequency to live-refresh frequency on mainnet, which I have not measured.
  - The reference keeper documentation describes an optional oracle updater that submits `updateMarkPrice` on a loop ([plether-app backend README, "Optional: Start The On-Chain Oracle Updater"](https://github.com/Plether-Fi/plether-app/blob/master/apps/backend/README.md#7-optional-start-the-on-chain-oracle-updater)). If it runs, it narrows the window but cannot close it: any fill after its refresh brings the revert back until its next cycle.
- **Attacker economics:** no attacker profit. Anyone can sustain the state for the cost of committing and executing minimum-size orders, but it also happens without anyone trying.

## References

- Code (pinned to `a82d4fef`): [`OrderRouterExecutionSidecar.sol` L1058-L1059](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1058-L1059), [`PositionProtectionBook.sol` L880-L894](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L880-L894), [`PletherOracle.sol` L892-L896](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L892-L896)
- Release parameters used by the PoC: [`DEPLOYMENT.md` L128-L140](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L128-L140)
- **Prior audit, MED-28** ([Cecuro report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)): its fix rejects arming when the accepted mark is ahead of the Engine mark. This report is the opposite case, which that fix's guard accepts but its added `getLatestPrice` call rejects. In other words, the MED-28 fix introduces a new bug.
- **Upstream PR #103** (https://github.com/Plether-Fi/plether-core/pull/103, merged 2026-10-10, before `81919218`): it fixed the same `PriceOutOfOrder` symptom when the root cause was Pyth storage lagging the Engine mark, and that fix is still in place. The root cause here is different: the later separate accepted basket is not advanced by order execution, and `eadc5c58` made protection admission depend on it.
- **Known-issue check:** I found no statement of this behaviour in the program's known-issues list, the `SECURITY.md` sections it names, the Cecuro report, the SC Audit Studio pre-audit, or the AI scan findings.



## Proof of Concept

## Proof of Concept

Runs in the repo's own Foundry suite at the pinned commit. No fork or RPC is needed. Only Pyth (the repo's `MockPyth`) and USDG are mocked, and no in-scope contract is modified. The test overrides `_riskParams()` with the release values in [`DEPLOYMENT.md`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L128-L140): maintenance margin 10 bps, initial margin 20 bps, frozen margin 300 bps, VPI 0.01e18, bounty 10 bps with a 1 USDG minimum.

How the PoC creates the precondition:
- Order fills use the repo's `BasePerpTest._mockPythUpdateData`, which sets the historical tick's publish time to the pending order's `commitTime + 1` ([`BasePerpTest.sol` L301-L330](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/BasePerpTest.sol#L301-L330)). Bob commits after the live refresh, so his fill's tick is newer than the accepted basket.
- The PoC asserts this (`assertGt(markTime, acceptedTime)`) and asserts that the accepted basket is still within `orderExecutionStalenessLimit`.
- Blocks advance one second at a time.

Setup and run:
```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the file below as packages/perps/test/poc/ProtectionEngineAheadPoC.t.sol
FOUNDRY_TEST=test/poc forge test --root packages/perps --match-path test/poc/ProtectionEngineAheadPoC.t.sol -vv
```
(`FOUNDRY_TEST=test/poc` makes forge compile only this test and its imports instead of the repo's whole test suite. That suite is very large; compiling all of it ran out of memory on our 24 GB machine.)

Expected output:
```
[PASS] test_HistoricalExecutionBlocksProtection() (gas: 7055862)
[PASS] test_ReplaceBlockedAndRefreshIsNotDurable() (gas: 7282956)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

What the assertions prove:
- **Control:** creation succeeds while the accepted basket equals the Engine mark.
- **After Bob's unrelated fill:** the Engine mark is ahead and the accepted basket is still fresh (both asserted).
- **Exact reverts:** `createPositionProtection` (twice, across blocks), `commitOpenOrderWithProtection` by a fresh trader, the zero-fee `updateMarkPrice` replay, and `replacePositionProtection` all revert with the exact `PletherOracle__PriceOutOfOrder(acceptedTime, markTime)`.
- **Refresh does not last:** a paid refresh followed by one more fill breaks replacement again.

`packages/perps/test/poc/ProtectionEngineAheadPoC.t.sol`:
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice PoC. Historical order execution advances the Engine mark past the Oracle's accepted live
///         basket. PositionProtectionBook._freshProtectionObservation() reads oracle.getLatestPrice(MarkRefresh), which
///         reverts PriceOutOfOrder in that state. Result: after any permissionless execution on a newer tick, traders
///         cannot create or replace TP/SL, or open with protection, until someone pays for a fresh live refresh.
///         Production risk parameters; only Pyth (MockPyth) and USDG are mocked; no in-scope contract is modified.
contract ProtectionEngineAheadPoCTest is BasePerpTest {

    address internal alice = address(0xA11CE);
    address internal carol = address(0xCA201);
    address internal bob = address(0xB0B);
    address internal execKeeper = address(0xE0EC);

    IPositionProtectionActions internal book;
    IPletherOracle internal oracle;

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
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        oracle = IPletherOracle(address(router.pletherOracle()));
        _fundTrader(alice, 20_000e6);
        _fundTrader(carol, 20_000e6);
        _fundTrader(bob, 20_000e6);
    }

    /// Live (non-historical) Pyth update at the current block time.
    function _live(
        uint256 price
    ) internal view returns (bytes[] memory d) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 prev = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, prev));
        }
        d = new bytes[](1);
        d[0] = abi.encode(ticks);
    }

    function _nextBlock() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _params(
        uint256 slTenths
    ) internal view returns (PositionProtectionTypes.PositionProtectionParams memory p) {
        p.stopLossTriggerPrice = engine.lastMarkPrice() * slTenths / 10; // LONG USD loses when the basket rises
        p.closeLimits.maxExecutionNotionalAssets = type(uint256).max;
        p.closeLimits.maxGrossAccountDebitAssets = type(uint256).max;
        p.closeLimits.maxActionChargeAssets = type(uint256).max;
        p.closeLimits.maxExplicitFeesAssets = type(uint256).max;
    }

    function _openRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = 10_000e18;
        request.marginDelta = 2000e6;
        request.targetPrice = 1;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 1 | 2 | 4,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// Any account commits a minimum-size order and any keeper executes it on the next historical tick.
    function _unrelatedExecution() internal {
        vm.prank(bob);
        uint64 id = router.commitOrder(CfdTypes.Side.SHORT, 1000e18, 500e6, 0, false);
        bytes[] memory hist = _mockPythUpdateData(1e8);
        vm.prank(execKeeper);
        router.executeOrder(id, hist);
    }

    function test_HistoricalExecutionBlocksProtection() public {
        // Alice opens a LONG; a live refresh then syncs the accepted basket with the Engine mark.
        vm.prank(alice);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 2000e6, 0, false);
        vm.prank(execKeeper);
        router.executeOrder(openId, _mockPythUpdateData(1e8));
        _nextBlock();
        router.updateMarkPrice(_live(1e8));
        (, uint256 accPrice, uint64 accTime) = oracle.getAcceptedBasketMark();
        assertEq(accTime, engine.lastMarkTime(), "accepted basket synced with Engine mark");
        assertEq(accPrice, engine.lastMarkPrice(), "accepted price synced with Engine mark");

        PositionProtectionTypes.PositionProtectionParams memory p11 = _params(11);

        // Control: protection creation succeeds in the synced state.
        uint256 snap = vm.snapshotState();
        vm.prank(alice);
        uint64 controlId = book.createPositionProtection(p11);
        assertGt(controlId, 0, "control: creation works when synced");
        vm.revertToState(snap);

        // An unrelated order is executed by a permissionless keeper on a later historical tick.
        _unrelatedExecution();
        uint64 markTime = engine.lastMarkTime();
        (,, uint64 acc) = oracle.getAcceptedBasketMark();
        assertGt(markTime, acc, "Engine mark advanced past the accepted basket");
        assertLe(block.timestamp - acc, oracle.orderExecutionStalenessLimit(), "accepted basket is still fresh");

        bytes memory outOfOrder =
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acc, markTime);

        // 1. createPositionProtection reverts, and stays reverted in later blocks.
        vm.prank(alice);
        vm.expectRevert(outOfOrder);
        book.createPositionProtection(p11);
        _nextBlock();
        vm.prank(alice);
        vm.expectRevert(outOfOrder);
        book.createPositionProtection(p11);

        // 2. commitOpenOrderWithProtection by a fresh trader reverts.
        OrderV3Types.OrderRequest memory req = _openRequest();
        vm.prank(carol);
        vm.expectRevert(outOfOrder);
        book.commitOpenOrderWithProtection(req, p11);

        // 3. The zero-fee replay of the accepted basket cannot repair it; a paid live Pyth update is required.
        vm.expectRevert(outOfOrder);
        router.updateMarkPrice(new bytes[](0));
    }

    function test_ReplaceBlockedAndRefreshIsNotDurable() public {
        vm.prank(alice);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 2000e6, 0, false);
        vm.prank(execKeeper);
        router.executeOrder(openId, _mockPythUpdateData(1e8));
        _nextBlock();
        router.updateMarkPrice(_live(1e8));
        PositionProtectionTypes.PositionProtectionParams memory p11 = _params(11);
        PositionProtectionTypes.PositionProtectionParams memory p12 = _params(12);
        vm.prank(alice);
        uint64 pid = book.createPositionProtection(p11);

        // 4. replacePositionProtection (tightening the stop) reverts after an unrelated execution.
        _unrelatedExecution();
        uint64 markTime = engine.lastMarkTime();
        (,, uint64 acc) = oracle.getAcceptedBasketMark();
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acc, markTime));
        book.replacePositionProtection(pid, p12);

        // 5. A paid refresh repairs it, but one more execution before Alice's next transaction breaks it again.
        _nextBlock();
        router.updateMarkPrice(_live(1e8));
        _unrelatedExecution();
        markTime = engine.lastMarkTime();
        (,, acc) = oracle.getAcceptedBasketMark();
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acc, markTime));
        book.replacePositionProtection(pid, p12);
    }

}
```

