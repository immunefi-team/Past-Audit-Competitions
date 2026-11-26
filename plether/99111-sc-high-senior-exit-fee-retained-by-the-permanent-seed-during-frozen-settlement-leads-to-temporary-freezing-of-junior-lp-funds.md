# \#99111 [SC-High] Senior exit fee retained by the permanent seed during frozen settlement leads to temporary freezing of Junior LP funds

**Submitted on Nov 11th 2026 at 23:19:04 UTC by @Emmac for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99111
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

HousePool charges a 25 bps fee on Senior redemptions funded while the oracle is frozen and leaves that fee inside Senior
principal for the remaining Senior holders. When the funded redemption is the last non-seed Senior exit, the only Senior
holder left is the permanent seed position, which can never redeem or transfer. HousePool still treats that seed-owned
principal as live Senior exposure. The Junior withdrawal covenant requires Junior to keep a quarter of it, and the Senior
coupon charges Junior 8% a year on it. The last Junior LPs to exit are left with a matured redemption that cannot be
funded, refunded or cancelled, while the coupon keeps moving their principal into the unredeemable seed. At release
parameters, a 30,000,000 USDG Senior exit settled in the weekend window leaves 18,760.36 USDG of Junior funds frozen for the
full year measured by the proof of concept, during which 6,003.31 USDG of those funds is transferred into the seed.

## Vulnerability Details

### Background

- LP epochs settle every hour. While the oracle is frozen (Friday 17:00 to Sunday 17:00 New York time), settlement
  stays open for redemptions and charges a tranche-local exit fee: 25 bps for Senior and 75 bps for Junior. The fee stays
  with the tranche ("Frozen-close spread and frozen LP exit fees stay with LPs", ACCOUNTING_SPEC.md invariant 13).
- Each tranche has a seed position: a permanent minimum share supply funded by the owner with 1 USDG and held by the seed
  receiver. The seed floor can never be redeemed or transferred.
- Junior redemptions are capped by a covenant. Junior must keep enough principal to back active protected Senior exposure
  at the configured Senior share: Junior must hold at least exposure / 4 at the release value `maxSeniorShareBps = 8000`.
- Senior earns an 8% coupon that is paid out of Junior principal on every reconcile.

The defect is that the seed, a bootstrap position that is never meant to carry economic weight, can receive a large
frozen exit fee, and HousePool then applies the covenant and the coupon to that balance exactly as if it belonged to a
Senior LP.

### 1. The frozen exit fee stays in Senior principal

The fee is selected purely from the regime at settlement time and keeps its constructor default in the release
deployment:

```solidity
// packages/perps/src/HousePool.sol:250-257 (constructor)
poolConfig = PoolConfig({
    seniorRateBps: 800,
    markStalenessLimit: 60,
    seniorFrozenLpFeeBps: 25,
    juniorFrozenLpFeeBps: 75,
    maxSeniorExposureAssets: type(uint256).max,
    maxSeniorShareBps: 10_000
});

// packages/perps/src/HousePool.sol:1147-1155
function _settlementFeeBps(bool isSenior, bool oracleFrozen) internal view returns (uint256) {
    if (!oracleFrozen) {
        return 0;
    }
    return isSenior ? poolConfig.seniorFrozenLpFeeBps : poolConfig.juniorFrozenLpFeeBps;
}
```

The release wrapper `ArbitrumSepoliaReleaseHousePool` (`script/DeployPerpsArbitrumSepolia.s.sol:68-83`) overrides only
`maxSeniorExposureAssets` (40,000,000 USDG) and `maxSeniorShareBps` (8000). The live v2.0.0 HousePool on Arbitrum Sepolia
(`0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e`) returns `seniorFrozenLpFeeBps() = 25`, `maxSeniorShareBps() = 8000` and
`seniorRateBps() = 800`.

The Senior phase funds redemptions net of that fee and removes only the net amount from Senior principal and from
accounted assets:

```solidity
// packages/perps/src/HousePool.sol:853-861 (_settleLpEpoch)
seniorPhase = _fundRedemptionPhase(
    seniorVault, result.cutoffEpoch, seniorPhase, _settlementFeeBps(true, statusSnapshot.oracleFrozen)
);
if (seniorPhase.fundedAssets > 0) {
    HousePoolWaterfallAccountingLib.WaterfallState memory nextState =
        HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets);
    _setWaterfallState(nextState);
    accountedAssets -= seniorPhase.fundedAssets;
```

```solidity
// packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:119-131
function scaleSeniorOnWithdraw(WaterfallState memory state, uint256 fundedAssets)
    internal pure returns (WaterfallState memory nextState)
{
    nextState = state;
    if (fundedAssets == 0) {
        return nextState;
    }
    nextState.seniorHighWaterMark -= Math.mulDiv(
        state.seniorHighWaterMark, fundedAssets, state.seniorPrincipal, Math.Rounding.Floor
    );
    nextState.seniorPrincipal -= fundedAssets;
}
```

The withheld 25 bps therefore remains in `seniorPrincipal` and in the Senior high-water mark, owned pro rata by
whichever Senior shares are still outstanding after the burn.

### 2. After the last non-seed exit, the fee belongs to a position that can never leave

When the settlement burns the last non-seed Senior shares, the only outstanding Senior shares are the seed floor. The
seed is created once and its floor can neither be redeemed nor transferred:

```solidity
// packages/perps/src/HousePool.sol:706 and 730-731
/// @notice Seeds a tranche with a permanent minimum share supply backed by real settlement token.
if (toSenior ? seniorSeedInitialized : juniorSeedInitialized) {
    revert HousePool__SeedAlreadyInitialized();
}

// packages/perps/src/TrancheVault.sol:1630-1637
function _unlockedOwnerShares(address owner) internal view returns (uint256 ownerShares) {
    ownerShares = balanceOf(owner);
    if (owner == seedReceiver && seedReceiver != address(0)) {
        ownerShares = ownerShares > seedShareFloor ? ownerShares - seedShareFloor : 0;
    }
}

// packages/perps/src/TrancheVault.sol:322-324 (_update)
if (from == seedReceiver && from != address(0) && balanceOf(from) - amount < seedShareFloor) {
    revert TrancheVault__SeedFloorBreached();
}
```

No function in HousePool or TrancheVault lowers `seedShareFloor`, re-assigns the seed position or sweeps Senior
principal. The retained fee becomes permanently unwithdrawable Senior principal.

### 3. Seed-owned principal still drives the Junior withdrawal covenant

The Junior redemption budget is computed from the full Senior principal and high-water mark, with no distinction between
LP-owned and seed-owned principal:

```solidity
// packages/perps/src/HousePool.sol:1205-1217
function _juniorSettlementCapacity(
    uint256 freeAssets,
    HousePoolWaterfallAccountingLib.WaterfallState memory state
) internal view returns (uint256) {
    uint256 ratioCap = HousePoolSeniorCapacityLib.juniorWithdrawalRatioCap(
        state.seniorPrincipal,
        state.seniorHighWaterMark,
        state.juniorPrincipal,
        reservedSeniorDepositAssets,
        poolConfig.maxSeniorShareBps
    );
    return _min(freeAssets, _min(state.juniorPrincipal, ratioCap));
}

// packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol:64-90
uint256 exposure = protectedExposure(seniorPrincipal, seniorHighWaterMark);
...
uint256 requiredJunior =
    _saturatingMulDiv(exposure, BPS - maxSeniorShareBps, maxSeniorShareBps, Math.Rounding.Ceil);
return juniorPrincipal > requiredJunior ? juniorPrincipal - requiredJunior : 0;
```

Junior must therefore keep a quarter of the seed-owned principal at all times. In the proof of concept the seed owns
75,007.29 USDG after the Senior exit and 75,041.44 USDG once the coupon has accrued to the time the Junior exit is
settled, so 18,760.36 USDG of Junior principal cannot be redeemed.

### 4. Seed-owned principal still charges the Senior coupon to Junior

```solidity
// packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:57-60 (planReconcile)
if (couponElapsed > 0 && state.juniorSupply > 0) {
    (plan.state.waterfall,) =
        HousePoolWaterfallAccountingLib.paySeniorCoupon(plan.state.waterfall, seniorRateBps, couponElapsed);
}
```

