# \#99960 [SC-Medium] A trader withdraws free settlement before a full close and the close waives the action charge that settlement would have paid

**Submitted on Nov 15th 2026 at 22:04:15 UTC by @v4rvl for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99960
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

A trader closes a 100,000 USDG long (1000 lots at price 1.00, 500 USDG margin) and is assessed an action charge of 481,576,883 atoms. If the trader keeps its free settlement, the close collects the whole charge. If the trader first withdraws its free settlement through `MarginClearinghouse.withdraw`, the same close collects only 410,099,676 atoms and waives 71,477,207 atoms. Against the same close without the withdrawal, the trader ends 71,477,207 atoms richer and pool assets end 71,477,207 atoms lower (LP bob loses 71,405,801 atoms and the seed LP loses 71,406 atoms). The treasury is unchanged.

The trader acts alone, with its own funds, through public entrypoints. In a depleted-depth world (price 1.90, 10 lots, 200 USDG margin) the waiver is 152,552,363 atoms. In larger thin-margin positions it is 342,478,920 and 713,303,618 atoms.

The full close collects its action charge from the account's buckets in a fixed order and waives the rest (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1093-1133`). Free settlement is one of those buckets, but nothing reserves it for the close. The withdrawal guard checks position health, unsettled carry and the VPI reserve, never the charge of a close. The accounting specification says a full close waives "only the shortfall after all eligible funds are exhausted", and it forbids shielding settlement by parking it in committed order margin before terminal settlement. The code closes the parking route and leaves the withdrawal route open.

This is not a profitable attack on its own. A full open and close cycle loses money, pool assets still rise on every close, and the route is closed in degraded mode. The loss is VPI and carry revenue that the specification assigns to the LPs and that they do not receive.

## Vulnerability Detail

1. **The close collects from buckets in a fixed order and waives the rest.** `packages/perps/src/libraries/CfdEnginePlanLib.sol:1093-1133` (`_planCloseCashSources`):

```solidity
        uint256 collectibleFreeSettlementAssets = snap.accountBuckets.freeSettlementAssets + releasedVpiReserveAssets;
        if (terminal) {
            // The full-close reserve is released before pledge attribution and cash collection.
            collectibleFreeSettlementAssets += delta.liquidationReserveReleaseAssets;
        }
        ...
        if (terminal) {
            delta.actionCommittedMarginConsumedAssets = actionChargeAfterFreeAssets
                < snap.lockedBuckets.committedOrderMarginAssets
                ? actionChargeAfterFreeAssets
                : snap.lockedBuckets.committedOrderMarginAssets;
        }
        delta.actionChargeCollectedAssets = delta.vpiRebateReserveConsumedAssets + delta.actionReserveConsumedAssets
            + freeConsumedAssets + delta.actionChargeFromReleasedMarginAssets
            + delta.actionCommittedMarginConsumedAssets;
        delta.actionChargeWaivedAssets = delta.actionChargeToCollectAssets - delta.actionChargeCollectedAssets;
```

   The order is: spendable action reserve, then free settlement plus released VPI backing and the released liquidation reserve (`:1111-1115`), then released pledge, then committed order margin (`:1124-1129`). Whatever is left is waived (`:1133`) and reported in `ActionChargeSettled` (`packages/perps/src/CfdEngineSettlementSidecar.sol:673-679`).

2. **Free settlement can be withdrawn while the position is open.** `MarginClearinghouse.withdraw` (`packages/perps/src/MarginClearinghouse.sol:403`) calls `_withdraw`, which debits the balance and calls the engine guard (`packages/perps/src/MarginClearinghouse.sol:441-465`, guard call at `:464`). The engine's `checkWithdraw` (`packages/perps/src/CfdEngine.sol:730`) runs `validateWithdraw` (`packages/perps/src/CfdEngineSettlementSidecar.sol:255-285`). For an account with a position, the checks are degraded mode, a live mark, carry realisation and `_requireWithdrawPositionHealthy` (`packages/perps/src/CfdEngineSettlementSidecar.sol:287-305`):

```solidity
        if (engine.unsettledCarryAssets(account) != 0) {
            revert ICfdEngineTypes.CfdEngine__WithdrawBlockedByOpenPosition();
        }
        IMarginClearinghouse clearinghouse = IMarginClearinghouse(host.clearinghouse());
        if (clearinghouse.vpiRebateReserveAssets(account) < _negativeVpiReserveTarget(pos.vpiAccrued)) {
            revert ICfdEngineTypes.CfdEngine__WithdrawBlockedByOpenPosition();
        }
        uint256 reachableAssets = clearinghouse.pnlPledgeAssets(account) + engine.traderClaimBalanceAssets(account);
        if (_withdrawPositionLiquidatable(engine, pos, account, price, reachableAssets)) {
            revert ICfdEngineTypes.CfdEngine__WithdrawBlockedByOpenPosition();
        }
```

   No check compares the remaining free settlement with the action charge that a full close of the position would assess, or with a pending close order.

3. **The close commit does not reserve anything for the charge.** `CfdEnginePlanner.planCloseCommit` (`packages/perps/src/CfdEnginePlanner.sol:19-24`) calls `CloseCommitmentLib.project` (`packages/perps/src/libraries/CloseCommitmentLib.sol:13-55`), which checks the size, the size quantum and the mark, and projects carry. It does not assess or lock the action charge.

