# \#100071 [SC-Low] The Engine still accepts `renounceOwnership()`, so after a renounce a degraded-mode latch blocks LP settlement and no account can clear it

**Submitted on Nov 16th 2026 at 07:25:33 UTC by @Nyksx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100071
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description

## Summary

`HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator` override `renounceOwnership()` to revert, but `CfdEngine` does not, although its owner is the only caller of `clearDegradedMode()` and `recapitalizePool()`. If the Engine owner renounces, the next degraded-mode latch blocks LP epoch settlement with no account able to end it, also after every position has closed, so LP principal may stay in the pool without an exit path (1,005,412.09 USDG in the PoC, where settlement still reverts 52 weeks after the latch). No attacker is needed for the latch: profit-taking by separate traders sets it when the book is at its admission limit or within about 0.35% of the pool of it, and an ordinary LP redemption brings the book to that limit. 

## Finding Description

Three ownership-bearing contracts refuse a renounce. `HousePool` states the reason in its NatSpec, and `OrderRouterAdmin.sol:124-127` and `EmergencyPauseCoordinator.sol:101-104` carry the same override:

```solidity
// packages/perps/src/HousePool.sol:264-267 (a82d4fef)
/// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0)); // a renounce reverts here
}
```

`CfdEngine` inherits its ownership from `CfdTreasury`, and neither contract has this override. The same holds for `CfdEngineAdmin` (`CfdEngineAdmin.sol:17`) and `MarginClearinghouse` (`MarginClearinghouse.sol:30`). These three contracts therefore expose the unmodified OpenZeppelin function:

```solidity
// packages/perps/src/CfdTreasury.sol:10 (a82d4fef)
abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step { // no renounceOwnership() override

// packages/perps/src/CfdEngine.sol:46 (a82d4fef)
contract CfdEngine is ICfdEngineTypes, IWithdrawGuard, ICfdEngineAdminHost, CfdTreasury, ReentrancyGuardTransient {

// lib/openzeppelin-contracts/contracts/access/Ownable.sol:76-78
function renounceOwnership() public virtual onlyOwner {
    _transferOwnership(address(0)); // one call, no delay, no second step
}

// lib/openzeppelin-contracts/contracts/access/Ownable2Step.sol:52-55
function _transferOwnership(address newOwner) internal virtual override {
    delete _pendingOwner; // a transfer that was pending is deleted as well
    super._transferOwnership(newOwner);
}
```

A renounce cannot be undone. `transferOwnership()` is `onlyOwner` (`Ownable2Step.sol:43`), `acceptOwnership()` requires the caller to be the pending owner (`Ownable2Step.sol:60-66`), and the pending owner is deleted by the renounce. `CfdEngine` is deployed directly and has no upgrade path.

The Engine owner is the only caller of the two recovery functions for degraded mode. `clearDegradedMode()` contains the only statement that resets the latch:

```solidity
// packages/perps/src/CfdEngine.sol:632-634 (a82d4fef)
function recapitalizePool(
    uint256 amountAssets
) external onlyOwner nonReentrant { // owner-only

// packages/perps/src/CfdEngine.sol:658-667 (a82d4fef)
function clearDegradedMode() external onlyOwner { // owner-only
    if (!degradedMode) {
        revert CfdEngine__NotDegraded();
    }
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false; // the only place in packages/perps/src that resets the latch
    emit DegradedModeCleared();
}
```

The latch itself needs no privileged caller. Any close or liquidation that leaves pool cash net of trader claims below the larger side's maximum profit sets it:

```solidity
// packages/perps/src/CfdEngine.sol:1364-1368 (a82d4fef)
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true; // sticky: a later solvent state does not reset it
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

While the latch is set, every LP epoch settlement reverts:

```solidity
// packages/perps/src/HousePool.sol:832 (a82d4fef), inside _settleLpEpoch()
_requireWithdrawalsLive(statusSnapshot);

