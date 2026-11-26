# \#100022 [SC-Low] Committing an open order collects carry from the account without applying the order's maxGrossAccountDebitAssets and maxActionChargeAssets

**Submitted on Nov 16th 2026 at 04:17:50 UTC by @Kirkeelee for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100022
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Every order carries financial bounds that cap what the order may take from the account. When an account with a position commits an open (increase) order, the commit collects all carry the position has accrued and pays it to the pool. This debit is not counted against `maxGrossAccountDebitAssets` or `maxActionChargeAssets`, at commit or at execution. A close order committed by the same account with the same bounds is rejected. In the PoC an order that allows at most 500 USDG of debit and 10 USDG of charges takes 1,282 USDG from the account when it is committed, and then executes.

## Vulnerability Details

The bounds are mandatory and zero means zero:

```solidity
// OrderV3Types.sol:131-134
/// @notice Mandatory financial authority supplied with every V3 order.
/// @dev Maximum values are inclusive and zero is a real zero allowance. Minimum values are inclusive.
///      `maxGrossAccountDebitAssets` covers settlement debit, trader-claim consumption, and the reserved bounty;
```

Committing an open locks the bounty and the order margin. Both calls realize the account's pending carry first, which debits its settlement balance and pays the pool:

```solidity
// MarginClearinghouse.sol:818-824
function lockReservedSettlement(address account, uint256 amountAssets) external onlyReservedSettlementLocker {
    _checkpointCarryBeforeMarginChange(account);
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
}

// MarginClearinghouse.sol:620-633
function reserveCommittedOrderMargin(address account, uint64 orderId, uint256 amountAssets) external onlyEngineOrOrderRouter {
    ...
    _checkpointCarryBeforeMarginChange(account);
    _lockMargin(account, IMarginClearinghouse.MarginBucket.CommittedOrder, amountAssets);
```

The Book checks the bounds against the commitment effects the Router reports:

```solidity
// OrderLifecycleBook.sol:327-346
function recordCommitment(uint64 orderId, CfdEnginePlanTypes.CloseCommitment calldata effects) external onlyRouter {
    ...
    uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
    if (gross > pending.bounds.maxGrossAccountDebitAssets) {
        revert OrderLifecycleBook__CommitmentBoundExceeded(
            OrderV3Types.ConstraintKind.GrossAccountDebit, gross, pending.bounds.maxGrossAccountDebitAssets
        );
    }
    if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) {
        revert OrderLifecycleBook__CommitmentBoundExceeded(
            OrderV3Types.ConstraintKind.ActionCharge,
            effects.carryCollectedAssets,
            pending.bounds.maxActionChargeAssets
        );
    }
```

For a close, the effects come from the close-commit projection and contain the carry collected. For an open, the Router fills in only the bounty and leaves `carryCollectedAssets` at zero, although the same function then collects the carry:

```solidity
// router/OrderReservationAccounting.sol:108-128
function _reserveExecutionBounty(address account, uint256 sizeDelta, uint256 bounty, bool isClose)
    internal
    returns (CfdEnginePlanTypes.CloseCommitment memory effects)
{
    if (isClose) {
        ...
        return _reserveCloseExecutionBounty(account, sizeDelta);
    }
    effects.bountyFromFreeAssets = bounty;          // every carry field stays 0
    ...
    clearinghouse.lockReservedSettlement(account, bounty);   // realizes pending carry
}
```

The margin reservation runs after the effects were recorded (`OrderCommitHandler.sol:27-33`), so its carry checkpoint is not seen by the Book either. The carry is already collected when the order executes, so the execution-time bound checks do not count it. The spec states the rule only for closes: "Collected commitment carry is included exactly once in gross account debit, assessed/collected action charges, and carry totals for the user's financial bounds" (ACCOUNTING_SPEC.md:943-944).

Result for an account with a position and accrued carry `C`:

- open order with `maxGrossAccountDebitAssets < C` or `maxActionChargeAssets < C`: commit succeeds, `C` is debited, the order executes;
- close order with the same bounds: commit reverts with `OrderLifecycleBook__CommitmentBoundExceeded`.

Suggested fix: return the projected carry in the open path's commitment effects, as the close path does, so `recordCommitment` applies the bounds.

## Impact Details

- An open order debits the account by more than the supplied `maxGrossAccountDebitAssets` and collects more charges than `maxActionChargeAssets`. In the PoC the bounds are 500 USDG and 10 USDG and the commit takes 1,282.19 USDG.
- The stored commitment for the order records zero carry collected.
- The amount taken is the position's accrued carry and is paid to the pool.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L131-L134
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L346
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L128
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L27-L33
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L818-L824
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L620-L633
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L939-L945


## Proof of Concept

## Proof of Concept

Save as `test/scripts/OpenCommitCarryBoundsPoC.t.sol` in the repository root (next to `MockUSDGPerpsLifecycle.t.sol`, whose release-deployment setup it copies) and run from the repository root:

```bash
forge test --match-path test/scripts/OpenCommitCarryBoundsPoC.t.sol -vv
```

- `test_OpenCommit_CollectsCarryAboveTheUsersBounds`: a LONG held for 26 weeks, then an increase order with `maxGrossAccountDebitAssets = 500 USDG` and `maxActionChargeAssets = 10 USDG`. The commit debits 1,282.19 USDG to the pool and the order executes.
- `test_Control_CloseCommit_SameBoundsRevert`: same account and bounds with a close order. The commit reverts with `OrderLifecycleBook__CommitmentBoundExceeded`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {DeployPerpsArbitrumSepolia, MockUSDG} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Mirrors the Arbitrum precompile's block-number view inside the local test EVM.
contract PoCArbSys {
    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }
}