4. **The parking route is closed, the withdrawal route is not.** The specification says "a user must not shield otherwise reachable settlement by parking it in queued committed margin right before terminal settlement" (`packages/perps/ACCOUNTING_SPEC.md:1067`). The code enforces that sentence twice: a full close takes the charge from committed order margin (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1124-1129`), and `reserveCommittedOrderMargin` checkpoints carry before it locks margin (`packages/perps/src/MarginClearinghouse.sol:632`). For a flat account with a pending open, withdrawals are also bounded by the order's funded floor (`packages/perps/src/CfdEngineSettlementSidecar.sol:261-272`). Withdrawing the same settlement to the trader's wallet is not bounded at all, and it has the same effect on the close.

5. **Result.** The trader moves its free settlement out, the full close finds only pledge and liquidation reserve, and the planner waives the difference. The waived amount is exactly the free settlement the close would have consumed: with free settlement left equal to the waiver nothing is waived, and with one atom less exactly one atom is waived (`test_Ordinary_P100_WithdrawFirstWaivesCloseCharge`, worlds W2 and W2b).

## Impact

Program impact row (Low): "Contract fails to deliver promised returns, but doesn't lose value."

**Promised return.** The specification assigns the close's VPI and carry charge to the LPs whenever the account can pay it. It says a full close must "waive only the shortfall after all eligible funds are exhausted" (`packages/perps/ACCOUNTING_SPEC.md:1035-1036`), lists free settlement among those funds (`packages/perps/ACCOUNTING_SPEC.md:1042-1044`), and says "A waiver at the engine settlement boundary implies no eligible cash remains" (`packages/perps/ACCOUNTING_SPEC.md:1045`). It also says a user must not shield reachable settlement before terminal settlement (`packages/perps/ACCOUNTING_SPEC.md:1067`). With one withdrawal before the close, the trader keeps settlement that the close would have collected, and the LPs do not receive that part of the charge.

**Fair control.** The control is the same close of the same position at the same price and block, with the free settlement kept (world W1). The only change is the trader's withdrawal (W3 between commit and execution, W4 before the commit). Measured deltas, in USDG atoms:

| world | assessed | recovered | waived | trader | pool assets | LP bob | seed LP | treasury |
|---|---|---|---|---|---|---|---|---|
| price 1.00, 1000 lots, margin 500, W1 control | 481,576,883 | 481,576,883 | 0 | -481,577,008 | +441,577,008 | +441,135,863 | +441,145 | +40,000,000 |
| same, W3 free settlement withdrawn | 481,576,883 | 410,099,676 | 71,477,207 | -410,099,801 | +370,099,801 | +369,730,062 | +369,739 | +40,000,000 |
| W3 - W1 | | | | +71,477,207 | -71,477,207 | -71,405,801 | -71,406 | 0 |
| price 1.90, depleted depth, 10 lots, margin 200, W1 control | 359,974,025 | 359,974,025 | 0 | -359,975,623 | +359,215,623 | +340,988,342 | +18,227,281 | +760,000 |
| same, W3 free settlement withdrawn | 359,974,025 | 207,421,662 | 152,552,363 | -207,423,260 | +206,663,260 | +196,176,770 | +10,486,490 | +760,000 |
| W3 - W1 | | | | +152,552,363 | -152,552,363 | -144,811,572 | -7,740,791 | 0 |

- **Who gains and who loses.** The trader keeps exactly the waived amount. The pool, and through it the Junior and Senior LPs, receive that much less. The treasury's execution fee is collected first and does not change. No keeper is involved.
- **When it is non-zero.** The waiver is the charge minus what the pledge and the liquidation reserve can pay. In the first world, 310,099,801 + 100,000,000 = 410,099,801 (pledge plus reserve), and the close recovers 410,099,676 of the charge. So the route pays only for positions whose close charge exceeds their pledge plus reserve: thin-margin positions on the minority side after skew grew or pool depth fell.
- **Timing does not matter.** W4, where the trader withdraws before it commits the close, gives the same trader and pool deltas as W3 (the waiver differs by one atom of rounding).
- **An order bound does not help the LP.** With `maxActionChargeAssets` set to 100 USDG and the free settlement withdrawn, the close is refused (status 3, Failed, reason 8, ConstraintViolation) and the position stays open (`test_Ordinary_P100_BoundedOrderIsRefused`). The bound is a trader-side option and plays no part in the waiver.

**Why Low and not higher.** No deposited value is lost. Pool assets rise on every measured close, and the LPs are never pushed below their starting assets. The amount is revenue the LPs should have received, not principal. "Theft of unclaimed yield" (High) was considered: the waived charge was never credited to the LPs, so it is not yield they held. "Griefing" (Medium) does not fit either: the trader gains, and the LP's loss is limited to the waived charge.

## Limits

- **Not profitable on its own.** A full open and close cycle with two accounts of one owner loses money in every measured size, because both legs pay fees and the lifetime VPI clamp (`packages/perps/src/libraries/CloseAccountingLib.sol:112-118`) caps a close rebate at the VPI paid at open. The withdrawal only makes the loss smaller, by the waived amount (`test_TwoAccountCycle_NotProfitable`, amounts in atoms):

| A lots | A margin | waived | group result, withdraw-first | group result, control | pool assets, withdraw-first | pool assets, control |
|---|---|---|---|---|---|---|
| 1000 | 500,000,000 | 71,477,207 | -894,242,989 | -965,720,196 | +617,122,989 | +688,600,196 |
| 2000 | 1,000,000,000 | 342,478,920 | -1,474,244,344 | -1,816,723,263 | +1,077,124,344 | +1,419,603,263 |
| 3000 | 1,600,000,000 | 713,303,618 | -2,154,325,952 | -2,867,629,569 | +1,637,165,952 | +2,350,469,569 |

  The group difference equals the waiver within one atom of rounding (71,477,207, 342,478,919 and 713,303,617).
- **Pool assets still rise.** The LPs receive less than the charge, but every measured close still adds to pool assets.
- **Not in degraded mode.** `validateWithdraw` reverts `CfdEngine__DegradedMode` for an account with a position (`packages/perps/src/CfdEngineSettlementSidecar.sol:274-276`).
- **Only thin-margin positions.** The waiver is non-zero only when the close charge exceeds pledge plus liquidation reserve.
- **The waiver mechanism is documented.** `packages/perps/ACCOUNTING_SPEC.md:1187` and `packages/perps/WHITEPAPER.md:341` say an uncollectible terminal action charge or remainder is waived, so that a full close can always complete. The report does not dispute the waiver. It disputes that the trader can make the charge uncollectible by moving eligible settlement out first, which the same specification forbids for the parking route (`packages/perps/ACCOUNTING_SPEC.md:1067`). No sentence in the documents names the withdrawal route, either to allow it or to forbid it.
- **The fixture's `MockPyth` stands in for Pyth and its token for USDG.** Both are out-of-scope components.

## References

- `packages/perps/ACCOUNTING_SPEC.md:1035-1036`: on a full close, the unused pledge and liquidation reserve are released into free settlement "before collecting action charges", then the close must "waive only the shortfall after all eligible funds are exhausted".
- `packages/perps/ACCOUNTING_SPEC.md:1042-1045`: full closes collect action charges from spendable action reserve, free settlement, safely released pledge and committed order margin, and "A waiver at the engine settlement boundary implies no eligible cash remains".
- `packages/perps/ACCOUNTING_SPEC.md:1067`: "a user must not shield otherwise reachable settlement by parking it in queued committed margin right before terminal settlement".
- `packages/perps/ACCOUNTING_SPEC.md:1187` and `packages/perps/WHITEPAPER.md:341`: an uncollectible terminal action charge is waived. These sentences describe the waiver. They do not say whether a trader may create the shortfall.
- Code: `packages/perps/src/libraries/CfdEnginePlanLib.sol:1093-1133`, `packages/perps/src/MarginClearinghouse.sol:441-465`, `packages/perps/src/CfdEngineSettlementSidecar.sol:255-305`, `packages/perps/src/libraries/CloseCommitmentLib.sol:13-55`.

## Proof of Concept

## Proof of Concept

### Steps to reproduce

1. Set the release risk parameters through the timelocked owner path and fund the actors.
2. The trader opens 1000 lots long at price 1.00 with 500 USDG margin. bob shorts to 98 % of the skew cap, so the trader's close raises skew.
3. Snapshot. W1: the trader commits a full close and it executes.
4. Revert to the snapshot. W3: the trader commits the full close, calls `clearinghouse.withdraw(trader, freeSettlement)`, and the close executes. W4: the same withdrawal before the commit.
5. Compare the `ActionChargeSettled` event (assessed, recovered, waived) and the balance changes of the trader, the pool, the LPs and the treasury.
6. Repeat in a depleted-depth world (price 1.90, bob redeems Junior through ordinary LP epochs).
7. Run a full cycle with two accounts of one owner (A long, B short, A withdraws and closes, B closes) to show the cycle is not profitable.

### Proof of concept

Save the file below as `packages/perps/test/perps/CloseChargeWaiver.t.sol` and run, from `packages/perps`:

```sh
forge test --match-path 'test/perps/CloseChargeWaiver.t.sol' -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";

import {console2} from "forge-std/console2.sol";

import {OrderRouterPythTestBase} from "./shared/OrderRouterTestBase.sol";

/// A terminal full close collects its assessed action charge from the account's buckets in a fixed order
/// (CfdEnginePlanLib.sol:1093-1133) and waives whatever is left. Nothing ties the account's free settlement to that
/// charge: the trader withdraws its free settlement through MarginClearinghouse.withdraw (guard
/// CfdEngineSettlementSidecar.sol:287-305 checks health, unsettled carry and the VPI reserve only) and the close then
/// waives the part the free settlement would have paid. The control is the same close without the withdrawal.
/// Public entrypoints only. Tokens are minted only to fund actors. All amounts are in USDG atoms (6 decimals).
contract CloseChargeWaiverTest is OrderRouterPythTestBase {
    address internal keeper = address(0x4EE9);
    address internal trader = address(0x71C7); // the account that closes and withdraws first

    uint256 internal constant T0 = SETUP_TIMESTAMP + 10 hours;
    uint256 internal constant LOT = 100e18;
    uint256 internal constant MAX_EPOCHS = 8;
    bytes32 internal constant ACTION_CHARGE_SETTLED = keccak256("ActionChargeSettled(address,uint256,uint256,uint256)");

    uint256 internal now_;
    int64 internal px;

    // ------------------------------------------------------------------ fixture helpers
    /// Release parameters of packages/perps/DEPLOYMENT.md (0.1 % maintenance, 0.2 % initial, 3 % frozen margin,
    /// 0.04 % execution fee, 0.50 % frozen-close spread, Senior cap 40M and 80 %), set through the timelocked owner path.
    function _release() internal {
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8_000;
        pc.seniorRateBps = 800;
        pool.proposePoolConfig(pc);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        ICfdEngineAdminHost.EngineRiskConfig memory config;
        config.riskParams = CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
        config.executionFeeBps = 4;
        config.frozenCloseSpreadBps = 50;
        config.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(config);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
    }

    function _blob(int64 price, uint256 t, uint256 tPrev) internal pure returns (bytes[] memory d) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(FEED_A, MockPyth.MockPrice(price, 0, int32(-8), t, tPrev));
        u[1] = MockPyth.FeedUpdate(FEED_B, MockPyth.MockPrice(price, 0, int32(-8), t, tPrev));
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _tick(uint256 dt) internal {
        now_ += dt;
        vm.warp(now_);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _mark() internal {
        _tick(2);
        bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
        vm.prank(keeper);
        router.updateMarkPrice(b);
    }

    function _fund(address who, uint256 amt) internal {
        settlementToken.mint(who, amt);
        vm.startPrank(who);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(who, amt);
        vm.stopPrank();
    }

    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 maxCharge)
        internal
        view
        returns (OrderV3Types.OrderRequest memory req)
    {
        req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(now_, size, margin, isClose, gasleft())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: maxCharge,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    function _commit(address a, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 maxCharge)
        internal
        returns (uint64 id)
    {
        if (engine.lastMarkTime() + 40 < now_) _mark();
        _tick(1);
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose, maxCharge);
        vm.prank(a);
        id = router.commitOrder(req);
    }

    function _exec(address a, uint64 id) internal returns (OrderV3Types.ExecutionResult memory r) {
        _tick(2);
        bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
        vm.prank(a);
        r = router.executeOrder(id, b);
    }

    function _must(address a, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        uint64 id = _commit(a, side, size, margin, isClose, type(uint256).max);
        OrderV3Types.ExecutionResult memory r = _exec(a, id);
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "trade not executed");
    }

    function _size(address a) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _cash(address a) internal view returns (int256) {
        return int256(settlementToken.balanceOf(a) + clearinghouse.balanceAssets(a));
    }

    function _depth() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    function _treasury() internal view returns (int256) {
        address t = engine.protocolTreasury();
        return int256(settlementToken.balanceOf(t) + clearinghouse.balanceAssets(t));
    }

    /// bob is the fixture's Junior LP (1,000,000 USDG). In these worlds he also holds the short book, which stays
    /// open and unchanged through the trader's close, so his change across the close is his Junior share value.
    function _bobTotal() internal view returns (int256) {
        return _cash(bob) + int256(juniorVault.estimateRedeemAssets(juniorVault.balanceOf(bob)));
    }

    /// The fixture's seed LP (this test contract) holds Junior and Senior shares.
    function _seedTotal() internal view returns (int256) {
        return int256(juniorVault.estimateRedeemAssets(juniorVault.balanceOf(address(this))))
            + int256(seniorVault.estimateRedeemAssets(seniorVault.balanceOf(address(this))));
    }

    function _free(address a) internal view returns (uint256) {
        return clearinghouse.getAccountAssetBuckets(a).freeSettlementAssets;
    }

    /// bob redeems Junior through ordinary LP epochs until the pool cap binds (lowers pool depth).
    function _epochs() internal {
        for (uint256 i; i < MAX_EPOCHS; ++i) {
            _mark();
            (,,, uint256 maxJ) = pool.getPendingTrancheState();
            uint256 jp = pool.juniorPrincipal();
            uint256 bal = juniorVault.balanceOf(bob);
            uint256 shares = maxJ >= jp ? bal : bal * maxJ / jp * 9_999 / 10_000;
            uint256 mx = juniorVault.maxRequestRedeem(bob);
            if (shares > mx) shares = mx;
            if (shares == 0 || maxJ < 10e6) break;
            vm.prank(bob);
            uint256 id = juniorVault.requestRedeem(shares, bob, bob);
            now_ = pool.lpEpochStart(id) + 30;
            vm.warp(now_);
            _tick(1);
            bytes[] memory b = _blob(px, now_ - 1, now_ - 2);
            vm.prank(keeper);
            router.settleLpEpoch(b);
            _mark();
            uint256 c = juniorVault.claimableRedeemRequest(id, bob);
            if (c > 0) {
                vm.prank(bob);
                juniorVault.claimRedeem(id, c, bob, bob);
            }
        }
        require(!engine.degradedMode(), "degraded");
    }

    // ------------------------------------------------------------------ one close, measured
    struct Res {
        bool executed;
        uint256 withdrawn;
        uint256 freeAtExec;
        uint256 assessed;
        uint256 recovered;
        uint256 waived;
        int256 trader; // wallet + clearinghouse: the withdrawn amount stays in the trader's wallet and is counted
        int256 pool; // pool.totalAssets()
        int256 lpBob; // bob's Junior shares + cash
        int256 lpSeed; // seed LP's Junior + Senior shares
        int256 treasury;
        int256 keeperW;
    }

    function _snapRes() internal view returns (Res memory b) {
        b.trader = _cash(trader);
        b.pool = int256(pool.totalAssets());
        b.lpBob = _bobTotal();
        b.lpSeed = _seedTotal();
        b.treasury = _treasury();
        b.keeperW = _cash(keeper);
    }

    function _parseCharge(Res memory r) internal {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length > 0 && logs[i].topics[0] == ACTION_CHARGE_SETTLED) {
                (r.assessed, r.recovered, r.waived) = abi.decode(logs[i].data, (uint256, uint256, uint256));
            }
        }
    }

    /// The trader's full close. `withdrawAmt` of free settlement is withdrawn through MarginClearinghouse.withdraw,
    /// either right before the close commit (`beforeCommit`) or between commit and execution. Deltas run from just
    /// before the close (and before any withdrawal) to just after its execution.
    function _closeWorld(uint256 withdrawAmt, bool beforeCommit, uint256 maxCharge) internal returns (Res memory r) {
        Res memory b = _snapRes();
        _mark(); // a withdrawal with an open position needs a fresh mark
        if (beforeCommit && withdrawAmt > 0) {
            vm.prank(trader);
            clearinghouse.withdraw(trader, withdrawAmt);
        }
        uint64 id = _commit(trader, CfdTypes.Side.LONG, _size(trader), 0, true, maxCharge);
        if (!beforeCommit && withdrawAmt > 0) {
            vm.prank(trader);
            clearinghouse.withdraw(trader, withdrawAmt);
        }
        r.withdrawn = withdrawAmt;
        r.freeAtExec = _free(trader);
        vm.recordLogs();
        OrderV3Types.ExecutionResult memory x = _exec(trader, id);
        _parseCharge(r);
        r.executed = x.status == OrderV3Types.LifecycleStatus.Executed;
        if (!r.executed) {
            console2.log("  close status (3 = Failed)", uint256(uint8(x.status)));
            console2.log("  close terminal reason (8 = ConstraintViolation)", uint256(uint8(x.terminalReason)));
        }
        Res memory a = _snapRes();
        r.trader = a.trader - b.trader;
        r.pool = a.pool - b.pool;
        r.lpBob = a.lpBob - b.lpBob;
        r.lpSeed = a.lpSeed - b.lpSeed;
        r.treasury = a.treasury - b.treasury;
        r.keeperW = a.keeperW - b.keeperW;
    }

    function _log(string memory tag, Res memory r) internal pure {
        console2.log(tag);
        console2.log("  close executed", r.executed);
        console2.log("  free settlement withdrawn by the trader", r.withdrawn);
        console2.log("  free settlement left at execution", r.freeAtExec);
        console2.log("  action charge assessed", r.assessed);
        console2.log("  action charge recovered", r.recovered);
        console2.log("  action charge waived", r.waived);
        console2.log("  delta trader", r.trader);
        console2.log("  delta pool assets", r.pool);
        console2.log("  delta LP bob", r.lpBob);
        console2.log("  delta seed LP", r.lpSeed);
        console2.log("  delta treasury", r.treasury);
        console2.log("  delta keeper", r.keeperW);
    }

    function _diff(string memory tag, Res memory a, Res memory c) internal pure {
        console2.log(tag);
        console2.log("  trader", a.trader - c.trader);
        console2.log("  pool assets", a.pool - c.pool);
        console2.log("  LP bob", a.lpBob - c.lpBob);
        console2.log("  seed LP", a.lpSeed - c.lpSeed);
        console2.log("  treasury", a.treasury - c.treasury);
    }

    /// Ordinary market at price p: the trader opens a long first (book flat), then bob's short book takes the skew to
    /// 98 % of the cap, so the trader is the minority side and his full close raises skew. No LP redemption, no
    /// claims, not degraded.
    function _buildOrdinary(int64 p, uint256 traderLots, uint256 traderMargin, uint256 traderFunds) internal {
        _release();
        now_ = T0;
        vm.warp(now_);
        px = p;
        _mark();
        _fund(bob, 400_000e6);
        _fund(trader, traderFunds);
        _must(trader, CfdTypes.Side.LONG, traderLots * LOT, traderMargin, false);
        uint256 lots = traderLots + (_depth() * 4 / 10) * 98 / 100 / uint256(uint64(px));
        _must(bob, CfdTypes.Side.SHORT, lots * LOT, 60_000e6, false);
        console2.log("  pool depth", _depth());
        console2.log("  trader lots / bob short lots", traderLots, lots);
        console2.log("  trader pnl pledge", clearinghouse.pnlPledgeAssets(trader));
        console2.log("  trader liquidation reserve", clearinghouse.liquidationReserveAssets(trader));
        console2.log("  trader free settlement", _free(trader));
    }

    /// W1 control (no withdrawal), W2 free settlement left exactly at the W3 waiver, W2b one atom less,
    /// W3 all free settlement withdrawn between commit and execution, W4 all of it withdrawn before the commit.
    function _worlds(string memory tag) internal {
        uint256 s = vm.snapshotState();
        uint256 free0 = _free(trader);
        Res memory w1 = _closeWorld(0, false, type(uint256).max);
        vm.revertToState(s);
        Res memory w3 = _closeWorld(free0, false, type(uint256).max);
        vm.revertToState(s);
        uint256 keep = w3.waived;
        Res memory w2 = _closeWorld(free0 > keep ? free0 - keep : 0, false, type(uint256).max);
        vm.revertToState(s);
        Res memory w2b = _closeWorld(keep > 0 && free0 >= keep ? free0 - keep + 1 : free0, false, type(uint256).max);
        vm.revertToState(s);
        Res memory w4 = _closeWorld(free0, true, type(uint256).max);
        vm.revertToState(s);
        console2.log(string.concat("==================== ", tag));
        _log("W1 control: free settlement kept", w1);
        _log("W2 free settlement left = W3 waiver", w2);
        _log("W2b free settlement left = W3 waiver - 1 atom", w2b);
        _log("W3 all free settlement withdrawn between commit and execution", w3);
        _log("W4 all free settlement withdrawn before the commit", w4);
        _diff("W3 - W1 (withdraw first minus control)", w3, w1);
        _diff("W4 - W1 (withdraw before commit minus control)", w4, w1);
        assertEq(w1.waived, 0, "control waives nothing");
        assertEq(w2.waived, 0, "boundary waives nothing");
        assertEq(w2b.waived, 1, "one atom below the boundary waives one atom");
        assertGt(w3.waived, 0, "withdraw first creates a waiver");
        assertEq(w3.trader - w1.trader, int256(w3.waived), "trader keeps the waived amount");
        assertEq(w1.pool - w3.pool, int256(w3.waived), "pool loses the waived amount");
        assertEq(w3.treasury, w1.treasury, "treasury unchanged");
        assertApproxEqAbs(w4.waived, w3.waived, 1, "same waiver (1 atom of rounding) when the withdrawal precedes the commit");
    }

    // ------------------------------------------------------------------ tests
    /// Ordinary market, price 1.00, trader 1000 lots (100,000 USDG notional) at 500 USDG margin, 2,000 USDG funds.
    function test_Ordinary_P100_WithdrawFirstWaivesCloseCharge() public {
        _buildOrdinary(100_000_000, 1000, 500e6, 2_000e6);
        _worlds("ordinary market, price 1.00, 1000 lots, margin 500");
    }

    /// Order-level bound: maxActionChargeAssets = 100 USDG with all free settlement withdrawn first.
    function test_Ordinary_P100_BoundedOrderIsRefused() public {
        _buildOrdinary(100_000_000, 1000, 500e6, 2_000e6);
        uint256 free0 = _free(trader);
        Res memory w = _closeWorld(free0, false, 100e6);
        _log("BOUNDED close (max action charge 100 USDG), all free settlement withdrawn", w);
        console2.log("  trader lots still open", _size(trader) / LOT);
        assertFalse(w.executed, "bounded close refused");
    }

    /// Depleted-depth world: bob shorts first, the trader opens 10 lots at 200 USDG margin with 1,000 USDG funds at
    /// price 1.90, then bob redeems Junior through ordinary LP epochs until the cap binds.
    function test_DepletedDepth_P190_WithdrawFirstWaivesCloseCharge() public {
        _release();
        now_ = T0;
        vm.warp(now_);
        px = 190_000_000;
        _mark();
        _fund(bob, 300_000e6);
        _fund(trader, 1_000e6);
        uint256 lots = (_depth() * 4 / 10) * 98 / 100 / uint256(uint64(px));
        _must(bob, CfdTypes.Side.SHORT, lots * LOT, 40_000e6, false);
        _must(trader, CfdTypes.Side.LONG, 10 * LOT, 200e6, false);
        _epochs();
        console2.log("  pool depth after LP redemptions", _depth());
        console2.log("  trader pnl pledge", clearinghouse.pnlPledgeAssets(trader));
        console2.log("  trader liquidation reserve", clearinghouse.liquidationReserveAssets(trader));
        console2.log("  trader free settlement", _free(trader));
        _worlds("depleted depth, price 1.90, 10 lots, margin 200");
    }

    // ------------------------------------------------------------------ one owner, two accounts, full cycle
    address internal atkA = address(0xA11CE); // long first, thin margin
    address internal atkB = address(0xB0B0B); // short book that makes A the minority

    /// One owner with two accounts on a flat book at price 1.00: A opens a long of `aLots`, B shorts to 98 % of the
    /// skew cap, A closes (with or without withdrawing its free settlement first), B closes. The book is flat again.
    function _cycle(uint256 aLots, uint256 aMargin, bool withdrawFirst)
        internal
        returns (int256 group, int256 poolD, int256 treas, uint256 waived)
    {
        _release();
        now_ = T0;
        vm.warp(now_);
        px = 100_000_000;
        _mark();
        _fund(atkA, 20_000e6);
        _fund(atkB, 100_000e6);
        int256 g0 = _cash(atkA) + _cash(atkB);
        int256 p0 = int256(pool.totalAssets());
        int256 t0 = _treasury();
        _must(atkA, CfdTypes.Side.LONG, aLots * LOT, aMargin, false);
        uint256 bLots = aLots + (_depth() * 4 / 10) * 98 / 100 / uint256(uint64(px));
        _must(atkB, CfdTypes.Side.SHORT, bLots * LOT, 8_000e6, false);
        _mark();
        uint64 id = _commit(atkA, CfdTypes.Side.LONG, _size(atkA), 0, true, type(uint256).max);
        if (withdrawFirst) {
            uint256 f = _free(atkA);
            vm.prank(atkA);
            clearinghouse.withdraw(atkA, f);
        }
        vm.recordLogs();
        OrderV3Types.ExecutionResult memory x = _exec(atkA, id);
        Res memory r;
        _parseCharge(r);
        require(x.status == OrderV3Types.LifecycleStatus.Executed, "A close not executed");
        _must(atkB, CfdTypes.Side.SHORT, _size(atkB), 0, true);
        require(_size(atkA) == 0 && _size(atkB) == 0, "book not flat");
        group = _cash(atkA) + _cash(atkB) - g0;
        poolD = int256(pool.totalAssets()) - p0;
        treas = _treasury() - t0;
        waived = r.waived;
        console2.log(withdrawFirst ? "  withdraw-first: A close assessed / recovered / waived" : "  control: A close assessed / recovered / waived", r.assessed, r.recovered, r.waived);
    }

    function test_TwoAccountCycle_NotProfitable() public {
        uint256[3] memory lotsA = [uint256(1000), 2000, 3000];
        uint256[3] memory marginA = [uint256(500e6), 1_000e6, 1_600e6];
        for (uint256 i; i < 3; ++i) {
            uint256 s = vm.snapshotState();
            (int256 gW, int256 pW, int256 tW, uint256 w) = _cycle(lotsA[i], marginA[i], true);
            vm.revertToState(s);
            (int256 gC, int256 pC, int256 tC,) = _cycle(lotsA[i], marginA[i], false);
            vm.revertToState(s);
            console2.log("A lots / A margin / waived", lotsA[i], marginA[i], w);
            console2.log("  group result, withdraw-first", gW);
            console2.log("  group result, control", gC);
            console2.log("  group difference (withdraw-first - control)", gW - gC);
            console2.log("  pool assets, withdraw-first", pW);
            console2.log("  pool assets, control", pC);
            console2.log("  treasury, withdraw-first", tW);
            console2.log("  treasury, control", tC);
            assertLt(gW, 0, "cycle loses money");
            assertGt(pW, 0, "pool assets still rise");
            assertApproxEqAbs(gW - gC, int256(w), 1, "group keeps the waiver (1 atom of rounding)");
            assertEq(tW, tC, "treasury unchanged");
        }
    }
}
```

### Output

Forge output of the command above (`poc-output.txt`):

```
$ forge test --match-path test/perps/CloseChargeWaiver.t.sol -vv

