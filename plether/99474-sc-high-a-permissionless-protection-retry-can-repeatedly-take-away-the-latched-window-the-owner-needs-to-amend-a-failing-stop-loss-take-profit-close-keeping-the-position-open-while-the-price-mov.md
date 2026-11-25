# \#99474 [SC-High] A permissionless protection retry can repeatedly take away the Latched window the owner needs to amend a failing stop-loss / take-profit close, keeping the position open while the price moves against it

**Submitted on Nov 13th 2026 at 20:03:16 UTC by @LT3513 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99474
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

When a triggered take-profit / stop-loss close fails on one of the owner's stored close limits, the protection goes to `Latched`. The documented repair is that the owner amends the limits and the close is retried. But `retryPositionProtectionClose` can be called by any address, with no payment and no delay, and it moves the protection back to `Triggered` with the same stored limits. Amending the limits of an already-triggered protection is only possible while it is `Latched` with an empty order queue; cancel, replace and cancel-and-close require `Armed`; and the Router rejects every ordinary order from the account (full close, partial close, increase) while a protection is active.

So in every round there is a race for the `Latched` window: if a third party's retry lands before the owner's amend, the owner cannot repair or close the position in that round and has to wait for the next attempt to be processed. A third party that keeps winning that race (for example by processing the failing attempt and retrying in one transaction, so `Latched` never exists between transactions) keeps the position open with limits that cannot be met. The third party does not move the price and does not by itself cause a liquidation; what it removes is the owner's otherwise available exit while the market moves.

In the proof of concept one position, one trigger and one price/time schedule are run three ways (the first two differ only in transaction order; the third also has the owner add margin). If the owner's amend lands in the first `Latched` window she exits with a loss of 1,408.96 USDG. If the third party wins every round and the owner does nothing else, the position is liquidated six rounds later and she loses 1,914.40 USDG (505.44 more, 25% of the 2,000 USDG position margin). If she adds margin to avoid liquidation and wins the race in the sixth round, she exits with a loss of 1,909.16 USDG (500.20 more). The third party holds no collateral and no tokens; it pays gas (about 1.59M of execution gas per round in the test) and any oracle update fee.

## Severity

Medium — "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)" in the Immunefi smart-contract impact classification. The damage is extra loss on a user position whose exit is withheld while the price moves. I do not claim High or a permanent freeze: there is a precondition on the victim's own limits, it is an ordering race the owner can win, and the owner can still add margin to postpone liquidation (all three are set out under "Likelihood and preconditions"). If the griefing class is judged not to apply, I ask that it be considered as an Insight: the owner-amend recovery that the protocol documents is not reliably available.

## Root cause

All paths are in `packages/perps/src` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

1. `PositionProtectionBook.sol:382-406` — `retryPositionProtectionClose(uint64)` looks the protection up with `_activeProtection` (no caller check; the owner-only functions use `_ownedActiveProtection`), takes no payment and has no delay. It requires `Latched` and an empty account queue, asks the Router to commit a new close attempt, and `_recordCloseAttempt` (`:564-578`) sets `status = Triggered`.
2. `router/OrderLiquidationBatchLogic.sol:657-680` — `_protectionCloseRequest` builds that attempt from the limits stored in the Book at that moment (`closeFinancialLimits(protectionId)`), so a retry made before an amend carries the old limits.
3. `PositionProtectionBook.sol:315-328` — `amendLatchedPositionProtectionCloseLimits` is owner-only and reverts `OrderRouter__ProtectionNotLatched` unless the status is `Latched`, and `OrderRouter__PendingOrdersExist` unless the account queue is empty. There is no way to amend while `Triggered`.
4. `PositionProtectionBook.sol:408-419` (`_cancel`), `:264-277` (`_replace`), `:341-348` (`cancelPositionProtectionAndCommitClose`) accept only `PendingOpen` / `Armed`.
5. `router/OrderLiquidationBatchLogic.sol:313-315` — a public `commitOrder` reverts `OrderRouter__ProtectionActive` while the account has an active protection. The PoC shows this for a full close, a partial close and an increase order.
6. `PositionProtectionBook.sol:662-668` — when an attempt fails and the position is unchanged, the protection relatches; there is no attempt counter. An expired attempt relatches the same way (PoC: `test_EXITS_AttemptExpiryOnlyRelatches`).
7. `OrderRouterExecutionSidecar.sol:447-452` — while the oracle is live (`!request.oracleFrozen`; both checks are skipped in a frozen market, which I have not tested), a fresh attempt cannot be processed in the block that created it and needs an oracle publish time later than its commit. Once a retry lands, the owner cannot chase it in the same block and must wait for the next round (PoC: `test_RACE_OneRetryBetweenOwnersTwoTransactionsLocksAgain`).

For a protection that has already triggered, `Latched` is the only state in which the owner can change its limits, and any address can end that state.

The protocol's own documents describe the amend as the repair, and ask the protocol's own keeper not to do what the contract lets any address do:

- WORKING_WITH_AI_AGENTS.md:283-284: "If a queued close fails and the protected side and size still match, protection becomes `Latched`; the owner may amend its limits before retrying."
- Cecuro audit report, MED-9 fix comment: "Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them. [...] Tight limits may prevent execution until conditions improve or the owner amends them."
- DEPLOYMENT.md:420-421 (keeper guidance): "Investigate planner, constraint, or other persistent failures before retrying; do not continuously queue the same failing attempt."

