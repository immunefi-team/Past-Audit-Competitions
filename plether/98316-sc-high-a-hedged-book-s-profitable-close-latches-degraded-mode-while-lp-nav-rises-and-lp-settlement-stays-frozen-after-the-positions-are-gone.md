# \#98316 [SC-High] A hedged book's profitable close latches degraded mode while LP NAV rises, and LP settlement stays frozen after the positions are gone

**Submitted on Nov 7th 2026 at 00:11:10 UTC by @humbleservant for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98316
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

`CfdEngine` latches `degradedMode` when a close leaves the pool's cash `E` below `L`, the larger of the two sides' worst-case payouts, and only the owner can clear it. One trader holding a LONG and a SHORT of about 2.0M USDG notional each, in two accounts, sets the latch with one ordinary close of the winning leg after a 0.84% basket move. Cash then misses the spec's raw `E >= L` test by 121.66 USDG while LP NAV goes up: signed terminal equity, by the accounting spec's own formula, rises by 10,541.89 USDG to 2,023,542.79 USDG, and the terminal deficit is 0. The latch then outlives its cause. Once the trader closes the other leg and withdraws, `E` exceeds `L` by 2,016,153.96 USDG, so the documented clear condition holds, yet LP epoch settlement still reverts, both LP max-withdraw views return 0, matured deposits stay in escrow and matured redemptions stay unfunded, at +1 h, +24 h and +72 h, until the owner calls `clearDegradedMode()`. For that whole time no LP capital can leave the pool. The owner's clear alone, with nothing else changed, turns the Junior max-withdraw view from 0 to 749,226.10 USDG and lets settlement activate 100,000 USDG of matured deposits.

## Vulnerability Details

### 1. The latch compares cash with the worst-case envelope, and only the owner clears it

Close and liquidation settlement call `_enterDegradedModeIfInsolvent` (`CfdEngine.sol` L1351-1368):