Every reconcile moves 8% a year of the seed-owned principal out of Junior principal and into the seed. Because the seed's
principal grows, the covenant floor grows with it.

### 5. The Junior redemption is stranded with no exit path

A Junior LP's matured full redemption is funded only down to the covenant floor. The unfunded remainder of the head epoch
is refunded only when its shares are worthless, and otherwise the funding loop stops on a zero budget:

```solidity
// packages/perps/src/HousePool.sol:956-990 (_fundRedemptionPhase)
uint256 fullHeadAssets = REDEMPTION_MATH_SIDECAR.netAssetsForShares(
    remainingShares, phase.pricingPrincipal, phase.pricingSupply, 1, 1000, feeBps
);
if (fullHeadAssets == 0) {
    ... vault.refundRedeemEpochRemainder(epochId, remainingShares) ...
}
if (phase.fundedAssets >= phase.budget) {
    break;
}
```

The matured, partially funded request cannot be cancelled, and every later settlement reverts because no epoch can make
progress:

```solidity
// packages/perps/src/TrancheVault.sol:894-896 (cancelRedeemRequest)
if (currentLpEpoch() >= requestId || epoch.fundedShares != 0 || epoch.fundedAssets != 0 || epoch.refundEnabled)
{
    revert TrancheVault__RedeemCancellationUnavailable();
}

// packages/perps/src/HousePool.sol:1183-1185 (_emitLpEpochSettled)
if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
    revert HousePool__NoLpEpochProgress();
}
```

The Junior LP's remaining shares stay escrowed in the vault. They are not funded, not refundable, not cancellable and not
transferable, and the coupon keeps reducing the principal behind them.

### 6. This is not the intended role of the seed

- `initializeSeedPosition` defines the seed as "a permanent minimum share supply backed by real settlement token"
  (`HousePool.sol:706`), funded with 1 USDG per tranche at deployment (`DEPLOYMENT.md`, v2.0.0 release manifest).
- The documentation routes only rounding dust to "the permanent seed account" (`SECURITY.md:918`, `SECURITY.md:1416`).
- With 1 USDG seeds the two seeds offset each other: the Senior seed requires 0.25 USDG of Junior backing, which the Junior
  seed alone provides, so non-seed Junior LPs can always exit in full. The control run in the proof of concept shows exactly
  this: the same Senior exit settled in the live market leaves 1.02 USDG in the Senior seed and the Junior LP withdraws
  everything.
- The covenant exists to keep Junior capital behind Senior LPs. `SECURITY.md:1209-1213` lists how a covenant restriction
  is cured: "funded Senior redemptions, Junior deposits, or subsequent revenue". Seed-owned exposure can never be redeemed,
  so the first cure does not exist for it, and no Senior LP remains to be protected.
- The frozen exit fee compensates remaining LPs for stale-window exits. Here no LP remains, and the fee is locked in a
  position that no LP owns.

### 7. Reachability

LP settlement is permissionless whenever the oracle is frozen. Caller restrictions apply only to live settlement with open
positions:

```solidity
// packages/perps/src/HousePool.sol:809-819 (settleLpEpoch)
if (accountingSnapshot.hasOpenPositions && !statusSnapshot.oracleFrozen) {
    if (
        msg.sender != ENGINE.orderRouter() || ENGINE.lastMarkPrice() != expectedMarkPrice
            || statusSnapshot.lastMarkTime != expectedPublishTime
    ) {
        revert HousePool__Unauthorized();
    }
    ...
}
return _settleLpEpoch(accountingSnapshot, statusSnapshot, cutoffEpoch);
```

A request matures at the next hourly boundary (or the one after, inside the five-minute cutoff). Any final Senior exit
requested from the hour before the Friday close until the hour before the Sunday reopen matures into a frozen epoch, a
window of 48 of the 168 hours in a week. The project's LP epoch keeper (`script/LpEpochKeeper.s.sol`) is built to settle
matured LP work during frozen windows through the cached-mark route, and any other address can do the same. No special
role, capital or market move is required; the book can be flat.

