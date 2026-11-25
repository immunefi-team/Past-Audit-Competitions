# \#99570 [SC-High] Frozen-window senior exit fee accrues to the floored senior seed and pins the last junior LP's redemption

**Submitted on Nov 14th 2026 at 09:39:06 UTC by @Nyksx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99570
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Summary

When the last non-seed senior capital exits in an LP epoch settled while the oracle is frozen, the 25 bps senior frozen LP fee stays in the senior tranche, where the only remaining holder is the floored senior seed. The junior covenant keeps requiring junior principal to back that unredeemable seed exposure, so the last junior LP's matured redemption is only partly funded and cannot be cancelled, and later `settleLpEpoch()` calls with no other pending LP work revert with `HousePool__NoLpEpochProgress`. 

## Finding Description

LP redemptions settled during `oracleFrozen` pay a tranche-local frozen fee (25 bps senior, 75 bps junior in the `HousePool` constructor). The fee is withheld from the redeemer and stays in the tranche, so it accrues pro rata to the senior shares that remain after the burn. Each tranche also holds a seed position whose floor shares cannot be redeemed, and whose floor can only be raised. If the shares redeemed in a frozen pass are all the remaining non-seed senior shares, the whole fee of that pass accrues to the senior seed. With the release parameters, a 200,000 USDG senior exit moves the senior seed from 1.00 USDG to 501.37 USDG.

The junior covenant treats that seed value like any other protected senior exposure and requires `max(S, H) * 2,000 / 8,000` of junior principal to stay behind. When the last junior LP exits, the request is funded only down to that floor. The remainder (one quarter of the seed exposure minus the junior seed) stays escrowed in a matured request that cannot be cancelled, and settlement makes no further progress for that request.

**1. The senior frozen fee is withheld and kept in senior principal and the high-water mark.** The senior phase is priced with the frozen fee, and only the net funded assets leave senior principal:

```solidity
// packages/perps/src/HousePool.sol:1147-1155 (a82d4fef)
function _settlementFeeBps(
    bool isSenior,
    bool oracleFrozen
) internal view returns (uint256) {
    if (!oracleFrozen) {
        return 0;
    }
    return isSenior ? poolConfig.seniorFrozenLpFeeBps : poolConfig.juniorFrozenLpFeeBps; // 25 bps for senior
}

// packages/perps/src/HousePool.sol:854-860 (a82d4fef)
seniorPhase = _fundRedemptionPhase(
    seniorVault, result.cutoffEpoch, seniorPhase, _settlementFeeBps(true, statusSnapshot.oracleFrozen)
);
if (seniorPhase.fundedAssets > 0) {
    HousePoolWaterfallAccountingLib.WaterfallState memory nextState =
        HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets); // net of fee
    _setWaterfallState(nextState);
```

```solidity
// packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:112-131 (a82d4fef)
/// @notice Removes senior principal and the same economic fraction of its high-water-mark entitlement.
/// ...
///      Retained fees stay protected, and junior is unchanged.
function scaleSeniorOnWithdraw(
    WaterfallState memory state,
    uint256 fundedAssets
) internal pure returns (WaterfallState memory nextState) {
    nextState = state;
    if (fundedAssets == 0) {
        return nextState;
    }
    nextState.seniorHighWaterMark -= Math.mulDiv(
        state.seniorHighWaterMark, fundedAssets, state.seniorPrincipal, Math.Rounding.Floor
    ); // H' = H - floor(H * A / P): the retained fee stays in H as well as in P
    nextState.seniorPrincipal -= fundedAssets;
}
```

**2. The senior seed cannot redeem its floor, and the floor cannot be lowered.**