Ran 4 tests for test/perps/CloseChargeWaiver.t.sol:CloseChargeWaiverTest
[PASS] test_DepletedDepth_P190_WithdrawFirstWaivesCloseCharge() (gas: 39845160)
Logs:
    pool depth after LP redemptions 20722518922
    trader pnl pledge 198100000
    trader liquidation reserve 1900000
    trader free settlement 799240000
  ==================== depleted depth, price 1.90, 10 lots, margin 200
  W1 control: free settlement kept
    close executed true
    free settlement withdrawn by the trader 0
    free settlement left at execution 799240000
    action charge assessed 359974025
    action charge recovered 359974025
    action charge waived 0
    delta trader -359975623
    delta pool assets 359215623
    delta LP bob 340988342
    delta seed LP 18227281
    delta treasury 760000
    delta keeper 0
  W2 free settlement left = W3 waiver
    close executed true
    free settlement withdrawn by the trader 646687637
    free settlement left at execution 152552363
    action charge assessed 359974025
    action charge recovered 359974025
    action charge waived 0
    delta trader -359975623
    delta pool assets 359215623
    delta LP bob 340988342
    delta seed LP 18227281
    delta treasury 760000
    delta keeper 0
  W2b free settlement left = W3 waiver - 1 atom
    close executed true
    free settlement withdrawn by the trader 646687638
    free settlement left at execution 152552362
    action charge assessed 359974025
    action charge recovered 359974024
    action charge waived 1
    delta trader -359975622
    delta pool assets 359215622
    delta LP bob 340988341
    delta seed LP 18227281
    delta treasury 760000
    delta keeper 0
  W3 all free settlement withdrawn between commit and execution
    close executed true
    free settlement withdrawn by the trader 799240000
    free settlement left at execution 0
    action charge assessed 359974025
    action charge recovered 207421662
    action charge waived 152552363
    delta trader -207423260
    delta pool assets 206663260
    delta LP bob 196176770
    delta seed LP 10486490
    delta treasury 760000
    delta keeper 0
  W4 all free settlement withdrawn before the commit
    close executed true
    free settlement withdrawn by the trader 799240000
    free settlement left at execution 0
    action charge assessed 359974025
    action charge recovered 207421662
    action charge waived 152552363
    delta trader -207423260
    delta pool assets 206663260
    delta LP bob 196176770
    delta seed LP 10486490
    delta treasury 760000
    delta keeper 0
  W3 - W1 (withdraw first minus control)
    trader 152552363
    pool assets -152552363
    LP bob -144811572
    seed LP -7740791
    treasury 0
  W4 - W1 (withdraw before commit minus control)
    trader 152552363
    pool assets -152552363
    LP bob -144811572
    seed LP -7740791
    treasury 0