### Recommended mitigation

Exclude the seed floor's share of Senior principal from covenant exposure and from coupon accrual, or route a frozen exit
fee that would otherwise be retained only by the seed floor to Junior principal or to `unassignedAssets`. Waiving the
frozen exit fee on a Senior redemption that leaves only the seed outstanding also removes the defect.

## Impact Details

Selected impact: **High. Temporary freezing of funds for at least 24 hours.**

The proof of concept runs at release parameters on an unmodified checkout. Each scenario is compared with a control in
which the identical Senior exit is settled one epoch earlier in the live market.

| Scenario | Seed-owned Senior principal after the exit | Junior funds frozen after a full exit request | Frozen at 1h, 24h, 72h, 30d and 365d | Junior principal after 365 days |
| --- | --- | --- | --- | --- |
| 30,000,000 Senior / 10,000,000 Junior, request on Friday afternoon | 75,007.29 USDG | 18,760.36 USDG | Yes | 12,757.04 USDG |
| 30,000,000 / 10,000,000, request inside the frozen window | 75,008.66 USDG | 18,760.36 USDG | Yes | 12,757.04 USDG |
| 30,000,000 / 10,000,000, live control | 1.02 USDG | 0 (full exit) | Not applicable | Not applicable |
| 1,000,000 / 1,000,000, request on Friday afternoon | 2,501.23 USDG | 625.59 USDG | Yes | 425.40 USDG |

- **Freeze of Junior LP funds.** The Junior LP's 19,164,043,585,461 escrowed shares (18,760.36 USDG of principal) remain
  unfunded, unrefundable and uncancellable at every check through 365 days, and every settlement attempt reverts with
  `HousePool__NoLpEpochProgress`. The freeze is lifted only by a timelocked governance increase of `maxSeniorShareBps`,
  which takes at least 72 hours and cannot reach 10,000 after activation, so the covenant floor never reaches zero, or by
  new Junior capital or revenue, which moves the frozen amount onto whichever Junior LP exits last. A timelocked change of
  the coupon rate stops the drain but does not release the frozen funds.
- **Permanent loss of value into the seed.** The stranded fee and every later coupon payment on it accrue to a position
  that nobody can withdraw, including the owner. During the year measured, 6,003.31 USDG of the frozen Junior principal was
  transferred into the seed, whose Senior principal grew from 75,007.29 to 81,044.75 USDG.
- **Scale.** The frozen amount is 25% of the stranded fee, which is 25 bps of the final non-seed Senior exit. At the
  40,000,000 USDG release Senior cap, a final weekend exit strands 100,000 USDG, freezes 25,000 USDG of Junior funds and
  drains 8,000 USDG of Junior principal into the seed in the first year. For the Senior book currently live on Arbitrum
  Sepolia (11,112,650 USDG), the same exit strands 27,781 USDG and freezes 6,945 USDG.
- **Recurrence.** Every future Senior cycle that ends with a final exit settled in a frozen window adds another 25 bps of
  that exit to the seed, so the frozen Junior floor and the coupon drain accumulate over the life of the pool.

## References

- HousePool frozen fee defaults: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L250-L257
- HousePool Senior phase funding: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L853-L861
- HousePool `_settlementFeeBps`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1147-L1155
- HousePool seed initialization: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L706-L731
- HousePool permissionless frozen settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L809-L819
- HousePool Junior budget and covenant: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L881 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1205-L1217
- HousePool redemption phase: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L956-L990
- HousePool no-progress revert: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1183-L1185
- TrancheVault seed floor: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1630-L1637 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L322-L324
- TrancheVault cancellation guard: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L894-L896
- Junior covenant: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol#L64-L90
- Senior withdrawal scaling: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L119-L131
- Senior coupon in reconcile: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L57-L60
- Frozen LP exit fees stay with LPs: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1296
- Covenant as an LP-liquidity restriction and its cures: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1205-L1213
- Seed dust routing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L918 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1416
- Release HousePool wrapper: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/script/DeployPerpsArbitrumSepolia.s.sol#L68-L83
- LP epoch keeper: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/script/LpEpochKeeper.s.sol
- Live v2.0.0 HousePool on Arbitrum Sepolia: https://sepolia.arbiscan.io/address/0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e


