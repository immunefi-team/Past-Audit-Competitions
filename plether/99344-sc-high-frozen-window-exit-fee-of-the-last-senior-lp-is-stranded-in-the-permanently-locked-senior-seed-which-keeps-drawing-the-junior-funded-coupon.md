# \#99344 [SC-High] Frozen-window exit fee of the last senior LP is stranded in the permanently locked senior seed, which keeps drawing the junior-funded coupon

**Submitted on Nov 13th 2026 at 09:41:44 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99344
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**

## Description

## Brief/Intro

During `oracleFrozen`, `HousePool._settleLpEpoch` pays senior redemptions net of a 25 bps exit fee and debits senior principal by the net amount only, so the fee stays with the remaining senior shares. When the last real senior LP exits, the only shares left are the seed floor, which can never be redeemed, transferred or burned. The fee is then permanently frozen, junior LPs keep paying the 8% senior coupon on it, and a quarter of it pins junior capital through the covenant.

## Vulnerability Details

During `oracleFrozen`, `HousePool._settleLpEpoch` pays senior redemptions net of a 25 bps exit fee and debits senior principal by the net amount only, so the fee stays with the remaining senior shares. When the last real senior LP exits, the only shares left are the seed floor, which can never be redeemed, transferred or burned. The fee is then permanently frozen, junior LPs keep paying the 8% senior coupon on it, and a quarter of it pins junior capital through the covenant.

## Details and Code Snippet

1. The retained fee stays in senior principal

`packages/perps/src/HousePool.sol:845-863`, in `_settleLpEpoch`:

```solidity
RedemptionPhase memory seniorPhase = RedemptionPhase({
    pricingPrincipal: seniorPrincipal,
    pricingSupply: _seniorShareSupply(),
    budget: _min(freeAssets, seniorPrincipal),
    fundedShares: 0,
    fundedAssets: 0,
    processedEpochs: 0,
    backlog: false
});
seniorPhase = _fundRedemptionPhase(
    seniorVault, result.cutoffEpoch, seniorPhase, _settlementFeeBps(true, statusSnapshot.oracleFrozen)
);
if (seniorPhase.fundedAssets > 0) {
    HousePoolWaterfallAccountingLib.WaterfallState memory nextState =
        HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets);
    _setWaterfallState(nextState);
    accountedAssets -= seniorPhase.fundedAssets;
    freeAssets -= seniorPhase.fundedAssets;
}
```

`_settlementFeeBps` (`:1147-1155`) returns `seniorFrozenLpFeeBps` whenever `oracleFrozen` is true. Its constructor default is 25 bps (`:253`), and the release wrapper `ArbitrumSepoliaReleaseHousePool` (`script/DeployPerpsArbitrumSepolia.s.sol:71-83`) overrides only the two senior caps. The funded amount is the net payout from `HousePoolRedemptionMathLib.netAssetsForShares` (`HousePoolRedemptionMathLib.sol:45-47`):

```solidity
uint256 grossAssets =
    Math.mulDiv(shares, principal + virtualAssets, supply + virtualShares, Math.Rounding.Floor);
return Math.mulDiv(grossAssets, BPS - feeBps, BPS, Math.Rounding.Floor);
```

The burned shares were worth `grossAssets`. `scaleSeniorOnWithdraw` (`HousePoolWaterfallAccountingLib.sol:119`, documented as "Retained fees stay protected") removes only the net amount from `seniorPrincipal` and scales the high-water mark by the same fraction. The difference stays in senior principal, owned pro rata by the senior shares still outstanding. `ACCOUNTING_SPEC.md:578` states the intent: "the fee remains inside the same tranche and therefore benefits incumbent LPs of that tranche only."

2. After the last senior exit, the only owner is the seed floor

`HousePool.initializeSeedPosition` (`HousePool.sol:714-762`) mints the seed shares and locks all of them at `:760`:

```solidity
ITrancheVaultBootstrap(targetVault).configureSeedPosition(receiver, shares);
```