// packages/perps/src/HousePool.sol:1864-1870 (a82d4fef)
function _requireWithdrawalsLive(
    HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot
) internal pure {
    if (statusSnapshot.degradedMode) {
        revert HousePool__DegradedMode(); // no redemption is funded and no deposit activates
    }
}
```

LP epoch settlement is the only path that moves LP principal out of the pool. `HousePool` sends redemption assets to a tranche vault only inside that settlement (`HousePool.sol:1007`), and the vault's `withdraw()`, `redeem()` and `claimRedeem()` pay only requests that a settlement has already funded (`TrancheVault.sol:998-1021`). New LP deposits are refused as well, because `_entryStatusBlocked()` returns true while the latch is set (`HousePool.sol:1139-1145`). The `HousePool` owner is a separate owner and holds no function that resets the Engine's latch.

The resulting sequence is the following:

1. The Engine owner calls `engine.renounceOwnership()`. Trading, order execution and LP requests continue as before.
2. A close or liquidation latches degraded mode. No attacker is needed for this step. In the PoC, six separate trader accounts, each acting only in its own interest, fill the book to 3,510.14 USDG (about 0.35% of the pool) below the admission limit, the basket falls 2%, and the latch is set when the second LONG holder takes profit. This needs the basket to move in favour of the side with the smaller maximum profit (a fall, below a price of 1.00), and no SHORT holder to reduce before the LONG holders take profit. The book does not have to be full beforehand. LP redemption funding can pay out all assets above the reserve of `L`, trader claims and `B` (`HousePool.sol:1242-1253`), so a redemption request larger than the free assets leaves the book at the admission limit. In the PoC a book with 253,132.54 USDG of headroom has none left after one LP settlement, and the first take-profit after a 2% decline then sets the latch (the second one after a 1% decline). The step can also be taken on purpose: two trader accounts that hold a hedged pair set the latch at a cost of 4,289.39 USDG, which is the path of the earlier report "A hedged pair at the admission limit latches degraded mode and keeps LP settlement frozen after solvency returns until the owner acts".
3. `clearDegradedMode()` and `recapitalizePool()` revert with `OwnableUnauthorizedAccount` for every account, and no account can become owner.
4. `settleLpEpoch()` reverts with `HousePool__DegradedMode` from then on. In the PoC this holds 2 hours, 25 hours, 7 days and 52 weeks after the latch, on a book with no open position and no liability.


## Impact Explanation

After an Engine renounce, a degraded-mode latch leaves LP epoch settlement without a recovery path. Senior and junior principal that has not yet been funded for redemption stays in the pool, and a junior deposit request whose epoch has started stays in escrow, because its cancel path is closed (`TrancheVault.sol:851-860`). Claims that were funded before the latch stay claimable, and a senior pending deposit can be cancelled once the HousePool owner or pauser sets the LP-settlement hold. In the PoC the pool holds 1,005,412.09 USDG with no open position and no liability, the LP's matured redemption (250,851.78 USDG in the control) is not funded, and settlement still reverts 52 weeks after the latch. 

## Likelihood Explanation

The path needs the Engine owner to call `renounceOwnership()`. The renounce is not treated here as an operating mistake alone, because the project documents and two spec tests consider a renounced Engine for the treasury schedule, and the step that follows it is taken by untrusted accounts. The call is a single transaction without a delay or a confirmation step, and it cannot be reversed. After the renounce no further privileged step and no attacker is needed. In the PoC, profit-taking by separate traders, each acting only in its own interest, sets the latch after a 2% basket decline on a book that is within about 0.35% of the pool of its admission limit, provided the decline favours the side with the smaller maximum profit and the losing side has not reduced first. An ordinary LP redemption that is larger than the free assets brings a book with ample headroom to that limit. Two trader accounts can also set the latch on purpose for about 4,289 USDG at release parameters, measured with the price move in the next block and without carry for a longer holding time.

## Recommendation

Consider adding the override that the three sibling contracts already carry to `CfdTreasury` (which covers `CfdEngine`), `CfdEngineAdmin` and `MarginClearinghouse`:

```diff
 abstract contract CfdTreasury is ICfdEngineTypes, Ownable2Step {
+    /// @notice Ownership cannot be renounced because the owner holds degraded-mode and recapitalization recovery.
+    function renounceOwnership() public view override onlyOwner {
+        revert OwnableInvalidOwner(address(0));
+    }
```

## Proof of Concept

## Proof of Concept

The test uses the repository's `BasePerpTest` stack at `a82d4fef` with the `DEPLOYMENT.md` release risk parameters. It applies the release Router configuration (minimum open notional 1,000 USDG, adverse confidence multiplier 2,500 bps) through the Router timelock. Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked. Orders are committed through the `BasePerpTest` router, which is the test-only `LegacyOrderRouterHarness` (`packages/perps/test/utils/LegacyOrderRouterHarness.sol`); its scalar `commitOrder()` delegatecalls the production V3 `commitOrder()`. A third-party keeper executes the orders with `executeOrder()` and calls the permissionless `settleLpEpoch()` on the Router. The test contract deploys the stack and is therefore the owner of the Engine, the HousePool and the Router admin. The only owner call in the test bodies of the attack path is `engine.renounceOwnership()`; the control calls `engine.clearDegradedMode()`. The fixture helper `_fundJunior()` provides the 1,000,000 USDG junior deposit in `setUp()` and installs its first mark with one call as the Router. This happens before the renounce, with no open position.

- `test_RenounceIsBlockedOnPoolAndRouterAdminButNotOnTheEngine()` shows that `pool.renounceOwnership()` and `routerAdmin.renounceOwnership()` revert with `OwnableInvalidOwner(address(0))`, while `engine.renounceOwnership()` succeeds and deletes a pending transfer. Afterwards `acceptOwnership()` reverts for the former pending owner and `transferOwnership()` reverts for the former owner. The same call succeeds on `engineAdmin` and `clearinghouse`.
- `test_Control_EngineOwnerPresent_ClearRestoresLpSettlement()` (control) keeps the Engine owner. Two trader accounts latch degraded mode with a hedged pair, every position then closes, and LP settlement reverts with `HousePool__DegradedMode`. One owner call to `clearDegradedMode()` succeeds on that book, settlement resumes, and the LP receives 250,851.78 USDG.
- `test_RenouncedEngine_LatchOnAClosedBookBlocksLpSettlementWithNoRecoveryCaller()` renounces first and then runs the same steps. `clearDegradedMode()`, `recapitalizePool()`, `transferOwnership()` and `acceptOwnership()` revert with `OwnableUnauthorizedAccount` for the former owner, the keeper, the LP, a trader and a funded third party. Keeper settlement reverts with `HousePool__DegradedMode` 2 hours, 25 hours, 7 days and 52 weeks after the latch, the LP's redemption request stays pending, and the pool keeps 1,005,412.09 USDG. A flat trader account withdraws its full balance at the end.
- `test_RenouncedEngine_ProfitTakingBySeparateTradersSetsTheLatchWithoutIntent()` needs no coordination between the accounts. After the renounce, three LONG and three SHORT accounts of different sizes (830,000 units per side) open at 0.80, which leaves 3,510.14 USDG of admission headroom (about 0.35% of the pool). The basket falls 2% and the LONG holders take profit one after another, while the SHORT holders keep their positions. The first take-profit does not latch, and the second one does (`E` 993,937.17 USDG against `L` 996,000.00 USDG). The scenario needs the move to favour the side with the smaller maximum profit, and no SHORT holder to reduce first. All positions then close, `clearDegradedMode()` reverts, and keeper settlement reverts with `HousePool__DegradedMode` 2 hours, 25 hours and 52 weeks after the latch.
- `test_RenouncedEngine_LpRedemptionBringsTheBookToTheLimit_OneTakeProfitSetsTheLatch()` starts from a book at 75% of that size, with 253,132.54 USDG of admission headroom. The LP requests a redemption of 30% of its shares, and the keeper settlement one hour later pays 253,132.54 USDG to the vault, which leaves no headroom. After a 1% decline the second take-profit sets the latch, and after a 2% decline the first one does (`E` 745,546.27 USDG against `L` 747,000.00 USDG). The LP claims the 253,132.54 USDG that was funded before the latch. The rest of its request stays pending, keeper settlement reverts 2 hours, 25 hours and 52 weeks after the latch, and the pool still holds 751,012.76 USDG.

Command (from the repository root):

```bash
forge test --root packages/perps --match-path test/Submission_C10.t.sol -vv
```

Key output (excerpt):

```text
[PASS] test_Control_EngineOwnerPresent_ClearRestoresLpSettlement()
  at the latch: E (pool assets net of trader claims): 997721.106446
  at the latch: L (larger side's maximum profit): 999360.000000
  book closed: E: 1005412.087251
  book closed: L: 0.000000
  cost of the hedged pair (fees, VPI, bounties): 4289.390916
  control: LP redemption paid: 250851.784183
[PASS] test_RenounceIsBlockedOnPoolAndRouterAdminButNotOnTheEngine()
[PASS] test_RenouncedEngine_LatchOnAClosedBookBlocksLpSettlementWithNoRecoveryCaller()
  at the latch: E (pool assets net of trader claims): 997721.106446
  at the latch: L (larger side's maximum profit): 999360.000000
  book closed: E: 1005412.087251
  book closed: L: 0.000000
  cost of the hedged pair (fees, VPI, bounties): 4289.390916
  USDG held by the HousePool: 1005412.087251
  hours since the latch, LP settlement still reverts: 2
  hours since the latch, LP settlement still reverts: 25
  hours since the latch, LP settlement still reverts: 168
  hours since the latch, LP settlement still reverts: 8736
  flat trader account withdrew: 10073.424543
[PASS] test_RenouncedEngine_LpRedemptionBringsTheBookToTheLimit_OneTakeProfitSetsTheLatch()
  after opens: admission headroom left, E - (L + B): 253132.542840
  USDG that left the pool for the redemption: 253132.542840
  after LP settlement: admission headroom left: 0.000000
  at the latch: E: 745546.265034
  at the latch: L: 747000.000000
  LP claimed the part funded before the latch: 253132.542840
  hours since the latch, LP settlement still reverts: 2
  hours since the latch, LP settlement still reverts: 25
  hours since the latch, LP settlement still reverts: 8736
  USDG still held by the HousePool: 751012.756080
[PASS] test_RenouncedEngine_ProfitTakingBySeparateTradersSetsTheLatchWithoutIntent()
  after opens: E: 1002000.135383
  after opens: L (SHORT side max profit): 996000.000000
  after opens: admission headroom left, E - (L + B): 3510.135383
  at the latch: E: 993937.169299
  at the latch: L: 996000.000000
  book closed: E: 1005001.430339
  book closed: L: 0.000000
  hours since the latch, LP settlement still reverts: 2
  hours since the latch, LP settlement still reverts: 25
  hours since the latch, LP settlement still reverts: 8736
Suite result: ok. 5 passed; 0 failed; 0 skipped
```

Test file `packages/perps/test/Submission_C10.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title Submission C-10: the Engine still accepts renounceOwnership(), which leaves degraded mode with no caller
/// @notice `HousePool`, `OrderRouterAdmin` and `EmergencyPauseCoordinator` override `renounceOwnership()` to revert.
///         `CfdEngine` does not, and it holds the only two recovery functions for degraded mode,
///         `clearDegradedMode()` and `recapitalizePool()`, both owner-only. After the Engine owner renounces, the next
///         degraded-mode latch blocks LP epoch settlement with no caller that can end it, also after every position
///         has closed and the pool holds no liability.
/// @dev    Release risk parameters (DEPLOYMENT.md) and release Router config. Only Pyth (MockPyth) and USDG
///         (MockSettlementToken) are mocked. Orders are committed through the BasePerpTest router, the test-only
///         `LegacyOrderRouterHarness`, whose scalar `commitOrder()` delegatecalls the production V3 `commitOrder()`;
///         a third-party keeper executes them with `router.executeOrder()`, and LP settlement goes through the
///         permissionless `router.settleLpEpoch()`. The latch is reached in three ways: by a hedged pair of two trader
///         accounts, by profit-taking of separate trader accounts on a book near its admission limit, and by one
///         take-profit after an LP redemption has brought the book to that limit. The test contract is the deployer
///         and therefore the owner of the Engine, HousePool and Router admin. The only owner call in the test bodies
///         is `engine.renounceOwnership()`, except in the control, which calls `engine.clearDegradedMode()`.
contract Submission_C10 is BasePerpTest {

    address internal constant TRADER_LONG = address(0xA77A10);
    address internal constant TRADER_SHORT = address(0xA77B10);
    address internal constant BYSTANDER = address(0xB0B010);
    address internal constant JUNIOR_LP = address(0x1A10);
    address internal constant KEEPER = address(0x4EE710);
    address internal constant NEW_OWNER = address(0x0E0E10);
    address internal constant DONOR = address(0xD0D010);

    uint256 internal constant OPEN_PRICE = 0.8e8; // basket != $1.00
    uint256 internal constant MOVED_PRICE = 0.792e8; // -1.0% basket move: the LONG (smaller max-profit) leg wins
    uint256 internal constant LEG1 = 400_000e18;
    uint256 internal constant LEG2 = 432_800e18; // LEG1 + LEG2 sized to the E >= L + B admission limit
    uint256 internal constant TRADER_DEPOSIT = 40_000e6; // per account
    uint256 internal constant BYSTANDER_SIZE = 10_000e18; // ~8,000 USDG notional
    uint256 internal constant BYSTANDER_MARGIN = 1000e6;
    uint256 internal constant BYSTANDER_DEPOSIT = 10_000e6;
    uint256 internal constant LP_DEPOSIT = 1_000_000e6;

    uint256 internal lpRedeemShares;
    uint256 internal lpRedeemId;

    // ------------------------------------------------------------------ release configuration

    /// @dev DEPLOYMENT.md "Initial release parameters". Execution fee 4 bps and settlement buffer 25 bps are already
    ///      the engine defaults in this stack.
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

    /// @dev The 1,000,000 USDG junior tranche is held by a dedicated LP instead of the test contract (the owner).
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _fundJunior(JUNIOR_LP, LP_DEPOSIT);

        // Release Router initialization values (DEPLOYMENT.md), applied through the 72 h router timelock.
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6;
        config.adverseConfidenceMultiplierBps = 2500;
        config.positionProtectionTriggerBountyAssets = 200_000;
        _setRouterConfig(config);
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release min open notional");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");

        // Start the scenario on the following Monday 06:00 UTC, away from weekend and FAD windows.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(OPEN_PRICE);

        _fundTrader(TRADER_LONG, TRADER_DEPOSIT);
        _fundTrader(TRADER_SHORT, TRADER_DEPOSIT);
        _fundTrader(BYSTANDER, BYSTANDER_DEPOSIT);

        assertEq(engine.owner(), address(this), "test contract owns the Engine");
        assertEq(pool.owner(), address(this), "test contract owns the HousePool");
    }

    // ------------------------------------------------------------------ helpers

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _nextOracleSecond() internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        _nextOracleSecond();
        bytes[] memory data = _livePriceData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    /// @dev Real order path: the account commits, a third-party keeper executes with a post-commit Pyth update.
    function _routerTrade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "router order executed");
    }

    /// @dev Permissionless keeper LP settlement through the Router (fresh PoolReconcile mark, then HousePool settle).
    function _keeperSettleLp(
        uint256 price
    ) internal returns (bool ok, bytes4 err) {
        _nextOracleSecond();
        bytes[] memory data = _livePriceData(price);
        vm.prank(KEEPER);
        try router.settleLpEpoch(data) {
            ok = true;
        } catch (bytes memory reason) {
            if (reason.length >= 4) {
                err = bytes4(reason);
            }
        }
    }

    /// @dev E = pool.totalAssets() - trader claims (CfdEngine._availableCashForFreshPoolPayouts()).
    function _effectiveAssets() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    /// @dev The junior LP queues a redemption of a quarter of its shares for the next LP epoch.
    function _lpRequestsRedeem() internal {
        lpRedeemShares = juniorVault.balanceOf(JUNIOR_LP) / 4;
        vm.prank(JUNIOR_LP);
        lpRedeemId = juniorVault.requestRedeem(lpRedeemShares, JUNIOR_LP, JUNIOR_LP);
        assertGt(lpRedeemShares, 0, "redeem request queued");
    }

    function _assertLpRedeemUnfunded() internal view {
        assertEq(juniorVault.claimableRedeemRequest(lpRedeemId, JUNIOR_LP), 0, "nothing claimable");
        assertEq(juniorVault.pendingRedeemRequest(lpRedeemId, JUNIOR_LP), lpRedeemShares, "request still pending");
    }

    function _lpClaimsRedeem() internal returns (uint256 assetsOut) {
        uint256 claimable = juniorVault.claimableRedeemRequest(lpRedeemId, JUNIOR_LP);
        assertEq(claimable, lpRedeemShares, "whole request funded");
        uint256 before = settlementToken.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeem(lpRedeemId, claimable, JUNIOR_LP, JUNIOR_LP);
        assetsOut = settlementToken.balanceOf(JUNIOR_LP) - before;
        assertGt(assetsOut, 200_000e6, "LP receives its redemption");
    }

    /// @dev Two trader accounts open a hedged LONG/SHORT pair, a bystander opens a small LONG, the LP queues a
    ///      redemption. After a 1% basket decline the winning LONG leg closes first, which latches degraded mode
    ///      (the path of the earlier report on the hedged-pair latch). Then every remaining position closes.
    function _latchThenCloseEveryPosition() internal returns (uint256 latchedAt) {
        _routerTrade(TRADER_LONG, CfdTypes.Side.LONG, LEG1, 12_000e6, false, OPEN_PRICE);
        _routerTrade(TRADER_SHORT, CfdTypes.Side.SHORT, LEG1, 12_000e6, false, OPEN_PRICE);
        _routerTrade(TRADER_LONG, CfdTypes.Side.LONG, LEG2, 13_000e6, false, OPEN_PRICE);
        _routerTrade(TRADER_SHORT, CfdTypes.Side.SHORT, LEG2, 13_000e6, false, OPEN_PRICE);
        _routerTrade(BYSTANDER, CfdTypes.Side.LONG, BYSTANDER_SIZE, BYSTANDER_MARGIN, false, OPEN_PRICE);
        assertFalse(engine.degradedMode(), "not degraded after opens");
        _lpRequestsRedeem();

        _refreshMark(MOVED_PRICE);
        _routerTrade(TRADER_LONG, CfdTypes.Side.LONG, LEG1 + LEG2, 0, true, MOVED_PRICE);
        assertTrue(engine.degradedMode(), "the winning hedged leg latches degraded mode");
        latchedAt = vm.getBlockTimestamp();
        emit log_named_decimal_uint("at the latch: E (pool assets net of trader claims)", _effectiveAssets(), 6);
        emit log_named_decimal_uint("at the latch: L (larger side's maximum profit)", _maxLiability(), 6);

        _routerTrade(TRADER_SHORT, CfdTypes.Side.SHORT, LEG1 + LEG2, 0, true, MOVED_PRICE);
        _routerTrade(BYSTANDER, CfdTypes.Side.LONG, BYSTANDER_SIZE, 0, true, MOVED_PRICE);
        (uint256 sizeLong,,,,,,) = engine.positions(TRADER_LONG);
        (uint256 sizeShort,,,,,,) = engine.positions(TRADER_SHORT);
        (uint256 sizeBystander,,,,,,) = engine.positions(BYSTANDER);
        assertEq(sizeLong + sizeShort + sizeBystander, 0, "no position is open");
        assertEq(_maxLiability(), 0, "the pool has no position liability");
        assertGt(_effectiveAssets(), 990_000e6, "the pool holds its assets");
        assertTrue(engine.degradedMode(), "the latch stays set on a book with no position");
        emit log_named_decimal_uint("book closed: E", _effectiveAssets(), 6);
        emit log_named_decimal_uint("book closed: L", _maxLiability(), 6);
        uint256 pairBalance = clearinghouse.balanceAssets(TRADER_LONG) + clearinghouse.balanceAssets(TRADER_SHORT);
        emit log_named_decimal_uint(
            "cost of the hedged pair (fees, VPI, bounties)", 2 * TRADER_DEPOSIT - pairBalance, 6
        );
        assertLt(2 * TRADER_DEPOSIT - pairBalance, 5000e6, "the pair costs under 5,000 USDG");
    }

    function _expectNotOwner(
        address caller
    ) internal {
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableUnauthorizedAccount.selector, caller));
    }

    // ------------------------------------------------------------------ tests

    /// @dev The three contracts of the earlier fix refuse a renounce. The Engine, its admin module and the
    ///      clearinghouse accept it. After the Engine renounce the owner is the zero address, a transfer that was
    ///      pending is deleted, and no account can become owner again.
    function test_RenounceIsBlockedOnPoolAndRouterAdminButNotOnTheEngine() public {
        assertEq(routerAdmin.owner(), address(this), "test contract owns the Router admin");
        assertEq(engineAdmin.owner(), address(this), "test contract owns the Engine admin");
        assertEq(clearinghouse.owner(), address(this), "test contract owns the clearinghouse");

        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableInvalidOwner.selector, address(0)));
        pool.renounceOwnership();
        vm.expectRevert(abi.encodeWithSelector(Ownable.OwnableInvalidOwner.selector, address(0)));
        routerAdmin.renounceOwnership();

        // A transfer to a new owner is pending when the renounce happens.
        engine.transferOwnership(NEW_OWNER);
        assertEq(engine.pendingOwner(), NEW_OWNER, "transfer pending");

        engine.renounceOwnership();
        assertEq(engine.owner(), address(0), "the Engine has no owner");
        assertEq(engine.pendingOwner(), address(0), "the pending transfer is deleted");

        // No account can become owner again.
        vm.prank(NEW_OWNER);
        _expectNotOwner(NEW_OWNER);
        engine.acceptOwnership();
        _expectNotOwner(address(this));
        engine.transferOwnership(NEW_OWNER);
        _expectNotOwner(address(this));
        engine.renounceOwnership();

        // The two sibling contracts without the override behave the same way.
        engineAdmin.renounceOwnership();
        assertEq(engineAdmin.owner(), address(0), "the Engine admin has no owner");
        clearinghouse.renounceOwnership();
        assertEq(clearinghouse.owner(), address(0), "the clearinghouse has no owner");

        // The HousePool and Router admin keep their owner, but neither holds a function that clears the latch.
        assertEq(pool.owner(), address(this), "HousePool owner unchanged");
        assertEq(routerAdmin.owner(), address(this), "Router admin owner unchanged");
    }

    /// @dev Control: the Engine keeps its owner. The same latch on the same closed book is cleared by one owner call,
    ///      and the LP redemption is funded.
    function test_Control_EngineOwnerPresent_ClearRestoresLpSettlement() public {
        uint256 latchedAt = _latchThenCloseEveryPosition();

        vm.warp(latchedAt + 2 hours); // the redeem epoch has matured
        (bool ok, bytes4 err) = _keeperSettleLp(MOVED_PRICE);
        assertFalse(ok, "LP settlement blocked while latched");
        assertEq(err, IHousePool.HousePool__DegradedMode.selector, "HousePool__DegradedMode");

        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "the owner clears the latch on the closed book");
        (ok,) = _keeperSettleLp(MOVED_PRICE);
        assertTrue(ok, "control: LP settlement succeeds after the owner call");
        uint256 out = _lpClaimsRedeem();
        emit log_named_decimal_uint("control: LP redemption paid", out, 6);
    }

    /// @dev The Engine owner renounces first. Trading and LP requests work as before. The same latch on the same
    ///      closed book then has no caller: both recovery functions revert for every account, LP settlement reverts
    ///      2 hours, 25 hours, 7 days and 52 weeks later, and the LP's matured redemption is never funded.
    function test_RenouncedEngine_LatchOnAClosedBookBlocksLpSettlementWithNoRecoveryCaller() public {
        engine.renounceOwnership();
        assertEq(engine.owner(), address(0), "the Engine has no owner");

        uint256 latchedAt = _latchThenCloseEveryPosition();
        uint256 poolCash = settlementToken.balanceOf(address(pool));
        emit log_named_decimal_uint("USDG held by the HousePool", poolCash, 6);
        assertGt(poolCash, 990_000e6, "LP capital is in the pool");

        // Neither recovery function has a caller. The former Engine owner still owns the HousePool and Router admin.
        address[5] memory callers = [address(this), KEEPER, JUNIOR_LP, TRADER_LONG, DONOR];
        settlementToken.mint(DONOR, 100_000e6);
        vm.prank(DONOR);
        settlementToken.approve(address(engine), 100_000e6);
        for (uint256 i; i < callers.length; ++i) {
            vm.prank(callers[i]);
            _expectNotOwner(callers[i]);
            engine.clearDegradedMode();
            vm.prank(callers[i]);
            _expectNotOwner(callers[i]);
            engine.recapitalizePool(100_000e6);
            vm.prank(callers[i]);
            _expectNotOwner(callers[i]);
            engine.transferOwnership(callers[i]);
            vm.prank(callers[i]);
            _expectNotOwner(callers[i]);
            engine.acceptOwnership();
        }

        // LP settlement reverts at every later time. The times are Mondays or Tuesdays in live market hours.
        uint256[4] memory waits = [uint256(2 hours), 25 hours, 7 days, 52 weeks];
        for (uint256 i; i < waits.length; ++i) {
            vm.warp(latchedAt + waits[i]);
            (bool ok, bytes4 err) = _keeperSettleLp(MOVED_PRICE);
            assertFalse(ok, "LP settlement blocked");
            assertEq(err, IHousePool.HousePool__DegradedMode.selector, "HousePool__DegradedMode");
            _assertLpRedeemUnfunded();
            assertEq(pool.getMaxJuniorWithdraw(), 0, "junior withdraw capacity is zero");
            assertEq(_maxLiability(), 0, "still no position liability");
            assertTrue(engine.degradedMode(), "still latched");
            emit log_named_uint("hours since the latch, LP settlement still reverts", waits[i] / 1 hours);
        }
        assertEq(settlementToken.balanceOf(address(pool)), poolCash, "the pool still holds the same USDG");
        assertEq(settlementToken.balanceOf(JUNIOR_LP), 0, "the LP has received nothing");

        // Traders are not held: a flat account withdraws its settlement balance while the latch is set.
        uint256 free = clearinghouse.balanceAssets(BYSTANDER);
        vm.prank(BYSTANDER);
        clearinghouse.withdraw(BYSTANDER, free);
        assertEq(settlementToken.balanceOf(BYSTANDER), free, "a flat trader account can still withdraw");
        emit log_named_decimal_uint("flat trader account withdrew", free, 6);
    }

    /// @dev Three LONG and three SHORT accounts, each with its own deposit and one position.
    function _sixAccounts() internal pure returns (address[3] memory longs, address[3] memory shorts) {
        longs = [address(0x10A1), address(0x10A2), address(0x10A3)];
        shorts = [address(0x50A1), address(0x50A2), address(0x50A3)];
    }

    /// @dev Funds the six accounts and opens their positions at 0.80, alternating sides.
    function _openSixAccountBook(
        uint256[3] memory longSizes,
        uint256[3] memory shortSizes
    ) internal {
        (address[3] memory longs, address[3] memory shorts) = _sixAccounts();
        for (uint256 i; i < 3; ++i) {
            _fundTrader(longs[i], TRADER_DEPOSIT);
            _fundTrader(shorts[i], TRADER_DEPOSIT);
        }
        for (uint256 i; i < 3; ++i) {
            _routerTrade(longs[i], CfdTypes.Side.LONG, longSizes[i], 12_000e6, false, OPEN_PRICE);
            _routerTrade(shorts[i], CfdTypes.Side.SHORT, shortSizes[i], 12_000e6, false, OPEN_PRICE);
        }
        assertFalse(engine.degradedMode(), "not degraded after opens");
    }

    /// @dev Admission headroom E - (L + B), where B is the settlement buffer (25 bps of L, rounded up).
    function _admissionHeadroom() internal view returns (uint256) {
        uint256 l = _maxLiability();
        uint256 b = (l * engine.settlementBufferBps() + 9999) / 10_000;
        uint256 e = _effectiveAssets();
        assertGe(e, l + b, "the book is inside the admission limit");
        return e - (l + b);
    }

    /// @dev No attacker: six separate trader accounts, each acting only in its own interest, fill the book to just
    ///      under the `E >= L + B` admission limit. The basket falls 2% and the LONG holders take profit one after
    ///      another while the SHORT holders keep their positions. The second take-profit sets the latch. Every
    ///      position then closes, and LP settlement has no recovery caller.
    function test_RenouncedEngine_ProfitTakingBySeparateTradersSetsTheLatchWithoutIntent() public {
        engine.renounceOwnership();

        (address[3] memory longs, address[3] memory shorts) = _sixAccounts();
        uint256[3] memory longSizes = [uint256(300_000e18), 280_000e18, 250_000e18]; // 830,000 units
        uint256[3] memory shortSizes = [uint256(350_000e18), 250_000e18, 230_000e18]; // 830,000 units
        uint256 fallenPrice = 0.784e8; // -2.0% basket move

        _openSixAccountBook(longSizes, shortSizes);
        emit log_named_decimal_uint("after opens: E", _effectiveAssets(), 6);
        emit log_named_decimal_uint("after opens: L (SHORT side max profit)", _maxLiability(), 6);
        emit log_named_decimal_uint("after opens: admission headroom left, E - (L + B)", _admissionHeadroom(), 6);
        _lpRequestsRedeem();

        // The basket falls 2%. The LONG holders are in profit and close; nobody acts against the pool on purpose.
        _refreshMark(fallenPrice);
        _routerTrade(longs[0], CfdTypes.Side.LONG, longSizes[0], 0, true, fallenPrice);
        assertFalse(engine.degradedMode(), "the first take-profit does not latch");
        _routerTrade(longs[1], CfdTypes.Side.LONG, longSizes[1], 0, true, fallenPrice);
        assertTrue(engine.degradedMode(), "the second take-profit latches degraded mode");
        uint256 latchedAt = vm.getBlockTimestamp();
        emit log_named_decimal_uint("at the latch: E", _effectiveAssets(), 6);
        emit log_named_decimal_uint("at the latch: L", _maxLiability(), 6);

        // The remaining holders close as well.
        _routerTrade(longs[2], CfdTypes.Side.LONG, longSizes[2], 0, true, fallenPrice);
        for (uint256 i; i < 3; ++i) {
            _routerTrade(shorts[i], CfdTypes.Side.SHORT, shortSizes[i], 0, true, fallenPrice);
        }
        assertEq(_maxLiability(), 0, "the pool has no position liability");
        assertGt(_effectiveAssets(), 990_000e6, "the pool holds its assets");
        assertTrue(engine.degradedMode(), "the latch stays set on a book with no position");
        emit log_named_decimal_uint("book closed: E", _effectiveAssets(), 6);
        emit log_named_decimal_uint("book closed: L", _maxLiability(), 6);

        vm.prank(KEEPER);
        _expectNotOwner(KEEPER);
        engine.clearDegradedMode();
        _expectNotOwner(address(this));
        engine.clearDegradedMode();

        uint256[3] memory waits = [uint256(2 hours), 25 hours, 52 weeks];
        for (uint256 i; i < waits.length; ++i) {
            vm.warp(latchedAt + waits[i]);
            (bool ok, bytes4 err) = _keeperSettleLp(fallenPrice);
            assertFalse(ok, "LP settlement blocked");
            assertEq(err, IHousePool.HousePool__DegradedMode.selector, "HousePool__DegradedMode");
            _assertLpRedeemUnfunded();
            emit log_named_uint("hours since the latch, LP settlement still reverts", waits[i] / 1 hours);
        }
    }

    /// @dev No attacker and no full book: the six accounts fill the book to 75% of the size above, which leaves about
    ///      253,000 USDG of admission headroom. An LP asks to redeem 30% of its shares. The keeper settlement pays out
    ///      all assets above the reserve, which leaves the book at the admission limit. After a 2% decline the first
    ///      take-profit sets the latch; after a 1% decline the second one does. The unfunded part of the LP's request
    ///      and its remaining shares then have no exit, while the part funded before the latch stays claimable.
    function test_RenouncedEngine_LpRedemptionBringsTheBookToTheLimit_OneTakeProfitSetsTheLatch() public {
        engine.renounceOwnership();

        (address[3] memory longs, address[3] memory shorts) = _sixAccounts();
        uint256[3] memory longSizes = [uint256(225_000e18), 210_000e18, 187_500e18]; // 622,500 units
        uint256[3] memory shortSizes = [uint256(262_500e18), 187_500e18, 172_500e18]; // 622,500 units

        _openSixAccountBook(longSizes, shortSizes);
        uint256 headroomBefore = _admissionHeadroom();
        emit log_named_decimal_uint("after opens: admission headroom left, E - (L + B)", headroomBefore, 6);
        assertGt(headroomBefore, 250_000e6, "the book is far inside the admission limit");

        // An ordinary LP redemption request, funded by an ordinary keeper settlement one hour later.
        lpRedeemShares = (juniorVault.balanceOf(JUNIOR_LP) * 30) / 100;
        vm.prank(JUNIOR_LP);
        lpRedeemId = juniorVault.requestRedeem(lpRedeemShares, JUNIOR_LP, JUNIOR_LP);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        (bool ok, bytes4 err) = _keeperSettleLp(OPEN_PRICE);
        assertTrue(ok, "LP settlement succeeds");
        uint256 paidToVault = poolBefore - settlementToken.balanceOf(address(pool));
        uint256 fundedShares = juniorVault.claimableRedeemRequest(lpRedeemId, JUNIOR_LP);
        uint256 headroomAfter = _admissionHeadroom();
        emit log_named_decimal_uint("USDG that left the pool for the redemption", paidToVault, 6);
        emit log_named_decimal_uint("after LP settlement: admission headroom left", headroomAfter, 6);
        assertGt(fundedShares, 0, "part of the request is funded");
        assertLt(fundedShares, lpRedeemShares, "the request is larger than the free assets");
        assertLt(headroomAfter, 1e6, "the redemption leaves the book at the admission limit");
        assertFalse(engine.degradedMode(), "not degraded after LP settlement");

        // A 1% decline: the first take-profit does not latch, the second one does.
        uint256 snap = vm.snapshotState();
        _refreshMark(MOVED_PRICE);
        _routerTrade(longs[0], CfdTypes.Side.LONG, longSizes[0], 0, true, MOVED_PRICE);
        assertFalse(engine.degradedMode(), "1% decline: the first take-profit does not latch");
        _routerTrade(longs[1], CfdTypes.Side.LONG, longSizes[1], 0, true, MOVED_PRICE);
        assertTrue(engine.degradedMode(), "1% decline: the second take-profit latches degraded mode");
        vm.revertToState(snap);

        // A 2% decline: the first take-profit latches.
        uint256 fallenPrice = 0.784e8;
        _refreshMark(fallenPrice);
        _routerTrade(longs[0], CfdTypes.Side.LONG, longSizes[0], 0, true, fallenPrice);
        assertTrue(engine.degradedMode(), "2% decline: the first take-profit latches degraded mode");
        uint256 latchedAt = vm.getBlockTimestamp();
        emit log_named_decimal_uint("at the latch: E", _effectiveAssets(), 6);
        emit log_named_decimal_uint("at the latch: L", _maxLiability(), 6);

        for (uint256 i = 1; i < 3; ++i) {
            _routerTrade(longs[i], CfdTypes.Side.LONG, longSizes[i], 0, true, fallenPrice);
        }
        for (uint256 i; i < 3; ++i) {
            _routerTrade(shorts[i], CfdTypes.Side.SHORT, shortSizes[i], 0, true, fallenPrice);
        }
        assertEq(_maxLiability(), 0, "the pool has no position liability");
        assertTrue(engine.degradedMode(), "the latch stays set on a book with no position");

        // The part funded before the latch is claimable. The rest of the request stays pending.
        uint256 lpBefore = settlementToken.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeem(lpRedeemId, fundedShares, JUNIOR_LP, JUNIOR_LP);
        emit log_named_decimal_uint(
            "LP claimed the part funded before the latch", settlementToken.balanceOf(JUNIOR_LP) - lpBefore, 6
        );
        uint256 pendingShares = juniorVault.pendingRedeemRequest(lpRedeemId, JUNIOR_LP);
        assertEq(pendingShares, lpRedeemShares - fundedShares, "the rest of the request is still pending");

        uint256[3] memory waits = [uint256(2 hours), 25 hours, 52 weeks];
        for (uint256 i; i < waits.length; ++i) {
            vm.warp(latchedAt + waits[i]);
            (ok, err) = _keeperSettleLp(fallenPrice);
            assertFalse(ok, "LP settlement blocked");
            assertEq(err, IHousePool.HousePool__DegradedMode.selector, "HousePool__DegradedMode");
            assertEq(juniorVault.pendingRedeemRequest(lpRedeemId, JUNIOR_LP), pendingShares, "request still pending");
            assertEq(juniorVault.claimableRedeemRequest(lpRedeemId, JUNIOR_LP), 0, "nothing more is funded");
            emit log_named_uint("hours since the latch, LP settlement still reverts", waits[i] / 1 hours);
        }
        emit log_named_decimal_uint("USDG still held by the HousePool", settlementToken.balanceOf(address(pool)), 6);
    }

}
```