## Proof of Concept


## Proof of Concept

Place the file below at `packages/perps/test/perps/poc/SeniorSeedFrozenFeeStrandPoC.t.sol` on commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with no other change, and run from the repository root:

```sh
forge test --root packages/perps --match-path test/perps/poc/SeniorSeedFrozenFeeStrandPoC.t.sol -vv
```

The test extends the repository's `BasePerpTest` fixture and mocks only Pyth and USDG. Setup installs the release Senior
caps (40,000,000 USDG and 8000 bps) and the release router confidence policy through the owner timelock, which is the
state the release HousePool wrapper creates at construction, and `test_ReleaseParameters` asserts the release values.
Every LP action and every settlement after setup is performed by a plain address through a public entrypoint. No trading
is involved; the book is flat throughout.

Each test runs three scenarios from one snapshot:

- A: the last Senior LP requests a full exit on Friday afternoon and an independent keeper settles it inside the frozen
  window.
- B: the Senior LP requests and is settled inside the frozen window, accepting the documented fee.
- C (control): the identical exit is settled one epoch earlier in the live market.

After the reopen, the Junior LP requests a full exit. The test then asserts at 1 hour, 24 hours, 72 hours, 30 days and
365 days that settlement reverts with `HousePool__NoLpEpochProgress` and that the remaining shares are still pending, not
claimable, not refundable and not cancellable. It also asserts that the seed floor can neither request redemption nor
transfer.

### Test source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