```solidity
// packages/perps/src/TrancheVault.sol:322-324 (a82d4fef)
if (from == seedReceiver && from != address(0) && balanceOf(from) - amount < seedShareFloor) {
    revert TrancheVault__SeedFloorBreached(); // floored seed shares cannot be transferred or escrowed
}

// packages/perps/src/TrancheVault.sol:1630-1637 (a82d4fef), used by maxRequestRedeem() at :1328-1335
function _unlockedOwnerShares(
    address owner
) internal view returns (uint256 ownerShares) {
    ownerShares = balanceOf(owner);
    if (owner == seedReceiver && seedReceiver != address(0)) {
        ownerShares = ownerShares > seedShareFloor ? ownerShares - seedShareFloor : 0; // 0 for the seed
    }
}

// packages/perps/src/TrancheVault.sol:1567-1571 (a82d4fef), configureSeedPosition()
if (balanceOf(receiver) < floorShares || floorShares < seedShareFloor) {
    revert TrancheVault__InvalidSeedPosition(); // the floor can only grow
}
```

**3. The junior covenant counts the seed's exposure in full.** Junior funding is capped by `_juniorSettlementCapacity()`, which subtracts the junior principal needed to back `max(S, H)` at `maxSeniorShareBps`:

```solidity
// packages/perps/src/HousePool.sol:880-881 (a82d4fef)
} else {
    juniorPhase.budget = _juniorSettlementCapacity(freeAssets, _getWaterfallState());

// packages/perps/src/HousePool.sol:1205-1216 (a82d4fef)
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

// packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol:75-89 (a82d4fef), juniorWithdrawalRatioCap()
uint256 exposure = protectedExposure(seniorPrincipal, seniorHighWaterMark); // includes the seed-owned fee
// ...
uint256 requiredJunior =
    _saturatingMulDiv(exposure, BPS - maxSeniorShareBps, maxSeniorShareBps, Math.Rounding.Ceil);
return juniorPrincipal > requiredJunior ? juniorPrincipal - requiredJunior : 0; // 0 once J reaches the floor
```

**4. The partly funded junior request cannot be cancelled, and settlements with nothing else to process revert.**

```solidity
// packages/perps/src/TrancheVault.sol:893-897 (a82d4fef), cancelRedeemRequest()
RedeemEpoch storage epoch = redeemEpochs[requestId];
if (currentLpEpoch() >= requestId || epoch.fundedShares != 0 || epoch.fundedAssets != 0 || epoch.refundEnabled)
{
    revert TrancheVault__RedeemCancellationUnavailable(); // matured and partly funded
}

// packages/perps/src/HousePool.sol:1183-1185 (a82d4fef), _emitLpEpochSettled()
if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
    revert HousePool__NoLpEpochProgress(); // reached when the junior budget is zero and no other LP work is pending
}
```

**5. Governance can lower the requirement only through the 72-hour timelock.** `maxSeniorShareBps` is changed through a 72-hour timelock (packages/perps/SECURITY.md:100) and must stay below 10,000:

```solidity
// packages/perps/src/HousePool.sol:1708-1710 (a82d4fef)
if (config.maxSeniorShareBps >= 10_000) {
    revert HousePool__InvalidMaxSeniorShareBps(); // at 9,999 the requirement becomes ceil(exposure / 9,999)
}
```

The condition arises when a frozen settlement pass redeems all remaining non-seed senior shares, and junior LPs later redeem down to the covenant floor. Fees retained in earlier frozen passes are shared pro rata with the other senior holders, who take their part with them when they exit, so the seed captures the full fee only in the final frozen pass that leaves it as the only senior holder. The frozen junior amount is therefore about 6.25 bps of the senior principal redeemed in that pass, minus the junior seed. Frozen LP settlement is permissionless (`OrderRouter.settleLpEpoch()`), and a redemption whose epoch matures inside a weekend or holiday closure can be settled at the frozen fee. Senior coupon accrual also runs on the seed's increased principal: while no other junior capital exists, the 8% coupon is paid from the escrowed junior value, which falls from 124.45 USDG to 121.17 USDG over 30 days in the proof of concept.


## Impact Explanation

About 124.45 USDG of the last junior LP's redemption remains escrowed in a matured request that the LP cannot cancel or claim, and the proof of concept shows it still frozen after 30 days with no other LP activity. The freeze lasts at least until governance raises `maxSeniorShareBps` through the 72-hour timelock, or until a third party deposits new junior capital, which moves the same freeze to that depositor. The frozen amount is about 6.25 bps of the senior principal redeemed in the frozen pass that leaves only the seed, and only the last junior LP is affected. 

