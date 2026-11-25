# \#99124 [SC-Low] Renouncing CfdEngine ownership permanently freezes matured LP deposits and redemptions after degraded mode

**Submitted on Nov 12th 2026 at 00:36:56 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99124
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description

## Brief/Intro

The remediation for known finding LOW-59 prevents ownership renunciation on `HousePool`, `OrderRouterAdmin`, and `EmergencyPauseCoordinator`, because those contracts must retain owner-only emergency recovery authority. The audit's recommendation was broader: disable renunciation in **every governance-bearing perps contract**.

The frozen `CfdEngine` remains renounceable through `CfdTreasury -> Ownable2Step`. This omission affects two unique recovery functions that no other contract or role can invoke:

- `recapitalizePool`, the non-dilutive claimant-recapitalization route; and
- `clearDegradedMode`, the only transition that clears the global degraded latch.

After the owner renounces, any later degraded-mode latch becomes permanent. Degraded mode is ordinary containment: `SECURITY.md:246` lists clearing it "once raw solvency is restored" among the owner's powers. Without an owner, the pool can return to full solvency and still never leave degraded mode. New risk stays disabled and LP epoch settlement keeps reverting.

The PoC uses only public entry points and production V3 order execution; no pool balance is burned or forged. An LP funds the pool with `1,000,000` USDG and queues one quarter of its shares for redemption, two traders build a matched `995,000` USDG book inside every release limit, and another user queues a `25,000` USDG deposit. After the renounce, the basket moves 1.04%, both traders close at the same tick, and a permissionless keeper executes the two closes in one FIFO batch. Paying the winner first leaves a transient prefix below the solvency envelope, so degraded mode latches; collecting the loser's loss then leaves the pool fully solvent with no open positions. From then on, LP settlement reverts with `HousePool__DegradedMode` immediately, after 24 hours and after 30 days, and the deposit and redemption reject cancellation and refund. A control test runs the identical sequence with the owner in place: the owner clears the latch and both requests become claimable.

## Root Cause

`CfdEngine` inherits its ownership implementation from `CfdTreasury`:

```solidity
abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step {
    constructor() Ownable(msg.sender) {
        _currentTreasury.account = msg.sender;
    }
}
```

Neither `CfdTreasury` nor `CfdEngine` overrides `renounceOwnership()`. The inherited OpenZeppelin implementation therefore sets both ownership fields to zero permanently.

By contrast, the LOW-59 remediation added this protection to the other recovery-bearing contracts:

```solidity
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

The Engine's recovery routes remain exclusively owner-gated:

```solidity
function recapitalizePool(uint256 amountAssets) external onlyOwner nonReentrant { ... }