## Why a close attempt fails on a limit only an amend can fix

A protection close has no price limit of its own: its target price is the non-binding `CAP_PRICE` (LONG) or `1` (SHORT) (`OrderLiquidationBatchLogic.sol:667`). The bounds the owner has are the four stored close limits copied into the attempt (`:674-677`):

- `maxExecutionNotionalAssets` — size × execution price.
- `maxGrossAccountDebitAssets` — the total the close may debit from the account.
- `maxActionChargeAssets`, `maxExplicitFeesAssets`.

In this market a LONG loses when the basket price rises. For a LONG stop-loss the notional ceiling is therefore the owner's "do not close me worse than price X" bound, and the debit ceiling is her "do not realise more than X" bound. Both get harder to meet as the price keeps rising. If the market gaps through the stop and past the bound, the attempt fails `ConstraintViolation`, and later attempts fail too unless the price comes back. Both bounds are exercised in the PoC (`test_MATCHED_...` uses the notional ceiling and measures the loss; `test_ATTACK_LossCeilingVariant...` uses the debit ceiling and shows only that the same lock holds for three rounds, with no loss measurement). I have not tested SHORT positions; for a SHORT the notional ceiling loosens as the price falls, so the debit ceiling would be the relevant bound there.

## Attack path

1. Victim holds a position with a stop-loss whose stored close limit is tighter than where the market trades after the trigger (PoC: LONG 10,000 at 1.00, 2,000 USDG margin, stop 1.10, accepted worst price 1.12, first price after the gap 1.13).
2. A keeper triggers the stop. The queued close fails `ConstraintViolation` when processed.
3. The third party calls `router.executeOrder(attempt)` and `retryPositionProtectionClose(protectionId)` in one transaction in the first block where the attempt can be processed. The protection is `Triggered` again before any other transaction sees `Latched`. If someone else processes the attempt first, a plain `retryPositionProtectionClose` that lands before the owner's amend has the same effect.
4. Repeat each round. If nobody processes an attempt, it expires after the execution window; `expireOrder` + retry (both permissionless) starts another window.
5. In each round the third party wins, the owner's calls revert: amend `OrderRouter__ProtectionNotLatched`; cancel, replace, cancel-and-close `OrderRouter__ProtectionNotArmed`; `commitOrder` (close, partial close, increase) `OrderRouter__ProtectionActive`.
6. While this lasts, the position follows the market. If the price keeps moving against it, it reaches liquidation unless the owner adds margin.

## Impact (measured)

One position, one trigger, one schedule (one block and one second per step, price 1.13 + 0.01 per step). Control and run A are the matched pair: the only difference between them is which transaction is ordered first in each round. Run B is a second scenario on the same schedule in which the owner also adds margin, so it differs from the control by more than ordering. All figures below are asserted in `test_MATCHED_SameScheduleControlVsLock`.

| Run | Owner's loss (USDG) | vs control |
|---|---|---|
| Control: attempt fails at 1.13, owner's amend lands in the first `Latched` window, retry fills at 1.14 | 1,408.96 | — |
| A: third party first in every round, owner does nothing else; liquidated at 1.19 after 6 rounds | 1,914.40 | +505.44 |
| B: third party first in rounds 0–4, owner adds 1,000 USDG margin, wins round 5, amended close fills at 1.19 | 1,909.16 | +500.20 |

What the owner can still do while the protection is held in `Triggered` (each is a test with assertions):