## Likelihood Explanation

No attacker or privileged action is needed. Oracle-frozen windows recur every weekend and on configured holidays, and frozen LP settlement is permissionless, so a keeper may settle a matured senior epoch inside such a window. The state additionally requires that the last non-seed senior exit settles in a frozen pass and that the junior tranche later winds down to its last LP, which corresponds to an early-stage or wind-down scenario. 

## Recommendation

Consider not charging the part of the senior frozen fee that would accrue to the floored seed shares, so that retained fees benefit only redeemable incumbents, as packages/perps/ACCOUNTING_SPEC.md:578 describes.

## Proof of Concept

## Proof of Concept

The test deploys the production `HousePool`, tranche vaults, engine, oracle and Router from `BasePerpTest` with the release risk parameters, 1 USDG senior and junior seeds, `maxSeniorExposureAssets = 40,000,000e6` and `maxSeniorShareBps = 8,000` (set through the owner's timelocked pool configuration), and the constructor's 25/75 bps frozen LP fees and 8% senior coupon. Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked. Every LP settlement is a permissionless keeper call to `router.settleLpEpoch()`, and all LP actions use the public ERC-7540 vault functions; no call pranks a protocol contract. The Router is `LegacyOrderRouterHarness`, a subclass of `OrderRouter` that only adds a commit helper not used here. A junior LP deposits 300,000 USDG and a senior LP deposits 200,000 USDG; the senior LP then redeems all of its shares, and the next Tuesday the junior LP redeems all of its shares.

1. `test_Control_LiveSeniorExit_LastJuniorExitsFully()` settles the senior exit on Thursday in a live market (0 bps fee). The seed stays at 1.00 USDG and the junior LP's full redemption is funded in one settlement.
2. `test_FrozenSeniorExit_PinsLastJuniorRedeem()` settles the same senior exit on Friday at 21:00:01 UTC (17:00 New York), when the oracle is frozen. It shows that the seed captures the fee, that junior funding stops exactly at the covenant floor, that the request cannot be cancelled, that `settleLpEpoch()` with no other pending LP work reverts with `HousePool__NoLpEpochProgress` after 1, 3 and 30 days, and that the escrowed junior value declines while the seed grows.
3. `test_NewJuniorDepositOnlyMovesThePin()` shows that a new 10,000 USDG junior deposit releases the first LP and that a similar amount is then frozen when the new LP redeems.

Command (from the repository root, forge 1.9.0, run against unmodified a82d4fef sources):

```
forge test --root packages/perps --match-path test/Submission_H02.t.sol -vv
```

Key output (excerpt: test results only, compiler warnings omitted; USDG atoms with 6 decimals):

```
Ran 3 tests for test/Submission_H02.t.sol:Submission_H02Test
[PASS] test_Control_LiveSeniorExit_LastJuniorExitsFully() (gas: 6587478)
Logs:
  senior seed value before senior exit (USDG atoms): 1001816
  senior LP paid: 200091324200
  senior seed value after senior exit: 1002274
  junior LP paid: 299908674538
  junior LP shares left in matured redeem: 0
  junior LP assets left in matured redeem: 0

[PASS] test_FrozenSeniorExit_PinsLastJuniorRedeem() (gas: 8456259)
Logs:
  senior seed value before senior exit (USDG atoms): 1001816
  senior LP paid: 199645753424
  senior seed value after senior exit: 501367345
  junior LP paid: 299729021664
  junior LP shares left in matured redeem: 124733610037
  junior LP assets left in matured redeem: 124447278
  senior protected exposure max(S, H): 501779929
  junior principal required by covenant: 125444983
  junior principal after settlement: 125444983
  senior seed value after 30 more days: 505078798
  junior LP escrowed assets after 30 more days: 121174145

[PASS] test_NewJuniorDepositOnlyMovesThePin() (gas: 9928908)
Logs:
  senior seed value before senior exit (USDG atoms): 1001816
  senior LP paid: 199645753424
  senior seed value after senior exit: 501367345
  junior LP paid: 299729021664
  junior LP shares left in matured redeem: 124733610037
  junior LP assets left in matured redeem: 124447278
  first junior LP released after the new deposit: 124438131
  second junior LP paid: 9875527503
  second junior LP assets now pinned: 124454226

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 10.85ms (15.42ms CPU time)

Ran 1 test suite in 11.24ms (10.85ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

The frozen senior exit pays 199,645.75 USDG instead of 200,091.32 USDG in the control, and the senior seed rises from 1.00 to 501.37 USDG. The covenant requires 125.44 USDG of junior principal (one quarter of the 501.78 USDG protected exposure), of which about 1 USDG is the junior seed, so 124.45 USDG of the junior LP's redemption remains escrowed. After 30 more days the escrowed value is 121.17 USDG and the seed is worth 505.08 USDG.

`packages/perps/test/Submission_H02.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice A senior exit settled while the oracle is frozen leaves its 25 bps frozen LP fee inside the senior
///         tranche. When the exiting LP is the last non-seed senior holder, the fee accrues to the floored senior seed.
///         The junior covenant keeps counting that unredeemable exposure, so the last junior LP's matured redeem is
///         only partly funded and cannot be cancelled, and later settlements with no other pending LP work revert
///         with NoLpEpochProgress.
///
///         Setup: the real HousePool, TrancheVaults, CfdEngine, PletherOracle and OrderRouter from BasePerpTest
///         (the Router is `LegacyOrderRouterHarness`, a subclass that only adds an unused test commit helper; the
///         `settleLpEpoch()` and `updateMarkPrice()` entrypoints called here are the production OrderRouter code).
///         Only Pyth (MockPyth) and USDG (MockSettlementToken) are mocked. Release parameters: risk params from
///         DEPLOYMENT.md, 1 USDG senior and junior seeds, maxSeniorExposureAssets = 40M, maxSeniorShareBps = 8,000,
///         and the HousePool constructor's frozen LP fees (senior 25 bps, junior 75 bps) and 8% senior coupon.
///         All LP settlements are permissionless keeper calls to `router.settleLpEpoch()`.
contract Submission_H02Test is BasePerpTest {

    address internal constant JUNIOR_LP = address(0x1F0000);
    address internal constant SENIOR_LP = address(0x1F0001);
    address internal constant JUNIOR_LP_2 = address(0x1F0002);
    address internal constant KEEPER = address(0x4EE700);

    uint256 internal constant TUE_11_00 = 1_710_241_200; // Tue 2024-03-12 11:00 UTC
    uint256 internal constant THU_14_00 = 1_710_424_800; // Thu 2024-03-14 14:00 UTC (live market, control)
    uint256 internal constant FRI_20_50 = 1_710_535_800; // Fri 2024-03-15 20:50 UTC (before the 20:55 cutoff)
    uint256 internal constant FRI_21_00 = 1_710_536_400; // Fri 2024-03-15 21:00 UTC = 17:00 New York (EDT): frozen
    uint256 internal constant TUE_NEXT_14_00 = 1_710_856_800; // Tue 2024-03-19 14:00 UTC (live market)

    uint256 internal constant MAX_SENIOR_SHARE_BPS = 8000;

    // ---------------------------------------------------------------------------------------------
    // Release configuration
    // ---------------------------------------------------------------------------------------------

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
        return 1e6; // release junior seed: 1 USDG
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6; // release senior seed: 1 USDG
    }

    function setUp() public override {
        super.setUp();
        // Release senior capacity (DEPLOYMENT.md): 40M absolute exposure and the 80% senior-share covenant.
        _setSeniorCapacity(40_000_000e6, MAX_SENIOR_SHARE_BPS);
        assertEq(pool.seniorFrozenLpFeeBps(), 25, "release senior frozen LP fee");
        assertEq(pool.seniorRateBps(), 800, "release senior coupon");
    }

    // ---------------------------------------------------------------------------------------------
    // Helpers (permissionless keeper and ordinary ERC-7540 LP actions only)
    // ---------------------------------------------------------------------------------------------

    function _payload(
        uint256 t
    ) internal pure returns (bytes[] memory d) {
        MockPyth.FeedUpdate[] memory ups = new MockPyth.FeedUpdate[](2);
        ups[0].id = BASE_PYTH_FEED_A;
        ups[0].price = MockPyth.MockPrice({price: 1e8, conf: 0, expo: -8, publishTime: t, prevPublishTime: t - 1});
        ups[1].id = BASE_PYTH_FEED_B;
        ups[1].price = MockPyth.MockPrice({price: 1e8, conf: 0, expo: -8, publishTime: t, prevPublishTime: t - 1});
        d = new bytes[](1);
        d[0] = abi.encode(ups);
    }

    function _at(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _keeperSettle() internal {
        bytes[] memory d = _payload(vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.settleLpEpoch(d);
    }

    function _deposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp
    ) internal {
        uint256 id = vault.controllerDepositHead(lp);
        uint256 claimable = vault.claimableDepositRequest(id, lp);
        vm.prank(lp);
        vault.claimDeposit(id, claimable, lp, lp);
    }

    function _claimRedeem(
        TrancheVault vault,
        address lp
    ) internal returns (uint256 assets) {
        uint256 id = vault.controllerRedeemHead(lp);
        uint256 shares = vault.claimableRedeemRequest(id, lp);
        if (shares == 0) {
            return 0;
        }
        vm.prank(lp);
        assets = vault.claimRedeem(id, shares, lp, lp);
    }

    /// @dev Junior principal the covenant requires: ceil(max(S, H) * (10,000 - Q) / Q) with Q = 8,000.
    function _requiredJunior() internal view returns (uint256 exposure, uint256 required) {
        uint256 s = pool.seniorPrincipal();
        uint256 h = pool.seniorHighWaterMark();
        exposure = s > h ? s : h;
        required = (exposure * (10_000 - MAX_SENIOR_SHARE_BPS) + MAX_SENIOR_SHARE_BPS - 1) / MAX_SENIOR_SHARE_BPS;
    }

    // ---------------------------------------------------------------------------------------------
    // Scenario steps
    // ---------------------------------------------------------------------------------------------

    /// @dev Step 1: a junior LP deposits 300,000 USDG, then a senior LP deposits 200,000 USDG.
    function _fundTranches() internal returns (uint256 seedSeniorShares) {
        _at(TUE_11_00);
        _deposit(juniorVault, JUNIOR_LP, 300_000e6);
        _at(TUE_11_00 + 1 hours + 1);
        _keeperSettle();
        _claimDeposit(juniorVault, JUNIOR_LP);
        _deposit(seniorVault, SENIOR_LP, 200_000e6);
        _at(TUE_11_00 + 2 hours + 1);
        _keeperSettle();
        _claimDeposit(seniorVault, SENIOR_LP);

        seedSeniorShares = seniorVault.balanceOf(address(this));
        assertEq(seniorVault.seedShareFloor(), seedSeniorShares, "the senior seed holds exactly its permanent floor");
        emit log_named_uint(
            "senior seed value before senior exit (USDG atoms)", seniorVault.convertToAssets(seedSeniorShares)
        );
    }

    /// @dev Step 2: the only non-seed senior LP requests a full exit at `requestAt`; a keeper settles at `settleAt`.
    function _seniorFullExit(
        uint256 requestAt,
        uint256 settleAt,
        bool expectFrozen,
        uint256 seedSeniorShares
    ) internal returns (uint256 seniorPaid, uint256 seedValue) {
        _at(requestAt);
        // Any keeper refreshes the engine mark (the frozen regime reuses this cached mark).
        bytes[] memory markUpdate = _payload(vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.updateMarkPrice(markUpdate);
        uint256 seniorShares = seniorVault.maxRequestRedeem(SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.requestRedeem(seniorShares, SENIOR_LP, SENIOR_LP);

        _at(settleAt);
        assertEq(engine.isOracleFrozen(), expectFrozen, "oracle regime at the senior settlement");
        _keeperSettle();
        seniorPaid = _claimRedeem(seniorVault, SENIOR_LP);
        assertEq(seniorVault.totalSupply(), seedSeniorShares, "only the floored seed remains in senior");
        assertEq(seniorVault.maxRequestRedeem(address(this)), 0, "the seed floor cannot be redeemed");
        seedValue = seniorVault.convertToAssets(seedSeniorShares);
        emit log_named_uint("senior LP paid", seniorPaid);
        emit log_named_uint("senior seed value after senior exit", seedValue);
    }

    /// @dev Step 3: next Tuesday (live market) the junior LP requests a full exit and a keeper settles it.
    function _juniorFullExit() internal returns (uint256 requestId, uint256 juniorPaid, uint256 stuckShares) {
        _at(TUE_NEXT_14_00);
        uint256 juniorShares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        requestId = juniorVault.requestRedeem(juniorShares, JUNIOR_LP, JUNIOR_LP);
        _at(TUE_NEXT_14_00 + 1 hours + 1);
        assertFalse(engine.isOracleFrozen(), "junior exit settles in a live market");
        _keeperSettle();
        juniorPaid = _claimRedeem(juniorVault, JUNIOR_LP);
        stuckShares = juniorVault.balanceOf(address(juniorVault)); // shares still escrowed in the matured request
        emit log_named_uint("junior LP paid", juniorPaid);
        emit log_named_uint("junior LP shares left in matured redeem", stuckShares);
        emit log_named_uint("junior LP assets left in matured redeem", juniorVault.convertToAssets(stuckShares));
    }

    // ---------------------------------------------------------------------------------------------
    // Control: the same exit sequence with the senior exit settled in a live market
    // ---------------------------------------------------------------------------------------------

    function test_Control_LiveSeniorExit_LastJuniorExitsFully() public {
        uint256 seedSeniorShares = _fundTranches();

        // Senior exit requested Thursday 14:00 UTC and settled after the 15:00 boundary, live market (0 bps fee).
        (, uint256 seedValue) = _seniorFullExit(THU_14_00, THU_14_00 + 1 hours + 1, false, seedSeniorShares);
        assertLt(seedValue, 2e6, "without a frozen fee the seed stays near its 1 USDG deposit");

        (uint256 requestId,, uint256 stuckShares) = _juniorFullExit();
        assertEq(stuckShares, 0, "control: the junior LP's full redeem is funded in one settlement");
        assertEq(juniorVault.claimableRedeemRequest(requestId, JUNIOR_LP), 0, "control: fully claimed");
        assertGt(pool.getMaxJuniorWithdraw(), 0, "control: junior withdrawal capacity remains");
    }

    // ---------------------------------------------------------------------------------------------
    // Finding: the senior exit settles while the oracle is frozen
    // ---------------------------------------------------------------------------------------------

    function test_FrozenSeniorExit_PinsLastJuniorRedeem() public {
        uint256 seedSeniorShares = _fundTranches();

        // Senior exit requested Friday 20:50 UTC; the epoch matures at 21:00 UTC (17:00 New York, EDT), when the
        // oracle is frozen, and a keeper settles it one second later.
        (uint256 seniorPaid, uint256 seedValue) = _seniorFullExit(FRI_20_50, FRI_21_00 + 1, true, seedSeniorShares);
        assertLt(seniorPaid, 200_000e6, "the 25 bps frozen fee is withheld from the senior LP");
        assertGt(seedValue, 500e6, "the withheld fee now belongs to the floored seed");

        (uint256 requestId, uint256 juniorPaid, uint256 stuckShares) = _juniorFullExit();
        uint256 stuckAssets = juniorVault.convertToAssets(stuckShares);
        (uint256 seedExposure, uint256 requiredJunior) = _requiredJunior();
        emit log_named_uint("senior protected exposure max(S, H)", seedExposure);
        emit log_named_uint("junior principal required by covenant", requiredJunior);
        emit log_named_uint("junior principal after settlement", pool.juniorPrincipal());

        assertGt(juniorPaid, 0, "the request is partly funded");
        assertGt(stuckAssets, 120e6, "about a quarter of the seed-owned senior value stays in the junior LP's request");
        assertEq(pool.juniorPrincipal(), requiredJunior, "funding stopped exactly at the covenant floor");
        assertEq(pool.getMaxJuniorWithdraw(), 0, "the covenant leaves zero junior withdrawal capacity");
        assertEq(juniorVault.claimableRedeemRequest(requestId, JUNIOR_LP), 0, "nothing more is claimable");
        assertFalse(juniorVault.redeemRefundPending(requestId, JUNIOR_LP), "no refund path is opened");

        // The matured request cannot be cancelled.
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        vm.prank(JUNIOR_LP);
        juniorVault.cancelRedeemRequest(requestId, JUNIOR_LP, JUNIOR_LP);

        // With no other pending LP work, later permissionless settlements revert: after 1 day, 3 days (the 72 h
        // governance delay) and 30 days.
        uint256[3] memory delays = [uint256(1 days), 3 days, 30 days];
        for (uint256 i; i < delays.length; i++) {
            _at(TUE_NEXT_14_00 + 1 hours + 1 + delays[i]);
            assertFalse(engine.isOracleFrozen(), "retry in a live market");
            bytes[] memory d = _payload(vm.getBlockTimestamp());
            vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
            vm.prank(KEEPER);
            router.settleLpEpoch(d);
            assertEq(juniorVault.balanceOf(address(juniorVault)), stuckShares, "junior LP shares still escrowed");
            assertEq(juniorVault.claimableRedeemRequest(requestId, JUNIOR_LP), 0, "still nothing claimable");
        }

        // Meanwhile the 8% senior coupon on the seed's inflated principal keeps accruing out of junior.
        uint256 seedValueLater = seniorVault.convertToAssets(seedSeniorShares);
        uint256 stuckAssetsLater = juniorVault.convertToAssets(stuckShares);
        emit log_named_uint("senior seed value after 30 more days", seedValueLater);
        emit log_named_uint("junior LP escrowed assets after 30 more days", stuckAssetsLater);
        assertGt(seedValueLater, seedValue, "the seed keeps growing from the coupon");
        assertLt(stuckAssetsLater, stuckAssets, "the coupon is paid out of the escrowed junior value");
    }

    /// @notice Without governance, fresh junior capital releases the request, and the pin then moves to the newest
    ///         junior LP.
    function test_NewJuniorDepositOnlyMovesThePin() public {
        uint256 seedSeniorShares = _fundTranches();
        _seniorFullExit(FRI_20_50, FRI_21_00 + 1, true, seedSeniorShares);
        (uint256 requestId,,) = _juniorFullExit();

        // A new junior LP deposits 10,000 USDG; activation settles first, then the stuck request is funded.
        _at(TUE_NEXT_14_00 + 2 hours);
        _deposit(juniorVault, JUNIOR_LP_2, 10_000e6);
        _at(TUE_NEXT_14_00 + 3 hours + 1);
        _keeperSettle();
        _claimDeposit(juniorVault, JUNIOR_LP_2);
        _at(TUE_NEXT_14_00 + 4 hours + 1);
        _keeperSettle();
        uint256 released = _claimRedeem(juniorVault, JUNIOR_LP);
        emit log_named_uint("first junior LP released after the new deposit", released);
        assertGt(released, 120e6, "fresh junior capital releases the first LP");
        assertEq(juniorVault.claimableRedeemRequest(requestId, JUNIOR_LP), 0, "first LP fully claimed");

        // The new junior LP now exits and is pinned by the same seed-owned senior exposure.
        _at(TUE_NEXT_14_00 + 6 hours);
        uint256 shares2 = juniorVault.maxRequestRedeem(JUNIOR_LP_2);
        vm.prank(JUNIOR_LP_2);
        juniorVault.requestRedeem(shares2, JUNIOR_LP_2, JUNIOR_LP_2);
        _at(TUE_NEXT_14_00 + 7 hours + 1);
        _keeperSettle();
        uint256 paid2 = _claimRedeem(juniorVault, JUNIOR_LP_2);
        uint256 stuck2 = juniorVault.convertToAssets(juniorVault.balanceOf(address(juniorVault)));
        emit log_named_uint("second junior LP paid", paid2);
        emit log_named_uint("second junior LP assets now pinned", stuck2);
        assertGt(stuck2, 120e6, "the pin moved to the newest junior LP");
        assertEq(pool.getMaxJuniorWithdraw(), 0, "junior withdrawal capacity is zero again");
    }

}
```