/// @notice Same setup as the repo's test/scripts/MockUSDGPerpsLifecycle.t.sol: runs the release deploy script
/// (script/DeployPerpsArbitrumSepolia.s.sol, release parameters) inside the test EVM. Only Pyth and USDG are mocks.
/// Adds 1,000,000 USDG of Junior and 1,000,000 USDG of Senior liquidity through the normal async LP flow.
abstract contract ReleaseStackPoCBase is Test {
    address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    address internal constant ARBSYS = address(100);
    uint256 internal constant DEPLOYER_KEY = 0xA11CE;
    /// @dev Monday 2024-03-04 10:00 UTC (FX market open, no FAD window) — same anchor as the repo tests.
    uint256 internal constant MONDAY_MORNING = 1_709_532_000;
    uint256 internal constant CAP_PRICE = 2e8;
    uint256 internal constant START_PRICE = 120_000_000; // basket mark 1.20 (8 decimals)
    uint256 internal constant SIZE_QUANTUM = 100e18; // positionSizeQuantum = 1e20 raw units
    uint256 internal constant JUNIOR_LP_ASSETS = 1_000_000e6;
    uint256 internal constant SENIOR_LP_ASSETS = 1_000_000e6;
    uint8 internal constant MODE_LIVE = 1; // allowedExecutionModes bit for ExecutionMode.Live (1 << (Live-1))
    uint8 internal constant MODES_ALL = 7; // Live | Fad | Frozen

    DeployPerpsArbitrumSepolia.DeployedContracts internal stack;
    MockUSDG internal usdg;
    CfdEngine internal engine;
    HousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;
    MarginClearinghouse internal clearinghouse;
    OrderRouter internal router;
    OrderLifecycleBook internal lifecycleBook;
    TerminalNavBookV2 internal navBook;
    EmergencyPauseCoordinator internal coordinator;
    PletherOracle internal oracle;
    MockPyth internal pyth;

    /// Trusted roles: OWNER (deployer, owner of every perps contract) and GUARDIAN.
    address internal OWNER;
    address internal GUARDIAN;
    /// Untrusted actors (anyone can be a keeper).
    address internal KEEPER;
    address internal JUNIOR_LP;
    address internal SENIOR_LP;

    uint256 internal currentBasketPrice;
    uint256 internal _clientNonce;

    function setUp() public virtual {
        vm.chainId(421_614);
        vm.warp(MONDAY_MORNING);
        vm.etch(ARBSYS, address(new PoCArbSys()).code);
        vm.etch(RELEASE_PYTH, address(new MockPyth()).code);
        pyth = MockPyth(RELEASE_PYTH);
        vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
        stack = DeployPerpsArbitrumSepolia(
            deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")
        ).run();

        usdg = stack.settlementToken;
        engine = stack.engine;
        pool = stack.housePool;
        seniorVault = stack.seniorVault;
        juniorVault = stack.juniorVault;
        clearinghouse = stack.clearinghouse;
        router = stack.router;
        lifecycleBook = stack.lifecycleBook;
        navBook = stack.terminalNavBook;
        coordinator = stack.emergencyPauseCoordinator;
        oracle = PletherOracle(stack.pletherOracle);

        OWNER = vm.addr(DEPLOYER_KEY);
        GUARDIAN = makeAddr("guardian");
        KEEPER = makeAddr("keeper");
        JUNIOR_LP = makeAddr("juniorLP");
        SENIOR_LP = makeAddr("seniorLP");
        vm.label(OWNER, "OWNER");

        // Production bootstrap (DEPLOYMENT.md): guardian set, 1 USDG permanent seed per tranche, trading activated.
        vm.startPrank(OWNER);
        coordinator.setGuardian(GUARDIAN);
        usdg.mint(OWNER, 2e6);
        usdg.approve(address(pool), 2e6);
        pool.initializeSeedPosition(false, 1e6, OWNER);
        pool.initializeSeedPosition(true, 1e6, OWNER);
        pool.activateTrading();
        vm.stopPrank();

        _setBasketPrice(START_PRICE);
        router.updateMarkPrice(_updateData());

        // Ordinary LP liquidity through the async epoch flow (Junior first, then Senior), like the v2.0.0 testnet.
        _lpDeposit(juniorVault, JUNIOR_LP, JUNIOR_LP_ASSETS);
        _lpDeposit(seniorVault, SENIOR_LP, SENIOR_LP_ASSETS);
    }

    // ------------------------------------------------------------------ oracle -----------------------------------
    /// @notice Moves every release feed by the same normalized factor so the basket mark becomes `basketPrice`
    /// (8 decimals), publishTime = now. Inverse FX feeds keep their own units. Does NOT push it on-chain: pass
    /// `_updateData()` to a Router call (executeOrder / updateMarkPrice / settleLpEpoch / liquidation).
    function _setBasketPrice(uint256 basketPrice) internal {
        _setBasketPriceAt(basketPrice, vm.getBlockTimestamp());
    }

    function _setBasketPriceAt(uint256 basketPrice, uint256 publishTime) internal {
        currentBasketPrice = basketPrice;
        for (uint256 i; i < 6; ++i) {
            uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
            uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
            pyth.setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), publishTime);
        }
    }

    /// @dev Non-encoded payload: MockPyth answers with the individually configured feed prices.
    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _pushMark(uint256 basketPrice) internal {
        _setBasketPrice(basketPrice);
        vm.prank(KEEPER);
        router.updateMarkPrice(_updateData());
    }

    // ------------------------------------------------------------------ LPs --------------------------------------
    /// @notice requestDeposit -> warp to the target epoch -> keeper settles via Router (atomic oracle refresh) -> claim.
    function _lpDeposit(TrancheVault vault, address lp, uint256 assets) internal returns (uint256 shares) {
        usdg.mint(lp, assets);
        vm.startPrank(lp);
        usdg.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _settleEpoch(requestId);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, assets, lp, lp);
    }

    /// @notice requestRedeem (after the deposit cooldown) -> settle the target epoch -> returns requestId (claim it
    /// yourself with vault.claimRedeem(requestId, shares, lp, lp)).
    function _lpRequestRedeemAndSettle(TrancheVault vault, address lp, uint256 shares)
        internal
        returns (uint256 requestId)
    {
        uint256 cooldownEnd = vault.lastDepositTime(lp) + vault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) vm.warp(cooldownEnd);
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
        _settleEpoch(requestId);
    }

    function _settleEpoch(uint256 epochId) internal {
        uint256 start = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < start) vm.warp(start);
        vm.roll(vm.getBlockNumber() + 1);
        _setBasketPrice(currentBasketPrice);
        vm.prank(KEEPER);
        router.settleLpEpoch(_updateData());
    }

    // ------------------------------------------------------------------ traders ----------------------------------
    function _fundTrader(address trader, uint256 assets) internal {
        usdg.mint(trader, assets);
        vm.startPrank(trader);
        usdg.approve(address(clearinghouse), assets);
        clearinghouse.depositMargin(assets);
        vm.stopPrank();
    }

    /// @notice Permissive bounds (live mode only by default); tighten fields in your own request if the test needs to.
    function _bounds(uint256 maxPostSize) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: MODE_LIVE,
            expectedConfigHash: lifecycleBook.currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint128).max,
            maxExecutionNotionalAssets: type(uint128).max,
            maxGrossAccountDebitAssets: type(uint128).max,
            maxActionChargeAssets: type(uint128).max,
            maxExplicitFeesAssets: type(uint128).max,
            maxPostPositionSize: maxPostSize,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// @dev LONG gains when the basket falls: an opening LONG accepts any fill >= targetPrice, a closing LONG any
    /// fill <= targetPrice (and the reverse for SHORT). Permissive targets: 1 / CAP_PRICE.
    function _permissiveTarget(CfdTypes.Side side, bool isClose) internal pure returns (uint256) {
        bool wantsHigh = (side == CfdTypes.Side.LONG) != isClose;
        return wantsHigh ? 1 : CAP_PRICE;
    }

    function _commit(address trader, CfdTypes.Side side, uint256 sizeDelta, uint256 marginDelta, bool isClose)
        internal
        returns (uint64 orderId)
    {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("poc", trader, ++_clientNonce));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = _permissiveTarget(side, isClose);
        request.isClose = isClose;
        request.bounds = _bounds(isClose ? type(uint128).max : type(uint128).max);
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    /// @notice Keeper executes `orderId` at basket price `fillPrice` after the minimum delay (7 s, 2 blocks).
    function _execute(uint64 orderId, uint256 fillPrice) internal returns (OrderV3Types.ExecutionResult memory r) {
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrice(fillPrice);
        vm.prank(KEEPER);
        r = router.executeOrder(orderId, _updateData());
    }

    function _open(address trader, CfdTypes.Side side, uint256 size, uint256 margin, uint256 fillPrice)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        r = _execute(_commit(trader, side, size, margin, false), fillPrice);
    }

    function _close(address trader, CfdTypes.Side side, uint256 size, uint256 fillPrice)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        r = _execute(_commit(trader, side, size, 0, true), fillPrice);
    }

    function _positionSize(address trader) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(trader);
    }

    function _assertExecuted(OrderV3Types.ExecutionResult memory r, string memory what) internal pure {
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), what);
    }
}