function clearDegradedMode() external onlyOwner {
    if (!degradedMode) revert CfdEngine__NotDegraded();
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
}
```

Once `owner() == address(0)`, ownership cannot be transferred or accepted and neither recovery call can ever succeed.

## End-to-End Sequence

1. A Junior LP supplies `1,000,000` USDG through the public asynchronous vault and permissionless Router settlement path.
2. After the ordinary one-hour cooldown, that LP places one quarter of its shares into the public redemption queue.
3. Two traders fund public Clearinghouse accounts and build matched `995,000` USDG LONG and SHORT positions at `1.00` through V3 `commitOrder` and `executeOrder`, inside the release 40% skew limit and the 25 bps settlement buffer.
4. Another user submits a public `25,000` USDG Junior deposit request.
5. The Engine owner calls the inherited public `renounceOwnership()` function. The Engine records zero owner and zero pending owner.
6. At the next LP epoch the basket has moved 1.04% to `0.9896`. Both traders commit full closes at that price, and a permissionless keeper executes them in one `executeOrderBatch` call at the same tick. The winner is paid first, the prefix falls below the solvency envelope, and degraded mode latches. The loser's loss is then collected.
7. Every position is closed and every trader claim is paid. The canonical protocol accounting snapshot reports `1,006,649.05` USDG of effective solvency against zero maximum liability, so the economic precondition of `clearDegradedMode` holds.
8. `clearDegradedMode()` and `recapitalizePool()` both revert with `OwnableUnauthorizedAccount`, including for the former owner.
9. Public `OrderRouter.settleLpEpoch` and the cached-mark `HousePool.settleLpEpoch` both revert with `HousePool__DegradedMode`.
10. The same reverts repeat with fresh oracle updates more than 24 hours later and again 30 days later.
11. `cancelPendingDeposit` reverts with `DepositEpochAlreadyActive`; `cancelRedeemRequest` reverts with `RedeemCancellationUnavailable`; and `claimRedeemRefund` reverts with `RedeemRefundUnavailable`. The full deposit and every queued redemption share remain pending with zero claimable output.
12. Control: the same steps without step 5. The owner calls `clearDegradedMode()`, the next epoch settles, the full `25,000` USDG deposit becomes claimable, and the redemption becomes claimable for `251,661.76` USDG.

## Impact Details

The demonstrated state is permanent for the deployed contracts:

- raw solvency has recovered;
- the global degraded latch remains set;
- no account can clear it or use the Engine's recapitalization route;
- new opens remain disabled by the Engine planner; and
- LP epoch settlement remains disabled, stranding matured deposits and existing LP redemption shares;
- the matured deposit cannot be cancelled; and
- the matured redemption cannot be cancelled or refunded.

This directly proves the contest's Critical impact: **permanent freezing of funds**. The freeze is not a liquidity shortfall: no position is open, every trader claim is paid, and the pool holds `1,006,649.05` USDG of effective solvency against zero liability. The control test shows that the only missing step is the owner's `clearDegradedMode()` call. The perps stack is intentionally non-upgradeable, so this is an irreversible authorization failure: adding cash, waiting, supplying fresh prices, and invoking every user cancellation and refund path cannot restore LP settlement or release either escrow.

The precondition is the same governance lifecycle action already accepted as the defect in LOW-59: the current owner calls inherited `renounceOwnership()`. LOW-59 prescribed disabling that action in every governance-bearing perps contract, and the contest expressly permits reports proving an acknowledged fix incomplete.

## Incomplete-Fix Analysis

LOW-59 was titled **“Owner can leave emergency recovery ownerless.”** Its root cause and precondition were the inherited `Ownable.renounceOwnership()` function and a later incident requiring owner-only recovery. Its remediation stated:

> Override renounceOwnership() in every governance-bearing perps contract and make it revert.

The frozen fix protects only `HousePool`, `OrderRouterAdmin`, and `EmergencyPauseCoordinator`. It omits the Engine even though the same release introduced/retained owner-only recapitalization and degraded-mode clearing there.

The scope page's "Known Issues and Previous Audits" section accepts a report on the root cause of a fixed or Acknowledged finding when it shows that the fix at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` is incomplete or introduces a new bug.

This PoC proves that omission through the exact later-incident pattern described by LOW-59.

Renouncing Engine ownership is explicitly documented and regression-tested, rather than an unmodelled admin mistake. `SECURITY.md:128` describes how the treasury schedule behaves after the Engine owner renounces, and the project's own regression `test_TreasuryProposal_ActivationDoesNotRequireAnOwner` (`TreasuryProposal.t.sol:340-352`) calls `engine.renounceOwnership()` and asserts that treasury activation remains functional after renunciation. The scope page keeps in scope "a legitimate, documented action by a trusted role that triggers a bug". That is this report: the documented action is the renounce, and the bug is that `CfdEngine` still holds the only `clearDegradedMode()` and `recapitalizePool()` authority, so the renounce removes every path out of degraded mode. That regression covers treasury activation only; it never checks degraded-mode recovery.

## Recommendation

Override `renounceOwnership()` in `CfdEngine` (or `CfdTreasury`) with the same reverting implementation used by the other governance-bearing perps contracts:

```solidity
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

Keep two-step ownership transfer available. Add a regression that the Engine owner cannot renounce while `clearDegradedMode` and `recapitalizePool` remain sole-owner recovery powers.

## References

- [`CfdTreasury` ownership inheritance](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L4-L10)
- [`CfdEngine.recapitalizePool`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L653)
- [`CfdEngine.clearDegradedMode`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L655-L667)
- [`HousePool` non-renunciation override](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L264-L267)
- [`HousePool._requireWithdrawalsLive`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1869)
- [`TrancheVault` matured-deposit and redemption cancellation guards](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L826-L928)
- [`SECURITY.md` non-upgradeable deployment guarantee](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L29-L67)
- [`SECURITY.md` treasury activation after the Engine owner renounces](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L128)
- [`SECURITY.md` owner clears degraded mode once raw solvency is restored](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L246-L247)
- [`TreasuryProposal` ownerless-activation regression](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/admin/TreasuryProposal.t.sol#L340-L352)

## Proof of Concept

## Proof of Concept

Save the following file as `packages/perps/test/perps/regression/EngineRenunciationDegradedRecovery.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ProtocolLensViewTypes} from "@plether/perps/interfaces/ProtocolLensViewTypes.sol";

/// @notice After the Engine owner renounces, an ordinary trading sequence on a 1.04% basket move latches degraded
///         mode while the pool ends fully solvent. Nobody can clear the latch, so LP settlement and both pending LP
///         requests stay frozen forever. The control test runs the same sequence with an owner, who clears it.
/// @dev Only Pyth and USDG are mocked. USDG is only minted to fund LPs and traders; no pool balance is burned or
///      forged. Every protocol action uses a public entry point. No vm.store or in-scope-contract impersonation.
contract EngineRenunciationDegradedRecoveryTest is BasePerpTest {

    address internal constant LIQUIDITY_PROVIDER = address(0xE11001);
    address internal constant WINNER_LONG = address(0xE11002);
    address internal constant LOSER_SHORT = address(0xE11003);
    address internal constant DEPOSITOR = address(0xE11004);
    address internal constant KEEPER = address(0xE11005);

    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant MOVED_PRICE = 98_960_000; // 0.9896, a 1.04% basket move
    uint256 internal constant LP_ASSETS = 1_000_000e6;
    uint256 internal constant TOTAL_SIZE = 995_000e18;
    uint256 internal constant CHUNK_SIZE = 300_000e18;
    uint256 internal constant CHUNK_MARGIN = 10_000e6;
    uint256 internal constant PENDING_DEPOSIT = 25_000e6;

    struct LpRequests {
        uint256 depositId;
        uint256 redeemId;
        uint256 redeemShares;
        uint256 sharesLeft;
    }

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function test_RenouncedEngineCannotLeaveTradingLatchedDegradedMode() public {
        LpRequests memory r = _setUpBookAndLpRequests();

        engine.renounceOwnership();
        assertEq(engine.owner(), address(0));
        assertEq(engine.pendingOwner(), address(0));

        _latchDegradedModeByTrading(r.depositId);

        ProtocolLensViewTypes.ProtocolAccountingSnapshot memory snap =
            engineProtocolLens.getProtocolAccountingSnapshot();
        assertEq(snap.maxLiabilityAssets, 0, "every position is closed");
        assertGe(snap.effectiveSolvencyAssets, snap.maxLiabilityAssets, "the pool is fully solvent");
        emit log_named_uint("effective solvency assets", snap.effectiveSolvencyAssets);
        emit log_named_uint("maximum liability", snap.maxLiabilityAssets);

        // The former owner (this test contract) and everyone else are locked out of both recovery routes.
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engine.clearDegradedMode();
        settlementToken.mint(address(this), 1e6);
        settlementToken.approve(address(engine), 1e6);
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, address(this)));
        engine.recapitalizePool(1e6);

        _assertLpRequestsFrozen(r);
        vm.warp(vm.getBlockTimestamp() + 24 hours + 1);
        _assertLpRequestsFrozen(r);
        vm.warp(vm.getBlockTimestamp() + 30 days);
        _assertLpRequestsFrozen(r);

        assertEq(juniorVault.balanceOf(LIQUIDITY_PROVIDER), r.sharesLeft);
        assertEq(settlementToken.balanceOf(DEPOSITOR), 0, "all depositor USDG stays in vault escrow");
        emit log_named_uint("deposit frozen (USDG atoms)", PENDING_DEPOSIT);
        emit log_named_uint("redemption shares frozen", r.redeemShares);
    }

    function test_Control_OwnedEngineClearsSameLatchAndReleasesLpRequests() public {
        LpRequests memory r = _setUpBookAndLpRequests();

        _latchDegradedModeByTrading(r.depositId);

        engine.clearDegradedMode();
        assertFalse(engine.degradedMode());
        _settleLpEpochAt(MOVED_PRICE);

        assertEq(juniorVault.claimableDepositRequest(r.depositId, DEPOSITOR), PENDING_DEPOSIT);
        assertGt(juniorVault.claimableRedeemAssets(r.redeemId, LIQUIDITY_PROVIDER), 0);
        emit log_named_uint(
            "control: redemption assets claimable", juniorVault.claimableRedeemAssets(r.redeemId, LIQUIDITY_PROVIDER)
        );
    }

    /// @dev An LP funds the pool and queues a redemption, two traders build a matched 995k book inside every release
    ///      limit, and a second user queues a Junior deposit.
    function _setUpBookAndLpRequests() private returns (LpRequests memory r) {
        _publicJuniorDeposit(LIQUIDITY_PROVIDER, LP_ASSETS);

        uint256 cooldownEnd = juniorVault.lastDepositTime(LIQUIDITY_PROVIDER) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) vm.warp(cooldownEnd);
        uint256 shares = juniorVault.balanceOf(LIQUIDITY_PROVIDER);
        r.redeemShares = shares / 4;
        r.sharesLeft = shares - r.redeemShares;
        vm.prank(LIQUIDITY_PROVIDER);
        r.redeemId = juniorVault.requestRedeem(r.redeemShares, LIQUIDITY_PROVIDER, LIQUIDITY_PROVIDER);

        _fundTrader(WINNER_LONG, 100_000e6);
        _fundTrader(LOSER_SHORT, 100_000e6);
        for (uint256 i; i < 3; ++i) {
            _commitAndExecute(WINNER_LONG, CfdTypes.Side.LONG, CHUNK_SIZE, CHUNK_MARGIN, false, ENTRY_PRICE);
            _commitAndExecute(LOSER_SHORT, CfdTypes.Side.SHORT, CHUNK_SIZE, CHUNK_MARGIN, false, ENTRY_PRICE);
        }
        _commitAndExecute(WINNER_LONG, CfdTypes.Side.LONG, 95_000e18, CHUNK_MARGIN, false, ENTRY_PRICE);
        _commitAndExecute(LOSER_SHORT, CfdTypes.Side.SHORT, 95_000e18, CHUNK_MARGIN, false, ENTRY_PRICE);
        assertEq(_maxLiability(), 995_000e6);
        assertFalse(engine.degradedMode());

        settlementToken.mint(DEPOSITOR, PENDING_DEPOSIT);
        vm.startPrank(DEPOSITOR);
        settlementToken.approve(address(juniorVault), PENDING_DEPOSIT);
        r.depositId = juniorVault.requestDeposit(PENDING_DEPOSIT, DEPOSITOR, DEPOSITOR);
        vm.stopPrank();
        assertEq(juniorVault.pendingDepositRequest(r.depositId, DEPOSITOR), PENDING_DEPOSIT);
    }

    /// @dev Both traders close at the same 1.04%-moved tick and a permissionless keeper runs one FIFO batch. Paying
    ///      the winner first leaves a transient prefix below the solvency envelope, which latches degraded mode;
    ///      collecting the loser's loss then leaves the pool solvent with no open liability.
    function _latchDegradedModeByTrading(
        uint256 depositId
    ) private {
        _warpToEpoch(depositId);
        uint64 winnerClose = _commit(WINNER_LONG, CfdTypes.Side.LONG, TOTAL_SIZE, 0, true, MOVED_PRICE);
        uint64 loserClose = _commit(LOSER_SHORT, CfdTypes.Side.SHORT, TOTAL_SIZE, 0, true, MOVED_PRICE);
        assertEq(loserClose, winnerClose + 1, "the two closes are consecutive FIFO items");

        bytes[] memory sameTick = _mockPythUpdateData(MOVED_PRICE);
        vm.prank(KEEPER);
        OrderV3Types.BatchResult memory batch = router.executeOrderBatch(loserClose, sameTick);
        assertEq(batch.terminalCount, 2, "both closes execute");

        (uint256 winnerSize,,,,,,) = engine.positions(WINNER_LONG);
        (uint256 loserSize,,,,,,) = engine.positions(LOSER_SHORT);
        assertEq(winnerSize, 0);
        assertEq(loserSize, 0);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "the winner was paid in full");
        assertTrue(engine.degradedMode(), "ordinary trading latched degraded mode");
    }

    function _assertLpRequestsFrozen(
        LpRequests memory r
    ) private {
        bytes[] memory updateData = _mockPythUpdateData(MOVED_PRICE);
        vm.prank(KEEPER);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        router.settleLpEpoch(updateData);

        vm.prank(KEEPER);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        pool.settleLpEpoch(0, 0);

        vm.prank(DEPOSITOR);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(r.depositId, DEPOSITOR, DEPOSITOR);

        vm.prank(LIQUIDITY_PROVIDER);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(r.redeemId, LIQUIDITY_PROVIDER, LIQUIDITY_PROVIDER);

        vm.prank(LIQUIDITY_PROVIDER);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemRefundUnavailable.selector);
        juniorVault.claimRedeemRefund(r.redeemId, LIQUIDITY_PROVIDER, LIQUIDITY_PROVIDER);

        assertEq(juniorVault.pendingDepositRequest(r.depositId, DEPOSITOR), PENDING_DEPOSIT);
        assertEq(juniorVault.claimableDepositRequest(r.depositId, DEPOSITOR), 0);
        assertEq(juniorVault.pendingRedeemRequest(r.redeemId, LIQUIDITY_PROVIDER), r.redeemShares);
        assertEq(juniorVault.claimableRedeemRequest(r.redeemId, LIQUIDITY_PROVIDER), 0);
        assertEq(juniorVault.claimableRedeemAssets(r.redeemId, LIQUIDITY_PROVIDER), 0);
        assertTrue(engine.degradedMode());
    }

    function _publicJuniorDeposit(
        address lp,
        uint256 assets
    ) private {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        uint256 requestId = juniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _warpToEpoch(requestId);
        _settleLpEpochAt(ENTRY_PRICE);
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, assets, lp, lp);
    }

    function _commitAndExecute(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 executionPrice
    ) private {
        uint64 orderId = _commit(account, side, size, margin, isClose, executionPrice);
        bytes[] memory updateData = _mockPythUpdateData(executionPrice);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 targetPrice
    ) private returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()) + uint256(uint160(account)));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = targetPrice;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
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
        });
        vm.prank(account);
        orderId = router.commitOrder(request);
    }

    function _settleLpEpochAt(
        uint256 price
    ) private {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(updateData);
    }

    function _warpToEpoch(
        uint256 requestId
    ) private {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < maturity) vm.warp(maturity);
    }

}
```

Run from the repository root:

```sh
forge test --root packages/perps \
  --match-path test/perps/regression/EngineRenunciationDegradedRecovery.t.sol -vv
```

Observed result:

```text
Ran 2 tests for test/perps/regression/EngineRenunciationDegradedRecovery.t.sol:EngineRenunciationDegradedRecoveryTest
[PASS] test_Control_OwnedEngineClearsSameLatchAndReleasesLpRequests() (gas: 21201920)
Logs:
  control: redemption assets claimable: 251661761965

[PASS] test_RenouncedEngineCannotLeaveTradingLatchedDegradedMode() (gas: 21609428)
Logs:
  effective solvency assets: 1006649054528
  maximum liability: 0
  deposit frozen (USDG atoms): 25000000000
  redemption shares frozen: 250002250018000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 33.49ms (38.47ms CPU time)
```

USDG amounts are in six-decimal atoms; LP shares use the vault's share decimals.

Only Pyth and USDG are mocked, and USDG is only minted to fund the LPs and traders: no pool balance is burned, minted or forged. The PoC uses public V3 order execution, public vault requests, and permissionless Router settlement; it does not use `vm.store`, impersonate an in-scope contract, or forge in-scope storage. The only privileged call is the owner's own `renounceOwnership()`, and the control test omits it.