The release seed is `1e6`, one USDG per tranche (`packages/perps/DEPLOYMENT.md:156`). In the vault, `seedShareFloor` (`TrancheVault.sol:139`) has one writer, `configureSeedPosition` (`:1557-1572`). It only accepts a floor at or above the current one, and the pool reaches it only through `initializeSeedPosition`, which reverts `HousePool__SeedAlreadyInitialized` on a second call for the same tranche. The floor is enforced on every transfer and burn (`:322-324`):

```solidity
if (from == seedReceiver && from != address(0) && balanceOf(from) - amount < seedShareFloor) {
    revert TrancheVault__SeedFloorBreached();
}
```

`maxRequestRedeem` (`:1328-1335`) returns `_unlockedOwnerShares` (`:1630-1637`), which is zero while the seed receiver holds only floor shares:

```solidity
ownerShares = balanceOf(owner);
if (owner == seedReceiver && seedReceiver != address(0)) {
    ownerShares = ownerShares > seedShareFloor ? ownerShares - seedShareFloor : 0;
}
```

So once a frozen-window redemption takes the last non-seed senior shares, every remaining senior share sits under the floor. The retained fee is now owned by shares that can never be redeemed, transferred or burned. No owner or pool function moves value out of them. `sweepExcess` acts only on raw balance above `accountedAssets` and `assignUnassignedAssets` only on `unassignedAssets`, and the retained fee is neither. New senior deposits enter at the post-exit share price, so they do not release it either.

3. The coupon keeps moving junior principal into it

`HousePoolReconcilePlanLib.planReconcile` (`HousePoolReconcilePlanLib.sol:57-60`):

```solidity
if (couponElapsed > 0 && state.juniorSupply > 0) {
    (plan.state.waterfall,) =
        HousePoolWaterfallAccountingLib.paySeniorCoupon(plan.state.waterfall, seniorRateBps, couponElapsed);
}
```

The guard requires junior owners but has no counterpart for senior owners other than the seed. `paySeniorCoupon` (`HousePoolWaterfallAccountingLib.sol:65`) takes `seniorPrincipal * seniorRateBps * elapsed / (10_000 * 365 days)` out of junior principal. It capitalizes that amount into senior principal and the high-water mark. At the release rate of 800 bps (`HousePool.sol:251`), the stranded balance grows by 8% a year. Junior LPs fund all of it, and all of it accrues to the locked seed. Each later reconcile charges the coupon on the coupons already capitalized.

4. Junior capital is pinned against it

`_juniorSettlementCapacity` (`HousePool.sol:1205-1217`) caps junior redemption funding with `HousePoolSeniorCapacityLib.juniorWithdrawalRatioCap` (`HousePoolSeniorCapacityLib.sol:64-90`):

```solidity
uint256 requiredJunior =
    _saturatingMulDiv(exposure, BPS - maxSeniorShareBps, maxSeniorShareBps, Math.Rounding.Ceil);
return juniorPrincipal > requiredJunior ? juniorPrincipal - requiredJunior : 0;
```

`exposure` is the larger of senior principal and its high-water mark, so it includes the stranded balance. At the release `maxSeniorShareBps` of 8,000, junior LPs must keep a quarter of that balance in the pool. Because the seed balance can never leave, that slice of junior capital is pinned permanently.

5. When the last exit pays the fee

The fee is set when an epoch is funded, not when it is requested. `ACCOUNTING_SPEC.md:570` says "a redemption request escrows shares without fixing its exchange rate or fee". `_requestEpochWindow` (`TrancheVault.sol:545-556`) assigns a request to the next hourly epoch, or the one after it once the request is within `LP_REQUEST_CUTOFF_DURATION` (5 minutes, `:39`) of the boundary. The Friday close is 17:00 New York time, which falls on an hour boundary. Any redemption requested from 15:55 New York time on a Friday therefore targets an epoch that starts at or after the close, and can only be funded in the frozen window. A request that matured earlier but was not settled before the close is in the same position, because frozen-window settlement is permissionless. The scan report finding "Permissionless frozen-state settlement can apply an unsnapshotted redemption fee" concerns who pays the fee and when. The issue here is who receives it.