- Add position margin directly with `CfdEngine.addMargin`: works, and postpones liquidation (with +1,000 USDG, liquidation moves from 1.19 to 1.29 on this schedule). It does not let her close; she stayed locked for all 16 rounds of that run.
- Deposit free collateral: credited exactly, but it is not position margin; with 50,000 USDG deposited the position is liquidated at the same price after the same number of rounds.
- Withdraw free collateral: works (17,999.60 of 19,995.60 withdrawn while locked). The exposure is what is committed to the position, not the whole account.
- Wait: an attempt that passes its 60-second execution window (fixture value) and is expired only relatches; one more retry starts the next window. (Liquidation, or the position changing side or size, ends the protection; those are not ways to close the position at the owner's choice.)

Third party: it needs no collateral. The helper contract it calls through ends with zero clearinghouse balance and zero tokens, and the calling address holds nothing except the liquidation reward it is credited in run A (both asserted). Its cost is native gas — about 1.59M gas of execution for one execute+retry call, measured as a `gasleft()` difference inside the test, so it excludes intrinsic transaction gas and is an approximation — plus the oracle update fee that `executeOrder` forwards (zero in the repository's Pyth mock; on a live deployment the caller pays Pyth's update fee). These are transaction costs, not protocol collateral. If it is also the liquidator it receives the liquidation reward (5.00 USDG in run A).

## Likelihood and preconditions

- The victim must have a triggered protection whose stored limit cannot be met at current prices. With all four limits at `type(uint256).max` there is nothing to amend and a retry only helps the owner.
- It is an ordering race, not an unconditional lock. The tests choose the transaction order; they show what each order produces, not that either side can always obtain it. An owner who processes her own failing attempt and whose amend lands directly behind it (two back-to-back transactions, or one batched transaction from a smart-contract account) wins the round if nothing lands in between — two `RACE` tests. The third party wins the round if its transaction is ordered first, or if one retry lands between the owner's two transactions — the other two `RACE` tests. The third party has to win every round; the owner has to win once.
- Against an owner who follows the documented flow (see `Latched`, then send an amend), answering each `Latched` with a retry is enough. Against an owner who runs her own racing bot, sustaining the lock needs a consistent ordering advantage, which I have not shown. The listed deployment is on Arbitrum Sepolia; on Arbitrum, transactions are ordered by arrival at the sequencer and there is no public mempool, so the contest is who reaches the sequencer first in the first block where the attempt can be processed, and a third party cannot see the owner's two transactions in order to place one between them.
- The third party does not control the price. If the price returns inside the owner's limit, the pending attempt simply fills.
- No privileged role and no special configuration are needed: the protection Book is part of the deployed stack and retry is public.

## Relation to known issues

- Cecuro AI audit report (2026-10-26), MED-9 "Keepers can retry protection closes without the trader's financial limits" (acknowledged). Its root cause is that protection closes carried no trader limits; its fix comment (quoted above) introduced the stored limits and the owner amend. This report is about that amend being pre-empted by the permissionless retry, not about missing limits. A reviewer who reads the MED-9 comment's "Tight limits may prevent execution until conditions improve or the owner amends them" as accepting any delay to the amend would treat this as covered; the comment does not mention that a third party can remove the amend window.
- Same report, MED-15 "Triggered SHORT protection can get stuck and leave a user unprotected": cause is the SHORT close target price of 1 against a zero oracle price. MED-30 "Trader can lose stop-loss automation during routine frozen markets" (on the program's known-issues list): cause is the absence of an automatic frozen-market trigger. Different causes.
- SECURITY.md says a `Triggered`/`Latched` protection cannot be cancelled or replaced and that "Retry is permissionless" (`:544-550`), that a "separate permissionless Book retry appends a new order id" (`:752`), that failed-attempt cleanup and retries are unpaid (`:1356`), and that operators must alert on a prolonged `Latched` state (`:1358-1359`). None of these says that a third party's retry removes the owner's amend; the same documents present the amend as available.

## Recommended fix

Give the owner a repair that does not depend on a state any address can end, keeping the existing ownership, lifecycle and queue checks:

1. Preferred: add an owner-only amend that is also valid while `Triggered`. Because an attempt copies the stored limits when it is committed (`OrderLiquidationBatchLogic.sol:674-677`), new limits would apply from the next attempt; the pending attempt keeps its old limits and fails or expires as it does today. The existing empty-queue check (`PositionProtectionBook.sol:323-325`) would have to allow exactly the protection's own current attempt (`linkedOrderId`) and nothing else. Or
2. Delay non-owner `retryPositionProtectionClose` after each relatch (a later block, or a short interval). This gives the owner a window to amend; it does not guarantee her transaction lands in it, so it is weaker than option 1. Or
3. An owner-only "amend and retry" / "cancel into an owner-authored close" valid in `Latched` and `Triggered`, which atomically terminates the pending attempt (clearing its queue entry and attempt bookkeeping) before committing the replacement.


## Proof of Concept


Save the file below as `packages/perps/test/perps/poc/RetryStarvesLatchedAmend.t.sol` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and run from `packages/perps`:

```
forge test --match-path test/perps/poc/RetryStarvesLatchedAmend.t.sol --match-test "test_(MATCHED|ATTACK|EXITS|RACE)" -vv
```

It uses the repository's own Foundry fixture (it inherits `test/perps/spec/trader/PositionProtection.t.sol` for deployment and helpers). No in-scope contract is modified; the mocks are the ones the repository's fixture already deploys for out-of-scope components: its Pyth mock and its mock settlement token (`MockSettlementToken`, standing in for USDG). The position is opened, protected, triggered, processed, retried, topped up and liquidated through the real Router, protection Book, Engine and clearinghouse entrypoints. The `--match-test` filter only keeps the inherited fixture tests out of the run.

Output of that command:

```
No files changed, compilation skipped
Ran 10 tests for test/perps/poc/RetryStarvesLatchedAmend.t.sol:RetryStarvesLatchedAmendTest
[PASS] test_ATTACK_LossCeilingVariantLocksTheSameWay() (gas: 15702597)
[PASS] test_EXITS_AddMarginPostponesLiquidationButNotTheLock() (gas: 69228994)
Logs:
  liquidation price after +1,000 USDG position margin (1e8): 129000000
  rounds locked: 16
[PASS] test_EXITS_AttemptExpiryOnlyRelatches() (gas: 11965755)
[PASS] test_EXITS_FreeCollateralCanStillBeWithdrawn() (gas: 10253983)
Logs:
  withdrawn while locked: 17999.600000
  left in clearinghouse : 1996.000000
[PASS] test_EXITS_FreeDepositIsCreditedButDoesNotMoveLiquidation() (gas: 30301667)
[PASS] test_MATCHED_SameScheduleControlVsLock() (gas: 57097078)
Logs:
  CONTROL   owner amends in first Latched window, exits at 1.14: loss: 1408.960000
  ATTACK A  locked every round, no top-up, liquidated:           loss: 1914.400000
  ATTACK B  locked 5 rounds, tops up, wins round 5, exits 1.19:  loss: 1909.160000
  A minus CONTROL: 505.440000
  B minus CONTROL: 500.200000
  A: rounds won by the third party before liquidation: 6
  A: liquidation price (1e8): 119000000
  A: liquidation reward credited to the third party (collateral): 5.000000
  gas used by one execute+retry transaction (native cost, not collateral): 1588868
[PASS] test_RACE_GrieferLandingFirstBeatsSmartAccountBatch() (gas: 11921098)
[PASS] test_RACE_OneRetryBetweenOwnersTwoTransactionsLocksAgain() (gas: 13763813)
[PASS] test_RACE_OwnerExecuteThenAmendBackToBackWinsAndExits() (gas: 15475987)
[PASS] test_RACE_SmartAccountBatchLandingFirstWinsAndExits() (gas: 8877220)
Suite result: ok. 10 passed; 0 failed; 0 skipped; finished in 63.37ms (198.37ms CPU time)
Ran 1 test suite in 64.47ms (63.37ms CPU time): 10 tests passed, 0 failed, 0 skipped (10 total tests)
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// A permissionless `retryPositionProtectionClose` can remove the `Latched` window that the owner-only
// `amendLatchedPositionProtectionCloseLimits` needs. Whoever is ordered first in each round wins it.
//
// Place at packages/perps/test/perps/poc/RetryStarvesLatchedAmend.t.sol and run from packages/perps:
//   forge test --match-path test/perps/poc/RetryStarvesLatchedAmend.t.sol --match-test "test_(MATCHED|ATTACK|EXITS|RACE)" -vv
//
// The test contract inherits the repository's PositionProtectionTest only for its deployment fixture and helpers.
// Every protocol step goes through the real Router / PositionProtectionBook / CfdEngine / MarginClearinghouse
// entrypoints. The only mocks are the fixture's own Pyth mock and mock settlement token (both out of scope).
// No in-scope contract is modified. `_mockPythUpdateData` (BasePerpTest) rolls one block and warps one second.
//
// Price/time schedule used by every test: step k is exactly one new block, one second later, with basket price
//   P(k) = 1.13 + 0.01 * k      (k = 0, 1, 2, ...)
// Step 0 is the first block in which the triggered close attempt can be processed.

import {PositionProtectionTest} from "../spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IRouterExec {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory);

}

/// @dev Unprivileged third party. Holds no collateral and no settlement tokens. Any native value it is given is
///      only forwarded as the oracle update fee (zero in the repository's Pyth mock); the Router refunds the rest.
contract RetryGriefer {

    receive() external payable {}

    /// Process the victim's pending (failing) close attempt and immediately queue the next one, in one transaction.
    function executeAndRetry(
        address router,
        address book,
        uint64 orderId,
        uint64 protectionId,
        bytes[] calldata data
    ) external payable returns (OrderV3Types.ExecutionResult memory r, uint64 retryId) {
        r = IRouterExec(router).executeOrder{value: msg.value}(orderId, data);
        retryId = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }

    /// Retry only: used when somebody else (the owner, a keeper) has already processed the failed attempt.
    function retryOnly(
        address book,
        uint64 protectionId
    ) external returns (uint64 retryId) {
        retryId = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }

}

/// @dev Minimal stand-in for a smart-contract trading account that can batch calls (as an ERC-4337 account can).
contract BatchingAccount {

    address internal immutable OWNER = msg.sender;

    function executeBatch(
        address[] calldata targets,
        bytes[] calldata data
    ) external {
        require(msg.sender == OWNER, "owner");
        for (uint256 i; i < targets.length; ++i) {
            (bool ok, bytes memory ret) = targets[i].call(data[i]);
            if (!ok) {
                assembly ("memory-safe") {
                    revert(add(ret, 0x20), mload(ret))
                }
            }
        }
    }

}

contract RetryStarvesLatchedAmendTest is PositionProtectionTest {

    address internal constant GRIEFER_EOA = address(0x6121EF);

    // ALICE: LONG 10,000 at 1.00 with 2,000 USDG margin (in this market a LONG loses when the basket price rises).
    // Stop-loss trigger 1.10. She accepts a close at up to 1.12: notional ceiling = 10,000 * 1.12 = 11,200 USDG.
    // (Protection closes carry no price limit of their own: the notional ceiling is the only price bound she has.)
    uint256 internal constant SL_TRIGGER = 110_000_000;
    uint256 internal constant SL_WORST_PRICE = 112_000_000;
    uint256 internal constant GAP_PRICE = 113_000_000; // P(0): first price after the gap through the stop
    uint256 internal constant STEP = 1_000_000; // 0.01 per step
    uint256 internal constant START_BALANCE = 20_000e6;

    uint256 internal lastGriefTxGas;

    // ------------------------------------------------------------------------------------------------ helpers

    function _price(
        uint256 step
    ) internal pure returns (uint256) {
        return GAP_PRICE + step * STEP;
    }

    /// @dev Advances exactly one block / one second and publishes P(step). One call per step, in every test.
    function _advanceTo(
        uint256 step
    ) internal returns (bytes[] memory data) {
        data = _mockPythUpdateData(_price(step));
    }

    function _st(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    function _assertStatus(
        uint64 protectionId,
        PositionProtectionTypes.PositionProtectionStatus expected
    ) internal view {
        assertEq(uint8(_st(protectionId)), uint8(expected), "protection status");
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _margin(
        address account
    ) internal view returns (uint256 margin) {
        (, margin,,,,,) = engine.positions(account);
    }

    function _wide() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _stopLossParams() internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params = _params(0, SL_TRIGGER);
        params.closeLimits.maxExecutionNotionalAssets = (POSITION_SIZE * SL_WORST_PRICE) / 1e20;
    }

    function _openProtectAndTrigger(
        PositionProtectionTypes.PositionProtectionParams memory params
    ) internal returns (uint64 protectionId, uint64 attemptId) {
        return _openProtectAndTriggerFor(ALICE, params);
    }

    /// @dev Real path: the trader commits an open with an attached stop-loss through the Book, a keeper fills it
    ///      through the Router at 1.00, the price gaps to 1.13 and a keeper triggers the stop.
    function _openProtectAndTriggerFor(
        address trader,
        PositionProtectionTypes.PositionProtectionParams memory params
    ) internal returns (uint64 protectionId, uint64 attemptId) {
        OrderV3Types.OrderRequest memory open =
            _boundedOpenRequest(CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, 0);
        vm.prank(trader);
        uint64 parentId;
        (parentId, protectionId) = protectionActions.commitOpenOrderWithProtection(open, params);
        bytes[] memory data = _mockPythUpdateData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(parentId, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open filled through the Router");
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Armed);
        assertEq(_size(trader), POSITION_SIZE, "position open");

        attemptId = _triggerAt(protectionId, GAP_PRICE);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);
    }

    /// @dev Every owner repair / exit entrypoint that depends on protection state, each with its exact revert.
    function _assertOwnerLockedOut(
        uint64 protectionId
    ) internal {
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wide());

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.replacePositionProtection(protectionId, _params(0, SL_TRIGGER + 30_000_000));

        OrderV3Types.OrderRequest memory manualClose = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, manualClose);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(manualClose);

        // A valid partial close (half the position) is an ordinary order too.
        OrderV3Types.OrderRequest memory partialClose = _manualCloseRequest(CfdTypes.Side.LONG);
        partialClose.sizeDelta = POSITION_SIZE / 2;
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(partialClose);

        // So is a valid increase order that brings margin with it.
        OrderV3Types.OrderRequest memory increase =
            _boundedOpenRequest(CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, 0);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(increase);
    }

    /// @dev One round won by the third party: its transaction is the first to process the attempt in this block.
    function _griefRound(
        RetryGriefer griefer,
        uint64 protectionId,
        uint64 attemptId,
        bytes[] memory data
    ) internal returns (uint64 nextAttemptId) {
        _assertOwnerLockedOut(protectionId);
        vm.prank(GRIEFER_EOA);
        uint256 gasBefore = gasleft();
        (OrderV3Types.ExecutionResult memory r, uint64 retryId) =
            griefer.executeAndRetry(address(router), address(protectionBook), attemptId, protectionId, data);
        lastGriefTxGas = gasBefore - gasleft();
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt fails on the stored limit");
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "limit failure");
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);
        assertEq(router.pendingOrderCounts(ALICE), 1, "fresh unpaid attempt queued");
        assertEq(_size(ALICE), POSITION_SIZE, "position still open");
        _assertOwnerLockedOut(protectionId);
        return retryId;
    }

    /// @dev Liquidation attempt at the current step's price, in the current block.
    function _liquidateNow(
        address liquidator,
        uint256 step
    ) internal returns (bool ok) {
        bytes[] memory data = _livePriceData(_price(step));
        vm.prank(liquidator);
        try router.executeLiquidation(ALICE, data) {
            ok = true;
        } catch {
            ok = false;
        }
    }

    /// @dev Third party wins every round from `firstStep` on; before each round anyone may liquidate.
    ///      Returns the number of rounds won and the step at which liquidation succeeded.
    function _lockUntilLiquidation(
        RetryGriefer griefer,
        uint64 protectionId,
        uint64 attemptId
    ) internal returns (uint256 rounds, uint256 liquidationStep) {
        for (uint256 step; step < 60; ++step) {
            bytes[] memory data = _advanceTo(step);
            if (_liquidateNow(GRIEFER_EOA, step)) {
                return (rounds, step);
            }
            attemptId = _griefRound(griefer, protectionId, attemptId, data);
            ++rounds;
        }
        revert("no liquidation within the schedule");
    }

    /// @dev CONTROL schedule: step 0 the attempt is processed and fails; the owner's amend lands in that first
    ///      `Latched` window; a retry is queued; step 1 it fills at P(1) = 1.14.
    function _runControl(
        uint64 protectionId,
        uint64 attemptId
    ) internal returns (uint256 loss) {
        bytes[] memory data = _advanceTo(0);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attemptId, data);
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched);

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wide());
        vm.prank(TRIGGER_KEEPER);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);

        data = _advanceTo(1);
        vm.prank(EXECUTION_KEEPER);
        r = router.executeOrder(retryId, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry executes at P(1)");
        assertEq(_size(ALICE), 0, "owner is out");
        loss = START_BALANCE - clearinghouse.balanceAssets(ALICE);
    }

    /// @dev ATTACK A: same schedule, third party first in every round, owner does nothing else.
    function _runLockedUntilLiquidation(
        uint64 protectionId,
        uint64 attemptId
    ) internal returns (uint256 loss, uint256 rounds, uint256 liquidationStep, uint256 reward) {
        RetryGriefer griefer = new RetryGriefer();
        (rounds, liquidationStep) = _lockUntilLiquidation(griefer, protectionId, attemptId);
        assertEq(_size(ALICE), 0, "liquidated");
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Liquidated);
        assertEq(clearinghouse.balanceAssets(address(griefer)), 0, "griefer contract never held collateral");
        assertEq(settlementToken.balanceOf(address(griefer)) + settlementToken.balanceOf(GRIEFER_EOA), 0);
        reward = clearinghouse.balanceAssets(GRIEFER_EOA);
        loss = START_BALANCE - clearinghouse.balanceAssets(ALICE);
    }

    /// @dev ATTACK B: same schedule. Third party first in rounds 0-4. The owner adds 1,000 USDG of position margin
    ///      at step 2 (allowed while locked) so she is not liquidated, wins round 5 (her two transactions land first
    ///      at P(5) = 1.18), and the amended close fills at P(6) = 1.19.
    function _runLockedFiveRoundsThenOwnerWins(
        uint64 protectionId,
        uint64 attemptId
    ) internal returns (uint256 loss) {
        RetryGriefer griefer = new RetryGriefer();
        bytes[] memory data;
        for (uint256 step; step < 5; ++step) {
            data = _advanceTo(step);
            if (step == 2) {
                uint256 marginBefore = _margin(ALICE);
                vm.prank(ALICE);
                engine.addMargin(ALICE, 1000e6);
                assertEq(_margin(ALICE), marginBefore + 1000e6, "direct margin top-up works while locked");
            }
            assertFalse(_liquidateNow(GRIEFER_EOA, step), "not liquidatable");
            attemptId = _griefRound(griefer, protectionId, attemptId, data);
        }

        data = _advanceTo(5);
        vm.prank(ALICE); // owner tx 1
        router.executeOrder(attemptId, data);
        vm.prank(ALICE); // owner tx 2, directly behind tx 1
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wide());
        vm.prank(GRIEFER_EOA); // too late: this retry now carries the amended limits
        uint64 retryId = griefer.retryOnly(address(protectionBook), protectionId);

        data = _advanceTo(6);
        assertFalse(_liquidateNow(GRIEFER_EOA, 6), "topped-up position is not liquidatable at 1.19");
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(retryId, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry executes at P(6)");
        assertEq(_size(ALICE), 0, "owner is out");
        loss = START_BALANCE - clearinghouse.balanceAssets(ALICE);
    }

    // -------------------------------------------------------------------------------------------------------------
    // MATCHED: one position, one trigger, one price/time schedule. CONTROL and ATTACK A differ only in who is
    // ordered first in each round, so their loss difference isolates the starved amend. ATTACK B is a second
    // scenario on the same schedule in which the owner also adds 1,000 USDG of margin (more than ordering differs).
    // -------------------------------------------------------------------------------------------------------------
    function test_MATCHED_SameScheduleControlVsLock() public {
        assertEq(clearinghouse.balanceAssets(ALICE), START_BALANCE);
        (uint64 protectionId, uint64 attemptId) = _openProtectAndTrigger(_stopLossParams());
        uint256 snapshot = vm.snapshotState();

        uint256 lossControl = _runControl(protectionId, attemptId);

        vm.revertToState(snapshot);
        (uint256 lossLiquidated, uint256 rounds, uint256 liquidationStep, uint256 reward) =
            _runLockedUntilLiquidation(protectionId, attemptId);
        uint256 gasPerRound = lastGriefTxGas;

        vm.revertToState(snapshot);
        uint256 lossOwnerWinsLate = _runLockedFiveRoundsThenOwnerWins(protectionId, attemptId);

        emit log_named_decimal_uint("CONTROL   owner amends in first Latched window, exits at 1.14: loss", lossControl, 6);
        emit log_named_decimal_uint("ATTACK A  locked every round, no top-up, liquidated:           loss", lossLiquidated, 6);
        emit log_named_decimal_uint("ATTACK B  locked 5 rounds, tops up, wins round 5, exits 1.19:  loss", lossOwnerWinsLate, 6);
        emit log_named_decimal_uint("A minus CONTROL", lossLiquidated - lossControl, 6);
        emit log_named_decimal_uint("B minus CONTROL", lossOwnerWinsLate - lossControl, 6);
        emit log_named_uint("A: rounds won by the third party before liquidation", rounds);
        emit log_named_uint("A: liquidation price (1e8)", _price(liquidationStep));
        emit log_named_decimal_uint("A: liquidation reward credited to the third party (collateral)", reward, 6);
        emit log_named_uint("gas used by one execute+retry transaction (native cost, not collateral)", gasPerRound);

        assertEq(rounds, EXPECTED_ROUNDS, "rounds");
        assertEq(_price(liquidationStep), EXPECTED_LIQUIDATION_PRICE, "liquidation price");
        assertEq(reward, EXPECTED_REWARD, "liquidation reward");
        assertEq(lossControl, EXPECTED_LOSS_CONTROL, "control loss");
        assertEq(lossLiquidated, EXPECTED_LOSS_LIQUIDATED, "locked-until-liquidation loss");
        assertEq(lossOwnerWinsLate, EXPECTED_LOSS_OWNER_WINS_LATE, "locked-five-rounds loss");
        assertGt(lossLiquidated, lossControl);
        assertGt(lossOwnerWinsLate, lossControl);
    }

    uint256 internal constant EXPECTED_ROUNDS = 6;
    uint256 internal constant EXPECTED_LIQUIDATION_PRICE = 119_000_000;
    uint256 internal constant EXPECTED_REWARD = 5_000_000;
    uint256 internal constant EXPECTED_LOSS_CONTROL = 1_408_960_000;
    uint256 internal constant EXPECTED_LOSS_LIQUIDATED = 1_914_400_000;
    uint256 internal constant EXPECTED_LOSS_OWNER_WINS_LATE = 1_909_160_000;

    /// Same lock with the other natural stop-loss bound: "close me only if the total debit is at most 1,250 USDG".
    /// At 1.13 the price loss alone is 1,300, and it grows as the price keeps rising.
    function test_ATTACK_LossCeilingVariantLocksTheSameWay() public {
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, SL_TRIGGER);
        params.closeLimits.maxGrossAccountDebitAssets = 1250e6;
        (uint64 protectionId, uint64 attemptId) = _openProtectAndTrigger(params);
        RetryGriefer griefer = new RetryGriefer();
        for (uint256 step; step < 3; ++step) {
            attemptId = _griefRound(griefer, protectionId, attemptId, _advanceTo(step));
        }
        assertEq(_size(ALICE), POSITION_SIZE);
    }

    // -------------------------------------------------------------------------------------------------------------
    // EXITS: what the owner can still do while the protection is held in `Triggered`, and whether it helps.
    // -------------------------------------------------------------------------------------------------------------

    /// Depositing free settlement is credited exactly, but does not change the outcome: on the same schedule the
    /// position is liquidated at the same price after the same number of rounds.
    function test_EXITS_FreeDepositIsCreditedButDoesNotMoveLiquidation() public {
        (uint64 protectionId, uint64 attemptId) = _openProtectAndTrigger(_stopLossParams());

        uint256 balanceBefore = clearinghouse.balanceAssets(ALICE);
        uint256 freeBefore = _freeSettlementAssets(ALICE);
        uint256 marginBefore = _margin(ALICE);
        settlementToken.mint(ALICE, 50_000e6);
        vm.startPrank(ALICE);
        settlementToken.approve(address(clearinghouse), 50_000e6);
        clearinghouse.deposit(ALICE, 50_000e6);
        vm.stopPrank();
        assertEq(clearinghouse.balanceAssets(ALICE), balanceBefore + 50_000e6, "deposit credited");
        assertEq(_freeSettlementAssets(ALICE), freeBefore + 50_000e6, "credited as free settlement");
        assertEq(_margin(ALICE), marginBefore, "position margin unchanged by a deposit");

        RetryGriefer griefer = new RetryGriefer();
        (uint256 rounds, uint256 liquidationStep) = _lockUntilLiquidation(griefer, protectionId, attemptId);
        assertEq(rounds, EXPECTED_ROUNDS, "same number of rounds as without the deposit");
        assertEq(_price(liquidationStep), EXPECTED_LIQUIDATION_PRICE, "same liquidation price as without it");
    }

    /// Direct position-margin top-up (`CfdEngine.addMargin`) is NOT blocked by the protection lock and does move
    /// the liquidation price. It keeps the position alive; it does not let the owner close it.
    function test_EXITS_AddMarginPostponesLiquidationButNotTheLock() public {
        (uint64 protectionId, uint64 attemptId) = _openProtectAndTrigger(_stopLossParams());
        uint256 marginBefore = _margin(ALICE);
        vm.prank(ALICE);
        engine.addMargin(ALICE, 1000e6);
        assertEq(_margin(ALICE), marginBefore + 1000e6, "margin added while Triggered");

        RetryGriefer griefer = new RetryGriefer();
        (uint256 rounds, uint256 liquidationStep) = _lockUntilLiquidation(griefer, protectionId, attemptId);
        assertGt(_price(liquidationStep), EXPECTED_LIQUIDATION_PRICE, "liquidation pushed out by the top-up");
        assertGt(rounds, EXPECTED_ROUNDS, "owner stayed locked for every extra round");
        emit log_named_uint("liquidation price after +1,000 USDG position margin (1e8)", _price(liquidationStep));
        emit log_named_uint("rounds locked", rounds);
    }

    /// Free settlement can be withdrawn while locked, so the exposure is what is committed to the position.
    function test_EXITS_FreeCollateralCanStillBeWithdrawn() public {
        (uint64 protectionId, uint64 attemptId) = _openProtectAndTrigger(_stopLossParams());
        RetryGriefer griefer = new RetryGriefer();
        attemptId = _griefRound(griefer, protectionId, attemptId, _advanceTo(0));

        uint256 free = _freeSettlementAssets(ALICE);
        uint256 balanceBefore = clearinghouse.balanceAssets(ALICE);
        uint256 walletBefore = settlementToken.balanceOf(ALICE);
        assertGt(free, 0);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, free);
        assertEq(settlementToken.balanceOf(ALICE), walletBefore + free, "wallet received all free settlement");
        assertEq(clearinghouse.balanceAssets(ALICE), balanceBefore - free, "clearinghouse debited the same amount");
        assertEq(_size(ALICE), POSITION_SIZE, "position itself is still open");
        _assertOwnerLockedOut(protectionId);
        emit log_named_decimal_uint("withdrawn while locked", free, 6);
        emit log_named_decimal_uint("left in clearinghouse ", clearinghouse.balanceAssets(ALICE), 6);
    }

    /// Nothing times out: when an attempt passes its deadline it only relatches, and the next retry locks again.
    function test_EXITS_AttemptExpiryOnlyRelatches() public {
        (uint64 protectionId, uint64 attemptId) = _openProtectAndTrigger(_stopLossParams());
        RetryGriefer griefer = new RetryGriefer();
        attemptId = _griefRound(griefer, protectionId, attemptId, _advanceTo(0));

        uint256 window = router.maxExecutionWindowSeconds();
        assertEq(window, 60, "fixture execution window");
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + window + 1);
        _assertOwnerLockedOut(protectionId); // locked for the whole window with nobody processing the attempt

        vm.prank(GRIEFER_EOA);
        OrderV3Types.ExecutionResult memory r = router.expireOrder(attemptId);
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched);
        vm.prank(GRIEFER_EOA);
        griefer.retryOnly(address(protectionBook), protectionId);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);
        _assertOwnerLockedOut(protectionId);
    }

    // -------------------------------------------------------------------------------------------------------------
    // RACE: the tests choose the transaction order; they show what each order produces, not who can obtain it.
    // -------------------------------------------------------------------------------------------------------------

    /// Owner first (externally owned account): she processes her own failing attempt and her amend lands directly
    /// behind it. She wins the round and exits on the next step.
    function test_RACE_OwnerExecuteThenAmendBackToBackWinsAndExits() public {
        (uint64 protectionId, uint64 attemptId) = _openProtectAndTrigger(_stopLossParams());
        RetryGriefer griefer = new RetryGriefer();
        attemptId = _griefRound(griefer, protectionId, attemptId, _advanceTo(0));
        attemptId = _griefRound(griefer, protectionId, attemptId, _advanceTo(1));

        bytes[] memory data = _advanceTo(2);
        vm.prank(ALICE); // tx 1
        router.executeOrder(attemptId, data);
        vm.prank(ALICE); // tx 2, same block, directly behind tx 1
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wide());

        vm.prank(GRIEFER_EOA); // late retry: it now queues a close with the amended limits
        uint64 retryId = griefer.retryOnly(address(protectionBook), protectionId);
        data = _advanceTo(3);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(retryId, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(_size(ALICE), 0, "owner out");
    }

    /// One retry between the owner's two transactions is enough to lose the round. No atomic bundle is needed.
    function test_RACE_OneRetryBetweenOwnersTwoTransactionsLocksAgain() public {
        (uint64 protectionId, uint64 attemptId) = _openProtectAndTrigger(_stopLossParams());

        for (uint256 step; step < 4; ++step) {
            bytes[] memory data = _advanceTo(step);
            vm.prank(ALICE); // tx 1: owner processes her failing attempt
            router.executeOrder(attemptId, data);
            _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched);

            vm.prank(GRIEFER_EOA); // any address
            attemptId = protectionActions.retryPositionProtectionClose(protectionId);

            vm.prank(ALICE); // tx 2: too late
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wide());

            // The fresh attempt cannot be processed in the block that created it, so the owner cannot chase it.
            vm.prank(ALICE);
            OrderV3Types.ExecutionResult memory r = router.executeOrder(attemptId, data);
            assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Pending), "same-block attempt stays pending");
            _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);
        }
        assertEq(_size(ALICE), POSITION_SIZE);
    }

    function _batch(
        uint64 attemptId,
        uint64 protectionId,
        bytes[] memory data
    ) internal view returns (address[] memory targets, bytes[] memory calls) {
        targets = new address[](2);
        calls = new bytes[](2);
        targets[0] = address(router);
        calls[0] = abi.encodeCall(IRouterExec.executeOrder, (attemptId, data));
        targets[1] = address(protectionBook);
        calls[1] = abi.encodeCall(
            IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits, (protectionId, _wide())
        );
    }

    /// Owner first (smart account): "process my failing attempt" + "amend" in ONE transaction that is the first to
    /// process the attempt. She wins the round and exits on the next step.
    function test_RACE_SmartAccountBatchLandingFirstWinsAndExits() public {
        vm.prank(BOB);
        BatchingAccount account = new BatchingAccount();
        _fundTrader(address(account), 20_000e6);
        (uint64 protectionId, uint64 attemptId) = _openProtectAndTriggerFor(address(account), _stopLossParams());

        bytes[] memory data = _advanceTo(0);
        (address[] memory targets, bytes[] memory calls) = _batch(attemptId, protectionId, data);
        vm.prank(BOB);
        account.executeBatch(targets, calls);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched);
        assertEq(protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets, type(uint256).max);

        vm.prank(TRIGGER_KEEPER);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        data = _advanceTo(1);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(retryId, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(_size(address(account)), 0, "smart-account owner out");
    }

    /// Third party first: the same batch then fails against the old attempt id and against the new one.
    function test_RACE_GrieferLandingFirstBeatsSmartAccountBatch() public {
        vm.prank(BOB);
        BatchingAccount account = new BatchingAccount();
        _fundTrader(address(account), 20_000e6);
        (uint64 protectionId, uint64 attemptId) = _openProtectAndTriggerFor(address(account), _stopLossParams());
        RetryGriefer griefer = new RetryGriefer();

        for (uint256 step; step < 3; ++step) {
            bytes[] memory data = _advanceTo(step);
            vm.prank(GRIEFER_EOA);
            (, uint64 retryId) =
                griefer.executeAndRetry(address(router), address(protectionBook), attemptId, protectionId, data);

            (address[] memory targets, bytes[] memory calls) = _batch(attemptId, protectionId, data);
            vm.prank(BOB);
            vm.expectRevert();
            account.executeBatch(targets, calls);

            (targets, calls) = _batch(retryId, protectionId, data);
            vm.prank(BOB);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            account.executeBatch(targets, calls);

            _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);
            attemptId = retryId;
        }
        assertEq(_size(address(account)), POSITION_SIZE, "smart-account owner still locked in");
    }

}
```