```solidity
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

`_maxLiability()` (L1044-1050) returns the larger of `longState.maxProfitAssets` and `shortState.maxProfitAssets`: the payout each side would owe at the far end of its price range (zero or `CAP_PRICE`). The flag has no timer, and the only code that clears it is the owner's call (L658-667):

```solidity
function clearDegradedMode() external onlyOwner {
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

`recapitalizePool` is `onlyOwner` too (L632-634), and a direct USDG transfer does not move `E`, because `HousePool.totalAssets()` returns the smaller of the raw balance and `accountedAssets` (`HousePool.sol` L426-429).

### 2. A hedged book pushes E below L without costing the pool anything

Admission requires `E >= L + B` after every open, and a book with both legs at that bound passes it. In my PoC two accounts of one trader open a LONG and a SHORT in four interleaved slices and, where room is left, a final top-up, about 2.0M USDG notional each at 5% margin, against about 2.01M USDG of LP capital (1,000,000 USDG deposited into each tranche, plus the fixture's tranche seeds and two smaller Junior deposits, less a funded redemption of 1% of the Junior LP's shares). Every open is admitted.

After an 84 bps fall of the basket, the LONG closes in full through the trader's own `commitOrder` call, which a keeper executes. The close pays the LONG out of pool cash while the SHORT's envelope keeps `L` where it was, so `E` ends 121.66 USDG below `L` and the flag latches. A book half that size does not latch at the same move (my control test).

Nothing of value is lost. Measured with the protocol's own NAV (`ACCOUNTING_SPEC.md` "Symmetric exact marked NAV", read through `CfdEngineProtocolLens.getHousePoolInputSnapshot`):

| | Before the move | At the latch |
|---|---|---|
| Terminal LP equity | 2,013,000.90 USDG | 2,023,542.79 USDG |
| Junior, assets per 1,000,000 shares | 998,430.32 | 1,008,831.14 |
| Senior, assets per 1,000,000 shares | 1,001,579.89 | 1,001,579.90 |
| Pool cash (`accountedAssets`) | 2,013,000.90 | 2,006,778.34 |

Signed terminal equity by the spec's formula (`ACCOUNTING_SPEC.md` L163-169) is +2,023,542.79 USDG at the latch, so the terminal deficit is 0. The SHORT's marked loss is collectible from its own margin. What fails is the spec's raw solvency check, which latches at `E < L` (L1205-1211), applied to a payout that would need a basket move of more than 100%, and cash fails it by 121.66 USDG.

### 3. The latch outlives the positions

After the latch the SHORT closes in full as well, and both accounts withdraw their whole balances. `E` is now 2,026,153.96 USDG, `L` is 10,000 USDG (two small unrelated positions stay open), and `HousePool.getFreeAssets()`, which does not apply the flag (`HousePool.sol` L1231-1237), reads 2,016,128.96 USDG. With no owner action, and a fresh keeper mark before each check:

| Surface | +1 h, +24 h, +72 h | After `clearDegradedMode()`, no donation |
|---|---|---|
| `HousePool.getMaxJuniorWithdraw()` / `getMaxSeniorWithdraw()` | 0 / 0 | 749,226.10 USDG / above 0 |
| Keeper `OrderRouter.settleLpEpoch` | reverts `HousePool__DegradedMode` | succeeds |
| Matured Junior deposit, 50,000 USDG | in escrow, not activated | activated |
| Senior deposit for the same epoch, 50,000 USDG | in escrow, not activated | activated |
| Matured Junior redemption | unfunded | funded |
| `TrancheVault.maxRequestDeposit`, both tranches | 0 / 0 | above 0 |
| New open (`commitOrder`) | reverts `OrderRouter__DegradedMode` | executes |

The settlement path calls `_requireWithdrawalsLive` (`HousePool.sol` L832), which reverts on the flag (L1864-1869). Both max-withdraw views return 0 through `HousePoolFreshnessLib.withdrawalsLive` (L43-45; `HousePool.sol` L1262-1284). Only the flag holds these shut: the owner's clear, with nothing else changed, reopens every one of them.

### 4. The LPs have no way out while the flag is set

- A matured Junior deposit cannot be cancelled (`TrancheVault.sol` L851-859; the same rule applies without the latch), so activation in a settlement is its only way out of escrow, and the latch blocks every settlement.
- A Senior deposit is committed from the request by design and waits on the same activation (a pauser's LP settlement hold would make it cancellable; no LP can set one).
- A matured redemption can no longer be cancelled, and only a settlement funds it.
- No one but the owner can clear the flag or recapitalize, and a donation does not count toward `E` (section 1).

Traders are in a different position. Closes and liquidations stay live, so trader collateral is not frozen: a trader with an open position has to close before withdrawing free collateral, and then can. Already-funded LP claims stay claimable from vault escrow.

### 5. Why the documented containment does not cover this

The design documents describe the latch as containment for raw insolvency: "the protocol contains raw insolvency with `degradedMode` instead" (`ACCOUNTING_SPEC.md` L37-38), "`degradedMode` is a containment latch" (L1203), and invariant 12, "Closes and liquidations may consume the buffer and complete into degraded mode" (L1294-1295). `EMERGENCY_RESPONSE_GUIDE.md` L45-47 adds that the latch blocks "all LP epoch settlement" and that the owner can clear it "once raw `E >= L`" (also `ACCOUNTING_SPEC.md` L1212 and `SECURITY.md` L942-943). So a close may set the flag, and the flag stays until the owner clears it. Two things set this case apart:

- **An untrusted trader sets the latch, and LP NAV rises when it fires.** The trigger is a book with no net exposure and a routine move. Cash misses the raw `E >= L` test by 121.66 USDG at the same moment signed terminal equity rises by 10,541.89 USDG, so the LPs are gaining, not losing, when the latch fires.
- **The clear condition then holds by more than 2M USDG, and release still waits on the owner with no deadline.** The spec allows "owner action to clear the mode after solvency is genuinely restored" (`ACCOUNTING_SPEC.md` L1222). After the unwind `E` exceeds `L` by 2,016,153.96 USDG, and every LP surface stays shut at +72 h. The emergency guide lists "unexpected degraded mode or insolvency whose cause is not promptly understood" as a trigger for the risk-off plus LP-entry pause (`EMERGENCY_RESPONSE_GUIDE.md` L111-119), and says governance "should recover only after" the root cause "is identified, fixed or isolated, and independently reviewed" (L221-223). The program's rules state: "No detection or response deadline is assumed unless explicitly documented for the competition", and "The existence of monitoring or pause functionality alone does not justify a downgrade."

The latch is not one of the known items either. Cecuro LOW-22 added the owner's `recapitalizePool`, which "does not automatically clear degraded mode", and LOW-73 concerns protection creation while degraded. Neither covers a trader setting the latch with no terminal deficit, or the latch outliving its cause, and the "Known Limitations" section of `SECURITY.md` (L1328-1457) does not mention degraded mode.

### 6. Cost and conditions

- The round costs the trader 15,551.66 USDG in my PoC, before any wait for the move (deposited minus withdrawn, across both accounts). Most of it lands with the LPs: terminal LP equity rises by 10,541.89 USDG at the latch and by 2,534.46 USDG more on the unwind. The LPs end richer. What they lose is the ability to move their capital.
- The smallest latching move in the PoC is 84 bps. The fixture's Pyth updates carry zero confidence, so the release adverse-confidence multiplier (2,500 bps of the confidence interval) never shifts an execution price there. At the release confidence cap of 10 bps, a LONG close executes up to about 2.5 bps worse (`PletherOracle.sol` L869, L905-917), so the latching move would be up to about 2.5 bps larger. The latch depends on the cumulative move from the book's entry price (`CfdMath.sol` L59-72), not on a one-day move. On daily fixings of the basket's six currencies from November 2023 to October 2026, from a random start day the basket first fell 0.84% or more below that day's level within 19 trading days at the median, and the median is the same at 0.865%. Both figures hold the threshold fixed. A rise works the same way with the SHORT as the winning leg (an exact mirror at my PoC's 1.00 entry against the 2.00 cap); my PoC runs the fall, so I count falls only. The threshold does not stay fixed while the book waits: the closing leg's accrued carry is realized at the close and raises the move needed, so the real wait is longer than that median, and LP flows move the threshold as well. I did not model the wait.
- My PoC funds each account with 150,000 USDG (5% margin per leg), and the book is sized to the pool's admission limit.
- The owner has two ways out. After the unwind, `clearDegradedMode()` alone, with no donation, reopens every surface in the section 3 table. At the latch, three owner calls (`recapitalizePool` for 121.66 USDG, `clearDegradedMode()`, and `assignUnassignedAssets`, which returns the donation as Junior shares worth 121.66 USDG, for a net cost of about 0) let the next settlement activate the matured Junior deposit, the one surface my PoC checks at that point. Recapitalizing and clearing alone are not enough: the settlement then reverts `HousePool__NoLpEpochProgress`. Until the owner acts, the freeze has no end.

### 7. Suggested fix

Let anyone clear the flag once `E >= L`, or clear it in the next settlement that finds `E >= L`, so LP settlement resumes as soon as the documented clear condition holds. To keep a hedged book from setting the latch at all, the latch could test the protocol's terminal NAV deficit instead of cash below the worst-case envelope.

## Impact Details

Impact: "Temporary freezing of funds for at least 1 hour". While the flag is set, no LP capital leaves the pool: settlement reverts and both max-withdraw views return 0, matured deposits (100,000 USDG in the PoC) can neither activate nor be cancelled, and matured redemptions stay unfunded. In the PoC that lasts 72 hours with the trader gone and `E` 2,016,153.96 USDG above `L`, and it ends only when the owner clears the flag.

The code is not deployed, so I understand the freezing duration is scaled to the minimum, and I selected the 1-hour line. The freeze in the PoC has no built-in end, and the program assumes no response deadline.

## References

- Frozen competition commit (the code is not deployed): https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Latch on close and liquidation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1368
- Worst-case envelope `L`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1044-L1050
- Owner-only clear: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667
- Owner-only recapitalization: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L634
- Pool assets capped at `accountedAssets`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L426-L429
- Settlement gate: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L832 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1869
- Max-withdraw views: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1262-L1284 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolFreshnessLib.sol#L43-L45
- Capacity view without the flag: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1231-L1237
- Matured deposit cancellation rule: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L859
- Execution-price confidence shift: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L905-L917
- Position PnL against the entry cost: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L59-L72
- `packages/perps/ACCOUNTING_SPEC.md` L37-38, L158-174 (NAV and terminal deficit), L1201-1238 (Degraded Mode), L1294-1295 (invariant 12)
- `packages/perps/SECURITY.md` L942-943 (clear condition), L1328-1457 (Known Limitations)
- `packages/perps/EMERGENCY_RESPONSE_GUIDE.md` L45-47, L111-119, L221-223
- `packages/perps/DEPLOYMENT.md` L146-147 (confidence settings), L156 (tranche seeds)
- Known items checked: Cecuro LOW-22 (owner recapitalization path) and LOW-73 (protection creation while degraded).

## Proof of Concept

## Proof of Concept

Save the file below as `packages/perps/test/poc/DegradedLatchFromHedgedClose.t.sol` in a checkout of commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It extends the repo's `BasePerpTest`, so the oracle, execution sidecars, engine, clearinghouse, HousePool and both TrancheVaults are the real contracts. The Router is the fixture's `LegacyOrderRouterHarness`, which is the production `OrderRouter` plus a test-only scalar commit overload that my PoC never calls, and the token and Pyth are the fixture's `MockSettlementToken` (6 decimals) and `MockPyth`. It installs the release risk configuration (constructor values through the fixture's risk-parameter hook, the rest through the owner's documented 72-hour setters) and reads those values back. Three fixture defaults stay: a basket of two equal-weight mock feeds that move together in place of the six-currency basket, tranche seeds of 1,000 USDG each (the release seed is 1 USDG), and Pyth updates with zero confidence (section 6). Every trade is the trader's own V3 `commitOrder` call, executed by an unprivileged keeper with a Pyth payload, and every LP action is the LP's own call. The owner appears only in `setUp`, in one reverting `clearDegradedMode()` probe at the latch in test 2, in `test_ownerExits` and in the reopening checks after the freeze. The fixture's LP funding in `setUp` also writes the engine mark through a Router prank before any trade.

The four tests:

1. `test_control_halfBookDoesNotLatch`: the same move on a book half the size does not latch, and LP settlement (both matured deposits activate), a trader withdrawal and a new open still work.
2. `test_hedgedCloseLatches_lpNavIntact`: one close latches the flag with `E < L`, the NAV figures above, and each frozen surface with its exact revert or zero read.
3. `test_walkAway_latchOutlivesThePositions`: both legs closed and withdrawn, every surface still shut at +1 h, +24 h and +72 h, then reopened by the owner's clear with no donation.
4. `test_ownerExits`: the owner's clear reverts before any donation, a recapitalization one atom short still reverts, and after the three owner calls the next settlement activates the matured Junior deposit.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

/// @notice Two accounts of one trader build a hedged book that every admission gate accepts (E >= L + B after each
///         leg). After a basket move of 0.84%, one ordinary close of the profitable leg pays out cash while the other
///         leg's envelope keeps L, so `_applyClose` latches `CfdEngine.degradedMode` (CfdEngine.sol:1351-1369). The
///         flag has no timer and only the owner clears it (CfdEngine.sol:658-667). Release parameters; production
///         Router path only: trader-signed V3 `commitOrder`, unprivileged keeper `executeOrder` with Pyth payloads.
/// @dev A LONG profits when the basket falls, a SHORT when it rises (CfdMath.sol:61-72). Amounts are 6-decimal USDG
///      atoms unless labelled otherwise.
contract DegradedLatchFromHedgedCloseTest is BasePerpTest {
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant LOT = 1e20; // CfdTypes.SIZE_QUANTUM: 100 synthetic tokens
    uint256 internal constant BP = 1e4; // one basis point of the 1.00 basket in 8-decimal price units
    uint256 internal constant ROUNDS = 4; // interleaved slices per side keep skew far inside 0.4 x depth
    uint256 internal constant LEG_MARGIN_BPS = 500; // 5% per slice, above the 3% FAD requirement after the move
    uint256 internal constant BUILD_SAFETY = 5_000e6; // the interleaved rounds stop this far under the bound
    uint256 internal constant LEG_DEPOSIT = 150_000e6;
    uint256 internal constant OTHER_LONG_SIZE = 10_000e18; // 10,000 USDG notional at 1.00
    uint256 internal constant OTHER_SHORT_SIZE = 2_000e18; // above the 1,000 USDG release minimum
    uint256 internal constant CLAIMABLE_DEPOSIT = 20_000e6;
    uint256 internal constant EARLY_DEPOSIT = 1_000e6;
    uint256 internal constant PENDING_DEPOSIT = 50_000e6; // each of the Junior and Senior requests made at 10:00
    uint256 internal constant PROBE_WITHDRAW = 500e6;
    uint256 internal constant NAV_SHARES = 1e15; // 1,000,000 tranche shares (9 decimals)
    /// @dev Monday 2024-03-11 06:00 UTC, one week after the fixture's SETUP_TIMESTAMP (also Monday 06:00 UTC).
    uint256 internal constant MONDAY = 1_710_136_800;
    /// @dev 10:59:30 UTC, thirty seconds before the 11:00 LP epoch boundary.
    uint256 internal constant MOVE_TIME = MONDAY + 4 hours + 59 minutes + 30 seconds;

    struct LpNav {
        uint256 equity; // terminal LP equity: physical assets - trader claims + collectible terminal price delta
        uint256 juniorPerMillionShares; // TrancheVault.convertToAssets(1,000,000 shares)
        uint256 seniorPerMillionShares;
        uint256 accountedAssets; // HousePool.accountedAssets()
    }

    address internal hedgeLong; // LONG leg of the hedged book
    address internal hedgeShort; // SHORT leg of the same book, a second account of the same trader
    address internal otherLong; // unrelated LONG trader with free collateral
    address internal otherShort; // unrelated SHORT trader posting thin margin
    address internal newTrader; // fresh account that tries to open
    address internal juniorLp;
    address internal seniorLp;
    address internal lpClaimable; // Junior deposit activated before any trade, claimed later
    address internal lpEarly; // Junior deposit that matures at 10:00 with the book open
    address internal lpPending; // Junior deposit requested at 10:00, matures at 11:00
    address internal lpSeniorPending; // Senior deposit requested at 10:00
    address internal keeper; // permissionless keeper EOA

    uint256 internal longSize;
    uint256 internal shortSize;
    uint256 internal fundedRedeemId;
    uint256 internal claimableDepositId;
    uint256 internal earlyDepositId;
    uint256 internal pendingDepositId;
    uint256 internal seniorPendingId;
    uint256 internal pendingRedeemId;
    uint256 internal pendingRedeemShares;
    uint256 internal latchedAt;
    uint256 internal movedPrice;

    // ---------------- Release configuration ----------------

    /// @dev Release risk parameters, passed to the CfdEngine constructor by
    ///      script/DeployPerpsArbitrumSepolia.s.sol:434-447 (DEPLOYMENT.md:131-141). executionFeeBps 4 and
    ///      settlementBufferBps 25 are the engine defaults (CfdEngine.sol:147, 153); frozenCloseSpreadBps 50 is the
    ///      fixture constructor argument (DEPLOYMENT.md:132).
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

    /// @dev Both tranches are funded by named LPs through the fixture's asynchronous deposit path.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        hedgeLong = makeAddr("hedgeLong");
        hedgeShort = makeAddr("hedgeShort");
        otherLong = makeAddr("otherLong");
        otherShort = makeAddr("otherShort");
        newTrader = makeAddr("newTrader");
        juniorLp = makeAddr("juniorLp");
        seniorLp = makeAddr("seniorLp");
        lpClaimable = makeAddr("lpClaimable");
        lpEarly = makeAddr("lpEarly");
        lpPending = makeAddr("lpPending");
        lpSeniorPending = makeAddr("lpSeniorPending");
        keeper = makeAddr("keeper");

        // Release values the shared fixture leaves at test defaults (DEPLOYMENT.md:147-148, 154-155), installed
        // through the owner's documented 72-hour setters, which reach the same end state as the release constructors.
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerConfig);
        uint256 activation = pool.poolConfigActivationTime();
        if (routerAdmin.routerConfigActivationTime() > activation) {
            activation = routerAdmin.routerConfigActivationTime();
        }
        vm.warp(activation);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();

        vm.warp(MONDAY);
        assertFalse(engine.isFadWindow(), "Monday is outside FAD");
        assertFalse(engine.isOracleFrozen(), "Monday is outside the frozen window");
        _fundJunior(juniorLp, 1_000_000e6);
        _fundSenior(seniorLp, 1_000_000e6);
        _assertReleaseConfig();
    }

    /// @dev Reads the live configuration back from the deployed contracts and pins every release value.
    function _assertReleaseConfig() internal view {
        CfdTypes.RiskParams memory rp = _engineRiskConfig().riskParams;
        assertEq(rp.vpiFactor, 0.01e18, "vpiFactor");
        assertEq(rp.maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(rp.maintMarginBps, 10, "maintMarginBps");
        assertEq(rp.initMarginBps, 20, "initMarginBps");
        assertEq(rp.fadMarginBps, 300, "fadMarginBps");
        assertEq(rp.baseCarryBps, 500, "baseCarryBps");
        assertEq(rp.minBountyAssets, 1e6, "minBountyAssets");
        assertEq(rp.bountyBps, 10, "bountyBps");
        assertEq(rp.keeperShareBps, 5000, "keeperShareBps");
        assertEq(rp.protocolShareBps, 0, "protocolShareBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "frozenCloseSpreadBps");
        assertEq(engine.fadRunwaySeconds(), 1 hours, "fadRunwaySeconds");
        assertEq(engine.CAP_PRICE(), 2e8, "CAP_PRICE");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "minOpenNotionalAssets");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "adverseConfidenceMultiplierBps");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "basketMaxConfidenceRatioBps");
        assertEq(router.maxPendingOrders(), 5, "maxPendingOrders");
        assertEq(router.positionProtectionTriggerBountyAssets(), 200_000, "positionProtectionTriggerBountyAssets");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), 8000, "maxSeniorShareBps");
    }

    // ---------------- Reads ----------------

    /// @dev via_ir folds repeated TIMESTAMP reads across vm.warp; the cheatcode read is authoritative.
    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _warpTo(uint256 timestamp) internal {
        if (_now() < timestamp) {
            vm.warp(timestamp);
        }
    }

    /// @dev E exactly as `_availableCashForFreshPoolPayouts` computes it (CfdEngine.sol:865-871).
    function _e() internal view returns (uint256) {
        uint256 physical = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return physical > claims ? physical - claims : 0;
    }

    /// @dev B = ceil(L * settlementBufferBps / 10_000) (SECURITY.md:928-932).
    function _bufferTarget() internal view returns (uint256) {
        return (_maxLiability() * engine.settlementBufferBps() + 9999) / 10_000;
    }

    /// @dev LP NAV as the pool prices LP entry and exit (ACCOUNTING_SPEC.md "Symmetric exact marked NAV"), each
    ///      tranche's share price, and the pool's accounted cash. Callers refresh the mark first: the tranche views
    ///      apply the mark only when it is fresh.
    function _lpNav(string memory label) internal returns (LpNav memory nav) {
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory s =
            engineProtocolLens.getHousePoolInputSnapshot(pool.markStalenessLimit());
        int256 equity = int256(s.physicalAssets) - int256(s.traderClaimBalanceAssets) + s.terminalLpPriceDeltaAssets;
        assertGt(equity, 0, "terminal LP equity is positive");
        nav.equity = uint256(equity);
        nav.juniorPerMillionShares = juniorVault.convertToAssets(NAV_SHARES);
        nav.seniorPerMillionShares = seniorVault.convertToAssets(NAV_SHARES);
        nav.accountedAssets = pool.accountedAssets();
        emit log_named_uint(string.concat(label, ": terminal LP equity"), nav.equity);
        emit log_named_uint(string.concat(label, ": Junior assets per 1,000,000 shares"), nav.juniorPerMillionShares);
        emit log_named_uint(string.concat(label, ": Senior assets per 1,000,000 shares"), nav.seniorPerMillionShares);
        emit log_named_uint(string.concat(label, ": pool accountedAssets"), nav.accountedAssets);
    }

    // ---------------- Production Router path ----------------

    function _sizeFor(uint256 notionalAssets) internal pure returns (uint256 size) {
        size = notionalAssets * 1e20 / PRICE;
        size -= size % LOT;
    }

    function _marginFor(uint256 size) internal pure returns (uint256) {
        return (size / 1e12) * LEG_MARGIN_BPS / 10_000;
    }

    /// @dev A V3 request with the trader's loosest bounds; the protocol's own checks are untouched.
    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        internal
        view
        returns (OrderV3Types.OrderRequest memory request)
    {
        uint256 window = router.maxExecutionWindowSeconds();
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(_now() + window),
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
    }

    /// @dev Trader commits through `OrderRouter.commitOrder(OrderRequest)`; an unprivileged keeper executes with a
    ///      Pyth payload. The fixture's test-only scalar commit adapter is never used.
    function _commitAndExecute(address account, OrderV3Types.OrderRequest memory request, uint256 price) internal {
        vm.prank(account);
        uint64 orderId = OrderRouter(address(router)).commitOrder(request);
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    /// @dev Each open is first screened by the two gates the Router consults at commit: HousePool.canIncreaseRisk()
    ///      and CfdEngineLens.previewOpenRevertCode at the cached mark (0 = OK).
    function _routerOpen(address account, CfdTypes.Side side, uint256 size, uint256 margin, uint256 price) internal {
        assertTrue(pool.canIncreaseRisk(), "canIncreaseRisk admits the leg");
        uint8 code = engineLens.previewOpenRevertCode(account, side, size, margin, engine.lastMarkPrice());
        assertEq(uint256(code), 0, "previewOpenRevertCode admits the leg");
        _commitAndExecute(account, _request(side, size, margin, false), price);
    }

    function _routerClose(address account, CfdTypes.Side side, uint256 size, uint256 price) internal {
        _commitAndExecute(account, _request(side, size, 0, true), price);
    }

    function _keeperMark(uint256 price) internal {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(keeper);
        router.updateMarkPrice(updateData);
    }

    function _keeperSettleLp(uint256 price) internal {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(keeper);
        router.settleLpEpoch(updateData);
    }

    function _requestDeposit(TrancheVault vault, address who, uint256 amount) internal returns (uint256 requestId) {
        settlementToken.mint(who, amount);
        vm.startPrank(who);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, who, who);
        vm.stopPrank();
    }

    // ---------------- Scenario ----------------

    /// @dev Monday 08:00 to 09:00 UTC: LP claims funded before any trade, two unrelated traders with live positions,
    ///      and a Junior deposit that matures at 10:00 with the book open.
    function _preTradeState() internal {
        uint256 shares = juniorVault.balanceOf(juniorLp) / 100;
        vm.prank(juniorLp);
        fundedRedeemId = juniorVault.requestRedeem(shares, juniorLp, juniorLp);
        claimableDepositId = _requestDeposit(juniorVault, lpClaimable, CLAIMABLE_DEPOSIT);
        _warpTo(pool.lpEpochStart(fundedRedeemId));
        _keeperSettleLp(PRICE);
        assertGt(juniorVault.claimableRedeemAssets(fundedRedeemId, juniorLp), 0, "redemption funded before any trade");
        assertEq(juniorVault.claimableDepositRequest(claimableDepositId, lpClaimable), CLAIMABLE_DEPOSIT, "activated");

        _fundTrader(otherLong, 5_000e6);
        _routerOpen(otherLong, CfdTypes.Side.LONG, OTHER_LONG_SIZE, 1_000e6, PRICE);
        _fundTrader(otherShort, 12e6);
        _routerOpen(otherShort, CfdTypes.Side.SHORT, OTHER_SHORT_SIZE, 10e6, PRICE);
        _fundTrader(newTrader, 5_000e6);
        earlyDepositId = _requestDeposit(juniorVault, lpEarly, EARLY_DEPOSIT);
    }

    /// @dev Largest further size whose engine preview is OK: the engine's own admission edge for this account.
    function _maxAdmissibleSize(address account, CfdTypes.Side side) internal view returns (uint256) {
        uint256 lo;
        uint256 hi = 2000; // lots: 200,000 USDG notional, far above the room the rounds leave
        uint256 price = engine.lastMarkPrice();
        while (lo < hi) {
            uint256 mid = (lo + hi + 1) / 2;
            if (engineLens.previewOpenRevertCode(account, side, mid * LOT, _marginFor(mid * LOT), price) == 0) {
                lo = mid;
            } else {
                hi = mid - 1;
            }
        }
        return lo * LOT;
    }

    /// @param fullBook true: both envelopes filled to the admission edge; false (control): half of that.
    function _buildBook(bool fullBook) internal {
        _fundTrader(hedgeLong, LEG_DEPOSIT);
        _fundTrader(hedgeShort, LEG_DEPOSIT);
        uint256 bound = _e() * 10_000 / (10_000 + engine.settlementBufferBps()) - BUILD_SAFETY;
        if (!fullBook) {
            bound = bound / 2;
        }
        for (uint256 round; round < ROUNDS; ++round) {
            uint256 remaining = ROUNDS - round;
            uint256 longSlice = _sizeFor((bound - _sideMaxProfit(CfdTypes.Side.LONG)) / remaining);
            _routerOpen(hedgeLong, CfdTypes.Side.LONG, longSlice, _marginFor(longSlice), PRICE);
            longSize += longSlice;
            uint256 shortSlice = _sizeFor((bound - _sideMaxProfit(CfdTypes.Side.SHORT)) / remaining);
            _routerOpen(hedgeShort, CfdTypes.Side.SHORT, shortSlice, _marginFor(shortSlice), PRICE);
            shortSize += shortSlice;
        }
        if (fullBook) {
            uint256 minSize = router.minOpenNotionalAssets() * 1e20 / PRICE;
            uint256 extraLong = _maxAdmissibleSize(hedgeLong, CfdTypes.Side.LONG);
            if (extraLong >= minSize) {
                _routerOpen(hedgeLong, CfdTypes.Side.LONG, extraLong, _marginFor(extraLong), PRICE);
                longSize += extraLong;
            }
            uint256 extraShort = _maxAdmissibleSize(hedgeShort, CfdTypes.Side.SHORT);
            if (extraShort >= minSize) {
                _routerOpen(hedgeShort, CfdTypes.Side.SHORT, extraShort, _marginFor(extraShort), PRICE);
                shortSize += extraShort;
            }
        }
        assertFalse(engine.degradedMode(), "the admitted book is not degraded");
        assertGe(_e(), _maxLiability() + _bufferTarget(), "every leg was admitted under E >= L + B");
    }

    /// @dev Monday 08:00 to 10:59:30 UTC, with pre-latch controls on the open book.
    function _scenarioToMove(bool fullBook) internal {
        _preTradeState();
        _buildBook(fullBook);
        vm.prank(otherLong);
        clearinghouse.withdraw(otherLong, PROBE_WITHDRAW); // free collateral of a live position withdraws
        assertGt(juniorVault.maxRequestDeposit(newTrader), 0, "junior entry open before the latch");
        assertGt(seniorVault.maxRequestDeposit(newTrader), 0, "senior entry open before the latch");

        // 10:00: a matured Junior deposit activates through the Router with the book open.
        _warpTo(pool.lpEpochStart(earlyDepositId));
        _keeperSettleLp(PRICE);
        assertEq(juniorVault.claimableDepositRequest(earlyDepositId, lpEarly), EARLY_DEPOSIT, "activates unlatched");

        // 10:00: requests that mature at 11:00. Senior requests are firm from the start (TrancheVault.sol:851-858).
        pendingDepositId = _requestDeposit(juniorVault, lpPending, PENDING_DEPOSIT);
        seniorPendingId = _requestDeposit(seniorVault, lpSeniorPending, PENDING_DEPOSIT);
        assertEq(seniorPendingId, pendingDepositId, "both requests share the 11:00 epoch");
        pendingRedeemShares = juniorVault.balanceOf(juniorLp) / 100;
        vm.prank(juniorLp);
        pendingRedeemId = juniorVault.requestRedeem(pendingRedeemShares, juniorLp, juniorLp);
        vm.prank(lpSeniorPending);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(seniorPendingId, lpSeniorPending, lpSeniorPending);

        _warpTo(MOVE_TIME);
        assertFalse(engine.isFadWindow(), "no FAD at the move");
        assertFalse(engine.isOracleFrozen(), "no frozen window at the move");
    }

    /// @dev Sets `movedPrice` to the smallest whole-basis-point fall at which the engine's own preview of the full
    ///      profitable close reports `triggersDegradedMode`, and checks that one basis point less does not.
    function _computeMove() internal {
        uint256 lo;
        uint256 hi = 300 * BP;
        assertTrue(engineLens.previewClose(hedgeLong, longSize, PRICE - hi).triggersDegradedMode, "3% latches");
        while (lo < hi) {
            uint256 mid = (lo + hi) / 2;
            if (engineLens.previewClose(hedgeLong, longSize, PRICE - mid).triggersDegradedMode) {
                hi = mid;
            } else {
                lo = mid + 1;
            }
        }
        movedPrice = PRICE - ((lo + BP - 1) / BP) * BP;
        assertTrue(engineLens.previewClose(hedgeLong, longSize, movedPrice).triggersDegradedMode, "move latches");
        assertFalse(
            engineLens.previewClose(hedgeLong, longSize, movedPrice + BP).triggersDegradedMode, "1 bp less does not"
        );
        emit log_named_uint("smallest latching fall of the basket, whole basis points", (PRICE - movedPrice) / BP);
    }

    /// @dev Full book, smallest latching move, profitable LONG closed in full through the Router.
    function _scenarioThroughLatch() internal {
        _scenarioToMove(true);
        _computeMove();
        _keeperMark(movedPrice);
        _routerClose(hedgeLong, CfdTypes.Side.LONG, longSize, movedPrice);
        latchedAt = _now();
        assertTrue(engine.degradedMode(), "the profitable close latched degraded mode");
        assertLt(_e(), _maxLiability(), "E < L");
    }

    /// @dev Every surface the latch gates, with its exact revert or zero read. Callers refresh the mark first.
    function _assertSurfacesFrozen() internal {
        assertTrue(engine.degradedMode(), "latched");
        assertFalse(pool.isWithdrawalLive(), "pool withdrawals not live");
        assertEq(pool.getMaxJuniorWithdraw(), 0, "junior max withdraw");
        assertEq(pool.getMaxSeniorWithdraw(), 0, "senior max withdraw");
        assertEq(juniorVault.maxRequestDeposit(newTrader), 0, "junior entry closed");
        assertEq(seniorVault.maxRequestDeposit(newTrader), 0, "senior entry closed");

        bytes[] memory updateData = _mockPythUpdateData(movedPrice);
        vm.prank(keeper);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        router.settleLpEpoch(updateData);

        assertGe(juniorVault.currentLpEpoch(), pendingDepositId, "the Junior deposit has matured");
        assertEq(juniorVault.pendingDepositRequest(pendingDepositId, lpPending), PENDING_DEPOSIT, "still in escrow");
        assertEq(juniorVault.claimableDepositRequest(pendingDepositId, lpPending), 0, "not activated");
        vm.prank(lpPending);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(pendingDepositId, lpPending, lpPending);
        assertEq(seniorVault.pendingDepositRequest(seniorPendingId, lpSeniorPending), PENDING_DEPOSIT, "senior escrow");
        assertEq(seniorVault.claimableDepositRequest(seniorPendingId, lpSeniorPending), 0, "senior not activated");
        vm.prank(lpSeniorPending);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(seniorPendingId, lpSeniorPending, lpSeniorPending);
        vm.prank(juniorLp);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(pendingRedeemId, juniorLp, juniorLp);
        assertEq(juniorVault.claimableRedeemRequest(pendingRedeemId, juniorLp), 0, "matured redemption unfunded");

        vm.prank(otherLong);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(otherLong, PROBE_WITHDRAW);

        OrderV3Types.OrderRequest memory request = _request(CfdTypes.Side.LONG, 10_000e18, 1_000e6, false);
        vm.prank(newTrader);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__DegradedMode.selector);
        OrderRouter(address(router)).commitOrder(request);
    }

    /// @dev Walk-away checkpoint: fresh mark, cash far above the envelope and its buffer, every surface still shut.
    function _assertLatchAloneHolds() internal {
        _keeperMark(movedPrice);
        assertGe(_e(), _maxLiability() + _bufferTarget(), "E >= L + B: cash is not what blocks");
        assertGt(_e(), 100 * _maxLiability(), "E >> L");
        assertGt(
            pool.getFreeAssets(),
            juniorVault.convertToAssets(pendingRedeemShares),
            "the ungated capacity view would fund the matured redemption in full"
        );
        _assertSurfacesFrozen();
        string memory label = string.concat("+", vm.toString((_now() - latchedAt) / 1 hours), " h, no owner action");
        emit log_named_uint(string.concat(label, ", every surface shut: E - L"), _e() - _maxLiability());
    }

    /// @dev The owner clears with no donation; each surface is then probed with nothing else changed.
    function _clearAndAssertReopened() internal {
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner clear succeeds with no donation");
        _keeperMark(movedPrice);
        assertTrue(pool.isWithdrawalLive(), "pool withdrawals live");
        uint256 juniorMax = pool.getMaxJuniorWithdraw();
        assertGt(juniorMax, 0, "junior max withdraw reopens");
        assertGt(pool.getMaxSeniorWithdraw(), 0, "senior max withdraw reopens");
        assertGt(juniorVault.maxRequestDeposit(newTrader), 0, "junior entry reopens");
        assertGt(seniorVault.maxRequestDeposit(newTrader), 0, "senior entry reopens");
        _keeperSettleLp(movedPrice);
        assertEq(juniorVault.claimableDepositRequest(pendingDepositId, lpPending), PENDING_DEPOSIT, "Junior activates");
        assertEq(
            seniorVault.claimableDepositRequest(seniorPendingId, lpSeniorPending), PENDING_DEPOSIT, "Senior activates"
        );
        assertEq(
            juniorVault.claimableRedeemRequest(pendingRedeemId, juniorLp), pendingRedeemShares, "redemption funded"
        );
        vm.prank(otherLong);
        clearinghouse.withdraw(otherLong, PROBE_WITHDRAW);
        _routerOpen(newTrader, CfdTypes.Side.LONG, 10_000e18, 1_000e6, movedPrice);
        string memory label = string.concat("+", vm.toString((_now() - latchedAt) / 1 hours), " h, owner clear");
        emit log_named_uint(string.concat(label, " (no donation), all reopened: maxJuniorWithdraw"), juniorMax);
    }

    // ---------------- Tests ----------------

    /// @notice Control: the same move applied to a half-size book does not latch, and the same surfaces stay live.
    function test_control_halfBookDoesNotLatch() public {
        uint256 snap = vm.snapshotState();
        _scenarioToMove(true);
        _computeMove();
        uint256 fullBookMovedPrice = movedPrice; // measured on the full book, then rolled back
        vm.revertToState(snap);
        assertEq(longSize, 0, "snapshot restored the book");

        _scenarioToMove(false);
        movedPrice = fullBookMovedPrice;
        int256 headroomBefore = int256(_e()) - int256(_maxLiability());
        uint256 eBefore = _e();
        _keeperMark(movedPrice);
        _routerClose(hedgeLong, CfdTypes.Side.LONG, longSize, movedPrice);
        int256 eConsumed = int256(eBefore) - int256(_e());
        assertGt(eConsumed, 0, "the close was paid");
        assertLt(eConsumed, headroomBefore, "payout stayed inside the headroom");
        assertFalse(engine.degradedMode(), "no latch with headroom");

        // 11:00: the matured Junior deposit cannot be cancelled here either (maturity rule), and it activates.
        _warpTo(pool.lpEpochStart(pendingDepositId));
        vm.prank(lpPending);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(pendingDepositId, lpPending, lpPending);
        _keeperSettleLp(movedPrice);
        assertEq(juniorVault.claimableDepositRequest(pendingDepositId, lpPending), PENDING_DEPOSIT, "Junior activated");
        assertEq(
            seniorVault.claimableDepositRequest(seniorPendingId, lpSeniorPending), PENDING_DEPOSIT, "Senior activated"
        );
        vm.prank(otherLong);
        clearinghouse.withdraw(otherLong, PROBE_WITHDRAW);
        _routerOpen(newTrader, CfdTypes.Side.LONG, 10_000e18, 1_000e6, movedPrice);
    }

    /// @notice Full book at the admission edge, smallest latching move, profitable leg closed through the Router:
    ///         E drops below L while LP NAV rises, and every surface the latch gates is shut.
    function test_hedgedCloseLatches_lpNavIntact() public {
        _scenarioToMove(true);
        _computeMove();

        // Executed control: one basis point less on the same book does not latch.
        uint256 snap = vm.snapshotState();
        _keeperMark(movedPrice + BP);
        _routerClose(hedgeLong, CfdTypes.Side.LONG, longSize, movedPrice + BP);
        assertFalse(engine.degradedMode(), "one basis point less does not latch (executed)");
        vm.revertToState(snap);

        _keeperMark(PRICE);
        LpNav memory beforeMove = _lpNav("before the move");
        _keeperMark(movedPrice);
        _routerClose(hedgeLong, CfdTypes.Side.LONG, longSize, movedPrice);
        latchedAt = _now();
        assertTrue(engine.degradedMode(), "an ordinary profitable close latched degraded mode");
        assertLt(_e(), _maxLiability(), "E < L");
        emit log_named_uint("L - E after the latching close", _maxLiability() - _e());

        // The latch is a cash-versus-envelope test: cash left the pool, LP value did not.
        _keeperMark(movedPrice);
        LpNav memory atLatch = _lpNav("at the latch");
        assertLt(atLatch.accountedAssets, beforeMove.accountedAssets, "cash paid out of the pool");
        assertGe(atLatch.equity, beforeMove.equity, "terminal LP equity does not fall");
        assertGt(atLatch.juniorPerMillionShares, beforeMove.juniorPerMillionShares, "Junior share price rises");
        assertGe(atLatch.seniorPerMillionShares, beforeMove.seniorPerMillionShares, "Senior share price holds");
        assertApproxEqRel(atLatch.seniorPerMillionShares, beforeMove.seniorPerMillionShares, 1e12, "Senior, 1 ppm");
        assertEq(pool.terminalDeficitAssets(), 0, "no terminal deficit");

        // 11:00: the 10:00 requests mature inside the latch.
        _warpTo(pool.lpEpochStart(pendingDepositId));
        _keeperMark(movedPrice);
        assertEq(
            uint256(engineLens.previewOpenRevertCode(newTrader, CfdTypes.Side.LONG, 10_000e18, 1_000e6, movedPrice)),
            uint256(uint8(CfdEnginePlanTypes.OpenRevertCode.DEGRADED_MODE)),
            "planner answers DEGRADED_MODE"
        );
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        _assertSurfacesFrozen();

        // Still live while latched: funded LP claims, closes, flat-account withdrawals, keeper liquidation.
        uint256 shares = juniorVault.claimableRedeemRequest(fundedRedeemId, juniorLp);
        uint256 usdgBefore = settlementToken.balanceOf(juniorLp);
        vm.prank(juniorLp);
        uint256 redeemed = juniorVault.claimRedeem(fundedRedeemId, shares, juniorLp, juniorLp);
        assertGt(redeemed, 0, "funded redemption claim");
        assertEq(settlementToken.balanceOf(juniorLp) - usdgBefore, redeemed, "paid from vault escrow");
        vm.prank(lpClaimable);
        assertGt(juniorVault.claimDeposit(claimableDepositId, CLAIMABLE_DEPOSIT, lpClaimable, lpClaimable), 0);
        vm.prank(lpEarly);
        assertGt(juniorVault.claimDeposit(earlyDepositId, EARLY_DEPOSIT, lpEarly, lpEarly), 0);
        _routerClose(otherLong, CfdTypes.Side.LONG, OTHER_LONG_SIZE, movedPrice);
        uint256 otherOut = clearinghouse.balanceAssets(otherLong);
        vm.prank(otherLong);
        clearinghouse.withdraw(otherLong, otherOut);
        assertTrue(engineLens.isLiquidatableAt(otherShort, movedPrice, pool.totalAssets()), "thin SHORT liquidatable");
        bytes[] memory updateData = _mockPythUpdateData(movedPrice);
        vm.prank(keeper);
        router.executeLiquidation(otherShort, updateData);
        (uint256 thinSize,,,,,,) = engine.positions(otherShort);
        assertEq(thinSize, 0, "liquidated by a keeper while latched");
        assertTrue(engine.degradedMode(), "the latch stays set");
    }

    /// @notice After the latch both accounts of the hedged book close everything and withdraw. With E far above L and
    ///         no owner action, every surface stays shut at +1 h, +24 h and +72 h. The owner's clear, with no
    ///         donation, then reopens each one: the flag is the only remaining cause.
    function test_walkAway_latchOutlivesThePositions() public {
        _scenarioThroughLatch();
        _keeperMark(movedPrice);
        LpNav memory atLatch = _lpNav("at the latch");

        // The SHORT leg closes in full through the Router too; both accounts are flat and withdraw everything.
        _routerClose(hedgeShort, CfdTypes.Side.SHORT, shortSize, movedPrice);
        uint256 outLong = clearinghouse.balanceAssets(hedgeLong);
        uint256 outShort = clearinghouse.balanceAssets(hedgeShort);
        vm.prank(hedgeLong);
        clearinghouse.withdraw(hedgeLong, outLong);
        vm.prank(hedgeShort);
        clearinghouse.withdraw(hedgeShort, outShort);
        (uint256 sizeLong,,,,,,) = engine.positions(hedgeLong);
        (uint256 sizeShort,,,,,,) = engine.positions(hedgeShort);
        assertEq(sizeLong + sizeShort, 0, "both legs closed");
        assertEq(clearinghouse.balanceAssets(hedgeLong) + clearinghouse.balanceAssets(hedgeShort), 0, "both withdrawn");
        emit log_named_uint("round cost, both accounts (deposited - withdrawn)", 2 * LEG_DEPOSIT - outLong - outShort);

        _keeperMark(movedPrice);
        LpNav memory afterUnwind = _lpNav("after the unwind");
        assertGe(afterUnwind.equity, atLatch.equity, "terminal LP equity does not fall on the unwind");
        assertGe(afterUnwind.juniorPerMillionShares, atLatch.juniorPerMillionShares, "Junior share price holds");
        assertGe(afterUnwind.seniorPerMillionShares, atLatch.seniorPerMillionShares, "Senior share price holds");
        emit log_named_uint("E after the unwind", _e());
        emit log_named_uint("L after the unwind", _maxLiability());
        emit log_named_uint("getFreeAssets (not gated by the flag) after the unwind", pool.getFreeAssets());
        assertTrue(engine.degradedMode(), "the latch outlives the positions that set it");

        _warpTo(latchedAt + 1 hours);
        _assertLatchAloneHolds();
        _warpTo(latchedAt + 24 hours);
        _assertLatchAloneHolds();
        uint256 snap = vm.snapshotState();
        _clearAndAssertReopened(); // probe at +24 h, rolled back: the clear alone reopens every surface
        vm.revertToState(snap);
        _warpTo(latchedAt + 72 hours);
        _assertLatchAloneHolds();
        _clearAndAssertReopened();
    }

    /// @notice The owner's exit at the latch: `clearDegradedMode` reverts while E < L, and a recapitalization of
    ///         exactly L - E (one atom less is not enough) lets it succeed. With no Senior loss to restore, the
    ///         donation is ownerless value: deposit activation stays deferred until the owner assigns it to a
    ///         tranche, which returns it to the owner as shares of equal value.
    function test_ownerExits() public {
        _scenarioThroughLatch();
        _keeperMark(movedPrice);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        uint256 deficit = _maxLiability() - _e();
        emit log_named_uint("owner recapitalization that lets clearDegradedMode succeed (L - E)", deficit);
        settlementToken.mint(address(this), deficit);
        settlementToken.approve(address(engine), deficit);
        uint256 snap = vm.snapshotState();
        engine.recapitalizePool(deficit - 1);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        vm.revertToState(snap);

        engine.recapitalizePool(deficit);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "recapitalizing exactly L - E lets the owner clear");
        assertEq(pool.pendingRecapitalizationAssets(), deficit, "the donation is queued as a recapitalization");
        assertEq(
            uint256(engineLens.previewOpenRevertCode(newTrader, CfdTypes.Side.LONG, 10_000e18, 1_000e6, movedPrice)),
            uint256(uint8(CfdEnginePlanTypes.OpenRevertCode.SOLVENCY_EXCEEDED)),
            "opens stay refused until E >= L + B"
        );

        // 11:00: the settlement makes no progress (no free cash for the redemption; entries blocked because the
        // donation, with no Senior loss to restore, is ownerless) until the owner assigns it to a tranche, which
        // mints matching shares to the owner's chosen receiver.
        _warpTo(pool.lpEpochStart(pendingDepositId));
        bytes[] memory updateData = _mockPythUpdateData(movedPrice);
        vm.prank(keeper);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        router.settleLpEpoch(updateData);
        _keeperMark(movedPrice);
        uint256 sharesBefore = juniorVault.balanceOf(address(this));
        pool.assignUnassignedAssets(false, address(this));
        uint256 donationValue = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)) - sharesBefore);
        assertApproxEqAbs(donationValue, deficit, 10, "the donation comes back as Junior shares of equal value");
        emit log_named_uint("value of the Junior shares minted to the owner for the donation", donationValue);
        _keeperSettleLp(movedPrice);
        assertEq(juniorVault.claimableDepositRequest(pendingDepositId, lpPending), PENDING_DEPOSIT, "Junior activates");
    }
}
```

Run from `packages/perps` (the repo compiles with via_ir, so the first run takes a few minutes):

```bash
FOUNDRY_PROFILE=quick forge test --match-path test/poc/DegradedLatchFromHedgedClose.t.sol -vv
```

Output:

```
Ran 4 tests for test/poc/DegradedLatchFromHedgedClose.t.sol:DegradedLatchFromHedgedCloseTest
[PASS] test_control_halfBookDoesNotLatch() (gas: 65539968)
Logs:
  smallest latching fall of the basket, whole basis points: 84