/// The user's maxGrossAccountDebitAssets / maxActionChargeAssets bounds are not applied to the carry that an
/// open (increase) order collects when it is committed. The same bounds do stop a close commitment.
contract OpenCommitCarryBoundsPoC is ReleaseStackPoCBase {
    address internal TRADER = makeAddr("trader");

    uint256 internal constant MAX_GROSS_DEBIT = 500e6; // the trader allows at most 500 USDG out of the account
    uint256 internal constant MAX_ACTION_CHARGE = 10e6; // and at most 10 USDG of charges

    function _commitBounded(CfdTypes.Side side, uint256 sizeDelta, uint256 marginDelta, bool isClose)
        internal
        returns (uint64 orderId)
    {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("bounded", ++_clientNonce));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = _permissiveTarget(side, isClose);
        request.isClose = isClose;
        request.bounds = _bounds(type(uint128).max);
        request.bounds.maxGrossAccountDebitAssets = MAX_GROSS_DEBIT;
        request.bounds.maxActionChargeAssets = MAX_ACTION_CHARGE;
        request.bounds.maxExplicitFeesAssets = MAX_ACTION_CHARGE;
        vm.prank(TRADER);
        orderId = router.commitOrder(request);
    }

    /// LONG 300,000 size (360,000 USDG notional) with 40,000 USDG margin, held for 26 weeks without touching it.
    function _positionWithAccruedCarry() internal {
        _fundTrader(TRADER, 60_000e6);
        _assertExecuted(_open(TRADER, CfdTypes.Side.LONG, 300_000e18, 40_000e6, START_PRICE), "open");
        vm.warp(vm.getBlockTimestamp() + 26 weeks);
        vm.roll(vm.getBlockNumber() + 1000);
        _pushMark(START_PRICE);
    }

    function test_OpenCommit_CollectsCarryAboveTheUsersBounds() public {
        _positionWithAccruedCarry();

        uint256 balanceBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = usdg.balanceOf(address(pool));

        // Increase by 1,000 size (1,200 USDG notional) with 100 USDG of margin, under the bounds above.
        uint64 orderId = _commitBounded(CfdTypes.Side.LONG, 1_000e18, 100e6, false);

        uint256 debitedAtCommit = balanceBefore - clearinghouse.balanceAssets(TRADER);
        uint256 poolReceived = usdg.balanceOf(address(pool)) - poolBefore;
        console2.log("maxGrossAccountDebitAssets ", MAX_GROSS_DEBIT);
        console2.log("maxActionChargeAssets      ", MAX_ACTION_CHARGE);
        console2.log("debited at open commit     ", debitedAtCommit);
        console2.log("paid to the pool as carry  ", poolReceived);

        assertEq(debitedAtCommit, poolReceived, "the commit debit is carry paid to the pool");
        assertGt(debitedAtCommit, MAX_GROSS_DEBIT, "account debited beyond maxGrossAccountDebitAssets");
        assertGt(debitedAtCommit, MAX_ACTION_CHARGE, "charge collected beyond maxActionChargeAssets");

        // The order also executes: the execution-time checks do not count the commit-time carry either.
        _assertExecuted(_execute(orderId, START_PRICE), "bounded increase executes");
        console2.log("debited commit + execution ", balanceBefore - clearinghouse.balanceAssets(TRADER));
    }

    /// Same account, same state, same bounds, but a close: the commitment is rejected by the bounds.
    function test_Control_CloseCommit_SameBoundsRevert() public {
        _positionWithAccruedCarry();
        vm.expectPartialRevert(IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector);
        this.commitBoundedClose();
    }

    function commitBoundedClose() external {
        _commitBounded(CfdTypes.Side.LONG, 1_000e18, 0, true);
    }
}
```

Output (commit a82d4fef):

```
Ran 2 tests for test/scripts/OpenCommitCarryBoundsPoC.t.sol:OpenCommitCarryBoundsPoC
[PASS] test_Control_CloseCommit_SameBoundsRevert() (gas: 5642755)
[PASS] test_OpenCommit_CollectsCarryAboveTheUsersBounds() (gas: 7971825)
Logs:
  maxGrossAccountDebitAssets  500000000
  maxActionChargeAssets       10000000
  debited at open commit      1282195693
  paid to the pool as carry   1282195693
  debited commit + execution  1284958130

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