## Impact Details
There is no exploit and no attacker profit. The exiting LP pays the configured 25 bps as documented, and token custody stays consistent. The value lies in where the fee ends up and what follows:

- The retained fee is frozen permanently. It equals 0.25% of the senior value funded in the final frozen-window exit. More generally, the seed receives its pro-rata share of every retained fee, and that share approaches all of it as the last real holders leave. In the test below, a 1,000,000 USDG exit leaves 2,500.87 USDG under the seed floor. A full final exit at the 40,000,000 USDG release senior cap would leave 100,000 USDG.
- Junior LPs pay the 8% coupon on that balance for as long as junior shares exist. In the test, the first year costs 200.15 USDG, which is capitalized into the locked balance, so each later year costs more. At the 40,000,000 USDG cap, the first year would cost 8,000 USDG.
- A quarter of the stranded senior exposure is pinned in junior principal through the covenant. In the test, that is 675.50 USDG after one year.
- The spec describes the fee as compensation for incumbent LPs, and monitoring would reasonably treat it that way. In this case it compensates no withdrawable holder, and nothing in the code or the documentation records that.

The precondition is ordinary: every real senior LP leaves the tranche, for example at wind-down or after a senior impairment, and the last redemption is funded during an oracle-frozen window.

## References


- `packages/perps/src/HousePool.sol:845-863`: senior redemption phase in `_settleLpEpoch`, with the net-only principal debit at `:858-861`
- `packages/perps/src/HousePool.sol:1147-1155`: `_settlementFeeBps`
- `packages/perps/src/HousePool.sol:250-257`: default pool configuration, with `seniorRateBps` 800 and `seniorFrozenLpFeeBps` 25
- `packages/perps/src/HousePool.sol:714-762`: `initializeSeedPosition`, with the seed floor set at `:760`
- `packages/perps/src/HousePool.sol:1205-1217`: `_juniorSettlementCapacity`
- `packages/perps/src/libraries/HousePoolRedemptionMathLib.sol:30-48`: `netAssetsForShares`
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:65`, `:119`: `paySeniorCoupon` and `scaleSeniorOnWithdraw`
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:57-60`: coupon guard in `planReconcile`
- `packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol:64-90`: `juniorWithdrawalRatioCap`
- `packages/perps/src/TrancheVault.sol:139`, `:322-324`, `:1328-1335`, `:1557-1572`, `:1630-1637`: `seedShareFloor`, the transfer and burn guard, `maxRequestRedeem`, `configureSeedPosition` and `_unlockedOwnerShares`
- `packages/perps/src/TrancheVault.sol:39`, `:545-556`: request cutoff and `_requestEpochWindow`
- `packages/perps/ACCOUNTING_SPEC.md:551-578`: Frozen-Window LP Fees
- `packages/perps/DEPLOYMENT.md:156`: release seed size
- `script/DeployPerpsArbitrumSepolia.s.sol:71-83`: `ArbitrumSepoliaReleaseHousePool`
- `audits/scan-plether-fi-plether-core-d4541b49.md`: "Permissionless frozen-state settlement can apply an unsnapshotted redemption fee"

## Proof of Concept

## Proof of Concept


### Scenario

The test uses the release parameters: a 1 USDG seed in each tranche, an 8% senior coupon, a 25 bps senior frozen fee, and senior caps of 40,000,000 USDG and 8,000 bps. One junior LP deposits 2,000,000 USDG and one senior LP deposits 1,000,000 USDG.

1. At 16:30 New York time on Friday 2024-03-08, while the market is open, the senior LP requests redemption of all its shares. The request targets the epoch that starts at the 17:00 close.
2. At the close the oracle is frozen, and epoch settlement funds the request at 25 bps below its gross value.
3. Only the seed's shares remain. The seed receiver cannot request a redemption or transfer a share, and the seed position cannot be initialized again.
4. One year later, a reconcile moves the 8% coupon from junior principal into the stranded senior balance. Junior withdrawal capacity stays a quarter of that balance short of junior principal.

### Test

Save as `packages/perps/test/perps/poc/solvency/SeedFloorStrandsFrozenExitFee.t.sol` and run from `packages/perps`:

```
forge test --match-path "test/perps/poc/solvency/SeedFloorStrandsFrozenExitFee.t.sol" -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../../shared/HousePoolTestBase.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice The frozen-oracle exit fee paid by the last real senior LP is retained in senior principal, which is then
///         owned only by the permanently locked senior seed. Junior LPs keep paying the senior coupon on it and must
///         keep a covenant reserve against it.
contract SeedFloorStrandsFrozenExitFeeTest is HousePoolAsyncTestBase {

    address internal constant SENIOR_SEED = address(0x5EED0001);
    address internal constant JUNIOR_SEED = address(0x5EED0002);
    address internal constant SENIOR_LP = address(0x5E410001);
    address internal constant JUNIOR_LP = address(0x7E410001);

    uint256 internal constant SENIOR_LP_ASSETS = 1_000_000e6;
    uint256 internal constant JUNIOR_LP_ASSETS = 2_000_000e6;

    /// @dev Friday 2024-03-08 21:30 UTC, 16:30 in New York, 30 minutes before the Friday FX close.
    uint256 internal constant FRIDAY_AFTERNOON = 1_709_933_400;

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev Release pool parameters (1 USDG seeds, 8% coupon, 25 bps senior frozen fee, 40M / 80% senior caps),
    ///      one junior LP and one real senior LP.
    function _releasePoolWithOneSeniorLp() internal returns (uint256 seniorLpShares) {
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(JUNIOR_LP, JUNIOR_LP_ASSETS);
        _fundSenior(SENIOR_LP, SENIOR_LP_ASSETS);
        seniorLpShares = seniorVault.balanceOf(SENIOR_LP);

        assertEq(pool.seniorFrozenLpFeeBps(), 25, "release senior frozen fee");
        assertEq(pool.seniorRateBps(), 800, "release senior coupon");
        assertEq(seniorVault.seedReceiver(), SENIOR_SEED, "senior seed receiver");
        assertEq(
            seniorVault.totalSupply(),
            seniorLpShares + seniorVault.seedShareFloor(),
            "senior supply is the LP plus the seed floor"
        );
    }

    /// @dev The last real senior LP requests a full exit on Friday afternoon. Its epoch cannot mature before the
    ///      22:00 UTC boundary, which is the Friday close, so it can only be funded under the frozen-oracle fee.
    function _lastSeniorExitsOverWeekend(
        uint256 seniorLpShares
    ) internal returns (uint256 grossAssets, uint256 paidAssets) {
        vm.warp(FRIDAY_AFTERNOON);
        assertFalse(engine.isOracleFrozen(), "the request is placed while the market is open");
        uint256 requestId = _requestAsyncRedeem(seniorVault, SENIOR_LP, seniorLpShares);

        vm.warp(pool.lpEpochStart(requestId));
        assertTrue(engine.isOracleFrozen(), "the request matures at the Friday close");
        grossAssets = seniorVault.convertToAssets(seniorLpShares);
        _settleLpEpochForTest();

        (, paidAssets) = _claimAsyncRedeem(seniorVault, requestId, SENIOR_LP);
    }

    function _log(
        string memory tag
    ) internal {
        uint256 seedShares = seniorVault.balanceOf(SENIOR_SEED);
        emit log_named_string("--- step", tag);
        emit log_named_uint("    senior total supply       ", seniorVault.totalSupply());
        emit log_named_uint("    senior seed share floor   ", seniorVault.seedShareFloor());
        emit log_named_uint("    senior seed share value   ", seniorVault.convertToAssets(seedShares));
        emit log_named_uint("    seed maxRequestRedeem     ", seniorVault.maxRequestRedeem(SENIOR_SEED));
        emit log_named_uint("    pool seniorPrincipal      ", pool.seniorPrincipal());
        emit log_named_uint("    pool juniorPrincipal      ", pool.juniorPrincipal());
        emit log_named_uint("    pool getMaxJuniorWithdraw ", pool.getMaxJuniorWithdraw());
    }

    function test_LastSeniorFrozenExitFeeIsStrandedInLockedSeedShares() public {
        uint256 seniorLpShares = _releasePoolWithOneSeniorLp();
        uint256 seedShares = seniorVault.balanceOf(SENIOR_SEED);
        uint256 seedValueBefore = seniorVault.convertToAssets(seedShares);
        _log("before the exit");

        (uint256 grossAssets, uint256 paidAssets) = _lastSeniorExitsOverWeekend(seniorLpShares);
        uint256 feeAssets = grossAssets - paidAssets;
        emit log_named_uint("    exiting LP gross value    ", grossAssets);
        emit log_named_uint("    exiting LP paid           ", paidAssets);
        emit log_named_uint("    retained frozen fee       ", feeAssets);
        _log("after the last senior exit");

        assertApproxEqAbs(feeAssets, grossAssets * 25 / 10_000, 1, "the exit pays the 25 bps frozen fee");
        assertEq(seniorVault.totalSupply(), seedShares, "only the seed shares remain");
        assertEq(seedShares, seniorVault.seedShareFloor(), "every remaining share sits under the floor");

        // Tolerance covers the ERC-4626 virtual offset: 1,000 virtual shares against 1e9 seed shares.
        uint256 seedValueAfter = seniorVault.convertToAssets(seedShares);
        assertApproxEqRel(
            seedValueAfter, seedValueBefore + feeAssets, 0.00001e18, "the retained fee now belongs to the seed shares"
        );
        assertGe(pool.seniorPrincipal(), feeAssets, "senior principal keeps the retained fee");

        assertEq(seniorVault.maxRequestRedeem(SENIOR_SEED), 0, "no seed share is redeemable");
        vm.prank(SENIOR_SEED);
        vm.expectRevert(
            abi.encodeWithSelector(TrancheVault.TrancheVault__ExceededMaxRequestRedeem.selector, SENIOR_SEED, 1, 0)
        );
        seniorVault.requestRedeem(1, SENIOR_SEED, SENIOR_SEED);

        vm.prank(SENIOR_SEED);
        vm.expectRevert(TrancheVault.TrancheVault__SeedFloorBreached.selector);
        seniorVault.transfer(address(0xBEEF), 1);

        settlementToken.mint(address(this), 1e6);
        settlementToken.approve(address(pool), 1e6);
        vm.expectRevert(IHousePool.HousePool__SeedAlreadyInitialized.selector);
        pool.initializeSeedPosition(true, 1e6, SENIOR_SEED);
    }

    function test_LockedSeniorSeedKeepsDrawingJuniorCoupon() public {
        uint256 seniorLpShares = _releasePoolWithOneSeniorLp();
        _lastSeniorExitsOverWeekend(seniorLpShares);

        uint256 seedShares = seniorVault.balanceOf(SENIOR_SEED);
        uint256 seniorBefore = pool.seniorPrincipal();
        uint256 juniorBefore = pool.juniorPrincipal();
        uint256 seedValueBefore = seniorVault.convertToAssets(seedShares);
        _log("after the last senior exit");

        vm.warp(vm.getBlockTimestamp() + 365 days);
        vm.prank(address(juniorVault));
        pool.reconcile();
        _log("one year later");

        uint256 couponAssets = pool.seniorPrincipal() - seniorBefore;
        emit log_named_uint("    coupon paid by junior     ", couponAssets);

        assertEq(juniorBefore - pool.juniorPrincipal(), couponAssets, "junior principal funds the coupon");
        assertApproxEqRel(couponAssets, seniorBefore * 8 / 100, 0.001e18, "8% a year on the stranded balance");
        assertApproxEqRel(
            seniorVault.convertToAssets(seedShares) - seedValueBefore,
            couponAssets,
            0.00001e18,
            "the whole coupon accrues to the locked seed shares"
        );
        assertEq(seniorVault.maxRequestRedeem(SENIOR_SEED), 0, "and stays unredeemable");
    }

    function test_JuniorMustKeepCovenantBackingForTheLockedSeniorSeed() public {
        uint256 seniorLpShares = _releasePoolWithOneSeniorLp();
        _lastSeniorExitsOverWeekend(seniorLpShares);
        vm.warp(vm.getBlockTimestamp() + 365 days);
        vm.prank(address(juniorVault));
        pool.reconcile();
        _log("one year later");

        uint256 seniorExposure =
            pool.seniorPrincipal() > pool.seniorHighWaterMark() ? pool.seniorPrincipal() : pool.seniorHighWaterMark();
        uint256 retainedJunior = pool.juniorPrincipal() - pool.getMaxJuniorWithdraw();
        emit log_named_uint("    locked senior exposure    ", seniorExposure);
        emit log_named_uint("    junior that cannot leave  ", retainedJunior);

        assertGt(seniorExposure, 2500e6, "the stranded senior balance is still there");
        assertGe(retainedJunior, seniorExposure / 4, "junior must keep a quarter of it as covenant backing");
    }

}
```