[PASS] test_hedgedCloseLatches_lpNavIntact() (gas: 42838695)
Logs:
  smallest latching fall of the basket, whole basis points: 84
  before the move: terminal LP equity: 2013000898414
  before the move: Junior assets per 1,000,000 shares: 998430316470
  before the move: Senior assets per 1,000,000 shares: 1001579890361
  before the move: pool accountedAssets: 2013000898414
  L - E after the latching close: 121657608
  at the latch: terminal LP equity: 2023542791855
  at the latch: Junior assets per 1,000,000 shares: 1008831136309
  at the latch: Senior assets per 1,000,000 shares: 1001579897983
  at the latch: pool accountedAssets: 2006778342392

[PASS] test_ownerExits() (gas: 38218740)
Logs:
  smallest latching fall of the basket, whole basis points: 84
  owner recapitalization that lets clearDegradedMode succeed (L - E): 121660458
  value of the Junior shares minted to the owner for the donation: 121660457

[PASS] test_walkAway_latchOutlivesThePositions() (gas: 50427532)
Logs:
  smallest latching fall of the basket, whole basis points: 84
  at the latch: terminal LP equity: 2023542789005
  at the latch: Junior assets per 1,000,000 shares: 1008831136002
  at the latch: Senior assets per 1,000,000 shares: 1001579895442
  at the latch: pool accountedAssets: 2006778339542
  round cost, both accounts (deposited - withdrawn): 15551657213
  after the unwind: terminal LP equity: 2026077253708
  after the unwind: Junior assets per 1,000,000 shares: 1011331680818
  after the unwind: Senior assets per 1,000,000 shares: 1001579900524
  after the unwind: pool accountedAssets: 2026153964245
  E after the unwind: 2026153964245
  L after the unwind: 10000000000
  getFreeAssets (not gated by the flag) after the unwind: 2016128964245
  +1 h, no owner action, every surface shut: E - L: 2016153964245
  +24 h, no owner action, every surface shut: E - L: 2016153964245
  +24 h, owner clear (no donation), all reopened: maxJuniorWithdraw: 749226100564
  +72 h, no owner action, every surface shut: E - L: 2016153964245
  +72 h, owner clear (no donation), all reopened: maxJuniorWithdraw: 749226100564

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 90.76ms (247.21ms CPU time)

Ran 1 test suite in 108.30ms (90.76ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

