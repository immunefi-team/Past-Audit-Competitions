# \#99129 [SC-Low] Open-order bounty reservation collects carry beyond signed debit ceilings and authenticates a zero-carry commitment

**Submitted on Nov 12th 2026 at 01:05:40 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99129
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Every V3 order contains mandatory financial limits. Their NatSpec says maximum values are inclusive, zero is a real zero allowance, and `maxGrossAccountDebitAssets` covers settlement debit and the reserved bounty. `SECURITY.md` separately promises that every fresh order pins these inclusive limits.

An account with an existing position can nevertheless submit a same-side increase that authorizes only the `0.200000` USDG opening bounty as its maximum gross debit and authorizes exactly zero action charge. Committing that order succeeds after taking `22.288291` USDG of accrued carry from the account and transferring it to HousePool. The actual gross debit is therefore `22.488291` USDG.

The bound check misses the charge because the open-order reservation path returns a commitment structure containing the bounty but zero carry and zero before/after settlement balances. `OrderLifecycleBook` validates and authenticates that incomplete structure. Execution later imports the authenticated zero, so it cannot reconstruct the earlier debit.

This is not a claim that the protocol invented a false fee. The position genuinely owed the accrued carry, and paying it reduces the corresponding liability. The conservative impact is failure of the order's promised financial-control surface and false authenticated accounting, without demonstrated net-value loss.

## Vulnerability Details

### Open bounty reservation checkpoints carry as a hidden side effect

`OrderCommitHandler._createPendingOrder` reserves the execution bounty before it records the commitment:

```solidity
CfdEnginePlanTypes.CloseCommitment memory effects =
    _reserveExecutionBounty(account, order.sizeDelta, executionBountyAssets, order.isClose);
clearinghouse.recordBountyReservation(
    account, IMarginClearinghouse.BountyKind.Order, orderId, executionBountyAssets
);
_onOrderCommitted(order, effects);
```

For an open or increase, `_reserveExecutionBounty` initializes only `bountyFromFreeAssets` and then locks the bounty:

```solidity
effects.bountyFromFreeAssets = bounty;
if (bounty == 0) {
    return effects;
}
if (clearinghouse.getAccountAssetBuckets(account).freeSettlementAssets < bounty) {
    revert OrderRouter__InsufficientFreeEquity();
}
clearinghouse.lockReservedSettlement(account, bounty);
```

The returned structure still reports zero `carryCollectedAssets`, `settlementBeforeAssets`, and `settlementAfterAssets`. However, `lockReservedSettlement` explicitly checkpoints carry before changing the reservation bucket:

```solidity
function lockReservedSettlement(address account, uint256 amountAssets)
    external
    onlyReservedSettlementLocker
{
    _checkpointCarryBeforeMarginChange(account);
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
}
```

That checkpoint calls `CfdEngine.realizeCarryBeforeMarginChange`. The Engine consumes accrued carry from account backing, transfers the collected USDG to HousePool, and records it as revenue.

### The lifecycle bound check validates the incomplete structure

The order request defines two relevant ceilings:

- `maxGrossAccountDebitAssets`, which expressly covers settlement debit and bounty; and
- `maxActionChargeAssets`, for which zero is expressly a real zero allowance.

`OrderLifecycleBook.recordCommitment` is intended to enforce both:

```solidity
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
if (gross > pending.bounds.maxGrossAccountDebitAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(...);
}
if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(...);
}
```

Because the open branch returned zero carry, the Book compares only the `0.200000` USDG bounty against the gross limit and compares zero against the zero action-charge limit. It then stores a commitment claiming that no carry was collected.

The close path does not have this mismatch. A close commitment calculates and records its commitment-time carry before `recordCommitment` checks the bounds. The defect is specific to opens and same-side increases whose nonzero bounty passes through `lockReservedSettlement`.

### Execution cannot recover the omitted debit

The later policy assessment adds the authenticated commitment fields:

```solidity
assessment.grossAccountDebitAssets += effects.carryCollectedAssets;
assessment.actionChargeAssessedAssets += effects.carryCollectedAssets;
assessment.actionChargeCollectedAssets += effects.carryCollectedAssets;
assessment.carryAssets += effects.carryCollectedAssets;
```

By execution time, the account's carry checkpoint and custody have already changed. Since the stored effects report zero, the execution assessment cannot infer or enforce the amount taken during commitment.

## End-to-End Result

The PoC uses release risk parameters and the following public sequence:

1. A Junior LP supplies `5,000,000` USDG through the asynchronous vault and public `OrderRouter.settleLpEpoch` path.
2. The trader deposits settlement assets through the public Clearinghouse.
3. The trader commits and a permissionless keeper executes a public V3 `1,000,000` USDG open.
4. One day passes, and a permissionless public mark refresh advances the carry indexes.
5. The trader commits a `100,000` USDG same-side increase with:
   - `maxExecutionBountyAssets = 0.200000` USDG;
   - `maxGrossAccountDebitAssets = 0.200000` USDG; and
   - `maxActionChargeAssets = 0`.