/// @title PoC: a frozen-window Senior exit fee strands in the permanent Senior seed and freezes Junior LP funds
/// @notice When the last non-seed Senior LPs are funded in an oracle-frozen LP settlement, the 25 bps frozen exit fee
///         stays in Senior principal, whose only remaining owner is the permanent Senior seed floor (cannot redeem,
///         cannot transfer, set once). That seed-owned principal still counts as Senior exposure in the Junior
///         withdrawal covenant (Junior must keep >= exposure / 4 at maxSeniorShareBps 8000) and keeps earning the 8%
///         Senior coupon out of Junior principal. A Junior LP's matured full redemption is funded only down to that
///         floor; the remainder stays escrowed with no cancel, refund or further funding path, while the coupon moves it
///         into the unwithdrawable seed.
///
///         Release configuration: DEPLOYMENT.md risk parameters, 1 USDG Senior/Junior seeds, Senior caps 40M / 8000 bps
///         (installed through the owner timelock in setUp, as the release HousePool wrapper does at construction),
///         frozen LP fees 25 / 75 bps (HousePool constructor defaults, not overridden by the release wrapper).
///         Only Pyth and USDG are mocked. LP actors and the settling keeper are plain addresses using public entrypoints.
///         No trading is needed: the book is flat throughout.
/// Run: forge test --root packages/perps --match-path test/perps/poc/SeniorSeedFrozenFeeStrandPoC.t.sol -vv
contract SeniorSeedFrozenFeeStrandPoC is BasePerpTest {

    address internal constant KEEPER = address(0x4EE9); // independent keeper: mark refreshes and LP settlements
    address internal constant JUNIOR_LP = address(0x71C7);
    address internal constant SENIOR_LP = address(0xB0B0); // the last non-seed Senior LP
    bytes4 internal constant NO_PROGRESS = 0x86cca6b8; // HousePool__NoLpEpochProgress()
    uint256 internal constant FRI = 1_717_718_400; // Friday 2024-06-07 00:00 UTC (DST: FX close 21:00 UTC)
    uint256 internal constant PX = 1e8;

    enum Mode {
        LiveControl, // Senior exit settled one epoch before the Friday close: no fee
        FridayRequestFrozenSettle, // request Friday afternoon, settled by a keeper inside the frozen window
        FrozenRequestFrozenSettle // request and settlement both inside the frozen window (fee knowingly accepted)
    }

    struct Result {
        uint256 seniorPaid;
        uint256 seedSeniorPrincipal;
        uint256 juniorPaid;
        uint256 juniorSharesStuck;
        uint256 juniorPrincipalLocked;
        uint256 juniorPrincipalAfterYear;
        uint256 seedSeniorPrincipalAfterYear;
    }

    // ------------------------------------------------------------------ release configuration

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _installReleaseConfig() internal {
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(rc);
        _setSeniorCapacity(40_000_000e6, 8000);
    }

    function test_ReleaseParameters() public {
        _installReleaseConfig();
        assertEq(pool.seniorFrozenLpFeeBps(), 25, "senior frozen LP fee");
        assertEq(pool.juniorFrozenLpFeeBps(), 75, "junior frozen LP fee");
        assertEq(pool.maxSeniorShareBps(), 8000, "maxSeniorShareBps");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "maxSeniorExposureAssets");
        assertEq(pool.seniorRateBps(), 800, "seniorRateBps");
        (uint256 vpiFactor, uint256 maxSkew, uint256 maint, uint256 init, uint256 fad, uint256 carry,,,,) =
            engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "vpiFactor");
        assertEq(maxSkew, 0.4e18, "maxSkewRatio");
        assertEq(maint, 10, "maintMarginBps");
        assertEq(init, 20, "initMarginBps");
        assertEq(fad, 300, "fadMarginBps");
        assertEq(carry, 500, "baseCarryBps");
    }

    // ------------------------------------------------------------------ helpers

    function _step(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _refreshAt(
        uint256 t
    ) internal {
        _step(t);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), SafeCast.toInt64(int256(PX)), 1e4, int32(-8), t, t - 1);
        bytes[] memory d = new bytes[](1);
        d[0] = abi.encode(PX);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function _firstTimeWithFrozen(
        uint256 from,
        bool frozen
    ) internal returns (uint256 t) {
        uint256 start = vm.getBlockTimestamp();
        for (t = from; t < from + 3 hours; ++t) {
            vm.warp(t);
            if (engine.isOracleFrozen() == frozen) {
                break;
            }
        }
        vm.warp(start);
    }

    function _settle() internal returns (bool ok, bytes memory err) {
        vm.prank(KEEPER);
        try pool.settleLpEpoch(0, 0) {
            ok = true;
        } catch (bytes memory e) {
            err = e;
        }
    }

    function _publicDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 id = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _refreshAt(vault.depositEpochStart(id) + 60);
        (bool ok,) = _settle();
        require(ok, "deposit settlement");
        vm.prank(lp);
        vault.claimDepositShares(id);
    }

    function _seniorExit(
        Mode mode,
        uint256 closeT
    ) internal returns (uint256 paid) {
        uint256 requestAt = mode == Mode.LiveControl
            ? closeT - 2 hours + 60
            : (mode == Mode.FridayRequestFrozenSettle ? closeT - 59 minutes : closeT + 1 hours);
        _step(requestAt);
        uint256 shares = seniorVault.maxRequestRedeem(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 id = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
        _step(seniorVault.depositEpochStart(id) + 10 minutes);
        assertEq(engine.isOracleFrozen(), mode != Mode.LiveControl, "settlement regime");
        (bool ok,) = _settle();
        assertTrue(ok, "senior exit settled by an independent keeper");
        uint256 claimable = seniorVault.claimableRedeemRequest(id, SENIOR_LP);
        vm.prank(SENIOR_LP);
        paid = seniorVault.claimRedeem(id, claimable, SENIOR_LP, SENIOR_LP);
        assertEq(seniorVault.totalSupply(), seniorVault.seedShareFloor(), "only the senior seed floor remains");
    }

    function _assertStillFrozen(
        uint256 id,
        uint256 stuck,
        string memory label
    ) internal {
        (bool ok, bytes memory err) = _settle();
        assertFalse(ok, string.concat(label, ": settlement makes no progress"));
        assertEq(bytes4(err), NO_PROGRESS, string.concat(label, ": NoLpEpochProgress"));
        assertEq(juniorVault.pendingRedeemRequest(id, JUNIOR_LP), stuck, string.concat(label, ": shares still stuck"));
        assertEq(juniorVault.claimableRedeemRequest(id, JUNIOR_LP), 0, string.concat(label, ": nothing claimable"));
        assertEq(juniorVault.refundableRedeemRequest(id, JUNIOR_LP), 0, string.concat(label, ": nothing refundable"));
        vm.prank(JUNIOR_LP);
        try juniorVault.cancelRedeemRequest(id, JUNIOR_LP) {
            revert(string.concat(label, ": matured junior redemption was cancelled"));
        } catch {}
    }

    function _run(
        Mode mode,
        uint256 juniorSize,
        uint256 seniorSize
    ) internal returns (Result memory r) {
        _installReleaseConfig();
        uint256 closeT = _firstTimeWithFrozen(FRI + 20 hours, true);
        uint256 openT = _firstTimeWithFrozen(FRI + 2 days + 20 hours, false);
        _refreshAt(FRI + 10 hours);
        _publicDeposit(juniorVault, JUNIOR_LP, juniorSize);
        _publicDeposit(seniorVault, SENIOR_LP, seniorSize);

        r.seniorPaid = _seniorExit(mode, closeT);
        r.seedSeniorPrincipal = pool.seniorPrincipal();

        // After the reopen, the Junior LP requests a full exit; the keeper settles it.
        _step(openT + 1 hours + 60);
        uint256 shares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 id = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
        _step(juniorVault.depositEpochStart(id) + 60);
        (bool ok,) = _settle();
        assertTrue(ok, "junior settlement");
        uint256 claimable = juniorVault.claimableRedeemRequest(id, JUNIOR_LP);
        if (claimable != 0) {
            vm.prank(JUNIOR_LP);
            r.juniorPaid = juniorVault.claimRedeem(id, claimable, JUNIOR_LP, JUNIOR_LP);
        }
        r.juniorSharesStuck = juniorVault.pendingRedeemRequest(id, JUNIOR_LP);
        r.juniorPrincipalLocked = pool.juniorPrincipal();
        if (mode == Mode.LiveControl) {
            assertEq(r.juniorSharesStuck, 0, "control: junior LP exits fully");
            return r;
        }

        uint256 t0 = vm.getBlockTimestamp();
        _step(t0 + 1 hours);
        _assertStillFrozen(id, r.juniorSharesStuck, "1h");
        _step(t0 + 24 hours + 60);
        _assertStillFrozen(id, r.juniorSharesStuck, "24h");
        _step(t0 + 72 hours + 60);
        _assertStillFrozen(id, r.juniorSharesStuck, "72h");
        _step(t0 + 30 days);
        _assertStillFrozen(id, r.juniorSharesStuck, "30d");
        _step(t0 + 365 days);
        _assertStillFrozen(id, r.juniorSharesStuck, "365d");
        (r.seedSeniorPrincipalAfterYear, r.juniorPrincipalAfterYear,,) = pool.getPendingTrancheState();

        // The seed floor that owns the stranded principal can neither request redemption nor transfer.
        address seed = seniorVault.seedReceiver();
        assertEq(seniorVault.maxRequestRedeem(seed), 0, "seed floor cannot request redemption");
        vm.prank(seed);
        try seniorVault.transfer(address(0xBEEF), 1) {
            revert("seed floor transferred");
        } catch {}
    }

    function _report(
        string memory tag,
        Result memory r
    ) internal pure {
        console2.log(tag);
        console2.log("  senior LP paid / seed-owned senior principal", r.seniorPaid, r.seedSeniorPrincipal);
        console2.log("  junior LP paid / junior shares stuck in escrow", r.juniorPaid, r.juniorSharesStuck);
        console2.log("  junior principal frozen / after 365 days", r.juniorPrincipalLocked, r.juniorPrincipalAfterYear);
        console2.log("  seed-owned senior principal after 365 days", r.seedSeniorPrincipalAfterYear);
    }

    function _compare(
        uint256 juniorSize,
        uint256 seniorSize
    ) internal {
        uint256 snap = vm.snapshotState();
        Result memory a = _run(Mode.FridayRequestFrozenSettle, juniorSize, seniorSize);
        vm.revertToState(snap);
        Result memory b = _run(Mode.FrozenRequestFrozenSettle, juniorSize, seniorSize);
        vm.revertToState(snap);
        Result memory c = _run(Mode.LiveControl, juniorSize, seniorSize);
        _report("A: request Friday afternoon, keeper settles inside the frozen window", a);
        _report("B: request and settlement inside the frozen window", b);
        _report("C: live control, identical exit one epoch earlier", c);

        uint256 expectedFee = seniorSize * 25 / 10_000;
        Result[2] memory frozenRuns = [a, b];
        for (uint256 i; i < 2; ++i) {
            Result memory x = frozenRuns[i];
            assertApproxEqRel(x.seedSeniorPrincipal, expectedFee, 0.01e18, "25 bps fee stranded with the seed");
            assertGt(x.juniorSharesStuck, 0, "junior LP cannot fully exit");
            assertGe(x.juniorPrincipalLocked * 4 + 4, x.seedSeniorPrincipal, "junior held at exposure / 4");
            assertLt(x.juniorPrincipalAfterYear, x.juniorPrincipalLocked, "coupon drains frozen junior into the seed");
            assertGt(x.seedSeniorPrincipalAfterYear, x.seedSeniorPrincipal, "seed principal grows from junior coupon");
        }
        assertLt(c.seedSeniorPrincipal, 2e6, "control: seed keeps about 1 USDG");
    }

    /// @notice 1M Senior / 1M Junior.
    function test_FrozenSeniorExitFreezesJunior_1M() public {
        _compare(1_000_000e6, 1_000_000e6);
    }

    /// @notice Release scale: 30M Senior (within the 40M / 80% caps) against 10M Junior.
    function test_FrozenSeniorExitFreezesJunior_ReleaseScale() public {
        _compare(10_000_000e6, 30_000_000e6);
    }

}
```

### Output on an unmodified checkout of the commit

```text
Ran 3 tests for test/perps/poc/SeniorSeedFrozenFeeStrandPoC.t.sol:SeniorSeedFrozenFeeStrandPoC
[PASS] test_FrozenSeniorExitFreezesJunior_1M() (gas: 123224306)
Logs:
  A: request Friday afternoon, keeper settles inside the frozen window
    senior LP paid / seed-owned senior principal 997583352738 2501229867
    junior LP paid / junior shares stuck in escrow 999290686569 637978685270
    junior principal frozen / after 365 days 625592139 425402655
    seed-owned senior principal after 365 days 2702558038
  B: request and settlement inside the frozen window
    senior LP paid / seed-owned senior principal 997601571916 2501275547
    junior LP paid / junior shares stuck in escrow 999272467371 637990332219
    junior principal frozen / after 365 days 625592143 425402658
    seed-owned senior principal after 365 days 2702558055
  C: live control, identical exit one epoch earlier
    senior LP paid / seed-owned senior principal 1000074429223 1020952
    junior LP paid / junior shares stuck in escrow 999925570290 0
    junior principal frozen / after 365 days 979061 0
    seed-owned senior principal after 365 days 0

[PASS] test_FrozenSeniorExitFreezesJunior_ReleaseScale() (gas: 123224149)
Logs:
  A: request Friday afternoon, keeper settles inside the frozen window
    senior LP paid / seed-owned senior principal 29927500582190 75007288086
    junior LP paid / junior shares stuck in escrow 9978699623761 19164043585461
    junior principal frozen / after 365 days 18760358810 12757043991
    seed-owned senior principal after 365 days 81044750058
  B: request and settlement inside the frozen window
    senior LP paid / seed-owned senior principal 29928047157533 75008657967
    junior LP paid / junior shares stuck in escrow 9978153047813 19165091547457
    junior principal frozen / after 365 days 18760358931 12757044074
    seed-owned senior principal after 365 days 81044750580
  C: live control, identical exit one epoch earlier
    senior LP paid / seed-owned senior principal 30002232876712 1020952
    junior LP paid / junior shares stuck in escrow 9997767122947 0
    junior principal frozen / after 365 days 978915 0
    seed-owned senior principal after 365 days 0

[PASS] test_ReleaseParameters() (gas: 563536)
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 169.92ms (329.58ms CPU time)

Ran 1 test suite in 170.53ms (169.92ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