### Logs

At commit `a82d4fef`, default profile:

```
Ran 3 tests for test/perps/poc/solvency/SeedFloorStrandsFrozenExitFee.t.sol:SeedFloorStrandsFrozenExitFeeTest
[PASS] test_JuniorMustKeepCovenantBackingForTheLockedSeniorSeed() (gas: 2645390)
Logs:
  --- step: one year later
      senior total supply       : 1000000000
      senior seed share floor   : 1000000000
      senior seed share value   : 2702015389
      seed maxRequestRedeem     : 0
      pool seniorPrincipal      : 2702018091
      pool juniorPrincipal      : 1999453817527
      pool getMaxJuniorWithdraw : 1998778313004
      locked senior exposure    : 2702018091
      junior that cannot leave  : 675504523

[PASS] test_LastSeniorFrozenExitFeeIsStrandedInLockedSeedShares() (gas: 2807392)
Logs:
  --- step: before the exit
      senior total supply       : 999326455991749
      senior seed share floor   : 1000000000
      senior seed share value   : 1000674
      seed maxRequestRedeem     : 0
      pool seniorPrincipal      : 1000001000675
      pool juniorPrincipal      : 2000000999325
      pool getMaxJuniorWithdraw : 1750000749156
      exiting LP gross value    : 1000347031962
      exiting LP paid           : 997846164382
      retained frozen fee       : 2500867580
  --- step: after the last senior exit
      senior total supply       : 1000000000
      senior seed share floor   : 1000000000
      senior seed share value   : 2501866102
      seed maxRequestRedeem     : 0
      pool seniorPrincipal      : 2501868603
      pool juniorPrincipal      : 1999653967015
      pool getMaxJuniorWithdraw : 1999028499864

[PASS] test_LockedSeniorSeedKeepsDrawingJuniorCoupon() (gas: 2795786)
Logs:
  --- step: after the last senior exit
      senior total supply       : 1000000000
      senior seed share floor   : 1000000000
      senior seed share value   : 2501866102
      seed maxRequestRedeem     : 0
      pool seniorPrincipal      : 2501868603
      pool juniorPrincipal      : 1999653967015
      pool getMaxJuniorWithdraw : 1999028499864
  --- step: one year later
      senior total supply       : 1000000000
      senior seed share floor   : 1000000000
      senior seed share value   : 2702015389
      seed maxRequestRedeem     : 0
      pool seniorPrincipal      : 2702018091
      pool juniorPrincipal      : 1999453817527
      pool getMaxJuniorWithdraw : 1998778313004
      coupon paid by junior     : 200149488

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 9.68ms (9.08ms CPU time)
```

Amounts are in 6-decimal USDG atoms. The exiting LP's shares were worth 1,000,347.031962 USDG. The LP was paid 997,846.164382 USDG, and 2,500.867580 USDG stayed in senior principal. After the exit, the senior supply equals the seed floor and the seed's shares are worth 2,501.866102 USDG. The remaining 0.002501 USDG of the 2,501.868603 USDG senior principal is the ERC-4626 virtual offset. `maxRequestRedeem` for the seed receiver is zero. One year later, junior principal has paid 200.149488 USDG of coupon into the locked balance. Junior withdrawal capacity also stays 675.504523 USDG short of junior principal because of the covenant.