[PASS] test_Ordinary_P100_BoundedOrderIsRefused() (gas: 12994490)
Logs:
    pool depth 1002769883505
    trader lots / bob short lots 1000 4928
    trader pnl pledge 310099801
    trader liquidation reserve 100000000
    trader free settlement 1500000000
    close status (3 = Failed) 3
    close terminal reason (8 = ConstraintViolation) 8
  BOUNDED close (max action charge 100 USDG), all free settlement withdrawn
    close executed false
    free settlement withdrawn by the trader 1500000000
    free settlement left at execution 0
    action charge assessed 0
    action charge recovered 0
    action charge waived 0
    delta trader -94
    delta pool assets 94
    delta LP bob 81
    delta seed LP 13
    delta treasury 0
    delta keeper 0
    trader lots still open 1000

[PASS] test_Ordinary_P100_WithdrawFirstWaivesCloseCharge() (gas: 33766185)
Logs:
    pool depth 1002769883505
    trader lots / bob short lots 1000 4928
    trader pnl pledge 310099801
    trader liquidation reserve 100000000
    trader free settlement 1500000000
  ==================== ordinary market, price 1.00, 1000 lots, margin 500
  W1 control: free settlement kept
    close executed true
    free settlement withdrawn by the trader 0
    free settlement left at execution 1500000000
    action charge assessed 481576883
    action charge recovered 481576883
    action charge waived 0
    delta trader -481577008
    delta pool assets 441577008
    delta LP bob 441135863
    delta seed LP 441145
    delta treasury 40000000
    delta keeper 0
  W2 free settlement left = W3 waiver
    close executed true
    free settlement withdrawn by the trader 1428522793
    free settlement left at execution 71477207
    action charge assessed 481576883
    action charge recovered 481576883
    action charge waived 0
    delta trader -481577008
    delta pool assets 441577008
    delta LP bob 441135863
    delta seed LP 441145
    delta treasury 40000000
    delta keeper 0
  W2b free settlement left = W3 waiver - 1 atom
    close executed true
    free settlement withdrawn by the trader 1428522794
    free settlement left at execution 71477206
    action charge assessed 481576883
    action charge recovered 481576882
    action charge waived 1
    delta trader -481577007
    delta pool assets 441577007
    delta LP bob 441135862
    delta seed LP 441145
    delta treasury 40000000
    delta keeper 0
  W3 all free settlement withdrawn between commit and execution
    close executed true
    free settlement withdrawn by the trader 1500000000
    free settlement left at execution 0
    action charge assessed 481576883
    action charge recovered 410099676
    action charge waived 71477207
    delta trader -410099801
    delta pool assets 370099801
    delta LP bob 369730062
    delta seed LP 369739
    delta treasury 40000000
    delta keeper 0
  W4 all free settlement withdrawn before the commit
    close executed true
    free settlement withdrawn by the trader 1500000000
    free settlement left at execution 0
    action charge assessed 481576883
    action charge recovered 410099677
    action charge waived 71477206
    delta trader -410099801
    delta pool assets 370099801
    delta LP bob 369730062
    delta seed LP 369739
    delta treasury 40000000
    delta keeper 0
  W3 - W1 (withdraw first minus control)
    trader 71477207
    pool assets -71477207
    LP bob -71405801
    seed LP -71406
    treasury 0
  W4 - W1 (withdraw before commit minus control)
    trader 71477207
    pool assets -71477207
    LP bob -71405801
    seed LP -71406
    treasury 0