6. The commit succeeds and remains pending.

Observed values:

| Observation | Amount |
| --- | ---: |
| Signed maximum action charge | `0` USDG |
| Signed maximum gross debit | `0.200000` USDG |
| Carry taken during commitment | `22.288291` USDG |
| Actual gross debit | `22.488291` USDG |
| USDG transferred to HousePool | `22.288291` USDG |
| Carry retained in authenticated commitment | `0` USDG |

The matching account-balance decrease and HousePool token increase prove that the discrepancy is not a view-only error.

## Impact Details

The demonstrated defect breaks the protocol's mandatory transaction-level financial authority:

- the account says that no action charge may be taken;
- the account caps total debit at the bounty;
- the protocol takes more than one hundred times that gross ceiling;
- the commit succeeds rather than reverting; and
- the authenticated record permanently says that no carry was taken.

This report does not claim theft. The `22.288291` USDG is legitimate accrued carry that the open position already owed, so realizing it does not create a new economic liability or prove loss of net equity. No unrelated attacker can currently invoke ordinary `commitOrder` for the victim's account.

This is the Low impact: the bounded-order interface fails to deliver its promised zero and maximum charge limits, without loss of net value.

## Why This Is Not a Known Issue

- **MED-9** concerned protocol-generated protection closes lacking the trader's selected financial limits. This report uses an ordinary caller-authored open/increase that contains the correct limits; the implementation silently exceeds them during commitment.
- **LOW-34** concerned a close evaluator reporting assessed carry after commitment-time carry had already been collected. Here an open commitment reports zero collected carry even though the custody transfer actually occurred.
- **LOW-53** concerned open previews overstating committable margin because bounty effects were omitted. It did not report carry collection exceeding `maxGrossAccountDebitAssets` and `maxActionChargeAssets` or an authenticated zero-carry commitment.
- The documented close-commitment path correctly includes carry in the bounded commitment. No checked-in audit or protocol document located during review disclosed the asymmetric open/increase path.

## Recommendation

Make the open reservation path return the actual economic effects of `lockReservedSettlement`. Snapshot settlement custody before the lock, collect the bounty, then populate:

- `carryCollectedAssets`;
- `carryOutstandingAssets`;
- `settlementAfterAssets`; and
- `freeSettlementAfterAssets`.

Pass those effects to `recordCommitment` exactly as the close path does, so the existing gross-debit and action-charge checks revert the complete commit atomically when carry exceeds the request's limits.

Alternatively, move the open carry checkpoint into a bounded Engine admission function that receives the complete V3 `ExecutionBounds`. Add a regression asserting that an increase with positive accrued carry and `maxActionChargeAssets = 0` reverts without changing account custody, carry checkpoints, HousePool cash, or lifecycle state.

## References