[PASS] test_TwoAccountCycle_NotProfitable() (gas: 82972940)
Logs:
    withdraw-first: A close assessed / recovered / waived 481576883 410099676 71477207
    control: A close assessed / recovered / waived 481576883 481576883 0
  A lots / A margin / waived 1000 500000000 71477207
    group result, withdraw-first -894242989
    group result, control -965720196
    group difference (withdraw-first - control) 71477207
    pool assets, withdraw-first 617122989
    pool assets, control 688600196
    treasury, withdraw-first 277120000
    treasury, control 277120000
    withdraw-first: A close assessed / recovered / waived 1062877620 720398700 342478920
    control: A close assessed / recovered / waived 1062877620 1062877620 0
  A lots / A margin / waived 2000 1000000000 342478920
    group result, withdraw-first -1474244344
    group result, control -1816723263
    group difference (withdraw-first - control) 342478919
    pool assets, withdraw-first 1077124344
    pool assets, control 1419603263
    treasury, withdraw-first 397120000
    treasury, control 397120000
    withdraw-first: A close assessed / recovered / waived 1744200690 1030897072 713303618
    control: A close assessed / recovered / waived 1744200690 1744200690 0
  A lots / A margin / waived 3000 1600000000 713303618
    group result, withdraw-first -2154325952
    group result, control -2867629569
    group difference (withdraw-first - control) 713303617
    pool assets, withdraw-first 1637165952
    pool assets, control 2350469569
    treasury, withdraw-first 517160000
    treasury, control 517160000

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 126.40ms (225.36ms CPU time)

Ran 1 test suite in 128.52ms (126.40ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

## Recommendation

Make the free settlement that a full close would consume unavailable to the trader while a position is open. Two logic fixes:

1. In `_requireWithdrawPositionHealthy` (`packages/perps/src/CfdEngineSettlementSidecar.sol:287-305`), compute the action charge a full close of the position would assess at the current mark (the planner already computes it), subtract what the pledge, liquidation reserve and spendable action reserve can pay, and revert if the withdrawal leaves free settlement below that remainder.
2. At least while a close order is pending, lock the close's assessed charge from free settlement at commit, the same way `reserveCommittedOrderMargin` locks margin for an open (`packages/perps/src/MarginClearinghouse.sol:620-655`), and release the lock at execution or cancellation.

Only the first fix also covers a withdrawal made before the commit.