- [`OrderCommitHandler._createPendingOrder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L17-L36)
- [`OrderReservationAccounting._reserveExecutionBounty`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L101-L128)
- [`MarginClearinghouse.lockReservedSettlement`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L812-L824)
- [`CfdEngine.realizeCarryBeforeMarginChange`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L531-L550)
- [`CfdEngine._realizeCarryFromSettlement`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1407-L1435)
- [`OrderLifecycleBook.recordCommitment`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L360)
- [`OrderV3Types.ExecutionBounds`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L131-L150)
- [`SECURITY.md` bounded-authority invariant](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L361-L369)

## Proof of Concept

## Proof of Concept

Save the following file as `packages/perps/test/perps/regression/OpenCommitCarryBoundsBypass.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Proves that committing an increase collects accrued carry outside the request's mandatory financial
///         bounds and then authenticates a commitment that reports zero carry.
/// @dev Every protocol action uses a public entry point. Only Pyth and USDG are mocked, as permitted by the contest.
///      No vm.store, in-scope-contract impersonation, legacy Router entrypoint, or forged protocol balance is used.
contract OpenCommitCarryBoundsBypassTest is BasePerpTest {

    address private constant LIQUIDITY_PROVIDER = address(0xCA770001);
    address private constant TRADER = address(0xCA770002);
    address private constant KEEPER = address(0xCA770003);

    uint256 private constant PRICE = 1e8;
    uint256 private constant LP_ASSETS = 5_000_000e6;
    uint256 private constant TRADER_FUNDING = 500_000e6;
    uint256 private constant INITIAL_SIZE = 1_000_000e18;
    uint256 private constant INITIAL_MARGIN = 100_000e6;
    uint256 private constant INCREASE_SIZE = 100_000e18;
    uint256 private constant INCREASE_MARGIN = 10_000e6;

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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_OpenIncreaseCommitCollectsCarryOutsideSignedBoundsAndReceipt() public {
        _activateLiquidityThroughPublicRouter(juniorVault, LIQUIDITY_PROVIDER, LP_ASSETS);
        _fundTrader(TRADER, TRADER_FUNDING);

        uint64 initialOrder = _commitOpen(
            INITIAL_SIZE,
            INITIAL_MARGIN,
            type(uint256).max,
            type(uint256).max,
            type(uint256).max,
            keccak256("initial-public-open")
        );
        _executeThroughPublicRouter(initialOrder);

        vm.warp(block.timestamp + 1 days);
        bytes[] memory markData = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.updateMarkPrice(markData);

        uint256 quotedBounty = _quoteOpenOrderExecutionBountyAssets(INCREASE_SIZE);
        assertGt(quotedBounty, 0, "increase must reserve the release open bounty");

        uint256 traderSettlementBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolCashBefore = settlementToken.balanceOf(address(pool));

        uint64 increaseOrder = _commitOpen(
            INCREASE_SIZE,
            INCREASE_MARGIN,
            quotedBounty,
            quotedBounty,
            0,
            keccak256("bounded-public-increase")
        );

        uint256 traderSettlementAfter = clearinghouse.balanceAssets(TRADER);
        uint256 poolCashAfter = settlementToken.balanceOf(address(pool));
        uint256 hiddenCarryDebit = traderSettlementBefore - traderSettlementAfter;

        assertGt(hiddenCarryDebit, 0, "existing position must pay accrued carry at increase commitment");
        assertEq(poolCashAfter - poolCashBefore, hiddenCarryDebit, "hidden debit is transferred to HousePool");
        assertGt(hiddenCarryDebit + quotedBounty, quotedBounty, "actual gross debit exceeds maxGrossAccountDebitAssets");
        assertGt(hiddenCarryDebit, 0, "actual action charge exceeds maxActionChargeAssets=0");

        OrderV3Types.PendingIntent memory pending = router.lifecycleBook().pendingIntent(increaseOrder);
        assertEq(pending.account, TRADER);
        assertEq(pending.executionBountyAssets, quotedBounty);
        assertEq(pending.bounds.maxGrossAccountDebitAssets, quotedBounty);
        assertEq(pending.bounds.maxActionChargeAssets, 0);
        assertEq(pending.commitment.carryCollectedAssets, 0, "authenticated commitment omits collected carry");
        assertEq(pending.commitment.settlementBeforeAssets, 0, "receipt omits the real pre-debit balance");
        assertEq(pending.commitment.settlementAfterAssets, 0, "receipt omits the real post-debit balance");

        console2.log("carry debited beyond signed bounds", hiddenCarryDebit);
        console2.log("authorized gross debit", quotedBounty);
        console2.log("actual gross debit", hiddenCarryDebit + quotedBounty);
        console2.log("HousePool USDG received", poolCashAfter - poolCashBefore);
    }

    function _activateLiquidityThroughPublicRouter(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) private {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();

        uint256 maturity = vault.depositEpochStart(requestId);
        if (block.timestamp < maturity) {
            vm.warp(maturity);
        }
        bytes[] memory updateData = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(updateData);

        uint256 claimableAssets = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, assets);
        vm.prank(lp);
        uint256 shares = vault.claimDeposit(requestId, claimableAssets, lp, lp);
        assertGt(shares, 0);
    }

    function _commitOpen(
        uint256 size,
        uint256 margin,
        uint256 maxExecutionBounty,
        uint256 maxGrossDebit,
        uint256 maxActionCharge,
        bytes32 clientOrderId
    ) private returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = PRICE;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: maxExecutionBounty,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: maxGrossDebit,
            maxActionChargeAssets: maxActionCharge,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });

        vm.prank(TRADER);
        orderId = router.commitOrder(request);
    }

    function _executeThroughPublicRouter(
        uint64 orderId
    ) private {
        bytes[] memory updateData = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(result.orderId, orderId);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

}
```

Run:

```sh
forge test --root packages/perps \
  --match-path test/perps/regression/OpenCommitCarryBoundsBypass.t.sol -vv
```

Observed result:

```text
Ran 1 test for test/perps/regression/OpenCommitCarryBoundsBypass.t.sol:OpenCommitCarryBoundsBypassTest
[PASS] test_OpenIncreaseCommitCollectsCarryOutsideSignedBoundsAndReceipt()
Logs:
  carry debited beyond signed bounds 22288291
  authorized gross debit 200000
  actual gross debit 22488291
  HousePool USDG received 22288291

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Logged amounts are USDG atoms with six decimals.
